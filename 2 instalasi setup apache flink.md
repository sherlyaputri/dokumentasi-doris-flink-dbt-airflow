# ⚡ Cara Instalasi & Setup Apache Flink

> **Apache Flink** adalah framework distributed stream processing untuk real-time data processing. Dalam dokumentasi ini, Flink digunakan untuk **migrasi data dari MariaDB ke Apache Doris** menggunakan Flink CDC & JDBC Connector.

---

## 📋 Table of Contents

- [Prerequisites](#prerequisites)
- [Arsitektur Apache Flink](#arsitektur-apache-flink)
- [Port yang Digunakan](#port-yang-digunakan)
- [Instalasi Apache Flink](#instalasi-apache-flink)
- [Konfigurasi Flink](#konfigurasi-flink-️-jika-perlu)
- [Download Connectors](#download-connectors-untuk-menjalankan-migrasi-data)
- [Start Flink Cluster](#start-flink-cluster)
- [Flink Web UI](#flink-web-ui)
- [Flink SQL Client](#flink-sql-client)
- [Troubleshooting](#troubleshooting)

---

## Prerequisites

| Komponen | Minimum Requirement      |
| -------- | ------------------------ |
| **OS**   | Debian 13 Trixie         |
| **CPU**  | 4 Cores (8+ recommended) |
| **RAM**  | 8 GB (16+ recommended)   |
| **Disk** | 50 GB SSD                |
| **Java** | JDK 17                   |

### Verifikasi Java 17

Java 17 sudah terinstall karena digunakan juga oleh Apache Doris.

```bash
# Verifikasi Java
java -version
# Output: openjdk version "17.x.x"
```

### Pastikan Service yang Dibutuhkan Sudah Running

| Service | Lokasi | Port | Status |
| ------- | ------ | ---- | ------ |
| **MariaDB** | Localhost (`127.0.0.1`) | 3306 | Harus sudah running (sumber data) |
| **Apache Doris FE** | Server SSH (`192.168.4.100`) | 18030, 19030 | Harus sudah running |
| **Apache Doris BE** | Server SSH (`192.168.4.100`) | 18040, 19060 | Harus sudah running |

```bash
# Cek MariaDB (localhost)
mysql -h 127.0.0.1 -P 3306 -u root -p -e "SELECT VERSION();"

# Cek Doris FE (server SSH)
curl http://192.168.4.100:18030/api/bootstrap
# Expected: {"status":"OK","msg":"Success"}

# Connect ke Doris via MySQL client
mysql -h 192.168.4.100 -P 19030 -u root
```

---

## Arsitektur Apache Flink

```
┌──────────────────────────────────────────────────────────────┐
│                     Apache Flink Cluster                     │
│                                                              │
│  ┌──────────────────────┐    ┌─────────────────────────────┐ │
│  │   JobManager (JM)    │    │   TaskManager (TM)          │ │
│  │                      │    │                             │ │
│  │ • Job Scheduling     │    │ • Task Execution            │ │
│  │ • Checkpoint Coord.  │◄──▶│ • Memory Management         │ │
│  │ • Resource Manager   │    │ • Network Buffers           │ │
│  │ • REST API (:8081)   │    │ • State Backend             │ │
│  └──────────────────────┘    └─────────────────────────────┘ │
└──────────────────────────────────────────────────────────────┘

Alur Migrasi (2 Metode):

  Metode 1 - JDBC (Batch / One-Time):
  MariaDB (localhost:3306) ──[JDBC]──▶ Flink ──[Doris Connector]──▶ Apache Doris (192.168.4.100)
  ✅ Job selesai otomatis setelah semua data tercopy

  Metode 2 - CDC (Real-Time / Streaming):
  MariaDB (localhost:3306) ──[CDC]──▶ Flink ──[Doris Connector]──▶ Apache Doris (192.168.4.100)
  🔄 Job berjalan terus, menangkap perubahan real-time
```

---

## Port yang Digunakan

### Port Apache Flink (Localhost)

| Port | Service | Keterangan |
| ---- | ------- | ---------- |
| **8081** | REST API / Web UI | Dashboard monitoring Flink |
| **6123** | JobManager RPC | Komunikasi internal JM ↔ TM |
| **6124** | JobManager Blob Server | Transfer file JAR antar node |
| **6125** | TaskManager Data | Transfer data antar TaskManager |

### Port MariaDB (Localhost)

| Port | Service | Keterangan |
| ---- | ------- | ---------- |
| **3306** | MariaDB | Sumber data (source) |

### Port Apache Doris (Server SSH: `192.168.4.100`)

| Port | Service | Keterangan |
| ---- | ------- | ---------- |
| **18030** | Doris FE HTTP | Web UI & REST API Doris (dipakai Flink Doris Connector) |
| **19030** | Doris FE MySQL | Koneksi MySQL client ke Doris |
| **18040** | Doris BE HTTP | Doris Backend webserver |
| **19060** | Doris BE Port | Doris Backend data port |

---

## Instalasi Apache Flink

```bash
# Download Apache Flink 1.18.0
wget https://archive.apache.org/dist/flink/flink-1.18.0/flink-1.18.0-bin-scala_2.12.tgz

# Extract
tar -xzf flink-1.18.0-bin-scala_2.12.tgz
cd flink-1.18.0
```

Struktur folder setelah extract:

```
flink-1.18.0/
├── bin/          # Scripts (start, stop, flink CLI, sql-client)
├── conf/         # Configuration files (flink-conf.yaml)
├── lib/          # Core libraries & connector JARs
├── opt/          # Optional libraries
├── plugins/      # Plugin directory
├── log/          # Log files (muncul setelah start)
└── examples/     # Contoh job bawaan
```

---

## Konfigurasi Flink ⚠️ Jika PERLU

Edit file `conf/flink-conf.yaml`:

```yaml
# ============================================
# Apache Flink 1.18.0 Configuration
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

# === Checkpointing ===
execution.checkpointing.interval: 60000
execution.checkpointing.mode: EXACTLY_ONCE
execution.checkpointing.min-pause: 500
execution.checkpointing.timeout: 600000
execution.checkpointing.max-concurrent-checkpoints: 1

# === State Backend ===
state.backend: rocksdb
state.checkpoints.dir: file:///tmp/flink-checkpoints
state.savepoints.dir: file:///tmp/flink-savepoints
state.backend.incremental: true

# === Restart Strategy ===
restart-strategy: fixed-delay
restart-strategy.fixed-delay.attempts: 3
restart-strategy.fixed-delay.delay: 30s
```

---

## Download Connectors untuk menjalankan migrasi data

Download connector JAR yang dibutuhkan ke folder `lib/`:

```bash
# Masuk ke folder lib
cd lib/

# ✅ 1. Flink JDBC Connector (untuk migrasi batch via JDBC)
wget https://repo1.maven.org/maven2/org/apache/flink/flink-connector-jdbc/3.1.2-1.18/flink-connector-jdbc-3.1.2-1.18.jar

# ✅ 2. Flink CDC MySQL/MariaDB Connector (untuk migrasi real-time via CDC)
wget https://repo1.maven.org/maven2/com/ververica/flink-sql-connector-mysql-cdc/3.0.1/flink-sql-connector-mysql-cdc-3.0.1.jar

# ✅ 3. Flink Doris Connector (untuk menulis ke Apache Doris)
wget https://repo1.maven.org/maven2/org/apache/doris/flink-doris-connector-1.18/24.0.0/flink-doris-connector-1.18-24.0.0.jar

# ✅ 4. MySQL JDBC Driver (dibutuhkan oleh JDBC & CDC connector)
wget https://repo1.maven.org/maven2/com/mysql/mysql-connector-j/8.3.0/mysql-connector-j-8.3.0.jar

# Kembali ke root folder Flink
cd ..
```

> 💡 **Catatan:** Semua connector JAR **harus** diletakkan di folder `lib/` agar otomatis ter-load saat Flink start.

---

## Start Flink Cluster

```bash
# Start Flink cluster (JobManager + TaskManager)
./bin/start-cluster.sh

# Output yang diharapkan:
# Starting cluster.
# Starting standalonesession daemon on host <hostname>.
# Starting taskexecutor daemon on host <hostname>.

# Verifikasi dengan JPS
jps
# Output:
# StandaloneSessionClusterEntrypoint  (JobManager)
# TaskManagerRunner                    (TaskManager)
```

### Stop Flink Cluster

```bash
./bin/stop-cluster.sh
```

---

## Flink Web UI

Akses **Flink Dashboard** di browser:

```
🌐 http://localhost:8081
```

| Menu | Keterangan |
| ---- | ---------- |
| **Overview** | Slot availability, running/completed jobs |
| **Running Jobs** | Detail job graph, task status, throughput |
| **Completed Jobs** | History job yang sudah selesai |
| **Task Managers** | Memory, CPU, slot allocation |
| **Job Manager** | Configuration, logs, stdout |

---

## Flink SQL Client

### Masuk ke SQL Client

```bash
./bin/sql-client.sh
```

Jika berhasil, akan muncul prompt:

```
Flink SQL>
```

### Setting Dasar di SQL Client

```sql
-- Set result mode
SET 'sql-client.execution.result-mode' = 'tableau';

-- Set checkpoint interval
SET 'execution.checkpointing.interval' = '10s';
```

---

## Troubleshooting

### Common Issues

| Problem | Solusi |
| ------- | ------ |
| `Could not find a suitable table factory` | Pastikan connector JAR ada di folder `lib/` lalu restart cluster |
| `ClassNotFoundException` | Download JAR yang sesuai versi Flink 1.18.0 |
| `No suitable driver found for jdbc:mysql` | Pastikan `mysql-connector-j-8.3.0.jar` ada di folder `lib/` |
| `Access denied for user` | Cek username/password dan GRANT privileges di MariaDB |
| `Binlog not enabled` | Aktifkan `log_bin = mysql-bin` dan `binlog_format = ROW` di MariaDB (hanya untuk CDC) |
| `Connection refused :18030` | Pastikan Apache Doris FE sudah running di server `192.168.4.100` |
| `TaskManager lost` | Tambah memory di `flink-conf.yaml` atau kurangi parallelism |
| `Checkpoint timeout` | Increase `execution.checkpointing.timeout` |
| `Backpressure HIGH` | Kurangi throughput source, tambah parallelism, atau optimize query |
| `OutOfMemoryError` | Increase `taskmanager.memory.process.size` |
| `Label already exists` | Ubah `sink.label-prefix` ke nama yang berbeda |

### Restart Flink Cluster

```bash
# Stop cluster
./bin/stop-cluster.sh

# Start cluster
./bin/start-cluster.sh
```

### Cek Log

```bash
# Log JobManager
tail -100f log/flink-*-standalonesession-*.log

# Log TaskManager
tail -100f log/flink-*-taskexecutor-*.log

# Cek error saja
grep -i "error\|exception" log/flink-*-standalonesession-*.log | tail -20
```

### Useful Commands

```bash
# List running jobs
./bin/flink list --running

# List completed jobs
./bin/flink list --all

# Cancel job
./bin/flink cancel <job-id>

# Cek cluster overview
curl http://localhost:8081/overview
```

---

## 📚 Referensi

- [Apache Flink 1.18 Documentation](https://nightlies.apache.org/flink/flink-docs-release-1.18/)
- [Flink JDBC Connector](https://nightlies.apache.org/flink/flink-docs-release-1.18/docs/connectors/table/jdbc/)
- [Flink CDC MySQL Connector](https://nightlies.apache.org/flink/flink-cdc-docs-release-3.0/docs/connectors/flink-sources/mysql-cdc/)
- [Flink Doris Connector](https://doris.apache.org/docs/ecosystem/flink-doris-connector)
- [Apache Flink GitHub](https://github.com/apache/flink)

---

> 📝 **Author**: Data Engineering Team
> 📅 **Last Updated**: 2026-02-11
> 🏷️ **Tags**: `apache-flink`, `setup`, `instalasi`, `standalone`
