# Ragnarok — AgentGarden

**Build, connect, and run AI agents on your business data.**

AgentGarden is a multi-tenant platform where every user gets a personal AI agent workspace: query data in natural language, automate multi-step workflows via MCP tools and other agents, and build agents that understand your business — without writing SQL or code.

The foundation is a governed data layer — **Metric Registry** (business vocabulary) + **Knowledge Graph** (schema relationships) — that makes data queries accurate and trustworthy. On top of that, Phase 2.5 adds an **AI Agent Orchestrator**: durable multi-agent runs, cron / webhook / event triggers, human-in-the-loop approvals, and per-run budget controls.

---

## Status

| Phase | Scope | State |
|-------|-------|-------|
| Phase 1 | Auth, admin panel, data sources, knowledge graph, metric registry, domain guard, conversation resolver, semantic + agentic query paths | Design approved |
| Phase 2 | User-facing agent builder, personal MCP, observability dashboard, additional LLM providers | Design approved |
| Phase 2.5 | AI Agent Orchestrator — workflows, triggers, durable runs, approvals, budgets, sub-agent delegation | Design drafted (see `docs/ORCHESTRATOR.md`) |
| Phase 3 | SPARQL endpoint, visual DAG editor, marketplace, durable queue, fine-tuned models | Backlog |

This repository currently contains the design documentation. Implementation has not started.

---

## Documentation

| Document | What it covers |
|----------|----------------|
| [`docs/PRD.md`](docs/PRD.md) | Product Requirements — personas, functional requirements, KPIs, phased roadmap, risks |
| [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) | System architecture — components, query flow, data model, tech stack, deployment |
| [`docs/ORCHESTRATOR.md`](docs/ORCHESTRATOR.md) | Phase 2.5 multi-agent orchestrator — workflow YAML, run state machine, triggers, HITL approvals, sub-agent-as-tool, architecture diagram, worked examples |
| [`docs/UI_REQUIREMENTS.md`](docs/UI_REQUIREMENTS.md) | UI specification for chat, admin panel, workflow builder, run inspector, approval inbox |
| [`docs/METADATA_CATALOG.md`](docs/METADATA_CATALOG.md) | Schema introspection, knowledge graph (RDF/TTL), metadata refresh, descriptions |
| [`docs/METRIC_REGISTRY.md`](docs/METRIC_REGISTRY.md) | Per-tenant metric definitions, YAML schema, AI-assisted suggestion, testing |

---

## Tech Stack

- **Frontend + Backend:** Next.js 15 + TypeScript
- **ORM:** Prisma
- **App Database:** PostgreSQL 16 + PGVector
- **LLM SDK:** Vercel AI SDK (`ai`)
- **Default LLM:** Claude Sonnet 4.6 (pluggable: Anthropic, OpenAI, Google, Groq, Mistral)
- **Knowledge Graph:** RDF TTL via N3.js
- **Tenant Data Sources:** Postgres, DuckDB, MySQL
- **Orchestrator (Phase 2.5):** Redis + BullMQ queue, Node worker pool, Postgres-backed run state
- **Deployment:** Docker Compose (local dev)

---

## License

[MIT](LICENSE)
