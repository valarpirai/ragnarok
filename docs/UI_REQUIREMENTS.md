# UI Requirements

Desktop only. Covers the chat interface (all users), the admin panel (admin and data steward roles), and the Orchestrator surfaces (Phase 2.5 — Workflow Builder, Run Inspector, Triggers, Approvals).

---

## Chat Interface

### Results Display

Every query response shows three components together:

1. **Text summary** — LLM-generated narrative highlighting key insights
2. **Data table** — paginated preview of raw results
3. **Chart** — auto-selected chart type based on data shape

**Chart interaction**
- System selects the default chart type (line, bar, pie, etc.) based on data
- User can override via natural language ("show this as a bar chart") or UI toggle in the chart panel

**Large result sets**
- Chat shows a paginated preview (e.g. first 50 rows)
- Download full result as CSV always available

**Export**
- Export result (chart + table + summary) as PDF or image

---

### Execution Trace (Collapsible)

Every response includes a collapsible **"Steps"** section. Collapsed by default.

```
▶ Steps                          ← collapsed by default

  ✓ Domain Guard       — In domain
  ✓ Conversation Resolver  — Resolved follow-up: "Show monthly revenue by region"
  ✓ Router             — Routed to: Agent Path → Sales Agent
  ✓ Agent: Sales Agent
      Step 1  Tool: get_top_customers   params: { limit: 10 }
                                        result: 10 records returned
      Step 2  Tool: send_summary_email  params: { to: "team@..." }
                                        result: Email sent
```

For the Semantic path:
```
▶ Steps

  ✓ Domain Guard       — In domain
  ✓ Router             — Routed to: Semantic Path
  ✓ Metric             — revenue: SUM(net_amount) FROM orders
  ✓ SQL                — [expandable SQL block]
  ✓ Executed on        — sales_db (Postgres)
```

Rules:
- Each message stores its own trace — persisted in DB
- Shown on every response, including errors
- MCP tool inputs and outputs shown (output truncated to summary if large)
- SQL block is syntax-highlighted and expandable
- Agent name and each tool call shown with step number

---

### Error Handling

- When a question is out of domain or data is unavailable: show "I can't answer that"
- No explanation, no suggested alternatives

---

### Conversation

- Full conversation history visible in sidebar
- Click any past conversation to resume from where it left off
- Follow-up questions resolved in context automatically (see ARCHITECTURE.md — Conversation Resolver)
- User can start a new conversation at any time

---

### Hints

- Per-tenant example questions shown in the chat input area
- Admin configures hints in the admin panel

---

### Loading & Abort

Step-by-step progress indicator during query execution:
- e.g. "Checking domain..." → "Resolving query..." → "Running query..." → "Generating response..."
- User can abort at any point during processing

---

### Flagging & Feedback

- Every response has a thumbs down (flag) button
- On flag: optional text field to type the correct answer or describe the issue
- Flagged responses logged for admin review

---

### Saved Queries

- Users can pin/save frequently used queries
- Saved queries appear as quick-launch shortcuts in the UI
- No scheduling — manually triggered only

---

### Multi Data Source

- Users do not select which data source to query
- System auto-routes using the Knowledge Graph and admin-provided descriptions — transparent to the user

---

### Personal Agent Builder

Any user (not just admins) can create personal agents scoped to their account.

- Sidebar section **"My Agents"** lists personal agents alongside shared agents
- Create agent form: name, description, system prompt, LLM model, max iterations
- Add MCP servers from the user's personal connections or tenant-shared servers
- Select specific tools per MCP server
- Test agent inline before saving
- Personal agents are auto-selectable by the router or invokable by name in chat
- Edit / deactivate at any time — changes take effect immediately

---

### Workflow Runs from Chat (Phase 2.5)

When the Query Router classifies a chat message as matching a workflow trigger, the chat thread shows a **Run Card** instead of a plain answer:

```
┌───────────────────────────────────────────────────────┐
│  ▶ Workflow: Outreach Campaign      Run #471          │
│  state: running                                       │
│                                                       │
│  ✓ pull_churn_list   (1.8s)                           │
│  ✓ research_each     (parallel × 10, 12.4s)           │
│  ▸ draft_emails      ...                              │
│                                                       │
│  [ Open Run Inspector ]   [ Cancel ]                  │
└───────────────────────────────────────────────────────┘
```

