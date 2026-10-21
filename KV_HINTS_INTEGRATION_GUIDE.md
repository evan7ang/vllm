# KV Hints vLLM 侧对接文档（同事 A → B / C）

> 版本：v1.0（2026-10-20）
> 代码位置：vLLM fork 分支 `kv-hints-g1`（基于 main `81f7c69577`，未提交、待拆 PR）
> 关联文档：`KV_HINTS_TRIPLE_REPO_PLAN.md`（三仓方案）、`KV_HINTS_VLLM_STATUS.md`（上游现状）、`KV_HINTS_MOTOR_GUIDE.md`（Motor 接入协议）
> 读者：同事 B（vllm-ascend 平台适配）、同事 C（Motor 翻译桥 + 联调）

---

## 0. 一句话总览

**方案内全部代码项已完成并通过本机（无 GPU）验证**：G1 SET_PRIORITY 执行层（#59714 结构）、OpenAI `extra_body.kv_hints` 通道（契约① v0）、设备无关 worker 接口（契约② v0）、G2→G1 COPY_TO 执行层（#58198 原语机制）。B/C 可按本文档面向契约编程，上游收敛后在适配层吸收变化。

| 层 | 状态 | 关键文件 |
| --- | --- | --- |
| 协议层（信封） | ✅ 已在 main | `vllm/v1/kv_hints/protocol.py` |
| Action 契约 + dispatch | ✅ 本分支新增 | `vllm/v1/kv_hints/{actions,dispatch}.py` |
| G1 状态机 | ✅ 本分支新增 | `vllm/v1/core/kv_cache_priority.py` |
| block_pool/scheduler 集成 | ✅ 本分支修改 | `vllm/v1/core/block_pool.py`、`v1/core/sched/scheduler.py` |
| OpenAI 暴露 | ✅ 本分支新增 | `vllm/v1/kv_hints/wire.py` + `entrypoints/openai/{chat_completion,completion}/` |
| 设备接缝（契约②） | ✅ 本分支新增 | `vllm/v1/kv_offload/transfer_backend.py` + `cpu/gpu_worker.py` |
| COPY_TO 原语 | ✅ 本分支新增 | `.../offloading/prefetch.py` + `vllm/v1/kv_hints/copy_to.py` |
| 文档 | ✅ 本分支新增 | `vllm/v1/kv_hints/README.md`（引擎侧契约权威文本） |

---

## 1. 契约①（A×C）：OpenAI 通道 wire schema

### 1.1 请求形态

Chat Completions 与 Completions 请求体新增可选字段 `kv_hints`（对象），JSON 形态与引擎信封一一对应：

```json
{
  "model": "...",
  "messages": [...],
  "kv_hints": {
    "protocol_version": "0.1",
    "message_id": "router/message-1",
    "actions": [
      {
        "action_id": "retain-prefix",
        "action_type": "vllm.set_priority",
        "action_version": "1.0",
        "payload": {
          "target": "request_cached",
          "location": "local_g1",
          "claim_id": "router/session-1",
          "revision": 1,
          "value": 10,
          "ttl_seconds": 600
        }
      },
      {
        "action_id": "prefetch-session-1",
        "action_type": "vllm.copy_to",
        "action_version": "1.0",
        "payload": {
          "target": ["<gpu-block-hash-hex>", "..."],
          "src": {"tier": "cpu"},
          "dst": {"tier": "gpu"}
        }
      }
    ]
  }
}
```

**⚠️ 待冻结标注**：字段命名按 #59714 实际契约（set_priority）与 #62382 披露的 RFC #59708 收敛方向（copy_to = `target/src/dst`，location = `{tier, engine_id?}`）定的 v0。契约①冻结后如有改名，只需改 `v1/kv_hints/actions.py` + `wire.py` 两处，C 侧 golden vectors 同步更新即可。

### 1.2 两层校验语义（S5/S8 的依据）

| 层 | 校验内容 | 失败行为 |
| --- | --- | --- |
| API 边界 | 信封形状（protocol_version/message_id/actions 必填、action 四字段、未知字段拒绝） | **4xx**，请求失败 |
| 引擎 dispatch | 非法/未知的 **action**（payload 语义、未注册类型、错误版本、非 `vllm.*` 前缀） | **跳过该 action**（`rejected` 时 WARN 日志；`unsupported` 静默跳过），请求与信封内其他 action 不受影响 |

