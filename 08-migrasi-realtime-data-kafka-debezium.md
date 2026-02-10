# 📡 Cara Migrasi & Realtime Data Menggunakan Kafka & Debezium

> Panduan menggunakan **Apache Kafka** + **Debezium** untuk Change Data Capture (CDC) real-time dari database sumber ke **Apache Doris** sebagai data warehouse.

---

## 📋 Table of Contents

- [Arsitektur Pipeline](#arsitektur-pipeline)
- [Prerequisites](#prerequisites)
- [Skenario 1: MySQL CDC → Kafka → Doris](#skenario-1-mysql-cdc--kafka--doris)
- [Skenario 2: PostgreSQL CDC → Kafka → Doris](#skenario-2-postgresql-cdc--kafka--doris)
- [Skenario 3: Kafka → Flink → Doris (Stream Processing)](#skenario-3-kafka--flink--doris)
- [Consumer Application (Python)](#consumer-application-python)
- [Monitoring & Troubleshooting](#monitoring--troubleshooting)

---

## Arsitektur Pipeline

```
┌────────────┐    ┌───────────────┐    ┌─────────────┐    ┌────────────┐
│ Source DB   │    │ Kafka Connect │    │ Apache      │    │ Target     │
│            │    │ (Debezium)    │    │ Kafka       │    │            │
│ MySQL ─────┼───▶│ MySQL CDC ────┼───▶│ topic:      │───▶│ Doris      │
│ (binlog)   │    │ Connector     │    │ cdc.users   │    │ (Routine   │
│            │    │               │    │ cdc.orders  │    │  Load /    │
│ Postgres ──┼───▶│ PG CDC ───────┼───▶│ cdc.txn     │    │  Flink)    │
│ (WAL)      │    │ Connector     │    │             │    │            │
└────────────┘    └───────────────┘    └─────────────┘    └────────────┘
                                             │
                                       ┌─────▼─────┐
                                       │ Consumers  │
                                       │ • Flink    │
                                       │ • Python   │
                                       │ • Spark    │
                                       └───────────┘
```

---

## Prerequisites

Pastikan Kafka stack sudah running (lihat `04-instalasi-setup-kafka-debezium.md`):

```bash
# Cek semua service running
docker compose ps

# Expected: zookeeper, kafka, kafka-connect, schema-registry, kafka-ui
# 🌐 Kafka UI: http://localhost:8080
# 🔌 Kafka Connect: http://localhost:8083
```

---

## Skenario 1: MySQL CDC → Kafka → Doris

### Step 1: Deploy Debezium MySQL Connector

```bash
curl -X POST http://localhost:8083/connectors \
  -H "Content-Type: application/json" \
  -d '{
    "name": "mysql-ecommerce-cdc",
    "config": {
        "connector.class": "io.debezium.connector.mysql.MySqlConnector",
        "tasks.max": "1",
        "database.hostname": "mysql-source",
        "database.port": "3306",
        "database.user": "debezium",
        "database.password": "debezium_pwd",
        "database.server.id": "184054",
        "topic.prefix": "mysql-cdc",
        "database.include.list": "ecommerce_db",
        "table.include.list": "ecommerce_db.users,ecommerce_db.orders",
        "schema.history.internal.kafka.bootstrap.servers": "kafka:9092",
        "schema.history.internal.kafka.topic": "schema-changes.mysql",
        "snapshot.mode": "initial",
        "transforms": "unwrap",
        "transforms.unwrap.type": "io.debezium.transforms.ExtractNewRecordState",
        "transforms.unwrap.drop.tombstones": "true",
        "transforms.unwrap.delete.handling.mode": "rewrite",
        "transforms.unwrap.add.fields": "op,table,source.ts_ms",
        "key.converter": "org.apache.kafka.connect.json.JsonConverter",
        "key.converter.schemas.enable": "false",
        "value.converter": "org.apache.kafka.connect.json.JsonConverter",
        "value.converter.schemas.enable": "false"
    }
}'
```

### Step 2: Verifikasi Data di Kafka

```bash
# Cek topics yang dibuat Debezium
docker exec kafka kafka-topics --bootstrap-server localhost:9092 --list
# Output:
# mysql-cdc.ecommerce_db.users
# mysql-cdc.ecommerce_db.orders

# Consume messages
docker exec kafka kafka-console-consumer \
  --bootstrap-server localhost:9092 \
  --topic mysql-cdc.ecommerce_db.users \
  --from-beginning --max-messages 3

# Contoh output (setelah ExtractNewRecordState):
# {"user_id":1,"username":"budi_dev","email":"budi@mail.com",
#  "city":"Jakarta","__op":"r","__table":"users","__deleted":"false"}
```

### Step 3: Load ke Doris (Routine Load)

```sql
-- Di Doris: buat Routine Load job
-- Routine Load = Doris native Kafka consumer

CREATE ROUTINE LOAD ods_ecommerce.load_users_from_kafka ON ods_users
COLUMNS(user_id, username, email, full_name, phone, city, is_active, 
        created_at, updated_at, __op, __table, __deleted)
PROPERTIES (
    "format" = "json",
    "jsonpaths" = "[\"$.user_id\",\"$.username\",\"$.email\",\"$.full_name\",
                    \"$.phone\",\"$.city\",\"$.is_active\",\"$.created_at\",
                    \"$.updated_at\"]",
    "max_batch_interval" = "10",
    "max_batch_rows" = "200000",
    "max_error_number" = "100",
    "strict_mode" = "false",
    "timezone" = "Asia/Jakarta"
)
FROM KAFKA (
    "kafka_broker_list" = "kafka:9092",
    "kafka_topic" = "mysql-cdc.ecommerce_db.users",
    "property.group.id" = "doris_routine_load_users",
    "property.kafka_default_offsets" = "OFFSET_BEGINNING"
);

-- Routine Load untuk Orders
CREATE ROUTINE LOAD ods_ecommerce.load_orders_from_kafka ON ods_orders
COLUMNS(order_id, user_id, product_name, quantity, total_price, 
        order_status, payment_method, created_at, updated_at)
PROPERTIES (
    "format" = "json",
    "max_batch_interval" = "10",
    "max_batch_rows" = "200000",
    "strict_mode" = "false"
)
FROM KAFKA (
    "kafka_broker_list" = "kafka:9092",
    "kafka_topic" = "mysql-cdc.ecommerce_db.orders",
    "property.group.id" = "doris_routine_load_orders",
    "property.kafka_default_offsets" = "OFFSET_BEGINNING"
);

-- ✅ Cek status Routine Load
SHOW ROUTINE LOAD FOR ods_ecommerce.load_users_from_kafka\G;

-- Cek semua routine loads
SHOW ALL ROUTINE LOAD\G;

-- Pause / Resume / Stop
PAUSE ROUTINE LOAD FOR ods_ecommerce.load_users_from_kafka;
RESUME ROUTINE LOAD FOR ods_ecommerce.load_users_from_kafka;
STOP ROUTINE LOAD FOR ods_ecommerce.load_users_from_kafka;
```

### Step 4: Test End-to-End

```sql
-- 🧪 Di MySQL: buat perubahan
INSERT INTO users (username, email, full_name, city)
VALUES ('kafka_test', 'kafka@test.com', 'Kafka Test User', 'Bali');

UPDATE users SET city = 'Denpasar' WHERE username = 'kafka_test';

-- 🔍 Di Doris: verifikasi (tunggu ~10 detik)
SELECT * FROM ods_ecommerce.ods_users ORDER BY updated_at DESC LIMIT 5;
```

---

## Skenario 2: PostgreSQL CDC → Kafka → Doris

### Deploy PostgreSQL Connector

```bash
curl -X POST http://localhost:8083/connectors \
  -H "Content-Type: application/json" \
  -d '{
    "name": "postgres-cdc-connector",
    "config": {
        "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
        "tasks.max": "1",
        "database.hostname": "postgres-source",
        "database.port": "5432",
        "database.user": "debezium",
        "database.password": "debezium_pwd",
        "database.dbname": "mydb",
        "topic.prefix": "pg-cdc",
        "schema.include.list": "public",
        "table.include.list": "public.customers,public.transactions",
        "plugin.name": "pgoutput",
        "slot.name": "debezium_slot",
        "publication.name": "debezium_pub",
        "snapshot.mode": "initial",
        "transforms": "unwrap",
        "transforms.unwrap.type": "io.debezium.transforms.ExtractNewRecordState",
        "transforms.unwrap.drop.tombstones": "true",
        "transforms.unwrap.delete.handling.mode": "rewrite",
        "key.converter": "org.apache.kafka.connect.json.JsonConverter",
        "key.converter.schemas.enable": "false",
        "value.converter": "org.apache.kafka.connect.json.JsonConverter",
        "value.converter.schemas.enable": "false"
    }
}'

# Verify
curl -s http://localhost:8083/connectors/postgres-cdc-connector/status | python3 -m json.tool
```

---

## Skenario 3: Kafka → Flink → Doris (Stream Processing)

> Gunakan Flink untuk consume dari Kafka, transform, lalu sink ke Doris

```sql
-- Di Flink SQL Client:
SET 'execution.runtime-mode' = 'streaming';
SET 'execution.checkpointing.interval' = '30s';

-- Source: Kafka topic
CREATE TABLE kafka_orders (
    order_id       BIGINT,
    user_id        BIGINT,
    product_name   STRING,
    quantity       INT,
    total_price    DECIMAL(15,2),
    order_status   STRING,
    payment_method STRING,
    created_at     STRING,
    `__op`         STRING,
    `__deleted`    STRING,
    proc_time AS PROCTIME()
) WITH (
    'connector' = 'kafka',
    'topic' = 'mysql-cdc.ecommerce_db.orders',
    'properties.bootstrap.servers' = 'kafka:9092',
    'properties.group.id' = 'flink-kafka-consumer',
    'scan.startup.mode' = 'earliest-offset',
    'format' = 'json',
    'json.ignore-parse-errors' = 'true'
);

-- Sink: Doris enriched table
CREATE TABLE doris_orders_enriched (
    order_id       BIGINT,
    user_id        BIGINT,
    product_name   STRING,
    quantity       INT,
    total_price    DECIMAL(15,2),
    order_status   STRING,
    payment_method STRING,
    order_tier     STRING,
    is_deleted     BOOLEAN,
    processed_at   TIMESTAMP(3)
) WITH (
    'connector' = 'doris',
    'fenodes' = 'doris-fe:8030',
    'table.identifier' = 'ods_ecommerce.ods_orders_enriched',
    'username' = 'root', 'password' = '',
    'sink.properties.format' = 'json',
    'sink.properties.read_json_by_line' = 'true',
    'sink.enable-2pc' = 'true',
    'sink.label-prefix' = 'flink_kafka_orders'
);

-- Transform & Load
INSERT INTO doris_orders_enriched
SELECT
    order_id, user_id, product_name, quantity, total_price,
    order_status, payment_method,
    CASE
        WHEN total_price >= 10000000 THEN 'Premium'
        WHEN total_price >= 1000000  THEN 'Standard'
        ELSE 'Basic'
    END AS order_tier,
    CASE WHEN `__deleted` = 'true' THEN TRUE ELSE FALSE END,
    proc_time
FROM kafka_orders
WHERE `__op` <> 'd';  -- Skip raw delete events
```

---

## Consumer Application (Python)

```python
# consumer_doris_loader.py
# Python consumer yang baca Kafka dan load ke Doris

from kafka import KafkaConsumer
import json
import requests
import time
from datetime import datetime

# ==============================
# 📡 Kafka Consumer Config
# ==============================
consumer = KafkaConsumer(
    'mysql-cdc.ecommerce_db.orders',
    bootstrap_servers=['localhost:29092'],
    group_id='python-doris-loader',
    auto_offset_reset='earliest',
    enable_auto_commit=True,
    value_deserializer=lambda m: json.loads(m.decode('utf-8')),
)

# ==============================
# 📤 Doris Stream Load
# ==============================
DORIS_FE = 'http://localhost:8030'
DORIS_DB = 'ods_ecommerce'
DORIS_TABLE = 'ods_orders'
DORIS_USER = 'root'
DORIS_PASS = ''

def stream_load_to_doris(records: list):
    """Batch load records ke Doris via Stream Load API"""
    if not records:
        return
    
    url = f'{DORIS_FE}/api/{DORIS_DB}/{DORIS_TABLE}/_stream_load'
    label = f'python_load_{datetime.now().strftime("%Y%m%d_%H%M%S_%f")}'
    
    headers = {
        'label': label,
        'format': 'json',
        'strip_outer_array': 'true',
        'Content-Type': 'application/json',
    }
    
    response = requests.put(
        url, headers=headers,
        data=json.dumps(records),
        auth=(DORIS_USER, DORIS_PASS),
        timeout=60
    )
    
    result = response.json()
    if result.get('Status') == 'Success':
        print(f"✅ Loaded {len(records)} records | Label: {label}")
    else:
        print(f"❌ Load failed: {result.get('Message')}")

# ==============================
# 🔄 Main Consumer Loop
# ==============================
batch = []
BATCH_SIZE = 100
FLUSH_INTERVAL = 10  # seconds
last_flush = time.time()

print("🚀 Starting Kafka consumer...")

for message in consumer:
    record = message.value
    
    # Skip deleted records
    if record.get('__deleted') == 'true':
        continue
    
    # Clean record (remove meta fields)
    clean = {k: v for k, v in record.items() if not k.startswith('__')}
    batch.append(clean)
    
    # Flush ketika batch penuh atau interval tercapai
    if len(batch) >= BATCH_SIZE or (time.time() - last_flush) >= FLUSH_INTERVAL:
        stream_load_to_doris(batch)
        batch = []
        last_flush = time.time()
```

---

## Monitoring & Troubleshooting

### Monitoring Commands

```bash
# Cek connector status
curl -s http://localhost:8083/connectors/mysql-ecommerce-cdc/status | python3 -m json.tool

# Cek consumer lag
docker exec kafka kafka-consumer-groups \
  --bootstrap-server localhost:9092 \
  --describe --group doris_routine_load_users

# Cek topic messages count
docker exec kafka kafka-run-class kafka.tools.GetOffsetShell \
  --broker-list localhost:9092 \
  --topic mysql-cdc.ecommerce_db.users

# Di Doris: cek routine load
SHOW ROUTINE LOAD\G;
```

### Common Issues

| Problem | Solusi |
|---------|--------|
| Connector FAILED | `curl .../connectors/<name>/status` → cek error message |
| Consumer lag tinggi | Tambah partitions atau consumer instances |
| Routine Load PAUSED | Cek error: `SHOW ROUTINE LOAD\G`, fix lalu `RESUME` |
| Data duplikat di Doris | Gunakan UNIQUE KEY table di Doris |
| Missing events | Cek `snapshot.mode`, mulai dari `initial` |
| Schema mismatch | Pastikan kolom di Doris match dengan JSON fields |

### Connector Management

```bash
# Pause connector
curl -X PUT http://localhost:8083/connectors/mysql-ecommerce-cdc/pause

# Resume connector
curl -X PUT http://localhost:8083/connectors/mysql-ecommerce-cdc/resume

# Restart connector
curl -X POST http://localhost:8083/connectors/mysql-ecommerce-cdc/restart

# Delete connector
curl -X DELETE http://localhost:8083/connectors/mysql-ecommerce-cdc
```

---

## 📚 Referensi

- [Debezium Documentation](https://debezium.io/documentation/)
- [Doris Routine Load](https://doris.apache.org/docs/data-operate/import/routine-load-manual/)
- [Apache Kafka Documentation](https://kafka.apache.org/documentation/)

---

> 📝 **Author**: Data Engineering Team  
> 📅 **Last Updated**: 2026-02-10  
> 🏷️ **Tags**: `kafka`, `debezium`, `cdc`, `doris`, `real-time`, `routine-load`
