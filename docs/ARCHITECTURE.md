# AgentGarden — Architecture

**Version:** 2.3
**Changes:** Multi-tenant RLS, Domain Guard, Conversation Resolver, KG (N3.js/TTL), Metric Registry (replaces Cube.js), Ontop deferred to Phase 3, Orchestrator added in Phase 2.5 (see ORCHESTRATOR.md)
**Date:** 2026-05-06
**Status:** Approved for Phase 1; Phase 2.5 orchestrator design in ORCHESTRATOR.md

---

## 1. Tech Stack

| Layer | Technology |
|-------|------------|
| Frontend | Next.js 15 + TypeScript |
| Backend | Next.js API Routes + TypeScript |
| ORM | Prisma |
| App Database | PostgreSQL (platform metadata, tenants, configs) |
| LLM SDK | Vercel AI SDK (`ai`) |
| Default LLM | Claude Sonnet 4.6 (pluggable) |
| Semantic Layer | Metric Registry (YAML per tenant, stored in DB) |
| Vector Store | PGVector (in app Postgres) |
| Knowledge Graph | RDF/TTL files via N3.js (metadata relationships) |
| SPARQL Engine | Oxigraph (Phase 3, for complex KG traversal) |
| Tenant Data Sources | Postgres, DuckDB, MySQL (configurable per tenant) |
| Auth | Internal auth (session-based, Phase 1) |
| Orchestrator Queue | Redis + BullMQ (Phase 2.5) |
| Orchestrator Workers | Node worker pool (stateless, Phase 2.5) |
| Scheduler | Postgres advisory-lock leader election (Phase 2.5) |
| Deployment | Docker Compose (local dev) |

---

## 2. Multi-Tenancy Model

- **Isolation:** Row-level isolation via `tenant_id` on all shared tables
- **Postgres RLS:** Enabled on all tenant-scoped tables
- **Per-tenant configuration:**
  - System prompt / business context
  - Data source connections (Postgres, DuckDB, MySQL)
  - Metric definitions (Metric Registry YAML)
  - LLM provider + model selection
  - MCP server registrations
  - Agent definitions

---

## 3. System Components

### 3.1 Frontend (Next.js)
- Chat interface with streaming responses
- Admin panel (data sources, agents, MCP servers, LLM config)
- Observability dashboard (LLM usage, agent execution, query metrics)
- Multi-turn conversation history per tenant/user

### 3.2 Backend (Next.js API Routes)
- `/api/chat` — streaming chat endpoint (Vercel AI SDK)
- `/api/admin/*` — tenant admin CRUD
- `/api/query/*` — query execution and result formatting
- `/api/metrics/*` — observability data

### 3.3 LLM Gateway
- Built on Vercel AI SDK provider abstraction
- Supported providers: Anthropic, OpenAI, Google, Groq, Mistral
- Per-tenant default model config stored in DB
- Auto-router: selects model based on task type + cost + latency

### 3.4 Metric Registry
- Per-tenant metric definitions stored as YAML in DB
- Each metric: name, description, SQL expression, base table
- LLM uses metric definitions + KG subgraph (FK join paths) to generate SQL directly
- No external service — fully owned pipeline
- Admin manages metrics via admin panel (create, edit, version)

### 3.5 Domain Guard
- First gate — runs before any query processing
- Fast model (Haiku / Gemini Flash) checks if question is within tenant's domain
- Uses tenant's system prompt + business context as boundary definition
- Returns `{ allowed: boolean, reason: string }`
- If blocked: returns tenant-configured refusal message, logs attempt
- Blocks: out-of-domain questions, questions about unconfigured data sources

### 3.6 Conversation Resolver
- Runs after Domain Guard, before Query Router
- Receives current message + last 10 messages (sliding window)
- Detects follow-up type: co-reference | refinement | new question
- If follow-up: merges conversation context into a standalone, self-contained query
- If new question: passes through unchanged
- Output is always a fully resolved query — Query Router never sees partial context

### 3.7 Query Router
- LLM-based classifier (fast model)
- Routes to: Semantic path | Agentic path
- Falls back gracefully: Semantic → Agentic → Error
- SPARQL path reserved as future extension point (not built in Phase 1/2)

### 3.8 Agentic Fallback
- Vercel AI SDK tool use + multi-step agent
- RAG via PGVector: retrieves relevant schema context
- Text-to-SQL with self-correction loop (max 2 retries)
- SQL validation before execution

