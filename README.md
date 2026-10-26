# Graph RAG AI Agent

Design documentation for a local-first, graph-augmented multi-agent assistant. The proposed system combines SQLite graph memory, ChromaDB semantic retrieval, and manager/worker language models served through LM Studio.

> **Project status: architecture showcase.** The published application snapshot contains `README.md` and `LICENSE`; it does not include executable source, dependency manifests, configuration examples, or tests. This documentation update adds an improvement plan. The architecture below describes intended behavior, which cannot yet be reproduced or independently verified from this repository.

See [improvement_repo.md](improvement_repo.md) for the delivery roadmap, implementation requirements, and acceptance checks.

## What you can use today

- Review the proposed memory model, retrieval strategy, and integration boundaries.
- Use the [MIT license](LICENSE) for the material published here.
- Plan a runnable baseline using the improvement checklist.

There is no supported installation or startup command for this checkout. The previously described `app/`, `prompts/`, `whatsapp_bridge/`, `requirements.txt`, and `.env.example` are absent from the application snapshot. No accessible, versioned implementation is linked here. A working service, trained model, or deployment is not supplied.

## Design goals

The design addresses two problems: preserving relationships across conversation history, and choosing a retrieval effort appropriate to each query. A greeting should require less historical context than a request to reconstruct a project across several sessions.

The intended features are:

- **Graph-grounded memory:** typed interaction nodes, parent lineage, entity links, and hierarchical summaries.
- **Progressive retrieval depth:** shallow lookups for simple queries and broader graph traversal for research requests.
- **Local inference and persistence:** a local model endpoint, SQLite, and local vector storage.
- **Optional external tools:** news feeds, Gmail, Google Calendar, and a WhatsApp interface, with explicit data-flow and authorization boundaries.

Latency, retrieval quality, concurrency behavior, hardware compatibility, and security properties have not been established by committed source or tests.

## Proposed architecture

| Component | Intended responsibility |
| --- | --- |
| FastAPI service | Receive requests through `/query` and expose `/health` |
| Orchestrator | Intake, retrieval-depth selection, context assembly, routing, worker dispatch, and persistence |
| Manager model | Route requests, choose retrieval depth, and compose final responses |
| Worker model | Select structured tools and interpret their results within bounded rounds |
| SQLite graph store | Persist nodes, edges, entities, tasks, deadlines, and summaries |
| ChromaDB | Retrieve semantically related interactions, summaries, graph nodes, and entities |
| Specialist agents | News retrieval, email reading, calendar operations, and local task operations |
| Optional WhatsApp bridge | Forward allowlisted conversations to the API and return replies |

The original design names Python, FastAPI, Pydantic, SQLite, ChromaDB, LM Studio, the OpenAI-compatible client protocol, and Node.js for the bridge. These are proposed components, not a validated dependency set. Publish and test manifests before recommending versions or platform requirements.

## Proposed graph memory

Each request would produce typed nodes with parent links:

| Depth | Node types | Intended content |
| --- | --- | --- |
| 0 | `intake` | User input |
| 1 | `route` | Routing decision |
| 2 | `plan`, `final` | Execution plan or response |
| 3 | `agent_start`, `agent_result` | Worker dispatch and results |

Parent lineage would use SQLite recursive queries. A separate `graph_edges` table would link related entities across sessions. The implementation must distinguish acyclic parent lineage from cross-session links, which may form cycles and require bounded traversal.

The proposed entity index covers people, email contacts, topics, and dates. Regex extraction offers a deterministic baseline; optional model extraction would need schema validation and fixtures. No extraction accuracy or timing result is available.

The design also proposes summaries grouped by request depth and four vector collections:

| Collection | Intended use |
| --- | --- |
| `interaction_memory` | Interaction summaries |
| `level_summaries` | Compressed views of request history |
| `graph_embeddings` | Individual graph nodes |
| `entity_index` | Fuzzy entity matching |

SQLite FTS5 is the intended exact-search path, with a fallback when the runtime lacks that capability. Both paths need tests; compatibility should be detected at runtime rather than assumed from an operating-system name.

## Proposed progressive depth escalation

