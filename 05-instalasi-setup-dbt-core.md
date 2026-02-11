# 🔧 Cara Instalasi & Setup DBT Core

> **DBT (Data Build Tool) Core** adalah open-source tool untuk data transformation di dalam data warehouse. DBT memungkinkan data analysts dan engineers menulis transformasi data menggunakan SQL, dan mengelolanya seperti software engineering (version control, testing, documentation).

---

## 📋 Table of Contents

- [Prerequisites](#prerequisites)
- [Konsep DBT](#konsep-dbt)
- [Instalasi DBT Core](#instalasi-dbt-core)
  - [Install via pip](#install-via-pip)
  - [Install dengan Adapter Spesifik](#daftar-install-dengan-adapter-spesifik)
- [Inisialisasi Project DBT](#inisialisasi-project-dbt)
- [Konfigurasi profiles.yml](#konfigurasi-profilesyml)
- [Struktur Project DBT](#struktur-project-dbt)
- [Membuat Model](#membuat-model)
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
#buat folder khusus untuk dbt
mkdir dbt
cd dbt/

# Install Python 3.10 (Ubuntu)
sudo apt update
sudo apt install python3.13-venv

# Verify
python3 --version
pip3 --version

# Buat virtual environment (RECOMMENDED)
python3 -m venv ~/dbt-env
source ~/dbt-env/bin/activate
```
---


## Konsep DBT

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
│       │                │                    │                │
│       ▼                ▼                    ▼                │
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

---

## Instalasi DBT Core

### Install via pip

```bash
# Pastikan virtual environment sudah aktif
source ~/dbt-env/bin/activate

# Install dbt-core
pip install dbt-core dbt doris

# Verify
dbt --version
# Output:
# Core:
#   - installed: 1.7.x
#   - latest:    1.7.x - Up to date!
```

### Daftar Install dengan Adapter Spesifik

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
Langkah ini menghubungkan dbt (alat transformasi) dengan Apache Doris (tempat penyimpanan data) agar dbt memiliki izin untuk membaca, mengubah, dan membuat tabel di dalamnya.

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
      host: 192.168.4.100
      password: ''
      port: 9030
      schema: dbt
      threads: 1
      type: doris
      username: root
      target: dev
```
Setelah memastikan isi file profiles.yml sudah sesuai, maka langkah selanjutnya test koneksi antara dbt dan doris dengan melakukan perintah dibawah ini

```bash
# Test koneksi
dbt debug

# Output yang diharapkan:
# ✅ Connection test: [OK connection ok]
# ✅ All checks passed!
```

---

## Struktur Project DBT

### dbt_project.yml
File ini adalah pusat pengaturan proyek. Di sini kita memberitahu dbt bagaimana cara memperlakukan model-model kamu secara umum.

```yaml
# Name your project! Project names should contain only lowercase characters
# and underscores. A good package name should reflect your organization's
# name or the intended use of these models
name: 'my_data_warehouse'
version: '1.0.0'

# This setting configures which "profile" dbt uses for this project.
profile: 'my_data_warehouse'

# These configurations specify where dbt should look for different types of files.
# The `model-paths` config, for example, states that models in this project can be
# found in the "models/" directory. You probably won't need to change these!
model-paths: ["models"]
analysis-paths: ["analyses"]
test-paths: ["tests"]
seed-paths: ["seeds"]
macro-paths: ["macros"]
snapshot-paths: ["snapshots"]

clean-targets:         # directories to be removed by `dbt clean`
  - "target"
  - "dbt_packages"


# Configuring models
# Full documentation: https://docs.getdbt.com/docs/configuring-models

# In this example config, we tell dbt to build all models in the example/
# directory as views. These settings can be overridden in the individual model
# files using the `{{ config(...) }}` macro.

models:
  my_doris_project:
    # Ini akan berlaku untuk SEMUA model di dalam project ini
    +replication_num: "1"
    +materialized: table

    # Anda bisa menghapus blok bawah ini jika ingin settingnya merata
    example:
      +materialized: view # Misal khusus folder example jadi view
```
---

### resources.yml
File sources.yml berfungsi sebagai definisi sumber data mentah (raw data). File ini memetakan nama tabel fisik yang ada di database (Apache Doris) ke dalam nama logis yang bisa dipanggil oleh dbt.
Jika file sources.yml belum ada, maka buat terlebih dahulu 

```bash
#masuk ke folder project dbt
cd ~/dbt/my_doris_project

#buat folder khusus untuk sources.yml
mkdir staging

#buat file sources.yml
vim sources.yml
```
Setelah masuk ke dalam tampilan new file, masukkan daftar schema dan table yang ingin digunakan. Dibawah ini adalah contoh file sources.yml 

```bash                                   
version: 2

sources:
  - name: doris_ap
    schema: ta15_wlogisticsidb_ap
    tables:
      - name: ENTBT
      - name: ENTKENDARAAN
      - name: ENTLV
      - name: ENTBA
      - name: ENTBE
      - name: ENTBI
      - name: ENTCAREA
      - name: ENTCDNODE
      - name: ENTBTTY
      - name: supWilByBPS

  - name: doris_dw
    schema: ta15_wlogisticsidb_dw
    tables:
      - name: ENTOBJPROP

  - name: source_logistik
    schema: db_logistik
    tables:
      - name: shipment
      - name: tracking

  - name: doris_or
    schema: orders
    tables:
      - name: users
      - name: products
      - name: orders
```

### Recommended Folder Structure

```
my_data_warehouse/
├── dbt_project.yml       # Project configuration
├── models/               # SQL models
│   └── example/
│       ├── my_first_dbt_model.sql
│       ├── my_second_dbt_model.sql
│       └── schema.yml
├── seeds/                # CSV seed files
├── snapshots/            # SCD snapshots
├── tests/                # Custom tests
├── macros/               # Reusable SQL functions
├── analyses/             # Ad-hoc analyses
├── staging/
|       └──  sources.yml   
└── README.md
```
---

## Membuat Model
Buat model di folder models/
```bash
#masuk ke folder
cd dbt/my_data_warehouse/models/

#buat file sql
vim orders.sql
```

### orders.sql

```yaml
# models/orders.sql
{{ config(
    materialized='incremental',
    unique_key='order_id',
    engine='OLAP'
) }}

WITH source_orders AS (
    SELECT * FROM {{ source('doris_or', 'orders') }}
),

source_users AS (
    SELECT * FROM {{ source('doris_or', 'users') }}
),

source_products AS (
    SELECT * FROM {{ source('doris_or', 'products') }}
)

SELECT
    o.order_id,
    o.order_date,
    o.user_id,
    u.username,
    u.city,
    o.product_id,
    p.product_name,
    p.category,
    p.price,
    (o.quantity * p.price) as total_amount,
    CURRENT_TIMESTAMP() as dbt_update_at

FROM source_orders o
LEFT JOIN source_users u ON o.user_id = u.user_id
LEFT JOIN source_products p ON o.product_id = p.product_id

{% if is_incremental() %}
    WHERE o.order_date > (SELECT MAX(order_date) FROM {{ this }})
{% endif %}

```

## Runing DBT

```bash
# 1. Cek koneksi ke Doris (Wajib OK semua)
dbt debug

# 2. Jalankan semua model
dbt run

# 3. Menjalankan model secara spesifik
dbt run --select shipment.sql

# 4. Jalankan model spesifik & refresh total (hapus tabel lama, buat baru)
dbt run --select shipment --full-refresh

# 📄 Documentation
dbt docs generate            # Generate documentation
dbt docs serve --port 8080   # Serve docs di browser
```

---


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

---

## 📚 Referensi

- [DBT Documentation](https://docs.getdbt.com/)
- [DBT Best Practices](https://docs.getdbt.com/guides/best-practices)
- [DBT Doris Adapter](https://github.com/apache/doris/tree/master/extension/dbt-doris)
- [DBT Package Hub](https://hub.getdbt.com/)
- [DBT Learn (Free Course)](https://courses.getdbt.com/)