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
  - [4. Membuat File Konfigurasi superset_config.py](#4-membuat-file-konfigurasi-superset_configpy)
  - [5. Set Environment Variable Superset](#5-set-environment-variable-superset)
  - [6. Inisialisasi Database & Admin](#6-inisialisasi-database--admin)
  - [7. Menjalankan Superset](#7-menjalankan-superset)
- [Koneksi ke Apache Doris (Langkah Key / Utama)](#koneksi-ke-apache-doris-langkah-key--utama)
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

## 4. Membuat File Konfigurasi `superset_config.py`

Sebelum menjalankan Superset, kita perlu membuat file konfigurasi Python (`superset_config.py`) yang berisi pengaturan-pengaturan penting. File ini akan dibaca otomatis oleh Superset melalui environment variable `SUPERSET_CONFIG_PATH`.

### Buat File dengan Vim

```bash
# Pastikan kamu berada di folder project Superset
cd ~/superset_project

# Buat dan edit file superset_config.py menggunakan vim
vim superset_config.py
```

### Isi File `superset_config.py`

Setelah vim terbuka, tekan `i` untuk masuk ke **INSERT mode**, lalu salin (paste) isi konfigurasi berikut:

```python
SECRET_KEY = "GANTI_DENGAN_HASIL_OPENSSL"
```

Untuk mencari `SECRET_KEY`, generate menggunakan perintah berikut di terminal (bukan di dalam vim):

```bash
openssl rand -base64 42
```

> 💡 Salin hasil output dari perintah di atas, lalu ganti `"GANTI_DENGAN_HASIL_OPENSSL"` di file `superset_config.py` dengan hasil tersebut.

### Simpan dan Keluar dari Vim

Setelah selesai menempel konfigurasi di atas:

1. Tekan **`Esc`** untuk keluar dari INSERT mode
2. Ketik **`:wq`** lalu tekan **Enter** untuk menyimpan dan keluar

```
Esc → :wq → Enter
```

---

## 5. Set Environment Variable Superset

Setelah file `superset_config.py` berhasil dibuat, set environment variable agar Superset membaca file tersebut:

```bash
# Arahkan Superset untuk membaca file konfigurasi superset_config.py
export SUPERSET_CONFIG_PATH=$(pwd)/superset_config.py

# Set Flask app
export FLASK_APP=superset
```

> 💡 **Penjelasan Langkah Kunci (Key Step):**
> - `export SUPERSET_CONFIG_PATH=$(pwd)/superset_config.py` → Memberi tahu Superset lokasi file konfigurasi yang baru saja kamu buat via `vim`. `SECRET_KEY` di dalamnya akan terbaca otomatis.
> - `export FLASK_APP=superset` → Memberitahu Flask bahwa aplikasi yang dijalankan adalah Superset.

---

## 6. Inisialisasi Database & Admin

```bash
# Inisialisasi database (buat tabel-tabel metadata)
superset db upgrade

# Buat user admin
superset fab create-admin

# Inisialisasi role dan permission
superset init
```

---

## 7. Menjalankan Superset

```bash
# Pastikan venv aktif
source ~/superset_project/venv/bin/activate

# Jalankan Superset
superset run -h 0.0.0.0 -p 8088 --with-threads --reload --debugger
```

Buka browser dan akses:

**URL: http://localhost:8088**

---

## Koneksi ke Apache Doris (Langkah Key / Utama)

Setelah Superset berjalan dan file konfigurasi `superset_config.py` berhasil dimuat (berkat `SUPERSET_CONFIG_PATH`), langkah selanjutnya adalah menghubungkan Superset ke Apache Doris. 

Berikut ini adalah langkah-langkah *key* (kunci) agar proses koneksi menjadi lebih mudah dimengerti:

### Langkah Koneksi:

1. Buka Superset di browser melalui URL: **`http://localhost:8088`**
2. Login menggunakan akun admin yang telah dibuat saat proses inisialisasi.
3. Akses menu **Settings** (ikon roda gigi di pojok kanan atas) → pilih **Database Connections**.
4. Klik tombol **+ Database** untuk mendaftarkan sumber data baru.
5. **Key Step:** Pilih tipe database: **MySQL**.
   *(Catatan: Apache Doris berinteraksi dan kompatibel penuh menggunakan protokol MySQL, jadi kita menggunakan driver MySQL untuk koneksinya).*

Langkah paling praktis adalah memasukkan URI koneksi secara langsung. Salin **SQLAlchemy URI** berikut ini:

```text
mysql+mysqlconnector://root:@127.0.0.1:9030/nama_database_kamu
```
*(Jangan lupa ganti `nama_database_kamu` dengan database sebenarnya yang ada di Apache Doris).*

Alternatif jika ingin mengisinya satu per satu melalui form (Basic):

| Field             | Nilai yang Harus Diisi                        |
|-------------------|-----------------------------------------------|
| **Database name** | Apache Doris                                  |
| **Host**          | `127.0.0.1` (atau IP server FE Apache Doris)  |
| **Port**          | `9030` (Port default FE Doris)                |
| **Database**      | *nama_database_kamu*                          |
| **Username**      | `root` (atau username Doris kamu)             |
| **Password**      | *(kosongkan jika tidak memakai password)*     |

6. **Key Step (Validasi):** Selalu klik tombol **Test Connection** terlebih dahulu. Jika muncul popup berhasil (sukses), artinya integrasi Superset dengan Doris telah bekerja.
7. Terakhir, klik **Connect** untuk menyimpan konfigurasi sumber data tersebut secara permanen.

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
