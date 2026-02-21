# Migrasi CDC dari MariaDB ke Apache Doris menggunakan Flink CDC

Dokumentasi ini menjelaskan cara men-setup pekerjaan **Streaming ELT** (Extract, Load, Transform) dari **MariaDB** ke **Apache Doris** menggunakan **Flink CDC**. Proses ini memungkinkan sinkronisasi skema dasar (*schema evolution*).

> **Catatan:** Flink CDC menggunakan konektor MySQL CDC untuk membaca binlog dari MariaDB, karena MariaDB kompatibel dengan protokol replikasi MySQL. Jadi meskipun source database adalah MariaDB, tipe konektor yang digunakan tetap `mysql`.

Setup ini dilakukan di **Linux** secara langsung (*bare metal / VM*), **tanpa menggunakan Docker**.

---

## 1. Download & Install Apache Flink 1.20.1

Apache Flink adalah engine pemrosesan data streaming yang digunakan Flink CDC sebagai platform eksekusi pipeline.

### 1.1 Download Apache Flink 1.20.1

```bash
# Download Apache Flink 1.20.1
wget https://archive.apache.org/dist/flink/flink-1.20.1/flink-1.20.1-bin-scala_2.12.tgz
```

### 1.2 Extract & Tempatkan di Direktori

```bash
# Extract file
tar -xzf flink-1.20.1-bin-scala_2.12.tgz

# Pindahkan ke direktori tujuan (sesuaikan path)
mv flink-1.20.1 /home/jordi/Downloads/flink-1.20.1
```

---

## 2. Download & Install Flink CDC 3.5.0

Flink CDC adalah framework di atas Apache Flink yang menyediakan kemampuan *Change Data Capture* secara pipeline.

### 2.1 Download Flink CDC 3.5.0

```bash
# Download Flink CDC 3.5.0
wget https://dlcdn.apache.org/flink/flink-cdc-3.5.0/flink-cdc-3.5.0-bin.tar.gz
```

### 2.2 Extract Flink CDC

```bash
# Extract file
tar -xzf flink-cdc-3.5.0-bin.tar.gz
```

### 2.3 Struktur Direktori Flink CDC

Setelah diekstrak, struktur direktori `flink-cdc-3.5.0` akan berisi:

```
flink-cdc-3.5.0/
├── bin/          # Script CLI Flink CDC (flink-cdc.sh)
├── lib/          # Library inti Flink CDC & konektor pipeline
├── log/          # File log
└── conf/         # File konfigurasi
```

---

## 3. Download Konektor (JARs)

Flink CDC membutuhkan konektor pipeline untuk membaca dari source dan menulis ke sink. Selain itu, konektor JDBC MySQL juga diperlukan oleh Apache Flink itu sendiri.

### 3.1 Konektor Pipeline — Taruh di `lib/` Flink CDC

Konektor ini **harus ditaruh di dalam folder `lib/` milik Flink CDC** (`flink-cdc-3.5.0/lib/`), **BUKAN** di `lib/` Apache Flink.

```bash
# Masuk ke direktori lib Flink CDC
cd flink-cdc-3.5.0/lib/

# Download MySQL Pipeline Connector (untuk membaca binlog dari MariaDB/MySQL)
wget https://repo1.maven.org/maven2/org/apache/flink/flink-cdc-pipeline-connector-mysql/3.5.0/flink-cdc-pipeline-connector-mysql-3.5.0.jar

# Download Apache Doris Pipeline Connector (untuk menulis ke Doris)
wget https://repo1.maven.org/maven2/org/apache/flink/flink-cdc-pipeline-connector-doris/3.5.0/flink-cdc-pipeline-connector-doris-3.5.0.jar
```

Setelah download, isi folder `flink-cdc-3.5.0/lib/` seharusnya berisi:

```
flink-cdc-3.5.0/lib/
├── flink-cdc-dist-3.5.0.jar                          # (bawaan dari extract)
├── flink-cdc-pipeline-connector-mysql-3.5.0.jar       # ✅ baru didownload
└── flink-cdc-pipeline-connector-doris-3.5.0.jar       # ✅ baru didownload
```

### 3.2 MySQL JDBC Connector — Taruh di `lib/` Apache Flink

Konektor JDBC ini **harus ditaruh di dalam folder `lib/` milik Apache Flink** (`flink-1.20.1/lib/`), karena konektor ini tidak lagi di-*bundle* bersama CDC connector.