C 侧规则：**畸形 hint 永不使请求失败**的前提是信封形状合法；形状错误会被 4xx。Motor 的降级重试（引擎 4xx → 摘除 kv_hints 重试一次）按 `KV_HINTS_MOTOR_GUIDE.md` 既有设计执行。

### 1.3 执行时机与结果词汇

| action_type | 执行时机 | abort/错误时 | preemption 时 |
| --- | --- | --- | --- |
| `vllm.set_priority` | 请求**成功完成**时（STOPPED/LENGTH_CAPPED），释放请求引用前 | 丢弃未应用的 | 保留到最终完成 |
| `vllm.copy_to` | 请求**准入（ingress）**时立即提交 | 不适用（无预留态挂请求） | 不适用 |

dispatch 结果状态（C 侧观测 `kvh=` 指标的取值）：`applied`（已执行/已提交）、`deferred`（当前不可行，可重试——copy_to 预留失败或源未就绪）、`duplicate`（同 revision 重放）、`rejected`（payload 冲突/目标缺失，WARN 日志）、`unsupported`（无消费者/功能未开/越界组合）。

### 1.4 set_priority 语义要点（S2/S3/S4/S6）

- 优先级是**软偏置**：值低的块先驱逐、LRU 破平局；不 pin 内存、不减容量、不改请求排序
- `target` 固定 `request_cached`：作用域 = 该请求在引擎 G1 池中的已缓存副本；未缓存尾部排除，空作用域 rejected——**Motor 不需要做块坐标换算**（v0.1 方案的 D-M3 在此路径上不适用）
- claim/revision 仲裁：`claim_id` 由 Motor 生成（按 session 命名空间）；revision 递增替换 value/TTL；同 revision 不同 payload 整单 rejected；同/旧 revision 不能续 TTL
- 清除声明：`value: null` + 省略 `ttl_seconds` + revision 递增（S3/S4 的 keeper piggyback 用这个形态）
- 每块最多 16 个 claim；过期由引擎惰性处理，过期/清除的 revision 记录保留到块复用/失效/reset（防止重放复活）

### 1.5 copy_to 语义要点（S7）

- `target` 是**引擎 GPU 块 hash 的 hex 列表**（每 GPU 块一个、按前缀顺序）——**必须用引擎 BlockStored KV 事件里发布的 hash，不能用 Conductor 自算的 XXH3 hash**（两者不同构，这是 R6 的具体落点，见 §4）
- 仅支持本地 `cpu → gpu`；`engine_id` 必须省略；其他组合报 `unsupported`
- 提交成功 ≠ 可读：块在传输完成后由引擎发布进前缀缓存，可读性通过正常 KV 事件对 Conductor 可见
- 引擎侧开关（引擎启动配置，非请求级）：
  ```json
  "kv_connector_extra_config": {"enable_request_free_prefetch": true, "prefetch_reserve_blocks": 0}
  ```
  前置条件：prefix caching 开启、单 KV cache group、block size == hash unit
- 不预留失败的提交：`DEFERRED` 时引擎零副作用，Motor 可对同目标重试

---

## 2. 契约②（A×B）：KVTransferBackend 设备接缝

### 2.1 接口（`vllm/v1/kv_offload/transfer_backend.py`）

```python
class KVTransferBackend(ABC):
    def select_copy_op(layer_refs_per_group, gpu_to_cpu,
                       host_memory_is_pinned=True) -> CopyOp: ...
    def descriptor_ptr_dtype() -> torch.dtype: ...
    def validate_device_tensor(tensor) -> None: ...
    def host_register(ptr, num_bytes) -> bool: ...
    def host_unregister(ptr) -> None: ...
```

| 方法 | 语义 | 错误语义 |
| --- | --- | --- |
| `select_copy_op` | 返回批拷贝可调用对象，统一签名 `(src, dst, sizes, is_src_access_order_any=...)`，在当前 stream 上下文异步执行 | 拷贝失败经 job completion 上抛 |
| `descriptor_ptr_dtype` | 描述符指针元素 dtype（CUDA=int64，XPU=uint64） | — |
| `validate_device_tensor` | 断言张量是本平台可寻址的设备张量 | assert 失败即拒绝构建 worker |
| `host_register` / `host_unregister` | host 内存 pin 注册（**best-effort 优化**） | **不得 raise**；返回 False / no-op 降级为 pageable 拷贝 |

