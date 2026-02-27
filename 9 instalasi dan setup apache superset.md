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
- [Koneksi ke Apache Doris](#koneksi-ke-apache-doris)
- [Troubleshooting](#troubleshooting)
- [Referensi](#referensi)

---

## Prerequisites

| Komponen   | Minimum Requirement      |
|------------|--------------------------|
| **OS**     | Debian 13 / Ubuntu 22.04 |
| **CPU**    | 4 Cores (8+ recommended) |
| **RAM**    | 8 GB (16+ recommended)   |
| **Disk**   | 30 GB SSD                |
| **Python** | 3.9 - 3.11               |

---

## Arsitektur Apache Superset

```
┌─────────────────────────────────────────────────────────────────┐
│                    Apache Superset Architecture                 │
│                                                                 │
│  ┌─────────────┐   ┌──────────────────────────────────────────┐ │
│  │   Browser   │──▶│           Superset Web Server            │ │
│  │ (Dashboard) │   │         (Flask - Port: 8088)             │ │
│  └─────────────┘   └───────────────┬──────────────────────────┘ │
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
sudo apt-get update

# Install Python dan dependency sistem
sudo apt install -y python3-pip
sudo apt-get install build-essential libssl-dev libffi-dev python3-dev python3-pip libsasl2-dev libldap2-dev default-libmysqlclient-dev

# Verifikasi Python
python3 --version
# Output: Python 3.11.x

# Install PostgreSQL (untuk metadata Superset)
sudo apt install -y postgresql postgresql-contrib
```

---

## 2. Setup Virtual Environment

Kita isolasi Superset dalam Virtual Environment agar tidak mengganggu sistem Python global.

```bash
# Buat folder untuk Superset
mkdir ~/superset_project
cd ~/superset_project

# Buat Virtual Environment
python3 -m venv venv

# Aktifkan Virtual Environment
source venv/bin/activate
```

**Tanda Berhasil**: Di terminal kamu sebelah kiri akan muncul tulisan `(venv)`.

```bash
# Update pip
pip install --upgrade pip setuptools wheel
```

---

## 3. Install Apache Superset

```bash
# Install Apache Superset
pip install apache-superset

# Install driver koneksi ke Apache Doris (via protokol MySQL)
pip install mysqlclient
```

---

## 4. Konfigurasi Superset

Set environment variable yang dibutuhkan Superset sebelum inisialisasi:

```bash
# Secret key - WAJIB diisi (gunakan perintah openssl di bawah untuk generate)
export SUPERSET_SECRET_KEY="doris_superset_kunci_rahasia_lokal_123_abc!"
export FLASK_APP=superset

# Cara generate SUPERSET_SECRET_KEY yang aman
openssl rand -base64 42
```

---

## 5. Inisialisasi Database & Admin

```bash
# Inisialisasi database (buat tabel-tabel metadata)
superset db upgrade

# Buat user admin
superset fab create-admin

# Inisialisasi role dan permission
superset init
```

---

## 6. Menjalankan Superset

```bash
# Pastikan venv aktif
source ~/superset_project/venv/bin/activate

# Jalankan Superset
superset run -h 0.0.0.0 -p 8088 --with-threads --reload --debugger
```

Buka browser dan akses:

**URL: http://localhost:8088**

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

| Field             | Nilai                         |
|-------------------|-------------------------------|
| **Database name** | Apache Doris                  |
| **Host**          | `127.0.0.1` atau IP server FE |
| **Port**          | `9030`                        |
| **Database**      | nama_database_kamu            |
| **Username**      | `root`                        |
| **Password**      | *(kosong jika belum diset)*   |

6. Klik **Test Connection** untuk memastikan koneksi berhasil.
7. Klik **Connect** untuk menyimpan konfigurasi.

---

## Troubleshooting

### Common Issues

| Problem                                    | Solusi                                                                  |
|--------------------------------------------|-------------------------------------------------------------------------|
| `Port 8088 already in use`                 | Ganti port: `superset run -p 8089 ...`                                  |
| `SECRET_KEY is not set`                    | Jalankan `export SUPERSET_SECRET_KEY="..."` sebelum perintah superset   |
| `ModuleNotFoundError: No module named ...` | Aktifkan venv: `source ~/superset_project/venv/bin/activate`            |
| `Cannot connect to Doris`                  | Pastikan FE Doris berjalan dan port `9030` tidak diblokir firewall      |
| `superset db upgrade fails`                | Pastikan `SUPERSET_SECRET_KEY` dan `FLASK_APP` sudah di-set             |

### Useful Commands

```bash
# Cek apakah port 8088 aktif
ss -tlnp | grep 8088

# Reset password admin (jika lupa)
source ~/superset_project/venv/bin/activate
superset fab reset-password --username admin

# Update Superset ke versi terbaru
pip install --upgrade apache-superset
superset db upgrade
superset init
```

---

## Referensi

- [Apache Superset Official Documentation](https://superset.apache.org/docs/intro)
- [Apache Superset GitHub](https://github.com/apache/superset)
---