```bash
# Masuk ke direktori lib Apache Flink
cd /home/jordi/Downloads/flink-1.20.1/lib/

# Download MySQL Connector Java (JDBC driver)
wget https://repo1.maven.org/maven2/mysql/mysql-connector-java/8.0.27/mysql-connector-java-8.0.27.jar
```

Setelah download, isi folder `flink-1.20.1/lib/` seharusnya menambah:

```
flink-1.20.1/lib/
├── ... (file JAR bawaan Flink)
└── mysql-connector-java-8.0.27.jar    # ✅ baru didownload
```

### 3.3 Ringkasan Penempatan JAR

| File JAR | Taruh di | Keterangan |
|----------|----------|------------|
| `flink-cdc-pipeline-connector-mysql-3.5.0.jar` | `flink-cdc-3.5.0/lib/` | Konektor pipeline untuk membaca CDC dari MariaDB/MySQL |
| `flink-cdc-pipeline-connector-doris-3.5.0.jar` | `flink-cdc-3.5.0/lib/` | Konektor pipeline untuk menulis ke Apache Doris |
| `mysql-connector-java-8.0.27.jar` | `flink-1.20.1/lib/` | JDBC Driver MySQL, dibutuhkan oleh Flink runtime |

> ⚠️ **Penting:** Jangan tertukar. Konektor **pipeline** masuk ke `lib/` **Flink CDC**, sedangkan konektor **JDBC** masuk ke `lib/` **Apache Flink**.

---

## 4. Setup Environment (FLINK_HOME)

Setelah semua komponen terdownload dan ditempatkan di folder yang benar, langkah selanjutnya adalah mengatur environment variable `FLINK_HOME` agar Flink CDC bisa menemukan instalasi Apache Flink.

> **Tips:** Agar tidak perlu menjalankan script ini setiap kali membuka terminal baru, tambahkan baris `export` secara permanen ke file `~/.bashrc`:
> 1. Buat file dengan vim:
>    ```bash
>    vim ~/.bashrc
>    ```
> 2. Tambahkan baris berikut di bagian paling bawah file:
>    ```bash
>    #!/bin/bash
>    
>    # Set Flink Home (SESUAIKAN DENGAN PATH ANDA)
>    export FLINK_HOME=/home/jordi/Downloads/flink-1.20.1
>    
>    # Tambahkan ke system PATH
>    export PATH=$FLINK_HOME/bin:$PATH
>    
>    echo "✅ FLINK_HOME berhasil di-set ke: $FLINK_HOME"
>    ```
> 3. Simpan dan keluar dari vim (tekan `Esc`, ketik `:wq`, lalu `Enter`)
> 4. Aktifkan perubahan:
>    ```bash
>    source ~/.bashrc
>    ```


### 4.1 Start Flink Cluster

Sebelum menjalankan Flink CDC, pastikan cluster Flink sudah berjalan:

```bash
# Start Flink standalone cluster
$FLINK_HOME/bin/start-cluster.sh
```

Verifikasi cluster berjalan dengan membuka **Flink Web UI** di browser:
```
http://localhost:8081
```

Untuk menghentikan cluster:
```bash
$FLINK_HOME/bin/stop-cluster.sh
```

---

## 5. Membuat Konfigurasi Pipeline YAML
### ⚠️ CONTOH KONFIGURASI YAML YANG SAYA PAKAI

Flink CDC versi pipeline memungkinkan eksekusi proses replikasi basis data dan data streaming mutasi **tanpa kodingan Java**, murni hanya memakai bahasa deklaratif format YAML.

Buat file baru, misalnya `mysql-to-doris-migrasi.yaml`, dan isikan konfigurasi berikut:

