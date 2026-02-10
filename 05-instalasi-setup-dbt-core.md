# 🔧 Cara Instalasi & Setup DBT Core

> **DBT (Data Build Tool) Core** adalah open-source tool untuk data transformation di dalam data warehouse. DBT memungkinkan data analysts dan engineers menulis transformasi data menggunakan SQL, dan mengelolanya seperti software engineering (version control, testing, documentation).

---

## 📋 Table of Contents

- [Prerequisites](#prerequisites)
- [Konsep DBT](#konsep-dbt)
- [Instalasi DBT Core](#instalasi-dbt-core)
  - [Install via pip](#install-via-pip)
  - [Install dengan Adapter Spesifik](#install-dengan-adapter-spesifik)
- [Inisialisasi Project DBT](#inisialisasi-project-dbt)
- [Konfigurasi profiles.yml](#konfigurasi-profilesyml)
- [Struktur Project DBT](#struktur-project-dbt)
- [Membuat Model](#membuat-model)
- [Testing & Documentation](#testing--documentation)
- [DBT Commands](#dbt-commands)
- [Best Practices](#best-practices)
- [Troubleshooting](#troubleshooting)

---

## Prerequisites

| Komponen         | Minimum Requirement        |
|------------------|----------------------------|
| **OS**           | Ubuntu 20.04+ / macOS / Windows |
| **Python**       | 3.8+ (recommended 3.10+)  |
| **pip**          | 21.0+                      |
| **Git**          | 2.0+                       |
| **Data Warehouse** | Apache Doris / PostgreSQL / BigQuery / Snowflake |

### Install Python & pip

```bash
# Install Python 3.10 (Ubuntu)
sudo apt update
sudo apt install -y python3.10 python3.10-venv python3-pip git

# Verify
python3 --version
pip3 --version

# Buat virtual environment (RECOMMENDED)
python3 -m venv ~/dbt-env
source ~/dbt-env/bin/activate

# Verify venv aktif
which python
# Output: /home/user/dbt-env/bin/python
```

---

## Konsep DBT

```
┌──────────────────────────────────────────────────────────────┐
│                      DBT Workflow                            │
│                                                              │
│  ┌───────────┐    ┌───────────┐    ┌──────────────────────┐  │
│  │ Raw Data  │    │  Staging  │    │    Marts / Final     │  │
│  │ (Sources) │───▶│  Models   │───▶│      Models          │  │
│  │           │    │           │    │                      │  │
│  │ • tables  │    │ • stg_*   │    │ • fct_* (facts)      │  │
│  │ • views   │    │ • cleaned │    │ • dim_* (dimensions) │  │
│  │ • streams │    │ • renamed │    │ • agg_* (aggregates) │  │
│  └───────────┘    └───────────┘    └──────────────────────┘  │
│       │                │                    │                 │
│       ▼                ▼                    ▼                 │
│  ┌─────────────────────────────────────────────────────┐     │
│  │              dbt Features                           │     │
│  │  • Tests (data quality)                             │     │
│  │  • Documentation (auto-generated)                   │     │
│  │  • Lineage Graph (data flow visualization)          │     │
│  │  • Snapshots (SCD Type 2)                           │     │
│  │  • Seeds (CSV reference data)                       │     │
│  │  • Macros (reusable SQL functions)                  │     │
│  └─────────────────────────────────────────────────────┘     │
└──────────────────────────────────────────────────────────────┘
```

---

## Instalasi DBT Core

### Install via pip

```bash
# Pastikan virtual environment sudah aktif
source ~/dbt-env/bin/activate

# Install dbt-core
pip install dbt-core

# Verify
dbt --version
# Output:
# Core:
#   - installed: 1.7.x
#   - latest:    1.7.x - Up to date!
```

### Install dengan Adapter Spesifik

```bash
# ===========================
# 🎯 Untuk Apache Doris
# ===========================
pip install dbt-doris

# ===========================
# 🐘 Untuk PostgreSQL
# ===========================
pip install dbt-postgres

# ===========================
# ❄️ Untuk Snowflake
# ===========================
pip install dbt-snowflake

# ===========================
# 🔵 Untuk BigQuery
# ===========================
pip install dbt-bigquery

# ===========================
# 🔴 Untuk MySQL
# ===========================
pip install dbt-mysql

# ===========================
# 📦 Install multiple adapters
# ===========================
pip install dbt-core dbt-doris dbt-postgres

# Verify semua plugins
dbt --version
```

---

## Inisialisasi Project DBT

```bash
# Buat project baru
dbt init my_data_warehouse

# Output interaktif:
# Which database would you like to use?
# [1] doris
# [2] postgres
# ...
# Enter a number: 1

# Masuk ke project
cd my_data_warehouse

# Struktur yang dibuat:
# my_data_warehouse/
# ├── dbt_project.yml       # Project configuration
# ├── models/               # SQL models
# │   └── example/
# │       ├── my_first_dbt_model.sql
# │       ├── my_second_dbt_model.sql
# │       └── schema.yml
# ├── seeds/                # CSV seed files
# ├── snapshots/            # SCD snapshots
# ├── tests/                # Custom tests
# ├── macros/               # Reusable SQL functions
# ├── analyses/             # Ad-hoc analyses
# └── README.md
```

---

## Konfigurasi profiles.yml

File `profiles.yml` berisi koneksi ke database. Lokasi default: `~/.dbt/profiles.yml`

```yaml
# ~/.dbt/profiles.yml

# ===========================
# Profile untuk Apache Doris
# ===========================
my_data_warehouse:
  target: dev
  outputs:
    dev:
      type: doris
      host: 127.0.0.1
      port: 9030
      username: root
      password: ""
      schema: demo_db        # Database name di Doris
      threads: 4

    staging:
      type: doris
      host: staging-doris.internal
      port: 9030
      username: dbt_user
      password: "{{ env_var('DBT_DORIS_PASSWORD') }}"
      schema: staging_db
      threads: 4

    production:
      type: doris
      host: prod-doris.internal
      port: 9030
      username: dbt_prod
      password: "{{ env_var('DBT_DORIS_PROD_PASSWORD') }}"
      schema: production_db
      threads: 8

# ===========================
# Profile untuk PostgreSQL
# ===========================
postgres_warehouse:
  target: dev
  outputs:
    dev:
      type: postgres
      host: localhost
      port: 5432
      user: dbt_user
      password: "{{ env_var('DBT_PG_PASSWORD') }}"
      dbname: analytics
      schema: public
      threads: 4
      keepalives_idle: 0
      connect_timeout: 10
      search_path: public,staging
      role: dbt_role
      sslmode: prefer
```

```bash
# Set environment variables (jika pakai env_var)
export DBT_DORIS_PASSWORD="my_secret_password"
export DBT_DORIS_PROD_PASSWORD="prod_secret_password"

# Test koneksi
dbt debug

# Output yang diharapkan:
# ✅ Connection test: [OK connection ok]
# ✅ All checks passed!
```

---

## Struktur Project DBT

### dbt_project.yml

```yaml
# dbt_project.yml
name: 'my_data_warehouse'
version: '1.0.0'
config-version: 2

# Profile yang digunakan (harus match dengan profiles.yml)
profile: 'my_data_warehouse'

# Lokasi file
model-paths: ["models"]
analysis-paths: ["analyses"]
test-paths: ["tests"]
seed-paths: ["seeds"]
macro-paths: ["macros"]
snapshot-paths: ["snapshots"]
asset-paths: ["assets"]

# Konfigurasi model
models:
  my_data_warehouse:
    # Materialization default
    +materialized: view

    # Staging layer
    staging:
      +materialized: view
      +schema: staging
      +tags: ['staging', 'daily']

    # Intermediate layer
    intermediate:
      +materialized: ephemeral
      +tags: ['intermediate']

    # Marts layer (final tables)
    marts:
      +materialized: table
      +schema: marts
      +tags: ['marts', 'production']
      
      finance:
        +schema: marts_finance
      marketing:
        +schema: marts_marketing

# Seeds configuration
seeds:
  my_data_warehouse:
    +schema: seeds

# Snapshots configuration
snapshots:
  my_data_warehouse:
    +target_schema: snapshots

# Variables
vars:
  start_date: '2024-01-01'
  currency: 'IDR'
  is_test_run: false
```

### Recommended Folder Structure

```
my_data_warehouse/
├── dbt_project.yml
├── packages.yml              # External packages
├── models/
│   ├── staging/              # Layer 1: Clean & rename
│   │   ├── stg_ecommerce/
│   │   │   ├── _stg_ecommerce__models.yml   # Schema + tests
│   │   │   ├── _stg_ecommerce__sources.yml  # Source definitions
│   │   │   ├── stg_ecommerce__users.sql
│   │   │   ├── stg_ecommerce__orders.sql
│   │   │   └── stg_ecommerce__products.sql
│   │   └── stg_payments/
│   │       ├── _stg_payments__models.yml
│   │       └── stg_payments__transactions.sql
│   │
│   ├── intermediate/         # Layer 2: Business logic joins
│   │   ├── int_orders_enriched.sql
│   │   └── int_user_order_summary.sql
│   │
│   └── marts/                # Layer 3: Final tables for BI
│       ├── finance/
│       │   ├── _finance__models.yml
│       │   ├── fct_revenue.sql
│       │   └── dim_payment_methods.sql
│       └── marketing/
│           ├── _marketing__models.yml
│           ├── fct_user_activity.sql
│           └── dim_users.sql
│
├── seeds/
│   ├── country_codes.csv
│   └── product_categories.csv
│
├── snapshots/
│   └── scd_users.sql
│
├── tests/
│   └── assert_positive_revenue.sql
│
├── macros/
│   ├── generate_schema_name.sql
│   ├── cents_to_idr.sql
│   └── date_spine.sql
│
└── analyses/
    └── ad_hoc_user_analysis.sql
```

---

## Membuat Model

### Sources (Definisi Tabel Sumber)

```yaml
# models/staging/stg_ecommerce/_stg_ecommerce__sources.yml
version: 2

sources:
  - name: ecommerce
    description: "Database e-commerce utama"
    database: ecommerce_db
    schema: public
    
    freshness:
      warn_after: { count: 12, period: hour }
      error_after: { count: 24, period: hour }
    loaded_at_field: updated_at
    
    tables:
      - name: users
        description: "Tabel data user"
        identifier: users  # Nama tabel asli di database  
        columns:
          - name: user_id
            description: "Primary key"
            tests:
              - unique
              - not_null

      - name: orders
        description: "Tabel data order"
        columns:
          - name: order_id
            tests:
              - unique
              - not_null
          - name: user_id
            tests:
              - not_null
              - relationships:
                  to: source('ecommerce', 'users')
                  field: user_id

      - name: products
        description: "Tabel master produk"
```

### Staging Models

```sql
-- models/staging/stg_ecommerce/stg_ecommerce__users.sql

{{
    config(
        materialized='view',
        tags=['staging', 'daily']
    )
}}

WITH source AS (
    SELECT * FROM {{ source('ecommerce', 'users') }}
),

renamed AS (
    SELECT
        -- 🔑 Primary Key
        user_id,
        
        -- 👤 User Info
        TRIM(username) AS username,
        LOWER(TRIM(email)) AS email,
        age,
        
        -- 📍 Location
        COALESCE(city, 'Unknown') AS city,
        
        -- 📅 Timestamps
        signup_date,
        last_login AS last_login_at,
        
        -- 🏷️ Status
        CASE 
            WHEN is_active = 1 THEN TRUE 
            ELSE FALSE 
        END AS is_active,
        
        -- 🧮 Computed
        DATEDIFF(CURRENT_DATE(), signup_date) AS days_since_signup
        
    FROM source
    WHERE user_id IS NOT NULL
)

SELECT * FROM renamed
```

```sql
-- models/staging/stg_ecommerce/stg_ecommerce__orders.sql

{{
    config(
        materialized='view'
    )
}}

WITH source AS (
    SELECT * FROM {{ source('ecommerce', 'orders') }}
),

cleaned AS (
    SELECT
        -- 🔑 Keys
        order_id,
        user_id,
        
        -- 📦 Order Details
        product_name,
        quantity,
        CAST(total_price AS DECIMAL(15, 2)) AS total_price,
        
        -- 🏷️ Status
        LOWER(TRIM(order_status)) AS order_status,
        CASE LOWER(TRIM(order_status))
            WHEN 'completed' THEN 3
            WHEN 'shipped' THEN 2
            WHEN 'processing' THEN 1
            WHEN 'pending' THEN 0
            WHEN 'cancelled' THEN -1
            ELSE -99
        END AS order_status_rank,
        
        -- 📅 Timestamps
        created_at,
        updated_at,
        
        -- 🧮 Computed
        CASE WHEN updated_at IS NOT NULL 
            THEN TIMESTAMPDIFF(HOUR, created_at, updated_at) 
            ELSE NULL 
        END AS processing_hours
        
    FROM source
    WHERE order_id IS NOT NULL
)

SELECT * FROM cleaned
```

### Intermediate Models

```sql
-- models/intermediate/int_orders_enriched.sql

{{
    config(
        materialized='ephemeral'
    )
}}

-- Ephemeral = tidak dibuat tabel, hanya CTE yang di-inject ke model downstream

WITH orders AS (
    SELECT * FROM {{ ref('stg_ecommerce__orders') }}
),

users AS (
    SELECT * FROM {{ ref('stg_ecommerce__users') }}
),

enriched AS (
    SELECT
        o.order_id,
        o.user_id,
        u.username,
        u.email,
        u.city,
        o.product_name,
        o.quantity,
        o.total_price,
        o.order_status,
        o.order_status_rank,
        o.created_at,
        o.updated_at,
        o.processing_hours,
        u.is_active AS is_user_active,
        u.days_since_signup,
        
        -- 🏷️ Order tier
        CASE
            WHEN o.total_price >= 10000000 THEN 'premium'
            WHEN o.total_price >= 1000000 THEN 'standard'
            ELSE 'basic'
        END AS order_tier
        
    FROM orders o
    LEFT JOIN users u ON o.user_id = u.user_id
)

SELECT * FROM enriched
```

### Mart Models (Final)

```sql
-- models/marts/finance/fct_revenue.sql

{{
    config(
        materialized='table',
        tags=['finance', 'daily'],
        unique_key='revenue_date'
    )
}}

WITH order_data AS (
    SELECT * FROM {{ ref('int_orders_enriched') }}
    WHERE order_status = 'completed'
),

daily_revenue AS (
    SELECT
        DATE(created_at) AS revenue_date,
        
        -- 📊 Revenue Metrics
        COUNT(DISTINCT order_id) AS total_orders,
        COUNT(DISTINCT user_id) AS unique_customers,
        SUM(total_price) AS gross_revenue,
        AVG(total_price) AS avg_order_value,
        MIN(total_price) AS min_order_value,
        MAX(total_price) AS max_order_value,
        
        -- 📊 By Tier
        SUM(CASE WHEN order_tier = 'premium' THEN total_price ELSE 0 END) AS premium_revenue,
        SUM(CASE WHEN order_tier = 'standard' THEN total_price ELSE 0 END) AS standard_revenue,
        SUM(CASE WHEN order_tier = 'basic' THEN total_price ELSE 0 END) AS basic_revenue,
        
        -- 📊 Item Metrics
        SUM(quantity) AS total_items_sold,
        AVG(CAST(quantity AS DOUBLE)) AS avg_items_per_order
        
    FROM order_data
    GROUP BY DATE(created_at)
)

SELECT
    *,
    -- 📈 Running totals
    SUM(gross_revenue) OVER (ORDER BY revenue_date) AS cumulative_revenue,
    -- 📈 Growth
    LAG(gross_revenue, 1) OVER (ORDER BY revenue_date) AS prev_day_revenue,
    ROUND(
        (gross_revenue - LAG(gross_revenue, 1) OVER (ORDER BY revenue_date)) 
        / NULLIF(LAG(gross_revenue, 1) OVER (ORDER BY revenue_date), 0) * 100,
        2
    ) AS revenue_growth_pct

FROM daily_revenue
ORDER BY revenue_date
```

```sql
-- models/marts/marketing/dim_users.sql

{{
    config(
        materialized='table',
        tags=['marketing', 'daily']
    )
}}

WITH users AS (
    SELECT * FROM {{ ref('stg_ecommerce__users') }}
),

user_orders AS (
    SELECT
        user_id,
        COUNT(DISTINCT order_id) AS lifetime_orders,
        SUM(total_price) AS lifetime_revenue,
        AVG(total_price) AS avg_order_value,
        MIN(created_at) AS first_order_at,
        MAX(created_at) AS last_order_at,
        DATEDIFF(MAX(created_at), MIN(created_at)) AS customer_tenure_days
    FROM {{ ref('stg_ecommerce__orders') }}
    WHERE order_status != 'cancelled'
    GROUP BY user_id
),

final AS (
    SELECT
        u.user_id,
        u.username,
        u.email,
        u.city,
        u.age,
        u.signup_date,
        u.last_login_at,
        u.is_active,
        u.days_since_signup,
        
        -- 💰 Order metrics
        COALESCE(uo.lifetime_orders, 0) AS lifetime_orders,
        COALESCE(uo.lifetime_revenue, 0) AS lifetime_revenue,
        COALESCE(uo.avg_order_value, 0) AS avg_order_value,
        uo.first_order_at,
        uo.last_order_at,
        
        -- 🏷️ Customer Segments
        CASE
            WHEN uo.lifetime_revenue >= 50000000 THEN 'VIP'
            WHEN uo.lifetime_revenue >= 10000000 THEN 'Gold'
            WHEN uo.lifetime_revenue >= 1000000 THEN 'Silver'
            WHEN uo.lifetime_orders > 0 THEN 'Bronze'
            ELSE 'Prospect'
        END AS customer_segment,
        
        -- 🏷️ Activity Status
        CASE
            WHEN u.last_login_at >= DATE_SUB(NOW(), INTERVAL 7 DAY) THEN 'Active'
            WHEN u.last_login_at >= DATE_SUB(NOW(), INTERVAL 30 DAY) THEN 'Warm'
            WHEN u.last_login_at >= DATE_SUB(NOW(), INTERVAL 90 DAY) THEN 'Cold'
            ELSE 'Dormant'
        END AS activity_status
        
    FROM users u
    LEFT JOIN user_orders uo ON u.user_id = uo.user_id
)

SELECT * FROM final
```

### Schema & Tests

```yaml
# models/marts/finance/_finance__models.yml
version: 2

models:
  - name: fct_revenue
    description: "Fact table: daily revenue metrics"
    columns:
      - name: revenue_date
        description: "Tanggal revenue"
        tests:
          - unique
          - not_null
      - name: gross_revenue
        description: "Total gross revenue per hari"
        tests:
          - not_null
          - dbt_utils.accepted_range:
              min_value: 0
      - name: total_orders
        tests:
          - not_null
          - dbt_utils.accepted_range:
              min_value: 0

  - name: dim_users
    description: "Dimension table: user profiles with metrics"
    columns:
      - name: user_id
        tests:
          - unique
          - not_null
      - name: customer_segment
        tests:
          - accepted_values:
              values: ['VIP', 'Gold', 'Silver', 'Bronze', 'Prospect']
```

### Macros (Reusable Functions)

```sql
-- macros/cents_to_idr.sql
{% macro cents_to_idr(column_name, decimal_places=2) %}
    ROUND(CAST({{ column_name }} AS DECIMAL(15, {{ decimal_places }})), {{ decimal_places }})
{% endmacro %}

-- macros/generate_schema_name.sql
-- Override default schema naming
{% macro generate_schema_name(custom_schema_name, node) %}
    {% if custom_schema_name is not none %}
        {{ custom_schema_name | trim }}
    {% else %}
        {{ target.schema }}
    {% endif %}
{% endmacro %}

-- macros/date_utils.sql
{% macro get_date_range(start_date, end_date) %}
    BETWEEN '{{ start_date }}' AND '{{ end_date }}'
{% endmacro %}

{% macro is_incremental_run() %}
    {% if is_incremental() %}
        TRUE
    {% else %}
        FALSE
    {% endif %}
{% endmacro %}
```

### Seeds

```csv
-- seeds/product_categories.csv
category_id,category_name,department
1,Laptop,Electronics
2,Keyboard,Accessories
3,Mouse,Accessories
4,Monitor,Electronics
5,USB Hub,Accessories
6,Headset,Audio
7,Webcam,Peripherals
```

```bash
# Load seed data
dbt seed

# Load specific seed
dbt seed --select product_categories
```

---

## DBT Commands

```bash
# ===========================
# 🔍 Debug & Test Connection
# ===========================
dbt debug                    # Test connection & project setup
dbt deps                     # Install packages dari packages.yml

# ===========================
# 🏗️ Run Models
# ===========================
dbt run                      # Run semua models
dbt run --select stg_ecommerce__users       # Run 1 model
dbt run --select staging.*                   # Run semua di folder staging
dbt run --select +fct_revenue               # Run model + semua upstream
dbt run --select fct_revenue+               # Run model + semua downstream
dbt run --select tag:finance                 # Run by tag
dbt run --exclude stg_ecommerce__products   # Exclude model tertentu
dbt run --full-refresh                       # Drop & recreate tables

# ===========================
# 🧪 Testing
# ===========================
dbt test                     # Run semua tests
dbt test --select stg_ecommerce__users      # Test 1 model
dbt test --select source:ecommerce          # Test sources

# ===========================
# 📄 Documentation
# ===========================
dbt docs generate            # Generate documentation
dbt docs serve --port 8080   # Serve docs di browser

# ===========================
# 📷 Snapshots
# ===========================
dbt snapshot                 # Run snapshots (SCD Type 2)

# ===========================
# 🌱 Seeds
# ===========================
dbt seed                     # Load semua CSV seeds
dbt seed --select country_codes  # Load specific seed

# ===========================
# 📊 Source Freshness
# ===========================
dbt source freshness         # Check source data freshness

# ===========================
# 🔄 Build (run + test)
# ===========================
dbt build                    # Run models + tests + snapshots + seeds
dbt build --select staging+  # Build staging dan downstream

# ===========================
# 📋 List & Show
# ===========================
dbt list                     # List semua resources
dbt list --select tag:finance  # List by tag
dbt show --select fct_revenue --limit 10  # Preview model output

# ===========================
# 🧹 Clean
# ===========================
dbt clean                    # Clean compiled files & target directory
```

---

## Best Practices

### 1. Naming Convention

```
📁 staging/    → stg_<source>__<table>.sql    (e.g., stg_ecommerce__orders.sql)
📁 intermediate/ → int_<description>.sql       (e.g., int_orders_enriched.sql)
📁 marts/      → fct_<entity>.sql              (e.g., fct_revenue.sql)
                 dim_<entity>.sql              (e.g., dim_users.sql)
                 agg_<description>.sql         (e.g., agg_monthly_sales.sql)
```

### 2. Packages (packages.yml)

```yaml
# packages.yml
packages:
  - package: dbt-labs/dbt_utils
    version: "1.1.1"
  
  - package: dbt-labs/codegen
    version: "0.12.1"

  - package: calogica/dbt_expectations
    version: "0.10.3"
```

```bash
# Install packages
dbt deps
```

### 3. Jinja Tips

```sql
-- Conditional logic
{% if target.name == 'dev' %}
    LIMIT 1000
{% endif %}

-- Loop
{% set payment_methods = ['credit_card', 'bank_transfer', 'e_wallet'] %}
{% for method in payment_methods %}
    SUM(CASE WHEN payment_method = '{{ method }}' THEN amount ELSE 0 END) 
        AS {{ method }}_amount
    {% if not loop.last %},{% endif %}
{% endfor %}

-- Variables
{{ var('start_date', '2024-01-01') }}
```

---

## Troubleshooting

| Problem | Solusi |
|---------|--------|
| `Could not find profile` | Cek `~/.dbt/profiles.yml` dan nama profile di `dbt_project.yml` |
| `Connection refused` | Pastikan database running dan port benar |
| `Compilation Error` | Cek Jinja syntax, ref() name, dan source() name |
| `Database Error` | Cek SQL syntax, materialization type, dan permissions |
| `Test failed` | Jalankan `dbt show --select <model>` untuk debug data |
| `dbt-doris not found` | Install adapter: `pip install dbt-doris` |

```bash
# Debug tips
dbt debug --config-dir
dbt compile --select my_model  # Lihat compiled SQL
cat target/compiled/my_data_warehouse/models/marts/fct_revenue.sql
```

---

## 📚 Referensi

- [DBT Documentation](https://docs.getdbt.com/)
- [DBT Best Practices](https://docs.getdbt.com/guides/best-practices)
- [DBT Doris Adapter](https://github.com/apache/doris/tree/master/extension/dbt-doris)
- [DBT Package Hub](https://hub.getdbt.com/)
- [DBT Learn (Free Course)](https://courses.getdbt.com/)

---

> 📝 **Author**: Data Engineering Team  
> 📅 **Last Updated**: 2026-02-10  
> 🏷️ **Tags**: `dbt`, `data-transformation`, `sql`, `data-modeling`, `elt`