- Steps populate live via SSE.
- "Open Run Inspector" deep-links to the full span tree (admin panel surface).
- "Cancel" available while run is `running` or `waiting_for_human`.
- On terminal state (`completed` / `failed` / `cancelled` / `budget_exceeded`), card collapses to a one-line summary with a link to the inspector.
- User can keep chatting during a run; messages are independent of the run's lifecycle.

### Approval Badge (Phase 2.5)

If the user is an approver on any pending `ApprovalRequest`:

- Sidebar shows a badge with the count.
- Click opens the Approval Inbox (admin panel surface) filtered to that user's pending items.

---

### Personal MCP Connections

Users can register their own MCP servers, visible only to themselves.

- Settings panel: **"My MCP Servers"**
- Add server: name, endpoint URL, auth config
- Test / ping before saving — descriptive error on failure
- Enable / disable per server
- Connected tools available when building personal agents
- Endpoint must match the tenant's admin-configured allowlist (blocked registrations show a clear error)

---

## Admin Panel

Admin panel capabilities for tenant administrators. Admins manage their own tenant only — no cross-tenant access.

---

### Tenant Setup

- Tenant self-registers to create an account
- First admin user is created as part of registration flow
- Development: tenant + seed data provisioned via setup script
- Invite-only user model — admin invites users by email and assigns role on invite

---

### Domain Configuration

AI-assisted setup flow:

```
Admin enters business context (free text)
    │
    ▼
System suggests topic list based on connected data sources + description
    │
    ▼
Admin reviews, adds/removes/refines topics
    │
    ▼
System generates refined domain description
    │
    ▼
Admin approves and saves
```

Config stored per tenant:
- Business context (free text)
- Allowed topic list
- Out-of-domain refusal message shown to users

---

### Data Source Management

- Add new data source: dynamic form per type (Postgres, DuckDB, MySQL)
- Test connection before saving — descriptive error shown on failure (e.g. "Authentication failed", "Host unreachable")
- All admin actions captured in audit log
- Schema introspection on connect → Knowledge Graph generated
- Manual refresh + scheduled refresh (see METADATA_CATALOG.md)
- Each refresh run stores delta metrics: resources added, removed, marked inactive
- Refresh history visible in admin panel

---

### Metric Registry

- Manual form: name, description, SQL expression, base table, default filters
- AI-assisted: system suggests metrics based on connected schema + column descriptions
- Test metric: run against data source with a sample query before activating
- Edit metric: takes effect immediately on future queries, no approval step
- Deactivate metric: soft delete — excluded from LLM context when inactive
- Metric usage analytics: frequency of use per metric, visible in admin panel

---

### Metadata Catalog

- Browse full resource hierarchy: Database → Schema → Table/View/Function/Sequence → Column
- Add/edit descriptions on any resource (admin and data steward)
- Inactive resources shown with deleted date
- Refresh history: per-run delta summary (added/removed/inactive counts)
- Schedule refresh: set cron expression per data source

---

### User Management

- Invite users by email — admin assigns role on invite (admin / data_steward / viewer)
- Revoke access / change role at any time
- View all active users in tenant

---

### LLM Provider Configuration

- Register LLM providers via dynamic form (fields vary per provider)
- Providers: Anthropic, OpenAI, Google, Groq, Mistral (extensible)
- Set default model per tenant
- Configure auto-router preference (cost vs speed vs quality)
- API keys stored encrypted

---

### MCP Server Management

- Register shared MCP server: name, endpoint URL, auth config
- Test / ping button to verify connectivity before making live
- Enable / disable per server
- Registered MCP tools exposed to agents at query time
- **MCP allowlist** — configure permitted endpoint patterns (e.g. `https://mcp.internal/*`); users cannot register personal MCP servers outside this list

---

### Agent Management

Admin defines **shared agents** available to all users in the tenant. Users create personal agents via the chat interface (see Personal Agent Builder above).

**Agent configuration:**
- Name + description (used by router for auto-selection)
- System prompt (defines agent purpose and behavior)
- Allowed MCP servers + specific tools from each server
- LLM model (select from registered providers)
- Max iterations (prevents infinite loops)
- Status (active / inactive)

