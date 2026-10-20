# KV Hints 端到端测试设计（三层环境：vLLM-only / Motor 联调 / NPU）

> 版本：v1.0（2026-10-20）
> 被测代码：vLLM fork 分支 `kv-hints-g1`（契约①②③实现完成，见 `KV_HINTS_INTEGRATION_GUIDE.md`）
> 关联文档：`KV_HINTS_TRIPLE_REPO_PLAN.md` §7 联调方案、`KV_HINTS_INTEGRATION_GUIDE.md` §4 场景表、`KV_HINTS_MOTOR_GUIDE.md`
> 用途：**本文档是测试设计，不绑定执行环境**。L1 随 PR 进 CI（GPU），L2 在 Motor 联调环境执行（G1 门），L3 在 NPU 环境执行（G2/G3 门）。
> 结论记录：本机（无 GPU）只出设计不执行；终态 = Motor 联调 + NPU 大模型。

---

## 0. 三层环境矩阵

同一套断言语汇，三种执行环境，逐层放大集成度：

| 层 | 环境 | 模型 | 目的 | 对应集成门 | Owner |
| --- | --- | --- | --- | --- | --- |
| **L1** | 单机 vLLM serve（GPU；CPU 可降级但非必需） | facebook/opt-125m 或 Qwen2.5-0.5B（≤0.5B） | 引擎侧契约黑盒验证：信封解析、dispatch、set_priority 生效 | 随 PR（CI GPU job） | A |
| **L2** | Motor union 拓扑 + 单 vLLM 实例 | 1.5B-3B 级（Motor 实际使用模型） | 全链路：X-Agent-Hint 头 → 翻译桥 → 引擎 → KV 事件回流 | G1（M4） | C |
| **L3** | vllm-ascend NPU 部署（生产形态） | 7B+ / DSV4（MLA） | 真传输 + 平台继承性 + 性能基线 | G2/G3（M6/M7） | B |

**环境变量约定**（跨层统一）：引擎侧 `enable_request_free_prefetch=true`（S7 起）；Motor 侧 `MOTOR_KV_HINT_BRIDGE=1`（D-M4 开关）。

---

## 1. 统一观测体系（跨层同一断言语汇）

| 观测面 | 指标/信号 | 断言方式 |
| --- | --- | --- |
| 前缀命中 | `vllm:prefix_cache_hits_total` / `vllm:prefix_cache_queries_total`（token 粒度计数器） | scrape `/metrics`，取**差值**；命中 token 数 ≈ 请求 prompt 块对齐 token 数 |
| prefetch | `vllm:kv_offload_prefetch_accepted/deferred/blocks_published` | scrape 差值；published>0 ⇒ 副本可查 |
| 引擎拒绝 | dispatcher WARN 日志（`Rejected KV hint ...`）；`unsupported` 为静默跳过（无日志） | 日志 grep（辅助断言） |
| Motor 侧 | `kvh=<n_actions,accepted>` 调度日志字段、Prometheus 计数 | 与引擎侧结果对账（S9 验收要求一致） |
| 闭环视图 | Conductor 索引 vs 引擎实际驻留（BlockStored 事件驱动） | S9 对账脚本（C 侧已有链路） |

**hash 采集纪律（R6 落点）**：copy_to 的 `target` 一律取自引擎 `BlockStored` 事件（L1/L2 可由测试 harness 直接订阅 ZMQ 事件流采集；L2 正式形态经 Conductor），**禁止**使用 Conductor 自算 XXH3 hash 直接注入。

---

## 2. L1 · vLLM-only E2E（E1-E7）

### 2.1 服务端配置

```bash
vllm serve facebook/opt-125m \
  --max-model-len 256 \
  --num-gpu-blocks-override 64 \
  --enable-prefix-caching \
  --port 8000
# 64 块 × 16 token = 1024 token KV 容量：2-3 个百 token 请求即打满 → 确定性内存压力
# opt-125m 无内置 chat template：主用 completions 端点（纯文本 prompt 驱动 APC）
# 若用 Qwen2.5-0.5B-Instruct（自带 template），chat/completions 双端点均可
```

### 2.2 关键构造物

- **压力函数** `pressure(n)`：发送 n 个互不相同、长度≈48 token 的请求（每个占 3 块），将池打满触发 LRU 驱逐
- **探针** `hit_check(prompt)`：发送该 prompt，返回本次请求的 cache hit tokens（metrics 差值）
- **顺序单客户端**：全部用例串行执行，消除并发噪声（E2E 不测并发——竞态由 CPU 单测覆盖）

### 2.3 用例明细

