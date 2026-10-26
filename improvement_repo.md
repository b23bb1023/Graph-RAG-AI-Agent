# Repository improvement plan

This plan turns the 20 October 2026 audit into implementation work for `b23bb1023/Graph-RAG-AI-Agent`. The application baseline was rechecked against `main` at [`558a19af5398ce35e58134ad35fc56cc9c40809a`](https://github.com/b23bb1023/Graph-RAG-AI-Agent/commit/558a19af5398ce35e58134ad35fc56cc9c40809a). Its complete tree contains only `README.md` and `LICENSE`, matching the audited snapshot. No `AGENTS.md`, application source, manifests, tests, or workflows are present in that tree. No open pull requests were present during this check.

**Scope:** this change improves documentation and records proposed technical work. It does not implement or validate the described application. Evidence below refers to the pinned pre-change README, so future README edits do not invalidate the references.

## Audit disposition

| Audit finding | Current evidence | Documentation change | Technical work still required |
| --- | --- | --- | --- |
| GRA-01, High: published project cannot run as described | The tree has two blobs; [the old setup](https://github.com/b23bb1023/Graph-RAG-AI-Agent/blob/558a19af5398ce35e58134ad35fc56cc9c40809a/README.md#setup) requires absent paths/manifests | README now labels the publication an architecture showcase and removes unusable installation/startup instructions | Publish a reproducible baseline or an accessible versioned implementation before presenting it as runnable |
| GRA-02, Medium: blanket local-only privacy promise conflicts with integrations | [The old opening](https://github.com/b23bb1023/Graph-RAG-AI-Agent/blob/558a19af5398ce35e58134ad35fc56cc9c40809a/README.md#what-sets-this-apart) and [security section](https://github.com/b23bb1023/Graph-RAG-AI-Agent/blob/558a19af5398ce35e58134ad35fc56cc9c40809a/README.md#security-boundaries) promise local-only behavior while [specialist agents](https://github.com/b23bb1023/Graph-RAG-AI-Agent/blob/558a19af5398ce35e58134ad35fc56cc9c40809a/README.md#specialist-agents) use external APIs | README separates intended local inference/storage from external traffic and names integration requirements | Verify actual data flows, opt-in controls, credential handling, and network-denied mode after source is available |

The following are delivery and verification requirements. Except for the missing implementation and inconsistent documentation above, they are **not confirmed runtime defects**: their code cannot currently be inspected.

## Priority 1 — Deliver a reproducible baseline

### 1. Publish the implementation or versioned implementation reference

**Evidence:** the old [project structure](https://github.com/b23bb1023/Graph-RAG-AI-Agent/blob/558a19af5398ce35e58134ad35fc56cc9c40809a/README.md#project-structure) names `app/`, `prompts/`, `whatsapp_bridge/`, and `requirements.txt`, none of which exists in the pinned tree. The old setup also requires `.env.example`, which is absent.

**Work:**

- Publish a small, sanitized API/memory slice first: configuration, `/health`, a local query route, and a model adapter with an offline test double.
- Include the prompt files actually required at startup. Describe implemented features separately from deferred agents and integrations.
- Declare the supported Python version and direct dependencies in a manifest, with a reproducible resolution/lock strategy. If the bridge ships, include its Node manifest and lockfile too.
- Supply `.env.example` containing placeholders only. Add ignore rules for real environment files, OAuth tokens, databases, vectors, logs, and bridge sessions. Inspect the publication for private content and credential literals before committing it.
- If source cannot be published here, link a genuinely accessible release/commit and state its access requirements. Keep this repository labeled as a design/showcase until then.

**Acceptance:** a reviewer can start from a clean checkout or the exact linked release, install the declared dependencies, start the API using documented commands, reach `/health`, and run one synthetic local query. Every required path in the quick start exists. Setup does not require private user files, live Google/WhatsApp credentials, or a model download for the stubbed smoke test.

### 2. Add a small CI gate for the published slice

**Evidence:** no workflow or test files are present. The audit reported zero Actions runs at its snapshot; this check does not infer anything about external/private CI.

**Work:** add install/import checks and a deterministic API smoke test on the supported runtime. Use temporary data directories and a model stub. Keep live integrations and model downloads out of the default job. Add a documentation path/link check so future startup instructions cannot silently reference missing files.

**Acceptance:** the job runs from a clean checkout, reaches the health endpoint, executes one local query, and fails on a missing entry point, import failure, or missing configuration template. Synthetic fixtures leave no personal data behind.

## Priority 2 — Establish privacy and authorization boundaries

### 3. Implement and verify local-only mode

**Evidence:** [external tools and OAuth setup](https://github.com/b23bb1023/Graph-RAG-AI-Agent/blob/558a19af5398ce35e58134ad35fc56cc9c40809a/README.md#google-cloud-setup) contradict the previous blanket offline promise. The proposed LM Studio URL is configurable, so a local default alone cannot prove local inference.

**Work:**

- Implement an explicit local-only mode that disables Google tools, feeds, and the WhatsApp bridge before tool registration/dispatch.
- Define how a local model endpoint is validated and how any deliberate remote endpoint override changes the privacy contract. Treat model acquisition as a separate network-enabled setup step.
- Document a per-tool request inventory: recipient, transmitted fields, received fields, local persistence, retention/deletion, and user controls.
- Store OAuth/bridge credentials outside committed files with restrictive filesystem access. Document refresh, revocation, cleanup, and backup behavior; redact private data from logs and error messages.

**Acceptance:** network-denied tests allow required loopback model traffic and reject external traffic. Disabled integrations cannot issue requests even if retrieved text requests them. Mock-provider traces show the exact approved payload for each enabled tool. Tests verify credentials and synthetic private message text do not appear in logs.

### 4. Bind source isolation to authenticated identity

**Evidence:** the old [memory isolation section](https://github.com/b23bb1023/Graph-RAG-AI-Agent/blob/558a19af5398ce35e58134ad35fc56cc9c40809a/README.md#memory-isolation--scaling) claims complete separation by `source_id`. No code demonstrates that mapping or query filtering.

**Proposed code areas:** API authentication/configuration, `app/core/context.py`, `app/memory/sqlite_store.py`, vector storage, graph builder/traverser, task/deadline tools, and the bridge identity mapper. These are intended paths, not current files.

**Work:** derive allowed source identity from authenticated caller context. Apply it to all writes, reads, exact/semantic searches, joins, recursive traversals, summaries, entity linking, deletion, and backups. Enforce identity ownership in server code; do not treat an arbitrary client `source_id` as authorization. Reject unset/placeholder production tokens and compare configured credentials safely.

**Acceptance:** two synthetic sources with overlapping entity names and identical query text cannot retrieve or mutate each other's nodes, edges, embeddings, summaries, tasks, or deadlines. A caller cannot switch sources by editing the API body or bridge metadata. Missing/incorrect authentication is rejected, including when a tool or retrieval fallback is used.

### 5. Separate tool selection from permission to act

**Evidence:** the old [specialist-agent table](https://github.com/b23bb1023/Graph-RAG-AI-Agent/blob/558a19af5398ce35e58134ad35fc56cc9c40809a/README.md#specialist-agents) includes event creation and task mutation. The security section names scopes, but scopes alone do not establish application authorization or user approval.

**Work:** validate tool arguments against strict schemas, enforce tool allowlists per caller, and require trusted user authorization for mutations. Treat retrieved email/feed/chat content as data, never permission. Confirm the intended event payload before sending a Calendar write; make retries idempotent. Document and request the least provider permissions that the implemented feature set needs.

**Acceptance:** malicious synthetic emails/feeds cannot create events or mutate tasks. Read-only operations cannot invoke write routes. An authorized write creates only the approved event once despite timeout/retry simulation. Tests cover unknown tools, malformed arguments, expired tokens, disallowed chats, and provider errors without leaking credentials.

## Priority 3 — Make graph retrieval correct and measurable

### 6. Define traversal, storage, and budget invariants

**Evidence:** the old [graph design](https://github.com/b23bb1023/Graph-RAG-AI-Agent/blob/558a19af5398ce35e58134ad35fc56cc9c40809a/README.md#graph-dag) mixes parent lineage and cross-session entity edges; [context assembly](https://github.com/b23bb1023/Graph-RAG-AI-Agent/blob/558a19af5398ce35e58134ad35fc56cc9c40809a/README.md#context-assembly) claims priority-aware token budgeting without code or tests.

**Work:**

- Specify acyclic parent lineage separately from potentially cyclic entity edges. Use visited sets and explicit depth/node/time limits for graph traversal.
- Parameterize SQL, enforce foreign keys/ownership, and define transactional behavior when SQLite and vector updates partly fail.
- Specify all depth/strategy contracts, including depth zero, empty results, malformed directives, and unavailable models.
- Count tokens for the full serialized model request, including system messages, tools, and reserved output. Define deterministic ordering/truncation when high-priority material alone exceeds budget.
- Detect FTS5 availability and test its fallback. Document whether vector failure is persistent or in-memory and how this changes retrieval guarantees.

**Acceptance:** deterministic fixtures cover depths 0–3, every strategy, repeated entities, cycles, deleted nodes, empty stores, malformed model responses, and unavailable SQLite/vector features. Results stay inside the source boundary and traversal/token limits. Partial writes recover without dangling ownership or silent loss claims.

### 7. Bound failures and publish reproducible measurements

**Evidence:** the old README advertises fallback behavior, microsecond operations, concurrency safety, and [broad hardware deployment](https://github.com/b23bb1023/Graph-RAG-AI-Agent/blob/558a19af5398ce35e58134ad35fc56cc9c40809a/README.md#deployment-footprint), but provides no implementation or benchmark artifacts.

**Work:** define request/tool timeouts, cancellation, bounded retries and worker rounds. Preserve useful error messages without raw private context. Build a synthetic evaluation set for exact/semantic retrieval and entity linking. Measure latency, memory, retrieval quality, and model failures with dataset/model/runtime/hardware versions recorded. Test the actual supported hardware rather than assuming model portability.

**Acceptance:** timeout and malformed-output fixtures terminate within documented limits. CI verifies deterministic correctness; a separate reproducible benchmark records hardware, model settings, corpus size, methodology, and results. README performance and hardware claims link to those measurements and state their limits.

## Definition of a polished first release

- [ ] A versioned implementation is available and its feature status is explicit.
- [ ] The documented clean-checkout quick start passes in CI with synthetic data.
- [ ] Dependency/configuration files exist, with no real credentials or user data committed.
- [ ] Local-only mode and enabled integration data flows are tested and documented.
- [ ] Source ownership, tool authorization, mutation approval, traversal bounds, and token budgets have regression checks.
- [ ] Retention/deletion, credential revocation, error handling, and known limitations are documented.
- [ ] Supported platforms and measured results are reproducible from versioned artifacts.

This plan does not establish production readiness. It supplies concrete completion criteria for a reviewable first release.