**publication 规则**：submit ≠ ready，只有 `get_finished()` 报告完成（end event 过）后副本才可发布。stream/event **不在接口内**——已由 `current_platform` 属性委托到 `torch.<device_type>`（NPU 自动获得 `torch.npu.Stream/Event`），B 无需接入。

### 2.2 B 侧接入步骤（vllm-ascend）

1. 实现 `KVTransferBackend` 五方法（copy 内核可对接 AscendStore 既有 copy-thread/aclMemcpyAsync 路径；`host_register` 若 TorchNPU 无等价 API，返回 False 走 pageable 兜底，正确性不受影响——方案 R4 风险预案）
2. 插件初始化时注册：
   ```python
   from vllm.v1.kv_offload.transfer_backend import register_kv_transfer_backend
   register_kv_transfer_backend(current_platform.device_type, NPUTransferBackend())
   ```
3. 解析顺序：显式注册 > 内置 CUDA-alike/XPU 后端 > None。注册后 `cpu/spec.py` 的平台门禁自动放行，`SingleDirectionOffloadingHandler`/`CPUOffloadingWorker` 构造时自动解析到 NPU 后端（也可经 `backend=` 参数显式注入）
4. mock 单测范式参照 `tests/v1/kv_offload/cpu/test_transfer_backend.py`（`StubBackend`）——**B 可不等 A 的 NPU 实现、直接用该文件模式先行开发**（方案 §8.3 机制 2）

### 2.3 G1（set_priority）对 NPU 的预期

调度侧代码（`block_pool.py`/`kv_cache_priority.py`/`scheduler.py`）零设备依赖——**A-5 验证预期零改动**。若 NPU 上 `tests/v1/kv_hints/`、`tests/v1/core/` 有失败，即暴露插件与核心的耦合问题（这正是验证点）。

---

## 3. 契约③（B×C）：KV 事件闭环

- 引擎事件：`vllm/distributed/kv_events.py` 的 `BlockStored.block_hashes`（`ExternalBlockHash`，session_id 已回显，PR #51381 已在 main）
- **prefetch 发布也走同一事件流**：copy_to 的块发布进前缀缓存后自然产生 BlockStored——Conductor 无需新通道即可观测 prefetch 结果
- **hash 同构性警告（R6）**：vLLM 引擎内块 hash = sha256(token_ids) 派生；Conductor 的 XXH3 LocalBlockHash 是另一体系。**C 侧 copy_to 的 target 必须回填引擎事件里的 hash**（Conductor 消费事件时按 session 记录 hash 列表），不能拿自算 hash 直发。vllm-ascend MLA 场景的 effective block_size 换算沿用 vllm-ascend#16038 先例
- staleness 语义（S6）：引擎对 hash 选择器按提交时刻快照解析；已不在引擎侧的内容 → `MISSING` → `rejected`（WARN），不扩大执行范围；Motor 依事件更新索引

---

## 4. 联调场景映射（§7.2 → 本分支能力）

| 场景 | 能力就绪 | Motor 注入方式 | 引擎配置 |
| --- | --- | --- | --- |
| S1 基线无 hint | ✅ | 不发 kv_hints | 无要求 |
| S2 Retain 保活 | ✅ | `set_priority{request_cached, value>0, ttl}` | 无要求（APC 开启） |
| S3 TTL 到期释放 | ✅ | piggyback `set_priority{value:null, revision+1}` | 无要求 |
| S4 finalize 即释 | ✅ | piggyback 清除 + keeper 不二次动作 | 无要求 |
| S5 失败隔离 | ✅ | 混入未知 action/坏 payload | 无要求 |
| S6 staleness | ✅ | 发已失效 hash 集 | 无要求 |
| S7 Prefetch | ✅（真传输需设备） | `copy_to{hashes, cpu→gpu}` | `enable_request_free_prefetch=true` |
| S8 降级矩阵 | ✅ | 引擎不支持 → 4xx/unsupported → 摘除重试 | 旧版本引擎 |
| S9 压测 | 待联调 | aiperf | 按 S2+S7 配置 |

验收可观测口径：Motor `kvh=<n_actions,accepted>` 与引擎 WARN 日志/`vllm:kv_offload_prefetch_*` 指标对账；KV 事件驱动的 Conductor 索引与引擎驻留一致。

