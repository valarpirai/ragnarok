# Metadata Catalog & Knowledge Graph

Tracks schema metadata for all tenant data sources. Supports manual and scheduled refresh, delta detection, soft deletes, and human-annotated descriptions. Metadata is mirrored into an RDF Knowledge Graph (TTL files) used for join path discovery and LLM context enrichment.

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

### On Description Update
```
Description saved
    │
    ├── Re-embed resource into PGVector (for agentic RAG)
    │       Upsert by resource id
    │
    └── Update dc:description triple in KG TTL file
            Re-write affected triples in storage/kg/{tenant_id}/{data_source_id}.ttl
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

## Knowledge Graph

The Knowledge Graph mirrors catalog metadata as RDF triples stored in TTL files per tenant. Used by the LLM for join path discovery and by the agentic fallback for RAG context enrichment.

### Resource Hierarchy

```
DataSource  (connection config — property of Database)
    └─ Database
           └─ Schema
                  ├─ Table
                  │    ├─ Column
                  │    ├─ Index
                  │    ├─ ForeignKey ──→ references Table + Column
                  │    └─ Trigger
                  ├─ View
                  │    └─ Column
                  ├─ Function
                  └─ Sequence
```

### IRI Scheme

Pattern: `https://ragnarok.ai/kg/{hash}`

Hash is a **UUID v5** generated deterministically from `{resource_type}:{tenant_id}:{db}:{schema}:{...path}`. Tenant isolation is baked into the hash — same resource always produces the same IRI, different tenants always produce different IRIs.

| Resource | Hash Input | IRI Example |
|----------|------------|-------------|
| DataSource | `datasource:t123:ds456` | `https://ragnarok.ai/kg/3f2a1b4c-...` |
| Database | `database:t123:mydb` | `https://ragnarok.ai/kg/7d8e9f0a-...` |
| Schema | `schema:t123:mydb:public` | `https://ragnarok.ai/kg/1c2d3e4f-...` |
| Table | `table:t123:mydb:public:orders` | `https://ragnarok.ai/kg/5a6b7c8d-...` |
| Column | `column:t123:mydb:public:orders:customer_id` | `https://ragnarok.ai/kg/1e2f3a4b-...` |
| Index | `index:t123:mydb:public:orders:idx_customer_id` | `https://ragnarok.ai/kg/4b5c6d7e-...` |
| ForeignKey | `foreignkey:t123:mydb:public:orders:fk_customer` | `https://ragnarok.ai/kg/9a0b1c2d-...` |
| View | `view:t123:mydb:public:v_monthly_sales` | `https://ragnarok.ai/kg/3c4d5e6f-...` |
| Function | `function:t123:mydb:public:calculate_tax` | `https://ragnarok.ai/kg/8f9a0b1c-...` |
| Trigger | `trigger:t123:mydb:public:orders:trg_audit` | `https://ragnarok.ai/kg/2d3e4f5a-...` |
| Sequence | `sequence:t123:mydb:public:order_id_seq` | `https://ragnarok.ai/kg/6b7c8d9e-...` |

### Ontology

Namespace: `https://ragnarok.ai/ontology#` (prefix: `dc`)

**Classes**

```
dc:DataSource
dc:Database
dc:Schema
dc:Table
dc:View
dc:Column
dc:Index
dc:ForeignKey
dc:Function
dc:Trigger
dc:Sequence
```

**Structural Properties**

| Property | Domain | Range | Meaning |
|----------|--------|-------|---------|
| `dc:dataSourceOf` | DataSource | Database | connection config belongs to database |
| `dc:hasSchema` | Database | Schema | database contains schema |
| `dc:hasTable` | Schema | Table | schema contains table |
| `dc:hasView` | Schema | View | schema contains view |
| `dc:hasColumn` | Table / View | Column | table or view has column |
| `dc:hasIndex` | Table | Index | table has index |
| `dc:hasForeignKey` | Table | ForeignKey | table has FK constraint |
| `dc:hasFunction` | Schema | Function | schema has function |
| `dc:hasTrigger` | Table | Trigger | table has trigger |
| `dc:hasSequence` | Schema | Sequence | schema has sequence |

**FK Properties**

| Property | Domain | Range |
|----------|--------|-------|
| `dc:fromColumn` | ForeignKey | Column |
| `dc:referencesTable` | ForeignKey | Table |
| `dc:referencesColumn` | ForeignKey | Column |

