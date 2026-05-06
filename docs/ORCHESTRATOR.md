# AgentGarden — Orchestrator (Phase 2.5)

**Version:** 0.1 (draft)
**Date:** 2026-05-06
**Status:** Proposed — extends ARCHITECTURE.md after Phase 2

---

## 1. Why an Orchestrator

Phase 1/2 give every user a single agent that runs inside a chat turn: one LLM loop, max iterations, request-bound streaming. That covers "ask a question, get an answer" and "run one tool flow".

It does **not** cover:
- multi-agent collaboration (planner → executor → reviewer)
- workflows that span minutes/hours and survive disconnects
- triggers that aren't a chat message (cron, webhook, data event)
- human approval gates between steps
- per-run budgets and retries

Phase 2.5 adds a **durable, multi-agent execution layer** on top of the existing agent runtime. The chat-turn agent stays. Workflows are an additional way to invoke agents — declarative, async, observable.

---

## 2. Goals & Non-Goals

**Goals**
- Run multi-step workflows that compose multiple agents and tools.
- Survive process restarts, network drops, and long waits (hours).
- Trigger workflows from cron, webhooks, and chat — not only chat.
- Pause for human approval and resume cleanly.
- Enforce per-run budgets (tokens, dollars, wall-clock, tool calls).
- Replay any past run for debugging and evals.

**Non-Goals (Phase 2.5)**
- Visual no-code DAG editor — YAML/JSON only in Phase 2.5; visual editor deferred to Phase 3.
- Cross-tenant agent marketplace (already Phase 3 in PRD).
- Distributed execution across regions — single-region worker pool.
- Streaming partial workflow output to chat UI — runs are observed via the Run Inspector, not the chat stream.

---

## 3. Vocabulary

| Term | Definition |
|------|------------|
| **Agent** | Existing unit: system prompt + LLM + MCP tools + max iterations. Unchanged from Phase 2. |
| **Workflow** | Versioned, declarative DAG of steps. Tenant-scoped. |
| **Step** | One node in a workflow: an agent call, a tool call, a sub-workflow, an approval, or a branch. |
| **Run** | A single execution of a workflow. Has its own state, blackboard, budget, and trace. |
| **Trigger** | What starts a run: chat message, cron, webhook, data event, manual. |
| **Checkpoint** | Persisted run state at step boundaries; lets a worker resume after restart. |
| **Blackboard** | Per-run shared key/value store steps read and write. Typed by step output schema. |
| **Approval** | A run state where execution halts until a human acts (approve / edit / reject). |
| **Budget** | Hard limits per run: tokens, USD, wall-clock seconds, tool-call count. |

---

## 3.1 Architecture Diagram