### 3.9 Agent Engine
- Two agent scopes: **shared** (admin-defined, available to all tenant users) and **personal** (user-defined, visible only to that user)
- Query Router auto-selects from both shared and personal agents based on query intent vs agent descriptions
- User can explicitly invoke any agent they have access to by naming it in the message
- Each conversation is a fresh start — no cross-conversation memory
- Multi-step execution: LLM plans → calls MCP tools → synthesizes result
- Interactive: if a tool requires user input, agent asks follow-up question mid-task and waits
- MCP tool scope: agent can only call tools from its configured MCP servers
- Vercel AI SDK `generateText` with tool use drives the execution loop

### 3.10 MCP Integration
- Two MCP server scopes:
  - **Shared** — registered by admin, available to all agents in the tenant
  - **Personal** — registered by individual users, available only within their personal agents
- Admin can configure a per-tenant allowlist of permitted MCP endpoint patterns
- Test/ping on registration before making live (both scopes)
- Vercel AI SDK MCP client connects at query time
- Tools from registered servers exposed to agents based on agent config
- Write operations permitted — scope controlled at MCP server + agent level

### 3.11 Observability
- Every query logged: tenant, SQL, tokens, latency, cost, path taken
- Per-message execution trace stored in `MessageExecutionTrace` (domain guard, resolver, router decision, agent steps, MCP tool calls, SQL, data source)
- Trace rendered in chat UI as collapsible "Steps" section per message
- LLM usage metrics aggregated per tenant
- Admin dashboard reads from QueryLog + MessageExecutionTrace
- Workflow runs (Phase 2.5) extend the trace: span tree of run → step → agent iteration → tool call → sub-run; live SSE tail in Run Inspector

### 3.12 Orchestrator (Phase 2.5)

Adds durable, multi-agent execution on top of the chat-turn agent runtime. Full design in `ORCHESTRATOR.md`. Summary of components:

- **Workflow registry** — versioned YAML definitions per tenant (DAG of steps; types: agent, tool, sub_workflow, approval, branch, parallel, loop)
- **Triggers** — chat, cron, webhook (HMAC-signed), event (internal bus), manual
- **Run queue** — Redis + BullMQ; jobs are AgentRun ids
- **Worker pool** — stateless Node workers; claim a run, execute steps, persist `RunCheckpoint` at every boundary, resume any run after restart
- **Scheduler** — single leader (Postgres advisory lock) emits cron jobs into the queue
- **Webhook receiver** — `/api/triggers/{token}` verifies HMAC and enqueues a run
- **Approval service** — `approval` step persists `ApprovalRequest`, run state goes to `waiting_for_human`; decision endpoint enqueues a resume job
- **Budget enforcement** — live counters per run (tokens, USD, wall-clock, tool calls); breach transitions run to `budget_exceeded`
- **Idempotency** — every tool call carries `hash(run_id + step_id + attempt_input)`; `ToolCallCache` replays successful results
- **Sub-agent-as-tool** — each agent exposed as synthetic tool `agent.<name>`; capability narrowing + max depth cap (default 3)
- **Run Inspector** — admin panel UI: span tree, live SSE tail, fork/replay from checkpoint

Run state machine: `queued → running → waiting_for_human → retrying → completed | failed | cancelled | budget_exceeded`. Terminal states are immutable; "rerun" is a fork from a checkpoint, not a mutation.

---

## 4. Query Flow

```
User Message
    │
    ▼
Auth + Tenant Context (load tenant config, LLM, system prompt)
    │
    ▼
[1] Domain Guard (fast model)
    ├── out of domain ──→ Refusal Response → Log
    │
    │ in domain
    ▼
[2] Conversation Resolver
    ├── follow-up → merge with conversation history → standalone query
    └── new question → pass through unchanged
    │
    ▼
[3] Query Router (fast LLM classifier)
    ├── Semantic Path
    │       │
    │       ▼
    │   Load: Metric Registry + KG subgraph (relevant tables + FK joins)
    │       │
    │       ▼
    │   LLM generates SQL (metric definitions + join paths as context)
    │       │
    │       ▼
    │   Validate → Execute on Tenant Data Source
    │
    ├── Agentic Path (data queries)
    │       │
    │       ▼
    │   RAG (PGVector schema context)
    │       │
    │       ▼
    │   Text-to-SQL → Validate → Execute (retry ≤2)
    │
    └── Agent Path (actions via MCP tools)
            │
            ▼
        Auto-select agent (by intent) OR user-named agent
            │
            ▼
        Agent loop: LLM plans → calls MCP tools
            │
            ├── tool needs user input → ask follow-up → wait → continue
            └── tool completes → next step or done
    │
    ▼
[4] LLM Response Generation (stream to client)
    │
    ▼
[5] Save to ConversationSession + Log to Observability
```

