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
# (Opsional) Set port Metabase jika port default 3000 sudah digunakan service lain
# Gunakan perintah ini HANYA jika address yang sama sudah memakai port 3000
export MB_JETTY_PORT=8080

# Jalankan Metabase secara manual (untuk testing)
cd metabase
java -jar metabase.jar
```

> 💡 **Kapan perlu `export MB_JETTY_PORT`?**
> Perintah ini **opsional** — hanya diperlukan jika kamu menjalankan Metabase di address yang sama dengan service lain yang sudah memakai port `3000` (misalnya Node.js, Grafana, dsb.). Jika port `3000` masih kosong, kamu bisa langsung menjalankan `java -jar metabase.jar` tanpa export port.

> ⚠️ **PERHATIAN**: Tunggu beberapa menit hingga muncul log `Metabase Initialization COMPLETE`. Proses pertama kali akan membutuhkan waktu lebih lama karena inisialisasi database.

Jika berhasil, buka browser dan akses:

- **Default (tanpa export port):** `http://localhost:3000`
- **Jika menggunakan `MB_JETTY_PORT=8080`:** `http://localhost:8080`

### ⚠️ Penting: Update URL Website di Pengaturan Admin

Jika kamu mengubah port Metabase (misalnya ke `8080`), **wajib** sesuaikan juga **URL Website** di pengaturan admin Metabase:

1. Login ke Metabase
2. Klik ikon **⚙️ Settings** (pojok kanan atas) → **Admin settings**
3. Pilih menu **Umum** (General)
4. Pada field **URL Website** (Site URL), ubah menjadi: `http://localhost:8080` (sesuaikan dengan port yang kamu gunakan)
5. Klik **Save**

> ❗ **Mengapa harus diubah?**
> Metabase menggunakan **URL Website** sebagai base URL saat membuat link visualisasi data, embed, dan share dashboard. Jika URL ini masih mengarah ke port lama (`3000`) padahal Metabase berjalan di port `8080`, maka link yang dibagikan akan **error / tidak bisa diakses**.

---

## Koneksi ke Apache Doris

Setelah Metabase berjalan, hubungkan ke Apache Doris sebagai sumber data.

### Langkah Koneksi:

1. Buka Metabase di browser (sesuaikan port yang digunakan)
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

## Troubleshooting

### Common Issues

| Problem                               | Solusi                                                                                          |
|---------------------------------------|-------------------------------------------------------------------------------------------------|
| `Port sudah digunakan`                | Ganti port: `export MB_JETTY_PORT=8081` sebelum menjalankan `java -jar metabase.jar`           |
| `Cannot connect to database`          | Pastikan Doris berjalan: `jps`                                                                  |
| `Java heap space / OutOfMemory`       | Tambahkan opsi JVM: `java -Xmx2g -jar metabase.jar`                                            |
| `Metabase not starting after restart` | Cek log: `sudo journalctl -u metabase -n 100 --no-pager`                                       |
| `Cannot connect to Doris`             | Pastikan FE Doris berjalan dan port `9030` tidak diblokir firewall                              |

---

## Referensi

- [Metabase Official Documentation](https://www.metabase.com/docs/latest/)
- [Metabase GitHub](https://github.com/metabase/metabase)
- [Metabase Download Page](https://www.metabase.com/docs/latest/installation-and-operation/running-the-metabase-jar-file)
- [Tutorial Menggunakan Metabase](https://medium.com/javanlabs/visualisasi-data-menggunakan-metabase-7a2b739e6be5)

---