```
                           ┌────────────────────────────────────────┐
                           │              TRIGGER SOURCES           │
                           │                                        │
                           │  Chat UI    Cron     Webhook    Event  │
                           │    │         │         │          │    │
                           └────┼─────────┼─────────┼──────────┼────┘
                                │         │         │          │
                                ▼         ▼         ▼          ▼
                           ┌────────────────────────────────────────┐
                           │      Next.js API  /api/triggers/*      │
                           │  ─ HMAC verify ─ tenant resolve ─ enq  │
                           └───────────────────┬────────────────────┘
                                               │
                                  enqueue AgentRun(state=queued)
                                               │
                                               ▼
              ┌─────────────────────────────────────────────────────────┐
              │                  Redis + BullMQ Queue                   │
              │   workflows.run    approvals.timeout    events.bus      │
              └────────────┬───────────────┬───────────────┬────────────┘
                           │               │               │
                           ▼               ▼               ▼
                   ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
                   │   Worker 1   │ │   Worker 2   │ │  Scheduler   │
                   │   (Node)     │ │   (Node)     │ │ (leader-lock)│
                   └──────┬───────┘ └──────┬───────┘ └──────┬───────┘
                          │                │                │
                          │ claim run      │                │ emit cron jobs
                          ▼                ▼                │
                   ┌──────────────────────────────────────┐ │
                   │           Run Loop (per worker)      │ │
                   │  ─ load WorkflowVersion              │ │
                   │  ─ pick next ready step              │ │
                   │  ─ check budget                      │ │
                   │  ─ execute step ───────┐             │ │
                   │  ─ checkpoint          │             │ │
                   │  ─ retry / branch      │             │ │
                   └────────────────────────┼─────────────┘ │
                                            │               │
              ┌─────────────────────────────┼───────────────┘
              │                             │
              ▼                             ▼
   ┌────────────────────┐        ┌──────────────────────┐
   │  Agent Runtime     │        │   Tool Executor      │
   │  (Vercel AI SDK    │        │   ─ MCP client       │
   │   generateText +   │        │   ─ idempotency cache│
   │   tool use loop)   │        │   ─ retry policy     │
   └─────────┬──────────┘        └──────────┬───────────┘
             │                              │
             ▼                              ▼
   ┌────────────────────┐        ┌──────────────────────┐
   │  LLM Gateway       │        │  MCP Servers         │
   │  Anthropic / OAI / │        │  (shared + personal) │
   │  Google / Groq     │        │   external + internal│
   └────────────────────┘        └──────────────────────┘

              ▲                             ▲
              │                             │
              └────────────┬────────────────┘
                           │ writes
                           ▼
   ┌─────────────────────────────────────────────────────────────────┐
   │                     PostgreSQL (App DB)                         │
   │                                                                 │
   │  Workflow / WorkflowVersion / Trigger                           │
   │  AgentRun / RunCheckpoint / ToolCallCache                       │
   │  ApprovalRequest / WorkflowEvent                                │
   │  MessageExecutionTrace (extended) / QueryLog / AuditLog         │
   │  PGVector (RAG) / Tenant / User / DataSource ...                │
   └────────────────────────────────┬────────────────────────────────┘
                                    │
                                    │ reads
                                    ▼
                       ┌──────────────────────────┐
                       │   Run Inspector UI       │
                       │  ─ live SSE tail         │
                       │  ─ span tree             │
                       │  ─ approve / fork / cancel│
                       └──────────────────────────┘
```

**Reading the diagram**
- The top half is the **ingress** path: every trigger funnels into one queue.
- Workers and the scheduler are the only components that mutate `AgentRun` state.
- The agent runtime and tool executor are the same primitives used by the chat-turn agent — workflows reuse them, they don't replace them.
- All durable state lives in Postgres; Redis only holds in-flight queue jobs and short-lived locks.

---

## 4. Workflow Definition

Workflows are stored as YAML in DB (versioned), edited via admin UI form (Phase 2.5) and a visual DAG editor (Phase 3). Authoring rules:

- A workflow has `id`, `name`, `version`, `triggers[]`, `inputs` (JSON Schema), `steps[]`, `output` (mapping from blackboard).
- Each step has a unique `id`, a `type`, and an `on` clause for branching/error routing.
- Step `type` values: `agent`, `tool`, `sub_workflow`, `approval`, `branch`, `parallel`, `loop`.
- Step inputs reference the blackboard via `${steps.<id>.output.<path>}` or `${input.<path>}`.
- Edges are implicit (sequential) unless `on` specifies `success | failure | timeout | rejected` targets.

Example:

```yaml
id: weekly_revenue_brief
name: Weekly Revenue Brief
version: 3
triggers:
  - type: cron
    schedule: "0 9 * * MON"
    timezone: Asia/Kolkata
inputs:
  type: object
  properties:
    region: { type: string, default: "ALL" }
budget:
  max_usd: 1.50
  max_tokens: 200000
  max_wall_seconds: 600
  max_tool_calls: 40
steps:
  - id: pull_metrics
    type: agent
    agent: SemanticQueryAgent
    input:
      question: "Weekly revenue by product for ${input.region}"
    on:
      failure: notify_failure

  - id: draft_brief
    type: agent
    agent: AnalystAgent
    input:
      data: ${steps.pull_metrics.output.rows}
      tone: executive

  - id: human_review
    type: approval
    approver_role: data_steward
    payload:
      draft: ${steps.draft_brief.output.markdown}
    on:
      rejected: draft_brief        # loop back with reviewer feedback
      timeout: notify_failure

  - id: send_email
    type: tool
    mcp_server: email_mcp
    tool: send_email
    input:
      to: leadership@tenant.com
      subject: "Weekly Revenue Brief"
      body: ${steps.human_review.output.approved_draft}

  - id: notify_failure
    type: tool
    mcp_server: slack_mcp
    tool: post_message
    input:
      channel: "#data-ops"
      text: "Weekly brief workflow failed: ${run.error}"

output:
  email_id: ${steps.send_email.output.message_id}
```

