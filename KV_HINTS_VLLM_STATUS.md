# vLLM KV Hints 实现现状梳理

> 版本：v1.0（2026-10-19）
> 取证基线：vLLM `main` @ `642c18d64c`（本地工作区实测）；GitHub 状态抓取日期 2026-10-19
> 取证方法：本地代码扫描（`grep kv_hints`）+ `git log` 合入历史 + GitHub issue/PR 页面逐一核实
> 关联文档：`KV_HINTS_TRIPLE_REPO_PLAN.md`（三仓方案）、`KV_HINTS_MOTOR_GUIDE.md`（Motor 接入协议）

---

## 0. 一句话结论

**传输层已合入、执行层零实现、OpenAI 入口零暴露；执行层的社区实现已呈"三方竞合"格局（RFC companion Draft / 独立实现 / 全家桶 Draft），全部未合入，契约尚未冻结。**

| 层 | 状态 | 证据 |
| --- | --- | --- |
| 传输协议（envelope） | ✅ 已合入 | `vllm/v1/kv_hints/protocol.py`（26 行，纯传输无语义） |
| Python 全链路传播 | ✅ 已合入 | `async_llm.py` → `input_processor.py` → `EngineCoreRequest` → `Request` |
| Rust gRPC 传播 | ✅ 已合入 | `inference.proto:60`（字段 17）+ convert/lower 全链路 |
| 消费者 | ⚠️ 仅透传/携带 | KVCR 外部透传、OffloadingConnector 携带，vLLM 内部零解释 |
| KV 事件 session_id | ✅ 已合入 | `kv_events.py:94`（Motor 闭环依赖） |
| 执行层（actions/dispatch/priority） | ❌ 无 | `actions.py`/`dispatch.py`/`kv_cache_priority.py` 不存在 |
| OpenAI/Anthropic HTTP 入口 | ❌ 无 | `vllm/entrypoints/` 零命中 |
| request-completion 生命周期钩子 | ❌ 无 | scheduler 无相关集成点 |
| COPY_TO / prefetch 执行 | ❌ 无（Draft） | [#58198](https://github.com/vllm-project/vllm/pull/58198) 未合入 |

---

## 1. 已落地部分（本地代码取证）

### 1.1 传输层协议

`vllm/v1/kv_hints/protocol.py`（全文 26 行）：

```python
class KvHintAction(msgspec.Struct, frozen=True):
    action_id: str
    action_type: str
    action_version: str
    # `payload` must contain JSON-compatible values. Action-specific schemas are
    # validated by the component implementing the action.
    payload: dict[str, Any]

class KvHintsEnvelope(msgspec.Struct, frozen=True):
    protocol_version: str
    message_id: str
    actions: list[KvHintAction]
```

- 纯传输结构，**无任何 action 语义**；payload 是 `dict[str, Any]`，schema 校验责任显式留给"实现该 action 的组件"（即未来的执行层）
- 导出：`vllm/v1/kv_hints/__init__.py`（`KvHintAction`、`KvHintsEnvelope`）
- 设计来源：[RFC #53421](https://github.com/vllm-project/vllm/issues/53421) → 实现合入 [PR #53423](https://github.com/vllm-project/vllm/pull/53423)（commit `986e217158`，"[Feature] Add first-class KV hints request envelope for programmatic KV management"）

### 1.2 Python 传播链路（每跳实测行号）

| 跳 | 位置 | 说明 |
| --- | --- | --- |
| 公开 API | `vllm/v1/engine/async_llm.py:414,600,733` | `AsyncLLM.generate(..., kv_hints=...)` |
| 请求装配 | `vllm/v1/engine/input_processor.py:387,554` | 注入 EngineCoreRequest |
| 引擎协议 | `vllm/engine/protocol.py:110` | `EngineCoreRequest.kv_hints` |
| 引擎桥接 | `vllm/v1/engine/llm_engine.py:241,282`、`vllm/v1/engine/__init__.py:164` | LLMEngine / EngineCoreRequest 定义 |
| 请求对象 | `vllm/v1/request.py:82,198,266` | `Request.kv_hints`（含 fork 复制） |

### 1.3 Rust gRPC 侧

- 协议定义：`rust/proto/inference.proto:60` — `KVHintsEnvelope kv_hints = 17;`
- 转换层：`rust/src/server/src/grpc/convert.rs:21`（`kv_hints_from_proto`，顶层请求字段直通）
- 内部传播：`engine-core-client/protocol/kv_hints.rs`、`llm/src/request.rs:53`、`text/src/lower.rs:67`（lowering 透传 + 专项测试）
- Python/Rust 兼容测试：`rust/src/engine-core-client/src/tests/python_compat.py:62`
- 注意：Rust OpenAI completions 路由（`routes/openai/completions/convert.rs:168`）当前硬编码 `kv_hints: None` —— **Rust 侧 HTTP 入口同样未暴露**

### 1.4 消费者现状（2 个，均无 vLLM 内执行）

| 消费者 | 位置 | 行为 |
| --- | --- | --- |
| KVCR tiering | `vllm/v1/kv_offload/tiering/kvcr/manager.py:605` | `on_new_request` 时 `submit_hint(request_id, hints=msgspec.to_builtins(hint))` **不透明透传**给外部 KVCR 服务；请求结束时 `discard_hint`。即 [RFC #59708](https://github.com/vllm-project/vllm/issues/59708) 所称 CASE 2（backend-owned opaque transport）的唯一现役实现 |
| OffloadingConnector | `vllm/distributed/kv_transfer/kv_connector/v1/offloading/scheduler.py:559` | `_create_req_context` 把 `req.kv_hints` 复制进 `ReqContext`（`v1/kv_offload/base.py:95`），供连接器侧读取；连接器自身**不解释**任何 action |

KVCR 适配器本体来自已合入的 [PR #53624](https://github.com/vllm-project/vllm/pull/53624)（commit `000c7df9ff`）；tiering 控制面线程化来自 [PR #58168](https://github.com/vllm-project/vllm/pull/58168)（commit `c5f8adc0b1`）。

### 1.5 KV 事件 session_id（Motor 闭环依赖）

- `vllm/distributed/kv_events.py:94` — KV 事件携带 `session_id: str | None` 字段
- 来自已合入的 [PR #51381](https://github.com/vllm-project/vllm/pull/51381)（commit `d9e2b5238d`，"[Core][KV Events] Echo session_id on GPU BlockStored events"）

### 1.6 测试现状

仅 3 处，全部只覆盖**传播语义**（不含任何执行语义）：

- `tests/v1/test_request.py:47` — EngineCoreRequest → Request 的 kv_hints 拷贝
- `tests/v1/kv_offload/tiering/test_kvcr_tier.py:353` — envelope 到 KVCR 的透传
- `tests/v1/kv_connector/unit/offloading_connector/test_scheduler.py:236` — req_context 保留 kv_hints

---

## 2. 未落地部分（缺口清单）

1. **执行层三件套**：`v1/kv_hints/actions.py`（typed action 契约 + 静态校验）、`v1/kv_hints/dispatch.py`（consumer 注册表/路由/失败隔离）、`v1/core/kv_cache_priority.py`（声明状态机）——[RFC #59708](https://github.com/vllm-project/vllm/issues/59708) 提议的文件布局在 main 上均不存在
2. **OpenAI/Anthropic HTTP 暴露**：`vllm/entrypoints/` 无 `kv_hints`/`agent_hint` 任何处理；Python `AsyncLLM.generate()` 与 Rust gRPC 是仅有的两个入口
3. **请求生命周期钩子**：scheduler 无 request-completion actions 应用点；block_pool 无牺牲块排序/声明清理集成
4. **COPY_TO / MOVE_TO**：无任何执行路径；最接近的机制 [PR #58198](https://github.com/vllm-project/vllm/pull/58198)（request-free CPU→GPU prefetch 原语）仍是 Draft
5. **action payload schema**：`payload: dict[str, Any]` 无 schema 校验、无版本协商、无失败隔离语义实现

---

## 3. 社区竞合格局（RFC / PR 全景）

### 3.1 RFC 链（主线演进）

| 编号 | 标题 | 状态 | 关系 |
| --- | --- | --- | --- |
| [RFC #51428](https://github.com/vllm-project/vllm/issues/51428) | Programmatic Session-Aware KV Cache Management | Open（karen-sy, 2026-08-07） | **总纲**。`KvHint{target, action: Share\|Prefetch\|Demote\|Pin\|Retain, bounds: TTL/priority/destination}`；三阶段路线含 SessionPrefixIndex |
| [RFC #53421](https://github.com/vllm-project/vllm/issues/53421) | First-Class, Orchestrator-Agnostic KV Hint Envelope | Open | 传输协议 RFC（总纲第一步）→ 已由 #53423 实现 |
| [PR #53423](https://github.com/vllm-project/vllm/pull/53423) | 传输层实现 | **已合入**（`986e217158`） | 即本文 §1.1 |
| [RFC #59708](https://github.com/vllm-project/vllm/issues/59708) | Modular and Extensible APIs & Execution Surface for Orchestrator-Driven KV Hint Policies | **Open，无人认领**（karen-sy, 2026-10-01，仅 1 条无关评论） | **执行层契约**：`vllm.copy_to` / `vllm.move_to` / `vllm.set_priority`，`action_version="1.0"`；`(action_type, action_version) → 唯一 consumer` 注册、冲突拒绝、per-action 失败隔离；CASE 1（vLLM 解释）/ CASE 2（backend 不透明透传）双通道。开放问题：带外 hint、请求路径内生成 hint、staleness/SessionPrefixIndexer |

### 3.2 执行层实现（三方竞合，均未合入）

| PR | 阵营 | 状态 | 方案要点 |
| --- | --- | --- | --- |
| [PR #59714](https://github.com/vllm-project/vllm/pull/59714) | NVIDIA（karen-sy），RFC #59708 companion | **Draft + needs-rebase**（10/06 起冲突），1 commit，零 review，E2E/perf "To be added" | `SetPriority{target, location, claim ID, revision, priority, TTL}`；`KvHintDispatcher` 挂 Scheduler（admission 校验 + 成功完成时执行）；`KVCachePriority` 挂 BlockPool（声明/重叠/过期/牺牲排序）；低优先级先驱逐、LRU 破平局 |
| [PR #38514](https://github.com/vllm-project/vllm/pull/38514) | Moreh（hyeongyun0916），源自旧 [RFC #37003](https://github.com/vllm-project/vllm/issues/37003) | **Open，活跃**（7 commits，10/18 push；ROCm 实测 +2~5p 命中率、TTFT −6~15%；hint-free 路径开销 ≈ 上游） | `kv.retain` action：token-range 指令 `{start, end, priority 1-100, duration, covers_output}`、per-scope hold、`PriorityEvictionQueue` sidecar 挂 BlockPool；**已实现 OpenAI 暴露**（chat/completions `kv_hints` 字段，畸形 400，作为 request metadata 传递）。作者明确表态：等 #59714 契约定型后**由他做对齐** |
| [PR #62382](https://github.com/vllm-project/vllm/pull/62382) | Mooncake 路线（zhhangBian），[RFC #57103](https://github.com/vllm-project/vllm/issues/57103) 参考实现 | **Draft**（10/19 开，29 commits，有冲突；前身 [#62380](https://github.com/vllm-project/vllm/pull/62380) 已关闭） | 范围最广：三原语 COPY_TO/MOVE_TO/SET_PRIORITY + dispatcher + CPU/Mooncake 执行 + Python/HTTP 独立控制面 + policy runtime + ACK/取消/epoch。自认"原语/dispatcher/priority 代码需与 #59714/#38514 收敛后才能上游合并"。2×A100 TP2 GSM8K 回放：受限 KV 下吞吐 +81~104%，working set 放得下时 −3.6%（控制开销） |

### 3.3 相邻机制（复用对象 / 竞争入口）

| 编号 | 内容 | 状态 | 与本方案关系 |
| --- | --- | --- | --- |
| [PR #58198](https://github.com/vllm-project/vllm/pull/58198) | request-free CPU→GPU prefetch 原语：`request_free_prefetch(block_hashes) → {ACCEPTED\|DEFERRED\|MISSING\|ALREADY_AVAILABLE\|UNSUPPORTED}` + `take_prefetch_completions()` | **Draft + needs-rebase**（yinli-systems, 2026-09-22） | M5 COPY_TO 的机器复用对象（目标预留/异步传输/完成事件三件套已就形） |
| [RFC #57103](https://github.com/vllm-project/vllm/issues/57103) | Programmable KV Cache：策略组合器 + 引擎统一资源审批（Retain/Pin/Prefetch/Demote/Share/Evict/Release），复用 #53421 envelope | Open（10/19 仍活跃） | 与 #59708 平行的另一路线 RFC；不建新传输层，押注策略层 |
| [RFC #52113](https://github.com/vllm-project/vllm/issues/52113) | Session-Aware KV Cache Hints：`agent_hint{session_id, cache_control{ephemeral, ttl}, context_management{offload/prefetch/evict}}`，引擎内 SessionAwareManager（SAM）+ 三段式 free list + 时间轮 TTL | Open | **OpenAI 入口的竞争方案**（`agent_hint` vs `kv_hints`）；request 级粒度、引擎内持有 session↔block 索引；点名 AscendStore 为首个 connector 集成候选 |
| [PR #47837](https://github.com/vllm-project/vllm/pull/47837) | priority-aware eviction（由调度优先级驱动） | 未合入 | #38514 声明的非重复对象之一 |
| [PR #48704](https://github.com/vllm-project/vllm/pull/48704) | 分层 offload 的 retention policy | 未合入 | 同上 |
| [agentic-api#195](https://github.com/vllm-project/agentic-api/pull/195) / [#359](https://github.com/vllm-project/agentic-api/issues/359) | server-managed prompt cache state + 前端集成 RFC | 外部仓库 | NVIDIA 方向：agent_hint 与 KvHint 收敛为一个 lifecycle contract，`cache_control` 只作协议层意图由控制面下沉 |
| [dynamo#13807](https://github.com/ai-dynamo/dynamo/pull/13807) | Dynamo 侧 SessionPrefixIndexer | 外部仓库 | #59708 开放问题 #3（staleness）的编排器侧解法 |

---

## 4. 尚未收敛的语义分歧（契约冻结前必须裁决）

1. **action 命名与寻址粒度**：`vllm.set_priority`（block id 集合，#59708/#59714）vs `kv.retain`（token range 指令，#38514）vs 策略组合器（#57103）
2. **OpenAI 入口**：`kv_hints` extra_body（#38514 路线）vs `agent_hint` 字段（#52113 路线）vs 仅 gRPC（现状）；三者并存即"两套竞争入口"风险
3. **保护语义实现**：优先级偏置驱逐队列（#59714）vs LRU 前置 drain + 优先级 sidecar 队列（#38514）vs 三段式 free list 重分类（#52113）——三者对"软保护"的实现机制完全不同
4. **执行时机**：request 成功完成时应用（#59714、RFC 建议的 hook）vs block 缓存命中时即 apply（#38514）
5. **session↔block 索引归属**：引擎内持有（SAM / SessionPrefixIndexer，#52113/#51428）vs 编排器持有（Motor KV Conductor 路线）——决定 staleness 由谁兜底
6. **重叠声明仲裁**：#59714 引入 claim ID + revision 模型；#38514 用 per-scope hold 取最大值；#52113 靠 ref 计数聚合——模型互不兼容

---

## 5. 对三仓方案的校准（M0 结论回填）

1. **M1 策略**：RFC #59708 契约未冻结且已有三个实现。**不新开竞争性 PR**。首选协同路径：评审/补强 #59714（结构即方案蓝本，且缺测试与 perf 证据），并跟踪 #38514 作者承诺的契约对齐。
2. **D-V2 修订**：SET_PRIORITY payload 按 #59714 实际字段（target, location, claim ID, revision, priority, ttl）定义，弃用方案 v0.1 的简化四元组。
3. **M2 竞合**：chat/completions `kv_hints` 通道已被 #38514 占位；`agent_hint` 被 #52113 占位。我们的暴露设计必须在 RFC #59708/#52113 讨论中对齐后落地，避免第三套入口。
4. **M5 复用**：COPY_TO 执行层应基于 #58198 的 prefetch 原语机制（目标预留/异步传输/完成事件）扩展，而非从零实现。
5. **Motor 桥（M3）**：翻译层产出 envelope、通道可替换的既定设计（R2 缓解）在当前格局下更加正确——上游入口二选一结果未出，Motor 侧两条都要能接。
6. **vllm-ascend 侧现状更新**（2026-10-19 调研）：AscendStore/KV Pool 生态活跃且 KV 事件已对接 conductor 消费（MLA effective block_size 对齐已有先行修复 vllm-ascend#16038）；NPU KV offload 已在跑（vllm-ascend#17539/#17932）；agent hint 已在 vllm-ascend 周会宣讲（2026.9.4，RFC #52113 co-author HiC4Sh1e）但零代码；vllm-ascend 另有本地 session 回收 RFC（vllm-ascend#17537 KVGC）需语义协调。

---

## 6. 引用链接汇总

**vLLM RFC**：[#51428](https://github.com/vllm-project/vllm/issues/51428)（总纲）· [#53421](https://github.com/vllm-project/vllm/issues/53421)（传输）· [#59708](https://github.com/vllm-project/vllm/issues/59708)（执行层）· [#57103](https://github.com/vllm-project/vllm/issues/57103)（可编程 KV）· [#52113](https://github.com/vllm-project/vllm/issues/52113)（agent_hint）· [#37003](https://github.com/vllm-project/vllm/issues/37003)（retention API）

**vLLM PR**：[#53423](https://github.com/vllm-project/vllm/pull/53423)（传输实现，已合入）· [#51381](https://github.com/vllm-project/vllm/pull/51381)（KV 事件 session_id，已合入）· [#53624](https://github.com/vllm-project/vllm/pull/53624)（KVCR 适配，已合入）· [#58168](https://github.com/vllm-project/vllm/pull/58168)（tiering 控制面，已合入）· [#59714](https://github.com/vllm-project/vllm/pull/59714)（G1 Retain，Draft）· [#38514](https://github.com/vllm-project/vllm/pull/38514)（kv.retain，Open）· [#62382](https://github.com/vllm-project/vllm/pull/62382)（可编程 KV 实现，Draft）· [#62380](https://github.com/vllm-project/vllm/pull/62380)（已关闭）· [#58198](https://github.com/vllm-project/vllm/pull/58198)（prefetch 原语，Draft）· [#47837](https://github.com/vllm-project/vllm/pull/47837) / [#48704](https://github.com/vllm-project/vllm/pull/48704)（相邻，未合入）

**外围**：[agentic-api#195](https://github.com/vllm-project/agentic-api/pull/195) · [agentic-api#359](https://github.com/vllm-project/agentic-api/issues/359) · [dynamo#13807](https://github.com/ai-dynamo/dynamo/pull/13807) · vllm-ascend：[#17537](https://github.com/vllm-project/vllm-ascend/issues/17537)（KVGC RFC）· [#16038](https://github.com/vllm-project/vllm-ascend/pull/16038)（conductor 对齐事件）· [周会议程 #10928](https://github.com/vllm-project/vllm-ascend/issues/10928)（2026.9.4 Agent Hint 议题）
