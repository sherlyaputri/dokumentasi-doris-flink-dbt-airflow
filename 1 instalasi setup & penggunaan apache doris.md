# 📦 Cara Instalasi & Setup Apache Doris

> **Apache Doris** adalah MPP (Massively Parallel Processing) analytical database yang sangat cepat untuk real-time analytics. Cocok untuk data warehouse, ad-hoc query, dan reporting.

---

## 📋 Table of Contents

- [Prerequisites](#prerequisites)
- [Arsitektur Apache Doris](#arsitektur-apache-doris)
- [Instalasi Apache Doris](#instalasi-apache-doris)
- [Konfigurasi Frontend (FE)](#konfigurasi-frontend-fe)
- [Konfigurasi Backend (BE)](#konfigurasi-backend-be)
- [Konfigurasi Kernel & System Limits](#konfigurasi-kernel--system-limits)
- [Verifikasi Instalasi](#verifikasi-instalasi)
- [Basic Operations](#basic-operations)
- [Troubleshooting](#troubleshooting)

---

## Prerequisites

| Komponen | Minimum Requirement      |
| -------- | ------------------------ |
| **OS**   | Debian 13 Trixie         |
| **CPU**  | 4 Cores (8+ recommended) |
| **RAM**  | 16 GB (24+ recommended)  |
| **Disk** | 100 GB SSD               |
| **Java** | JDK 17                   |

### Install Dependencies

```bash
# Update system
sudo apt update

# Install Java (JDK 17)
sudo apt install openjdk-17-jdk

# Verify Java
java -version
# Output: openjdk version "17.x.x"
```

---

## Arsitektur Apache Doris

```
┌─────────────────────────────────────────────────┐
│                  Apache Doris                   │
│                                                 │
│  ┌──────────────┐     ┌──────────────────────┐  │
│  │  Frontend(FE)│     │   Backend (BE)       │  │
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

```bash
# Download Apache Doris (sesuaikan versi)
wget https://apache-doris.selectdb.com/apache-doris-3.0.8-bin-x64-noavx2.tar.gz

# Extract
tar -xzf apache-doris-3.0.8-bin-x64-noavx2.tar.gz
cd apache-doris-3.0.8-bin-x64-noavx2
```

Struktur folder setelah extract:

```
apache-doris-3.0.8-bin-x64-noavx2/
├── fe/           # Frontend
│   ├── bin/
│   ├── conf/
│   └── lib/
└── be/           # Backend
    ├── bin/
    ├── conf/
    └── lib/
```

---

## Konfigurasi Frontend (FE)

> ⚠️ **Konfigurasi opsional** — edit hanya jika perlu menyesuaikan port atau network.

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

### Start & Cek FE

```bash
# Start FE
./fe/bin/start_fe.sh --daemon

# Cek FE status via API
curl http://localhost:8030/api/bootstrap
# Expected: {"status":"OK","msg":"Success"}

# Atau buka Web UI di browser
# http://localhost:8030/
```

---

## Konfigurasi Backend (BE)

> ⚠️ **Konfigurasi opsional** — edit hanya jika perlu menyesuaikan port atau network.

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
```

### Start BE

```bash
# Start BE
./be/bin/start_be.sh --daemon
```

---

## Konfigurasi Kernel & System Limits

> ❗ **Bagian ini diperlukan jika muncul error `vm.max_map_count` saat start BE.**
>
> Apache Doris Backend (BE) menggunakan mekanisme penyimpanan data yang sangat intensif dalam melakukan pemetaan memori (memory mapping), sehingga perlu menaikkan batas kernel.

### 1. Konfigurasi Virtual Memory (`vm.max_map_count`)

```bash
# Edit sysctl config
sudo vim /etc/sysctl.conf
```

Tambahkan baris berikut:

```
vm.max_map_count=2000000
```

Simpan & keluar (`:wq`), lalu terapkan tanpa restart:

```bash
sudo sysctl -p
```

### 2. Konfigurasi File Descriptors (Limits)

Doris membutuhkan izin untuk membuka banyak file sekaligus. Jika tidak diatur, BE akan gagal start atau sering crash.

```bash
# Edit limits config
sudo vim /etc/security/limits.conf
```

Tambahkan baris berikut **sebelum** baris `# End of file`:

```
* soft nofile 655350
* hard nofile 655350
```

Simpan & keluar (`:wq`), lalu paksa perubahan untuk sesi terminal yang sedang aktif:

```bash
ulimit -n 655350
```

### 3. Verifikasi Konfigurasi Kernel

```bash
# Cek vm.max_map_count (harus 2000000)
cat /proc/sys/vm/max_map_count

# Cek file descriptors (harus 655350)
ulimit -n
```

### 4. Jalankan Kembali Doris BE

```bash
./be/bin/start_be.sh --daemon

# Cek dengan JPS untuk melihat keduanya sudah aktif
jps
# Output yang diharapkan:
# DorisFE
# DorisBE
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

### Register BE ke FE

```sql
-- Di MySQL prompt, register BE
ALTER SYSTEM ADD BACKEND "your_be_host:9050";

-- Cek BE status
SHOW BACKENDS\G;
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

### Membuat Database, Table & Data Dummy

```sql
-- 🗄️ Buat Database
CREATE DATABASE IF NOT EXISTS demo_db;
USE demo_db;

-- 📊 Buat Table
CREATE TABLE IF NOT EXISTS sales_data_agg (
    user_id       INT NOT NULL,
    order_date    DATE NOT NULL,
    category      VARCHAR(20),
    amount        DOUBLE SUM
)
ENGINE=OLAP
AGGREGATE KEY(user_id, order_date, category)
DISTRIBUTED BY HASH(user_id) BUCKETS 10
PROPERTIES (
    "replication_num" = "1"
);

-- 📝 Insert Data Dummy
INSERT INTO sales_data_agg VALUES
(101, '2024-01-01', 'Electronics', 1500.50),
(102, '2024-01-01', 'Furniture', 890.00),
(103, '2024-01-02', 'Electronics', 450.25),
(104, '2024-01-02', 'Clothing', 120.00),
(105, '2024-01-03', 'Furniture', 2100.00);
```

---

## Troubleshooting

### Common Issues

| Problem                       | Solusi                                                                                           |
| ----------------------------- | ------------------------------------------------------------------------------------------------ |
| `Connection refused on 9030`  | Pastikan FE sudah running: `curl http://localhost:8030/api/bootstrap`                             |
| `Backend not alive`           | Cek BE log: `tail -f be/log/be.WARNING`, pastikan `priority_networks` benar                      |
| `No alive backends`           | Register BE: `ALTER SYSTEM ADD BACKEND "host:9050";`                                             |
| `Tablet not enough replicas`  | Set replication ke 1 untuk single node: `"replication_allocation" = "tag.location.default: 1"`    |
| `Java heap space`             | Increase `JAVA_OPTS` di `fe.conf`: `-Xmx` dan `-Xms`                                            |

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
```

---

## 📚 Referensi

- [Apache Doris Official Documentation](https://doris.apache.org)
- [Apache Doris GitHub](https://github.com/apache/doris)

---

> 📝 **Author**: Data Engineering Team
> 📅 **Last Updated**: 2026-02-11
> 🏷️ **Tags**: `apache-doris`, `olap`, `data-warehouse`, `analytics`