Validation at save time: schema check, DAG acyclicity, every referenced agent / MCP server / tool exists and is allowed for this tenant, blackboard references resolve.

---

## 5. Run State Machine

```
                  ┌──────────┐
       create ──▶ │ queued   │
                  └────┬─────┘
                       │  worker claims
                       ▼
                  ┌──────────┐    step error + retries left
              ┌──▶│ running  │──────────────┐
              │   └────┬─────┘              │
              │        │ step needs human   │
              │        ▼                    ▼
              │   ┌──────────┐         ┌──────────┐
              │   │ waiting  │         │ retrying │
              │   └────┬─────┘         └────┬─────┘
              │        │ approve / reject   │
              │        ▼                    │
              └────────┴────────────────────┘
                       │
                       ▼
              ┌──────────────────┐
              │ completed │ failed │ cancelled │ budget_exceeded
              └──────────────────┘
```

Transitions are recorded in `RunCheckpoint` with `from_state`, `to_state`, `step_id`, `reason`. Workers re-hydrate from the latest checkpoint on restart — never replay completed steps unless the run is explicitly forked.

Terminal states are immutable. To "rerun" a workflow with the same input, the operator forks a new run from a prior checkpoint or from scratch.

---

## 6. Trigger Types

| Trigger | Source | Latency | Notes |
|---------|--------|---------|-------|
| `chat` | User message in chat UI; router classifies "this is a workflow" intent. | < 1s | Run id streamed back to chat; user can keep chatting. |
| `cron` | Scheduler service; standard cron + timezone. | scheduled | Per-tenant cron table; misfires logged but not auto-replayed. |
| `webhook` | HTTPS POST to `/api/triggers/{token}`. | < 1s | HMAC-signed token per trigger; payload mapped to workflow `inputs`. |
| `event` | Internal pub/sub: `datasource.refreshed`, `flagged_answer.created`, `metric.changed`. | < 5s | In-process bus Phase 2.5; durable queue (NATS/Redis) Phase 3. |
| `manual` | Admin or user clicks "Run now" with input form. | immediate | Used for testing and ad-hoc execution. |

A workflow can declare multiple triggers; each fires an independent run.

---

## 7. Sub-Agent as Tool

A workflow's `agent` step exposes its result; an agent inside a workflow can also **delegate** to another agent by treating it as a tool.

- The orchestrator generates a synthetic MCP tool per agent in the tenant: `agent.<name>(input: string) → output: json`.
- Calls from one agent to another are recorded in the trace tree as parent → child runs.
- Depth is capped (`max_subagent_depth`, default 3) to prevent runaway recursion.
- Each sub-call has its own iteration cap and inherits the parent run's budget envelope.

This lets a Planner agent decompose work and dispatch to specialised agents (Sales, Ops, Finance) without the workflow author hand-wiring every branch.

---

## 8. Human-in-the-Loop

`approval` steps pause the run and create an `ApprovalRequest`:

- Routed to a role (`admin`, `data_steward`) or a specific user.
- Approver sees: workflow name, step payload, full trace up to this point.
- Actions: **approve**, **edit** (modify the payload, then approve), **reject** (with reason).
- On `reject`, the `on.rejected` edge targets a step (often the prior agent step) with the reason in the blackboard so the agent can revise and re-submit.
- Default timeout: 24h; configurable per step. Timeouts route via `on.timeout`.

Approval inbox lives in the admin panel; users with pending approvals also see a badge in the chat sidebar.

---

## 9. Reliability

**Retries.** Every step has a retry policy: `max_attempts`, `backoff` (exponential with jitter), `retry_on` (error codes / classes). Default for tool steps: 3 attempts, 2s/8s/30s. Default for agent steps: 1 attempt (LLM retries are inside the agent loop).

**Idempotency.** Every tool call sent to an MCP server includes an `idempotency_key = hash(run_id + step_id + attempt_input)`. The orchestrator caches successful results; on retry after partial failure, cached results are returned instead of re-invoking side-effectful tools.

**Dead-letter.** A run that exhausts retries on a non-recoverable step transitions to `failed` and is written to a `RunDLQ` view. Admin can inspect, edit input, and replay from the failed step.

**Budgets.** Tracked live as steps execute. On breach: run transitions to `budget_exceeded`, current step is cancelled, `on.failure` edges fire if defined.