### 4.1 Orchestrator Run Flow (Phase 2.5)

```
Trigger (chat | cron | webhook | event | manual)
    │
    ▼
/api/triggers/* (or chat router classifies "this is a workflow")
    │
    ▼
Enqueue AgentRun (state=queued)
    │
    ▼
Worker claims run → state=running → RunCheckpoint(start)
    │
    ▼
┌── Run Loop ───────────────────────────────────────────┐
│  while not terminal:                                  │
│    pick next ready step from DAG                      │
│    check budget (tokens / USD / wall / tool calls)    │
│    execute step:                                      │
│      agent       → invoke agent runtime + LLM gateway │
│      tool        → MCP client + ToolCallCache + retry │
│      sub_workflow→ enqueue child AgentRun (parent_id) │
│      approval    → write ApprovalRequest →            │
│                     state=waiting_for_human, release  │
│      branch      → evaluate condition, jump           │
│      parallel    → fan-out, join                      │
│      loop        → bounded iteration                  │
│    on success: write blackboard, RunCheckpoint        │
│    on failure: retry policy → on.failure / DLQ        │
└───────────────────────────────────────────────────────┘
    │
    ▼
Compute output mapping → state=completed → emit run.completed event
```

---

## 5. Data Model (App DB — Prisma)

Key entities:
- `Tenant` — tenant record, system prompt, LLM config
- `User` — internal user, belongs to tenant (roles: admin, data_steward, viewer)
- `DataSource` — connection config (type, encrypted credentials, status, last_synced_at) — property of Database in KG
- `DataSourceTable` — introspected table (status: active|inactive, description)
- `DataSourceColumn` — introspected column (status: active|inactive, description)
- `SchemaRefreshLog` — log of every schema refresh (manual or scheduled, delta summary)
- `SchemaRefreshSchedule` — cron schedule per data source
- **Knowledge Graph** — RDF TTL files at `storage/kg/{tenant_id}/{data_source_id}.ttl` representing: Database → Schema → Table/View → Column, plus Indexes, ForeignKeys, Functions, Triggers, Sequences with unique IRIs
- `MetricDefinition` — per-tenant metric (name, description, SQL expression, base table, version)
- `DomainConfig` — domain boundary definition, allowed topics, refusal message per tenant
- `Agent` — agent definition (name, description, system prompt, LLM model, max iterations, status, scope: shared|personal, owner_id: null if shared)
- `AgentMcpServer` — join table: which MCP servers + tools an agent can use
- `McpServer` — MCP server registration (endpoint, auth config, status, scope: shared|personal, owner_id: null if shared)
- `McpAllowlist` — per-tenant permitted MCP endpoint patterns (admin-configured)
- `LlmProvider` — available LLM providers + API keys (encrypted)
- `TenantInvite` — invite token, email, role, expiry
- `ConversationSession` — chat session per user (sliding 10-message window)
- `ConversationMessage` — individual messages within a session
- `MessageExecutionTrace` — per-message step trace (domain guard, resolver, router, agent steps, tool calls, SQL, data source used)
- `SavedQuery` — user-pinned queries (quick-launch shortcuts)
- `FlaggedAnswer` — flagged response with full context (question, SQL, result, user correction)
- `AuditLog` — all admin actions (actor, action, resource, before/after values)
- `QueryLog` — every query execution record (path, SQL, tokens, latency, cost, domain guard result)

**Orchestrator (Phase 2.5) — see ORCHESTRATOR.md §11 for full Prisma schema:**
- `Workflow` — tenant-scoped workflow record (current version, status: draft|active|archived)
- `WorkflowVersion` — versioned YAML + parsed JSON + validation result
- `Trigger` — chat | cron | webhook | event | manual; per-trigger config (cron expr, webhook token hash, event name)
- `AgentRun` — single execution; state machine; `parent_run_id` for sub-runs; budget JSON with live counters
- `RunCheckpoint` — append-only state transitions + blackboard snapshot at step boundaries
- `ApprovalRequest` — pending HITL decision; payload, role/user, timeout, decision payload
- `ToolCallCache` — idempotency key → result; replays cached result on retry
- `WorkflowEvent` — internal pub/sub events that can trigger workflows

---

## 6. Phased Roadmap

