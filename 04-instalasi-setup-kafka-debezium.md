# 📡 Cara Instalasi & Setup Kafka & Debezium

> **Apache Kafka** adalah distributed event streaming platform yang digunakan untuk membangun real-time data pipelines. **Debezium** adalah open-source CDC (Change Data Capture) platform yang berjalan di atas Kafka Connect, menangkap perubahan data row-level dari database secara real-time.

---

## 📋 Table of Contents

- [Prerequisites](#prerequisites)
- [Arsitektur Kafka & Debezium](#arsitektur-kafka--debezium)
- [Instalasi Kafka](#instalasi-kafka)
  - [Docker Compose: Full Stack](#docker-compose-full-stack)
- [Konfigurasi Kafka](#konfigurasi-kafka)
- [Instalasi & Setup Debezium](#instalasi--setup-debezium)
- [Konfigurasi Source Database](#konfigurasi-source-database)
- [Deploy Debezium Connector](#deploy-debezium-connector)
- [Kafka Operations](#kafka-operations)
- [Schema Registry](#schema-registry)
- [Monitoring](#monitoring)
- [Troubleshooting](#troubleshooting)

---

## Prerequisites

| Komponen         | Minimum Requirement         |
|------------------|-----------------------------|
| **OS**           | Ubuntu 20.04+ / CentOS 7+  |
| **CPU**          | 4 Cores                     |
| **RAM**          | 8 GB (16+ recommended)      |
| **Disk**         | 100 GB SSD                  |
| **Docker**       | 20.10+                      |
| **Docker Compose** | v2.0+                    |
| **Source DB**    | MySQL 5.7+ / PostgreSQL 10+ |

---

## Arsitektur Kafka & Debezium

```
┌──────────────────────────────────────────────────────────────────┐
│                  Kafka + Debezium Architecture                   │
│                                                                  │
│  ┌────────────┐     ┌──────────────────┐     ┌───────────────┐  │
│  │ Source DB   │     │  Kafka Connect   │     │  Apache Kafka  │  │
│  │            │     │  (Debezium)      │     │               │  │
│  │ ┌────────┐ │     │ ┌──────────────┐ │     │ ┌───────────┐ │  │
│  │ │ MySQL  │─┼────▶│ │ MySQL        │ │────▶│ │ Topic:    │ │  │
│  │ │ binlog │ │     │ │ Connector    │ │     │ │ dbserver1 │ │  │
│  │ └────────┘ │     │ └──────────────┘ │     │ │ .db.table │ │  │
│  │            │     │                  │     │ └───────────┘ │  │
│  │ ┌────────┐ │     │ ┌──────────────┐ │     │               │  │
│  │ │ Postgre│─┼────▶│ │ PostgreSQL   │ │────▶│ ┌───────────┐ │  │
│  │ │ WAL    │ │     │ │ Connector    │ │     │ │ Consumers │ │  │
│  │ └────────┘ │     │ └──────────────┘ │     │ │ (Flink,   │ │  │
│  └────────────┘     └──────────────────┘     │ │  Spark,   │ │  │
│                                              │ │  App)     │ │  │
│  ┌────────────┐     ┌──────────────────┐     │ └───────────┘ │  │
│  │ Zookeeper  │◄───▶│ Schema Registry  │     │               │  │
│  └────────────┘     └──────────────────┘     └───────────────┘  │
└──────────────────────────────────────────────────────────────────┘
```

---

## Instalasi Kafka

### Docker Compose: Full Stack

```yaml
# docker-compose.yml - Kafka + Zookeeper + Debezium + Kafka UI
version: "3.8"

services:
  # ===========================
  # Zookeeper
  # ===========================
  zookeeper:
    image: confluentinc/cp-zookeeper:7.6.0
    container_name: zookeeper
    hostname: zookeeper
    ports:
      - "2181:2181"
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000
      ZOOKEEPER_INIT_LIMIT: 5
      ZOOKEEPER_SYNC_LIMIT: 2
    volumes:
      - zookeeper-data:/var/lib/zookeeper/data
      - zookeeper-logs:/var/lib/zookeeper/log
    networks:
      - kafka-network
    restart: unless-stopped

  # ===========================
  # Apache Kafka Broker
  # ===========================
  kafka:
    image: confluentinc/cp-kafka:7.6.0
    container_name: kafka
    hostname: kafka
    depends_on:
      - zookeeper
    ports:
      - "9092:9092"
      - "29092:29092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092,PLAINTEXT_HOST://localhost:29092
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1
      KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS: 0
      KAFKA_AUTO_CREATE_TOPICS_ENABLE: "true"
      KAFKA_NUM_PARTITIONS: 3
      KAFKA_DEFAULT_REPLICATION_FACTOR: 1
      KAFKA_LOG_RETENTION_HOURS: 168         # 7 days
      KAFKA_LOG_RETENTION_BYTES: 1073741824  # 1 GB per partition
      KAFKA_LOG_SEGMENT_BYTES: 1073741824
      KAFKA_MESSAGE_MAX_BYTES: 10485760      # 10 MB
      KAFKA_COMPRESSION_TYPE: "lz4"
    volumes:
      - kafka-data:/var/lib/kafka/data
    networks:
      - kafka-network
    restart: unless-stopped

  # ===========================
  # Schema Registry
  # ===========================
  schema-registry:
    image: confluentinc/cp-schema-registry:7.6.0
    container_name: schema-registry
    hostname: schema-registry
    depends_on:
      - kafka
    ports:
      - "8081:8081"
    environment:
      SCHEMA_REGISTRY_HOST_NAME: schema-registry
      SCHEMA_REGISTRY_KAFKASTORE_BOOTSTRAP_SERVERS: kafka:9092
      SCHEMA_REGISTRY_LISTENERS: http://0.0.0.0:8081
    networks:
      - kafka-network
    restart: unless-stopped

  # ===========================
  # Kafka Connect (Debezium)
  # ===========================
  kafka-connect:
    image: debezium/connect:2.5
    container_name: kafka-connect
    hostname: kafka-connect
    depends_on:
      - kafka
      - schema-registry
    ports:
      - "8083:8083"
    environment:
      GROUP_ID: "debezium-connect-cluster"
      BOOTSTRAP_SERVERS: kafka:9092
      CONFIG_STORAGE_TOPIC: debezium_configs
      OFFSET_STORAGE_TOPIC: debezium_offsets
      STATUS_STORAGE_TOPIC: debezium_statuses
      CONFIG_STORAGE_REPLICATION_FACTOR: 1
      OFFSET_STORAGE_REPLICATION_FACTOR: 1
      STATUS_STORAGE_REPLICATION_FACTOR: 1
      KEY_CONVERTER: org.apache.kafka.connect.json.JsonConverter
      VALUE_CONVERTER: org.apache.kafka.connect.json.JsonConverter
      KEY_CONVERTER_SCHEMAS_ENABLE: "false"
      VALUE_CONVERTER_SCHEMAS_ENABLE: "false"
      CONNECT_KEY_CONVERTER_SCHEMA_REGISTRY_URL: http://schema-registry:8081
      CONNECT_VALUE_CONVERTER_SCHEMA_REGISTRY_URL: http://schema-registry:8081
      ENABLE_DEBEZIUM_SCRIPTING: "true"
    volumes:
      - kafka-connect-plugins:/kafka/connect
    networks:
      - kafka-network
    restart: unless-stopped

  # ===========================
  # Kafka UI (Monitoring)
  # ===========================
  kafka-ui:
    image: provectuslabs/kafka-ui:latest
    container_name: kafka-ui
    ports:
      - "8080:8080"
    environment:
      KAFKA_CLUSTERS_0_NAME: local-cluster
      KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: kafka:9092
      KAFKA_CLUSTERS_0_ZOOKEEPER: zookeeper:2181
      KAFKA_CLUSTERS_0_SCHEMAREGISTRY: http://schema-registry:8081
      KAFKA_CLUSTERS_0_KAFKACONNECT_0_NAME: debezium
      KAFKA_CLUSTERS_0_KAFKACONNECT_0_ADDRESS: http://kafka-connect:8083
      DYNAMIC_CONFIG_ENABLED: "true"
    depends_on:
      - kafka
      - kafka-connect
      - schema-registry
    networks:
      - kafka-network
    restart: unless-stopped

volumes:
  zookeeper-data:
  zookeeper-logs:
  kafka-data:
  kafka-connect-plugins:

networks:
  kafka-network:
    driver: bridge
```

```bash
# 🚀 Jalankan seluruh stack
docker compose up -d

# Cek semua container running
docker compose ps

# Cek Kafka broker
docker exec kafka kafka-broker-api-versions --bootstrap-server localhost:9092

# 🌐 Akses UI:
# Kafka UI:       http://localhost:8080
# Schema Registry: http://localhost:8081
# Kafka Connect:   http://localhost:8083
```

---

## Konfigurasi Source Database

### MySQL - Enable Binlog

```sql
-- Cek binlog status
SHOW VARIABLES LIKE 'log_bin';
SHOW VARIABLES LIKE 'binlog_format';
SHOW VARIABLES LIKE 'binlog_row_image';
SHOW VARIABLES LIKE 'server_id';
```

Edit `my.cnf`:

```ini
# /etc/mysql/my.cnf atau /etc/mysql/mysql.conf.d/mysqld.cnf
[mysqld]
# Wajib untuk Debezium CDC
server-id         = 223344
log_bin           = mysql-bin
binlog_format     = ROW
binlog_row_image  = FULL
expire_logs_days  = 7

# Pilihan: enable GTID (recommended)
gtid_mode                = ON
enforce_gtid_consistency = ON

# Performance
sync_binlog       = 1
innodb_flush_log_at_trx_commit = 1
```

```sql
-- Buat user khusus untuk Debezium
CREATE USER 'debezium'@'%' IDENTIFIED BY 'debezium_secret_pwd';

GRANT SELECT, RELOAD, SHOW DATABASES, REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'debezium'@'%';

-- Untuk MySQL 8+ jika pakai GTID
GRANT LOCK TABLES ON *.* TO 'debezium'@'%';

FLUSH PRIVILEGES;
```

### PostgreSQL - Enable WAL

Edit `postgresql.conf`:

```ini
# /etc/postgresql/14/main/postgresql.conf

# WAL Settings untuk Debezium
wal_level = logical
max_wal_senders = 4
max_replication_slots = 4

# Plugin (pilih salah satu)
# shared_preload_libraries = 'decoderbufs'  # atau pgoutput (default di PG 10+)
```

```sql
-- Buat user dan set permission
CREATE ROLE debezium WITH LOGIN PASSWORD 'debezium_secret_pwd' REPLICATION;

-- Grant akses ke database
GRANT CONNECT ON DATABASE mydb TO debezium;
GRANT USAGE ON SCHEMA public TO debezium;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO debezium;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO debezium;

-- Buat replication slot (opsional, Debezium akan buat otomatis)
SELECT * FROM pg_create_logical_replication_slot('debezium_slot', 'pgoutput');

-- Buat publication
CREATE PUBLICATION debezium_publication FOR ALL TABLES;
```

---

## Deploy Debezium Connector

### MySQL Connector

```bash
# Deploy MySQL CDC Connector
curl -X POST http://localhost:8083/connectors \
  -H "Content-Type: application/json" \
  -d '{
    "name": "mysql-source-connector",
    "config": {
        "connector.class": "io.debezium.connector.mysql.MySqlConnector",
        "tasks.max": "1",
        
        "database.hostname": "mysql-host",
        "database.port": "3306",
        "database.user": "debezium",
        "database.password": "debezium_secret_pwd",
        "database.server.id": "184054",
        
        "topic.prefix": "mysql-cdc",
        "database.include.list": "ecommerce_db",
        "table.include.list": "ecommerce_db.orders,ecommerce_db.users,ecommerce_db.products",
        
        "schema.history.internal.kafka.bootstrap.servers": "kafka:9092",
        "schema.history.internal.kafka.topic": "schema-changes.mysql",
        
        "include.schema.changes": "true",
        "snapshot.mode": "initial",
        "snapshot.locking.mode": "minimal",
        
        "decimal.handling.mode": "double",
        "time.precision.mode": "connect",
        "binary.handling.mode": "base64",
        
        "transforms": "unwrap,route",
        "transforms.unwrap.type": "io.debezium.transforms.ExtractNewRecordState",
        "transforms.unwrap.drop.tombstones": "true",
        "transforms.unwrap.delete.handling.mode": "rewrite",
        "transforms.unwrap.add.fields": "op,table,source.ts_ms",
        
        "transforms.route.type": "org.apache.kafka.connect.transforms.RegexRouter",
        "transforms.route.regex": "([^.]+)\\.([^.]+)\\.([^.]+)",
        "transforms.route.replacement": "cdc.$3",
        
        "key.converter": "org.apache.kafka.connect.json.JsonConverter",
        "key.converter.schemas.enable": "false",
        "value.converter": "org.apache.kafka.connect.json.JsonConverter",
        "value.converter.schemas.enable": "false",
        
        "heartbeat.interval.ms": "10000",
        "poll.interval.ms": "1000",
        "max.batch.size": "2048"
    }
}'
```

### PostgreSQL Connector

```bash
# Deploy PostgreSQL CDC Connector
curl -X POST http://localhost:8083/connectors \
  -H "Content-Type: application/json" \
  -d '{
    "name": "postgres-source-connector",
    "config": {
        "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
        "tasks.max": "1",
        
        "database.hostname": "postgres-host",
        "database.port": "5432",
        "database.user": "debezium",
        "database.password": "debezium_secret_pwd",
        "database.dbname": "mydb",
        
        "topic.prefix": "pg-cdc",
        "schema.include.list": "public",
        "table.include.list": "public.customers,public.transactions",
        
        "plugin.name": "pgoutput",
        "publication.name": "debezium_publication",
        "slot.name": "debezium_slot",
        
        "snapshot.mode": "initial",
        
        "transforms": "unwrap",
        "transforms.unwrap.type": "io.debezium.transforms.ExtractNewRecordState",
        "transforms.unwrap.drop.tombstones": "true",
        "transforms.unwrap.delete.handling.mode": "rewrite",
        "transforms.unwrap.add.fields": "op,table,source.ts_ms",
        
        "key.converter": "org.apache.kafka.connect.json.JsonConverter",
        "key.converter.schemas.enable": "false",
        "value.converter": "org.apache.kafka.connect.json.JsonConverter",
        "value.converter.schemas.enable": "false",
        
        "heartbeat.interval.ms": "10000"
    }
}'
```

### Cek Status Connector

```bash
# ✅ List semua connectors
curl -s http://localhost:8083/connectors | python3 -m json.tool

# ✅ Cek status connector
curl -s http://localhost:8083/connectors/mysql-source-connector/status | python3 -m json.tool

# Expected output:
# {
#   "name": "mysql-source-connector",
#   "connector": { "state": "RUNNING", "worker_id": "..." },
#   "tasks": [{ "id": 0, "state": "RUNNING", "worker_id": "..." }]
# }

# ✅ Cek konfigurasi connector
curl -s http://localhost:8083/connectors/mysql-source-connector/config | python3 -m json.tool

# ⏸️ Pause connector
curl -X PUT http://localhost:8083/connectors/mysql-source-connector/pause

# ▶️ Resume connector
curl -X PUT http://localhost:8083/connectors/mysql-source-connector/resume

# 🔄 Restart connector
curl -X POST http://localhost:8083/connectors/mysql-source-connector/restart

# 🗑️ Delete connector
curl -X DELETE http://localhost:8083/connectors/mysql-source-connector

# 📝 Update connector config
curl -X PUT http://localhost:8083/connectors/mysql-source-connector/config \
  -H "Content-Type: application/json" \
  -d '{ ...updated config... }'
```

---

## Kafka Operations

### Topic Management

```bash
# 📋 List semua topics
docker exec kafka kafka-topics --bootstrap-server localhost:9092 --list

# ➕ Buat topic manual
docker exec kafka kafka-topics --bootstrap-server localhost:9092 \
  --create \
  --topic my-custom-topic \
  --partitions 6 \
  --replication-factor 1 \
  --config retention.ms=604800000 \
  --config cleanup.policy=delete

# 📊 Describe topic
docker exec kafka kafka-topics --bootstrap-server localhost:9092 \
  --describe --topic cdc.orders

# 🗑️ Delete topic
docker exec kafka kafka-topics --bootstrap-server localhost:9092 \
  --delete --topic my-custom-topic

# ⚙️ Alter topic config
docker exec kafka kafka-configs --bootstrap-server localhost:9092 \
  --alter --entity-type topics --entity-name cdc.orders \
  --add-config retention.ms=259200000  # 3 days
```

### Produce & Consume Messages

```bash
# 📤 Produce messages
docker exec -it kafka kafka-console-producer \
  --bootstrap-server localhost:9092 \
  --topic test-topic \
  --property "key.separator=:" \
  --property "parse.key=true"

# Input (key:value):
# user1:{"name":"Budi","age":25}
# user2:{"name":"Siti","age":28}

# 📥 Consume messages (dari awal)
docker exec kafka kafka-console-consumer \
  --bootstrap-server localhost:9092 \
  --topic cdc.orders \
  --from-beginning \
  --max-messages 10 \
  --property print.key=true \
  --property key.separator=" | "

# 📥 Consume dengan format JSON (pretty print)
docker exec kafka kafka-console-consumer \
  --bootstrap-server localhost:9092 \
  --topic cdc.orders \
  --from-beginning \
  --max-messages 5 | python3 -m json.tool

# 📊 Cek consumer group lag
docker exec kafka kafka-consumer-groups \
  --bootstrap-server localhost:9092 \
  --describe --group debezium-connect-cluster

# Output example:
# GROUP                   TOPIC           PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG
# debezium-connect-cluster cdc.orders     0          1523            1523            0
# debezium-connect-cluster cdc.orders     1          1287            1290            3
```

### Contoh Event CDC dari Debezium

```json
// Event CREATE (insert) dari Debezium
{
  "before": null,
  "after": {
    "order_id": 1001,
    "user_id": 42,
    "product_name": "Laptop ASUS ROG",
    "quantity": 1,
    "total_price": 25000000.00,
    "order_status": "pending",
    "created_at": "2026-02-10T10:30:00Z"
  },
  "source": {
    "version": "2.5.0.Final",
    "connector": "mysql",
    "name": "mysql-cdc",
    "ts_ms": 1707554400000,
    "db": "ecommerce_db",
    "table": "orders",
    "server_id": 223344,
    "gtid": "d4e2fe35-...",
    "file": "mysql-bin.000003",
    "pos": 12345
  },
  "op": "c",  // c=create, u=update, d=delete, r=read(snapshot)
  "ts_ms": 1707554400123
}

// Event UPDATE setelah ExtractNewRecordState transform
{
  "order_id": 1001,
  "user_id": 42,
  "product_name": "Laptop ASUS ROG",
  "quantity": 1,
  "total_price": 25000000.00,
  "order_status": "shipped",
  "created_at": "2026-02-10T10:30:00Z",
  "__op": "u",
  "__table": "orders",
  "__source_ts_ms": 1707554500000,
  "__deleted": "false"
}
```

---

## Monitoring

### Via Kafka UI

```
🌐 http://localhost:8080
```

**Kafka UI** menyediakan:
- **Brokers**: Health, partitions, disk usage
- **Topics**: Messages, partitions, consumer groups
- **Consumers**: Lag monitoring per partition
- **Kafka Connect**: Connectors status, task details
- **Schema Registry**: Schema versions

### Health Check Scripts

```bash
#!/bin/bash
# health-check.sh - Monitoring script untuk Kafka & Debezium

echo "========================================="
echo "🔍 Kafka & Debezium Health Check"
echo "========================================="

# 1. Cek Kafka broker
echo -e "\n📡 Kafka Broker Status:"
docker exec kafka kafka-broker-api-versions --bootstrap-server localhost:9092 > /dev/null 2>&1
if [ $? -eq 0 ]; then
    echo "  ✅ Kafka broker is RUNNING"
else
    echo "  ❌ Kafka broker is DOWN"
fi

# 2. Cek Kafka Connect
echo -e "\n🔌 Kafka Connect Status:"
CONNECT_STATUS=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:8083/)
if [ "$CONNECT_STATUS" == "200" ]; then
    echo "  ✅ Kafka Connect is RUNNING"
else
    echo "  ❌ Kafka Connect is DOWN (HTTP: $CONNECT_STATUS)"
fi

# 3. Cek semua connectors
echo -e "\n📋 Connector Status:"
CONNECTORS=$(curl -s http://localhost:8083/connectors)
for CONNECTOR in $(echo $CONNECTORS | python3 -c "import sys, json; [print(c) for c in json.load(sys.stdin)]" 2>/dev/null); do
    STATUS=$(curl -s http://localhost:8083/connectors/$CONNECTOR/status | python3 -c "
import sys, json
data = json.load(sys.stdin)
connector_state = data['connector']['state']
tasks = data.get('tasks', [])
task_states = ', '.join([f\"Task {t['id']}: {t['state']}\" for t in tasks])
print(f'{connector_state} | {task_states}')
" 2>/dev/null)
    echo "  📌 $CONNECTOR: $STATUS"
done

# 4. Cek consumer group lag
echo -e "\n📊 Consumer Group Lag:"
docker exec kafka kafka-consumer-groups \
  --bootstrap-server localhost:9092 \
  --describe --all-groups 2>/dev/null | head -20
```

---

## Troubleshooting

| Problem | Solusi |
|---------|--------|
| `Connector FAILED` | Cek error: `curl http://localhost:8083/connectors/<name>/status` |
| `Access denied for binlog` | Grant: `GRANT REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'debezium'@'%'` |
| `No matching WAL entry` | Pastikan `wal_level = logical` di PostgreSQL |
| `Topic already exists` | Hapus topic: `kafka-topics --delete --topic <name>` |
| `Kafka Connect OOM` | Tambah memory: `KAFKA_HEAP_OPTS: "-Xms512m -Xmx2g"` |
| `Snapshot too slow` | Set `snapshot.mode=schema_only` (skip data lama) |
| `Consumer lag tinggi` | Tambah partitions & consumer instances |

### Reset Connector Offset

```bash
# Stop connector
curl -X PUT http://localhost:8083/connectors/mysql-source-connector/pause

# Delete connector  
curl -X DELETE http://localhost:8083/connectors/mysql-source-connector

# Delete offset topic (hati-hati di production!)
docker exec kafka kafka-topics --bootstrap-server localhost:9092 \
  --delete --topic debezium_offsets

# Re-deploy connector (akan snapshot ulang)
# ... curl -X POST ...
```

---

## 📚 Referensi

- [Apache Kafka Documentation](https://kafka.apache.org/documentation/)
- [Debezium Documentation](https://debezium.io/documentation/)
- [Debezium Connectors](https://debezium.io/documentation/reference/stable/connectors/)
- [Confluent Platform](https://docs.confluent.io/platform/current/overview.html)
- [Kafka UI GitHub](https://github.com/provectus/kafka-ui)

---

> 📝 **Author**: Data Engineering Team  
> 📅 **Last Updated**: 2026-02-10  
> 🏷️ **Tags**: `kafka`, `debezium`, `cdc`, `event-streaming`, `real-time`