**Cancellation.** A run can be cancelled by the initiator, an admin, or a parent run. Cancellation is cooperative — running tool calls finish or time out; no new steps start.

---

## 10. Observability

Every run produces a **trace tree**, an extension of the existing `MessageExecutionTrace`:

- Root span = run; children = steps; grandchildren = agent iterations / tool calls / sub-runs.
- Each span: start/end time, status, input hash, output hash, tokens, cost, error.
- The chat UI's collapsible "Steps" section is replaced by a link to the **Run Inspector** for workflow runs.

**Run Inspector** (admin panel):
- Live tail of an in-flight run (SSE stream).
- Span tree with expand/collapse, filterable by status and step type.
- Per-span input/output viewers (JSON pretty-printed; large payloads behind a "Load full" button).
- Replay: launch a new run from any checkpoint with edited input.

**Aggregations** in the observability dashboard:
- Workflow run counts, success rate, P50/P95 duration.
- Cost per workflow per day.
- Top failing steps by workflow.
- Approval queue depth and median time-to-approve.

---

## 11. Data Model (Prisma deltas)

```
Workflow
  id              uuid PK
  tenant_id       uuid FK
  name            string
  description     string?
  current_version int
  status          enum (draft, active, archived)
  created_by      uuid FK → User
  created_at      timestamp

WorkflowVersion
  id              uuid PK
  workflow_id     uuid FK
  version         int
  yaml            text
  parsed_json     json
  validation      json   -- result of save-time validation
  created_by      uuid FK
  created_at      timestamp
  unique (workflow_id, version)

Trigger
  id              uuid PK
  tenant_id       uuid FK
  workflow_id     uuid FK
  type            enum (chat, cron, webhook, event, manual)
  config          json   -- cron expr, webhook token, event name, etc.
  enabled         bool
  created_at      timestamp

AgentRun
  id              uuid PK
  tenant_id       uuid FK
  workflow_id     uuid FK?              -- null for ad-hoc agent runs
  workflow_version int?
  parent_run_id   uuid?                 -- for sub-agent / sub-workflow
  triggered_by    enum (chat, cron, webhook, event, manual, parent)
  trigger_id      uuid? FK              -- nullable for chat/parent
  initiator_user_id uuid? FK
  state           enum (queued, running, waiting_for_human, retrying,
                       completed, failed, cancelled, budget_exceeded)
  input_json      json
  output_json     json?
  error_json      json?
  budget_json     json   -- limits + live counters
  created_at      timestamp
  started_at      timestamp?
  ended_at        timestamp?

RunCheckpoint
  id              uuid PK
  run_id          uuid FK
  step_id         string
  from_state      string
  to_state        string
  blackboard      json   -- snapshot at this checkpoint
  reason          string?
  created_at      timestamp

ApprovalRequest
  id              uuid PK
  run_id          uuid FK
  step_id         string
  approver_role   enum?
  approver_user_id uuid? FK
  payload         json
  status          enum (pending, approved, rejected, expired)
  decision_by     uuid? FK
  decision_reason string?
  decision_payload json?  -- edited content on approve
  expires_at      timestamp
  created_at      timestamp
  decided_at      timestamp?

ToolCallCache
  id              uuid PK
  run_id          uuid FK
  idempotency_key string  unique
  result          json
  created_at      timestamp

WorkflowEvent
  id              uuid PK
  tenant_id       uuid FK
  type            string  -- e.g. datasource.refreshed
  payload         json
  emitted_at      timestamp
```

`AgentRun.parent_run_id` enables the run tree. `RunCheckpoint` is append-only and is the source of truth for resume.

---

## 12. Execution Engine

**Components**
- **API layer** (Next.js routes) — accepts triggers, enqueues runs, serves Run Inspector.
- **Queue** — Redis + BullMQ (Phase 2.5). Future: NATS JetStream or Temporal if durability needs grow.
- **Worker pool** — Node workers that claim runs, execute the run loop, persist checkpoints.
- **Scheduler** — single leader (Postgres advisory lock) that emits cron-triggered jobs into the queue.
- **Webhook receiver** — HTTPS endpoint, verifies HMAC, enqueues a run.

