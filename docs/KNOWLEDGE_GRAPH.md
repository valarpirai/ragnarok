# Knowledge Graph

RDF-based knowledge graph representing the metadata relationships across all tenant data sources. Stored as TTL/NT files per tenant. Used for join path discovery, impact analysis, and RAG context enrichment.

---

## Resource Hierarchy

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

---

## IRI Scheme

Pattern: `https://datachat.ai/kg/{tenant_id}/{resource_type}/{...path}`

IRIs are generated deterministically from path segments. Same resource always produces the same IRI — duplicates are impossible on re-sync.

| Resource | IRI Example |
|----------|-------------|
| DataSource | `https://datachat.ai/kg/t123/datasource/ds456` |
| Database | `https://datachat.ai/kg/t123/database/mydb` |
| Schema | `https://datachat.ai/kg/t123/schema/mydb/public` |
| Table | `https://datachat.ai/kg/t123/table/mydb/public/orders` |
| Column | `https://datachat.ai/kg/t123/column/mydb/public/orders/customer_id` |
| Index | `https://datachat.ai/kg/t123/index/mydb/public/orders/idx_customer_id` |
| ForeignKey | `https://datachat.ai/kg/t123/foreignkey/mydb/public/orders/fk_customer` |
| View | `https://datachat.ai/kg/t123/view/mydb/public/v_monthly_sales` |
| Function | `https://datachat.ai/kg/t123/function/mydb/public/calculate_tax` |
| Trigger | `https://datachat.ai/kg/t123/trigger/mydb/public/orders/trg_audit` |
| Sequence | `https://datachat.ai/kg/t123/sequence/mydb/public/order_id_seq` |

---

## Ontology

Namespace: `https://datachat.ai/ontology#` (prefix: `dc`)

### Classes

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

### Structural Properties

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

### FK Properties

| Property | Domain | Range |
|----------|--------|-------|
| `dc:fromColumn` | ForeignKey | Column |
| `dc:referencesTable` | ForeignKey | Table |
| `dc:referencesColumn` | ForeignKey | Column |

### Descriptive Properties

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

---

## TTL Example

```turtle
@prefix dc:   <https://datachat.ai/ontology#>
@prefix rdfs: <http://www.w3.org/2000/01/rdf-schema#>
@prefix xsd:  <http://www.w3.org/2001/XMLSchema#>
@prefix :     <https://datachat.ai/kg/t123/> .

:datasource/ds456
    a dc:DataSource ;
    dc:type         "postgres" ;
    dc:dataSourceOf :database/mydb .

:database/mydb
    a dc:Database ;
    rdfs:label   "mydb" ;
    dc:hasSchema :schema/mydb/public ;
    dc:status    "active" .

:schema/mydb/public
    a dc:Schema ;
    rdfs:label  "public" ;
    dc:hasTable :table/mydb/public/orders ;
    dc:hasTable :table/mydb/public/customers ;
    dc:hasView  :view/mydb/public/v_monthly_sales .

:table/mydb/public/orders
    a dc:Table ;
    rdfs:label       "orders" ;
    dc:description   "Customer purchase orders" ;
    dc:hasColumn     :column/mydb/public/orders/id ;
    dc:hasColumn     :column/mydb/public/orders/customer_id ;
    dc:hasForeignKey :foreignkey/mydb/public/orders/fk_customer ;
    dc:status        "active" ;
    dc:syncedAt      "2026-04-30T10:00:00Z"^^xsd:dateTime .

:column/mydb/public/orders/id
    a dc:Column ;
    rdfs:label     "id" ;
    dc:dataType    "integer" ;
    dc:isPrimaryKey true ;
    dc:isNullable  false .

:column/mydb/public/orders/customer_id
    a dc:Column ;
    rdfs:label    "customer_id" ;
    dc:dataType   "integer" ;
    dc:isNullable false .

:foreignkey/mydb/public/orders/fk_customer
    a dc:ForeignKey ;
    dc:fromColumn       :column/mydb/public/orders/customer_id ;
    dc:referencesTable  :table/mydb/public/customers ;
    dc:referencesColumn :column/mydb/public/customers/id .
```

---

## Storage

- **Format:** TTL (human-readable, compact) for storage; NT for streaming/appending during refresh
- **Location:** `./storage/kg/{tenant_id}/{data_source_id}.ttl`
- **Node.js library:** N3.js (pure TypeScript, supports TTL + NT read/write/parse)
- **On schema refresh:** regenerate affected triples; IRI uniqueness ensures no duplicates on merge
- **Future (Phase 3):** serve via SPARQL endpoint (Oxigraph WASM or Apache Jena Fuseki)

---

## Graph Population

Triggered after each schema introspection (connect or refresh):

```
Introspect data source
    │
    ├── Generate IRIs for all resources
    ├── Emit triples: Database → Schema → Table → Column
    ├── Emit triples: Table → Index
    ├── Emit triples: Table → ForeignKey → (referencesTable, referencesColumn)
    ├── Emit triples: Schema → View, Function, Sequence
    ├── Merge into tenant TTL file (upsert by IRI)
    └── Mark removed IRIs as dc:status "inactive"
```

### FK Detection per DB Type

| DB | Query Source |
|----|-------------|
| Postgres | `information_schema.referential_constraints` + `key_column_usage` |
| MySQL | `information_schema.key_column_usage` where `referenced_table_name IS NOT NULL` |
| DuckDB | `pragma_foreign_key_list(table_name)` |

---

## Use Cases

**Join path discovery (agentic RAG)**
Traverse FK edges to find how tables connect. Passed as context to the LLM so it knows join conditions without guessing.

**Cube.js schema generation**
Auto-suggest joins in Cube.js cube definitions by reading FK relationships from the graph.

**Impact analysis**
Given a column IRI, find all ForeignKeys, Views, and Functions that reference it.

**Metadata catalog navigation**
UI graph view: browse resource hierarchy, click through relationships.

**Description enrichment**
Descriptions added by admin/data steward are written as `dc:description` triples back into the TTL file.
