# CLAUDE.md — Ragnarok / AgentGarden

Project guidance for Claude Code working in this repo.

---

## What this repo is

Design documentation for **AgentGarden**: a multi-tenant AI agent platform with a governed data layer (Metric Registry + Knowledge Graph) and a Phase 2.5 multi-agent orchestrator (workflows, triggers, durable runs, HITL approvals).

No implementation yet — the repo currently holds Markdown specs only. When you propose code changes, mention that an implementation phase has not begun.

---

## Document index

Read the relevant doc(s) before answering or editing. All paths are relative to repo root.

| Topic | Doc |
|-------|-----|
| Product scope, personas, functional requirements, KPIs, roadmap, risks | `docs/PRD.md` |
| System components, query flow, Prisma data model, tech stack, deployment | `docs/ARCHITECTURE.md` |
| Phase 2.5 orchestrator design — workflow YAML, run state machine, triggers, sub-agent-as-tool, architecture diagram, worked user-flow examples | `docs/ORCHESTRATOR.md` |
| Chat UI, admin panel, workflow builder, run inspector, approval inbox specs | `docs/UI_REQUIREMENTS.md` |
| Schema introspection, RDF/TTL knowledge graph, refresh + delta detection, descriptions | `docs/METADATA_CATALOG.md` |
| Per-tenant metric YAML schema, AI-assisted suggestion, test-metric flow | `docs/METRIC_REGISTRY.md` |

When a question spans multiple docs, prefer this order: **PRD → ARCHITECTURE → ORCHESTRATOR → UI_REQUIREMENTS → METADATA_CATALOG → METRIC_REGISTRY**. PRD is the source of truth for *what* and *why*; the others detail *how*.

---

## Cross-doc conventions

- **Phases:** Phase 1 (foundation), Phase 2 (production-ready), Phase 2.5 (orchestrator), Phase 3 (scale). When you add a new feature, place it in a phase and update PRD §8 + ARCHITECTURE §6 + (if user-visible) UI_REQUIREMENTS together.
- **Tenant scoping:** every entity is tenant-scoped via `tenant_id` with Postgres RLS. Don't propose designs that bypass this.
- **Data source isolation:** tenant data sources are *read-only* in Phase 1; write operations come in via MCP tools in Phase 2+.
- **Agent runtime:** single agent per chat turn (Phase 2). Multi-agent composition is done via the orchestrator (Phase 2.5), not by extending the chat-turn agent.
- **Capability narrowing:** an agent calling another agent (sub-agent-as-tool) cannot reach tools the caller could not call directly.
- **Idempotency:** every orchestrator tool call carries `hash(run_id + step_id + attempt_input)`; retries replay cached results from `ToolCallCache`.

---

## Editing conventions

- Doc edits should preserve the existing section structure and version/date headers. Bump the version on material changes.
- Cross-link new sections from the table of contents in `README.md` and the index above when adding a new top-level doc.
- Avoid HTML inside Markdown comments (per global CLAUDE.md).
- Build large files incrementally — write a skeleton first, then fill sections via Edit (per global CLAUDE.md).
- Keep prose tight; this is a design repo, not a marketing site.

---

## Git conventions

- Default branch: `main`.
- Remote: `https://github.com/valarpirai/ragnarok.git` (`origin`).
- Commit messages use a `docs:` / `feat:` / `fix:` prefix and a concise imperative summary on the first line, followed by a blank line and a paragraph that explains the *why*.
- Co-authorship line included automatically when committing through Claude Code.
- Don't force-push `main`. Don't amend pushed commits.

---

## When asked to "update docs"

If the user says "update the docs" without specifying which:
1. Determine which docs are affected by the change (use the topic table above).
2. Update them as a coordinated set so they stay consistent.
3. Surface any cross-doc inconsistency you find while reading — don't silently fix unrelated drift; ask first.
