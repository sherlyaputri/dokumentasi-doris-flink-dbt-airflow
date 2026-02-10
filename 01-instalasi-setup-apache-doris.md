# 📦 Cara Instalasi & Setup Apache Doris

> **Apache Doris** adalah MPP (Massively Parallel Processing) analytical database yang sangat cepat untuk real-time analytics. Cocok untuk data warehouse, ad-hoc query, dan reporting.

---

## 📋 Table of Contents

- [Prerequisites](#prerequisites)
- [Arsitektur Apache Doris](#arsitektur-apache-doris)
- [Instalasi Apache Doris](#instalasi-apache-doris)
  - [Metode 1: Docker Compose (Recommended)](#metode-1-docker-compose-recommended)
  - [Metode 2: Manual Installation](#metode-2-manual-installation)
- [Konfigurasi Frontend (FE)](#konfigurasi-frontend-fe)
- [Konfigurasi Backend (BE)](#konfigurasi-backend-be)
- [Verifikasi Instalasi](#verifikasi-instalasi)
- [Basic Operations](#basic-operations)
- [Troubleshooting](#troubleshooting)

---

## Prerequisites

| Komponen         | Minimum Requirement       |
|------------------|---------------------------|
| **OS**           | Ubuntu 20.04+ / CentOS 7+ |
| **CPU**          | 4 Cores (8+ recommended)  |
| **RAM**          | 8 GB (16+ recommended)    |
| **Disk**         | 100 GB SSD                |
| **Java**         | JDK 8 atau JDK 11         |
| **Docker**       | 20.10+ (jika pakai Docker)|
| **Docker Compose** | v2.0+                   |

### Install Dependencies

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install Java (JDK 8)
sudo apt install -y openjdk-8-jdk

# Verify Java
java -version
# Output: openjdk version "1.8.0_xxx"

# Set JAVA_HOME
echo 'export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64' >> ~/.bashrc
source ~/.bashrc

# Install Docker & Docker Compose (jika belum ada)
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker $USER

# Install Docker Compose
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" \
  -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
```

### System Configuration

```bash
# Increase max map count (PENTING untuk Doris)
sudo sysctl -w vm.max_map_count=2000000
echo "vm.max_map_count=2000000" | sudo tee -a /etc/sysctl.conf

# Increase open file limit
echo "* soft nofile 65536" | sudo tee -a /etc/security/limits.conf
echo "* hard nofile 65536" | sudo tee -a /etc/security/limits.conf

# Disable swap (recommended)
sudo swapoff -a
```

---

## Arsitektur Apache Doris

```
┌─────────────────────────────────────────────────┐
│                  Apache Doris                   │
│                                                 │
│  ┌──────────────┐     ┌──────────────────────┐  │
│  │  Frontend(FE) │     │   Backend (BE)       │  │
│  │              │     │                      │  │
│  │ • MySQL Proto│     │ • Data Storage       │  │
│  │ • Meta Store │────▶│ • Query Execution    │  │
│  │ • Query Parse│     │ • Data Compaction    │  │
│  │ • Optimizer  │     │ • Tablet Management  │  │
│  └──────────────┘     └──────────────────────┘  │
│         │                      │                │
│         ▼                      ▼                │
│  ┌──────────────┐     ┌──────────────────────┐  │
│  │  Port: 8030  │     │   Port: 8040         │  │
│  │  Port: 9030  │     │   Port: 9060         │  │
│  │  (MySQL)     │     │   (Heartbeat)        │  │
│  └──────────────┘     └──────────────────────┘  │
└─────────────────────────────────────────────────┘
```

---

## Instalasi Apache Doris

### Metode 1: Docker Compose (Recommended)

Buat file `docker-compose.yml`:

```yaml
# docker-compose.yml
version: "3.8"

services:
  doris-fe:
    image: apache/doris:2.1.0-fe
    container_name: doris-fe
    hostname: doris-fe
    environment:
      - FE_SERVERS=fe1:172.20.80.2:9010
      - FE_ID=1
    ports:
      - "8030:8030"   # HTTP port
      - "9030:9030"   # MySQL protocol port
      - "9010:9010"   # Edit log port
    volumes:
      - doris-fe-data:/opt/apache-doris/fe/doris-meta
      - doris-fe-log:/opt/apache-doris/fe/log
    networks:
      doris-network:
        ipv4_address: 172.20.80.2
    restart: unless-stopped

  doris-be:
    image: apache/doris:2.1.0-be
    container_name: doris-be
    hostname: doris-be
    environment:
      - FE_SERVERS=fe1:172.20.80.2:9010
      - BE_ADDR=172.20.80.3:9050
    ports:
      - "8040:8040"   # HTTP port
      - "9050:9050"   # Thrift port
      - "9060:9060"   # BE port
    volumes:
      - doris-be-data:/opt/apache-doris/be/storage
      - doris-be-log:/opt/apache-doris/be/log
    depends_on:
      - doris-fe
    networks:
      doris-network:
        ipv4_address: 172.20.80.3
    restart: unless-stopped

volumes:
  doris-fe-data:
  doris-fe-log:
  doris-be-data:
  doris-be-log:

networks:
  doris-network:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.80.0/24
```

```bash
# Jalankan Docker Compose
docker compose up -d

# Cek status container
docker compose ps

# Lihat logs
docker compose logs -f doris-fe
docker compose logs -f doris-be
```

### Metode 2: Manual Installation

```bash
# Download Apache Doris (sesuaikan versi)
DORIS_VERSION="2.1.0"
wget https://apache-doris-releases.oss-accelerate.aliyuncs.com/apache-doris-${DORIS_VERSION}-bin-x64.tar.gz

# Extract
tar -xzf apache-doris-${DORIS_VERSION}-bin-x64.tar.gz
cd apache-doris-${DORIS_VERSION}-bin-x64

# Struktur folder
# ├── fe/           # Frontend
# │   ├── bin/
# │   ├── conf/
# │   └── lib/
# └── be/           # Backend
#     ├── bin/
#     ├── conf/
#     └── lib/
```

---

## Konfigurasi Frontend (FE)

Edit file `fe/conf/fe.conf`:

```properties
# ============================================
# Apache Doris FE Configuration
# ============================================

# Priority Networks - SESUAIKAN dengan IP server kamu
priority_networks = 192.168.1.0/24

# HTTP port untuk web UI
http_port = 8030

# MySQL protocol port (koneksi client)
query_port = 9030

# RPC port
rpc_port = 9020

# Edit log port (untuk replikasi FE)
edit_log_port = 9010

# Meta directory
meta_dir = /opt/apache-doris/fe/doris-meta

# JVM Memory Settings
JAVA_OPTS="-Xmx8192m -XX:+UseMembar -XX:SurvivorRatio=8 \
  -XX:MaxTenuringThreshold=7 -XX:+PrintGCDateStamps \
  -XX:+PrintGCDetails -XX:+UseConcMarkSweepGC \
  -XX:+UseParNewGC -XX:+CMSClassUnloadingEnabled \
  -XX:-CMSParallelRemarkEnabled -XX:CMSInitiatingOccupancyFraction=80 \
  -XX:SoftRefLRUPolicyMSPerMB=0"

# Log Settings
LOG_DIR = /opt/apache-doris/fe/log
sys_log_level = INFO
```

```bash
# Start FE
./fe/bin/start_fe.sh --daemon

# Cek FE status
curl http://localhost:8030/api/bootstrap
# Expected: {"status":"OK","msg":"Success"}
```

---

## Konfigurasi Backend (BE)

Edit file `be/conf/be.conf`:

```properties
# ============================================
# Apache Doris BE Configuration
# ============================================

# Priority Networks - SESUAIKAN dengan IP server
priority_networks = 192.168.1.0/24

# BE HTTP port
be_port = 9060

# Webserver port
webserver_port = 8040

# Heartbeat port
heartbeat_service_port = 9050

# Brpc port
brpc_port = 8060

# Storage root path (bisa multiple path, dipisah ;)
storage_root_path = /opt/apache-doris/be/storage

# Memory limit (80% dari total RAM)
mem_limit = 80%

# Storage directories
# Bisa menambah multiple disk untuk distribusi data
# storage_root_path = /disk1/doris;/disk2/doris;/disk3/doris
```

```bash
# Start BE
./be/bin/start_be.sh --daemon

# Register BE ke FE melalui MySQL client
mysql -h 127.0.0.1 -P 9030 -u root

# Di MySQL prompt:
ALTER SYSTEM ADD BACKEND "your_be_host:9050";

# Cek BE status
SHOW BACKENDS\G;
```

---

## Verifikasi Instalasi

### Connect via MySQL Client

```bash
# Install MySQL client
sudo apt install -y mysql-client

# Connect ke Doris (default: root tanpa password)
mysql -h 127.0.0.1 -P 9030 -u root
```

### Test Query

```sql
-- ✅ Cek versi
SELECT VERSION();

-- ✅ Cek status FE
SHOW FRONTENDS\G;

-- ✅ Cek status BE
SHOW BACKENDS\G;

-- ✅ Cek alive status
SHOW PROC '/backends'\G;
```

---

## Basic Operations

### Membuat Database & Table

```sql
-- 🗄️ Buat Database
CREATE DATABASE IF NOT EXISTS demo_db;
USE demo_db;

-- 📊 Buat Table (Duplicate Model - simpan semua data)
CREATE TABLE IF NOT EXISTS users (
    user_id        BIGINT         NOT NULL COMMENT "User ID",
    username       VARCHAR(100)   NOT NULL COMMENT "Username",
    email          VARCHAR(255)   NOT NULL COMMENT "Email Address",
    age            INT            NULL     COMMENT "Usia",
    city           VARCHAR(100)   NULL     COMMENT "Kota",
    signup_date    DATE           NOT NULL COMMENT "Tanggal Registrasi",
    last_login     DATETIME       NULL     COMMENT "Login Terakhir",
    is_active      BOOLEAN        DEFAULT TRUE COMMENT "Status Aktif"
)
DUPLICATE KEY(user_id)
DISTRIBUTED BY HASH(user_id) BUCKETS 8
PROPERTIES (
    "replication_allocation" = "tag.location.default: 1",
    "storage_format" = "V2",
    "enable_unique_key_merge_on_write" = "false"
);

-- 📊 Buat Table (Unique Model - untuk data yang sering di-update)
CREATE TABLE IF NOT EXISTS orders (
    order_id       BIGINT         NOT NULL COMMENT "Order ID",
    user_id        BIGINT         NOT NULL COMMENT "User ID",
    product_name   VARCHAR(255)   NOT NULL COMMENT "Nama Produk",
    quantity       INT            NOT NULL COMMENT "Jumlah",
    total_price    DECIMAL(15,2)  NOT NULL COMMENT "Total Harga",
    order_status   VARCHAR(50)    DEFAULT 'pending' COMMENT "Status",
    created_at     DATETIME       NOT NULL COMMENT "Waktu Order",
    updated_at     DATETIME       NULL     COMMENT "Waktu Update"
)
UNIQUE KEY(order_id)
DISTRIBUTED BY HASH(order_id) BUCKETS 8
PROPERTIES (
    "replication_allocation" = "tag.location.default: 1",
    "enable_unique_key_merge_on_write" = "true"
);

-- 📊 Buat Table (Aggregate Model - untuk data agregasi)
CREATE TABLE IF NOT EXISTS page_views (
    event_date     DATE           NOT NULL COMMENT "Tanggal Event",
    page_url       VARCHAR(500)   NOT NULL COMMENT "URL Halaman",
    device_type    VARCHAR(50)    NOT NULL COMMENT "Tipe Device",
    view_count     BIGINT         SUM      COMMENT "Total Views",
    unique_users   BITMAP         BITMAP_UNION COMMENT "Unique Users"
)
AGGREGATE KEY(event_date, page_url, device_type)
PARTITION BY RANGE(event_date) (
    PARTITION p202601 VALUES LESS THAN ("2026-02-01"),
    PARTITION p202602 VALUES LESS THAN ("2026-03-01"),
    PARTITION p202603 VALUES LESS THAN ("2026-04-01")
)
DISTRIBUTED BY HASH(page_url) BUCKETS 8
PROPERTIES (
    "replication_allocation" = "tag.location.default: 1",
    "dynamic_partition.enable" = "true",
    "dynamic_partition.time_unit" = "MONTH",
    "dynamic_partition.end" = "3",
    "dynamic_partition.prefix" = "p",
    "dynamic_partition.buckets" = "8"
);
```

### Insert & Query Data

```sql
-- 📝 Insert Data
INSERT INTO users VALUES
(1, 'budi_dev', 'budi@example.com', 25, 'Jakarta', '2024-01-15', '2026-02-10 10:30:00', TRUE),
(2, 'siti_data', 'siti@example.com', 28, 'Bandung', '2024-03-20', '2026-02-09 14:20:00', TRUE),
(3, 'andi_ops', 'andi@example.com', 32, 'Surabaya', '2023-11-05', '2026-02-08 09:15:00', FALSE),
(4, 'maya_eng', 'maya@example.com', 26, 'Yogyakarta', '2024-06-10', '2026-02-10 12:00:00', TRUE),
(5, 'reza_sec', 'reza@example.com', 30, 'Medan', '2024-02-28', '2026-02-07 16:45:00', TRUE);

INSERT INTO orders VALUES
(1001, 1, 'Laptop ASUS ROG', 1, 25000000.00, 'completed', '2026-02-01 10:00:00', '2026-02-02 15:30:00'),
(1002, 2, 'Mechanical Keyboard', 2, 3500000.00, 'shipped', '2026-02-05 11:30:00', '2026-02-06 09:00:00'),
(1003, 1, 'Monitor 4K 27"', 1, 8500000.00, 'pending', '2026-02-10 08:00:00', NULL),
(1004, 3, 'Mouse Wireless', 3, 1200000.00, 'completed', '2026-02-03 14:00:00', '2026-02-04 10:00:00'),
(1005, 4, 'USB Hub Type-C', 1, 450000.00, 'processing', '2026-02-09 16:00:00', NULL);

-- 🔍 Query Data
SELECT * FROM users WHERE is_active = TRUE;

-- Join Query
SELECT 
    u.username,
    u.email,
    o.product_name,
    o.total_price,
    o.order_status
FROM users u
INNER JOIN orders o ON u.user_id = o.user_id
WHERE o.order_status = 'completed'
ORDER BY o.total_price DESC;

-- Aggregate Query
SELECT 
    u.city,
    COUNT(DISTINCT o.order_id) AS total_orders,
    SUM(o.total_price) AS total_revenue,
    AVG(o.total_price) AS avg_order_value
FROM users u
LEFT JOIN orders o ON u.user_id = o.user_id
GROUP BY u.city
ORDER BY total_revenue DESC;
```

### Load Data via Stream Load

```bash
# Stream Load dari file CSV
curl -u root: \
  -H "label:load_users_$(date +%Y%m%d%H%M%S)" \
  -H "column_separator:," \
  -H "columns: user_id, username, email, age, city, signup_date, last_login, is_active" \
  -T /path/to/users.csv \
  http://localhost:8030/api/demo_db/users/_stream_load

# Stream Load dari file JSON
curl -u root: \
  -H "label:load_orders_json_$(date +%Y%m%d%H%M%S)" \
  -H "format:json" \
  -H "strip_outer_array:true" \
  -T /path/to/orders.json \
  http://localhost:8030/api/demo_db/orders/_stream_load
```

---

## Troubleshooting

### Common Issues

| Problem | Solusi |
|---------|--------|
| `Connection refused on 9030` | Pastikan FE sudah running: `curl http://localhost:8030/api/bootstrap` |
| `Backend not alive` | Cek BE log: `tail -f be/log/be.WARNING`, pastikan `priority_networks` benar |
| `No alive backends` | Register BE: `ALTER SYSTEM ADD BACKEND "host:9050";` |
| `Tablet not enough replicas` | Set replication ke 1 untuk single node: `"replication_allocation" = "tag.location.default: 1"` |
| `Java heap space` | Increase `JAVA_OPTS` di `fe.conf`: `-Xmx` dan `-Xms` |

### Useful Commands

```bash
# Cek FE log
tail -100f fe/log/fe.log

# Cek BE log
tail -100f be/log/be.INFO

# Restart FE
./fe/bin/stop_fe.sh && ./fe/bin/start_fe.sh --daemon

# Restart BE
./be/bin/stop_be.sh && ./be/bin/start_be.sh --daemon

# Docker: restart semua
docker compose restart
```

---

## 📚 Referensi

- [Apache Doris Official Documentation](https://doris.apache.org/docs/)
- [Apache Doris GitHub](https://github.com/apache/doris)
- [Docker Hub - Apache Doris](https://hub.docker.com/r/apache/doris)

---

> 📝 **Author**: Data Engineering Team  
> 📅 **Last Updated**: 2026-02-10  
> 🏷️ **Tags**: `apache-doris`, `olap`, `data-warehouse`, `analytics`
