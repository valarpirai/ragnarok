# Admin Requirements

Admin panel capabilities for tenant administrators. Admins manage their own tenant only — no cross-tenant access.

---

## Tenant Setup

- Tenant self-registers to create an account
- First admin user is created as part of registration flow
- Development: tenant + seed data provisioned via setup script
- Invite-only user model — admin invites users by email and assigns role on invite

---

## Domain Configuration

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

## Data Source Management

- Add new data source: dynamic form per type (Postgres, DuckDB, MySQL)
- Test connection before saving — descriptive error shown on failure (e.g. "Authentication failed", "Host unreachable")
- All admin actions captured in audit log
- Schema introspection on connect → Knowledge Graph generated
- Manual refresh + scheduled refresh (see METADATA_CATALOG.md)
- Each refresh run stores delta metrics: resources added, removed, marked inactive
- Refresh history visible in admin panel

---

## Metric Registry

- Manual form: name, description, SQL expression, base table, default filters
- AI-assisted: system suggests metrics based on connected schema + column descriptions
- Test metric: run against data source with a sample query before activating
- Edit metric: takes effect immediately on future queries, no approval step
- Deactivate metric: soft delete — excluded from LLM context when inactive
- Metric usage analytics: frequency of use per metric, visible in admin panel

---

## Metadata Catalog

- Browse full resource hierarchy: Database → Schema → Table/View/Function/Sequence → Column
- Add/edit descriptions on any resource (admin and data steward)
- Inactive resources shown with deleted date
- Refresh history: per-run delta summary (added/removed/inactive counts)
- Schedule refresh: set cron expression per data source

---

## User Management

- Invite users by email — admin assigns role on invite (admin / data_steward / viewer)
- Revoke access / change role at any time
- View all active users in tenant

---

## LLM Provider Configuration

- Register LLM providers via dynamic form (fields vary per provider)
- Providers: Anthropic, OpenAI, Google, Groq, Mistral (extensible)
- Set default model per tenant
- Configure auto-router preference (cost vs speed vs quality)
- API keys stored encrypted

---

## MCP Server Management

- Register MCP server: name, endpoint URL, auth config
- Test / ping button to verify connectivity before making live
- Enable / disable per server
- Registered MCP tools exposed to agents at query time

---

## Agent Management

- Define agents: name, description, allowed tools, MCP servers, LLM model
- Enable / disable agents per tenant
- Agents available to users via agentic fallback path

---

## Flagged Answers Review

Admin can review all user-flagged responses with full context:
- Original user question
- Resolved query sent to router
- Path taken (semantic / agentic)
- Generated SQL
- Result returned
- User's correction / feedback text
- Timestamp + user (anonymised if needed)

Used for improving metric definitions, domain config, and future model fine-tuning.

---

## Test as User Mode

- Admin can switch to "test as user" mode in the admin panel
- Simulates the full query flow (Domain Guard → Conversation Resolver → Router → Execute)
- Useful for verifying data sources, metrics, and domain config before going live

---

## Observability Dashboard

Tenant-scoped — admin sees only their own tenant's data:

- LLM usage: token counts, cost, model breakdown over time
- Query volume: total queries, path distribution (semantic vs agentic)
- Latency: P50/P95 response times
- Metric usage: most/least used metrics
- Domain guard: blocked query count + reasons
- Flagged answers: count + status (reviewed / unreviewed)
- Agent execution traces: tool calls, retries, errors

---

## Audit Log

All admin actions are logged:
- Data source add / edit / delete / refresh
- Metric create / edit / deactivate
- User invite / revoke / role change
- LLM provider add / edit
- MCP server add / edit / disable
- Domain config update

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
  id            uuid PK
  tenant_id     uuid FK
  user_id       uuid FK
  query_log_id  uuid FK → QueryLog
  original_question   string
  resolved_query      string
  path_taken          string
  generated_sql       string
  result_preview      json
  user_correction     string?
  reviewed_at         timestamp?
  reviewed_by         uuid? FK → User
  created_at          timestamp

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