```yaml
################################################################################
# Description: Migrasi CDC MariaDB ke Doris
################################################################################

source:
  type: mysql
  hostname: 192.168.2.23
  port: 3306
  username: jordi_stage
  password: stage12345
  tables: ta15_wlogisticsidb_ap.ENTBT,ta15_wlogisticsidb_ap.ENTBA,ta15_wlogisticsidb_ap.ENTBAC,ta15_wlogisticsidb_ap.ENTBACT,ta15_wlogisticsidb_ap.ENTBAP,ta15_wlogisticsidb_ap.ENTBAR,ta15_wlogisticsidb_ap.ENTBATY,ta15_wlogisticsidb_ap.ENTBE,ta15_wlogisticsidb_ap.ENTBEAD,ta15_wlogisticsidb_ap.ENTBEASN,ta15_wlogisticsidb_ap.ENTBEBANK,ta15_wlogisticsidb_ap.ENTBECR,ta15_wlogisticsidb_ap.ENTBECT,ta15_wlogisticsidb_ap.ENTBEDIV,ta15_wlogisticsidb_ap.ENTBEEDU,ta15_wlogisticsidb_ap.ENTBEEMP,ta15_wlogisticsidb_ap.ENTBEORG,ta15_wlogisticsidb_ap.ENTBEPU,ta15_wlogisticsidb_ap.ENTBER,ta15_wlogisticsidb_ap.ENTBESTOR,ta15_wlogisticsidb_ap.ENTBI,ta15_wlogisticsidb_ap.ENTBIBA,ta15_wlogisticsidb_ap.ENTBIC,ta15_wlogisticsidb_ap.ENTBICON,ta15_wlogisticsidb_ap.ENTBICOSTY,ta15_wlogisticsidb_ap.ENTBIDEPTY,ta15_wlogisticsidb_ap.ENTBIFA,ta15_wlogisticsidb_ap.ENTBIG,ta15_wlogisticsidb_ap.ENTBIIV,ta15_wlogisticsidb_ap.ENTBIVPLAN,ta15_wlogisticsidb_ap.ENTCAREA,ta15_wlogisticsidb_ap.ENTCDNODE,ta15_wlogisticsidb_ap.ENTBTTY,ta15_wlogisticsidb_ap.ENTLV,ta15_wlogisticsidb_ap.ENTCPCODE,ta15_wlogisticsidb_ap.ENTCPCODEAREA
  server-id: 5400
  server-time-zone: Asia/Jakarta

sink:
  type: doris
  fenodes: 192.168.4.100:18030
  username: root
  password: whnexp88
  table.create.properties.light_schema_change: true
  table.create.properties.replication_num: 1

route:
  - source-table: ta15_wlogisticsidb_ap.\.*
    sink-table: ods_prod_ta15_ap.<>
    replace-symbol: <>
    description: route all table production to staging sink

pipeline:
  name: migrasi
  parallelism: 1
  operator.uid.prefix: 100
```

### Penjelasan Konfigurasi Pipeline YAML

| Bagian | Parameter | Penjelasan |
|--------|-----------|------------|
| **`source`** | `type: mysql` | Menggunakan konektor MySQL CDC. Meskipun sumber data adalah **MariaDB**, konektor ini kompatibel karena MariaDB mendukung protokol binlog MySQL. |
| | `hostname` | IP address server MariaDB (`192.168.2.23`) |
| | `port` | Port MariaDB (default `3306`) |
| | `username` / `password` | Kredensial akses MariaDB |
| | `tables` | Daftar tabel spesifik yang akan direplikasi dari database `ta15_wlogisticsidb_ap`. Bisa menggunakan regex `\.*` untuk semua tabel, atau daftar eksplisit seperti di atas. |
| | `server-id` | ID unik untuk mengidentifikasi Flink CDC sebagai *slave* binlog MariaDB. Harus unik dan tidak boleh bentrok dengan slave lainnya. |
| | `server-time-zone` | Timezone server MariaDB (`Asia/Jakarta` = WIB / UTC+7) |
| **`sink`** | `type: doris` | Menulis data ke Apache Doris |
| | `fenodes` | Alamat Frontend node Doris (`192.168.4.100:18030`) |
| | `table.create.properties.light_schema_change` | Mengaktifkan *Light Schema Change* di Doris, sehingga operasi DDL (Add/Drop Column) bisa dilakukan secara cepat tanpa rebuild data. |
| | `table.create.properties.replication_num` | Jumlah replika tablet. Nilai `1` digunakan jika hanya ada satu BE node. |
| **`route`** | `source-table` | Regex untuk mencocokkan tabel sumber (`ta15_wlogisticsidb_ap.\.*` = semua tabel di database tersebut) |
| | `sink-table` | Format nama tabel tujuan di Doris. `<>` akan diganti otomatis dengan nama tabel asli. |
| | `replace-symbol` | Simbol yang akan di-replace (`<>`) dengan nama tabel source. |
| **`pipeline`** | `name` | Nama job Flink CDC (akan muncul di Flink Web UI) |
| | `parallelism` | Tingkat paralelisme eksekusi task |
| | `operator.uid.prefix` | Prefix unik untuk operator, berguna untuk checkpoint dan recovery |

---

## 6. Submit Job & Monitoring

Setelah file YAML (`mysql-to-doris-migrasi.yaml`) berhasil dibuat dan disimpan, jalankan pipeline migrasi CDC dengan perintah berikut:

