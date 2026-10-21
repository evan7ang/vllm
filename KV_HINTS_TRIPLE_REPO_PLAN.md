# KV Hint 三仓协同开发联调方案（Motor / vLLM / vLLM-Ascend）

> 版本：v0.2（2026-10-19）
> 基线：vLLM `main` @ `642c18d64c`；vllm-ascend `main`（CI 承诺 vLLM v0.31.0 tag）；MindIE-Motor 本地树
> 关联文档：`KV_HINTS_VLLM_STATUS.md`（上游 RFC/PR 现状权威梳理，随动态更新）、`KV_HINTS_MOTOR_GUIDE.md`（vLLM 侧接入协议）、Motor `docs/zh/design/20261009-01-agent-hint.md`
> 修订记录：v0.1 初稿；v0.2 回填 M0 社区调研结论（三方竞合格局、vllm-ascend 生态实况）、M1/M5a 策略修订、Motor 翻译桥双路径规划、新增 §8 三人并行分工方案

---

## 0. 目标与范围

**目标**：打通"Agent → Motor（编排）→ vLLM（执行）→ KV 事件回流 → Motor（路由闭环）"的块级 KV 管理链路，最终在昇腾 NPU 平台（vllm-ascend）上运行。

**范围**：

- vLLM：落地 RFC #59708 执行层（actions/dispatch/G1 优先级）+ OpenAI API 暴露 + offloading COPY_TO
- vllm-ascend：吸收上游 + NPU 平台适配（G1 验证、基于 AscendStore 既有路径的搬运层对接）
- Motor：agent_hint 协议 → KvHintsEnvelope 翻译桥 + 联调与观测

**非目标**（本期不做）：

- Mooncake/KVCR 等第三方 G2 后端的 NPU 移植（COPY_TO 联调用 SimpleCPUOffload/OffloadingConnector 内置 CPU 层）
- Motor Phase 5（prediction 预测性预热）
- 带外 hint 通道（依赖 RFC #59708 开放问题收敛，本期仅预留接口）

**开发模式**：三人分仓并行开发（§8），**契约先行**——第 0 周冻结三份接口契约后，三仓互不阻塞，最终按集成门统一联调。

---

## 1. 现状快照（v0.2 修订，依据 M0 调研）

### 1.1 vLLM

