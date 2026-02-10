# 🔄 Cara Migrasi & Realtime Data Menggunakan Apache Flink

> Panduan menggunakan **Apache Flink CDC** untuk migrasi data (batch) dan real-time streaming dari MySQL/PostgreSQL ke **Apache Doris**.

---

## 📋 Table of Contents

- [Arsitektur Pipeline](#arsitektur-pipeline)
- [Prerequisites & Setup](#prerequisites--setup)
- [Skenario 1: Migrasi Full Data (Batch)](#skenario-1-migrasi-full-data-batch)
- [Skenario 2: Real-time CDC dari MySQL](#skenario-2-real-time-cdc-dari-mysql)
- [Skenario 3: Multi-Table Sync](#skenario-3-multi-table-sync)
- [Skenario 4: Real-time Aggregation](#skenario-4-real-time-aggregation)
- [Monitoring & Troubleshooting](#monitoring--troubleshooting)

---

## Arsitektur Pipeline

```
┌──────────────┐    ┌──────────────────┐    ┌──────────────────┐
│ Source DBs    │    │  Apache Flink    │    │  Apache Doris    │
│              │    │                  │    │                  │
│ ┌──────────┐ │    │ ┌──────────────┐ │    │ ┌──────────────┐ │
│ │  MySQL   │─┼───▶│ │ Flink CDC    │─┼───▶│ │  ODS Layer   │ │
│ │ (binlog) │ │    │ │ Source       │ │    │ │ (raw tables) │ │
│ └──────────┘ │    │ └──────────────┘ │    │ └──────────────┘ │
│ ┌──────────┐ │    │ ┌──────────────┐ │    │ ┌──────────────┐ │
│ │ Postgres │─┼───▶│ │ Transform +  │─┼───▶│ │  DWS Layer   │ │
│ │  (WAL)   │ │    │ │ Aggregation  │ │    │ │ (aggregated) │ │
│ └──────────┘ │    │ └──────────────┘ │    │ └──────────────┘ │
└──────────────┘    └──────────────────┘    └──────────────────┘

Features: ✅ Exactly-once  ✅ Auto-recovery  ✅ Schema sync
```

---

## Prerequisites & Setup

### Download Connectors

```bash
mkdir -p flink-connectors && cd flink-connectors

# CDC Connectors
wget https://repo1.maven.org/maven2/com/ververica/flink-sql-connector-mysql-cdc/3.0.1/flink-sql-connector-mysql-cdc-3.0.1.jar
wget https://repo1.maven.org/maven2/com/ververica/flink-sql-connector-postgres-cdc/3.0.1/flink-sql-connector-postgres-cdc-3.0.1.jar

# Doris Connector
wget https://repo1.maven.org/maven2/org/apache/doris/flink-doris-connector-1.18/1.6.1/flink-doris-connector-1.18-1.6.1.jar

# JDBC + MySQL Driver
wget https://repo1.maven.org/maven2/org/apache/flink/flink-connector-jdbc/3.1.2-1.18/flink-connector-jdbc-3.1.2-1.18.jar
wget https://repo1.maven.org/maven2/mysql/mysql-connector-java/8.0.33/mysql-connector-java-8.0.33.jar
```

### Persiapkan MySQL Source

```sql
-- Pastikan binlog aktif di my.cnf:
-- server-id=1, log_bin=mysql-bin, binlog_format=ROW, binlog_row_image=FULL

CREATE DATABASE IF NOT EXISTS ecommerce_db;
USE ecommerce_db;

CREATE TABLE users (
    user_id     BIGINT PRIMARY KEY AUTO_INCREMENT,
    username    VARCHAR(100) NOT NULL,
    email       VARCHAR(255) NOT NULL UNIQUE,
    full_name   VARCHAR(200),
    city        VARCHAR(100),
    is_active   TINYINT(1) DEFAULT 1,
    created_at  DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at  DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

CREATE TABLE orders (
    order_id      BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id       BIGINT NOT NULL,
    product_name  VARCHAR(255) NOT NULL,
    quantity      INT NOT NULL,
    total_price   DECIMAL(15,2) NOT NULL,
    order_status  VARCHAR(50) DEFAULT 'pending',
    payment_method VARCHAR(50),
    created_at    DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at    DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

-- Sample data
INSERT INTO users (username, email, full_name, city) VALUES
('budi_dev', 'budi@mail.com', 'Budi Santoso', 'Jakarta'),
('siti_data', 'siti@mail.com', 'Siti Rahayu', 'Bandung'),
('andi_ops', 'andi@mail.com', 'Andi Pratama', 'Surabaya');

INSERT INTO orders (user_id, product_name, quantity, total_price, order_status, payment_method) VALUES
(1, 'Laptop ASUS ROG', 1, 25000000, 'completed', 'credit_card'),
(2, 'Mechanical Keyboard', 2, 3500000, 'shipped', 'bank_transfer'),
(1, 'Monitor 4K 27"', 1, 8500000, 'pending', 'e_wallet');
```

### Persiapkan Doris Target

```sql
-- Jalankan di Doris (mysql -h 127.0.0.1 -P 9030 -u root)
CREATE DATABASE IF NOT EXISTS ods_ecommerce;
USE ods_ecommerce;

CREATE TABLE ods_users (
    user_id    BIGINT NOT NULL, username VARCHAR(100),
    email      VARCHAR(255), full_name VARCHAR(200),
    city       VARCHAR(100), is_active BOOLEAN,
    created_at DATETIME, updated_at DATETIME
) UNIQUE KEY(user_id) DISTRIBUTED BY HASH(user_id) BUCKETS 4
PROPERTIES ("replication_allocation"="tag.location.default: 1",
             "enable_unique_key_merge_on_write"="true");

CREATE TABLE ods_orders (
    order_id     BIGINT NOT NULL, user_id BIGINT,
    product_name VARCHAR(255), quantity INT,
    total_price  DECIMAL(15,2), order_status VARCHAR(50),
    payment_method VARCHAR(50),
    created_at   DATETIME, updated_at DATETIME
) UNIQUE KEY(order_id) DISTRIBUTED BY HASH(order_id) BUCKETS 8
PROPERTIES ("replication_allocation"="tag.location.default: 1",
             "enable_unique_key_merge_on_write"="true");
```

---

## Skenario 1: Migrasi Full Data (Batch)

> **Use Case**: One-time full load dari MySQL ke Doris

```bash
# Masuk Flink SQL Client
docker exec -it flink-jobmanager /opt/flink/bin/sql-client.sh embedded \
  -l /opt/flink/lib/connectors
```

```sql
SET 'execution.runtime-mode' = 'batch';

-- Source: MySQL via JDBC
CREATE TABLE mysql_users_batch (
    user_id BIGINT, username STRING, email STRING,
    full_name STRING, city STRING, is_active TINYINT,
    created_at TIMESTAMP(3), updated_at TIMESTAMP(3),
    PRIMARY KEY (user_id) NOT ENFORCED
) WITH (
    'connector'  = 'jdbc',
    'url'        = 'jdbc:mysql://mysql-source:3306/ecommerce_db',
    'table-name' = 'users',
    'username'   = 'root',
    'password'   = 'root123'
);

-- Sink: Doris
CREATE TABLE doris_users_sink (
    user_id BIGINT, username STRING, email STRING,
    full_name STRING, city STRING, is_active BOOLEAN,
    created_at TIMESTAMP(3), updated_at TIMESTAMP(3)
) WITH (
    'connector'        = 'doris',
    'fenodes'          = 'doris-fe:8030',
    'table.identifier' = 'ods_ecommerce.ods_users',
    'username'         = 'root',
    'password'         = '',
    'sink.properties.format' = 'json',
    'sink.properties.read_json_by_line' = 'true',
    'sink.label-prefix' = 'flink_batch_users'
);

-- Execute batch migration
INSERT INTO doris_users_sink
SELECT user_id, username, email, full_name, city,
       CASE WHEN is_active = 1 THEN TRUE ELSE FALSE END,
       created_at, updated_at
FROM mysql_users_batch;
-- ✅ Job selesai setelah semua data ter-load
```

---

## Skenario 2: Real-time CDC dari MySQL

> **Use Case**: Capture setiap INSERT/UPDATE/DELETE secara real-time

```sql
SET 'execution.runtime-mode' = 'streaming';
SET 'execution.checkpointing.interval' = '30s';
SET 'execution.checkpointing.mode' = 'EXACTLY_ONCE';

-- 1️⃣ Source: MySQL CDC (menangkap binlog)
CREATE TABLE mysql_users_cdc (
    user_id BIGINT, username STRING, email STRING,
    full_name STRING, city STRING, is_active TINYINT,
    created_at TIMESTAMP(3), updated_at TIMESTAMP(3),
    PRIMARY KEY (user_id) NOT ENFORCED
) WITH (
    'connector'     = 'mysql-cdc',
    'hostname'      = 'mysql-source',
    'port'          = '3306',
    'username'      = 'root',
    'password'      = 'root123',
    'database-name' = 'ecommerce_db',
    'table-name'    = 'users',
    'server-id'     = '5401-5410',
    'server-time-zone' = 'Asia/Jakarta',
    'scan.startup.mode' = 'initial'  -- snapshot dulu, lalu streaming
);

-- 2️⃣ Sink: Doris (support delete)
CREATE TABLE doris_users_realtime (
    user_id BIGINT, username STRING, email STRING,
    full_name STRING, city STRING, is_active BOOLEAN,
    created_at TIMESTAMP(3), updated_at TIMESTAMP(3)
) WITH (
    'connector'        = 'doris',
    'fenodes'          = 'doris-fe:8030',
    'table.identifier' = 'ods_ecommerce.ods_users',
    'username'         = 'root',
    'password'         = '',
    'sink.properties.format' = 'json',
    'sink.properties.read_json_by_line' = 'true',
    'sink.enable-2pc'    = 'true',
    'sink.enable-delete' = 'true',
    'sink.label-prefix'  = 'flink_cdc_users',
    'sink.buffer-flush.max-rows' = '5000',
    'sink.buffer-flush.interval' = '5s'
);

-- 3️⃣ Execute CDC pipeline
INSERT INTO doris_users_realtime
SELECT user_id, username, email, full_name, city,
       CASE WHEN is_active = 1 THEN TRUE ELSE FALSE END,
       created_at, updated_at
FROM mysql_users_cdc;
-- 🔄 Job berjalan terus-menerus, menangkap setiap perubahan
```

### Verifikasi Real-time

```sql
-- 🧪 Di MySQL Source: test perubahan
INSERT INTO users (username, email, full_name, city)
VALUES ('test_cdc', 'test@mail.com', 'Test CDC User', 'Bali');

UPDATE users SET city = 'Denpasar' WHERE username = 'test_cdc';

DELETE FROM users WHERE username = 'test_cdc';

-- 🔍 Di Doris: verifikasi sync
SELECT * FROM ods_ecommerce.ods_users ORDER BY updated_at DESC LIMIT 5;
```

---

## Skenario 3: Multi-Table Sync

> Sync beberapa tabel dalam satu Flink job menggunakan `STATEMENT SET`

```sql
-- Definisikan semua source CDC & sink tables terlebih dahulu
-- (seperti contoh di atas untuk setiap tabel)

-- Bundle multiple pipelines dalam 1 job
BEGIN STATEMENT SET;

INSERT INTO doris_users_realtime
SELECT user_id, username, email, full_name, city,
       CASE WHEN is_active=1 THEN TRUE ELSE FALSE END,
       created_at, updated_at
FROM mysql_users_cdc;

INSERT INTO doris_orders_realtime
SELECT * FROM mysql_orders_cdc;

INSERT INTO doris_products_realtime
SELECT * FROM mysql_products_cdc;

END;
-- ✅ 1 Flink job sync 3 tabel sekaligus!
```

---

## Skenario 4: Real-time Aggregation

> **Use Case**: Agregasi revenue per hari secara real-time

```sql
-- Source dengan watermark untuk event-time processing
CREATE TABLE mysql_orders_stream (
    order_id BIGINT, user_id BIGINT,
    product_name STRING, quantity INT,
    total_price DECIMAL(15,2), order_status STRING,
    created_at TIMESTAMP(3),
    updated_at TIMESTAMP(3),
    WATERMARK FOR created_at AS created_at - INTERVAL '5' SECOND,
    PRIMARY KEY (order_id) NOT ENFORCED
) WITH (
    'connector'     = 'mysql-cdc',
    'hostname'      = 'mysql-source',
    'port'          = '3306',
    'username'      = 'root',
    'password'      = 'root123',
    'database-name' = 'ecommerce_db',
    'table-name'    = 'orders',
    'server-id'     = '5421-5430',
    'scan.startup.mode' = 'initial'
);

-- Sink: aggregated revenue
CREATE TABLE doris_daily_revenue (
    revenue_date   DATE,
    total_orders   BIGINT,
    total_revenue  DECIMAL(15,2),
    avg_order_val  DECIMAL(15,2),
    unique_buyers  BIGINT
) WITH (
    'connector'        = 'doris',
    'fenodes'          = 'doris-fe:8030',
    'table.identifier' = 'ods_ecommerce.dws_daily_revenue',
    'username' = 'root', 'password' = '',
    'sink.properties.format' = 'json',
    'sink.properties.read_json_by_line' = 'true',
    'sink.enable-2pc' = 'true',
    'sink.label-prefix' = 'flink_agg_revenue'
);

-- Tumbling window aggregation (per hari)
INSERT INTO doris_daily_revenue
SELECT
    CAST(window_start AS DATE) AS revenue_date,
    COUNT(*)                   AS total_orders,
    SUM(total_price)           AS total_revenue,
    AVG(total_price)           AS avg_order_val,
    COUNT(DISTINCT user_id)    AS unique_buyers
FROM TABLE(
    TUMBLE(TABLE mysql_orders_stream, DESCRIPTOR(created_at), INTERVAL '1' DAY)
)
WHERE order_status IN ('completed', 'shipped')
GROUP BY window_start, window_end;
```

---

## Monitoring & Troubleshooting

### Monitor Jobs

```bash
# Cek running jobs
curl -s http://localhost:8081/jobs/overview | python3 -m json.tool

# Cek checkpoint status
curl -s "http://localhost:8081/jobs/<job-id>/checkpoints" | python3 -m json.tool

# Cancel job
curl -X PATCH "http://localhost:8081/jobs/<job-id>?mode=cancel"
```

### Verify Data

```sql
-- Di Doris: bandingkan jumlah
SELECT 'mysql_source' AS src, COUNT(*) AS cnt FROM mysql_source_query
UNION ALL
SELECT 'doris_target', COUNT(*) FROM ods_ecommerce.ods_users;
```

### Common Issues

| Problem | Solusi |
|---------|--------|
| `Access denied for binlog` | Grant: `GRANT REPLICATION SLAVE, CLIENT ON *.* TO user` |
| `Server ID conflict` | Gunakan range unik: `'server-id' = '5401-5410'` |
| `Doris stream load failed` | Pastikan `sink.label-prefix` unik per job |
| `Checkpoint timeout` | Increase `execution.checkpointing.timeout` |
| `Backpressure HIGH` | Increase `sink.buffer-flush.interval` atau parallelism |
| `Table factory not found` | Pastikan connector JAR ada di `lib/connectors/` |

### Best Practices

```
✅ DO:
├── Gunakan EXACTLY_ONCE checkpointing
├── Set incremental checkpoint untuk state besar
├── Gunakan STATEMENT SET untuk multi-table sync
├── Monitor backpressure di Flink Web UI
└── Test CDC di staging sebelum production

❌ DON'T:
├── Jangan set checkpoint interval < 10 detik
├── Jangan gunakan parallelism terlalu tinggi untuk CDC source
├── Jangan lupa set sink.enable-delete untuk UNIQUE KEY tables
└── Jangan share server-id antar CDC jobs
```

---

## 📚 Referensi

- [Flink CDC Documentation](https://ververica.github.io/flink-cdc-connectors/)
- [Flink Doris Connector](https://doris.apache.org/docs/ecosystem/flink-doris-connector/)
- [Apache Flink SQL](https://nightlies.apache.org/flink/flink-docs-stable/docs/dev/table/sql/overview/)

---

> 📝 **Author**: Data Engineering Team  
> 📅 **Last Updated**: 2026-02-10  
> 🏷️ **Tags**: `flink-cdc`, `real-time`, `data-migration`, `mysql-cdc`, `doris`