### 6.1 Submit Job

```bash
# Pastikan FLINK_HOME sudah di-set dan cluster sudah berjalan
# Jalankan dari direktori Flink CDC
cd flink-cdc-3.5.0/

# Submit pipeline
bash bin/flink-cdc.sh mysql-to-doris-migrasi.yaml
```

### 6.2 Hasil Submit

Jika berhasil, output di terminal akan menampilkan:

```
Pipeline has been submitted to cluster.
Job ID: ae30f4580f1918bebf16752d4963dc54
Job Description: migrasi
```

### 6.3 Monitoring

| Tool | URL / Cara Akses | Keterangan |
|------|------------------|------------|
| **Flink Web UI** | `http://localhost:8081` | Melihat status job, metrik, task manager, checkpoint |
| **Doris Web UI** | `http://192.168.4.100:18030` | Melihat tabel yang otomatis dibuat dan data yang sudah tersinkronisasi |

### 6.4 Verifikasi Real-time Sync

Setelah job berjalan, setiap perubahan data (DML) pada MariaDB source akan otomatis tersinkronisasi ke Doris:

| Operasi di MariaDB | Hasil di Doris |
|---------------------|----------------|
| `INSERT INTO ta15_wlogisticsidb_ap.ENTBT ...` | Baris baru muncul di `ods_prod_ta15_ap.ENTBT` |
| `UPDATE ta15_wlogisticsidb_ap.ENTBT SET ...` | Data di `ods_prod_ta15_ap.ENTBT` ikut terupdate |
| `DELETE FROM ta15_wlogisticsidb_ap.ENTBT ...` | Data di `ods_prod_ta15_ap.ENTBT` ikut terhapus |
| `ALTER TABLE ta15_wlogisticsidb_ap.ENTBT ADD COLUMN ...` | Kolom baru otomatis ditambahkan di Doris (schema evolution) |

---

## 7. Prasyarat MariaDB (Konfigurasi Binlog)

Agar Flink CDC bisa membaca perubahan secara streaming, **MariaDB harus dikonfigurasi dengan binlog format ROW**. Pastikan konfigurasi berikut ada pada file `my.cnf` atau `mariadb.conf.d/`:

```ini
[mysqld]
server-id         = 1
log_bin           = mysql-bin
binlog_format     = ROW
binlog_row_image  = FULL
expire_logs_days  = 3
```

Setelah mengubah konfigurasi, restart service MariaDB:
```bash
sudo systemctl restart mariadb
```

Verifikasi binlog aktif:
```sql
SHOW VARIABLES LIKE 'log_bin';
SHOW VARIABLES LIKE 'binlog_format';
```

> **Penting:** User MariaDB yang digunakan oleh Flink CDC (`jordi_stage`) harus memiliki privilege:
> ```sql
> GRANT SELECT, RELOAD, SHOW DATABASES, REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'jordi_stage'@'%';
> FLUSH PRIVILEGES;
> ```

---

## 8. Troubleshooting

| Masalah | Kemungkinan Penyebab | Solusi |
|---------|---------------------|--------|
| `Could not find any class that implements Connector` | Konektor JAR tidak ditaruh di folder `lib/` yang benar | Pastikan pipeline connector ada di `flink-cdc-3.5.0/lib/` dan JDBC connector ada di `flink-1.20.1/lib/` |
| `Access denied for user` | Credential atau privilege MariaDB salah | Cek username/password dan pastikan user punya privilege REPLICATION SLAVE |
| `Server id conflict` | `server-id` bentrok dengan slave lain | Ganti `server-id` ke nilai unik yang lain |
| `Cannot find FLINK_HOME` | Environment variable belum diset | Jalankan script setup FLINK_HOME atau tambahkan ke `~/.bashrc` |
| `Cluster unreachable` | Flink cluster belum dijalankan | Jalankan `$FLINK_HOME/bin/start-cluster.sh` terlebih dahulu |
| `Doris connection refused` | FE node Doris tidak aktif atau port salah | Verifikasi Doris FE running dan port `18030` accessible |

---

## 📚 Referensi

- Langkah-langkah diringkas berdasarkan [Dokumentasi Resmi Flink CDC: Quickstart MySQL to Doris](https://nightlies.apache.org/flink/flink-cdc-docs-release-3.5/docs/get-started/quickstart/mysql-to-doris/), dengan mengaplikasikan konfigurasi yang spesifik pada lingkungan kerja server.
- [Apache Flink CDC GitHub](https://github.com/apache/flink-cdc)

---

