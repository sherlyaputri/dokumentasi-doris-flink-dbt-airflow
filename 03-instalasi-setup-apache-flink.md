# ⚡ Cara Instalasi & Setup Apache Flink

> **Apache Flink** adalah framework distributed stream processing yang powerful untuk real-time data processing. Flink mendukung batch dan stream processing dengan exactly-once semantics, low latency, dan high throughput.

---

## 📋 Table of Contents

- [Prerequisites](#prerequisites)
- [Arsitektur Apache Flink](#arsitektur-apache-flink)
- [Instalasi Apache Flink](#instalasi-apache-flink)
  - [Metode 1: Docker Compose (Recommended)](#metode-1-docker-compose-recommended)
  - [Metode 2: Standalone Cluster](#metode-2-standalone-cluster)
- [Konfigurasi Flink](#konfigurasi-flink)
- [Flink Web UI](#flink-web-ui)
- [Flink SQL Client](#flink-sql-client)
- [Contoh Job: Stream Processing](#contoh-job-stream-processing)
- [Connector Setup](#connector-setup)
- [Monitoring & Tuning](#monitoring--tuning)
- [Troubleshooting](#troubleshooting)

---

## Prerequisites

| Komponen         | Minimum Requirement          |
|------------------|------------------------------|
| **OS**           | Ubuntu 20.04+ / CentOS 7+   |
| **CPU**          | 4 Cores (8+ recommended)    |
| **RAM**          | 8 GB (16+ recommended)      |
| **Disk**         | 50 GB SSD                   |
| **Java**         | JDK 11 (JDK 8 untuk Flink < 1.17) |
| **Docker**       | 20.10+                      |
| **Docker Compose** | v2.0+                     |

### Install Java

```bash
# Install JDK 11
sudo apt update
sudo apt install -y openjdk-11-jdk

# Verify
java -version
# Output: openjdk version "11.0.x"

# Set JAVA_HOME
echo 'export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64' >> ~/.bashrc
source ~/.bashrc
echo $JAVA_HOME
```

---

## Arsitektur Apache Flink

```
┌──────────────────────────────────────────────────────────────┐
│                     Apache Flink Cluster                     │
│                                                              │
│  ┌──────────────────────┐    ┌─────────────────────────────┐ │
│  │   JobManager (JM)    │    │   TaskManager (TM) x N      │ │
│  │                      │    │                             │ │
│  │ • Job Scheduling     │    │ • Task Execution            │ │
│  │ • Checkpoint Coord.  │◄──▶│ • Memory Management         │ │
│  │ • Resource Manager   │    │ • Network Buffers           │ │
│  │ • REST API (:8081)   │    │ • State Backend             │ │
│  │                      │    │                             │ │
│  │ ┌──────────────────┐ │    │ ┌───────┐ ┌───────┐        │ │
│  │ │ Dispatcher       │ │    │ │ Slot 1│ │ Slot 2│ ...    │ │
│  │ │ ResourceManager  │ │    │ │       │ │       │        │ │
│  │ │ JobMaster        │ │    │ │ Task  │ │ Task  │        │ │
│  │ └──────────────────┘ │    │ └───────┘ └───────┘        │ │
│  └──────────────────────┘    └─────────────────────────────┘ │
│              │                           │                   │
│              ▼                           ▼                   │
│  ┌───────────────────────────────────────────────────────┐   │
│  │              State Backend (RocksDB / Heap)           │   │
│  │              Checkpoint Storage (HDFS / S3)           │   │
│  └───────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────┘

Data Flow:
  Source ──▶ Transformation ──▶ Transformation ──▶ Sink
  (Kafka)    (Map/Filter)       (Window/Join)     (Doris/DB)
```

---

## Instalasi Apache Flink

### Metode 1: Docker Compose (Recommended)

```yaml
# docker-compose.yml
version: "3.8"

services:
  # ===========================
  # Flink JobManager
  # ===========================
  flink-jobmanager:
    image: flink:1.18.1-scala_2.12-java11
    container_name: flink-jobmanager
    hostname: jobmanager
    ports:
      - "8081:8081"   # Web UI
    command: jobmanager
    environment:
      - |
        FLINK_PROPERTIES=
        jobmanager.rpc.address: jobmanager
        jobmanager.rpc.port: 6123
        jobmanager.memory.process.size: 2048m
        jobmanager.memory.jvm-overhead.min: 256m
        jobmanager.memory.jvm-overhead.max: 512m
        state.backend: rocksdb
        state.checkpoints.dir: file:///opt/flink/checkpoints
        state.savepoints.dir: file:///opt/flink/savepoints
        execution.checkpointing.interval: 60000
        execution.checkpointing.mode: EXACTLY_ONCE
        restart-strategy: fixed-delay
        restart-strategy.fixed-delay.attempts: 3
        restart-strategy.fixed-delay.delay: 10s
        web.upload.dir: /opt/flink/usrlib
    volumes:
      - flink-checkpoints:/opt/flink/checkpoints
      - flink-savepoints:/opt/flink/savepoints
      - flink-usrlib:/opt/flink/usrlib
      - ./flink-connectors:/opt/flink/lib/connectors
    networks:
      - flink-network
    restart: unless-stopped

  # ===========================
  # Flink TaskManager (Worker)
  # ===========================
  flink-taskmanager:
    image: flink:1.18.1-scala_2.12-java11
    container_name: flink-taskmanager
    hostname: taskmanager
    depends_on:
      - flink-jobmanager
    command: taskmanager
    environment:
      - |
        FLINK_PROPERTIES=
        jobmanager.rpc.address: jobmanager
        taskmanager.numberOfTaskSlots: 4
        taskmanager.memory.process.size: 4096m
        taskmanager.memory.managed.fraction: 0.4
        taskmanager.memory.network.fraction: 0.1
        taskmanager.memory.jvm-overhead.min: 256m
        taskmanager.memory.jvm-overhead.max: 512m
        state.backend: rocksdb
        state.checkpoints.dir: file:///opt/flink/checkpoints
    volumes:
      - flink-checkpoints:/opt/flink/checkpoints
      - flink-usrlib:/opt/flink/usrlib
      - ./flink-connectors:/opt/flink/lib/connectors
    networks:
      - flink-network
    restart: unless-stopped
    deploy:
      replicas: 2  # Jalankan 2 TaskManager

  # ===========================
  # Flink SQL Client
  # ===========================
  flink-sql-client:
    image: flink:1.18.1-scala_2.12-java11
    container_name: flink-sql-client
    depends_on:
      - flink-jobmanager
    command: >
      /bin/bash -c "
      sleep 10 &&
      /opt/flink/bin/sql-client.sh embedded -l /opt/flink/lib/connectors
      "
    environment:
      - |
        FLINK_PROPERTIES=
        jobmanager.rpc.address: jobmanager
        rest.address: jobmanager
        rest.port: 8081
    volumes:
      - ./flink-connectors:/opt/flink/lib/connectors
      - ./sql-scripts:/opt/flink/sql-scripts
    networks:
      - flink-network
    stdin_open: true
    tty: true

volumes:
  flink-checkpoints:
  flink-savepoints:
  flink-usrlib:

networks:
  flink-network:
    driver: bridge
```

```bash
# Buat folder untuk connectors
mkdir -p flink-connectors sql-scripts

# Jalankan cluster
docker compose up -d flink-jobmanager flink-taskmanager

# Cek status
docker compose ps

# Akses Flink Web UI
# 🌐 http://localhost:8081
```

### Metode 2: Standalone Cluster

```bash
# Download Apache Flink
FLINK_VERSION="1.18.1"
wget https://dlcdn.apache.org/flink/flink-${FLINK_VERSION}/flink-${FLINK_VERSION}-bin-scala_2.12.tgz

# Extract
tar -xzf flink-${FLINK_VERSION}-bin-scala_2.12.tgz
cd flink-${FLINK_VERSION}

# Struktur folder
# flink-1.18.1/
# ├── bin/          # Scripts (start, stop, flink CLI)
# ├── conf/         # Configuration files
# ├── lib/          # Core libraries
# ├── opt/          # Optional libraries (connectors)
# ├── plugins/      # Plugin directory
# └── log/          # Log files

# Start cluster
./bin/start-cluster.sh

# Output:
# Starting cluster.
# Starting standalonesession daemon on host xxx.
# Starting taskexecutor daemon on host xxx.

# Verifikasi
curl http://localhost:8081/overview
# Web UI: http://localhost:8081

# Stop cluster
./bin/stop-cluster.sh
```

---

## Konfigurasi Flink

### flink-conf.yaml

```yaml
# ============================================
# Apache Flink Configuration
# File: conf/flink-conf.yaml
# ============================================

# === JobManager ===
jobmanager.rpc.address: localhost
jobmanager.rpc.port: 6123
jobmanager.bind-host: 0.0.0.0
jobmanager.memory.process.size: 2048m

# === TaskManager ===
taskmanager.bind-host: 0.0.0.0
taskmanager.host: localhost
taskmanager.numberOfTaskSlots: 4
taskmanager.memory.process.size: 4096m
taskmanager.memory.managed.fraction: 0.4

# === Parallelism ===
parallelism.default: 4

# === Web UI ===
rest.port: 8081
rest.address: 0.0.0.0
rest.bind-address: 0.0.0.0
web.submit.enable: true
web.cancel.enable: true
web.upload.dir: /opt/flink/usrlib

# === Checkpointing ===
execution.checkpointing.interval: 60000
execution.checkpointing.mode: EXACTLY_ONCE
execution.checkpointing.min-pause: 500
execution.checkpointing.timeout: 600000
execution.checkpointing.max-concurrent-checkpoints: 1

# === State Backend ===
state.backend: rocksdb
state.backend.rocksdb.localdir: /tmp/flink-rocksdb
state.checkpoints.dir: file:///opt/flink/checkpoints
state.savepoints.dir: file:///opt/flink/savepoints
state.backend.incremental: true

# === Restart Strategy ===
restart-strategy: fixed-delay
restart-strategy.fixed-delay.attempts: 3
restart-strategy.fixed-delay.delay: 30s

# === High Availability (untuk production) ===
# high-availability: zookeeper
# high-availability.zookeeper.quorum: zk1:2181,zk2:2181,zk3:2181
# high-availability.storageDir: hdfs:///flink/ha/
# high-availability.zookeeper.path.root: /flink

# === History Server ===
jobmanager.archive.fs.dir: file:///opt/flink/completed-jobs
historyserver.web.address: 0.0.0.0
historyserver.web.port: 8082
historyserver.archive.fs.dir: file:///opt/flink/completed-jobs
historyserver.archive.fs.refresh-interval: 10000
```

---

## Flink Web UI

Akses **Flink Dashboard** di:

```
🌐 http://localhost:8081
```

Features:
- **Overview**: Slot availability, running jobs, completed jobs
- **Running Jobs**: Detail job graph, task status, throughput
- **Task Managers**: Memory, CPU, slot allocation
- **Job Manager**: Configuration, logs, stdout

---

## Flink SQL Client

### Download Connectors

```bash
# Download connectors ke folder flink-connectors/
cd flink-connectors/

# Kafka Connector
wget https://repo1.maven.org/maven2/org/apache/flink/flink-sql-connector-kafka/3.1.0-1.18/flink-sql-connector-kafka-3.1.0-1.18.jar

# JDBC Connector
wget https://repo1.maven.org/maven2/org/apache/flink/flink-connector-jdbc/3.1.2-1.18/flink-connector-jdbc-3.1.2-1.18.jar

# MySQL CDC Connector (untuk CDC)
wget https://repo1.maven.org/maven2/com/ververica/flink-sql-connector-mysql-cdc/3.0.1/flink-sql-connector-mysql-cdc-3.0.1.jar

# PostgreSQL CDC
wget https://repo1.maven.org/maven2/com/ververica/flink-sql-connector-postgres-cdc/3.0.1/flink-sql-connector-postgres-cdc-3.0.1.jar

# Elasticsearch Connector
wget https://repo1.maven.org/maven2/org/apache/flink/flink-sql-connector-elasticsearch7/3.0.1-1.17/flink-sql-connector-elasticsearch7-3.0.1-1.17.jar

# Doris Connector
wget https://repo1.maven.org/maven2/org/apache/doris/flink-doris-connector-1.18/1.6.1/flink-doris-connector-1.18-1.6.1.jar

# MySQL JDBC Driver
wget https://repo1.maven.org/maven2/mysql/mysql-connector-java/8.0.33/mysql-connector-java-8.0.33.jar
```

### Menggunakan SQL Client

```bash
# Masuk ke SQL client via Docker
docker exec -it flink-sql-client bash
/opt/flink/bin/sql-client.sh embedded -l /opt/flink/lib/connectors

# Atau standalone
./bin/sql-client.sh embedded
```

### Contoh Flink SQL

```sql
-- ============================================
-- 🔥 Flink SQL Examples
-- ============================================

-- Set execution mode
SET 'execution.runtime-mode' = 'streaming';
SET 'sql-client.execution.result-mode' = 'changelog';

-- ========================================
-- 📡 Source: Membaca dari Kafka
-- ========================================
CREATE TABLE kafka_orders (
    order_id       BIGINT,
    user_id        BIGINT,
    product_name   STRING,
    quantity        INT,
    total_price    DECIMAL(15, 2),
    order_status   STRING,
    order_time     TIMESTAMP(3),
    -- Definisi watermark untuk event-time processing
    WATERMARK FOR order_time AS order_time - INTERVAL '5' SECOND
) WITH (
    'connector' = 'kafka',
    'topic' = 'orders',
    'properties.bootstrap.servers' = 'kafka:9092',
    'properties.group.id' = 'flink-sql-consumer',
    'scan.startup.mode' = 'earliest-offset',
    'format' = 'json',
    'json.fail-on-missing-field' = 'false',
    'json.ignore-parse-errors' = 'true'
);

-- ========================================
-- 🎯 Sink: Menulis ke Apache Doris
-- ========================================
CREATE TABLE doris_order_summary (
    window_start   TIMESTAMP(3),
    window_end     TIMESTAMP(3),
    order_status   STRING,
    total_orders   BIGINT,
    total_revenue  DECIMAL(15, 2),
    avg_quantity   DOUBLE
) WITH (
    'connector' = 'doris',
    'fenodes' = 'doris-fe:8030',
    'table.identifier' = 'demo_db.order_summary',
    'username' = 'root',
    'password' = '',
    'sink.properties.format' = 'json',
    'sink.properties.read_json_by_line' = 'true',
    'sink.enable-2pc' = 'true',
    'sink.label-prefix' = 'flink_doris'
);

-- ========================================
-- 🔄 Transformation: Window Aggregation
-- ========================================
INSERT INTO doris_order_summary
SELECT 
    window_start,
    window_end,
    order_status,
    COUNT(*) AS total_orders,
    SUM(total_price) AS total_revenue,
    AVG(CAST(quantity AS DOUBLE)) AS avg_quantity
FROM TABLE(
    TUMBLE(TABLE kafka_orders, DESCRIPTOR(order_time), INTERVAL '5' MINUTE)
)
GROUP BY window_start, window_end, order_status;

-- ========================================
-- 📊 DataGen untuk testing (tanpa Kafka)
-- ========================================
CREATE TABLE datagen_source (
    order_id    BIGINT,
    user_id     BIGINT,
    amount      DECIMAL(10, 2),
    order_time  TIMESTAMP(3),
    WATERMARK FOR order_time AS order_time - INTERVAL '3' SECOND
) WITH (
    'connector' = 'datagen',
    'rows-per-second' = '100',
    'fields.order_id.kind' = 'sequence',
    'fields.order_id.start' = '1',
    'fields.order_id.end' = '100000',
    'fields.user_id.min' = '1',
    'fields.user_id.max' = '1000',
    'fields.amount.min' = '10',
    'fields.amount.max' = '10000'
);

-- Print ke console untuk debugging
CREATE TABLE print_sink (
    window_start  TIMESTAMP(3),
    total_orders  BIGINT,
    total_amount  DECIMAL(15, 2)
) WITH (
    'connector' = 'print'
);

INSERT INTO print_sink
SELECT 
    window_start,
    COUNT(*) AS total_orders,
    SUM(amount) AS total_amount
FROM TABLE(
    TUMBLE(TABLE datagen_source, DESCRIPTOR(order_time), INTERVAL '10' SECOND)
)
GROUP BY window_start;
```

---

## Contoh Job: Stream Processing

### Java Flink Job (DataStream API)

```java
// pom.xml dependencies:
// - flink-streaming-java
// - flink-connector-kafka
// - flink-connector-jdbc

import org.apache.flink.api.common.eventtime.WatermarkStrategy;
import org.apache.flink.api.common.functions.MapFunction;
import org.apache.flink.api.common.serialization.SimpleStringSchema;
import org.apache.flink.connector.kafka.source.KafkaSource;
import org.apache.flink.connector.kafka.source.enumerator.initializer.OffsetsInitializer;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.windowing.assigners.TumblingEventTimeWindows;
import org.apache.flink.streaming.api.windowing.time.Time;

public class OrderStreamJob {
    public static void main(String[] args) throws Exception {
        // 1️⃣ Setup execution environment
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(4);
        env.enableCheckpointing(60000); // 60 seconds

        // 2️⃣ Define Kafka Source
        KafkaSource<String> kafkaSource = KafkaSource.<String>builder()
            .setBootstrapServers("kafka:9092")
            .setTopics("orders")
            .setGroupId("flink-order-processor")
            .setStartingOffsets(OffsetsInitializer.earliest())
            .setValueOnlyDeserializer(new SimpleStringSchema())
            .build();

        // 3️⃣ Create DataStream
        DataStream<String> stream = env.fromSource(
            kafkaSource,
            WatermarkStrategy.forBoundedOutOfOrderness(Duration.ofSeconds(5)),
            "Kafka Source"
        );

        // 4️⃣ Transform & Process
        DataStream<OrderEvent> orders = stream
            .map(json -> parseOrder(json))    // Parse JSON
            .filter(order -> order.getAmount() > 0)  // Filter valid
            .keyBy(OrderEvent::getUserId)     // Key by user
            .window(TumblingEventTimeWindows.of(Time.minutes(5)))
            .reduce((a, b) -> {
                a.setTotalAmount(a.getTotalAmount() + b.getTotalAmount());
                a.setOrderCount(a.getOrderCount() + 1);
                return a;
            });

        // 5️⃣ Sink to target
        orders.addSink(new DorisSink<>());

        // 6️⃣ Execute
        env.execute("Order Stream Processing Job");
    }
}
```

### Submit Job

```bash
# Submit JAR job ke Flink cluster
./bin/flink run \
  --detached \
  --class com.example.OrderStreamJob \
  /path/to/order-stream-job.jar

# List running jobs
./bin/flink list --running

# Cancel job
./bin/flink cancel <job-id>

# Savepoint (untuk upgrade tanpa data loss)
./bin/flink savepoint <job-id> /opt/flink/savepoints/

# Resume dari savepoint
./bin/flink run \
  --fromSavepoint /opt/flink/savepoints/savepoint-xxx \
  --class com.example.OrderStreamJob \
  /path/to/order-stream-job.jar
```

---

## Monitoring & Tuning

### Key Metrics untuk Monitor

```bash
# Cek overview cluster
curl -s http://localhost:8081/overview | python3 -m json.tool

# Cek job details
curl -s http://localhost:8081/jobs | python3 -m json.tool

# TaskManager metrics
curl -s http://localhost:8081/taskmanagers | python3 -m json.tool
```

### Performance Tuning Tips

```yaml
# 1. Memory Tuning
taskmanager.memory.process.size: 8192m
taskmanager.memory.managed.fraction: 0.4  # untuk RocksDB state
taskmanager.memory.network.fraction: 0.1  # network buffers

# 2. Parallelism Tuning
# Set parallelism = jumlah Kafka partitions
parallelism.default: 8

# 3. Checkpoint Tuning (production)
execution.checkpointing.interval: 120000  # 2 menit
execution.checkpointing.min-pause: 60000
state.backend.incremental: true           # incremental checkpoints
state.backend.rocksdb.memory.managed: true

# 4. Network Buffer
taskmanager.network.memory.fraction: 0.1
taskmanager.network.memory.min: 64mb
taskmanager.network.memory.max: 1gb
```

---

## Troubleshooting

| Problem | Solusi |
|---------|--------|
| `Could not find a suitable table factory` | Pastikan connector JAR ada di `lib/` atau `lib/connectors/` |
| `ClassNotFoundException` | Download JAR yang sesuai versi Flink |
| `TaskManager lost` | Tambah memory atau kurangi parallelism |
| `Checkpoint timeout` | Increase `execution.checkpointing.timeout`, cek backpressure |
| `Backpressure HIGH` | Kurangi throughput source, tambah parallelism, atau optimize query |
| `OutOfMemoryError` | Increase `taskmanager.memory.process.size` |

### Debug Tips

```bash
# Cek logs JobManager
docker logs -f flink-jobmanager 2>&1 | tail -100

# Cek logs TaskManager
docker logs -f flink-taskmanager 2>&1 | tail -100

# Cek backpressure via REST API
curl http://localhost:8081/jobs/<job-id>/vertices/<vertex-id>/backpressure
```

---

## 📚 Referensi

- [Apache Flink Documentation](https://nightlies.apache.org/flink/flink-docs-stable/)
- [Flink SQL Connectors](https://nightlies.apache.org/flink/flink-docs-stable/docs/connectors/table/overview/)
- [Flink CDC Connectors](https://ververica.github.io/flink-cdc-connectors/)
- [Apache Flink GitHub](https://github.com/apache/flink)

---

> 📝 **Author**: Data Engineering Team  
> 📅 **Last Updated**: 2026-02-10  
> 🏷️ **Tags**: `apache-flink`, `stream-processing`, `real-time`, `cdc`, `etl`
