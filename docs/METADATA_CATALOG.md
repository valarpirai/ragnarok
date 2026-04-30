# Metadata Catalog

Tracks schema metadata for all tenant data sources. Supports manual and scheduled refresh, delta detection, soft deletes, and human-annotated descriptions.

---

## Roles

| Role | Can Refresh | Can Schedule | Browse Metadata | Add Descriptions |
|------|-------------|--------------|-----------------|-----------------|
| Admin | Yes | Yes | Yes | Yes |
| Data Steward | No | No | Yes | Yes |
| Viewer | No | No | No | No |

Descriptions are annotations only — no changes to the original data source.

---

## Schema Sync Flow

### On Data Source Connect (first time)
```
Connect → Test Connection
    │
    ▼
Introspect full hierarchy:
  Database → Schema → Tables, Views, Functions, Sequences
  Table → Columns, Indexes, ForeignKeys, Triggers
    │
    ▼
Insert all resources as active (DataSourceTable, DataSourceColumn, etc.)
    │
    ▼
Generate Knowledge Graph TTL (IRIs for all resources + relationship triples)
    │
    ▼
Embed resource descriptions into PGVector (for RAG)
    │
    ▼
Log SchemaRefreshLog (trigger: connect, delta: all new)
```

### On Manual or Scheduled Refresh
```
Fetch current schema from data source
    │
    ▼
Compare against stored resources
    │
    ├── New resource found     → INSERT as active
    ├── Column type changed    → UPDATE, log change
    ├── Resource missing       → mark status = inactive, set deleted_at
    └── Resource re-appeared   → mark status = active, clear deleted_at
    │
    ▼
Update Knowledge Graph TTL (regenerate affected triples, mark inactive IRIs)
    │
    ▼
Re-embed changed resources into PGVector
    │
    ▼
Log SchemaRefreshLog (delta summary: added/removed counts)
    │
    ▼
Notify admin if inactive resources detected
```

---

## Delta Detection Rules

- **Added resource** — appears in data source, not in DB → `INSERT status=active`
- **Removed resource** — in DB as active, not in data source → `status=inactive, deleted_at=now()`
- **Type changed** — column data_type differs → UPDATE, record previous type in log
- **Re-appeared resource** — previously inactive, now present again → `status=active, deleted_at=null`
- **No change** — skip, update `last_seen_at` only

---

## Data Model

```
DataSource
  id                  uuid PK
  tenant_id           uuid FK
  name                string
  type                enum (postgres, duckdb, mysql)
  host                string
  port                int
  database            string
  username            string
  encrypted_password  string
  status              enum (active, error, testing)
  last_synced_at      timestamp
  created_at          timestamp

DataSourceTable          (covers Tables and Views)
  id                  uuid PK
  data_source_id      uuid FK
  tenant_id           uuid FK
  schema_name         string  (e.g. "public")
  table_name          string
  resource_type       enum (table, view)
  description         string? (added by admin/steward)
  status              enum (active, inactive)
  first_seen_at       timestamp
  last_seen_at        timestamp
  deleted_at          timestamp?

DataSourceColumn
  id                  uuid PK
  table_id            uuid FK
  data_source_id      uuid FK
  tenant_id           uuid FK
  column_name         string
  data_type           string
  ordinal_position    int
  is_nullable         boolean
  description         string? (added by admin/steward)
  status              enum (active, inactive)
  first_seen_at       timestamp
  last_seen_at        timestamp
  deleted_at          timestamp?

DataSourceResource       (Indexes, Functions, Triggers, Sequences)
  id                  uuid PK
  data_source_id      uuid FK
  tenant_id           uuid FK
  schema_name         string
  resource_name       string
  resource_type       enum (index, function, trigger, sequence)
  parent_table_id     uuid? FK → DataSourceTable
  description         string? (added by admin/steward)
  metadata_json       json   (type-specific details)
  status              enum (active, inactive)
  first_seen_at       timestamp
  last_seen_at        timestamp
  deleted_at          timestamp?

SchemaRefreshLog
  id                  uuid PK
  data_source_id      uuid FK
  tenant_id           uuid FK
  trigger             enum (connect, manual, scheduled)
  triggered_by        uuid? FK → User (null if scheduled)
  started_at          timestamp
  completed_at        timestamp?
  status              enum (running, completed, failed)
  resources_added     int
  resources_removed   int
  error_message       string?

SchemaRefreshSchedule
  id                  uuid PK
  data_source_id      uuid FK
  tenant_id           uuid FK
  cron_expression     string  (e.g. "0 2 * * *" = daily at 2am)
  is_active           boolean
  last_run_at         timestamp?
  next_run_at         timestamp?
  created_by          uuid FK → User
```

---

## Metadata Browser (Admin Panel)

- Tree view: Database → Schema → Tables/Views/Functions/Sequences → Columns/Indexes/Triggers
- Inactive resources shown greyed out with deleted date
- Inline description editing (debounced save) — admin and data steward only
- Search/filter by resource name, type, schema
- Refresh history tab: list of SchemaRefreshLog entries with delta summary
- Schedule tab: set/edit cron expression per data source

---

## Sync on Description Update

When a description is added or updated by admin/data steward, two things happen:

```
Description saved
    │
    ├── Re-embed resource into PGVector (for agentic RAG)
    │       Upsert by resource id
    │
    └── Update dc:description triple in KG TTL file
            Re-write affected triples in storage/kg/{tenant_id}/{data_source_id}.ttl
```

This keeps both the agentic fallback (PGVector) and the Knowledge Graph (TTL) in sync with human annotations.