### Phase 1 — MVP (6–8 weeks)
- [ ] Prisma schema + Postgres setup
- [ ] Tenant self-registration + first admin creation
- [ ] Invite-only user management (invite by email, role assignment)
- [ ] Admin panel: data source management + domain config (AI-assisted)
- [ ] Test as user mode in admin panel
- [ ] Flagged answer review (full context: question, SQL, result, correction)
- [ ] Audit log for all admin actions
- [ ] Schema introspection on connect (Database → Schema → Table/View → Column + Indexes, FKs, Functions)
- [ ] Knowledge graph generation (TTL/NT via N3.js, unique IRIs per resource)
- [ ] Manual schema refresh with delta detection (inactive marking for removed resources)
- [ ] Scheduled schema refresh (cron per data source)
- [ ] Metadata catalog: browse resource hierarchy, add descriptions (admin + data steward)
- [ ] Description updates written back as dc:description triples into KG
- [ ] Metric Registry (per-tenant YAML metric definitions, admin CRUD)
- [ ] Domain Guard (per-tenant domain boundary enforcement)
- [ ] Conversation Resolver (multi-turn follow-up handling)
- [ ] Vercel AI SDK chat with streaming + progress steps + abort
- [ ] LLM gateway (Anthropic + OpenAI providers)
- [ ] Semantic query path (Metric Registry + KG context → LLM → SQL → execute)
- [ ] Basic agentic fallback (RAG + Text-to-SQL)
- [ ] Result display: text summary + paginated table + chart (system-selected, user-overridable)
- [ ] CSV download + PDF/image export
- [ ] Saved queries (pin/quick-launch)
- [ ] Feedback flagging (thumbs down + optional correction text)
- [ ] Per-tenant hints in chat UI
- [ ] Query + conversation logging

### Phase 2 — Production Ready (4–5 weeks)
- [ ] MCP server admin + agent builder
- [ ] Auto LLM router (cost + latency based)
- [ ] Observability dashboard
- [ ] SQL validation before execution
- [ ] Row-level security enforcement
- [ ] Additional LLM providers (Groq, Gemini, Mistral)

### Phase 2.5 — AI Agent Orchestrator (7 weeks) — see ORCHESTRATOR.md
**2.5a Foundation (3 weeks)**
- [ ] Workflow + WorkflowVersion + AgentRun + RunCheckpoint Prisma tables
- [ ] Workflow YAML parser + save-time validator (DAG check, capability check, blackboard refs)
- [ ] Redis + BullMQ queue + Node worker pool
- [ ] Run loop: claim → execute → checkpoint → resume on restart
- [ ] Step types: `agent`, `tool`, `branch`
- [ ] Manual trigger + Run Inspector (read-only span tree)

**2.5b Triggers + HITL (2 weeks)**
- [ ] Cron scheduler (Postgres advisory-lock leader)
- [ ] Webhook receiver `/api/triggers/{token}` with HMAC verify
- [ ] `approval` step + ApprovalRequest + Approval Inbox UI
- [ ] Live budget enforcement (tokens, USD, wall-clock, tool calls)
- [ ] ToolCallCache + per-step retry policies (exponential backoff)

**2.5c Composition (2 weeks)**
- [ ] `sub_workflow` and `parallel` step types
- [ ] Sub-agent-as-tool (synthetic `agent.<name>` MCP tool, capability narrowing, depth cap)
- [ ] Run fork / replay from checkpoint
- [ ] Workflow YAML editor (Monaco + schema validation) in admin panel
- [ ] Run DLQ view for failed runs

### Phase 3 — Scale (Ongoing)
- [ ] SPARQL endpoint via Oxigraph (serve KG TTL files for complex traversal)
- [ ] Chart / visualization generation
- [ ] Fine-tuned domain guard + routing model
- [ ] Anomaly detection + proactive insights
- [ ] Visual DAG editor for workflows (drag-and-drop, replaces YAML for most authors)
- [ ] Cross-tenant workflow / agent marketplace
- [ ] Durable queue (NATS JetStream or Temporal) if scale demands beyond Redis+BullMQ
- [ ] External event bus triggers (Kafka / pub/sub adapters)

---

## 7. Docker Compose Services (Local Dev)

| Service | Image | Port |
|---------|-------|------|
| `app` | Node 22 (Next.js) | 3000 |
| `postgres` | postgres:16 with pgvector | 5432 |
| `redis` (Phase 2.5) | redis:7 | 6379 |
| `worker` (Phase 2.5) | Node 22 worker pool | — |
| `scheduler` (Phase 2.5) | Node 22 (single instance, leader-locked) | — |
