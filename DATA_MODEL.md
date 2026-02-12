# Data Model Documentation

This document provides comprehensive documentation of the dbt data models in this e-commerce analytics project, including detailed descriptions, lineage, and business logic.

## Table of Contents
1. [Model Architecture](#model-architecture)
2. [Data Flow & Lineage](#data-flow--lineage)
3. [Seed Layer](#seed-layer)
4. [Staging Layer](#staging-layer)
5. [Intermediate Layer](#intermediate-layer)
6. [Marts Layer](#marts-layer)
7. [Model Dependencies](#model-dependencies)
8. [Materialization Strategy](#materialization-strategy)

---

## Model Architecture

This project follows a **medallion-style architecture** with clear layer separation:

```
┌─────────────────────────────────────────────────────────────┐
│                        MARTS LAYER                          │
│  Business-ready tables for BI tools and reporting           │
│  Schema: marts | Materialization: TABLE                    │
├─────────────────────────────────────────────────────────────┤
│                    INTERMEDIATE LAYER                       │
│  Business logic, complex joins, derived metrics             │
│  Schema: intermediate | Materialization: VIEW              │
├─────────────────────────────────────────────────────────────┤
│                      STAGING LAYER                          │
│  Cleaned, standardized source data                          │
│  Schema: staging | Materialization: VIEW                   │
├─────────────────────────────────────────────────────────────┤
│                       SEED LAYER                            │
│  Raw CSV data (simulates source systems)                    │
│  Schema: public | Materialization: SEED                    │
└─────────────────────────────────────────────────────────────┘
```

### Layer Guidelines

| Layer | Purpose | Naming Convention | Materialization |
|-------|---------|-------------------|-----------------|
| **Seed** | Raw source data | `raw_*` | Seed (CSV) |
| **Staging** | Cleaning & standardization | `stg_*` | View |
| **Intermediate** | Business logic | `int_*` | View |
| **Marts - Core** | Dimensions & facts | `dim_*`, `fct_*` | Table |
| **Marts - Reporting** | Summary reports | `rpt_*` | Table |

---

## Data Flow & Lineage

### Complete Lineage Graph

```
raw_customers.csv ──► stg_customers ──► int_customer_orders ──► dim_customers
                                              │
raw_orders.csv ─────► stg_orders ─────┐       │
                                      │       │
raw_products.csv ───► stg_products ───┴──► int_order_items ──┬──► fct_daily_sales ──► rpt_sales_summary
                                                              │
                                                              └──► (future models)
```

### Logical Data Flow

**Step 1: Raw Data (Seeds)**
- 6 customers
- 15 orders
- 5 products

**Step 2: Staging (Cleaning)**
- Trim whitespace
- Cast data types
- Lowercase standardization

**Step 3: Intermediate (Business Logic)**
- Join orders with products → `int_order_items`
- Calculate revenue, cost, margin
- Aggregate by customer → `int_customer_orders`

**Step 4: Marts (Analytics-Ready)**
- Create customer dimension with segmentation → `dim_customers`
- Create daily sales facts → `fct_daily_sales`
- Create summary report → `rpt_sales_summary`

---

## Seed Layer

### raw_customers.csv

**Purpose:** Customer master data (simulates CRM system)

**Location:** `seeds/raw_customers.csv`

**Schema:**
```yaml
columns:
  - customer_id: integer (Primary Key)
  - name: text
  - email: text
  - region: text
  - created_at: date
```

**Sample Data:**
```csv
customer_id,name,email,region,created_at
1,Alice,alice@example.com,North,2023-01-10
2,Bob,bob@example.com,South,2023-01-15
3,Charlie,charlie@example.com,East,2023-02-01
```

**Data Quality:**
- 6 total customers
- No duplicates
- All required fields populated

**Business Context:**
- Represents customers who have registered accounts
- `created_at` tracks customer acquisition date
- `region` used for geographic analysis

---

### raw_orders.csv

**Purpose:** Order transaction data (simulates e-commerce order system)

**Location:** `seeds/raw_orders.csv`

**Schema:**
```yaml
columns:
  - order_id: integer (Primary Key)
  - customer_id: integer (Foreign Key → raw_customers)
  - product_id: integer (Foreign Key → raw_products)
  - quantity: integer
  - unit_price: decimal(10,2)
  - order_date: date
  - status: text (enum: completed, pending, cancelled, returned)
```

**Sample Data:**
```csv
order_id,customer_id,product_id,quantity,unit_price,order_date,status
1,1,1,2,29.99,2023-03-01,completed
2,1,2,1,49.99,2023-03-05,completed
3,2,1,1,29.99,2023-03-02,returned
```

**Data Quality:**
- 15 total orders
- Mix of statuses: completed (10), returned (3), pending (1), cancelled (1)
- All orders link to valid customers and products

**Business Context:**
- Represents individual line items (one row per product per order)
- `status` tracks order lifecycle
- `unit_price` is price at time of order (may differ from current price)

---

### raw_products.csv

**Purpose:** Product catalog (simulates product management system)

**Location:** `seeds/raw_products.csv`

**Schema:**
```yaml
columns:
  - product_id: integer (Primary Key)
  - name: text
  - category: text
  - unit_cost: decimal(10,2)
```

**Sample Data:**
```csv
product_id,name,category,unit_cost
1,Widget A,Electronics,15.00
2,Widget B,Electronics,25.00
3,Gadget X,Home,10.00
```

**Data Quality:**
- 5 total products
- Categories: Electronics, Home, Outdoor
- All products have cost data for margin calculation

**Business Context:**
- `unit_cost` is the cost to the company (for margin calculation)
- Products are grouped into categories for analysis
- Catalog is relatively static (doesn't change frequently)

---

## Staging Layer

### stg_customers

**Purpose:** Cleaned and standardized customer data

**Location:** `models/staging/stg_customers.sql`

**SQL Logic:**
```sql
SELECT 
    customer_id,
    TRIM(name) AS name,
    LOWER(TRIM(email)) AS email,
    TRIM(region) AS region,
    created_at::date AS created_at
FROM {{ ref('raw_customers') }}
```

**Transformations:**
1. **Trim whitespace** from `name` and `region`
2. **Lowercase and trim** `email` for consistency
3. **Cast `created_at`** to proper date type

**Tests:**
```yaml
tests:
  - unique: customer_id
  - not_null: customer_id
```

**Downstream Models:**
- `int_customer_orders`

**Materialization:** `VIEW` (in `staging` schema)

**Business Rules:**
- Email addresses are always lowercase for deduplication
- Names and regions are trimmed of leading/trailing spaces

---

### stg_orders

**Purpose:** Cleaned and type-cast order data

**Location:** `models/staging/stg_orders.sql`

**SQL Logic:**
```sql
SELECT 
    order_id,
    customer_id,
    product_id,
    quantity::int AS quantity,
    unit_price::decimal(10,2) AS unit_price,
    order_date::date AS order_date,
    LOWER(TRIM(status)) AS status
FROM {{ ref('raw_orders') }}
```

**Transformations:**
1. **Cast `quantity`** to integer
2. **Cast `unit_price`** to decimal(10,2) for precision
3. **Cast `order_date`** to proper date type
4. **Lowercase and trim** `status` for consistency

**Tests:**
```yaml
tests:
  - unique: order_id
  - not_null: order_id, customer_id, product_id, order_date
  - accepted_values: status [completed, pending, cancelled, returned]

singular_tests:
  - no_negative_quantities.sql
  - no_future_order_dates.sql
```

**Downstream Models:**
- `int_order_items`

**Materialization:** `VIEW` (in `staging` schema)

**Business Rules:**
- All monetary values stored with 2 decimal precision
- Status values are lowercase for consistency
- Quantities must be positive (enforced by test)

---

### stg_products

**Purpose:** Cleaned product catalog data

**Location:** `models/staging/stg_products.sql`

**SQL Logic:**
```sql
SELECT 
    product_id,
    TRIM(name) AS product_name,
    TRIM(category) AS product_category,
    unit_cost::decimal(10,2) AS unit_cost
FROM {{ ref('raw_products') }}
```

**Transformations:**
1. **Trim whitespace** from `name` and `category`
2. **Rename** `name` to `product_name` for clarity
3. **Rename** `category` to `product_category` for clarity
4. **Cast `unit_cost`** to decimal(10,2) for precision

**Tests:**
```yaml
tests:
  - unique: product_id
  - not_null: product_id
```

**Downstream Models:**
- `int_order_items`

**Materialization:** `VIEW` (in `staging` schema)

**Business Rules:**
- Product names and categories are trimmed
- Cost values stored with 2 decimal precision

---

## Intermediate Layer

### int_order_items

**Purpose:** Enriched order data with product details and calculated metrics

**Location:** `models/intermediate/int_order_items.sql`

**SQL Logic:**
```sql
SELECT 
    o.order_id,
    o.customer_id,
    o.product_id,
    o.quantity,
    o.unit_price,
    o.order_date,
    o.status,
    p.product_name,
    p.product_category,
    
    -- Calculated metrics
    o.quantity * o.unit_price AS revenue,
    o.quantity * p.unit_cost AS cost,
    (o.quantity * o.unit_price) - (o.quantity * p.unit_cost) AS margin
    
FROM {{ ref('stg_orders') }} o
INNER JOIN {{ ref('stg_products') }} p 
    ON o.product_id = p.product_id
WHERE o.status IN ('completed', 'returned')  -- Exclude pending/cancelled
```

**Transformations:**
1. **Join orders with products** to enrich with product details
2. **Calculate revenue** = quantity × unit_price
3. **Calculate cost** = quantity × unit_cost
4. **Calculate margin** = revenue - cost
5. **Filter** to only completed and returned orders

**Tests:**
```yaml
tests:
  - not_null: customer_id, product_id, revenue, cost, margin
  - relationships:
      - customer_id → stg_customers.customer_id
      - product_id → stg_products.product_id

singular_tests:
  - no_negative_margin.sql
  - revenue_matches_calculation.sql
```

**Upstream Dependencies:**
- `stg_orders`
- `stg_products`

**Downstream Models:**
- `fct_daily_sales`

**Materialization:** `VIEW` (in `intermediate` schema)

**Business Rules:**
- Only include completed and returned orders (financial impact)
- Returned orders still count as revenue initially (refunds handled separately)
- Margin can be negative for returned items or discounted products

**Key Metrics:**
- `revenue`: Top-line sales amount
- `cost`: COGS (Cost of Goods Sold)
- `margin`: Gross profit per order line

---

### int_customer_orders

**Purpose:** Customer-level aggregations

**Location:** `models/intermediate/int_customer_orders.sql`

**SQL Logic:**
```sql
SELECT 
    c.customer_id,
    c.name,
    c.email,
    c.region,
    c.created_at,
    
    -- Aggregated metrics
    COUNT(DISTINCT oi.order_id) AS order_count,
    COALESCE(SUM(oi.revenue), 0) AS total_revenue,
    COALESCE(SUM(oi.quantity), 0) AS total_quantity
    
FROM {{ ref('stg_customers') }} c
LEFT JOIN {{ ref('int_order_items') }} oi 
    ON c.customer_id = oi.customer_id
GROUP BY 
    c.customer_id,
    c.name,
    c.email,
    c.region,
    c.created_at
```

**Transformations:**
1. **Left join customers with order items** (includes customers with no orders)
2. **Count distinct orders** per customer
3. **Sum revenue** per customer (with COALESCE for customers with no orders)
4. **Sum quantity** per customer

**Tests:**
```yaml
tests:
  - not_null: customer_id, total_revenue, order_count
```

**Upstream Dependencies:**
- `stg_customers`
- `int_order_items`

**Downstream Models:**
- `dim_customers`

**Materialization:** `VIEW` (in `intermediate` schema)

**Business Rules:**
- Include all customers (even those without orders)
- Use LEFT JOIN to preserve customer records
- COALESCE ensures 0 instead of NULL for customers with no orders

**Key Metrics:**
- `order_count`: Number of distinct orders placed
- `total_revenue`: Lifetime customer value
- `total_quantity`: Total items purchased

---

## Marts Layer

### Core: dim_customers

**Purpose:** Customer dimension table with segmentation for BI tools

**Location:** `models/marts/core/dim_customers.sql`

**SQL Logic:**
```sql
SELECT 
    customer_id,
    name,
    region,
    order_count,
    total_revenue,
    total_quantity,
    
    -- Customer segmentation
    CASE 
        WHEN total_revenue >= 300 THEN 'high_value'
        WHEN total_revenue >= 100 THEN 'medium_value'
        ELSE 'low_value'
    END AS segment
    
FROM {{ ref('int_customer_orders') }}
```

**Transformations:**
1. **Add customer segmentation** based on total revenue
   - High Value: >= $300 lifetime revenue
   - Medium Value: >= $100 lifetime revenue
   - Low Value: < $100 lifetime revenue

**Tests:**
```yaml
tests:
  - unique: customer_id
  - not_null: customer_id, total_revenue, order_count
  - accepted_values: segment [high_value, medium_value, low_value]
```

**Upstream Dependencies:**
- `int_customer_orders`

**Downstream Use Cases:**
- Customer segmentation dashboards
- Targeted marketing campaigns
- Customer lifetime value analysis

**Materialization:** `TABLE` (in `marts` schema)

**Business Rules:**
- Segmentation thresholds:
  - High Value: Top-tier customers (>= $300)
  - Medium Value: Regular customers (>= $100)
  - Low Value: Casual customers (< $100)

**Sample Output:**
```
customer_id | name    | region | order_count | total_revenue | segment
------------|---------|--------|-------------|---------------|---------------
1           | Alice   | North  | 5           | 450.00        | high_value
2           | Bob     | South  | 2           | 120.00        | medium_value
3           | Charlie | East   | 1           | 30.00         | low_value
```

---

### Core: fct_daily_sales

**Purpose:** Daily sales fact table aggregated by product

**Location:** `models/marts/core/fct_daily_sales.sql`

**SQL Logic:**
```sql
SELECT 
    order_date,
    product_id,
    product_name,
    product_category,
    
    -- Aggregated metrics
    COUNT(DISTINCT order_id) AS order_count,
    SUM(quantity) AS units_sold,
    SUM(revenue) AS revenue,
    SUM(cost) AS cost,
    SUM(margin) AS margin
    
FROM {{ ref('int_order_items') }}
GROUP BY 
    order_date,
    product_id,
    product_name,
    product_category
```

**Grain:** One row per date + product combination

**Transformations:**
1. **Aggregate to daily-product level**
2. **Sum all metrics** (revenue, cost, margin, quantity)
3. **Count distinct orders** per day-product

**Tests:**
```yaml
tests:
  - not_null: order_date, product_id, revenue
```

**Upstream Dependencies:**
- `int_order_items`

**Downstream Use Cases:**
- Daily sales dashboards
- Product performance analysis
- Trend analysis over time
- Category performance comparison

**Materialization:** `TABLE` (in `marts` schema)

**Business Rules:**
- Grain is date + product (multiple orders per day are aggregated)
- All monetary metrics are summed
- Only includes completed and returned orders (filtered in int_order_items)

**Sample Output:**
```
order_date | product_id | product_name | order_count | units_sold | revenue | cost   | margin
-----------|------------|--------------|-------------|------------|---------|--------|--------
2023-03-01 | 1          | Widget A     | 3           | 8          | 239.92  | 120.00 | 119.92
2023-03-01 | 2          | Widget B     | 2           | 2          | 99.98   | 50.00  | 49.98
2023-03-02 | 1          | Widget A     | 1           | 1          | 29.99   | 15.00  | 14.99
```

**Key Metrics:**
- `order_count`: Number of distinct orders
- `units_sold`: Total quantity sold
- `revenue`: Total sales amount
- `cost`: Total cost of goods sold
- `margin`: Total gross profit

---

### Reporting: rpt_sales_summary

**Purpose:** Overall business KPI summary (single-row report)

**Location:** `models/marts/reporting/rpt_sales_summary.sql`

**SQL Logic:**
```sql
SELECT 
    SUM(revenue) AS total_revenue,
    SUM(units_sold) AS total_units,
    SUM(margin) AS total_margin,
    COUNT(DISTINCT order_date) AS active_days,
    
    -- Calculated KPIs
    SUM(revenue) / NULLIF(SUM(units_sold), 0) AS avg_revenue_per_unit
    
FROM {{ ref('fct_daily_sales') }}
```

**Grain:** Single summary row (overall totals)

**Transformations:**
1. **Sum all revenue** across all days and products
2. **Sum all units sold**
3. **Sum total margin**
4. **Count distinct active days** (days with at least one order)
5. **Calculate average revenue per unit**

**Tests:**
```yaml
tests:
  - not_null: total_revenue, total_units, total_margin
```

**Upstream Dependencies:**
- `fct_daily_sales`

**Downstream Use Cases:**
- Executive dashboards
- KPI scorecards
- Business health monitoring
- Quick summary metrics

**Materialization:** `TABLE` (in `marts` schema)

**Business Rules:**
- Single row output (overall summary)
- NULLIF prevents division by zero in average calculation
- Includes all time periods (no date filter)

**Sample Output:**
```
total_revenue | total_units | total_margin | active_days | avg_revenue_per_unit
--------------|-------------|--------------|-------------|---------------------
1,249.85      | 42          | 624.85       | 8           | 29.76
```

**Key Metrics:**
- `total_revenue`: All-time revenue
- `total_units`: All-time units sold
- `total_margin`: All-time gross profit
- `active_days`: Number of days with sales activity
- `avg_revenue_per_unit`: Average selling price

---

## Model Dependencies

### Dependency Tree

```
Level 0 (Seeds):
  - raw_customers.csv
  - raw_orders.csv
  - raw_products.csv

Level 1 (Staging):
  - stg_customers ← raw_customers
  - stg_orders ← raw_orders
  - stg_products ← raw_products

Level 2 (Intermediate):
  - int_order_items ← stg_orders + stg_products
  - int_customer_orders ← stg_customers + int_order_items

Level 3 (Marts - Core):
  - dim_customers ← int_customer_orders
  - fct_daily_sales ← int_order_items

Level 4 (Marts - Reporting):
  - rpt_sales_summary ← fct_daily_sales
```

### Execution Order

When you run `dbt run`, models are built in this order:

1. **Seeds loaded** (raw_customers, raw_orders, raw_products)
2. **Staging views created** (stg_customers, stg_orders, stg_products)
3. **Intermediate views created** (int_order_items, int_customer_orders)
4. **Mart tables built** (dim_customers, fct_daily_sales)
5. **Report tables built** (rpt_sales_summary)

---

## Materialization Strategy

### Configuration

**Project-level** (`dbt_project.yml`):
```yaml
models:
  ecommerce_analytics:
    staging:
      +materialized: view
      +schema: staging
    intermediate:
      +materialized: view
      +schema: intermediate
    marts:
      +materialized: table
      +schema: marts
```

### Rationale

| Layer | Materialization | Why? |
|-------|-----------------|------|
| **Staging** | `VIEW` | - Fast to build<br>- Always reflects source<br>- No storage cost |
| **Intermediate** | `VIEW` | - Reusable logic<br>- No duplication<br>- Fast iteration |
| **Marts** | `TABLE` | - Query performance<br>- BI tool access<br>- Stable for dashboards |

### Performance Considerations

**Views (Staging & Intermediate):**
- ✓ Fast to build (< 1 second)
- ✓ Always fresh
- ✗ Query time includes upstream computation

**Tables (Marts):**
- ✓ Fast to query (pre-computed)
- ✓ Indexed for BI tools
- ✗ Slower to build (full refresh)

**For Production at Scale:**
Consider switching to **incremental** materialization for large fact tables:

```sql
{{ config(
    materialized='incremental',
    unique_key='order_date || product_id'
) }}
```

---

## Summary

### Model Count
- **Seeds:** 3 (raw_customers, raw_orders, raw_products)
- **Staging:** 3 (stg_customers, stg_orders, stg_products)
- **Intermediate:** 2 (int_order_items, int_customer_orders)
- **Marts - Core:** 2 (dim_customers, fct_daily_sales)
- **Marts - Reporting:** 1 (rpt_sales_summary)
- **Total:** 11 objects

### Data Volume
- **Customers:** 6
- **Orders:** 15 (10 completed, 3 returned, 1 pending, 1 cancelled)
- **Products:** 5
- **Order Items:** 12 (after filtering to completed/returned)

### Key Metrics Summary
- **Total Revenue:** ~$1,250
- **Total Margin:** ~$625
- **Average Order Value:** ~$83
- **Average Revenue Per Unit:** ~$30

---

## Next Steps

- **[Explore testing strategies](TESTING_GUIDE.md)** applied to these models
- **[Review deployment options](DEPLOYMENT_GUIDE.md)** for orchestrating these models
- **[Run the models locally](SETUP.md)** to see them in action
- **Extend the models** by adding your own transformations

---

**Questions?** Refer to the [dbt documentation](https://docs.getdbt.com/) or open an issue on GitHub.
