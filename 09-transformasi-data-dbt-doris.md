# 🔧 Cara Transformasi Data Menggunakan DBT Pada Apache Doris

> Panduan lengkap menggunakan **DBT (Data Build Tool)** untuk melakukan transformasi data di **Apache Doris** — dari raw data (ODS) menjadi clean data (DWD) dan aggregated data (DWS/ADS) siap pakai untuk analytics & reporting.

---

## 📋 Table of Contents

- [Arsitektur Data Warehouse](#arsitektur-data-warehouse)
- [Setup DBT + Doris](#setup-dbt--doris)
- [Layer 1: Staging (stg)](#layer-1-staging-stg)
- [Layer 2: Intermediate (int)](#layer-2-intermediate-int)
- [Layer 3: Marts - Facts & Dimensions](#layer-3-marts---facts--dimensions)
- [Testing & Data Quality](#testing--data-quality)
- [Incremental Models](#incremental-models)
- [Macros & Reusable Code](#macros--reusable-code)
- [Running DBT](#running-dbt)
- [Troubleshooting](#troubleshooting)

---

## Arsitektur Data Warehouse

```
┌──────────────────────────────────────────────────────────────┐
│               Data Warehouse Layers (DBT + Doris)           │
│                                                              │
│  ODS (Raw)         DWD (Cleaned)       DWS/ADS (Marts)      │
│  ┌──────────┐     ┌──────────────┐    ┌────────────────┐    │
│  │ ods_users│────▶│ stg_users    │───▶│ dim_users      │    │
│  │ ods_orders────▶│ stg_orders   │───▶│ fct_revenue    │    │
│  │ ods_products──▶│ stg_products │───▶│ fct_orders     │    │
│  └──────────┘     └──────┬───────┘    │ agg_monthly    │    │
│  (from CDC)              │            └────────────────┘    │
│                   ┌──────▼───────┐           │              │
│                   │ int_orders   │           ▼              │
│                   │ _enriched    │    ┌─────────────┐       │
│                   └──────────────┘    │  BI / Report │       │
│                   (intermediate)      │  Dashboard   │       │
│                                      └─────────────┘       │
│                                                              │
│  materialized:    materialized:       materialized:          │
│  (from source)    view                table                  │
└──────────────────────────────────────────────────────────────┘
```

---

## Setup DBT + Doris

### Install & Init

```bash
# Activate virtual environment
source ~/dbt-env/bin/activate

# Install dbt-doris
pip install dbt-core dbt-doris

# Init project
dbt init doris_warehouse
cd doris_warehouse
```

### profiles.yml

```yaml
# ~/.dbt/profiles.yml
doris_warehouse:
  target: dev
  outputs:
    dev:
      type: doris
      host: 127.0.0.1
      port: 9030
      username: root
      password: ""
      schema: dwd_ecommerce
      threads: 4
```

### dbt_project.yml

```yaml
name: 'doris_warehouse'
version: '1.0.0'
config-version: 2
profile: 'doris_warehouse'

model-paths: ["models"]
seed-paths: ["seeds"]
test-paths: ["tests"]
macro-paths: ["macros"]

models:
  doris_warehouse:
    staging:
      +materialized: view
      +schema: staging
    intermediate:
      +materialized: ephemeral
    marts:
      +materialized: table
      +schema: marts
```

```bash
# Test koneksi
dbt debug
# ✅ All checks passed!
```

### Sources Definition

```yaml
# models/staging/_sources.yml
version: 2

sources:
  - name: ods_ecommerce
    description: "ODS layer - raw data dari CDC pipeline"
    database: ods_ecommerce
    tables:
      - name: ods_users
        description: "Raw user data"
        columns:
          - name: user_id
            tests: [unique, not_null]
      - name: ods_orders
        description: "Raw order data"
        columns:
          - name: order_id
            tests: [unique, not_null]
      - name: ods_products
        description: "Raw product data"
```

---

## Layer 1: Staging (stg)

> Tujuan: **Clean, rename, type-cast** — tidak ada business logic

```sql
-- models/staging/stg_users.sql
{{
    config(materialized='view')
}}

WITH source AS (
    SELECT * FROM {{ source('ods_ecommerce', 'ods_users') }}
),

cleaned AS (
    SELECT
        user_id,
        TRIM(username) AS username,
        LOWER(TRIM(email)) AS email,
        COALESCE(TRIM(full_name), 'Unknown') AS full_name,
        COALESCE(phone, '-') AS phone,
        COALESCE(city, 'Unknown') AS city,
        COALESCE(is_active, TRUE) AS is_active,
        created_at,
        updated_at,
        -- Derived
        DATEDIFF(NOW(), created_at) AS account_age_days
    FROM source
    WHERE user_id IS NOT NULL
)

SELECT * FROM cleaned
```

```sql
-- models/staging/stg_orders.sql
{{
    config(materialized='view')
}}

WITH source AS (
    SELECT * FROM {{ source('ods_ecommerce', 'ods_orders') }}
),

cleaned AS (
    SELECT
        order_id,
        user_id,
        product_id,
        TRIM(product_name) AS product_name,
        quantity,
        CAST(unit_price AS DECIMAL(15,2)) AS unit_price,
        CAST(total_price AS DECIMAL(15,2)) AS total_price,
        LOWER(TRIM(order_status)) AS order_status,
        LOWER(TRIM(payment_method)) AS payment_method,
        COALESCE(shipping_city, 'Unknown') AS shipping_city,
        created_at,
        updated_at,
        -- Derived
        DATE(created_at) AS order_date,
        HOUR(created_at) AS order_hour
    FROM source
    WHERE order_id IS NOT NULL
      AND total_price > 0
)

SELECT * FROM cleaned
```

```sql
-- models/staging/stg_products.sql
{{
    config(materialized='view')
}}

SELECT
    product_id,
    TRIM(product_name) AS product_name,
    COALESCE(category, 'Uncategorized') AS category,
    CAST(price AS DECIMAL(15,2)) AS price,
    COALESCE(stock, 0) AS stock,
    COALESCE(is_available, TRUE) AS is_available,
    created_at,
    updated_at
FROM {{ source('ods_ecommerce', 'ods_products') }}
WHERE product_id IS NOT NULL
```

---

## Layer 2: Intermediate (int)

> Tujuan: **Join, enrich, business logic** — bridges staging → marts

```sql
-- models/intermediate/int_orders_enriched.sql
{{
    config(materialized='ephemeral')
}}

WITH orders AS (
    SELECT * FROM {{ ref('stg_orders') }}
),
users AS (
    SELECT * FROM {{ ref('stg_users') }}
),
products AS (
    SELECT * FROM {{ ref('stg_products') }}
),

enriched AS (
    SELECT
        -- Order
        o.order_id,
        o.order_date,
        o.order_hour,
        o.quantity,
        o.unit_price,
        o.total_price,
        o.order_status,
        o.payment_method,
        o.shipping_city,
        o.created_at AS order_created_at,
        
        -- User
        o.user_id,
        u.username,
        u.full_name AS customer_name,
        u.email AS customer_email,
        u.city AS customer_city,
        u.is_active AS is_customer_active,
        u.account_age_days,
        
        -- Product
        o.product_id,
        p.product_name,
        p.category AS product_category,
        p.price AS catalog_price,
        
        -- Business Logic
        CASE
            WHEN o.total_price >= 10000000 THEN 'Premium'
            WHEN o.total_price >= 1000000  THEN 'Standard'
            ELSE 'Basic'
        END AS order_tier,
        
        CASE
            WHEN u.city = o.shipping_city THEN 'Local'
            ELSE 'Inter-city'
        END AS shipping_type,
        
        ROUND(o.total_price / NULLIF(o.quantity, 0), 2) AS effective_unit_price,
        
        CASE
            WHEN o.order_status IN ('completed','shipped') THEN TRUE
            ELSE FALSE
        END AS is_successful_order
        
    FROM orders o
    LEFT JOIN users u ON o.user_id = u.user_id
    LEFT JOIN products p ON o.product_id = p.product_id
)

SELECT * FROM enriched
```

---

## Layer 3: Marts - Facts & Dimensions

### Dimension: dim_users

```sql
-- models/marts/dim_users.sql
{{
    config(
        materialized='table',
        unique_key='user_id',
        tags=['marts', 'daily']
    )
}}

WITH users AS (
    SELECT * FROM {{ ref('stg_users') }}
),

user_metrics AS (
    SELECT
        user_id,
        COUNT(DISTINCT order_id) AS lifetime_orders,
        SUM(total_price) AS lifetime_revenue,
        AVG(total_price) AS avg_order_value,
        MIN(order_date) AS first_order_date,
        MAX(order_date) AS last_order_date
    FROM {{ ref('stg_orders') }}
    WHERE order_status != 'cancelled'
    GROUP BY user_id
),

final AS (
    SELECT
        u.user_id,
        u.username,
        u.email,
        u.full_name,
        u.phone,
        u.city,
        u.is_active,
        u.created_at,
        u.account_age_days,
        
        COALESCE(m.lifetime_orders, 0) AS lifetime_orders,
        COALESCE(m.lifetime_revenue, 0) AS lifetime_revenue,
        COALESCE(m.avg_order_value, 0) AS avg_order_value,
        m.first_order_date,
        m.last_order_date,
        
        -- Customer Segmentation
        CASE
            WHEN m.lifetime_revenue >= 50000000 THEN 'VIP'
            WHEN m.lifetime_revenue >= 10000000 THEN 'Gold'
            WHEN m.lifetime_revenue >= 1000000  THEN 'Silver'
            WHEN m.lifetime_orders > 0          THEN 'Bronze'
            ELSE 'Prospect'
        END AS customer_segment,
        
        CASE
            WHEN m.last_order_date >= DATE_SUB(NOW(), INTERVAL 7 DAY)  THEN 'Active'
            WHEN m.last_order_date >= DATE_SUB(NOW(), INTERVAL 30 DAY) THEN 'Warm'
            WHEN m.last_order_date >= DATE_SUB(NOW(), INTERVAL 90 DAY) THEN 'Cold'
            WHEN m.lifetime_orders > 0 THEN 'Dormant'
            ELSE 'Never Purchased'
        END AS activity_status,
        
        NOW() AS dbt_updated_at
    FROM users u
    LEFT JOIN user_metrics m ON u.user_id = m.user_id
)

SELECT * FROM final
```

### Fact: fct_daily_revenue

```sql
-- models/marts/fct_daily_revenue.sql
{{
    config(
        materialized='table',
        unique_key='revenue_date',
        tags=['marts', 'daily']
    )
}}

WITH orders AS (
    SELECT * FROM {{ ref('int_orders_enriched') }}
    WHERE is_successful_order = TRUE
),

daily AS (
    SELECT
        order_date AS revenue_date,
        
        COUNT(DISTINCT order_id) AS total_orders,
        COUNT(DISTINCT user_id) AS unique_customers,
        SUM(total_price) AS gross_revenue,
        AVG(total_price) AS avg_order_value,
        SUM(quantity) AS total_items_sold,
        
        -- By payment method
        SUM(CASE WHEN payment_method='credit_card' THEN total_price ELSE 0 END) AS cc_revenue,
        SUM(CASE WHEN payment_method='bank_transfer' THEN total_price ELSE 0 END) AS bank_revenue,
        SUM(CASE WHEN payment_method='e_wallet' THEN total_price ELSE 0 END) AS ewallet_revenue,
        
        -- By tier
        COUNT(CASE WHEN order_tier='Premium' THEN 1 END) AS premium_orders,
        COUNT(CASE WHEN order_tier='Standard' THEN 1 END) AS standard_orders,
        COUNT(CASE WHEN order_tier='Basic' THEN 1 END) AS basic_orders,
        
        NOW() AS dbt_updated_at
    FROM orders
    GROUP BY order_date
)

SELECT
    *,
    SUM(gross_revenue) OVER (ORDER BY revenue_date) AS cumulative_revenue,
    LAG(gross_revenue, 1) OVER (ORDER BY revenue_date) AS prev_day_revenue,
    ROUND(
        (gross_revenue - LAG(gross_revenue,1) OVER (ORDER BY revenue_date))
        / NULLIF(LAG(gross_revenue,1) OVER (ORDER BY revenue_date), 0) * 100, 2
    ) AS revenue_growth_pct
FROM daily
ORDER BY revenue_date
```

### Fact: fct_product_performance

```sql
-- models/marts/fct_product_performance.sql
{{
    config(
        materialized='table',
        tags=['marts', 'daily']
    )
}}

SELECT
    product_id,
    product_name,
    product_category,
    COUNT(DISTINCT order_id) AS total_orders,
    SUM(quantity) AS total_units_sold,
    SUM(total_price) AS total_revenue,
    AVG(total_price) AS avg_order_value,
    COUNT(DISTINCT user_id) AS unique_buyers,
    MIN(order_date) AS first_sale_date,
    MAX(order_date) AS last_sale_date,
    
    ROUND(SUM(total_price) / NULLIF(SUM(quantity), 0), 2) AS revenue_per_unit,
    
    NOW() AS dbt_updated_at
FROM {{ ref('int_orders_enriched') }}
WHERE is_successful_order = TRUE
GROUP BY product_id, product_name, product_category
ORDER BY total_revenue DESC
```

---

## Testing & Data Quality

```yaml
# models/marts/_marts_schema.yml
version: 2

models:
  - name: dim_users
    description: "User dimension with segments"
    columns:
      - name: user_id
        tests: [unique, not_null]
      - name: customer_segment
        tests:
          - accepted_values:
              values: ['VIP','Gold','Silver','Bronze','Prospect']
      - name: lifetime_revenue
        tests:
          - dbt_utils.accepted_range:
              min_value: 0

  - name: fct_daily_revenue
    description: "Daily revenue facts"
    columns:
      - name: revenue_date
        tests: [unique, not_null]
      - name: gross_revenue
        tests: [not_null]
      - name: total_orders
        tests:
          - dbt_utils.accepted_range:
              min_value: 0
```

```sql
-- tests/assert_revenue_matches_orders.sql
-- Custom test: total revenue harus match

WITH revenue AS (
    SELECT SUM(gross_revenue) AS total FROM {{ ref('fct_daily_revenue') }}
),
orders AS (
    SELECT SUM(total_price) AS total
    FROM {{ ref('stg_orders') }}
    WHERE order_status IN ('completed', 'shipped')
)
SELECT *
FROM revenue r, orders o
WHERE ABS(r.total - o.total) > 1  -- tolerance 1 IDR
```

---

## Incremental Models

```sql
-- models/marts/fct_orders_incremental.sql
{{
    config(
        materialized='incremental',
        unique_key='order_id',
        incremental_strategy='append'
    )
}}

SELECT
    order_id, user_id, product_name, quantity,
    total_price, order_status, order_date,
    order_tier, shipping_type, customer_name,
    NOW() AS dbt_loaded_at
FROM {{ ref('int_orders_enriched') }}

{% if is_incremental() %}
    WHERE updated_at > (SELECT MAX(dbt_loaded_at) FROM {{ this }})
{% endif %}
```

---

## Macros & Reusable Code

```sql
-- macros/format_currency.sql
{% macro format_idr(column) %}
    CONCAT('Rp ', FORMAT_NUMBER({{ column }}, 0))
{% endmacro %}

-- macros/date_diff_days.sql
{% macro days_between(start_date, end_date) %}
    DATEDIFF({{ end_date }}, {{ start_date }})
{% endmacro %}
```

---

## Running DBT

```bash
# Run semua models
dbt run

# Run specific layer
dbt run --select staging.*
dbt run --select marts.*

# Run with dependencies
dbt run --select +fct_daily_revenue    # model + upstream
dbt run --select dim_users+            # model + downstream

# Test
dbt test
dbt test --select fct_daily_revenue

# Generate docs
dbt docs generate
dbt docs serve --port 8888

# Full build (run + test)
dbt build

# Preview output
dbt show --select fct_daily_revenue --limit 10
```

---

## Troubleshooting

| Problem | Solusi |
|---------|--------|
| `Source not found` | Cek database name di `_sources.yml` |
| `Doris syntax error` | Doris SQL sedikit berbeda dari MySQL, cek docs |
| `Incremental not working` | Pastikan `unique_key` ada di Doris table |
| `Materialization error` | Doris support: `table`, `view`, `incremental` |
| `Permission denied` | Grant user di Doris: `GRANT ALL ON db.* TO user` |

---

## 📚 Referensi

- [DBT Documentation](https://docs.getdbt.com/)
- [DBT Doris Adapter](https://github.com/apache/doris/tree/master/extension/dbt-doris)
- [Apache Doris SQL Reference](https://doris.apache.org/docs/sql-manual/)

---

> 📝 **Author**: Data Engineering Team  
> 📅 **Last Updated**: 2026-02-10  
> 🏷️ **Tags**: `dbt`, `doris`, `data-transformation`, `data-warehouse`, `elt`
