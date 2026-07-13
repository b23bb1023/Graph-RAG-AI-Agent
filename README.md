# Local Agent System — Graph-Augmented Multi-Agent Orchestrator

A fully local, security-first AI agent system with a custom **Graph RAG** memory architecture and **Progressive Depth Escalation** retrieval pipeline — built entirely from scratch on SQLite, ChromaDB, and local LLMs via LM Studio.

## What Sets This Apart

- **Custom Graph RAG** — No off-the-shelf RAG framework. Graph traversal, entity extraction, cross-session edge linking, hierarchical summarization, and token-budget-aware context assembly are all purpose-built for this multi-agent system.
- **Progressive Depth Escalation** — Not all queries pay the same retrieval cost. Greetings use a shallow SQLite lookup (depth=1), while deep research questions trigger a full BFS graph walk across sessions, entity edges, summaries, and structured data (depth=3).
- **SQLite as a graph database** — Recursive CTE lineage traversal, entity-based edge linking, and FTS5 full-text search — no Neo4j or specialized graph DB required.
- **Entity extraction without embeddings** — Deterministic regex-based entity extraction (person, email, topic, date, URL) works on first boot — no embedding model download needed.
- **Hierarchical level summarization** — After each request, sibling nodes at each depth are compressed by the worker LLM into summary paragraphs, creating a "memory pyramid" for efficient historical recall.
- **100% local** — All LLM inference, vector search, graph storage, and entity indexing run on-device. No data leaves the machine.

---

## Problem Statement

Flat vector stores and naive keyword search suffer from **memory amnesia** — every query is effectively isolated, with no graph-grounded understanding of past intents, entity relationships, or procedural histories. Traditional multi-agent systems compound this with **blind cost-equality** — every query, from "hello" to "find everything about Project X across all sessions," pays the same retrieval cost.

This system solves both problems with **query-aware depth reasoning** as a first-class architectural primitive.

---

## Architecture Overview

```
                          POST /query
                              |
                      ┌───────▼───────┐
                      │   FastAPI      │
                      │   (main.py)    │
                      └───────┬───────┘
                              |
                   ┌──────────▼──────────┐
                   │    Orchestrator      │
                   │  (orchestrator.py)   │
                   │                      │
                   │  1. Intake           │
                   │  2. Depth Reasoning  │◄── Progressive Depth Escalation
                   │  3. Context Building │
                   │  4. Routing          │
                   │  5. Plan Dispatch    │
                   │  6. Post-processing  │
                   └──┬───────┬──────┬───┘
                      │       │      │
              ┌───────▼┐ ┌───▼──┐ ┌▼──────────┐
              │Manager │ │Worker│ │4 Specialist│
              │  LLM   │ │ LLM  │ │  Agents    │
              │(9B MLX)│ │(2B MLX)│ │            │
              └───────┬┘ └──────┘ │news│mail   │
                      │            │cal │task   │
                      │            └────┬───────┘
                      │                 │
              ┌───────▼─────────────────▼──────┐
              │         Memory Subsystem        │
              │                                 │
              │  ┌────────────┐ ┌────────────┐ │
              │  │   SQLite   │ │  ChromaDB   │ │
              │  │  (13 tbls) │ │(4 collctns) │ │
              │  └────────────┘ └────────────┘ │
              │  ┌────────────┐ ┌────────────┐ │
              │  │Graph DAG   │ │Level        │ │
              │  │+ FTS5      │ │Summarizer   │ │
              │  └────────────┘ └────────────┘ │
              └────────────────────────────────┘
```

---

## Memory System: Graph RAG (From Scratch)

### Graph DAG

Every interaction is stored as a **typed node** in a directed acyclic graph:

| Depth | Node Type   | Purpose                           |
|-------|-------------|-----------------------------------|
| 0     | `intake`    | Raw user input                    |
| 1     | `route`     | Manager routing decision          |
| 2     | `plan`      | Structured execution plan         |
| 2     | `final`     | Final response (chat/refuse/plan) |
| 3     | `agent_start` | Worker dispatch metadata        |
| 3     | `agent_result` | Worker execution trace          |

