# 🗃️ Cara Instalasi & Setup CrateDB

> **CrateDB** adalah distributed SQL database yang dibangun di atas teknologi NoSQL. CrateDB menggabungkan fleksibilitas JSON/object support dengan kekuatan SQL standar, ideal untuk IoT, time-series, dan geospatial data.

---

## 📋 Table of Contents

- [Prerequisites](#prerequisites)
- [Arsitektur CrateDB](#arsitektur-cratedb)
- [Instalasi CrateDB](#instalasi-cratedb)
  - [Metode 1: Docker (Recommended)](#metode-1-docker-recommended)
  - [Metode 2: Docker Compose Cluster](#metode-2-docker-compose-cluster)
  - [Metode 3: Manual Installation (Debian/Ubuntu)](#metode-3-manual-installation)
- [Konfigurasi CrateDB](#konfigurasi-cratedb)
- [Akses Admin UI](#akses-admin-ui)
- [Basic Operations](#basic-operations)
- [Advanced Features](#advanced-features)
- [Monitoring & Maintenance](#monitoring--maintenance)
- [Troubleshooting](#troubleshooting)

---

## Prerequisites

| Komponen         | Minimum Requirement         |
|------------------|-----------------------------|
| **OS**           | Ubuntu 20.04+ / CentOS 7+  |
| **CPU**          | 4 Cores                     |
| **RAM**          | 4 GB (8+ recommended)       |
| **Disk**         | 50 GB SSD                   |
| **Java**         | JDK 11+ (bundled di CrateDB)|
| **Docker**       | 20.10+ (jika pakai Docker)  |

### System Tuning

```bash
# ⚙️ CrateDB butuh memory mapping yang besar
sudo sysctl -w vm.max_map_count=262144
echo "vm.max_map_count=262144" | sudo tee -a /etc/sysctl.conf

# Disable swap
sudo swapoff -a

# Increase file descriptors
echo "crate soft nofile 65536" | sudo tee -a /etc/security/limits.conf
echo "crate hard nofile 65536" | sudo tee -a /etc/security/limits.conf
```

---

## Arsitektur CrateDB

```
┌──────────────────────────────────────────────────────┐
│                   CrateDB Cluster                    │
│                                                      │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐     │
│  │  Node 1    │  │  Node 2    │  │  Node 3    │     │
│  │  (Master)  │  │  (Data)    │  │  (Data)    │     │
│  │            │  │            │  │            │     │
│  │ ┌────────┐ │  │ ┌────────┐ │  │ ┌────────┐ │     │
│  │ │ Shard  │ │  │ │ Shard  │ │  │ │ Shard  │ │     │
│  │ │  0-P   │ │  │ │  1-P   │ │  │ │  2-P   │ │     │
│  │ │  1-R   │ │  │ │  2-R   │ │  │ │  0-R   │ │     │
│  │ └────────┘ │  │ └────────┘ │  │ └────────┘ │     │
│  └────────────┘  └────────────┘  └────────────┘     │
│       │                │               │             │
│       ▼                ▼               ▼             │
│  ┌─────────────────────────────────────────────┐     │
│  │  PostgreSQL Wire Protocol (Port: 5432)      │     │
│  │  HTTP Endpoint (Port: 4200) - Admin UI      │     │
│  │  Transport (Port: 4300) - Inter-node         │     │
│  └─────────────────────────────────────────────┘     │
└──────────────────────────────────────────────────────┘
```

---

## Instalasi CrateDB

### Metode 1: Docker (Recommended)

```bash
# 🐳 Single node CrateDB
docker run -d \
  --name cratedb \
  --publish 4200:4200 \
  --publish 5432:5432 \
  --env CRATE_HEAP_SIZE=2g \
  crate:latest \
  -Cdiscovery.type=single-node

# Verifikasi
docker logs -f cratedb

# Cek status via HTTP
curl http://localhost:4200/
# Response: { "ok": true, "status": 200, ... }
```

### Metode 2: Docker Compose Cluster

```yaml
# docker-compose.yml - CrateDB 3-Node Cluster
version: "3.8"

services:
  cratedb-01:
    image: crate:latest
    container_name: cratedb-01
    hostname: cratedb-01
    ports:
      - "4200:4200"
      - "5432:5432"
    volumes:
      - crate-data-01:/data
    environment:
      CRATE_HEAP_SIZE: "2g"
    command:
      - crate
      - -Ccluster.name=crate-cluster
      - -Cnode.name=cratedb-01
      - -Cnode.attr.zone=a
      - -Cdiscovery.seed_hosts=cratedb-01,cratedb-02,cratedb-03
      - -Ccluster.initial_master_nodes=cratedb-01,cratedb-02,cratedb-03
      - -Cgateway.expected_data_nodes=3
      - -Cgateway.recover_after_data_nodes=2
    deploy:
      resources:
        limits:
          memory: 4g
    networks:
      - crate-network

  cratedb-02:
    image: crate:latest
    container_name: cratedb-02
    hostname: cratedb-02
    ports:
      - "4201:4200"
    volumes:
      - crate-data-02:/data
    environment:
      CRATE_HEAP_SIZE: "2g"
    command:
      - crate
      - -Ccluster.name=crate-cluster
      - -Cnode.name=cratedb-02
      - -Cnode.attr.zone=b
      - -Cdiscovery.seed_hosts=cratedb-01,cratedb-02,cratedb-03
      - -Ccluster.initial_master_nodes=cratedb-01,cratedb-02,cratedb-03
      - -Cgateway.expected_data_nodes=3
      - -Cgateway.recover_after_data_nodes=2
    deploy:
      resources:
        limits:
          memory: 4g
    networks:
      - crate-network

  cratedb-03:
    image: crate:latest
    container_name: cratedb-03
    hostname: cratedb-03
    ports:
      - "4202:4200"
    volumes:
      - crate-data-03:/data
    environment:
      CRATE_HEAP_SIZE: "2g"
    command:
      - crate
      - -Ccluster.name=crate-cluster
      - -Cnode.name=cratedb-03
      - -Cnode.attr.zone=c
      - -Cdiscovery.seed_hosts=cratedb-01,cratedb-02,cratedb-03
      - -Ccluster.initial_master_nodes=cratedb-01,cratedb-02,cratedb-03
      - -Cgateway.expected_data_nodes=3
      - -Cgateway.recover_after_data_nodes=2
    deploy:
      resources:
        limits:
          memory: 4g
    networks:
      - crate-network

volumes:
  crate-data-01:
  crate-data-02:
  crate-data-03:

networks:
  crate-network:
    driver: bridge
```

```bash
# Jalankan cluster
docker compose up -d

# Cek status cluster
curl "http://localhost:4200/_sql" \
  -H 'Content-Type: application/json' \
  -d '{"stmt": "SELECT id, name, hostname FROM sys.nodes ORDER BY name"}'
```

### Metode 3: Manual Installation

```bash
# 📦 Install via APT (Debian/Ubuntu)

# Import GPG key
wget https://cdn.crate.io/downloads/deb/DEB-GPG-KEY-crate
sudo apt-key add DEB-GPG-KEY-crate

# Add repository
echo "deb https://cdn.crate.io/downloads/deb/stable/ default main" \
  | sudo tee /etc/apt/sources.list.d/crate-stable.list

# Install
sudo apt update
sudo apt install -y crate

# Start service
sudo systemctl enable crate
sudo systemctl start crate

# Cek status
sudo systemctl status crate
```

---

## Konfigurasi CrateDB

File konfigurasi: `/etc/crate/crate.yml`

```yaml
# ============================================
# CrateDB Configuration
# ============================================

# Cluster Settings
cluster:
  name: my-crate-cluster
  routing:
    allocation:
      disk.watermark.low: "85%"
      disk.watermark.high: "90%"
      disk.watermark.flood_stage: "95%"

# Node Settings
node:
  name: crate-node-01
  max_local_storage_nodes: 1

# Network Settings
network:
  host: 0.0.0.0

# HTTP Settings (Admin UI)
http:
  port: 4200
  cors:
    enabled: true
    allow-origin: "*"

# PostgreSQL Wire Protocol
psql:
  port: 5432
  enabled: true

# Transport Settings (inter-node)
transport:
  port: 4300

# Path Settings
path:
  data: /var/lib/crate
  logs: /var/log/crate

# Gateway Settings
gateway:
  expected_data_nodes: 1
  recover_after_data_nodes: 1

# Memory Settings (di /etc/default/crate)
# CRATE_HEAP_SIZE=2g
```

---

## Akses Admin UI

CrateDB menyediakan **Admin UI** bawaan yang sangat berguna:

```
🌐 URL: http://localhost:4200/
```

Fitur Admin UI:
- **Overview**: Health cluster, jumlah nodes, dan shards
- **Console**: SQL console untuk menjalankan query
- **Tables**: Browse & inspect tabel
- **Cluster**: Monitor node health dan distribusi data
- **Monitoring**: Query statistics & performance metrics

---

## Basic Operations

### Connect via PostgreSQL Client

```bash
# Connect pakai psql
psql -h localhost -p 5432 -U crate

# Atau pakai crash (CrateDB shell)
pip install crash
crash --host localhost:4200
```

### CRUD Operations

```sql
-- 🗄️ Buat Table
CREATE TABLE IF NOT EXISTS sensor_data (
    sensor_id    TEXT NOT NULL,
    ts           TIMESTAMP WITH TIME ZONE NOT NULL,
    location     GEO_POINT,
    temperature  DOUBLE PRECISION,
    humidity     DOUBLE PRECISION,
    pressure     DOUBLE PRECISION,
    metadata     OBJECT (DYNAMIC) AS (
        firmware_version TEXT,
        battery_level    INTEGER,
        tags             ARRAY(TEXT)
    ),
    raw_payload  OBJECT (IGNORED)
) CLUSTERED BY (sensor_id) INTO 6 SHARDS
WITH (
    number_of_replicas = '1',
    column_policy = 'dynamic',
    "write.wait_for_active_shards" = 1,
    refresh_interval = 1000
);

-- 📝 Insert Data
INSERT INTO sensor_data (sensor_id, ts, location, temperature, humidity, pressure, metadata)
VALUES
    ('SENSOR-001', NOW(), [106.8456, -6.2088], 28.5, 75.2, 1013.25,
     {firmware_version = '2.1.0', battery_level = 85, tags = ['outdoor', 'jakarta']}),
    ('SENSOR-002', NOW(), [107.6191, -6.9175], 24.3, 82.1, 1015.50,
     {firmware_version = '2.1.0', battery_level = 92, tags = ['indoor', 'bandung']}),
    ('SENSOR-003', NOW(), [112.7508, -7.2575], 31.2, 68.5, 1010.80,
     {firmware_version = '2.0.5', battery_level = 45, tags = ['outdoor', 'surabaya']}),
    ('SENSOR-004', NOW(), [110.4203, -6.9666], 26.8, 78.9, 1012.30,
     {firmware_version = '2.1.0', battery_level = 71, tags = ['indoor', 'semarang']}),
    ('SENSOR-005', NOW(), [98.6722, 3.5952], 33.1, 65.0, 1009.15,
     {firmware_version = '1.9.8', battery_level = 23, tags = ['outdoor', 'medan']});

-- Refresh table untuk memastikan data bisa di-query
REFRESH TABLE sensor_data;

-- 🔍 Select Query
SELECT sensor_id, temperature, humidity, metadata['battery_level'] AS battery
FROM sensor_data
WHERE temperature > 25
ORDER BY temperature DESC;

-- 📊 Aggregate Query
SELECT 
    metadata['tags'][1] AS city,
    COUNT(*) AS sensor_count,
    AVG(temperature) AS avg_temp,
    MIN(humidity) AS min_humidity,
    MAX(humidity) AS max_humidity
FROM sensor_data
GROUP BY 1
ORDER BY avg_temp DESC;

-- 🔄 Update
UPDATE sensor_data
SET metadata['battery_level'] = 100
WHERE sensor_id = 'SENSOR-005';

-- 🗑️ Delete
DELETE FROM sensor_data
WHERE metadata['battery_level'] < 30;
```

---

## Advanced Features

### Full-Text Search

```sql
-- Buat table dengan fulltext index
CREATE TABLE articles (
    id         TEXT PRIMARY KEY,
    title      TEXT INDEX USING FULLTEXT WITH (analyzer = 'standard'),
    content    TEXT INDEX USING FULLTEXT WITH (analyzer = 'standard'),
    author     TEXT,
    published  TIMESTAMP WITH TIME ZONE,
    tags       ARRAY(TEXT)
);

-- Insert sample articles
INSERT INTO articles (id, title, content, author, published, tags)
VALUES
    ('art-001', 'Introduction to CrateDB',
     'CrateDB is a distributed SQL database designed for machine data. It combines the familiarity of SQL with the scalability of NoSQL.',
     'John Doe', '2026-01-15', ['database', 'sql', 'nosql']),
    ('art-002', 'Real-time IoT Analytics',
     'Using CrateDB for real-time analytics on IoT sensor data provides millisecond query response times.',
     'Jane Smith', '2026-02-01', ['iot', 'analytics', 'real-time']);

REFRESH TABLE articles;

-- Full-text search
SELECT id, title, _score
FROM articles
WHERE MATCH(content, 'distributed SQL scalability')
ORDER BY _score DESC;
```

### Geospatial Queries

```sql
-- 🗺️ Query berdasarkan lokasi (radius 50km dari Jakarta)
SELECT sensor_id, temperature,
       DISTANCE(location, [106.8456, -6.2088]) / 1000 AS distance_km
FROM sensor_data
WHERE DISTANCE(location, [106.8456, -6.2088]) < 50000
ORDER BY distance_km ASC;

-- Query dalam bounding box
SELECT sensor_id, location
FROM sensor_data
WHERE WITHIN(location, {
    type = 'Polygon',
    coordinates = [[[106.0, -7.0], [108.0, -7.0], [108.0, -6.0], [106.0, -6.0], [106.0, -7.0]]]
});
```

### Generated Columns & Partitioned Tables

```sql
-- Table dengan generated column & partition
CREATE TABLE IF NOT EXISTS event_logs (
    event_id     TEXT NOT NULL,
    event_type   TEXT NOT NULL,
    event_time   TIMESTAMP WITH TIME ZONE NOT NULL,
    event_month  TIMESTAMP WITH TIME ZONE
        GENERATED ALWAYS AS DATE_TRUNC('month', event_time),
    payload      OBJECT (DYNAMIC),
    severity     TEXT DEFAULT 'info'
) PARTITIONED BY (event_month)
CLUSTERED BY (event_id) INTO 4 SHARDS
WITH (number_of_replicas = 0);
```

---

## Monitoring & Maintenance

### Health Checks

```sql
-- 🏥 Cluster health
SELECT id, name, hostname, heap['used'] AS heap_used, 
       heap['max'] AS heap_max, fs['total']['available'] AS disk_available
FROM sys.nodes;

-- Table health
SELECT table_name, number_of_shards, number_of_replicas,
       health
FROM information_schema.tables
WHERE table_schema = 'doc';

-- Shard allocation
SELECT table_name, id, "primary", state, relocating_node
FROM sys.shards
WHERE table_name = 'sensor_data';

-- Active queries
SELECT * FROM sys.jobs ORDER BY started DESC LIMIT 10;

-- Query stats
SELECT stmt, count, avg_duration, max_duration
FROM sys.jobs_log
ORDER BY max_duration DESC
LIMIT 10;
```

### Backup & Restore

```bash
# Buat snapshot repository
curl -X PUT "http://localhost:4200/_sql" \
  -H 'Content-Type: application/json' \
  -d '{
    "stmt": "CREATE REPOSITORY my_backup TYPE fs WITH (location = '\''/backup/cratedb'\'', compress = true)"
  }'
```

```sql
-- Buat snapshot
CREATE SNAPSHOT my_backup.snapshot_20260210 ALL
WITH (wait_for_completion = true);

-- Restore snapshot
RESTORE SNAPSHOT my_backup.snapshot_20260210 ALL
WITH (wait_for_completion = true);

-- List snapshots
SELECT * FROM sys.snapshots;
```

---

## Troubleshooting

| Problem | Solusi |
|---------|--------|
| `max virtual memory areas too low` | `sudo sysctl -w vm.max_map_count=262144` |
| `UNASSIGNED shards` | `ALTER TABLE t SET (number_of_replicas = 0)` lalu set ulang |
| `CircuitBreakingException` | Kurangi `indices.breaker.query.limit` atau tambah RAM |
| `ShardNotAvailable` | Tunggu recovery, cek `SELECT * FROM sys.shards` |
| `Cluster RED` | Cek node yang mati, pastikan semua node online |
| `Connection refused 4200` | Pastikan service running: `sudo systemctl status crate` |

### Useful Debug Commands

```sql
-- Cek cluster settings
SELECT settings FROM sys.cluster;

-- Cek allocations yang gagal
SELECT table_name, shard_id, node_id, decisions
FROM sys.allocations
WHERE decisions['type'] = 'NO';

-- Kill long-running query  
KILL '<job_id>';

-- Optimize table
OPTIMIZE TABLE sensor_data;
```

---

## 📚 Referensi

- [CrateDB Official Documentation](https://cratedb.com/docs/)
- [CrateDB SQL Reference](https://cratedb.com/docs/crate/reference/en/latest/sql/)
- [CrateDB GitHub](https://github.com/crate/crate)
- [CrateDB Docker Hub](https://hub.docker.com/_/crate)

---

> 📝 **Author**: Data Engineering Team  
> 📅 **Last Updated**: 2026-02-10  
> 🏷️ **Tags**: `cratedb`, `distributed-database`, `iot`, `time-series`, `sql`