| 项 | 状态 |
| --- | --- |
| 传输层（KvHintsEnvelope/Action，PR #53423） | ✅ 已合入（`vllm/v1/kv_hints/protocol.py`，Python 全链路 + Rust gRPC 字段 17） |
| 执行层（actions/dispatch/SET_PRIORITY） | ❌ 未实现，且**三方竞合、契约未冻结**：PR [#59714](https://github.com/vllm-project/vllm/pull/59714)（RFC companion，Draft + needs-rebase，零 review）、[#38514](https://github.com/vllm-project/vllm/pull/38514)（kv.retain，活跃，作者承诺对齐 #59714 契约）、[#62382](https://github.com/vllm-project/vllm/pull/62382)（Mooncake 全家桶，Draft）；RFC [#59708](https://github.com/vllm-project/vllm/issues/59708) 开放且无人认领 |
| SET_PRIORITY 契约细节 | #59714 实际 payload：`{target, location, claim ID, revision, priority, TTL}`——claim/revision 用于同前缀多方声明的重叠仲裁（v0.1 的四元组定义作废） |
| OpenAI API 暴露 kv_hints | ❌ main 上无（仅 Python `AsyncLLM.generate()` / Rust gRPC 字段 17；Rust OpenAI completions 路由硬编码 `None`）。**上游存在两个竞争占位**：#38514 的 `kv_hints` 通道、RFC [#52113](https://github.com/vllm-project/vllm/issues/52113) 的 `agent_hint` 通道 |
| 消费者 | 仅 KVCR 透传（`v1/kv_offload/tiering/kvcr/manager.py:605`，CASE 2 不透明透传） |
| 调度侧平台无关性 | ✅ `v1/core/block_pool.py`、`kv_cache_manager.py` 零设备代码，NPU 可直接继承 |
| 搬运侧平台绑定 | ⚠️ `v1/kv_offload/cpu/gpu_worker.py`（`torch.cuda.Stream`/`is_xpu` 特判）、`host_register.py`（无 NPU 分支）；request-free prefetch 原语见 PR [#58198](https://github.com/vllm-project/vllm/pull/58198)（Draft，M5 复用对象） |

### 1.2 vllm-ascend（v0.1 的"生态缺席"判断已过时，重写）

| 项 | 状态 |
| --- | --- |
| 插件面 | `vllm.platform_plugins` entry point；覆盖 Platform/Worker/ModelRunner/attention backend；**不 fork 调度核心** |
| KV 搬运生态 | ✅ **活跃**：AscendStore/KV Pool 体系庞大（v1 重构 + 批量 transfer planning [#17822](https://github.com/vllm-project/vllm-ascend/pull/17822)、KV 事件发布 [#18849](https://github.com/vllm-project/vllm-ascend/pull/18849)、hybrid 传输规划、Mooncake connector 适配）；NPU KV offload 已在跑（CPU offload on Ascend [#17539](https://github.com/vllm-project/vllm-ascend/pull/17539)、NPU copy thread 修复 [#17932](https://github.com/vllm-project/vllm-ascend/issues/17932)/[#17933](https://github.com/vllm-project/vllm-ascend/pull/17933)） |
| KV 事件 → Conductor 闭环 | ✅ **已在生产验证**：[#16038](https://github.com/vllm-project/vllm-ascend/pull/16038) 修复 DSV4 Phase-1 offload 事件的 conductor-aligned effective block_size（MLA compress_ratio）——即本方案 R6 风险的实际先例，M4 直接复用其换算结论 |
| Agent Hint | 🟡 **意向明确、代码为零**：RFC #52113 co-author HiC4Sh1e 于 2026.9.4 周会宣讲 "Agent Hint For vllm-ascend"；全仓零实现（无撞车风险，有协同对象） |
| 相邻机制（需协调） | [#17537](https://github.com/vllm-project/vllm-ascend/issues/17537) KVGC（session 级僵尸块回收 RFC，env 门控平台 patch——与 S4 finalize 即释语义重叠，需收敛）；[#17844](https://github.com/vllm-project/vllm-ascend/pull/17844) A5 AI QoS（`kv_transfer_params.kv_priority` 是**传输路径**优先级，与 SET_PRIORITY **缓存驻留**优先级仅术语撞车，机制无关） |
| 功能矩阵 | 支持矩阵尚无 KV offloading 条目（A-4 待做）；APC 🟢、PD 分离 🟢、KVPP 🔵 |
| 版本策略 | `main` 对齐 vLLM main；`releases/vX.Y` 跟随 vLLM 发布；`rfc/<feature>` 分支用于特性协作 |

### 1.3 Motor

| 项 | 状态 |
| --- | --- |
| Agent Hint 协议 | ✅ Phase 1-3 已实现（`X-Agent-Hint` 头、session 亲和、软优先级） |
| cache_control 生命周期 | ✅ 协调器侧账本 + keeper（`domain/cache_declaration.py`），仅亲和 pin，**未触达引擎块级 KV** |
| KV Conductor | ✅ Rust 索引服务（HBM/CPU/Disk 三层，XXH3 block hash，ZMQ 订阅 KV 事件） |
| 引擎通道 | HTTP OpenAI 兼容（无 gRPC） |
| 已知卡点 | Phase 4 T-A2/A5 "依赖引擎消费能力确认（B1）"——正是本方案要解决的 |

**核心缺口总结（v0.2）**：Motor 的 hint 止步于协调器；vLLM 有信封无执行者且执行层契约未冻结；两者之间的桥（OpenAI 暴露 + 语义翻译）双方都缺，且 OpenAI 入口上游已有两个竞争占位。**对策**：内部契约先行（§8.2），翻译桥通道可替换，上游收敛前不押注单一入口形态。

---

## 2. 目标架构

```text
┌─────────────┐  X-Agent-Hint 头     ┌─────────────────────────────────────┐
│ Agent 框架   │ ──────────────────▶ │ Motor Coordinator                    │
│ (DSH/Pi/…)  │                     │  agent_hint 解析 → 声明账本(已有)      │
└─────────────┘                     │  【新增】KV Hint 桥:                  │
                                    │   声明+Conductor查询 → KvHintsEnvelope │
                                    └──────────────┬──────────────────────┘
                                                   │ OpenAI HTTP
                                                   │ (契约①: extra_body.kv_hints，
                                                   │  通道可替换，见 D-M4 双路径)
                                                   ▼
                                    ┌─────────────────────────────────────┐
                                    │ vLLM (vllm-ascend, NPU)              │
                                    │  ingress → Request.kv_hints (已有)    │
                                    │  【新增】dispatch → SET_PRIORITY 执行  │
                                    │  【新增】COPY_TO → offloading 执行    │
                                    │  block_pool 驱逐排序 (平台无关)        │
                                    └──────────────┬──────────────────────┘
                                                   │ KV events (ZMQ, 已有；
                                                   │  NPU 侧经 AscendStore 已验证)
                                                   ▼
                                    ┌─────────────────────────────────────┐
                                    │ Motor KV Conductor (已有)             │
                                    │  索引更新 → 缓存感知路由（闭环）         │
                                    └─────────────────────────────────────┘
```

**分层职责矩阵**（不变）：

| 层 | 归属 | 平台相关性 |
| --- | --- | --- |
| 协议层（envelope/action 类型、版本化） | vLLM | 无 |
| 语义翻译层（agent_hint → envelope） | **Motor** | 无 |
| dispatch/注册路由 | vLLM | 无 |
| G1 执行器（SET_PRIORITY、TTL、牺牲块排序） | vLLM | 无 |
| G2↔G1 搬运（COPY_TO/MOVE_TO worker） | 接口=vLLM（契约②），实现=平台插件（CUDA/XPU/NPU 各自） | **有** |
| API 暴露（OpenAI extra_body / gRPC） | vLLM | 无 |
| 块坐标/索引（target_blocks 来源） | Motor KV Conductor | 无 |
| 路由决策与命中观测 | Motor | 无 |

---

## 3. 里程碑与依赖关系（v0.2 修订）

```text
M0 社区对齐(✅已完成,结论沉淀) ──▶ M0.5 三份接口契约冻结 ──┬─▶ M1 vLLM 执行层(G1) ─▶ M2 OpenAI 暴露 ──┐
                                                          │                                          ├─▶ M4 三仓联调(G1 Retain) ─┐
                                                          ├─▶ M2a vllm-ascend 盘点/验证准备 ──────────┘                           │
                                                          ├─▶ M3 Motor 翻译桥(mock 自测) ─────────────────────────────────────────┤
                                                          └─▶ M5 COPY_TO ─▶ M5a NPU 搬运对接 ─▶ M6 三仓联调(Prefetch) ─▶ M7 端到端验收
```

| 里程碑 | 内容 | 前置 | 涉及仓库 |
| --- | --- | --- | --- |
| **M0** ✅ | 社区对齐调研：RFC #59708 无人认领、#59714/#38514/#62382 三方竞合、入口两占位、vllm-ascend 生态实况——**已完成（2026-10-19），结论沉淀于 `KV_HINTS_VLLM_STATUS.md`**；残余动作：按 §4 D-V1 协同策略执行社区互动 | 无 | vLLM |
| **M0.5** | **契约冻结**（新增）：契约① OpenAI 通道 wire schema（A×C）；契约② 设备无关 worker 接口（A×B）；契约③ KV 事件回流对齐（B×C）。见 §8.2 | M0 | 三仓 |
| **M1** | actions.py + dispatch.py + kv_cache_priority.py + block_pool 集成（G1 SET_PRIORITY）。**策略修订：协同 #59714（评审/补强/协助 rebase）或按其结构实现，不新开竞争 PR** | M0.5 | vLLM |
| **M2** | OpenAI 兼容 API 暴露 kv_hints（契约①通道）。上游入口收敛中（#38514 vs #52113），内部按契约先行、收敛后适配 | M1 | vLLM |
| **M2a** | vllm-ascend 吸收/验证：**前置放宽为"A 的可联调 feature 分支"（vllm-ascend 源码安装 vLLM，切分支成本低，无需等 release 合入）** | M1 分支可用 | vllm-ascend |
| **M3** | Motor KV Hint 桥：声明账本→envelope 翻译、通道注入、开关与观测。**前置放宽为契约①冻结即可全速并行（mock 引擎自测，不等 M2）** | M0.5 | Motor |
| **M4** | 三仓联调 · Retain 场景（NPU 部署） | M2a + M3 | 三仓 |
| **M5** | vLLM COPY_TO 执行层 + 设备无关 worker 接口（契约②落地；复用 [#58198](https://github.com/vllm-project/vllm/pull/58198) 的 prefetch 原语机制） | M1（与 M2-M4 并行支线） | vLLM |
| **M5a** | **内容修订：基于 AscendStore 既有搬运路径对接契约②**（盘点 → 适配/新增决策 → NPU 实现），不再从零写 host_register/stream | 契约②冻结 | vllm-ascend |
| **M6** | 三仓联调 · Prefetch 场景（pause→resume） | M4 + M5a | 三仓 |
| **M7** | 端到端验收 + 性能评估（命中率/驱逐行为/TTL 语义/aiperf） | M6 | 三仓 |

**关键路径**：M0 ✅ → M0.5 → 三线并行（A: M1→M2；B: 盘点→M5a；C: M3）→ M4 → M7。M5/M5a/M6 为并行支线，不阻塞 G1 主线上线。

---

## 4. vLLM 仓库工作分解

### 4.1 设计工作（M0 结论已回填）

| # | 设计项 | 要点 | 产出 |
| --- | --- | --- | --- |
| D-V1 | 社区协同执行（M0 结论转化） | RFC #59708 开放无人认领；#59714 Draft+needs-rebase 零 review 且缺测试/perf；#38514 活跃且作者承诺对齐 #59714 契约；#62382 全家桶 Draft 自认需收敛。**策略：优先评审/补强 #59714（补 CPU-only 契约测试与 perf 证据、协助 rebase）；不新开竞争 PR；在 RFC 下声明我们的差异化贡献（NPU 验证 + 编排器闭环证据 + OpenAI 暴露对齐）** | ✅ 已发帖（2026-10-20）：[#59714 评审](https://github.com/vllm-project/vllm/pull/59714#issuecomment-6305833875)、[#59708 意图声明](https://github.com/vllm-project/vllm/issues/59708#issuecomment-6305834117) |
| D-V2 | Action 契约设计（v0.2 修订） | `SET_PRIORITY{target_blocks, location, claim_id, revision, value, ttl_seconds}`（对齐 #59714 实际契约）：优先级只偏置驱逐不保证驻留；`value=null` 清除声明；**claim/revision 承担同前缀多方声明的重叠仲裁**；TTL 过期模型 | `actions.py` 类型+文档 |
| D-V3 | dispatch 注册设计 | `(action_type, action_version) → 唯一 consumer` 注册表；冲突拒绝；per-action 失败隔离（非法 payload 跳过不阻塞信封内其他 action）；为平台插件预留 executor 注册口（NPU/后端可注入自有实现，对应 RFC CASE 1/2 双通道） | `dispatch.py` 接口 |
| D-V4 | G1 状态设计 | `core/kv_cache_priority.py`：活跃声明、过期、重叠解析、上限、牺牲块排序；必须正确处理 cache hit/释放/复用/失效/reset 时刻（不依赖新 hint 到达） | 设计文档（RFC 附录或独立 md） |
| D-V5 | 生命周期钩子 | 请求完成时应用 request-completion actions（在释放 request 引用前）；不引入新的请求排序/准入策略 | block_pool 集成点定义 |
| D-V6 | OpenAI 暴露设计（v0.2 修订） | **三方入口竞合**：#38514 `kv_hints` 通道、#52113 `agent_hint` 通道、gRPC 字段 17（已合入）。内部按契约①（`extra_body.kv_hints`，JSON 形态与信封一一对应）先行实现；NVIDIA 已建议 agent_hint 与 KvHint 收敛为单一 lifecycle contract——**翻译桥保持通道可替换，收敛后适配** | API 提案 issue + 契约①文档 |
| D-V7 | 平台无关边界声明 | 明确"调度侧代码禁引入设备依赖；搬运侧一律走 `current_platform` 守卫/custom op/注册接口"。在 PR 描述中显式声明，作为评审基准 | PR 描述模板段落 |

### 4.2 实现工作

**Phase A（M1）：G1 SET_PRIORITY 执行层** —— 全部平台无关，CPU-only 可开发

| # | 文件 | 内容 |
| --- | --- | --- |
| I-V1 | `vllm/v1/kv_hints/actions.py` | `SetPriorityPayload` 等静态校验与 payload 类型（含 claim/revision）；不 import 调度器/分配器/后端 |
| I-V2 | `vllm/v1/kv_hints/dispatch.py` | consumer 注册表、envelope 解析、路由、失败隔离、action/job 关联 |
| I-V3 | `vllm/v1/core/kv_cache_priority.py` | 声明状态机：安装/续期/清除/过期；**claim/revision 重叠仲裁**；容量上限；victim 排序注入 |
| I-V4 | `vllm/v1/core/block_pool.py` | 牺牲块选择回调 + 块复用/失效时清理声明状态（小集成点） |
| I-V5 | `vllm/v1/core/sched/scheduler.py` | 请求完成钩子：应用 request-completion actions 后再释放引用 |
| I-V6 | `vllm/engine/protocol.py` 等 | EngineCoreRequest 携带解码后的 action 上下文（复用已有 `kv_hints` 字段，不重复信箱） |

### Phase B（M2）：OpenAI API 暴露（契约①通道）

| # | 文件 | 内容 |
| --- | --- | --- |
| I-V7 | `vllm/entrypoints/openai/protocol.py` | Chat/Completions 请求模型新增 `kv_hints`（extra_body 透传）；schema 校验失败 4xx、畸形 action 不使请求失败 |
| I-V8 | `vllm/entrypoints/openai/serving_chat.py` / `serving_completion.py` | 构造 `KvHintsEnvelope` 传入 `generate()` |
| I-V9 | （可选）Anthropic/gRPC 路由对齐 | Rust gRPC 已就绪（字段 17），Python 侧对齐即可 |

### Phase C（M5）：COPY_TO 执行层

| # | 文件 | 内容 |
| --- | --- | --- |
| I-V10 | `offloading/actions.py` | COPY_TO handler：目标解析、G1 预留、提交传输、completion 聚合（机制复用 [#58198](https://github.com/vllm-project/vllm/pull/58198) 的 `request_free_prefetch`/completion 设计） |
| I-V11 | `v1/kv_offload/` worker 接口抽象 | 把 `gpu_worker.py` 的设备操作收敛为可注册后端接口（**契约②，A×B 共签**；CUDA/XPU 现有实现迁移至此接口之下，NPU 实现由 vllm-ascend 注入）——M5 的核心设计产出，直接决定 M5a 的成本 |
| I-V12 | `host_register.py` | 平台分支重构：`cudaHostRegister`/`xpu_host_register`/NPU 等价物（接口留给插件） |
| I-V13 | completion 事件 | 传输完成后发布副本可查（"submitted ≠ ready"）；失败/取消时预留的释放路径 |

### 4.3 测试工作

| 层级 | 内容 | 运行环境 | 参考先例 |
| --- | --- | --- | --- |
| 单元（Phase A/B 全覆盖） | payload 静态校验（含 claim/revision）、注册冲突、失败隔离（未知 action/坏 payload/旧版本跳过）、声明仲裁（重叠/TTL/续期/清除）、TTL 过期惰性入队、block_pool 复用/失效清理声明、victim 排序生效、OpenAI schema（合法/畸形/未知字段） | **纯 CPU**（`tests/v1/core/` 全目录零 GPU 依赖为证） | `tests/v1/core/`、`tests/v1/test_request.py`、`tests/v1/kv_offload/`（mock+skipif 模式） |
| 集成（CPU） | envelope 全链路：OpenAI ingress → dispatch → 声明生效 → 驱逐顺序可观测（构造内存压力的确定性场景） | CPU | 现有 v1 core 集成测试风格 |
| 设备行为（GPU） | 真实内存压力下优先级偏置驱逐的端到端效果；与 APC/前缀命中交互；TTL 到期后恢复普通驱逐 | CUDA 实机（按需租用） | `.buildkite/` GPU job 配置 |
| 并发/安全 | 声明与 cache hit/free/reuse 竞态；copy 未完成不得可查（publication 语义）；预留计数精确释放 | CPU + GPU 各跑一轮 | RFC #59708 验证清单 |

**社区合规清单**（AGENTS.md 硬要求，每个 PR 必做）：

- [ ] 重复性检查三连（issue view / pr list 搜索）
- [ ] PR 描述：非重复声明 + 测试命令与结果 + AI 使用声明 + （若影响输出/精度）模型评估
- [ ] `pre-commit run --all-files`、行宽 88、Google docstring
- [ ] PR 前跑 `pr-checklist` skill
- [ ] 人工提交者逐行审阅（纯 code-agent PR 不被接受）

### 4.4 PR 拆分建议（按依赖序）

1. `actions.py` + `dispatch.py` + 单测（纯新增，最低争议）
2. `kv_cache_priority.py` + block_pool 集成 + SET_PRIORITY 消费者注册 + 单测
3. OpenAI API 暴露 + 单测
4. scheduler 生命周期钩子（request-completion actions）
5. COPY_TO：worker 接口抽象（重构 PR，行为不变）
6. COPY_TO：offloading handler + completion 事件

---

## 5. vLLM-Ascend 仓库工作分解

### 5.0 现状盘点（M2a/M5a 共同前置，v0.2 新增）

| # | 工作项 | 说明 |
| --- | --- | --- |
| B-0 | AscendStore/KV Pool 代码级盘点 | 摸清 [#17822](https://github.com/vllm-project/vllm-ascend/pull/17822)（v1 重构）、[#18849](https://github.com/vllm-project/vllm-ascend/pull/18849)（事件发布）、[#16038](https://github.com/vllm-project/vllm-ascend/pull/16038)（conductor 对齐）、[#17539](https://github.com/vllm-project/vllm-ascend/pull/17539)（NPU CPU offload）的代码现状；产出两个结论：① G1 验证预期零改动的证据清单；② M5a 对接点清单（既有 copy thread / host offload 路径 vs 契约②的差距） |

### 5.1 吸收与对齐（M2a）

| # | 工作项 | 说明 |
| --- | --- | --- |
| A-1 | main 分支对齐跟踪 | vLLM PR 合入后，vllm-ascend main 的 Ascend CI 自动看护；确认 kv-hint 相关改动不破坏 NPU 现有行为 |
| A-2 | release 分支策略 | G1 能力进入 vLLM vX.Y tag 后，vllm-ascend `releases/vX.Y` 吸收；**联调期直接用 vllm-ascend main + A 的 feature 分支源码组合（不等 release）** |
| A-3 | `rfc/kv-hints` feature 分支 | 按特性分支规范开 `rfc/kv-hints`；参与 #SIG-Ascend 周会对齐排期（周会议程帖 [#10928](https://github.com/vllm-project/vllm-ascend/issues/10928)） |
| A-4 | 支持矩阵更新 | 在 `docs/source/user_guide/support_matrix/supported_features.md` 增加 KV Hint / KV offloading 条目（G1 🟢 → COPY_TO 按实际进度） |

### 5.2 G1 验证（M2a，以验证为主、预期零代码）

| # | 验证项 | 方法 |
| --- | --- | --- |
| A-5 | 调度栈继承性 | NPU 上跑 A 分支新增的 CPU-only 单测（`tests/v1/core/`、`tests/v1/kv_hints/`）——理论上全绿，若失败即暴露插件与核心的耦合问题 |
| A-6 | NPU e2e 行为 | vllm-ascend 环境发送带 `SET_PRIORITY` 的请求，验证：声明前缀在内存压力下被保留、TTL 到期恢复驱逐、APC 命中不受影响 |
| A-7 | Graph Mode 交互 | ACL 图模式下调度侧行为一致性（调度侧不走图，但需确认无副作用） |

### 5.3 NPU 搬运层（M5a，v0.2 修订：对接 AscendStore 既有路径）

| # | 工作项 | 说明 |
| --- | --- | --- |
| A-8 | 契约② NPU 实现决策 | 基于 B-0 盘点结论，决定"适配既有 AscendStore/copy-thread 路径"还是"按契约②新增 NPU 后端"；输出对接设计（含 #17539/#17933 既有实现的复用面） |
| A-9 | stream/event 管理 | `torch.npu` stream 池与 event 对接契约②接口 |
| A-10 | copy kernel 对接 | `swap_blocks` 等价内核（TorchNPU 或 aclMemcpyAsync 路径），对接 `current_platform` 分发 |
| A-11 | NPU 单测 | 参照 `tests/v1/kv_offload/cpu/test_gpu_worker.py` 的 mock+skipif 模式，NPU CI 任务接入 |
| A-12 | e2e prefetch 验证 | G2→G1 COPY_TO 在 NPU 的正确性与性能（含部分作用域/失败取消场景） |
| A-13 | 事件链路验证（v0.2 新增） | 验证 AscendStore BlockStored 事件 → Conductor 消费的对齐性，复用 #16038 的 MLA effective block_size 换算结论；支撑 S6 staleness 场景与 C 侧闭环视图 |
| A-14 | 生态协调（v0.2 新增） | ① KVGC [#17537](https://github.com/vllm-project/vllm-ascend/issues/17537)：S4 finalize 即释语义收敛（避免两套机制并存）；② 与 HiC4Sh1e 对齐 agent_hint 路线（RFC #52113）；③ A5 QoS [#17844](https://github.com/vllm-project/vllm-ascend/pull/17844)：确认 `kv_priority`（传输 QoS）与 SET_PRIORITY（驻留声明）术语/机制边界 |

**风险预案**：优先复用 AscendStore 既有实现；若 TorchNPU 侧 pinned-memory 注册或异步 memcpy 能力受限，M5a 退化为同步拷贝实现（正确性优先，性能优化后置），COPY_TO 语义不受影响。

---

## 6. Motor 仓库工作分解

### 6.1 设计工作

| # | 设计项 | 要点 | 产出 |
| --- | --- | --- | --- |
| D-M1 | KV Hint 桥总体设计 | 定位：`cache_declaration`（意图账本）+ KV Conductor（块坐标来源）→ `KvHintsEnvelope`（引擎可执行载荷）。明确"协调器意图 ≠ 引擎执行结果"，引擎反馈靠 KV 事件闭环 | `docs/zh/design/` 新设计文档（延续 agent-hint 编号：`202610XX-0X-kv-hint-bridge.md`） |
| D-M2 | 语义映射（v0.2 修订） | `cache_control{type:pin, ttl_s, range}` → `vllm.set_priority{target_blocks, location:G1, claim_id, revision, value, ttl_seconds}`；**Motor 为每个声明生成稳定 claim_id、续期时 revision 递增**；TTL 双重钳制（解析层 [60,3600] + 引擎侧）以谁为准；`pin` 无 TTL（永久）是否入账（建议：暂不映射，等块级语义明确） | 映射表（进 D-M1 文档） |
| D-M3 | 块坐标换算 | `range{unit:message}` → `attach_block_offsets` 已有坐标 → KV Conductor 查询 → `target_blocks`（XXH3 LocalBlockHash 列表）；`unit=token` 直接换算；staleness 容错策略（目标缺失→跳过该 action + WARN，依赖 vLLM 失败隔离语义）；**块大小换算以引擎 `block_size` 为准，NPU 侧 MLA 场景参照 #16038 的 effective block_size 先例** | 换算与容错规则 |
| D-M4 | 通道决策（v0.2 修订：双路径矩阵） | **Path A（本期落地）**：OpenAI `extra_body.kv_hints`（契约①，依赖 M2 语义冻结而非代码合入）；**Path B（备选降级）**：`agent_hint` 透传（RFC #52113 路线，引擎内 SAM 自解析、免块坐标换算，但 request 级粒度且上游未实现）。桥只产 envelope、通道可替换；降级矩阵：引擎不支持 → 仅协调器亲和（今日行为）→ Path B（若上游采纳）→ Path A 双层生效。开关 `MOTOR_KV_HINT_BRIDGE`（延续 env 门控惯例） | 配置 schema + 双路径决策记录 |
| D-M5 | 带外清除 piggyback | keeper sweep 到期时若无在途请求，把 `set_priority(value=null, claim_id)` 挂到该 session 下一个请求；并记录 pending-until-next-request 状态（对接 RFC #59708 开放问题 #1，预留未来带外通道接口） | keeper 扩展设计 |
| D-M6 | 信任边界 | 引擎 payload 只含服务端换算的块坐标（剥离客户端伪造字段，延续 `_CACHE_SERVER_ONLY_FIELDS` 纪律）；`strict_priority` 仍不进入引擎通道 | 安全说明 |

### 6.2 实现工作

| # | 模块 | 内容 |
| --- | --- | --- |
| I-M1 | `motor/coordinator/domain/kv_hint_bridge.py`（新增） | `build_envelope(declaration, conductor_result) -> KvHintsEnvelope`；JSON 兼容构造；never-raise（畸形输入 WARN 丢弃） |
| I-M2 | `scheduler/runtime/scheduler_client.py` | 请求发出前：查询声明 + Conductor → 构造 envelope → 注入 OpenAI 请求 `extra_body.kv_hints`；keeper 扩展：到期/清除的 piggyback（D-M5） |
| I-M3 | `common/resources/http_msg_spec.py` / 请求组装层 | OpenAI 请求体新增 kv_hints 字段的透传与降级（引擎 4xx 时自动摘除重试一次 + WARN，防引擎版本不齐导致请求失败） |
| I-M4 | KV Conductor 协作接口 | `conductor_api_client.py` 增加按 session 前缀查询 block hash 的方法（若无现成接口）；结果带代际戳防 staleness |
| I-M5 | 观测 | 调度日志追加 `kvh=<n_actions,accepted>`；Prometheus 计数（构造/发送/引擎接受/失败隔离次数）；与既有 `prio=`/`aff=` 字段并列 |
| I-M6 | 引擎反馈闭环（只读） | 消费 KV 事件已有链路（Conductor ZMQ）上增加"声明前缀的驻留状态"视图，供验收与排障（不做控制动作） |

### 6.3 测试工作

| 层级 | 内容 | 参考 |
| --- | --- | --- |
| 单元 | 翻译层：声明→envelope 的全映射用例（pin/ttl_s/range 各形态、越界、畸形）；TTL 换算；block hash 换算正确性；claim/revision 生成与递增 | `tests/coordinator/domain/` 现有风格 |
| 单元 | keeper piggyback：到期→挂下一个请求→发送后清账；finalize 不二次发送 | `tests/coordinator/scheduler/runtime/test_cache_lifecycle.py` 扩展 |
| 集成 | **假引擎（mock OpenAI server，按契约①实现）**验证 extra_body 载荷与降级重试；golden vector 固定 envelope 形态（延续 Dify 插件互操作惯例） | `tests/coordinator/scheduler/test_cache_lifecycle_e2e.py` 风格 |
| e2e | 三 harness（DSH/Pi/Hermes）+ Motor + 真实 vLLM 联调场景（§7） | aiperf agent-hint benchmark 设计可复用 |

**Motor 侧验收红线**（延续既有原则）：默认开关关闭时请求路径与基线逐字节一致；畸形 hint 永不使请求失败；引擎不支持时完整降级。

---

## 7. 三仓联调方案（M4 / M6 / M7）

### 7.1 联调环境

| 拓扑 | 用途 | 阶段 |
| --- | --- | --- |
| Motor union 拓扑 + 单 vLLM 实例（NPU） | G1 Retain 基础联调 | M4 |
| Motor + PD 分离 + Conductor 三层索引 | 接近生产形态 | M6/M7 |
| GPU 环境（vLLM main + CPU offload） | COPY_TO 上游行为预验证（在 NPU 搬运层就位前先行验证语义） | M5 期间并行 |
| **Motor + mock 引擎（契约①假服务）** | C 侧全链路自测（不依赖 A 代码） | M0.5 后即可 |

### 7.2 联调场景矩阵

| # | 场景 | 注入方式 | 验证点 | 归属验证 |
| --- | --- | --- | --- | --- |
| S1 | 基线无 hint | 不发 hint | 请求行为与主干一致（引擎无消费者时静默无害） | Motor |
| S2 | Retain 保活 | DSH 发 `cache_control{pin, ttl_s=600}` | 内存压力下声明前缀不被驱逐；未声明会话正常 LRU | 三仓 |
| S3 | TTL 到期释放 | S2 后静置 > TTL | keeper piggyback 清除 → 前缀恢复可驱逐；KV 事件反映状态 | Motor+vLLM |
| S4 | finalize 即释 | 会话 finalize | 声明立即清除，keeper 不二次动作；与 vllm-ascend KVGC（#17537）语义协调 | Motor+vLLM |
| S5 | 失败隔离 | 混入未知 action_type/坏 payload/旧 action_version | 该 action 被跳过，其余 action 与请求本身不受影响；Motor 侧 WARN 计数 | vLLM |
| S6 | staleness | Motor 故意发送已失效块集 | 引擎显式报告缺失/部分作用域，不扩大执行范围；Motor 依事件更新索引（NPU 侧 MLA block_size 对齐参照 #16038） | 三仓 |
| S7 | Prefetch（M6） | agent_status=pause → resume | `copy_to(G2→G1)` 完成后前缀可查、命中；失败/取消时预留正确释放 | 三仓 |
| S8 | 降级矩阵 | 引擎回滚到无 kv-hint 版本 | Motor 自动摘除 kv_hints 重试成功，退化为今日行为 | Motor |
| S9 | 多轮压测 | aiperf agent-hint benchmark | 吞吐/TTFT 无回退；声明会话命中率提升量化 | 三仓 |

### 7.3 验收指标（M7）

- **功能**：S1-S8 全绿；S9 无性能回退（TTFT/TPOT/吞吐偏差 < 噪声带）
- **效果**：对照无 Retain 基线，声明会话的前缀缓存命中率提升（目标值在联调时按工作负载定标；#38514 的实测基线可作参考：命中率 +2~5p、TTFT −6~15%）
- **可观测**：Motor `kvh=` 指标与 vLLM 侧声明状态视图对账一致；KV 事件驱动的 Conductor 索引与引擎实际驻留一致
- **平台**：NPU 与 GPU 行为差异仅限搬运层性能，语义一致

### 7.4 联调分工（v0.2 新增，与 §8 对应）

| 集成门 | 内容 | Owner | 参与 |
| --- | --- | --- | --- |
| G1（M4 前） | Retain 全链路：契约①通道真引擎贯通、S1-S6 | **C**（场景执行与对账） | A（上游正确性）、B（NPU 环境） |
| G2（M6 前） | Prefetch 全链路：S7 + M5a 设备验证 | **B**（平台） | A、C |
| G3（M7） | 验收：S8-S9 + 指标对账 | **C** | 三人 |

---

## 8. 三人并行分工方案（v0.2 新增）

### 8.1 角色与任务分配

一人主导一仓（对应该仓 PR 的人工提交者与终审责任人），任务条目引用前文编号：

### 同事 A —— vLLM 上游执行层

| 优先级 | 任务 | 内容 |
| --- | --- | --- |
| P0 | D-V1 | 执行社区协同策略（评审/补强 #59714，RFC 下声明意图） |
| P0 | Phase A（I-V1~I-V6） | G1 SET_PRIORITY 执行层，CPU-only |
| P0 | Phase B（I-V7~I-V9） | OpenAI 契约①通道暴露 |
| P1 | 契约② + I-V11/I-V12 | worker 接口抽象（与 B 共签后实施） |
| P2 | I-V10/I-V13 | COPY_TO handler + completion 事件（支线，带宽不足可后置） |

### 同事 B —— vllm-ascend 平台适配

| 优先级 | 任务 | 内容 |
| --- | --- | --- |
| P0 | B-0 | AscendStore/KV Pool 代码级盘点（产出 G1 零改动证据 + M5a 对接点清单） |
| P0 | A-3/A-4/A-14 | `rfc/kv-hints` 分支、支持矩阵、生态协调（KVGC/HiC4Sh1e/A5 QoS） |
| P1 | 契约② NPU 侧 + A-8~A-11 | 按契约②先写 NPU 实现 + mock 单测（不等 A 代码） |
| P1 | A-13 | 事件链路验证（MLA block_size 对齐） |
| P2 | A-5~A-7 | G1 验证（A 的 M1 分支就绪后立即执行，脚本先行准备） |
| P2 | A-12 | e2e prefetch 验证 |

### 同事 C —— Motor 翻译桥 + 联调集成（整合者）

| 优先级 | 任务 | 内容 |
| --- | --- | --- |
| P0 | D-M1~D-M6 | 桥设计文档 + 双路径决策记录 |
| P0 | I-M1/I-M2/I-M4 | 翻译桥、keeper piggyback、Conductor 查询 |
| P0 | 契约① | wire schema 起草（与 A 共签） |
| P1 | I-M3/I-M5/I-M6 | HTTP 透传降级、观测、反馈视图 |
| P1 | mock 引擎 harness | 按契约①实现假 OpenAI server，C 全部工作在其上自测闭环 |
| P2 | §7 联调 | 联调手册、S1-S9 执行、aiperf 基准、验收对账（G1/G3 owner） |

### 8.2 接口契约先行（并行开发的前置门）

第 0 周由三人共同签署三份契约，冻结后各自开发不再互等：

| 契约 | 签署方 | 内容 | 冻结依据 |
| --- | --- | --- | --- |
| ① OpenAI 通道 wire schema | A × C | `extra_body.kv_hints` 的 JSON 形态 = `KvHintsEnvelope` 信封结构（RFC #53421 已定型）+ `vllm.set_priority` payload 字段（`target_blocks/location/claim_id/revision/value/ttl_seconds`）+ 畸形处理语义（4xx vs 跳过） | RFC #53421/#59708 文本 + `protocol.py` 现有类型 |
| ② 设备无关 worker 接口 | A × B | I-V11 的方法签名（内存注册/stream/copy/completion 查询）、错误语义、publication 规则；CUDA 实现归 A、NPU 实现归 B | `gpu_worker.py` 现有操作面 + #58198 原语设计 |
| ③ KV 事件回流 | B × C | 事件字段消费约定（session_id、block hash、effective block_size）、Conductor 查询接口的代际戳语义 | PR #51381 + vllm-ascend #16038 先例 |

**契约变更规则**：冻结后任何变更必须在三人周会公告并同步修订本文档；C 侧 golden vectors 与 B 侧 mock 单测随之更新。

### 8.3 并行时序与集成门

```text
第0周        三人：契约①②③冻结（1-2 天）──────────────┐
                                                       │
同事 A: M1 Phase A ──▶ M2 Phase B ──▶ (P1)契约②落地 ─▶ (P2)COPY_TO
同事 B: B-0盘点 ──▶ A-3/4/14 ──▶ M5a(mock单测) ──▶ A-5/6/7 验证 ──▶ A-12
同事 C: D-M设计 ──▶ I-M1/2/4 ──▶ mock harness 自测 ──▶ I-M3/5/6 ──▶ 联调手册
                                                       │
集成门 G1(M4): A 的 M1+M2 分支 × B 的 NPU 环境 × C 的 Motor 桥 → S1-S6
集成门 G2(M6): + M5 Phase C × M5a → S7
集成门 G3(M7): S8-S9 + 验收对账
```

**关键机制**：

1. **C 永不等 A 的真实代码**：mock 引擎按契约①实现，C 的开发与自测在 M0.5 后即全速推进
2. **B 用分支不用 release**：vllm-ascend 源码安装 vLLM，A 的 feature 分支就绪即可切入验证（M2a 前置从"合入 release"放宽为"分支可用"）
3. **A 的 M5 支线可让路**：M1+M2 是主线承诺；COPY_TO（Phase C）若带宽不足整体后置，不影响 G1 集成
4. **上游漂移隔离**：A 在 PR 层吸收上游契约变化（D-V1 持续跟踪），B/C 只面向契约①②编程，上游变动不外溢

### 8.4 协作节奏

- **周会**：每周一次 30 分钟三人对齐（契约变更、分支状态、阻塞项）；日常异步（文档/PR 评论）
- **分支约定**：A 在 vLLM fork 开 `kv-hints-g1` 分支序列；B 在 vllm-ascend `rfc/kv-hints`；C 在 Motor feature 分支
- **联调环境**：NPU 资源由 B 管理；联调手册与场景脚本由 C 维护在 Motor 仓

---

## 9. 风险与开放问题（v0.2 修订）

| # | 风险 | 影响 | 缓解 |
| --- | --- | --- | --- |
| R1 | RFC #59708 评审演进 / 三方竞合走向不定（#59714 补强 or #38514 对齐后合入） | M1 接口返工 | M0 已完成（结论见 `KV_HINTS_VLLM_STATUS.md`）；A 执行 D-V1 协同策略；实现严格按 RFC 已定型契约（wire 名、语义），接口薄、状态机独立成模块 |
| R2 | OpenAI 入口三方竞合（#38514 `kv_hints` / #52113 `agent_hint` / gRPC） | M2/M3 卡住 | 契约①先行 + 桥通道可替换（Path A/B 双路径，D-M4）；上游收敛后仅适配层改动 |
| R3 | 版本耦合窗口（vllm-ascend release 分支吸收滞后） | M4 推迟 | 已放宽：联调期用 vllm-ascend main + A 的 feature 分支源码组合；生产化等待 release 对齐 |
| R4 | NPU 搬运层能力受限（pinned memory/异步 memcpy） | M5a 延期 | 优先复用 AscendStore 既有路径（B-0 盘点）；退化为同步拷贝实现保语义 |
| R5 | 带外 hint（无请求附着）需求提前出现 | keeper piggyback 覆盖不足 | 明确本期范围：hint 全部请求附着；带外通道列为 vLLM 侧后续 RFC 议题 |
| R6 | Motor 侧块坐标与引擎块大小不一致 | target_blocks 错位 | 换算统一以引擎 `block_size` 为准；**NPU MLA 场景直接复用 vllm-ascend #16038 的 effective block_size 先例**（已验证） |
| R7 | 开发期内上游契约漂移（三方竞合任何一方合入都会改契约） | B/C 侧返工 | 契约①②③隔离：B/C 只面向契约编程；A 在 PR/适配层吸收上游变化；周会同步机制 |
| R8 | vllm-ascend 内部路线撞车（KVGC #17537 与 S4 语义重叠；agent_hint 路线 HiC4Sh1e 团队可能启动） | 双实现并存、联调混乱 | A-14 生态协调前置：KVGC 收敛到上游机制或明确分工；agent_hint 路线与 C 的 Path B 备选对齐 |

---

## 10. 附录

### 10.1 关键代码位置速查

**vLLM**（基线 `642c18d64c` 实测）

- 信封类型：`vllm/v1/kv_hints/protocol.py`；导出：`vllm/v1/kv_hints/__init__.py`
- Python 传播：`v1/engine/async_llm.py:414` → `v1/engine/input_processor.py:387,554` → `engine/protocol.py:110` → `v1/request.py:198`
- 消费者：`v1/kv_offload/tiering/kvcr/manager.py:605`（透传）、`distributed/kv_transfer/kv_connector/v1/offloading/scheduler.py:559`（携带）
- 搬运平台绑定：`v1/kv_offload/cpu/gpu_worker.py`、`host_register.py`、`swap_blocks_triton.py`
- KV 事件：`vllm/distributed/kv_events.py:94`（session_id 字段）
- Rust：`rust/proto/inference.proto:60`（字段 17）、`rust/src/server/src/grpc/convert.rs:21`；OpenAI completions 路由硬编码 `kv_hints: None`

### vllm-ascend

- 支持矩阵：`docs/source/user_guide/support_matrix/supported_features.md`（KV offloading 条目待加）
- AscendStore：`vllm_ascend/distributed/kv_transfer/kv_pool/ascend_store/`（事件聚合/连接器）；KVGC 提案模块：`vllm_ascend/core/kvgc/`（RFC #17537，未合入）
- 事件→Conductor 对齐先例：PR #16038（MLA effective block_size）
- 分支策略：`main`（对齐 vLLM main）/ `releases/vX.Y` / `rfc/<feature>`；周会议程：issue #10928

### Motor

- hint 解析：`motor/coordinator/domain/agent_hint.py`、`agent_hint_wire.py`
- 声明账本：`motor/coordinator/domain/cache_declaration.py`；keeper：`coordinator/scheduler/runtime/scheduler_client.py`
- 块坐标：`attach_block_offsets`（server 端填充，`_CACHE_SERVER_ONLY_FIELDS` 纪律）
- 索引服务：`motor/kv_conductor/`（Rust：HBM radix tree / CPU/Disk continuation edges / XXH3 hashing）
- 引擎探活：`coordinator/api_client/native_engine_api_client.py`

### 10.2 RFC / PR / 文档索引（v0.2 补全）

| 类型 | 编号 | 内容 | 状态 |
| --- | --- | --- | --- |
| vLLM RFC | [#51428](https://github.com/vllm-project/vllm/issues/51428) | Programmatic Session-Aware KV Cache Management（总纲） | Open |
| vLLM RFC | [#53421](https://github.com/vllm-project/vllm/issues/53421) / [PR #53423](https://github.com/vllm-project/vllm/pull/53423) | 信封传输层 | **已合入**（commit `986e217158`） |
| vLLM RFC | [#59708](https://github.com/vllm-project/vllm/issues/59708) / [PR #59714](https://github.com/vllm-project/vllm/pull/59714) | 执行层原语 + G1 Retain（本方案 M1/M5 依据） | Open（无人认领）/ Draft |
| vLLM PR | [#38514](https://github.com/vllm-project/vllm/pull/38514) | kv.retain 实现（含 OpenAI `kv_hints` 通道；作者承诺对齐 #59714 契约） | Open，活跃 |
| vLLM PR | [#62382](https://github.com/vllm-project/vllm/pull/62382) / #62380 | Mooncake RFC #57103 实现（全家桶）/ 其前身 | Draft / Closed |
| vLLM RFC | [#57103](https://github.com/vllm-project/vllm/issues/57103) | Programmable KV Cache（Mooncake 路线） | Open |
| vLLM RFC | [#52113](https://github.com/vllm-project/vllm/issues/52113) | agent_hint schema（竞争性 OpenAI 入口；Path B 依据） | Open |
| vLLM PR | [#58198](https://github.com/vllm-project/vllm/pull/58198) | 无请求 G2→G1 prefetch 原语（COPY_TO 机器复用对象） | Draft |
| vLLM PR | [#51381](https://github.com/vllm-project/vllm/pull/51381) | KV 事件 session_id tagging（Motor 闭环依赖） | **已合入**（commit `d9e2b5238d`） |
| vLLM PR | [#53624](https://github.com/vllm-project/vllm/pull/53624) | KVCR 二级缓存适配 | **已合入**（commit `000c7df9ff`） |
| vllm-ascend | [#17537](https://github.com/vllm-project/vllm-ascend/issues/17537) / [#16038](https://github.com/vllm-project/vllm-ascend/pull/16038) / [#17539](https://github.com/vllm-project/vllm-ascend/pull/17539) | KVGC RFC / conductor 事件对齐 / NPU CPU offload | Open / Open / Open |
| Motor 设计 | `20261009-01` ~ `20261015-03` | agent-hint Phase 1-4 + cache_control 生命周期 | — |
| 本仓库 | `KV_HINTS_VLLM_STATUS.md` | **上游现状权威梳理（M0 结论沉淀，随动态更新）** | — |
| 本仓库 | `KV_HINTS_INTEGRATION_GUIDE.md` | **vLLM 侧实现交付 + B/C 对接说明（T0-T6 完成后沉淀）** | — |
| 本仓库 | `KV_HINTS_UPSTREAM_DRAFTS.md` | **D-V1 上游评论草稿（#59714 评审 + #59708 意图声明，待人工发帖）** | — |
| 本仓库 | `KV_HINTS_MOTOR_GUIDE.md` | vLLM 侧现状与 Motor 接入协议 | — |
