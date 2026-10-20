# D-V1 上游评论草稿（发帖前人工审阅）

> 生成：2026-10-20；作者：同事 A（人工提交者）
> 规则（AGENTS.md）：由人工提交者逐行审阅后以本人身份发帖；两份草稿均为**技术贡献型**评论，不含未经验证的声明。
> 事实基线：`KV_HINTS_VLLM_STATUS.md`（2026-10-19 取证）+ 2026-10-20 复核（#59708 Open、#59714/#58198 未合入、#38514 已转 Draft）。

---

## 草稿一：PR #59714 评审 + 补强（发在该 PR 下）

**发帖前核查**：

- [ ] PR 仍是 Draft / 未被关闭或大幅重构
- [ ] 评论区没有维护者已提出相同意见（若已提出，改为 +1 并补充我们的测试证据）
- [ ] 最新 commit 未与本草稿描述冲突（rebase 后接口未变）

**性质**：技术评审 + 测试贡献要约。**不要**附内部文档链接；只描述可独立复现的技术内容。

---

Thank you for driving the execution surface for RFC #59708 — the structure here
(action contract + dispatcher + pool-integrated priority state machine) matches
how we believe this should be built, so we implemented it independently on a
local branch following this PR's structure to validate the contract end to end.
A few review points and an offer below.

### What we verified by independent implementation

1. **The SET_PRIORITY contract holds up in practice.** The payload
   (`target: request_cached`, `location: local_g1`, `claim_id`, `revision`,
   `value`, `ttl_seconds`) plus the invariant that clearing (`value: null`)
   must omit `ttl_seconds` while setting requires a finite positive TTL is
   self-consistent and covers the orchestrator flows we need (retain on
   success, clear on finalize/TTL expiry, non-renewable replays).
2. **Failure isolation behaves as documented.** Per-action isolation works:
   an unknown `action_type` or unregistered version reports `unsupported`,
   a malformed payload reports `rejected`, and a valid sibling action in the
   same envelope still applies. Non-`vllm.*` actions are left untouched for
   backend-owned consumers. One nit: `unsupported` outcomes are currently
   silent — a `logger.debug` (or counting them in the connector stats) would
   help orchestrators notice protocol drift.
3. **The pool integration preserves free-list/LRU invariants.** We wired the
   victim-selection callback into `get_new_blocks` and the declaration
   lifecycle into touch/free/unpin/evict/reset paths on current `main` (the
   block pool has since gained the metrics collector, `evict_blocks`, and
   reuse-watcher hooks). All of it composes without reducing capacity, and
   the lazy heap + compaction design keeps the hint-free path allocation-only.

### Offer: CPU-only contract tests

This PR currently has no tests (`E2E/perf: to be added`). We wrote a test
suite that runs entirely on CPU (`tests/v1/core/` needs no device) and would
gladly contribute it here, including:

- payload validation (msgspec constraints are enforced on the decode path —
  direct construction skips `Meta` bounds — so dispatch must go through
  `msgspec.convert`, which the dispatcher already does),
- dispatcher semantics: duplicate registration rejection, protocol-version
  mismatch, per-request deferred-action isolation, idempotent `finish()`,
- the full overlap matrix from the state machine: same/older revision cannot
  renew TTL, conflicting payload at the same revision rejects the whole
  scope, expiry during heap compaction still returns blocks to LRU, recycled
  block IDs do not resurrect claims (reuse/evict/reset), per-block claim
  limit (16) preserves existing declarations,
- scheduler-level tests: completion applies a valid hint despite an invalid
  sibling; aborted requests discard deferred actions; async scheduling.

If you'd like, we can rebase these onto this PR's branch directly.

**Rebase assistance**: the PR conflicts with current `main` in
`block_pool.py` and `scheduler.py`. We have already adapted the structure to
the latest base and can share the diff to speed that up.

**Perf**: we have no GPU numbers to offer yet (our validation environment is
CPU-only); the orchestrator-closed-loop E2E will run on NPU first. We'll post
numbers from our deployment validation when available, but would encourage
merging the contract + tests ahead of that.

