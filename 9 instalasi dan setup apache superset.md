# 🚀 Cara Instalasi & Setup Apache Superset

> **Apache Superset** adalah platform Business Intelligence (BI) dan eksplorasi data modern berbasis web yang bersifat open-source. Superset mendukung koneksi ke berbagai database, termasuk Apache Doris, dan memungkinkan pembuatan visualisasi, chart, dan dashboard yang interaktif tanpa batasan.

---

## 📋 Table of Contents

- [Prerequisites](#prerequisites)
- [Arsitektur Apache Superset](#arsitektur-apache-superset)
- [Instalasi Apache Superset](#instalasi-apache-superset)
  - [1. Prasyarat Sistem (System Dependencies)](#1-prasyarat-sistem-system-dependencies)
  - [2. Setup Virtual Environment](#2-setup-virtual-environment)
  - [3. Install Apache Superset](#3-install-apache-superset)
  - [4. Konfigurasi Superset](#4-konfigurasi-superset)
  - [5. Inisialisasi Database & Admin](#5-inisialisasi-database--admin)
  - [6. Menjalankan Superset](#6-menjalankan-superset)
  - [7. Setup sebagai Systemd Service](#7-setup-sebagai-systemd-service)
- [Koneksi ke Apache Doris](#koneksi-ke-apache-doris)
- [Verifikasi Instalasi](#verifikasi-instalasi)
- [Troubleshooting](#troubleshooting)
- [Referensi](#referensi)

---

## Prerequisites

| Komponen       | Minimum Requirement              |
|----------------|----------------------------------|
| **OS**         | Debian 13 / Ubuntu 22.04         |
| **CPU**        | 4 Cores (8+ recommended)         |
| **RAM**        | 8 GB (16+ recommended)           |
| **Disk**       | 30 GB SSD                        |
| **Python**     | 3.9 - 3.11                       |
| **PostgreSQL** | 14+ (untuk metadata Superset)    |
| **Redis**      | 6+ (untuk caching & Celery)      |

---

## Arsitektur Apache Superset

```
┌─────────────────────────────────────────────────────────────────┐
│                    Apache Superset Architecture                 │
│                                                                 │
│  ┌─────────────┐   ┌──────────────────────────────────────────┐ │
│  │   Browser   │──▶│           Superset Web Server            │ │
│  │ (Dashboard) │   │       (Flask/Gunicorn - Port: 8088)      │ │
│  └─────────────┘   └───────────────┬──────────────────────────┘ │
│                                    │                            │
│              ┌─────────────────────┼──────────────┐            │
│              ▼                     ▼              ▼             │
│   ┌──────────────┐    ┌─────────────────┐  ┌────────────────┐  │
│   │  PostgreSQL  │    │      Redis      │  │ Celery Worker  │  │
│   │  (Metadata)  │    │   (Cache/Queue) │  │ (Async Query)  │  │
│   └──────────────┘    └─────────────────┘  └────────────────┘  │
│                                    │                            │
│                    ┌───────────────┼───────────────┐           │
│                    ▼               ▼               ▼           │
│         ┌──────────────┐  ┌─────────────┐  ┌────────────────┐  │
│         │ Apache Doris │  │    MySQL    │  │  Sumber Lainnya│  │
│         │ (Data Source)│  │  (MariaDB)  │  │                │  │
│         └──────────────┘  └─────────────┘  └────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Instalasi Apache Superset

---

## 1. Prasyarat Sistem (System Dependencies)

```bash
# Update repository
sudo apt update && sudo apt upgrade -y

# Install Python dan dependency sistem
sudo apt install -y python3-pip python3-venv python3-dev build-essential \
    libssl-dev libffi-dev libsasl2-dev libldap2-dev default-libmysqlclient-dev \
    libpq-dev pkg-config

# Verifikasi Python
python3 --version
# Output: Python 3.11.x

# Install PostgreSQL (untuk metadata Superset)
sudo apt install -y postgresql postgresql-contrib

# Install Redis (untuk caching)
sudo apt install -y redis-server

# Aktifkan Redis
sudo systemctl enable redis-server
sudo systemctl start redis-server

# Verifikasi Redis
redis-cli ping
# Output: PONG
```

---

## 2. Setup Virtual Environment

Kita isolasi Superset dalam Virtual Environment agar tidak mengganggu sistem Python global.

```bash
# Buat folder untuk Superset
mkdir ~/superset_project
cd ~/superset_project

# Buat Virtual Environment
python3 -m venv superset_venv

# Aktifkan Virtual Environment
source superset_venv/bin/activate
```

**Tanda Berhasil**: Di terminal kamu sebelah kiri akan muncul tulisan `(superset_venv)`.

```bash
# Update pip
pip install --upgrade pip setuptools wheel
```

---

## 3. Install Apache Superset

```bash
# Install Apache Superset
pip install apache-superset

# Install driver koneksi database yang dibutuhkan
# Driver MySQL (untuk koneksi ke Apache Doris)
pip install mysqlclient

# Driver PostgreSQL (untuk metadata Superset)
pip install psycopg2-binary

# Driver tambahan (opsional, sesuai kebutuhan)
# pip install sqlalchemy-doris  # Driver khusus Doris (alternatif)
```

---

## 4. Konfigurasi Superset

Buat file konfigurasi utama Superset:

```bash
# Buat file konfigurasi
vim ~/superset_project/superset_config.py
```

Isikan konfigurasi berikut:

```python
# ============================================
# Apache Superset Configuration
# ============================================

import os

# Secret key - WAJIB diganti dengan string acak yang panjang
SECRET_KEY = 'ganti_dengan_secret_key_yang_sangat_panjang_dan_acak_minimal_42_karakter'

# Konfigurasi Database Metadata (PostgreSQL)
SQLALCHEMY_DATABASE_URI = 'postgresql+psycopg2://superset_user:password_aman@localhost/superset_db'

# Konfigurasi Redis (Cache)
CACHE_CONFIG = {
    'CACHE_TYPE': 'RedisCache',
    'CACHE_DEFAULT_TIMEOUT': 300,
    'CACHE_KEY_PREFIX': 'superset_',
    'CACHE_REDIS_URL': 'redis://localhost:6379/0',
}

# Konfigurasi Celery (Async Query)
class CeleryConfig:
    broker_url = 'redis://localhost:6379/0'
    imports = ('superset.sql_lab',)
    result_backend = 'redis://localhost:6379/0'
    worker_prefetch_multiplier = 1
    task_acks_late = False

CELERY_CONFIG = CeleryConfig

# Timezone
BABEL_DEFAULT_LOCALE = 'en'
SUPERSET_WEBSERVER_TIMEOUT = 300

# Feature flags (aktifkan fitur-fitur tambahan)
FEATURE_FLAGS = {
    'ENABLE_TEMPLATE_PROCESSING': True,
    'DASHBOARD_NATIVE_FILTERS': True,
    'GLOBAL_ASYNC_QUERIES': True,
}
```

Set environment variable agar Superset tahu lokasi konfigurasi:

```bash
echo 'export SUPERSET_CONFIG_PATH=~/superset_project/superset_config.py' >> ~/.bashrc
source ~/.bashrc
```

---

## 5. Inisialisasi Database & Admin

Setup database metadata untuk Superset menggunakan PostgreSQL:

```bash
# Masuk ke PostgreSQL
sudo -u postgres psql

# Buat user dan database untuk Superset
CREATE USER superset_user WITH PASSWORD 'password_aman';
CREATE DATABASE superset_db OWNER superset_user;
GRANT ALL PRIVILEGES ON DATABASE superset_db TO superset_user;

# Keluar
\q
```

Inisialisasi database Superset:

```bash
# Pastikan venv aktif
source ~/superset_project/superset_venv/bin/activate

# Set environment variable
export SUPERSET_CONFIG_PATH=~/superset_project/superset_config.py
export FLASK_APP=superset

# Inisialisasi database (buat tabel-tabel metadata)
superset db upgrade

# Muat data awal (contoh chart dan dashboard)
superset load_examples

# Inisialisasi role dan permission
superset init

# Buat user admin
superset fab create-admin \
    --username admin \
    --firstname Admin \
    --lastname Superset \
    --email admin@example.com \
    --password admin_password_aman
```

---

## 6. Menjalankan Superset

```bash
# Pastikan venv aktif dan env variable sudah di-set
source ~/superset_project/superset_venv/bin/activate
export SUPERSET_CONFIG_PATH=~/superset_project/superset_config.py

# Jalankan Superset (development mode)
superset run -p 8088 --with-threads --reload --debugger
```

> ⚠️ **PERHATIAN**: Mode di atas hanya untuk testing. Untuk production, gunakan **Gunicorn** seperti langkah di bawah.

```bash
# Jalankan dengan Gunicorn (production mode)
gunicorn \
    --bind 0.0.0.0:8088 \
    --workers 4 \
    --worker-class gevent \
    --timeout 120 \
    --limit-request-line 0 \
    --limit-request-field_size 0 \
    "superset.app:create_app()"
```

Buka browser dan akses:

**URL: http://localhost:8088**

---

## 7. Setup sebagai Systemd Service

Agar Superset otomatis berjalan saat server restart.

```bash
# Buat file service untuk Superset Web
sudo vim /etc/systemd/system/superset.service
```

Isikan konfigurasi berikut:

```ini
[Unit]
Description=Apache Superset BI Server
After=network.target postgresql.service redis.service

[Service]
User=wahana-express
Group=wahana-express
WorkingDirectory=/home/wahana-express/superset_project
Environment="SUPERSET_CONFIG_PATH=/home/wahana-express/superset_project/superset_config.py"
ExecStart=/home/wahana-express/superset_project/superset_venv/bin/gunicorn \
    --bind 0.0.0.0:8088 \
    --workers 4 \
    --worker-class gevent \
    --timeout 120 \
    --limit-request-line 0 \
    --limit-request-field_size 0 \
    "superset.app:create_app()"
Restart=always
RestartSec=5
StandardOutput=journal
StandardError=journal
SyslogIdentifier=superset

[Install]
WantedBy=multi-user.target
```

Aktifkan dan jalankan service:

```bash
# Reload systemd
sudo systemctl daemon-reload

# Enable agar otomatis start saat boot
sudo systemctl enable superset

# Start service
sudo systemctl start superset

# Cek status
sudo systemctl status superset
```

---

## Koneksi ke Apache Doris

Setelah Superset berjalan, hubungkan ke Apache Doris sebagai sumber data.

### Langkah Koneksi:

1. Buka Superset di browser: `http://localhost:8088`
2. Login dengan akun admin yang sudah dibuat
3. Klik menu **Settings** (pojok kanan atas) → **Database Connections**
4. Klik tombol **+ Database** (tambah database baru)
5. Pilih tipe database: **MySQL** *(Apache Doris kompatibel dengan protokol MySQL)*

Isi form koneksi dengan SQLAlchemy URI berikut:

```
mysql+mysqlconnector://root:@127.0.0.1:9030/nama_database_kamu
```

Atau isi secara terpisah melalui form:

| Field              | Nilai                          |
|--------------------|--------------------------------|
| **Database name**  | Apache Doris                   |
| **Host**           | `127.0.0.1` atau IP server FE  |
| **Port**           | `9030`                         |
| **Database**       | nama_database_kamu             |
| **Username**       | `root`                         |
| **Password**       | *(kosong jika belum diset)*    |

6. Klik **Test Connection** untuk memastikan koneksi berhasil.
7. Klik **Connect** untuk menyimpan konfigurasi.

---

## Verifikasi Instalasi

```bash
# Cek status Superset service
sudo systemctl status superset

# Lihat log secara real-time
sudo journalctl -u superset -f

# Cek apakah port 8088 aktif
ss -tlnp | grep 8088

# Cek Redis berjalan
redis-cli ping

# Cek PostgreSQL berjalan
sudo systemctl status postgresql
```

---

## Troubleshooting

### Common Issues

| Problem                                    | Solusi                                                                                   |
|--------------------------------------------|------------------------------------------------------------------------------------------|
| `Port 8088 already in use`                 | Ganti port di ExecStart: `--bind 0.0.0.0:8089`                                          |
| `SECRET_KEY is not set`                    | Pastikan `SECRET_KEY` diisi di `superset_config.py` (min. 42 karakter)                  |
| `ModuleNotFoundError: No module named ...` | Aktifkan venv: `source ~/superset_project/superset_venv/bin/activate`                   |
| `Cannot connect to Redis`                  | Pastikan Redis berjalan: `sudo systemctl status redis-server`                            |
| `Cannot connect to PostgreSQL`             | Cek koneksi: `psql -U superset_user -d superset_db -h localhost`                        |
| `Cannot connect to Doris`                  | Pastikan FE Doris berjalan dan port `9030` tidak diblokir firewall                       |
| `superset db upgrade fails`                | Pastikan `SUPERSET_CONFIG_PATH` sudah di-set dan PostgreSQL sudah berjalan               |

### Useful Commands

```bash
# Restart Superset
sudo systemctl restart superset

# Stop Superset
sudo systemctl stop superset

# Lihat log Superset
sudo journalctl -u superset -n 100 --no-pager

# Reset password admin (jika lupa)
source ~/superset_project/superset_venv/bin/activate
export SUPERSET_CONFIG_PATH=~/superset_project/superset_config.py
superset fab reset-password --username admin

# Update Superset ke versi terbaru
pip install --upgrade apache-superset
superset db upgrade
superset init
```

---

## 📚 Referensi

- [Apache Superset Official Documentation](https://superset.apache.org/docs/intro)
- [Apache Superset GitHub](https://github.com/apache/superset)
- [Apache Superset Installation Guide](https://superset.apache.org/docs/installation/installing-superset-from-scratch)

---