Nodes are linked via `parent_id` foreign keys, forming a natural DAG traversable ancestrally (what led here?) and descendentally (what resulted from this?) using recursive CTEs.

### Entity Extraction

A deterministic regex engine extracts 4 entity types from every interaction:

| Entity Type    | Example Match                |
|----------------|------------------------------|
| `person`       | "Alice said she would..."    |
| `email_contact`| `alice@example.com`          |
| `topic`        | "about **Project Alpha**"    |
| `date`         | "due on **15 Jan 2025**"     |

This runs in microseconds per node — no embedding model, no LLM call. A secondary `extract_entities_with_llm()` path is available for richer extraction when desired.

### Cross-Session Edge Linking

After each node is created, `GraphBuilder._link_cross_session()` searches the entity index for past nodes sharing the same entities. Matching pairs are connected via the `graph_edges` table with `edge_type='entity_overlap'` and a weight proportional to overlap count. This enables graph traversal **across sessions** — follow entity-linked edges to discover related conversations spanning days or weeks.

### Hierarchical Level Summarization (Vectorless RAG)

After every request completes, the `LevelSummarizer`:

1. Groups all graph nodes for the request by depth
2. Calls the worker LLM to compress each level into a paragraph
3. Stores summaries in both **SQLite** (exact, queryable by depth/branch) and **ChromaDB** (semantic search via `level_summaries` collection)

This creates a "memory pyramid" — the manager can read increasingly compressed views of past sessions without token-flooding on raw node dumps.

### Multi-Collection Vector Store

Four ChromaDB collections serve distinct retrieval purposes:

| Collection              | Purpose                                      |
|-------------------------|----------------------------------------------|
| `interaction_memory`    | User-LLM interaction summaries               |
| `level_summaries`       | Hierarchical session summaries               |
| `graph_embeddings`      | Individual graph node embeddings             |
| `entity_index`          | Entity name embeddings for fuzzy matching    |

### FTS5 with Graceful Degradation

