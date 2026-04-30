# Metric Registry

Per-tenant business metric definitions. Replaces Cube.js. Stored as YAML in the app DB. Used by the LLM alongside the Knowledge Graph to generate accurate SQL without guessing table structures.

---

## What a Metric Is

A metric is a named, reusable SQL expression with enough context for the LLM to use it correctly:

```yaml
- name: revenue
  description: "Net revenue after discounts and returns. Use for all sales performance questions."
  sql: "SUM(net_amount)"
  table: orders
  filters:
    - "status != 'cancelled'"

- name: order_count
  description: "Total number of completed orders"
  sql: "COUNT(*)"
  table: orders
  filters:
    - "status = 'completed'"

- name: avg_order_value
  description: "Average value per completed order"
  sql: "AVG(net_amount)"
  table: orders
  filters:
    - "status = 'completed'"

- name: active_customers
  description: "Customers who placed at least one order in the last 30 days"
  sql: "COUNT(DISTINCT customer_id)"
  table: orders
  filters:
    - "created_at >= NOW() - INTERVAL '30 days'"
```

---

## How It Works at Query Time

```
User: "What is monthly revenue by region for electronics?"
    │
    ▼
Load tenant's Metric Registry
    → finds: revenue (sql: SUM(net_amount), table: orders)
    │
    ▼
Load KG subgraph for relevant tables
    → orders hasColumn region
    → orders -[foreignKeyTo]→ products (via product_id)
    → products hasColumn category
    │
    ▼
LLM receives context:
    [metric] revenue = SUM(net_amount) FROM orders WHERE status != 'cancelled'
    [join]   orders.product_id → products.id
    [column] orders.region, orders.created_at, products.category
    │
    ▼
LLM generates:
    SELECT
      DATE_TRUNC('month', o.created_at) AS month,
      o.region,
      SUM(o.net_amount) AS revenue
    FROM orders o
    JOIN products p ON o.product_id = p.id
    WHERE o.status != 'cancelled'
      AND p.category = 'electronics'
      AND o.tenant_id = $1
    GROUP BY 1, 2
    ORDER BY 1
```

The LLM never guesses joins — they come from the KG. The LLM never guesses what "revenue" means — it comes from the Metric Registry.

---

## Data Model

```
MetricDefinition
  id            uuid PK
  tenant_id     uuid FK
  name          string          (unique per tenant)
  description   string          (shown to LLM as context)
  sql           string          (SQL expression, e.g. SUM(net_amount))
  base_table    string          (anchor table for the metric)
  filters       string[]        (default WHERE conditions always applied)
  version       int             (incremented on each edit)
  is_active     boolean
  created_by    uuid FK → User
  created_at    timestamp
  updated_at    timestamp
```

---

## Admin Panel

- List all metrics for the tenant
- Create / edit metric (name, description, SQL expression, base table, default filters)
- Test metric: run against connected data source with sample query
- Deactivate metric (soft delete — inactive metrics excluded from LLM context)
- Version history: see previous versions of a metric definition

---

## Why Descriptions Matter

The `description` field is passed verbatim to the LLM. A good description:
- Tells the LLM **when to use** the metric ("use for all sales performance questions")
- Disambiguates similar metrics ("revenue after discounts" vs "gross revenue")
- Prevents the LLM from picking the wrong metric

Bad description → wrong metric selected → wrong SQL → wrong answer.

---

## Comparison with Cube.js

| | Cube.js | Metric Registry |
|--|---------|-----------------|
| Service | Separate Node.js service | Stored in our DB |
| Config | YAML files or JS | YAML stored as text in DB |
| SQL generation | Cube.js compiler | LLM with KG context |
| Pre-aggregations | Yes | No (use materialized views if needed) |
| External BI tools | Yes (REST/GraphQL API) | No (not in scope) |
| Multi-tenant dynamic schemas | Complex | Simple (one row per metric per tenant) |
| Join awareness | Defined in Cube schema | From Knowledge Graph automatically |