**Agent invocation:**
- Auto-selected by Query Router based on query intent + agent descriptions
- OR user explicitly names the agent in their message (e.g. "use the Sales Agent to...")

**Agent execution:**
- Receives query + current conversation context (fresh start each conversation)
- Plans steps using LLM
- Calls MCP tools as needed (any action the tool supports — read, write, external API)
- If a tool requires user input → agent asks follow-up question, waits for response, then continues
- Returns final result streamed to user

**MCP tool scope:**
- Agent can only call tools from its configured MCP servers
- Write operations permitted if the MCP tool supports them
- Tool authorization is defined at MCP server registration level

---

### Workflow Builder (Phase 2.5)

Admin authors workflows that compose agents and tools into multi-step automations. Phase 2.5 ships YAML-only authoring; visual DAG editor is Phase 3.

**List view**
- Table: name, current version, status (draft / active / archived), trigger types, last run, success rate
- Filter by status and trigger type
- "New workflow" button → blank YAML template
- "Import" accepts a YAML file

**Editor view**
- Monaco editor with YAML schema validation (live error markers)
- Right pane: preview of parsed DAG (steps + edges) — read-only block diagram
- "Validate" button: runs save-time validation (DAG acyclicity, capability check, blackboard refs, MCP allowlist)
- "Save as new version" creates a new `WorkflowVersion`; older versions remain queryable
- "Activate version" promotes a version to be the default for new runs; in-flight runs keep their version
- "Run now" opens the manual trigger panel (input form generated from `inputs` JSON Schema)

**Version history**
- List of versions with diff view (YAML diff)
- Per-version stats: runs, success rate, average cost
- Rollback: activate any earlier version

---

### Triggers (Phase 2.5)

Per-workflow triggers panel:

- **Chat** — toggle on/off; router uses workflow name + description to classify
- **Cron** — cron expression + timezone picker (defaults to tenant timezone); next 3 fire times shown as preview
- **Webhook** — generates a one-time URL + HMAC token (shown once, then hashed); rotate / revoke buttons; recent receipts log
- **Event** — pick from internal event types (`datasource.refreshed`, `flagged_answer.created`, `metric.changed`, ...)
- **Manual** — always available; not configurable

Per-trigger toggle (enabled / disabled). Disabling a trigger does not affect in-flight runs.

---

### Run Inspector (Phase 2.5)

Single most important orchestrator surface. Two modes: **list** and **detail**.

**List**
```
Run #  Workflow              Trigger    State            Duration   Cost     Started
─────────────────────────────────────────────────────────────────────────────────────
882    Deal Escalation       webhook    waiting_for_human  ─        $0.02   2m ago
471    Outreach Campaign     chat       completed         24.1s     $0.18   18m ago
470    Weekly Revenue Brief  cron       failed            8.2s      $0.04   2h ago
```
- Filters: workflow, trigger type, state, date range
- Search: by run id, initiator, error class
- Bulk actions: cancel selected (only pending/running)

**Detail**
```
Run #471   Outreach Campaign   v3
state: completed     duration: 24.1s    cost: $0.18    initiator: @sales_lead

Span tree                                              ┌─ Live tail (SSE) ──┐
─────────────────────────────────────                  │ [stream of events] │
✓ pull_churn_list           1.8s   $0.01               └────────────────────┘
✓ research_each (parallel × 10)  12.4s  $0.11
  ├─ profile[1]             1.2s
  ├─ profile[2]             1.4s
  └─ ... (8 more)
✓ draft_emails              9.6s   $0.06
  └─ tool: agent.WriterAgent
       └─ Run #1024 (sub-run, expand)

[ Fork from checkpoint ▼ ]  [ Cancel ]  [ Re-run with same input ]
```

- Span click: opens side panel with input JSON, output JSON, error (if any), tokens, cost, span timestamps
- Large payloads: shown truncated with "Load full" button
- "Fork from checkpoint" lets admin pick a step, edit input, and start a new run from that point — original run stays terminal
- Live tail visible only while run is non-terminal

---

### Approval Inbox (Phase 2.5)

Lists `ApprovalRequest`s where the current user is the approver (by role or by direct assignment).

**List**
- Columns: workflow, run id, step id, requested at, expires in, requester
- Filters: pending / approved / rejected / expired