**Descriptive Properties**

| Property | Domain | Range |
|----------|--------|-------|
| `rdfs:label` | any | string (resource name) |
| `dc:description` | any | string (human annotation) |
| `dc:dataType` | Column | string |
| `dc:isNullable` | Column | boolean |
| `dc:isPrimaryKey` | Column | boolean |
| `dc:status` | any | "active" \| "inactive" |
| `dc:tenantId` | any | string |
| `dc:syncedAt` | any | xsd:dateTime |

### TTL Example

```turtle
@prefix dc:   <https://ragnarok.ai/ontology#>
@prefix rdfs: <http://www.w3.org/2000/01/rdf-schema#>
@prefix xsd:  <http://www.w3.org/2001/XMLSchema#>
@prefix :     <https://ragnarok.ai/kg/> .

# IRIs below are UUID v5 hashes — shown truncated for readability

:3f2a1b4c-...                          # datasource:t123:ds456
    a dc:DataSource ;
    dc:type         "postgres" ;
    dc:dataSourceOf :7d8e9f0a-... .    # database:t123:mydb

:7d8e9f0a-...                          # database:t123:mydb
    a dc:Database ;
    rdfs:label   "mydb" ;
    dc:hasSchema :1c2d3e4f-... ;       # schema:t123:mydb:public
    dc:status    "active" .

:1c2d3e4f-...                          # schema:t123:mydb:public
    a dc:Schema ;
    rdfs:label  "public" ;
    dc:hasTable :5a6b7c8d-... ;        # table:t123:mydb:public:orders
    dc:hasTable :9e0f1a2b-... ;        # table:t123:mydb:public:customers
    dc:hasView  :3c4d5e6f-... .        # view:t123:mydb:public:v_monthly_sales

:5a6b7c8d-...                          # table:t123:mydb:public:orders
    a dc:Table ;
    rdfs:label       "orders" ;
    dc:description   "Customer purchase orders" ;
    dc:hasColumn     :7a8b9c0d-... ;   # column:...:orders:id
    dc:hasColumn     :1e2f3a4b-... ;   # column:...:orders:customer_id
    dc:hasForeignKey :9a0b1c2d-... ;   # foreignkey:...:orders:fk_customer
    dc:status        "active" ;
    dc:syncedAt      "2026-04-30T10:00:00Z"^^xsd:dateTime .

:7a8b9c0d-...                          # column:t123:mydb:public:orders:id
    a dc:Column ;
    rdfs:label      "id" ;
    dc:dataType     "integer" ;
    dc:isPrimaryKey true ;
    dc:isNullable   false .

:1e2f3a4b-...                          # column:t123:mydb:public:orders:customer_id
    a dc:Column ;
    rdfs:label    "customer_id" ;
    dc:dataType   "integer" ;
    dc:isNullable false .

:9a0b1c2d-...                          # foreignkey:t123:mydb:public:orders:fk_customer
    a dc:ForeignKey ;
    dc:fromColumn       :1e2f3a4b-... ;   # orders.customer_id
    dc:referencesTable  :9e0f1a2b-... ;   # customers
    dc:referencesColumn :5c6d7e8f-... .   # customers.id
```

### Storage

- **Format:** TTL (human-readable, compact) for storage; NT for streaming/appending during refresh
- **Location:** `./storage/kg/{tenant_id}/{data_source_id}.ttl`
- **Node.js library:** N3.js (pure TypeScript, supports TTL + NT read/write/parse)
- **On schema refresh:** regenerate affected triples; IRI uniqueness ensures no duplicates on merge
- **Future (Phase 3):** serve via SPARQL endpoint (Oxigraph WASM or Apache Jena Fuseki)

### FK Detection per DB Type

| DB | Query Source |
|----|-------------|
| Postgres | `information_schema.referential_constraints` + `key_column_usage` |
| MySQL | `information_schema.key_column_usage` where `referenced_table_name IS NOT NULL` |
| DuckDB | `pragma_foreign_key_list(table_name)` |

### Use Cases

**Join path discovery**
Traverse FK edges to find how tables connect. Passed as context to the LLM so it knows join conditions without guessing.

**Impact analysis**
Given a column IRI, find all ForeignKeys, Views, and Functions that reference it.

**Metadata catalog navigation**
UI graph view: browse resource hierarchy, click through relationships.

**Description enrichment**
Descriptions added by admin/data steward are written as `dc:description` triples back into the TTL file.