---

## 5. 已知限制与边界（诚实清单）

1. **入口只覆盖 chat/completions**：Anthropic API、Responses API、Rust gRPC 路由未接（gRPC 字段 17 传输层在 main 已有，但 completions 路由硬编码 None）；beam search 分支不携带 kv_hints
2. **copy_to 目标解析是全有全无**：第一个不可解析 chunk 即截断（MISSING→rejected 整单），不支持跨空洞多段加载；显式 remote targeting / move_to 未实现
3. **单 KV group + block size == hash unit** 才支持 copy_to（混合模型 UNSUPPORTED）
4. `n>1` 时每个子请求各自携带同一信封（claim 相同、块集不同 → 引擎侧去重，语义安全但 Motor 观测会看到多条 applied）
5. copy_to 预留目标块期间按普通分配计容量（不 pin）；被 reset 会 CANCELLED
6. OpenAI 层 `kv_hints` 为 dict 透传 + 严格校验，**未进 OpenAI API schema 文档**（上游入口收敛前刻意保持低调，避免第三套入口承诺）

---

## 6. 验证复现（无 GPU 环境）

```bash
# 环境（本机已验证）
uv venv --python 3.12 && source .venv/bin/activate
uv pip install "torch==2.14.0" torchvision --torch-backend=cpu
VLLM_TARGET_DEVICE=empty uv pip install -e . --no-build-isolation
uv pip install pytest pytest-xdist pytest-asyncio pytest-forked pytest-rerunfailures pytest-timeout tblib cloudpickle

# 运行（注意两个环境变量；torchvision 必装否则 MM 用例级联失败）
VLLM_TARGET_DEVICE=cpu OMP_NUM_THREADS=1 .venv/bin/python -m pytest \
  tests/v1/kv_hints/ tests/v1/kv_offload/ \
  tests/v1/kv_connector/unit/offloading_connector/ \
  tests/v1/core/test_prefix_caching.py tests/v1/core/test_scheduler.py \
  tests/v1/core/test_priority_scheduler_random.py tests/v1/test_request.py \
  -q -n 8
# 基线：1847 passed / 32 failed（test_gpu_worker 的 triton 既有缺失，非本分支引入）
```

NPU 联调（B）建议先跑：`tests/v1/kv_hints/` + `tests/v1/core/`（A-5 继承性验证）→ 契约② mock 单测 → 真传输 e2e（S7）。

---

## 7. 上游协同注意（D-V1，拆 PR 时用）

| 本分支内容 | 上游对应 | 拆 PR 建议 |
| --- | --- | --- |
| `actions.py`+`dispatch.py`+`kv_hints/` 单测 | #59714（结构同源） | PR1：纯新增 + 测试补强，作为 #59714 的协同评论素材 |
| `kv_cache_priority.py`+block_pool/scheduler 集成 | #59714 | PR2 |
| `wire.py`+OpenAI 暴露 | #38514 占位 `kv_hints` 通道 | PR3（等契约①/RFC 收敛表态） |
| `transfer_backend.py`+gpu_worker 重构 | 无对应 PR（新提案） | ✅ **已提 [#62792](https://github.com/vllm-project/vllm/pull/62792)**（2026-10-20）；set_priority/OpenAI/COPY_TO 三面按 D-V1 承诺暂缓，待 #59714 作者回应后定形态 |
| `prefetch.py`+offloading 集成 | #58198（结构同源） | PR5（与其收敛或等待其 rebase 后对齐） |
| `copy_to.py`+注册 | #62382 方向 | 随 PR1/PR5 或单独，视 RFC 进展 |

承诺：**不新开竞争入口**——`extra_body.kv_hints` 与 #38514 通道同形，收敛后只改 `wire.py` 适配层。

---

## 8. 待办与风险

- [ ] 契约①冻结后：核对 `actions.py` 字段命名 + C 侧 golden vectors
- [ ] D-V1：#59714 评审评论 + RFC #59708 意图声明（人工发帖，草稿备好）
- [ ] pre-commit 全套（ruff 已过；typos 等未跑）+ `pr-checklist` skill → 拆 3-5 个 PR
- [ ] R7 上游漂移：#59714/#38514/#58198 任一合入即触发适配层返工，周会跟踪
