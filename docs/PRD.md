**Product Requirements Document (PRD)**

**Product Name:**  
**AgentGarden** – Build, Connect, and Run AI Agents on Your Data

**Version:** 1.3 (adds AI Agent Orchestrator capabilities — Phase 2.5)
**Date:** May 6, 2026
**Status:** Approved — see ARCHITECTURE.md and ORCHESTRATOR.md for current technical decisions

---

### 1. Executive Summary

**AgentGarden** is a multi-tenant platform where users build, connect, and run AI agents on top of their business data and external tools.

The foundation is a governed data layer — **Metric Registry** (business vocabulary) + **Knowledge Graph** (schema relationships) — that makes data queries accurate and trustworthy. On top of that, any user can create personal agents, connect MCP servers, and automate workflows — not just query data.

**Phase 2.5 adds an AI Agent Orchestrator** that turns AgentGarden from a single-agent chat platform into a coordinator of multi-agent workflows: durable runs that survive disconnects, cron and webhook triggers, human-in-the-loop approvals, and per-run budget controls. Agents can delegate to other agents, and runs can be replayed for debugging and evals.

**Core Value Proposition:**
Give every user a personal AI agent workspace: query data in natural language, automate multi-step workflows via MCP tools and other agents, and build agents that understand your business — without writing SQL or code.

---

### 2. Problem Statement

- Business users waste hours waiting for reports or learning SQL.
- Direct Text-to-SQL has inconsistent accuracy and security risks.
- No single source of truth for business metrics (“revenue”, “active customer”, etc.).
- Different databases (Postgres vs DuckDB) require different query strategies.
- Users need to automate workflows and connect external tools — not just read data.
- Agent and MCP setup is locked behind admin roles, blocking individual productivity.
- A single-agent chat turn can't run long jobs, wait on external events, or pause for human approval.
- Multi-step automations need to survive process restarts, retry safely, and stay within hard cost/time budgets.
- Need for governance, auditability, and row-level security.

---

### 3. Objectives & Success Metrics

**Business Goals**
- Increase data self-service adoption by 70% within 3 months.
- Reduce time-to-insight from hours to <30 seconds.
- Achieve ≥95% query success rate on governed metrics.

**Product KPIs**
- Query Accuracy: ≥95% (Semantic path), ≥85% (Agentic path)
- Average Response Time: <8 seconds (Semantic), <15 seconds (Agentic)
- User Satisfaction (CSAT): ≥4.5/5
- Cost per Query: <$0.05 (target)
- % of queries routed to Semantic Layer: ≥75%

**Orchestrator KPIs (Phase 2.5)**
- Workflow Run Success Rate: ≥97% (excluding intentional rejections)
- Run Resume Rate after Worker Restart: 100% (no data loss)
- Median Time-to-Approve (HITL): <2h during business hours
- Budget Breach Rate: <0.5% of runs
- Idempotent Tool Replay: 100% of retried side-effectful calls return cached result

---

### 4. User Personas

1. **Business Analyst** (Primary) – Chennai/Tamil Nadu team  
   Needs monthly trends, regional performance, product-wise analysis.

2. **Sales Manager**  
   Wants real-time pipeline status, top customers, forecast accuracy.

3. **Operations Head**  
   Asks about inventory, supply chain metrics, exceptions.

4. **Executive**  
   High-level summaries, charts, “what-if” questions.

---

### 5. Functional Requirements

#### 5.1 Core Features

**FR-01: Natural Language Chat Interface**
- Streaming responses with step-by-step progress indicator
- User can abort any in-progress request
- Conversation history with ability to resume past conversations
- Follow-up questions resolved in context (co-reference, refinement)
- Per-tenant hints shown in chat input to guide users
- Desktop only

**FR-02: Result Display**
- Every response shows text summary + data table + chart together
- Chart type auto-selected by system; user can override via NL or UI toggle
- Paginated preview for large result sets
- Download full result as CSV
- Export result (chart + table + summary) as PDF or image

**FR-03: Domain Guard**
- Per-tenant domain boundary enforcement
- Out-of-domain questions return “I can't answer that”
- No explanation or suggested alternatives shown to user

**FR-04: Intelligent Query Routing**
- LLM-based router: Semantic path (Metric Registry + KG) or Agentic path (RAG + Text-to-SQL)
- Auto-routes to correct data source via Knowledge Graph — no user selection needed

**FR-05: Governed Semantic Path (Primary)**
- Metric Registry: per-tenant named metric definitions (SQL expressions + descriptions)
- Knowledge Graph: RDF graph of Database → Schema → Table → Column + relationships (TTL/NT)
- LLM generates SQL using Metric Registry + KG join paths as context

**FR-06: Agentic Fallback**
- RAG (PGVector) + Text-to-SQL with self-correction loop (max 2 retries)
- MCP tool support for registered tenant agents

**FR-07: Secure Query Execution**
- Read-only connections
- Automatic row-level security (tenant_id injected at query time)
- SQL validation before execution
- Max row limit (configurable, default 10,000)