**Run loop (per worker)**
```
1. Claim AgentRun (state=queued) → set state=running, record checkpoint
2. Load WorkflowVersion + parsed DAG
3. While not terminal:
     a. Pick next ready step from DAG (respect dependencies + branches)
     b. Check budget — if breached, transition to budget_exceeded and break
     c. Execute step (agent / tool / sub_workflow / approval / ...)
        - For approval: write ApprovalRequest, set state=waiting_for_human, sleep
        - For agent: invoke existing agent runtime, stream tokens to trace
        - For tool: call MCP via cache, retry per policy
     d. On success: write step output to blackboard, checkpoint
     e. On error: apply retry / on.failure / on.timeout
4. Compute output mapping → set state=completed, checkpoint
5. Emit run.completed event (may trigger downstream workflows)
```

Workers are stateless; any worker can pick up any run. Resume on restart: read the latest `RunCheckpoint`, restore blackboard, continue at the next ready step.

---

## 13. API Surface

| Method | Path | Purpose |
|--------|------|---------|
| `POST` | `/api/workflows` | Create draft workflow (yaml). |
| `PUT` | `/api/workflows/:id` | Save new version. |
| `POST` | `/api/workflows/:id/activate` | Activate a version. |
| `POST` | `/api/workflows/:id/run` | Manual trigger; returns `run_id`. |
| `GET` | `/api/runs/:id` | Run summary + state. |
| `GET` | `/api/runs/:id/trace` | Span tree (paginated). |
| `GET` | `/api/runs/:id/stream` | SSE live tail. |
| `POST` | `/api/runs/:id/cancel` | Cancel. |
| `POST` | `/api/runs/:id/fork` | Replay from checkpoint with edited input. |
| `POST` | `/api/approvals/:id/decision` | approve / reject / edit. |
| `POST` | `/api/triggers/:token` | Webhook receiver (per-trigger token). |

All endpoints are tenant-scoped via session; webhook tokens are per-trigger, rotated on demand.

---

## 14. Security & Permissions

- **Workflow authoring** restricted to `admin` (Phase 2.5). `data_steward` can author in Phase 3.
- **Trigger creation** is admin-only. Webhook tokens are shown once and stored hashed.
- **MCP allowlist** from existing config applies to workflow tool steps too — a workflow cannot call a tool the tenant has not allowed.
- **Sub-agent calls** respect each agent's existing MCP scope; an agent cannot call tools through a sub-agent that it could not call directly (capability narrowing, never widening).
- **Approvals** check the approver's role at decision time, not at request creation, so role changes take effect immediately.
- **Run isolation** — every run carries `tenant_id`; workers refuse cross-tenant access; RLS enforced at DB layer (consistent with existing tenant model).
- **Secrets** in workflow YAML are referenced by name (`${secret.stripe_key}`), never inlined; resolved by the worker against a per-tenant secret store.

---

## 14.1 User Action Flow Examples

Four worked examples showing how a user (or external system) interacts with the orchestrator end to end. Each example shows the sequence of components, the run state transitions, and the persistence side effects.

---

### Example A — Chat-triggered multi-agent workflow

**Scenario:** Sales manager types in chat: "Plan an outreach campaign to our top 10 churning customers and draft personalised emails."

```
[User: chat]                       [Backend]                     [Worker]                [Persistence]

  types message  ─────────────▶  /api/chat
                                    │
                                    ├─ Domain Guard ──▶ in domain
                                    ├─ Resolver      ──▶ standalone query
                                    ├─ Router        ──▶ matches workflow
                                    │                    "outreach_campaign"
                                    │
                                    │  enqueue AgentRun
                                    │  (trigger=chat,
                                    │   input={top_n:10})
                                    │                          ┌─▶ AgentRun  (queued)
                                    │                          │
                                    └─ stream {run_id} to UI ──┘

  sees "Run #471 started"                                        Worker claims run
   (chat keeps streaming                                          │
    progress events via SSE)                                      ├─▶ AgentRun.state=running
                                                                  │   RunCheckpoint(start)
                                                                  │
                                                                  ├─ Step: pull_churn_list
                                                                  │    type: agent
                                                                  │    agent: SemanticQueryAgent
                                                                  │    └─ executes SQL via KG ──▶ rows
                                                                  │   blackboard.churn = rows
                                                                  │   RunCheckpoint(pull_churn_list)
                                                                  │
                                                                  ├─ Step: research_each (parallel × 10)
                                                                  │    type: agent
                                                                  │    agent: ResearchAgent
                                                                  │    └─ MCP: company_lookup, news_search
                                                                  │   blackboard.profiles[i] = ...
                                                                  │   RunCheckpoint(research_each)
                                                                  │
                                                                  ├─ Step: draft_emails
                                                                  │    type: agent
                                                                  │    agent: WriterAgent
                                                                  │   blackboard.drafts = [...]
                                                                  │   RunCheckpoint(draft_emails)
                                                                  │
                                                                  └─ AgentRun.state=completed
                                                                      RunCheckpoint(end)

  sees "Run #471 completed"  ◀─────────────  SSE: run.completed
  click → opens Run Inspector
  reviews 10 drafts
  clicks "Send all" (separate
    user action; not auto-sent)
```

