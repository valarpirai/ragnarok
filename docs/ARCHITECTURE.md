# DataChat AI — Architecture

**Version:** 2.2
**Changes:** Multi-tenant RLS, Domain Guard, Conversation Resolver, KG (N3.js/TTL), Metric Registry (replaces Cube.js), Ontop deferred to Phase 3
**Date:** 2026-04-30
**Status:** Approved for Phase 1

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

### 3.9 MCP Integration
- Tenants register MCP servers via admin panel
- Vercel AI SDK MCP client connects at query time
- MCP tools exposed to agent as callable tools

### 3.10 Observability
- Every query logged: tenant, SQL, tokens, latency, cost, path taken
- LLM usage metrics aggregated per tenant
- Agent execution traces stored in DB
- Admin dashboard reads from observability tables

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
    └── Agentic Path
            │
            ▼
        RAG (PGVector schema context)
            │
            ▼
        Text-to-SQL (with MCP tools)
            │
            ▼
        Validate → Execute (retry ≤2)
    │
    ▼
[4] LLM Response Generation (stream to client)
    │
    ▼
[5] Save to ConversationSession + Log to Observability
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
- `Agent` — agent definitions per tenant
- `McpServer` — MCP server registrations per tenant
- `LlmProvider` — available LLM providers + API keys (encrypted)
- `TenantInvite` — invite token, email, role, expiry
- `ConversationSession` — chat session per user (sliding 10-message window)
- `ConversationMessage` — individual messages within a session
- `SavedQuery` — user-pinned queries (quick-launch shortcuts)
- `FlaggedAnswer` — flagged response with full context (question, SQL, result, user correction)
- `AuditLog` — all admin actions (actor, action, resource, before/after values)
- `QueryLog` — every query execution record (path, SQL, tokens, latency, cost, domain guard result)

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

### Phase 3 — Scale (Ongoing)
- [ ] SPARQL endpoint via Oxigraph (serve KG TTL files for complex traversal)
- [ ] Chart / visualization generation
- [ ] Fine-tuned domain guard + routing model
- [ ] Anomaly detection + proactive insights

---

## 7. Docker Compose Services (Local Dev)

| Service | Image | Port |
|---------|-------|------|
| `app` | Node 22 (Next.js) | 3000 |
| `postgres` | postgres:16 with pgvector | 5432 |