**FR-08: Saved Queries**
- Users can pin/save frequently used queries
- Saved queries shown as quick-launch shortcuts in chat UI
- Manually triggered only — no scheduling

**FR-09: Feedback & Flagging**
- Every response has a flag (thumbs down) button
- On flag: optional text field to describe the issue or provide the correct answer
- Flagged responses logged for admin review

**FR-10: Observability**
- Log every query: SQL, tokens, latency, cost, path taken, domain guard result
- Admin dashboard: LLM usage, agent execution, query metrics, domain guard blocks, flagged answers — tenant-scoped

**FR-11: Admin — Tenant Setup**
- Tenant self-registers; first admin created on registration
- Invite-only user model — admin invites by email, assigns role on invite
- Roles: admin, data_steward, viewer

**FR-12: Admin — Domain Configuration**
- AI-assisted setup: system suggests topics from connected schema + admin-provided context
- Admin configures: business context (free text) + allowed topic list
- All admin actions captured in audit log

**FR-13: Admin — Data Source Management**
- Dynamic connection form per type (Postgres, DuckDB, MySQL)
- Test connection with descriptive error on failure
- Schema introspection on connect → Knowledge Graph generated
- Manual + scheduled schema refresh with per-run delta summary

**FR-14: Admin — Metric Registry**
- Manual metric definition + AI-suggested metrics from schema
- Test metric against data source before activating
- Metric usage analytics (frequency per metric)
- Edits take effect immediately; soft-delete for deactivation

**FR-15: Admin — Flagged Answer Review**
- Admin sees full context: original question, resolved query, path, SQL, result, user correction
- Mark reviewed; used for metric and domain config improvement

**FR-16: Admin — Test as User Mode**
- Admin can simulate the full query flow to verify config before going live

**FR-17: Admin — MCP & Agent Management**
- Register shared MCP servers (available to all users in the tenant) with test/ping button
- Define shared agents: tools, MCP servers, LLM model, max iterations
- Govern which MCP servers users are allowed to connect personally

**FR-18: User — Personal Agent Builder**
- Any user can create personal agents (visible only to themselves)
- Configure: name, description, system prompt, allowed MCP servers + tools, LLM model
- Personal agents appear in the chat sidebar alongside shared agents
- Personal agents auto-selectable by the router or explicitly invokable by name

**FR-19: User — Personal MCP Connection**
- Users can register personal MCP servers (scoped to their account only)
- Same connect/test/ping flow as admin MCP management
- Admin can restrict which MCP server endpoints are permitted (allowlist per tenant)
- Personal MCP tools available only within that user's agents

**FR-20: Orchestrator — Workflow Definition (Phase 2.5)**
- Admin authors workflows as YAML in the admin panel (versioned, validated on save)
- Step types: agent, tool, sub_workflow, approval, branch, parallel, loop
- Inputs declared via JSON Schema; outputs mapped from a per-run blackboard
- Inactive/active status; activating a new version does not disturb in-flight runs

**FR-21: Orchestrator — Triggers (Phase 2.5)**
- Trigger types: chat, cron (with timezone), webhook (HMAC-signed token), event (internal bus), manual
- A workflow may declare multiple triggers
- Webhook tokens shown once at creation, stored hashed, rotatable
- Cron misfires logged; not auto-replayed

**FR-22: Orchestrator — Durable Runs**
- Every run persisted with state machine: queued → running → waiting_for_human → retrying → completed | failed | cancelled | budget_exceeded
- Runs survive worker restart and resume from latest checkpoint
- Run can be cancelled by initiator, admin, or parent run
- Run can be forked (replay from any checkpoint with edited input)

**FR-23: Orchestrator — Human-in-the-Loop Approvals**
- `approval` step pauses run and creates an ApprovalRequest routed to a role or specific user
- Approver actions: approve, edit-and-approve, reject (with reason)
- Default 24h timeout, configurable per step; on timeout the workflow's `on.timeout` edge fires
- Approval Inbox UI in admin panel; pending approvals also show as a chat sidebar badge for the approver

**FR-24: Orchestrator — Budgets, Retries, Idempotency**
- Per-run budget caps: tokens, USD, wall-clock seconds, tool-call count — enforced live
- Retry policy per step (max attempts, exponential backoff, retry-on classes)
- Idempotency key per tool call; successful results cached and replayed on retry
- Failed runs land in a Run DLQ view for admin inspection and replay

**FR-25: Orchestrator — Sub-Agent Delegation**
- Each agent in the tenant is exposed as a synthetic tool (`agent.<name>`) callable by other agents
- Capability narrowing: a calling agent cannot reach tools through a sub-agent that it could not call directly
- Configurable max sub-agent depth (default 3)
- Sub-runs share the parent run's budget envelope

**FR-26: Orchestrator — Run Inspector & Observability**
- Run Inspector UI: span tree of workflow → steps → agent iterations → tool calls → sub-runs
- Live SSE tail of in-flight runs
- Per-span input/output viewers; large payloads behind a "Load full" button
- Aggregations: run counts, success rate, P50/P95 duration, cost per workflow, top failing steps, approval queue depth

