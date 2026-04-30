**Product Requirements Document (PRD)**

**Product Name:**  
**DataChat AI** – Natural Language Data Intelligence Agent

**Version:** 1.1 (updated after clarification sessions)
**Date:** April 30, 2026
**Status:** Approved — see ARCHITECTURE.md for current technical decisions

---

### 1. Executive Summary

**DataChat AI** is a multi-tenant, intelligent conversational interface that allows internal users to query their business data (Postgres, DuckDB, MySQL) using natural language.

It combines a **Metric Registry** (governed business vocabulary) with a **Knowledge Graph** (metadata relationships) and an **agentic fallback** to deliver accurate, secure, and fast answers with natural language explanations.

**Core Value Proposition:**  
Turn “Can I get the monthly sales trend for electronics in Tamil Nadu?” into instant, trustworthy insights — without writing SQL or needing a data analyst.

---

### 2. Problem Statement

- Business users waste hours waiting for reports or learning SQL.
- Direct Text-to-SQL has inconsistent accuracy and security risks.
- No single source of truth for business metrics (“revenue”, “active customer”, etc.).
- Different databases (Postgres vs DuckDB) require different query strategies.
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
- Register MCP servers with test/ping button
- Define agents: tools, MCP servers, LLM model

#### 5.2 Non-Functional Requirements

- **Security:** SOC2 / GDPR compliant, no raw user input in SQL
- **Scalability:** Support 500 concurrent users
- **Performance:** P95 latency <12s
- **Availability:** 99.5%
- **Auditability:** Full query audit trail

---

### 6. Architecture Overview

See `ARCHITECTURE.md` for the full technical design.

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

**Phase 2 (4–5 weeks):** MCP servers, agent builder, auto LLM router, observability dashboard, SQL validation, RLS enforcement, additional LLM providers.

**Phase 3 (Ongoing):** SPARQL endpoint, visualization generation, fine-tuned models, anomaly detection.

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

---

### 10. Out of Scope (Phase 1)
- Write / Update operations on tenant data
- Complex ML model training
- Voice input
- Mobile-native app
- External BI tool connectivity