| # | 场景 | 步骤 | 断言 | 对应 §7.2 |
| --- | --- | --- | --- | --- |
| E1 | 基线无 hint | 无 kv_hints 请求 ×2 | 200 + 输出非空；两次命中一致（APC 基线工作） | S1 |
| E2 | 信封 4xx 边界 | 3 个变体：缺 `message_id`；信封未知字段；action 缺 `action_type` | 全部 4xx | 契约① |
| E3 | 失败隔离 | 合法信封内混入：坏 payload action + 未知 `action_type` + 非 `vllm.*` action + 合法 set_priority | **200** 正常输出；日志含 1 条 `Rejected KV hint`；合法 action 仍生效（由 E4 式探针验证） | S5 |
| E4 | Retain 保活 | ① P1（96 token）+ `set_priority{value:10, ttl_seconds:600, claim_id:"t/s1", revision:1}` 完成请求 ② `pressure(10)` ③ `hit_check(P1)` + `hit_check(P2 对照)` | P1 命中 ≈ 80 token（96 按 16 块对齐）；P2 命中 = 0；两断言差值 ≥ 64 token | S2 |
| E5 | TTL 到期释放 | E4 后 sleep(TTL+2s)，`pressure(10)`，`hit_check(P1)` | P1 命中 = 0（恢复普通 LRU） | S3 |
| E6 | 主动清除 | 同 E4（新 claim），随后发 `set_priority{value:null, revision:2}` 请求，`pressure(10)`，`hit_check(P1)` | P1 命中 = 0 | S3/S4 |
| E7 | 降级面 | ① `copy_to{hashes, cpu→gpu}`（未配 offloading connector）② copy_to `target` 含非 hex | 两个请求均 **200** 正常输出（unsupported / rejected 跳过） | S8 |

**E4 是本层核心**：证明"hint 经 HTTP → ingress 解析 → 完成时应用 → 驱逐偏置生效"全链路真 serve 可观测。

### 2.4 Harness 骨架（可直接落地）

```python
# tests/entrypoints/openai/completion/test_kv_hints_serving.py
import json, time, urllib.request
import pytest
from tests.utils import RemoteOpenAIServer
from vllm.platforms import current_platform

MODEL = "facebook/opt-125m"
pytestmark = pytest.mark.skipif(
    not current_platform.is_cpu(), reason="L1 runs on a CPU-build host"
)  # GPU CI 上按需替换门控为 is_cuda_alike()

SERVER_ARGS = ["--max-model-len", "256", "--num-gpu-blocks-override", "64",
               "--enable-prefix-caching"]

@pytest.fixture(scope="module")
def server():
    with RemoteOpenAIServer(MODEL, SERVER_ARGS) as s:
        yield s

def _complete(server, prompt, kv_hints=None):
    body = {"model": MODEL, "prompt": prompt, "max_tokens": 1, "temperature": 0}
    if kv_hints is not None:
        body["kv_hints"] = kv_hints
    req = urllib.request.Request(
        server.url_for("completions"),
        data=json.dumps(body).encode(),
        headers={"Content-Type": "application/json"},
    )
    try:
        with urllib.request.urlopen(req) as r:
            return r.status, json.loads(r.read())
    except urllib.error.HTTPError as e:
        return e.code, e.read()

def _metrics(server):
    with urllib.request.urlopen(server.url_for("metrics")) as r:
        text = r.read().decode()
    hits = queries = 0
    for line in text.splitlines():
        if line.startswith("vllm:prefix_cache_hits_total"):
            hits += float(line.rsplit(" ", 1)[1])
        elif line.startswith("vllm:prefix_cache_queries_total"):
            queries += float(line.rsplit(" ", 1)[1])
    return hits, queries

def _hit_tokens(server, prompt, kv_hints=None):
    h0, _ = _metrics(server)
    _complete(server, prompt, kv_hints)
    h1, _ = _metrics(server)
    return h1 - h0

def _set_priority(value=10, claim_id="t/s1", revision=1, ttl=600):
    return {
        "protocol_version": "0.1", "message_id": f"m{revision}",
        "actions": [{"action_id": "a1", "action_type": "vllm.set_priority",
                     "action_version": "1.0",
                     "payload": {"target": "request_cached",
                                 "location": "local_g1",
                                 "claim_id": claim_id, "revision": revision,
                                 "value": value,
                                 "ttl_seconds": ttl if value is not None else None}}],
    }

def test_retain_survives_pressure(server):
    p1 = " ".join(f"retain{i}" for i in range(80))
    _complete(server, p1, kv_hints=_set_priority())
    # 打满 64 块池
    for i in range(10):
        _complete(server, "filler " * 40 + str(i))
    assert _hit_tokens(server, p1) >= 80          # 被保留
    p2 = "control " * 90
    _complete(server, p2)                          # 对照组先被驱逐一次
    for i in range(10, 20):
        _complete(server, "filler " * 40 + str(i))
    assert _hit_tokens(server, p2) < 16            # 未声明 → 已被驱逐
```

（E2/E3/E5/E6/E7 按同构骨架实现；完整 payload 见 `vllm/v1/kv_hints/README.md` 与 `KV_HINTS_INTEGRATION_GUIDE.md` §1。）

---

## 3. L2 · Motor 联调 E2E（M1-M5）

