# 📊 Cara Instalasi & Setup Metabase

> **Metabase** adalah platform Business Intelligence (BI) open-source yang memungkinkan pengguna untuk membuat dashboard, melakukan eksplorasi data, dan membuat laporan visual tanpa perlu menguasai SQL secara mendalam. Sangat cocok digunakan bersama Apache Doris sebagai sumber data analitik.

---

## 📋 Table of Contents

- [Prerequisites](#prerequisites)
- [Arsitektur Metabase](#arsitektur-metabase)
- [Instalasi Metabase](#instalasi-metabase)
  - [1. Prasyarat Sistem (System Dependencies)](#1-prasyarat-sistem-system-dependencies)
  - [2. Download Metabase JAR](#2-download-metabase-jar)
  - [3. Konfigurasi Database Metadata (PostgreSQL)](#3-konfigurasi-database-metadata-postgresql)
  - [4. Konfigurasi Environment Variables](#4-konfigurasi-environment-variables)
  - [5. Menjalankan Metabase](#5-menjalankan-metabase)
  - [6. Setup sebagai Systemd Service](#6-setup-sebagai-systemd-service)
- [Koneksi ke Apache Doris](#koneksi-ke-apache-doris)
- [Verifikasi Instalasi](#verifikasi-instalasi)
- [Troubleshooting](#troubleshooting)
- [Referensi](#referensi)

---

## Prerequisites

| Komponen       | Minimum Requirement             |
|----------------|---------------------------------|
| **OS**         | Debian 13 / Ubuntu 22.04        |
| **CPU**        | 2 Cores (4+ recommended)        |
| **RAM**        | 4 GB (8+ recommended)           |
| **Disk**       | 20 GB SSD                       |
| **Java**       | JDK 17 atau JDK 21              |
| **PostgreSQL** | 14+ (untuk metadata Metabase)   |

---

## Arsitektur Metabase

```
┌──────────────────────────────────────────────────────────────┐
│                       Metabase Architecture                  │
│                                                              │
│  ┌─────────────┐     ┌──────────────────┐                   │
│  │   Browser   │────▶│  Metabase Server │                   │
│  │ (Dashboard) │     │  (Java/JAR App)  │                   │
│  └─────────────┘     │  Port: 3000      │                   │
│                      └────────┬─────────┘                   │
│                               │                             │
│              ┌────────────────┼─────────────────┐           │
│              ▼                ▼                 ▼           │
│   ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐  │
│   │  PostgreSQL  │  │ Apache Doris │  │ Sumber Data Lain │  │
│   │  (Metadata)  │  │ (Data Source)│  │ (MySQL, PG, dst) │  │
│   └──────────────┘  └──────────────┘  └──────────────────┘  │
└──────────────────────────────────────────────────────────────┘
```

---

## Instalasi Metabase

---

## 1. Prasyarat Sistem (System Dependencies)

```bash
# Update repository
sudo apt update && sudo apt upgrade -y

# Install Java 21 (JDK)
sudo apt install -y openjdk-21-jdk

# Verifikasi Java
java -version
# Output: openjdk version "21.x.x"

# Install PostgreSQL (untuk database metadata Metabase)
sudo apt install -y postgresql postgresql-contrib

# Verifikasi PostgreSQL
psql --version
```

---

## 2. Download Metabase JAR

```bash
# Buat folder untuk Metabase
sudo mkdir -p /opt/metabase
cd /opt/metabase

# Download Metabase versi terbaru (cek versi terbaru di https://www.metabase.com/downloads)
sudo wget https://downloads.metabase.com/v0.53.4/metabase.jar

# Verifikasi file berhasil diunduh
ls -lh metabase.jar
```

---

## 3. Konfigurasi Database Metadata (PostgreSQL)

Metabase membutuhkan database untuk menyimpan konfigurasi, dashboard, dan user. Kita gunakan PostgreSQL agar lebih stabil dibanding H2 (default).

```bash
# Masuk ke PostgreSQL
sudo -u postgres psql

# Buat user dan database untuk Metabase
CREATE USER metabase_user WITH PASSWORD 'password_aman_kamu';
CREATE DATABASE metabase_db OWNER metabase_user;
GRANT ALL PRIVILEGES ON DATABASE metabase_db TO metabase_user;

# Keluar dari PostgreSQL
\q
```

---

## 4. Konfigurasi Environment Variables

Buat file konfigurasi environment untuk Metabase:

```bash
# Buat file environment
sudo vim /opt/metabase/metabase.env
```

Isikan variabel berikut:

```properties
# ============================================
# Metabase Environment Configuration
# ============================================

# Konfigurasi Database Metadata (PostgreSQL)
MB_DB_TYPE=postgres
MB_DB_DBNAME=metabase_db
MB_DB_PORT=5432
MB_DB_USER=metabase_user
MB_DB_PASS=password_aman_kamu
MB_DB_HOST=localhost

# Port default Metabase
MB_JETTY_PORT=3000

# Timezone
JAVA_TIMEZONE=Asia/Jakarta

# Site URL (sesuaikan dengan IP atau domain server)
MB_SITE_URL=http://localhost:3000
```

---

## 5. Menjalankan Metabase

```bash
# Jalankan Metabase secara manual (untuk testing)
cd /opt/metabase
java -jar metabase.jar
```

> ⚠️ **PERHATIAN**: Tunggu beberapa menit hingga muncul log `Metabase Initialization COMPLETE`. Proses pertama kali akan membutuhkan waktu lebih lama karena inisialisasi database.

Jika berhasil, buka browser dan akses:

**URL: http://localhost:3000**

---

## 6. Setup sebagai Systemd Service

Agar Metabase otomatis berjalan saat server restart, kita daftarkan sebagai systemd service.

```bash
# Buat user khusus untuk Metabase (best practice keamanan)
sudo useradd -r -s /bin/false metabase

# Set kepemilikan folder
sudo chown -R metabase:metabase /opt/metabase

# Buat file service
sudo vim /etc/systemd/system/metabase.service
```

Isikan konfigurasi service berikut:

```ini
[Unit]
Description=Metabase BI Server
After=network.target postgresql.service

[Service]
User=metabase
Group=metabase
WorkingDirectory=/opt/metabase
EnvironmentFile=/opt/metabase/metabase.env
ExecStart=/usr/bin/java -jar /opt/metabase/metabase.jar
Restart=always
RestartSec=5
StandardOutput=journal
StandardError=journal
SyslogIdentifier=metabase

[Install]
WantedBy=multi-user.target
```

Aktifkan dan jalankan service:

```bash
# Reload systemd
sudo systemctl daemon-reload

# Enable agar otomatis start saat boot
sudo systemctl enable metabase

# Start service
sudo systemctl start metabase

# Cek status
sudo systemctl status metabase
```

---

## Koneksi ke Apache Doris

Setelah Metabase berjalan, hubungkan ke Apache Doris sebagai sumber data.

### Langkah Koneksi:

1. Buka Metabase di browser: `http://localhost:3000`
2. Login dengan akun admin
3. Klik ikon **⚙️ Settings** (pojok kanan atas) → **Admin settings**
4. Pilih menu **Databases** → **Add database**
5. Pilih tipe database: **MySQL** *(Apache Doris kompatibel dengan protokol MySQL)*

Isi form koneksi:

| Field              | Nilai                          |
|--------------------|--------------------------------|
| **Display name**   | Apache Doris                   |
| **Host**           | `127.0.0.1` atau IP server FE  |
| **Port**           | `9030`                         |
| **Database name**  | nama_database_kamu             |
| **Username**       | `root`                         |
| **Password**       | *(kosong jika belum diset)*    |

6. Klik **Save** → Metabase akan melakukan test koneksi otomatis.

---

## Verifikasi Instalasi

```bash
# Cek status service
sudo systemctl status metabase

# Lihat log secara real-time
sudo journalctl -u metabase -f

# Cek apakah port 3000 aktif
ss -tlnp | grep 3000
```

---

## Troubleshooting

### Common Issues

| Problem                               | Solusi                                                                                          |
|---------------------------------------|-------------------------------------------------------------------------------------------------|
| `Port 3000 already in use`            | Ganti port di `metabase.env`: `MB_JETTY_PORT=3001`                                             |
| `Cannot connect to database`          | Pastikan PostgreSQL berjalan: `sudo systemctl status postgresql`                                |
| `Java heap space / OutOfMemory`       | Tambahkan opsi JVM: `ExecStart=/usr/bin/java -Xmx2g -jar /opt/metabase/metabase.jar`           |
| `Metabase not starting after restart` | Cek log: `sudo journalctl -u metabase -n 100 --no-pager`                                       |
| `Cannot connect to Doris`             | Pastikan FE Doris berjalan dan port `9030` tidak diblokir firewall                              |

### Useful Commands

```bash
# Restart Metabase
sudo systemctl restart metabase

# Stop Metabase
sudo systemctl stop metabase

# Lihat log
sudo journalctl -u metabase -n 100 --no-pager

# Cek versi Java yang digunakan
java -version
```

---

## 📚 Referensi

- [Metabase Official Documentation](https://www.metabase.com/docs/latest/)
- [Metabase GitHub](https://github.com/metabase/metabase)
- [Metabase Download Page](https://www.metabase.com/downloads)

---