Full-text search uses SQLite FTS5 with porter tokenizer and auto-sync triggers when available, falling back to LIKE-based substring search on systems where FTS5 is not compiled in (e.g., Apple's system SQLite). The detection function `_check_fts5()` auto-adapts at startup.

---

## Progressive Depth Escalation

The core innovation — query-aware retrieval that decides **per-query** how much historical context to retrieve.

### Pipeline

```
User Query
    │
    ▼
┌─────────────────────┐
│ 1. Shallow Pass      │   depth=1, focused strategy
│    (Depth Reasoner)  │   ~microseconds, SQLite lookup
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│ 2. Manager Reasons   │   LLM call → TraversalDirective
│    about Depth       │   depth: 0|1|2|3
│                      │   scope: broad|focused|exact|none
│                      │   branch_filter, entities, budget
└─────────┬───────────┘
          │
    ┌─────┴──────┐
    │            │
 depth≤1       depth>1
    │            │
    ▼            ▼
  Return      ┌─────────────────────────┐
  Initial     │ 3. Deep Enrichment       │
  Context     │    Full BFS graph walk   │
              │    Multi-collection      │
              │    semantic search       │
              │    Level summaries       │
              │    Cross-session edges   │
              │    Structured data       │
              └─────────────────────────┘
```

### Depth Levels

| Depth | Trigger                                 | Retrieval Scope                              |
|-------|-----------------------------------------|----------------------------------------------|
| 0     | Greetings, trivial queries              | No history retrieval                         |
| 1     | Simple lookups, "what's today's news"   | Recent node summaries + exact keyword matches|
| 2     | "Progress on Project X," status updates | Ancestor lineage, level summaries, semantic  |
| 3     | "Everything about ACME since last month"| Full graph windows, all collections, entities|

### Three-Tier Depth Reasoning

1. **LLM call** → Manager model (qwen3.5-9b-mlx) analyzes the shallow pass results and outputs a structured `TraversalDirective` with exact depth, scope, branch filter, entity types, time weight, and token budget
2. **Keyword heuristic** → If the LLM call fails or times out, 50+ keyword patterns estimate depth from the query text
3. **Safe default** → Falls back to `depth=1, focused scope` — the system never hard-fails on retrieval

### Retrieval Strategies (Strategy Pattern)

| Strategy  | Scope         | Use Case                                         |
|-----------|---------------|--------------------------------------------------|
| `Broad`   | Full graph, all summaries, cross-session | Research, exploration, comprehensive history    |
| `Focused` | Branch-filtered, targeted summaries    | Daily updates, specific topic lookup             |
| `Exact`   | FTS5 only, structured data             | Precise deadline/task queries                    |
| `None`    | Skip all retrieval                     | Trivial queries, greetings                       |

### Context Assembly

The `ContextAssembler` builds a priority-ordered `ContextWindow` with a configurable token budget:

| Priority | Section                  | Source                    |
|----------|--------------------------|---------------------------|
| 0 (top)  | Structured data          | Deadlines, tasks, calendar|
| 1        | Level summaries          | Hierarchical summaries    |
| 1        | Graph history            | Graph node windows        |
| 2        | Exact memory matches     | FTS5 / LIKE search        |
| 2        | Related past interactions| ChromaDB semantic search  |
| 3        | Known entities           | Entity index              |

Each section is formatted and fitted within the remaining budget — critical data is never truncated by lower-priority context.

---

## Agent Architecture

### Manager LLM

- **Model**: `qwen3.5-9b-mlx` (configurable via `MANAGER_MODEL`)
- **Responsibilities**: Request routing (chat/plan/refuse), depth reasoning, final response composition
- **Features**: 2-attempt retry with repair prompt, keyword-based fallback routing

### Worker LLM

- **Model**: `qwen3.5-2b-mlx` (configurable via `WORKER_MODEL`)
- **Responsibilities**: Structured tool selection, tool result interpretation
- **Features**: Up to 2 tool-calling rounds, auto-resolution of missing tool names, schema-validated output

### Specialist Agents

| Agent             | Tool(s)            | Capabilities                                              |
|-------------------|--------------------|-----------------------------------------------------------|
| `news_agent`      | `fetch_news`       | YouTube channel + web RSS headlines, compressed summaries |
| `mail_agent`      | `fetch_emails`     | Gmail API (read-only), classroom detection, deadline extraction |
| `calendar_agent`  | `get_events`, `create_event` | Read today/tomorrow events, create deadline events   |
| `task_agent`      | `list_tasks`, `list_deadlines`, `add_task`, `complete_task` | Full task CRUD |

---

## Project Structure

```
agent 6/
├── app/
│   ├── main.py                          # FastAPI app, /query and /health endpoints
│   ├── config.py                        # Pydantic settings (30+ env vars)
│   ├── orchestrator.py                  # Control plane: intake → context → route → plan → dispatch → persist
│   ├── core/
│   │   ├── context.py                   # RunContext dataclass (request-scoped state)
│   │   └── prompts.py                   # PromptCatalog (live file reloading, template rendering)
│   ├── memory/
│   │   ├── graph_retriever.py           # Facade: progressive depth API + legacy build()
│   │   ├── sqlite_store.py              # 13 tables, recursive CTEs, FTS5, entity index, edges
│   │   ├── vector_store.py              # 4 ChromaDB collections with in-memory fallback
│   │   ├── graph/
│   │   │   ├── models.py                # TraversalDirective, ContextWindow, LevelSummary
│   │   │   ├── builder.py               # GraphBuilder: node creation + entity extraction + cross-session linking
│   │   │   ├── traverser.py             # GraphTraverser: BFS with lineage expansion, sibling walk
│   │   │   └── summarizer.py            # LevelSummarizer: hierarchical compression via worker LLM
│   │   └── retrieval/
│   │       ├── depth_reasoner.py        # 3-tier depth reasoning (LLM → keyword → fallback)
│   │       ├── strategies.py            # 4 strategy implementations + factory
│   │       └── assembler.py             # Priority-based, token-budget-aware context assembly
│   ├── llm/
│   │   ├── client.py                    # OpenAI SDK wrapper for LM Studio
│   │   ├── manager.py                   # Route requests, compose final answers
│   │   └── worker.py                    # Structured tool-calling loop (max 2 rounds)
│   ├── agents/
│   │   ├── registry.py                  # Agent function registry
│   │   ├── news_agent.py
│   │   ├── mail_agent.py
│   │   ├── calendar_agent.py
│   │   └── task_agent.py
│   ├── tools/
│   │   ├── google_auth.py               # Google OAuth flow
│   │   ├── news_tools.py                # YouTube + web feed fetchers
│   │   ├── email_tools.py               # Gmail API client
│   │   ├── calendar_tools.py            # Google Calendar client
│   │   └── task_tools.py                # Local task/deadline CRUD
│   ├── schemas/                         # Pydantic models: ManagerDecision, Plan, WorkerOutput, etc.
│   ├── verifier/                        # Plan and result validation
│   └── utils/                           # Structured logger, JSON utilities, datetime helpers
├── prompts/
│   ├── soul.md                          # Permanent behavior contract
│   ├── manager.json                     # 4 scenarios: reason_depth, route_request, repair, compose
│   └── worker.json                      # 4 agent scenarios + shared repair/user templates
├── data/                                # Runtime data (SQLite DB, ChromaDB, logs, Google token)
├── whatsapp_bridge/                     # Node.js WhatsApp bridge (whatsapp-web.js)
├── requirements.txt
└── .env
```

---

## Tech Stack

| Layer              | Technology                              |
|--------------------|-----------------------------------------|
| API Server         | FastAPI 0.136 + Uvicorn 0.46            |
| LLM Inference      | LM Studio (local) via OpenAI SDK 2.36   |
| Models             | Qwen 3.5 (9B manager, 2B worker, MLX)  |
| Vector Store       | ChromaDB 1.5 (4 collections)            |
| Persistence        | SQLite 3 (WAL mode, 13 tables, FTS5)    |
| External APIs      | Google API Client (Gmail, Calendar)      |
| Data Validation    | Pydantic 2.13 + Pydantic-Settings 2.14  |
| WhatsApp Bridge    | Node.js, whatsapp-web.js                |
| Language           | Python 3.12                              |

---

## Setup

### Prerequisites

- Python 3.11+
- Node.js 18+
- [LM Studio](https://lmstudio.ai/) running locally at `http://127.0.0.1:1234/v1`
- A Google Cloud OAuth Desktop client JSON for Gmail and Calendar access

### Installation

```bash
# Clone or extract the project
cd "agent 6"

# Create and activate virtual environment
python3 -m venv venv
source venv/bin/activate  # macOS/Linux

# Install Python dependencies
pip install -r requirements.txt

# Copy and configure environment variables
cp .env.example .env
# Edit .env — set INTERNAL_TOKEN and any model overrides

# Install WhatsApp bridge dependencies (optional)
cd whatsapp_bridge && npm install && cd ..
```

### Google Cloud Setup

1. Create a [Google Cloud project](https://console.cloud.google.com/)
2. Enable the **Gmail API** and **Google Calendar API**
3. Create an OAuth client of type **Desktop app**
4. Download the JSON client file and save as `credentials.json` in the project root
5. The first run will open a browser for OAuth consent and save the token to `data/google_token.json`

### Environment Variables (.env)

| Variable                         | Default                    | Description                              |
|----------------------------------|----------------------------|------------------------------------------|
| `LM_STUDIO_BASE_URL`             | `http://127.0.0.1:1234/v1` | LM Studio OpenAI-compatible endpoint     |
| `LM_STUDIO_API_KEY`              | `lm-studio`                | API key for LM Studio                    |
| `MANAGER_MODEL`                  | `qwen3.5-9b-mlx`           | Model for routing, depth reasoning       |
| `WORKER_MODEL`                   | `qwen3.5-2b-mlx`           | Model for tool execution                 |
| `INTERNAL_TOKEN`                 | `change-me-now`            | Auth token for /query endpoint           |
| `LLM_TIMEOUT_SECONDS`            | `240`                      | LLM call timeout                         |
| `MANAGER_MAX_TOKENS`             | `384`                      | Max tokens for manager responses         |
| `WORKER_MAX_TOKENS`              | `256`                      | Max tokens for worker responses          |
| `MAX_INPUT_CHARS`                | `5000`                     | Max input length                         |
| `DATA_DIR` / `DB_PATH`           | `data/`                    | Persistent data location                 |
| `MAIL_LOOKBACK_HOURS`            | `24`                       | Default mail fetch window                |
| `LOCAL_TIMEZONE`                 | `Asia/Kolkata`             | Timezone for date handling               |

---

## Running

### 1. Start LM Studio

Launch LM Studio, load the manager and worker models, and start the local server on port 1234.

### 2. Start the API Server

```bash
source venv/bin/activate
python -m uvicorn app.main:app --host 127.0.0.1 --port 8000
```

### 3. Start the WhatsApp Bridge (optional)

```bash
cd whatsapp_bridge
node server.js
```

Scan the QR code with WhatsApp, then set the allowlisted chat ID.

### 4. Send Requests

```bash
# Direct API
curl -X POST http://127.0.0.1:8000/query \
  -H "Content-Type: application/json" \
  -H "X-Internal-Token: change-me-now" \
  -d '{"message": "give me today'\''s news summary"}'

# Health check
curl http://127.0.0.1:8000/health
```

### API Response Format

```json
{
  "reply": "Today's tech news summary...",
  "mode": "plan",
  "plan": {"steps": [{"agent": "news_agent", "task": "..."}]},
  "results": [{"agent": "news_agent", "status": "ok", "message": "...", "data": {...}}],
  "request_id": "a1b2c3d4e5f6",
  "source_id": "default"
}
```

---

## Security Boundaries

- **Gmail**: Read-only scope (`gmail.readonly`). No send, edit, or delete code paths.
- **Calendar**: Events-only scope (`calendar.events`). No edit or delete code paths.
- **Auth header**: All `/query` requests require `X-Internal-Token` matching `INTERNAL_TOKEN`.
- **Structured logging**: Logs contain digests and metadata, never raw model outputs.
- **Source isolation**: Each `source_id` (WhatsApp chat, API key) maintains its own independent memory graph, entity index, and summaries.
- **100% local**: All LLM inference, vector storage, and graph persistence run on-device. No data is sent to external services.

---

## Memory Isolation & Scaling

- Each `source_id` is a complete memory boundary — separate graph nodes, edges, level summaries, entities, deadlines, tasks, and vector collections
- The system scales horizontally by source_id isolation, not by adding vector database nodes
- SQLite with WAL mode handles concurrent requests within the same source_id safely
- The WhatsApp bridge forwards the chat ID as `source_id`, giving each conversation its own independent memory

---

## Design Patterns

| Pattern            | Usage                                                   |
|--------------------|----------------------------------------------------------|
| Strategy           | Retrieval strategies (Broad, Focused, Exact, None)       |
| Facade             | `GraphMemoryRetriever` wraps the entire memory subsystem |
| Three-tier fallback| Depth reasoning, routing, summarization, vector search   |
| Priority-based budget| Context assembly with token-constrained section ordering|
| Recursive CTE      | Graph lineage traversal (ancestors + descendants)         |
| Foreign key cascade| Data integrity across 13 tables                          |

---

## Deployment Footprint

- **Single Python process** + LM Studio
- **One SQLite file** for all persistent state (`data/app.db`)
- **One ChromaDB directory** for vector embeddings (`data/chroma/`)
- Ready for Raspberry Pi, Mac Mini, or any local server deployment
- No Docker required (optional), no cloud services
