# 🔄 Migrasi Data Menggunakan Catalogs di Apache Doris

> **Catalog** di Apache Doris memungkinkan kita menghubungkan database eksternal (MySQL, MariaDB, PostgreSQL, dll.) secara langsung tanpa ETL tools tambahan. Data dari sumber eksternal bisa di-query langsung atau dimigrasikan ke dalam Doris.

---

## 📋 Table of Contents

- [Konsep Catalog](#konsep-catalog)
- [Prerequisites](#prerequisites)
- [Membuat JDBC Catalog](#membuat-jdbc-catalog)
- [Melihat & Mengelola Catalog](#melihat--mengelola-catalog)
- [Query Data dari Catalog](#query-data-dari-catalog)
- [Migrasi Data ke Internal Doris](#migrasi-data-ke-internal-doris)
- [Troubleshooting](#troubleshooting)

---

## Konsep Catalog

Apache Doris memiliki sistem **Multi-Catalog** yang terdiri dari:

```
┌─────────────────────────────────────────────────────────┐
│                    Apache Doris                         │
│                                                         │
│  ┌─────────────────┐     ┌───────────────────────────┐  │
│  │    internal      │     │   mariadb_source (JDBC)   │  │
│  │   (Default)      │     │                           │  │
│  │                  │     │  • MariaDB / MySQL        │  │
│  │  Data tersimpan  │◀────│  • Query langsung         │  │
│  │  di dalam Doris  │     │  • Migrasi via CTAS       │  │
│  └─────────────────┘     └───────────────────────────┘  │
│                                                         │
│  Catalog.Database.Table                                 │
│  contoh: mariadb_source.my_db.users                     │
└─────────────────────────────────────────────────────────┘
```

| Catalog Type | Keterangan |
| ------------ | ---------- |
| `internal`   | Catalog bawaan Doris, data disimpan di dalam Doris (BE) |
| `jdbc`       | Catalog eksternal, terhubung ke database lain via JDBC |

---

## Prerequisites

| Komponen | Keterangan |
| -------- | ---------- |
| **Apache Doris** | FE & BE sudah running |
| **MySQL Client** | Untuk koneksi ke Doris (`mysql -h 127.0.0.1 -P 9030 -u root`) |
| **Database Sumber** | MariaDB / MySQL yang ingin dimigrasikan |
| **Network** | Doris (BE) harus bisa mengakses IP & port database sumber |

```bash
# Connect ke Doris via MySQL client
mysql -h 127.0.0.1 -P 9030 -u root
```

---

## Membuat JDBC Catalog

### Syntax Dasar

```sql
CREATE CATALOG <nama_catalog> PROPERTIES (
    "type"            = "jdbc",
    "user"            = "<username>",
    "password"        = "<password>",
    "jdbc_url"        = "jdbc:mysql://<host>:<port>/<db_name>?<params>",
    "driver_url"      = "https://<url_ke_jdbc_driver.jar>",
    "driver_class"    = "com.mysql.jdbc.Driver",
    "checksum"        = "<checksum_hash>"
);
```

### Contoh: Membuat Catalog untuk MariaDB / MySQL

```sql
CREATE CATALOG `mariadb_source` PROPERTIES (
    "type"            = "jdbc",
    "user"            = "jordi_stage",
    "password"        = "*XXX",
    "jdbc_url"        = "jdbc:mysql://192.168.2.23:3306/ta15_wlogisticsidb_ap?zeroDateTimeBehavior=convertToNull&yearIsDateType=false&useUnicode=true&rewriteBatchedStatements=true&characterEncoding=utf-8&tinyInt1isBit=false",
    "driver_url"      = "https://repo1.maven.org/maven2/mysql/mysql-connector-java/5.1.49/mysql-connector-java-5.1.49.jar",
    "driver_class"    = "com.mysql.jdbc.Driver",
    "checksum"        = "b46c5a50b6d707b37bd34e27e0f6cbaf"
);
```

> 💡 **Catatan:**
> - `jdbc_url` — Sesuaikan IP, port, database, dan parameter koneksi MariaDB/MySQL kamu.
> - `driver_url` — Gunakan URL HTTP/HTTPS (misalnya dari repositori Maven) agar Doris dapat secara otomatis mengunduh `.jar` JDBC driver.
> - `checksum` — (Opsional) MD5 hash untuk memverifikasi integritas file driver `.jar` yang diunduh.
> - Untuk MariaDB, bisa juga menggunakan `org.mariadb.jdbc.Driver` dengan driver MariaDB Connector/J.

---

## Melihat & Mengelola Catalog

### Melihat Semua Catalog

```sql
SHOW CATALOGS\G;
```

Contoh output:

```
*************************** 1. row ***************************
     CatalogId: 0
   CatalogName: internal
          Type: internal
     IsCurrent: Yes
    CreateTime: NULL
LastUpdateTime: NULL
       Comment: Doris internal catalog

*************************** 2. row ***************************
     CatalogId: 1769483912883
   CatalogName: mariadb_source
          Type: jdbc
     IsCurrent: No
    CreateTime: 2026-01-29 13:13:46
LastUpdateTime: 2026-02-07 13:53:36
       Comment:
```

### Pindah ke Catalog Tertentu

```sql
-- Pindah ke catalog eksternal
SWITCH mariadb_source;

-- Lihat database yang tersedia
SHOW DATABASES;

-- Gunakan database tertentu
USE nama_database;

-- Lihat tabel yang tersedia
SHOW TABLES;

-- Kembali ke internal catalog
SWITCH internal;
```

### Menghapus Catalog

```sql
DROP CATALOG mariadb_source;
```

### Mengubah Properti Catalog

```sql
ALTER CATALOG mariadb_source SET PROPERTIES (
    "password" = "password_baru"
);
```

---

## Query Data dari Catalog

Setelah catalog dibuat, kamu bisa langsung query data dari sumber eksternal **tanpa perlu migrasi**:

```sql
-- Format: catalog_name.database_name.table_name
SELECT * FROM mariadb_source.nama_database.nama_table LIMIT 10;

-- Bisa juga pakai SWITCH dulu
SWITCH mariadb_source;
USE nama_database;
SELECT * FROM nama_table LIMIT 10;
```

> ⚠️ **Perhatian:** Query langsung ke catalog eksternal akan membaca data dari sumber aslinya secara real-time. Untuk performa yang lebih baik pada analytics, sebaiknya migrasikan data ke internal Doris.

---

## Migrasi Data ke Internal Doris

### Langkah 1: Siapkan Database di Internal Doris

```sql
-- Pastikan berada di internal catalog
SWITCH internal;

-- Buat database tujuan (jika belum ada)
CREATE DATABASE IF NOT EXISTS nama_database_tujuan;
```

### Langkah 2: Migrasi Tabel dengan CTAS (Create Table As Select)

Gunakan perintah **CTAS** untuk membuat tabel sekaligus menyalin data dari catalog eksternal ke internal Doris:

```sql
CREATE TABLE internal.nama_database_tujuan.nama_table
PROPERTIES ("replication_num" = "1")
AS
SELECT * FROM mariadb_source.nama_database.nama_table;
```


## Troubleshooting

### Common Issues

| Problem | Solusi |
| ------- | ------ |
| `Can't connect to MySQL server` | Pastikan IP & port database sumber bisa diakses dari Doris BE |
| `Access denied for user` | Cek username & password di properti catalog |
| `Driver class not found` | Pastikan file `.jar` JDBC driver ada di `fe/lib/` dan `be/lib/` |
| `Unknown database` | Cek nama database dengan `SHOW DATABASES` setelah `SWITCH catalog` |
| `Table not found` | Coba `REFRESH CATALOG nama_catalog;` untuk refresh metadata |
| `replication_num should be less than...` | Gunakan `"replication_num" = "1"` jika single node |

### Cek Koneksi Catalog

```sql
-- Pastikan catalog aktif
SHOW CATALOGS\G;

-- Pindah ke catalog dan cek database
SWITCH mariadb_source;
SHOW DATABASES;

-- Cek tabel dalam database
USE nama_database;
SHOW TABLES;

-- Cek struktur tabel
DESC nama_table;
```

---

## 📚 Referensi

- [Apache Doris Multi-Catalog](https://doris.apache.org/docs/lakehouse/catalogs/jdbc-catalog-overview)
- [Apache Doris CTAS](https://doris.apache.org/docs/3.x/sql-manual/sql-statements/table-and-view/table/CREATE-TABLE)

---