State path: `queued → running → completed`. No approval gate in this example because drafts are returned for the user to send manually. If the workflow author wants automatic send, they add an `approval` step before a `send_email` tool step (see Example C).

---

### Example B — Cron-triggered weekly report (no user in the loop)

**Scenario:** Admin configured `weekly_revenue_brief` to run every Monday 09:00 IST and email the result. No human triggers it; the user only sees results in their inbox.

```
[Scheduler]                  [Queue]               [Worker]                    [External]

  Mon 09:00 IST
  acquires leader lock
  finds enabled cron triggers
    │
    ├─ enqueue AgentRun ─────▶  workflows.run
    │  trigger_id=T_42
    │  workflow=weekly_revenue_brief v3
    │
    │                                              claim run
    │                                                │
    │                                                ├─ Step: pull_metrics    (agent)
    │                                                │   ─ Semantic path SQL
    │                                                │
    │                                                ├─ Step: draft_brief     (agent)
    │                                                │   ─ AnalystAgent → markdown
    │                                                │
    │                                                ├─ Step: human_review    (approval)
    │                                                │   creates ApprovalRequest
    │                                                │   AgentRun.state=waiting_for_human
    │                                                │   worker releases run
    │                                                ▼
    │                                       [stops here until approver acts]

  ── 2h later, data steward opens Approval Inbox ──

  approver clicks Approve  ──▶  /api/approvals/A_99/decision
                                    │
                                    ├─ ApprovalRequest.status=approved
                                    └─ enqueue AgentRun resume ──▶ workflows.run

                                                    re-claim run
                                                      │
                                                      ├─ Step: send_email     (tool)
                                                      │   MCP email_mcp.send_email
                                                      │   idempotency_key set
                                                      │                         ─────▶ SMTP send
                                                      │                                   │
                                                      │   ToolCallCache.write   ◀────────┘
                                                      │
                                                      └─ AgentRun.state=completed

  Inbox at leadership@tenant.com:
    "Weekly Revenue Brief — Week of May 6"
```

State path: `queued → running → waiting_for_human → running → completed`. The approval pause survives any worker restart — the run is persisted as `waiting_for_human` and only re-enters the queue when the approval decision endpoint enqueues a resume job.

---

### Example C — Webhook-triggered with HITL approval

**Scenario:** External CRM POSTs a webhook when a high-value deal stalls. Workflow drafts an executive escalation message; an admin must approve before it sends.

```
[CRM]                       [Webhook receiver]                 [Worker]                   [Approver]

  POST /api/triggers/wh_a8c2
       X-Signature: ...
       body: {deal_id:123, stalled_days:21}
        │
        ▼
  HMAC verify ─ ok
  tenant resolved from token
  workflow=deal_escalation v1
  enqueue AgentRun
  return 202 Accepted ─────▶ {run_id: 882}                 claim run
                                                             │
                                                             ├─ Step: enrich_deal     (agent)
                                                             │   ─ MCP crm_lookup
                                                             │
                                                             ├─ Step: draft_message   (agent)
                                                             │
                                                             ├─ Step: approve_send    (approval)
                                                             │   ApprovalRequest A_551
                                                             │   role=admin
                                                             │   timeout=4h           ────▶ admin notified
                                                             │   AgentRun.state=waiting_for_human

                                                                                       admin opens inbox
                                                                                          │
                                                                                          ├─ EDIT message
                                                                                          ├─ click Approve
                                                                                          ▼
                                                                  /api/approvals/A_551/decision
                                                                  status=approved
                                                                  payload={message:"<edited>"}
                                                             ◀────  enqueue resume

                                                             re-claim run
                                                             ├─ Step: send_slack_dm   (tool)
                                                             │   text=blackboard.approve_send.approved_payload
                                                             │
                                                             └─ AgentRun.state=completed

  ── timeout branch (alternate path) ──
  if no decision in 4h:
    ApprovalRequest.status=expired
    on.timeout edge → notify_pm step
    AgentRun.state=completed (different output)
```

