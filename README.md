# Health & Delivery Data Warehouse
### A Kimball-Style Dimensional Model in SQLite

---

## Why I Built This

After receiving feedback from a data engineering interview, I was encouraged to deepen my skills in:
- SQL and data engineering fundamentals
- Dimensional modelling and different data architectures
- Understanding the **"why"** behind design and implementation decisions

This project is my direct response to that feedback — a hands-on data warehouse built from scratch, applying Kimball dimensional modelling principles across two business domains (health insurance and logistics delivery).

---

## Architecture Overview

```
warehouse_db.db (single SQLite file)
│
├── shared_dim_date          ← conformed dimension (shared across domains)
│
├── health_dim_clients       ← health insurance domain
├── health_dim_products
├── health_fct_lapses
│
├── delivery_dim_customer    ← logistics delivery domain
├── delivery_dim_location
├── delivery_dim_carrier
├── delivery_fct_deliveries
│
├── raw_health_lapses        ← staging layer
└── raw_delivery_data        ← staging layer
```

---

## Kimball Dimensional Modelling Principles Applied

### 1. Grain Definition
Every fact table has a clearly defined grain — the most atomic level of detail:

| Fact Table | Grain |
|---|---|
| `health_fct_lapses` | One row per policy lapse event |
| `delivery_fct_deliveries` | One row per delivery event |

**Why this matters:** Defining the grain first prevents ambiguous queries and ensures measures are meaningful. Adding finer grain later breaks existing reports; defining it correctly upfront avoids this.

---

### 2. Fact Tables — Measures vs Degenerate Dimensions

**Measures** (numeric, additive):
- `delay_minutes` — how late the delivery was
- `distance_km` — distance travelled
- `weight_kg` — package weight
- `premium_at_lapse` — premium amount at time of lapse
- `outstanding_balance` — balance owed

**Degenerate dimensions** (kept on fact, no separate table):
- `delivery_id` — transaction identifier with no attributes
- `lapse_status` — status flag with no additional descriptive attributes

**Why degenerate dimensions:** Creating a separate `dim_status` table with one column would add joins without adding value. Kimball recommends keeping atomic transaction identifiers on the fact table.

---

### 3. Surrogate Keys vs Business Keys

Every dimension table has:
- A **surrogate key** (`customer_key`, `client_key`) — system-generated integer, used for joins
- A **business key** (`customer_id`, `client_id`) — the original source system identifier, kept for lineage

```sql
-- Example: dim_customer
customer_key  INTEGER PRIMARY KEY  -- surrogate (join key)
customer_id   TEXT UNIQUE          -- business key (source system)
```

**Why surrogate keys:** Business keys can change (a customer_id reused, a product_code reformatted). Surrogate keys are stable and make joins faster on integer comparisons.

---

### 4. Conformed Dimensions

`shared_dim_date` is a **conformed dimension** — one table shared across both fact tables.

```sql
-- health fact joins to shared date
JOIN shared_dim_date d ON health_fct_lapses.lapse_date_key = d.date_key

-- delivery fact joins to the same table
JOIN shared_dim_date d ON delivery_fct_deliveries.delivered_date_key = d.date_key
```

**Why conformed dimensions:** This enables cross-domain queries — comparing deliveries and lapses in the same month — which would be impossible if each domain had its own date table with different definitions of "month" or "quarter".

---

### 5. Role-Playing Dimensions

`shared_dim_date` appears **twice** on `health_fct_lapses`:

```sql
lapse_date_key          → shared_dim_date  (when the lapse occurred)
reinstatement_date_key  → shared_dim_date  (when it was reinstated)
```

Same physical table, two different roles. This avoids duplicating the date table while supporting multiple date perspectives on the same event.

---

### 6. SCD Strategy (Slowly Changing Dimensions)

| Dimension | SCD Type | Reason |
|---|---|---|
| `shared_dim_date` | Static | Calendar never changes |
| `health_dim_clients` | Type 1 (overwrite) | Current income is what matters for risk analysis |
| `health_dim_products` | Type 1 (overwrite) | Product changes replace old definition |
| `delivery_dim_customer` | Type 1 (overwrite) | No history required for delivery analytics |
| `delivery_dim_location` | Type 1 (overwrite) | City attributes rarely change |

**If historical tracking were needed** (e.g. tracking how a client's income changed over time), we would implement **SCD Type 2** using `start_date`, `end_date`, and `is_current` columns — as demonstrated in the companion [Org Hierarchy History Tracker](https://github.com/JabulaniMcineka/org-hierarchy-tracker) project.

---

### 7. Staging Layer

Raw data lands in staging tables first:
- `raw_health_lapses` — raw lapse events before transformation
- `raw_delivery_data` — raw delivery records before loading

**Why staging:** Staging separates ingestion from transformation. If the load fails, raw data is preserved. It also supports re-running transformations without re-extracting from source.

---

## Data Architecture — Why SQLite?

This project uses SQLite to demonstrate that **dimensional modelling concepts are database-agnostic**. The same design decisions apply in:
- PostgreSQL (used in the Health Insurance Pipeline project)
- Redshift, BigQuery, Snowflake (cloud data warehouses)
- SQL Server (enterprise environments)

The patterns demonstrated here — conformed dimensions, surrogate keys, role-playing dimensions, SCD strategy — transfer directly to any platform.

---

## Cross-Domain Analytics

Because both fact tables share `shared_dim_date`, cross-domain queries are possible:

```sql
-- Deliveries and lapses by month (cross-domain)
SELECT 
    d.month_name,
    d.year,
    COUNT(DISTINCT f.delivery_key) AS deliveries,
    COUNT(DISTINCT l.lapse_id)     AS lapses
FROM shared_dim_date d
LEFT JOIN delivery_fct_deliveries f ON f.delivered_date_key = d.date_key
LEFT JOIN health_fct_lapses l ON l.lapse_date_key = d.date_key
GROUP BY d.month_name, d.year
ORDER BY d.year, d.month_number;

-- Carrier performance analysis
SELECT 
    ca.carrier_name,
    COUNT(*) AS total_deliveries,
    ROUND(AVG(f.delay_minutes), 1) AS avg_delay,
    ROUND(100.0 * SUM(CASE WHEN f.delay_minutes = 0 THEN 1 ELSE 0 END) / COUNT(*), 1) AS on_time_pct
FROM delivery_fct_deliveries f
JOIN delivery_dim_carrier ca ON f.carrier_key = ca.carrier_key
GROUP BY ca.carrier_name
ORDER BY on_time_pct DESC;
```

---

## What This Project Demonstrates

| Feedback | How This Project Addresses It |
|---|---|
| Deepen SQL skills | Complex multi-table joins, aggregations, cross-domain queries |
| Data engineering fundamentals | Staging → dimensions → facts loading order |
| Dimensional modelling exposure | Full Kimball star schema across two domains |
| Understanding the "why" | Every design decision documented with justification |
| Different data architectures | Data warehouse pattern with conformed dimensions |

---

## Related Projects

- **[Health Insurance Data Pipeline](https://github.com/JabulaniMcineka/health-pipeline)** — AWS Lambda, PostgreSQL, dbt, Docker
- **[Org Hierarchy History Tracker](https://github.com/JabulaniMcineka/org-hierarchy-tracker)** — SCD Type 2 implementation

---

## Resources Applied

- Kimball, R. — *The Data Warehouse Toolkit* (dimensional modelling methodology)
- Reis & Housley — *Fundamentals of Data Engineering* (pipeline architecture)
- 100 Days of Code — Python scripts for data cleaning and loading