#### 5.2 Non-Functional Requirements

- **Security:** SOC2 / GDPR compliant, no raw user input in SQL
- **Scalability:** Support 500 concurrent users
- **Performance:** P95 latency <12s
- **Availability:** 99.5%
- **Auditability:** Full query audit trail

---

### 6. Architecture Overview

See `ARCHITECTURE.md` for the full technical design and `ORCHESTRATOR.md` for the Phase 2.5 multi-agent orchestrator design (workflows, triggers, durable runs, approvals, budgets).

**Key Layers:**
- Frontend: Next.js 15 + TypeScript
- Backend: Next.js API Routes + TypeScript + Prisma
- Semantic Layer: Metric Registry (YAML in DB) + Knowledge Graph (RDF/TTL via N3.js)
- AI: Claude Sonnet 4.6 default, pluggable LLM gateway via Vercel AI SDK
- Data: Postgres (app DB + PGVector) + tenant data sources (Postgres, DuckDB, MySQL)

**Query Flow:**
1. User Message → Auth + Tenant Context
2. Domain Guard → block out-of-domain questions
3. Conversation Resolver → resolve follow-ups into standalone query
4. Router → Semantic path (Metric Registry + KG → SQL) or Agentic path (RAG + Text-to-SQL)
5. Validate → Execute → Stream LLM Response → Log

---

### 7. Technical Stack

| Layer | Technology |
|-------|------------|
| Frontend + Backend | Next.js 15 + TypeScript |
| ORM | Prisma |
| LLM SDK | Vercel AI SDK (`ai`) |
| Default LLM | Claude Sonnet 4.6 (pluggable) |
| Semantic Layer | Metric Registry (YAML in DB) |
| Knowledge Graph | RDF TTL/NT files via N3.js |
| Vector Store | PGVector |
| Tenant Data Sources | Postgres, DuckDB, MySQL |
| Deployment | Docker Compose (local dev) |

---

### 8. Phased Roadmap

See `ARCHITECTURE.md` Section 6 for the full phased roadmap.

**Phase 1 (6–8 weeks):** Auth, admin panel, data source management, metadata catalog, knowledge graph, metric registry, domain guard, conversation resolver, chat with streaming, semantic + agentic query paths, logging.

**Phase 2 (4–5 weeks):** User-facing agent builder, user-facing MCP connections, admin MCP governance (allowlist), shared agent management, auto LLM router, observability dashboard, SQL validation, RLS enforcement, additional LLM providers.

**Phase 2.5 — AI Agent Orchestrator (7 weeks):**
- 2.5a Foundation (3 weeks): Workflow + WorkflowVersion + AgentRun + RunCheckpoint, YAML parser/validator, Redis+BullMQ queue, worker pool, manual trigger, step types `agent`/`tool`/`branch`, read-only Run Inspector.
- 2.5b Triggers + HITL (2 weeks): Cron scheduler, webhook receiver, `approval` step + Approval Inbox, live budgets, ToolCallCache, retry policies.
- 2.5c Composition (2 weeks): `sub_workflow` and `parallel` step types, sub-agent-as-tool, run fork/replay, workflow YAML editor (Monaco + schema validation).

**Phase 3 (Ongoing):** SPARQL endpoint, visualization generation, agent marketplace (share agents across tenants), visual DAG editor, durable queue (NATS/Temporal) if scale demands, external event bus triggers, fine-tuned models, anomaly detection.

---

### 9. Assumptions & Risks

**Assumptions**
- Core business metrics can be defined in the Metric Registry.
- Admins/data stewards will maintain schema descriptions in the metadata catalog.

**Risks & Mitigations**
- LLM hallucination → Metric Registry + KG provide grounded context; validation before execution
- Metric description quality → Admin tooling + test-metric feature in admin panel
- Cost overruns → Domain Guard + semantic path priority; fast model for routing/guard
- Schema drift → Scheduled schema refresh + delta detection + KG re-sync
- User-connected MCP abuse → Admin allowlist controls which endpoints users can register; personal MCP scoped to user only
- Agent sprawl → Personal agents visible only to creator; shared agents require admin approval
- Runaway workflow cost → Per-run budget caps (tokens, USD, wall-clock, tool calls) enforced live; tenant-level daily ceiling
- Sub-agent recursion → Max depth cap (default 3); capability narrowing prevents privilege escalation through sub-calls
- Long-running runs orphaned → Stateless workers + Postgres-backed checkpoints; any worker can resume any run after restart
- Side-effect duplication on retry → Idempotency keys + ToolCallCache replay successful results instead of re-invoking tools
- Webhook abuse → Per-trigger HMAC tokens, rate-limited endpoints, allowlist of source IPs (configurable)

---

### 10. Out of Scope (Phase 1)
- Write / Update operations on tenant data
- Complex ML model training
- Voice input
- Mobile-native app
- External BI tool connectivity

