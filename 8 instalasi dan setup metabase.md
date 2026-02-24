# 📊 Cara Instalasi & Setup Metabase

> **Metabase** adalah platform Business Intelligence (BI) open-source yang memungkinkan pengguna untuk membuat dashboard, melakukan eksplorasi data, dan membuat laporan visual tanpa perlu menguasai SQL secara mendalam. Sangat cocok digunakan bersama Apache Doris sebagai sumber data analitik.

---

## 📋 Table of Contents

- [Prerequisites](#prerequisites)
- [Arsitektur Metabase](#arsitektur-metabase)
- [Instalasi Metabase](#instalasi-metabase)
  - [1. Prasyarat Sistem (System Dependencies)](#1-prasyarat-sistem-system-dependencies)
  - [2. Download Metabase JAR](#2-download-metabase-jar)
  - [3. Menjalankan Metabase](#3-menjalankan-metabase)
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

# Install Java (JDK 17)
sudo apt install openjdk-17-jdk

# Verify Java
java -version
# Output: openjdk version "17.x.x"
```
---

## 2. Download Metabase JAR

```bash
# Buat folder untuk Metabase
mkdir metabase
cd metabase

# Download Metabase versi 4 untuk jdk 17 dan versi 5 untuk jdk 21 (cek versi terbaru di https://www.metabase.com/downloads)
wget https://downloads.metabase.com/v0.49.13/metabase.jar

# Verifikasi file berhasil diunduh
ls -lh metabase.jar
```

---

## 3. Menjalankan Metabase

```bash
# Jalankan Metabase secara manual (untuk testing)
cd metabase
java -jar metabase.jar
```

> ⚠️ **PERHATIAN**: Tunggu beberapa menit hingga muncul log `Metabase Initialization COMPLETE`. Proses pertama kali akan membutuhkan waktu lebih lama karena inisialisasi database.

Jika berhasil, buka browser dan akses:

**URL: http://localhost:3000**


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
| `Cannot connect to database`          | Pastikan Doris berjalan: `jps`                                |
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

## Referensi

- [Metabase Official Documentation](https://www.metabase.com/docs/latest/)
- [Metabase GitHub](https://github.com/metabase/metabase)
- [Metabase Download Page](https://www.metabase.com/docs/latest/installation-and-operation/running-the-metabase-jar-file)
- [Tutorial Menggunakan Metabase](https://medium.com/javanlabs/visualisasi-data-menggunakan-metabase-7a2b739e6be5)

---