**拓扑**：Motor union（DSH/Pi/Hermes 任一 harness）+ 单 vLLM 实例（L1 同款小池配置）；Motor `MOTOR_KV_HINT_BRIDGE=1`。**Motor 不感知 L1 的 kv_hints 注入**——全部经 `X-Agent-Hint` 头进入翻译桥。

| # | 场景 | 步骤 | 断言 |
| --- | --- | --- | --- |
| M1 | 语义映射 | DSH 发 `cache_control{type:pin, ttl_s:600, range:message}` | 引擎收到（或 mock 引擎捕获）`set_priority{request_cached, local_g1, claim_id=桥生成, revision=1, ttl=600}`；字段一一对应 `KV_HINTS_INTEGRATION_GUIDE.md` §1.1 |
| M2 | keeper piggyback | M1 会话静置 > TTL → keeper 扫描 | 后续请求携带 `set_priority{value:null, revision:2}`；引擎侧声明清除（探针验证可驱逐） |
| M3 | finalize 即释 | 会话 finalize | 声明立即清除；keeper 无二次动作（S4） |
| M4 | 降级自动摘除 | 引擎回滚到无 kv_hints 版本（或 mock 返回 4xx） | Motor 自动摘除重试成功，业务无感；`kvh=` 计数含失败样本 |
| M5 | 多会话并发 | 会话 A 声明保留 + 会话 B 高流量 | A 命中率不降、B 正常 LRU；`kvh=` 与引擎 rejected/applied 对账一致 |

**归属**：M 系列用例脚本维护在 Motor 仓（C 按其 harness 惯例落地）；本设计只锁定输入/断言语汇。**S6 staleness** 在 L2 预演（Motor 发故意失效 hash 集 → 引擎 rejected → Motor 依事件更新索引）。

---

## 4. L3 · NPU 生产形态（N1-N4）

| # | 场景 | 内容 | 断言/指标 |
| --- | --- | --- | --- |
| N1 | A-5 继承性 | `tests/v1/kv_hints/` + `tests/v1/core/` 在 vllm-ascend NPU CI 跑 | 全绿（预期零改动；失败即插件-核心耦合问题） |
| N2 | S7 真传输 | pause→resume：Motor 发 copy_to（hash 来自事件/Conductor） | `prefetch_accepted=1`；`blocks_published>0`；resume 请求命中 ≈ 全量；失败/取消路径预留正确释放 |
| N3 | MLA block_size 对账 | DSV4 类模型 | Conductor 索引块数 = 引擎事件块数（effective block_size 换算，复用 vllm-ascend#16038 结论） |
| N4 | 性能基线（S9） | aiperf agent-hint benchmark | 吞吐/TTFT/TPOT 偏差 < 噪声带；声明会话命中率提升按工作负载定标（#38514 参考：+2~5p / TTFT −6~15%） |

**NPU 服务端配置**：vllm-ascend `rfc/kv-hints` 分支 + A 的 `kv-hints-g1` 源码组合；`enable_request_free_prefetch=true` 依赖 B 的 `KVTransferBackend` 注册（`KV_HINTS_INTEGRATION_GUIDE.md` §2）。

---

## 5. 文件布局与归属

| 文件 | 层 | 仓库 | Owner |
| --- | --- | --- | --- |
| `tests/entrypoints/openai/completion/test_kv_hints_serving.py` | L1 | vLLM（本分支，随 PR3/CI） | A |
| M 系列场景脚本 + golden vectors | L2 | Motor | C |
| N 系列任务 + NPU CI 接入 | L3 | vllm-ascend | B |
| 本设计文档 | — | vLLM fork `kv-hint` 分支 | A |

## 6. 执行顺序与集成门

```text
L1（PR 合入即跑）──▶ G1 门 M4 = L2 全绿 ──▶ G2 门 M6 = L2 + N2 ──▶ G3 门 M7 = L2 + N2-N4 + S9
```

任一层失败的处理纪律：L1 失败 = 引擎实现问题（A 修）；L2 失败先分类（桥/引擎/账本，按 `KV_HINTS_INTEGRATION_GUIDE.md` §1.2 两层校验语义定位）；L3 失败 = 平台适配问题（B 修）。

## 7. 风险

| # | 风险 | 缓解 |
| --- | --- | --- |
| 1 | opt-125m 无 chat template | L1 主用 completions；L2/L3 换带 template 的模型 |
| 2 | `--num-gpu-blocks-override` 在 L3 大模型上构造的压力失真 | L3 的 S7 用真实 pause→resume 负载，不依赖 override |
| 3 | TTL 断言时钟敏感 | TTL ≥ 3s + sleep 余量；CI 上 TTL 用例标记 retry |
| 4 | metrics 标签随引擎数变化 | helper 对 `vllm:prefix_cache_*` 按**求和**取值（见骨架） |
| 5 | Motor 桥与引擎版本错配 | M4 降级用例即回归项；对账失败按两层校验语义定位 |
