# 🌊 Cara Instalasi Apache Airflow

> **Apache Airflow** adalah platform open-source untuk membuat, menjadwalkan, dan memonitoring workflow (DAG - Directed Acyclic Graph). Airflow sangat populer untuk orchestrating data pipelines, ETL jobs, dan automasi tugas-tugas data engineering.

---

## 📋 Table of Contents

- [Prerequisites](#prerequisites)
- [Arsitektur Apache Airflow](#arsitektur-apache-airflow)
- [Instalasi Apache Airflow](#instalasi-apache-airflow)
  - [1. Prasyarat Sistem (System Dependencies)](#1-prasyarat-sistem-system-dependencies)
  - [2. Setup Project & Virtual Environment](#2-setup-project--virtual-environment-️)
  - [3. Install Airflow](#3-install-airflow)
  - [4. Konfigurasi Airflow](#4-konfigurasi-airflow)
  - [5. Menjalankan Airflow](#5-menjalankan-airflow)
  - [6. Akses Web UI](#6-akses-web-ui)
- [Membuat DAG Pertama](#membuat-dag-pertama)
- [Referensi](#referensi)

---

## Prerequisites

| Komponen         | Minimum Requirement          |
|------------------|------------------------------|
| **OS**           | Debian 13     |
| **CPU**          | 4 Cores                     |
| **RAM**          | 8 GB (16+ recommended)      |
| **Disk**         | 50 GB SSD                   |
| **Python**       | 3.8 - 3.13                  |

---

## Arsitektur Apache Airflow

```
┌──────────────────────────────────────────────────────────────────┐
│                     Apache Airflow Architecture                  │
│                                                                  │
│  ┌──────────────┐   ┌──────────────┐   ┌──────────────────────┐  │
│  │  Web Server  │   │  Scheduler   │   │  Executor            │  │
│  │              │   │              │   │                      │  │
│  │ • Flask App  │   │ • Parse DAGs │   │ • LocalExecutor      │  │
│  │ • REST API   │   │ • Schedule   │   │ • CeleryExecutor     │  │
│  │ • UI (:8080) │   │   Tasks      │   │ • KubernetesExecutor │  │
│  │ • Auth/RBAC  │   │ • Trigger    │   │                      │  │
│  └──────┬───────┘   └──────┬───────┘   └──────────┬───────────┘  │
│         │                  │                       │             │
│         ▼                  ▼                       ▼             │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │                   Metadata Database                         │ │
│  │              (PostgreSQL / MySQL / SQLite)                  │ │
│  │                                                             │ │
│  │  • DAG definitions    • Task instances                      │ │
│  │  • Run history        • Variables & Connections             │ │
│  │  • XCom data          • Logs metadata                       │ │
│  └─────────────────────────────────────────────────────────────┘ │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │                     DAGs Directory                          │ │
│  │                    /airflow/dags/                           │ │
│  │                                                             │ │
│  │  📄 dag_etl_daily.py     📄 dag_dbt_run.py                  │ │
│  │  📄 dag_kafka_monitor.py 📄 dag_data_quality.py             │ │
│  └─────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────┘
```

---

## Instalasi Apache Airflow Metode pip Install (Standalone)

Dokumen ini berisi langkah-langkah instalasi Apache Airflow di server Debian/Ubuntu menggunakan Python Virtual Environment. Konfigurasi ini disesuaikan untuk kebutuhan **Data Engineering** (koneksi ke Apache Doris & dbt).

---

## 1. Prasyarat Sistem (System Dependencies)
Sebelum menginstal Python library, pastikan paket sistem operasi berikut terinstall agar tidak terjadi error saat kompilasi (terutama untuk `mysqlclient`).

Jalankan di terminal server:
```bash
# Update repository
sudo apt update

# Install Python, pip, venv, dan library database wajib
# PENTING: 'default-libmysqlclient-dev' dibutuhkan agar Airflow bisa konek ke Doris
sudo apt install -y python3-pip python3-venv build-essential libssl-dev libffi-dev python3-dev default-libmysqlclient-dev

# Cek Versi Python
python3 --version
```
---

## 2. Setup Project & Virtual Environment 
Kita tidak menginstall Airflow di sistem global agar tidak merusak aplikasi lain. Kita pakai Virtual Environment.
```bash
# 1. Buat folder kerja kamu (Ganti 'nama_user' dengan nama kamu)
mkdir ~/airflow_project
cd ~/airflow_project

# 2. Buat lingkungan isolasi Python (Virtual Environment)
python3 -m venv airflow_venv

# 3. Aktifkan lingkungan tersebut
source airflow_venv/bin/activate
```
**Tanda Berhasil**: Di terminal kamu sebelah kiri akan muncul tulisan `(airflow_venv)`.

---

## 3. Install Airflow 
Kita akan menginstall Airflow versi stabil (3.1.7) beserta plugin untuk MySQL (Doris) dan dbt.

```bash
# Set variabel versi
AIRFLOW_VERSION=3.1.7

PYTHON_VERSION="$(python -c 'import sys; print(f"{sys.version_info.major}.{sys.version_info.minor}")')"

CONSTRAINT_URL="https://raw.githubusercontent.com/apache/airflow/constraints-${AIRFLOW_VERSION}/constraints-${PYTHON_VERSION}.txt"

# Install Airflow Core
pip install "apache-airflow==${AIRFLOW_VERSION}" --constraint "${CONSTRAINT_URL}"

# Install Konektor Database & dbt
pip install apache-airflow-providers-mysql dbt-core dbt-doris
```
---


## 4. Konfigurasi Airflow

Jalankan inisialisasi awal untuk membuat file konfigurasi.
```bash
export AIRFLOW_HOME=~/airflow
airflow db init
```
⚠️ **Wajib Setting Timezone (PENTING!)**
Agar data logistik tidak selisih jam (WIB vs UTC), kita harus ubah settingan waktu.

1. Buka file konfigurasi: vim ~/airflow_project/airflow.cfg
2. Tekan / untuk mencari kata: default_timezone
3. Ubah utc menjadi Asia/Jakarta.

```bash
default_timezone = Asia/Jakarta
```
4. Simpan dan Keluar.

---

## 5. Menjalankan Airflow
Sekarang saatnya menyalakan mesin Airflow!

```
# Pastikan kamu masih di folder project dan venv aktif
airflow standalone
```
🛑 **PERHATIAN**: Baca Output Terminal!
Setelah perintah dijalankan, terminal akan menampilkan kotak berisi Username dan Password atau untuk melihat dan merubah password kamu, buka terminal baru dan jalankan perintah ini:
```bash
cd ~/airflow/simple_auth_manager_passwords.json.generated
```
**(Sesuaikan ~/airflow dengan folder instalasi kamu jika berbeda)**

---

## 6. Akses Web UI
Buka browser di laptop kamu dan akses alamat berikut:

**URL: http://localhost:8080**

Masukkan username admin dan password yang tadi dicatat.

---

### Fitur Utama:
- **DAGs**: List, toggle on/off, trigger manual run
- **Graph**: Visualisasi dependency antar task
- **Grid**: Status history per task
- **Calendar**: View run schedule
- **Code**: Lihat source code DAG
- **Admin > Connections**: Manage database connections
- **Admin > Variables**: Manage variables
- **Browse > Task Instances**: Monitor task status

---

## Membuat DAG Pertama
DAG hanyalah sebuah file Python .py yang disimpan di folder dags/. Jika dalam folder airflow tidak ditemukan folder dags, maka kita membuat manual terlebih dahulu dengan melakukan
```bash
cd airflow
mkdir dags
```
Di dalamnya file python tersebut, kamu mendefinisikan:
1. Siapa pemiliknya (owner).
2. Kapan dijalankan (schedule).
3. Apa yang dikerjakan (Tasks/Operators).
4. Bagaimana urutannya (Dependencies).

Berikut gambaran dari file python yang disimpan di folder dags/
```bash
from airflow import DAG
from airflow.operators.bash import BashOperator
from datetime import datetime, timedelta

# 1. KONFIGURASI PATH (LOKASI FILE)
# Lokasi folder project dbt kamu (tempat file dbt_project.yml berada)
PATH_DBT_PROJECT = '/home/sherly/dbt/my_doris_project'

# Lokasi file executable dbt di dalam virtual environment
# Kita pakai full path ini supaya Airflow tidak bingung cari "dbt" di mana
PATH_DBT_EXECUTABLE = '/home/sherly/dbt/venv/bin/dbt'

# 2. DEFAULT ARGUMENTS (PENGATURAN UMUM)
# Pengaturan ini akan diterapkan ke semua task di dalam DAG ini
default_args = {
    'owner': 'sherly',          # Pemilik DAG (muncul di filter UI)
    'depends_on_past': False,   # Jika run sebelumnya gagal, run sekarang tetap jalan
    'email_on_failure': False,  # Tidak kirim email jika gagal (biar inbox ga penuh)
    'email_on_retry': False,    # Tidak kirim email saat mencoba ulang
    'retries': 0,               # PENTING: Jangan retry otomatis. 
                                # Karena jadwalnya tiap menit, kalau gagal biarkan saja,
                                # nanti menit berikutnya akan coba lagi.
}

# 3. DEFINISI DAG (IDENTITAS PIPELINE)
with DAG(
    'wahana_dbt_orders_update', # ID unik DAG (tidak boleh ada spasi)
    default_args=default_args,
    description='update table doris via dbt setiap menit',
    
    # CRON EXPRESSION: '* * * * *' artinya jalan SETIAP MENIT
    schedule='* * * * *',       
    
    # Tanggal mulai efektif DAG ini. 
    # Karena catchup=False, tanggal lampau ini tidak akan memicu job lama.
    start_date=datetime(2026, 1, 1), 
    
    # PENTING: False artinya "Jangan jalankan job yang terlewat dari 1 Jan sampai sekarang".
    # Airflow hanya akan menjalankan job mulai dari detik ini ke depan.
    catchup=False,              
    
    # SANGAT PENTING: Hanya boleh ada 1 proses yang jalan dalam satu waktu.
    # Mencegah penumpukan jika proses dbt ternyata butuh waktu > 1 menit.
    max_active_runs=1,          
    
    tags=['wahana', 'dbt'],     # Label untuk memudahkan pencarian di UI
) as dag:

    # 4. DEFINISI TASK (PEKERJAAN)
    dbt_run = BashOperator(
        task_id='dbt_run_incremental', # Nama task di kotak hijau UI
        
        # Perintah Terminal yang akan dijalankan:
        # 1. Pindah ke folder project (cd ...)
        # 2. Jalankan dbt run khusus model 'orders' saja (--select orders)
        bash_command=f'cd {PATH_DBT_PROJECT} && {PATH_DBT_EXECUTABLE} run --select orders',
    )


    # 5. EKSEKUSI / DEPENDENSI
    # Karena cuma ada 1 task, kita cukup panggil variabelnya saja.
    dbt_run

```
---
## Referensi

- [Apache Airflow Official Documentation](https://airflow.apache.org/docs/apache-airflow/stable/start.html)

---


