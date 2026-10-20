# vLLM KV Hint 协议接入指南（Motor 视角）

> 调研基线：vllm-project/vllm `main` @ `642c18d64c`（2026-10-16 首次调研，2026-10-19 复核上游动态）
> 面向读者：Motor（编排器/orchestrator）侧开发者，需要通过 KV Hint 协议驱动 vLLM 执行 KV 缓存管理动作。
> 配套文档：`KV_HINTS_VLLM_STATUS.md`（上游 RFC/PR 竞合格局的完整梳理，随上游动态更新）

---

## 1. 协议背景与 RFC 链路

| 阶段 | Issue / PR | 内容 | 状态 |
| --- | --- | --- | --- |
| 总纲 | [#51428](https://github.com/vllm-project/vllm/issues/51428) | Programmatic Session-Aware KV Cache Management（编排器驱动 KV 管理的总体模型，定义 SessionPrefixIndexer 等概念） | RFC 开放 |
| ① 传输层 | [#53421](https://github.com/vllm-project/vllm/issues/53421) / PR [#53423](https://github.com/vllm-project/vllm/pull/53423) | typed + versioned 的 `KvHintsEnvelope` / `KvHintAction` 信封，随请求全链路传播 | **已合入**（commit `986e217158`） |
| ② 执行层 | [#59708](https://github.com/vllm-project/vllm/issues/59708) / PR [#59714](https://github.com/vllm-project/vllm/pull/59714) | 动作原语 `COPY_TO` / `MOVE_TO` / `SET_PRIORITY` + dispatch 路由 + G1 优先级执行器 | **未实现**（RFC 开放且无人认领；已有三个竞争实现均未合入，见 §4） |
| 相关 | [#57103](https://github.com/vllm-project/vllm/issues/57103)（Mooncake 可编程 KV RFC，其实现 [#62382](https://github.com/vllm-project/vllm/pull/62382) 为 Draft）、[#52113](https://github.com/vllm-project/vllm/issues/52113)（agent_hint RFC，**竞争性 OpenAI 入口**）、[#38514](https://github.com/vllm-project/vllm/pull/38514)（kv.retain 实现，自带 OpenAI `kv_hints` 通道）、[#53624](https://github.com/vllm-project/vllm/pull/53624)（KVCR 二级缓存适配，已合入） | 生态/其他消费路径 | — |

**一句话现状：信封已建好并随请求全链路流转，但 vLLM 自身还不会"解释"其中任何动作；当前唯一会消费 payload 的是 KVCR 后端（透传模式）。**

---

## 2. 当前实现状态（截至调研基线）

### 2.1 已落地：传输层 ✅

**Python 类型定义**（`vllm/v1/kv_hints/protocol.py`）：

```python
class KvHintAction(msgspec.Struct, frozen=True):
    action_id: str
    action_type: str
    action_version: str
    payload: dict[str, Any]   # 必须是 JSON 兼容值；由动作实现方校验

class KvHintsEnvelope(msgspec.Struct, frozen=True):
    protocol_version: str
    message_id: str
    actions: list[KvHintAction]
```

**Python 传播链**（完整打通）：

```text
AsyncLLM.generate(kv_hints=...)              # vllm/v1/engine/async_llm.py:414
  → InputProcessor.process_inputs(kv_hints=) # vllm/v1/engine/input_processor.py:387,554
  → EngineCoreRequest.kv_hints               # vllm/engine/protocol.py:110
  → Request.kv_hints                         # vllm/v1/request.py:198
```

同步引擎入口 `LLMEngine`（`vllm/v1/engine/llm_engine.py:241`）同样支持。

**Rust / gRPC 传播链**（完整打通）：

- proto 定义（`rust/proto/inference.proto`）：

```protobuf
message KVHintsEnvelope {           // GenerateRequest.kv_hints = 字段 17
  string protocol_version = 1;
  string message_id = 2;
  repeated KVHintAction actions = 3;
}
message KVHintAction {
  string action_id = 1;
  string action_type = 2;
  string action_version = 3;
  google.protobuf.Struct payload = 4;   // JSON 对象
}
```

- Rust 侧类型：`rust/src/engine-core-client/src/protocol/kv_hints.rs`；gRPC 转换：`rust/src/server/src/grpc/convert.rs`（`kv_hints_from_proto`）。

### 2.2 已落地的消费者（仅 2 个，且均为"透传/携带"，不解释语义）

| 消费者 | 代码位置 | 行为 |
| --- | --- | --- |
| OffloadingConnector（调度侧） | `vllm/distributed/kv_transfer/kv_connector/v1/offloading/scheduler.py:559` | 将 `req.kv_hints` 原样放入 `ReqContext`（`vllm/v1/kv_offload/base.py:95`），不解析、不执行 |
| KVCR tiering manager | `vllm/v1/kv_offload/tiering/kvcr/manager.py:605` | `on_new_request` 时 `msgspec.to_builtins(hint)` 后调用 `kvcr.submit_hint(request_id, hints)`；`on_request_finished` 时 `discard_hint`。payload 语义完全由 KVCR 后端定义（对应 RFC #59708 的 CASE 2：opaque envelope） |

### 2.3 未实现：执行层 ❌（Motor 侧不应依赖）

RFC #59708 提议的以下能力在本仓库**均不存在**：

- ❌ `vllm/v1/kv_hints/actions.py` — typed action 契约（`copy_to` / `move_to` / `set_priority`）
- ❌ `vllm/v1/kv_hints/dispatch.py` — consumer 注册与路由
- ❌ `vllm/v1/core/kv_cache_priority.py` — G1 优先级状态（声明/过期/重叠仲裁/牺牲块排序）
- ❌ Scheduler / `kv_cache_manager` / `block_pool` 对 kv_hints 的任何感知
- ❌ G2→G1 prefetch 执行路径（PR [#58198](https://github.com/vllm-project/vllm/pull/58198) 方向，本身仍为 Draft）与 hint 的打通
- ❌ 执行结果反馈给编排器的回传管道（"submitted ≠ ready"）

### 2.4 入口限制

| 入口 | kv_hints 支持 |
| --- | --- |
| Python `AsyncLLM.generate()` / `LLMEngine.generate()` | ✅ 唯一官方 Python 入口 |
| Rust gRPC `GenerateRequest.kv_hints`（字段 17） | ✅ |
| Python OpenAI HTTP API（`vllm/entrypoints/openai/`） | ❌ 完全未暴露（竞品 PR [#38514](https://github.com/vllm-project/vllm/pull/38514) 已实现 `kv_hints` 通道、RFC [#52113](https://github.com/vllm-project/vllm/issues/52113) 提出 `agent_hint` 通道，上游收敛前勿押注单一形态） |
| Rust OpenAI HTTP completions 路由（`rust/src/server/src/routes/openai/completions/convert.rs`） | ❌ 硬编码 `kv_hints: None` |
| `SamplingParams.extra_args` / `kv_transfer_params` | ❌（RFC 明确反对继续用此非类型化通道） |

---

## 3. Motor 接入方式

### 3.1 协议约定（RFC #53421 定义的 durability 规则）

1. `protocol_version`（信封级）与 `action_version`（动作级）**独立演进**。
2. schema 破坏性变更才 bump 对应版本；新增可选字段、新增 action_type 无需 bump 信封版本。
3. **每条 V1 请求要么带 `KvHintsEnvelope` 要么带 `None`**；无 hint 的请求行为不变。
4. 消费方对不支持的 `action_version` / 非法 payload 应**跳过该 action，不影响信封内其他 action**（失败隔离）；原始入站适配器应拒绝不支持的信封版本。
5. payload 对传输层完全透明，语义由实现该 action_type 的组件定义。

> ⚠️ 当前实现现状：Python 入站路径**尚未做** `protocol_version` 校验（RFC 第 4 条的"入站拒绝"尚未落地）。Motor 侧应自律使用当前约定版本（RFC 示例为 `"0.1"`），并预期未来 vLLM 收紧校验。

### 3.2 Python 侧发送示例

```python
from vllm.v1.kv_hints import KvHintAction, KvHintsEnvelope
from vllm import SamplingParams

envelope = KvHintsEnvelope(
    protocol_version="0.1",
    message_id="msg-123",
    actions=[
        KvHintAction(
            action_id="a1",
            action_type="<vendor>/<action>",   # 自定义命名空间，见 §3.4
            action_version="1.0",
            payload={"key": "value"},          # JSON 兼容
        )
    ],
)

collector = await llm.generate(
    prompts,
    SamplingParams(),
    kv_hints=envelope,
)
```

同步场景同理：`LLMEngine.generate(..., kv_hints=envelope)`。

### 3.3 gRPC 侧发送示例

通过 Rust 服务器 gRPC 接口，`GenerateRequest` 上设置字段 `kv_hints`：

```jsonc
{
  "kv_hints": {
    "protocol_version": "0.1",
    "message_id": "msg-123",
    "actions": [
      {
        "action_id": "a1",
        "action_type": "kvcr.example",
        "action_version": "1.0",
        "payload": { "block_hashes": ["..."] }
      }
    ]
  }
}
```

`payload` 使用 `google.protobuf.Struct`（即任意 JSON 对象）。

### 3.4 action_type 命名空间约定（Motor 必读）

- RFC #59708 预留了 vLLM 原生命名空间：`vllm.copy_to`、`vllm.move_to`、`vllm.set_priority`（`action_version = "1.0"`）。
- **当前发送这三个 action_type 不会有任何效果**——没有注册的消费者，会被静默忽略（不会报错，也不会影响请求本身）。
- Motor 当前应只使用**自家/后端命名空间**（如 KVCR 定义的 action type），前提是 vLLM 部署中配置了对应的后端连接器（当前即 KVCR tiering 适配器）。
- 后端未配置时，hint 全程无人消费（只在 request 对象上挂着，请求结束即丢弃），**无副作用但也无效果**。

### 3.5 生命周期语义

- Hint 是**请求元数据**，不是独立控制消息：**必须附着在某条推理请求上发送**（"piggyback"模式）。当前不存在异步/无请求 hint 通道（RFC #59708 开放问题之一）。
- KVCR 消费生命周期：新请求进入时 `submit_hint`，请求完成时 `discard_hint`。即 hint 的有效期与该请求在引擎内的生命周期绑定。
- Motor 若要在"请求间隙"执行动作（如工具调用暂停期 Retain 前缀），目前只能把 hint 附着在下一个可用请求上提前发出。

---

## 4. 未来执行层预告（RFC #59708，Motor 可提前对齐）

vLLM 计划提供的三个原语（wire 名已在 RFC 定型；但**契约尚未冻结**——三个竞争实现并存：[#59714](https://github.com/vllm-project/vllm/pull/59714)（RFC companion，Draft）、[#38514](https://github.com/vllm-project/vllm/pull/38514)（kv.retain，活跃，作者承诺对齐 #59714 契约）、[#62382](https://github.com/vllm-project/vllm/pull/62382)（Mooncake 全家桶，Draft），payload 字段细节以最终合入版本为准）：

| 原语 | wire 名 | 语义 | 典型用途 |
| --- | --- | --- | --- |
| COPY_TO | `vllm.copy_to` | 将 `src` 中目标块复制到 `dst`（保留源） | G2→G1 prefetch、worker 间 P2P 复制 |
| MOVE_TO | `vllm.move_to` | 将 `src` 中目标块移动到 `dst` | 降级（demote）腾出 HBM |
| SET_PRIORITY | `vllm.set_priority` | 对选定副本设置/清除驱逐优先级声明（可带 TTL；`value=null` 清除声明） | G1 Retain（工具暂停期保会话前缀） |

设计要点（Motor 对齐用）：

- 每个支持的 `(action_type, action_version)` 有且仅有一个注册消费者，冲突注册被拒绝。
- 数字优先级只影响**驱逐顺序**，不保证驻留（"biases eviction; does not guarantee residency"）。
- 缺失目标、部分作用域必须显式报告，不得擅自扩大执行范围。
- 提交 ≠ 就绪：COPY_TO 完成后副本才会注册为可查；Motor 需要依赖未来的事件/结果回传确认。
- 预定代码落点：`vllm/v1/kv_hints/{actions,dispatch}.py`、`vllm/v1/core/kv_cache_priority.py`、`block_pool.py` 牺牲块集成点、offloading connector 的 COPY_TO handler。
- SET_PRIORITY 的实际 payload 比 wire 签名更细：#59714 实现为 `{target, location, claim ID, revision, priority, TTL}`——claim ID + revision 用于**重叠声明的仲裁**（同前缀多方声明），Motor 侧声明账本需预留这两个维度。

---

## 5. Motor 侧最佳实践建议

1. **版本自律**：当前 vLLM 入站不校验版本，Motor 应固定发送 `protocol_version="0.1"` 并内建版本协商逻辑，避免未来 vLLM 收紧后批量失败。
2. **幂等与追踪**：`message_id` 全局唯一、`action_id` 信封内唯一，便于失败隔离与对账（RFC 的 per-action failure isolation 依赖此约定）。
3. **不依赖隐式成功**：当前无执行结果回传。Motor 需要通过后端自身的 API（如 KVCR 的状态接口）或观测 KV 命中率来间接确认效果。
4. **失败隔离假设**：单条 action 非法不应拖垮整条请求；但不要依赖"非法 action 会返回错误"——当前是被静默忽略。
5. **环境前置检查**：部署侧需确认 vLLM 启用了对应 KV 传输连接器（如 OffloadingConnector + KVCR tiering），否则 hint 无消费者。
6. **勿用旧通道**：不要再往 `SamplingParams.extra_args["kv_transfer_params"]["kv_hint"]` 塞 hint（Dynamo 早期原型的做法），RFC 已明确该路径将被 typed envelope 取代。
7. **关注 SessionPrefixIndexer**：编排器对 block 驻留的视图可能过期（RFC #59708 开放问题 #3）。Motor 发出的块地址类 hint（未来的 copy/move）应做好目标不存在/部分命中的容错。

---

## 6. 测试与验证参考

- `tests/v1/test_request.py:47` — 信封从 EngineCoreRequest 复制到 Request
- `tests/v1/kv_offload/tiering/test_kvcr_tier.py:353` — KVCR `submit_hint` / `discard_hint` 生命周期
- `tests/v1/kv_connector/unit/offloading_connector/test_scheduler.py` — ReqContext 携带 hint
- Rust 侧：`rust/src/engine-core-client/src/tests/python_compat.py`（Python/Rust 类型兼容）、`rust/src/server/src/grpc/convert.rs` 的 `kv_hints_propagate_from_top_level_request_field`

---

## 7. 速查表

**Motor 今天能做的：**

- ✅ 通过 Python `AsyncLLM.generate(kv_hints=...)` 或 Rust gRPC 字段 17 发送 typed envelope
- ✅ 让配置了 KVCR 的 vLLM 部署消费自定义 action（opaque 透传）
- ✅ 依赖 per-request 生命周期自动管理 hint（submit/discard）

**Motor 今天不能做的：**

- ❌ 触发 vLLM 原生 KV 动作（copy/move/set_priority 均未实现）
- ❌ 通过 OpenAI HTTP API 携带 hint
- ❌ 获取执行结果反馈/回执
- ❌ 发送不附着于请求的异步 hint
- ❌ 期望非法 action 得到显式报错

**盯梢清单（上游动态，完整梳理见 `KV_HINTS_VLLM_STATUS.md`）：**

- RFC #59708 契约冻结与落地进度（actions.py / dispatch.py / kv_cache_priority.py）
- 执行层三方竞合走向：PR #59714（G1 Retain，Draft + needs-rebase）、#38514（kv.retain，活跃）、#62382（Mooncake 路线，Draft）
- OpenAI 入口收敛：#38514 的 `kv_hints` 通道 vs #52113 的 `agent_hint` 通道（NVIDIA 已建议二者收敛为一个 lifecycle contract）
- PR #58198（G2→G1 prefetch 原语，Draft）、#57103（Mooncake RFC）