---

## 草稿二：RFC #59708 意图声明（发在该 RFC 下）

**发帖前核查**：

- [ ] RFC 仍 Open 且无新的认领声明
- [ ] #38514 仍为 Draft（若已合入，需改写"入口对齐"段落）

**性质**：意图声明 + 契约收敛输入。按 AGENTS.md，这是 AI 协助起草、人工审阅并署名发布的。

---

We are building an orchestrator-driven agentic serving stack (orchestrator +
session-affine KV routing, deployed on both GPU and Ascend NPU) and intend to
implement the execution surface described in this RFC. Current status and
where we plan to contribute rather than compete:

### Implemented on a branch, following the existing PRs' structures

- **G1 SET_PRIORITY** following #59714's design (claim/revision overlap
  arbitration, TTL, priority-biased eviction with LRU tie-breaking, applied
  at successful completion before request references are released).
- **G2→G1 COPY_TO** following #58198's request-free prefetch primitive
  (reserve behind a configured free-block reserve, async transfer,
  publication into the prefix cache on completion, CANCELLED on reset).
  In our flows the orchestrator targets explicit block hashes captured at
  submission time - the hashes it consumes from `BlockStored` KV events.
- **OpenAI exposure** as `extra_body.kv_hints`, matching the channel shape
  #38514 introduced (single entry; we do not intend a third scheme), with a
  two-layer validation split we found necessary in practice: malformed
  *envelope* shape fails fast at the API boundary (4xx), while malformed
  *individual actions* are skipped with per-action isolation so one bad hint
  never fails an otherwise valid request.

### Differentiated evidence we can bring to this RFC

1. **Platform-neutral validation on NPU (vllm-ascend).** The scheduler-side
   surface (dispatch, priority state machine, block-pool integration) is
   device-independent; we will validate it unchanged on Ascend NPU, and we
   extracted the device-bound operations of the CPU offload worker into a
   registrable backend interface so an NPU copy path can be injected as a
   plugin rather than forking the offloading core. If the RFC wants an
   extensibility story for CASE-1 executors on non-CUDA platforms, we'd like
   to propose that seam.
2. **Orchestrator closed-loop evidence.** Our orchestrator drives retain /
   clear (TTL expiry, session finalize, piggybacked clears on the next
   request) and copy_to (pause→resume prefetch) purely from KV-event
   feedback, including staleness handling for hash-selector targets that no
   longer resolve engine-side (report partial/missing scope, never widen
   execution). We'll publish the scenario matrix and results.
3. **Contract convergence input.** Two points where independent
   implementations seem to be converging and where pinning the contract down
   early would avoid churn:
   - `copy_to` payload shape: `target` (content selector) + `src`/`dst`
     locations carrying `tier` (+ optional `engine_id`), with hash selectors
     capturing their scope at submission time.
   - the outcome vocabulary for executor-local results
     (`applied / deferred / duplicate / rejected / unsupported`) and its
     mapping to synchronous vs asynchronous (completion-event) feasibility.

Happy to fold our test suite (CPU-only, covers the overlap/expiry/isolation
matrix) into #59714, and to align our copy_to executor with #58198's
mechanism as it lands. We will not open competing PRs for these surfaces.

---

## 发帖操作说明（A 本人执行）

1. 两份草稿分开发：草稿一 → `https://github.com/vllm-project/vllm/pull/59714`（Review comment）；草稿二 → `https://github.com/vllm-project/vllm/issues/59708`
2. 发帖前跑一遍核查清单；若 #59714 评论区已有同类意见，草稿一压缩为"确认 + 补充测试要约"
3. 语气校准：草稿一是外部评审口吻；不要出现内部代号（Motor/Conductor/同事），统一用 "our orchestrator / our deployment"
4. 若维护者询问 perf/E2E 数据，引用 §7.3 验收指标框架并承诺 NPU 联调后补（不要承诺具体日期）
5. 发帖后把链接回填到本文件与 `KV_HINTS_TRIPLE_REPO_PLAN.md` §4.1 D-V1 状态