**Detail**
- Workflow name + run trace up to this point (read-only span tree)
- Payload viewer (JSON or rendered preview, e.g. markdown drafts rendered)
- Action buttons:
  - **Approve** — pass payload through unchanged
  - **Edit and Approve** — opens an inline editor on the payload, then approves with edited content
  - **Reject** — required reason text field; routes via `on.rejected` edge
- All decisions recorded with actor + timestamp + diff (when edited)

---

### Run DLQ (Phase 2.5)

Failed runs (state = `failed` or `budget_exceeded`) listed for admin triage:

- Same shape as Run Inspector list, filtered to failures
- Per-run: failure reason, failing step, last attempt error
- Action: "Replay from failed step" with optional input edit; original run stays terminal, a new fork is created

---

### Flagged Answers Review

Admin can review all user-flagged responses with full context:
- Original user question
- Full execution trace (domain guard, resolver, router decision, agent steps, MCP tool calls, SQL)
- Result returned
- User's correction / feedback text
- Timestamp + user (anonymised if needed)

Used for improving metric definitions, domain config, and future model fine-tuning.

---

### Test as User Mode

- Admin can switch to "test as user" mode in the admin panel
- Simulates the full query flow (Domain Guard → Conversation Resolver → Router → Execute)
- Useful for verifying data sources, metrics, and domain config before going live

---

### Observability Dashboard

Tenant-scoped — admin sees only their own tenant's data:

- LLM usage: token counts, cost, model breakdown over time
- Query volume: total queries, path distribution (semantic vs agentic)
- Latency: P50/P95 response times
- Metric usage: most/least used metrics
- Domain guard: blocked query count + reasons
- Flagged answers: count + status (reviewed / unreviewed)
- Agent execution traces: tool calls, retries, errors
- **Workflows (Phase 2.5):** runs per workflow, success rate, P50/P95 duration, cost per workflow per day, top failing steps
- **Approvals (Phase 2.5):** queue depth, median time-to-approve, expiry rate
- **Budgets (Phase 2.5):** breach count per workflow, budget headroom heatmap
- **Sub-agent depth (Phase 2.5):** distribution of run trees, max-depth-cap rejections

---

### Audit Log

All admin actions are logged:
- Data source add / edit / delete / refresh
- Metric create / edit / deactivate
- User invite / revoke / role change
- LLM provider add / edit
- MCP server add / edit / disable
- Domain config update
- Workflow create / save version / activate / archive (Phase 2.5)
- Trigger enable / disable / token rotate (Phase 2.5)
- Run cancel / fork / replay (Phase 2.5)
- Approval decision (approve / edit-approve / reject) (Phase 2.5)

Each log entry: action, actor (user id), timestamp, before/after values where applicable.

---

## Data Model Additions

```
TenantInvite
  id            uuid PK
  tenant_id     uuid FK
  email         string
  role          enum (admin, data_steward, viewer)
  invited_by    uuid FK → User
  token         string (unique invite link token)
  accepted_at   timestamp?
  expires_at    timestamp
  created_at    timestamp

FlaggedAnswer
  id                    uuid PK
  tenant_id             uuid FK
  user_id               uuid FK
  conversation_message_id uuid FK → ConversationMessage
  execution_trace_id    uuid FK → MessageExecutionTrace
  original_question     string
  result_preview        json
  user_correction       string?
  reviewed_at           timestamp?
  reviewed_by           uuid? FK → User
  created_at            timestamp

AuditLog
  id            uuid PK
  tenant_id     uuid FK
  actor_id      uuid FK → User
  action        string  (e.g. "datasource.create", "metric.edit")
  resource_type string
  resource_id   uuid
  before_json   json?
  after_json    json?
  created_at    timestamp
```

---

## Platform

- Desktop browser only
- No mobile or tablet support

## Out of Scope

- Sharing conversations or results with colleagues
- Mobile / tablet UI
- Visual drag-and-drop DAG editor (Phase 3)
- Cross-tenant workflow / agent marketplace (Phase 3)

Note: scheduled / recurring runs *are* in scope from Phase 2.5 via cron triggers (see Triggers above). Ad-hoc scheduled queries (without a workflow wrapper) remain out of scope.