A shallow pass would gather initial context, then a manager-produced `TraversalDirective` would choose the scope, depth, entity filters, and token budget. Invalid or unavailable model output would fall back to a bounded heuristic and then a conservative default.

| Depth | Example request | Intended retrieval |
| --- | --- | --- |
| 0 | A greeting | No historical retrieval |
| 1 | A simple lookup | Recent summaries and exact matches |
| 2 | Project status | Relevant lineage, summaries, and semantic matches |
| 3 | Cross-session research | Bounded graph windows, entity links, and multiple collections |

The proposed strategies are `Broad`, `Focused`, `Exact`, and `None`. Context assembly would prioritize structured tasks/deadlines, then summaries and graph history, then exact/semantic matches and entity information. Token limits must apply to the complete serialized model input, with explicit handling when high-priority content alone exceeds the budget.

## Privacy and external data flows

**Local-first does not mean that every feature is offline.** The design intends local model inference and persistence. Enabling an integration requires network traffic and can transmit content or metadata to another service. Even a local model endpoint must be checked: the configured URL determines where requests go.

The following table describes intended flows and requirements for a future implementation. It does not establish current runtime behavior.

| Feature | Intended data flow | Required control before enabling |
| --- | --- | --- |
| Local models | Prompts, retrieved context, and tool results to a loopback LM Studio endpoint | Verify the endpoint is local; document model acquisition separately; test with external network access denied |
| SQLite / ChromaDB | Conversation content, entities, summaries, tasks, and embeddings to local storage | Document paths, retention, deletion, backups, and filesystem access |
| News / RSS | Requests and connection metadata to configured feed hosts; returned content into the assistant | Disable feed tools in offline mode; do not send conversation history as a feed request |
| Gmail | OAuth authorization and API requests to Google; selected message content into local processing | Opt-in setup, minimum read scope, protected token storage, and redacted logs |
| Google Calendar | Event queries and, when approved, event details sent to Google | Separate read/write authorization; explicit approval for creation; document requested scopes |
| WhatsApp bridge | Message content and chat identifiers pass through WhatsApp infrastructure and the local bridge; replies travel back | Keep the bridge off in offline mode; verify chat allowlists and identity mapping |

The proposed OAuth scopes are `gmail.readonly` and `calendar.events`. A scope is a permission grant, not proof that application code limits operations appropriately. The future release must document exact scopes, credential locations, token refresh/revocation, and provider-facing content. The earlier `data/google_token.json` location is a design proposal, not a file supplied here.

**Offline-mode requirement:** external integrations must be disabled by default in an explicit local-only mode, with outbound requests blocked in tests. No implemented switch or configuration recipe is available in this repository yet.

## Security requirements to verify

The intended design includes token-authenticated API calls, per-source memory boundaries, structured output validation, bounded workers, and redacted logs. These remain requirements until source and tests are published.

A release should demonstrate that:

- Caller identity determines permitted `source_id` values; a client-provided identifier alone cannot grant access to another source.
- Every graph, vector, entity, summary, task, and deadline query enforces that boundary.
- Retrieved emails, feeds, and messages are untrusted input and cannot authorize calendar or task mutations.
- Tool permissions, schemas, timeouts, and iteration limits are enforced in code.
- Credentials, raw private content, and model traces are excluded from logs and version control.
- Placeholder tokens such as `change-me-now` cannot be used as deployment credentials.

## Delivery roadmap

1. Publish a sanitized, minimal runnable implementation with manifests and safe example configuration, or link an accessible implementation pinned to a release/commit.
2. Add a clean-checkout health check and one deterministic local query using synthetic data and a model stub.
3. Test source isolation, graph traversal, budgets, failure handling, and external-tool authorization.
4. Document integration setup only after the configuration, scopes, and runtime controls can be checked against source.
5. Add measured retrieval/latency results and a tested hardware matrix before making performance or deployment claims.

The detailed tasks and completion criteria are in [improvement_repo.md](improvement_repo.md). Contributions should keep implemented behavior, proposed behavior, and measured results clearly identified.

## License

Published material is available under the [MIT license](LICENSE).