State path on approval: `queued → running → waiting_for_human → running → completed`. Edited payload is captured in `ApprovalRequest.decision_payload` and exposed on the blackboard so the next step uses the human-edited version.

---

### Example D — Agent calling a sub-agent

**Scenario:** A Planner agent receives a complex query, decides to delegate part of it to a Sales agent and another part to a Finance agent, then synthesises the answer. Triggered from chat; runs as a single workflow with sub-runs.

```
                ┌─────────────────────────────────────────┐
                │   AgentRun #1001  (workflow=planner)    │
                │   state: running                        │
                │                                         │
                │   PlannerAgent loop                     │
                │     │                                   │
                │     ├─ tool call: agent.SalesAgent      │──┐
                │     │                                   │  │ spawns
                │     │                                   │  │ child run
                │     │                                   │  ▼
                │     │                          ┌──────────────────────────────┐
                │     │                          │  AgentRun #1002              │
                │     │                          │  parent_run_id=1001          │
                │     │                          │  triggered_by=parent         │
                │     │                          │  agent=SalesAgent            │
                │     │                          │  ─ MCP crm_lookup            │
                │     │                          │  ─ Semantic SQL: top deals   │
                │     │                          │  state: completed            │
                │     │                          └──────────────────────────────┘
                │     │     ◀── output JSON ────────┘
                │     │
                │     ├─ tool call: agent.FinanceAgent    │──┐
                │     │                                   │  ▼
                │     │                          ┌──────────────────────────────┐
                │     │                          │  AgentRun #1003              │
                │     │                          │  parent_run_id=1001          │
                │     │                          │  agent=FinanceAgent          │
                │     │                          │  ─ MCP gl_lookup             │
                │     │                          │  state: completed            │
                │     │                          └──────────────────────────────┘
                │     │     ◀── output JSON ────────┘
                │     │
                │     └─ synthesises final answer         │
                │                                         │
                │   AgentRun.state=completed              │
                └─────────────────────────────────────────┘
```

Trace tree in the Run Inspector:
```
Run #1001  PlannerAgent           ✓ 8.2s    $0.04
  ├─ tool: agent.SalesAgent       ✓ 3.1s
  │    └─ Run #1002  SalesAgent   ✓ 3.0s    $0.01
  │         ├─ tool: crm_lookup   ✓
  │         └─ sql: top_deals     ✓
  ├─ tool: agent.FinanceAgent     ✓ 2.6s
  │    └─ Run #1003  FinanceAgent ✓ 2.5s    $0.01
  │         └─ tool: gl_lookup    ✓
  └─ synthesise                   ✓ 1.4s
```

Capability rules in effect:
- `PlannerAgent` can only call `agent.SalesAgent` and `agent.FinanceAgent` if its `AgentMcpServer` config grants the synthetic agent-tool entry for each.
- Sub-agents inherit the parent run's budget envelope; if Sales agent uses $0.40 of a $0.50 cap, only $0.10 is left for Finance.
- `max_subagent_depth` defaults to 3 — a sub-agent calling another sub-agent is allowed; the fourth level is rejected.

---

## 15. Phased Rollout

**Phase 2.5a — Foundation (3 weeks)**
- Workflow + WorkflowVersion + AgentRun + RunCheckpoint tables.
- YAML parser + validator.
- Worker pool + Redis queue + manual trigger only.
- Step types: `agent`, `tool`, `branch`.
- Run Inspector (read-only span tree).

**Phase 2.5b — Triggers + HITL (2 weeks)**
- Cron scheduler + webhook receiver.
- `approval` step + Approval Inbox UI.
- Budgets enforced live.
- ToolCallCache + retry policies.

**Phase 2.5c — Composition (2 weeks)**
- `sub_workflow` and `parallel` step types.
- Sub-agent-as-tool.
- Run fork / replay from checkpoint.
- Workflow YAML editor in admin panel (Monaco + schema validation).

**Phase 3 (later)**
- Visual DAG editor.
- Cross-tenant workflow templates / marketplace.
- Durable queue (NATS / Temporal) if scale demands.
- `event` triggers backed by external event bus.

