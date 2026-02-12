# 🔄 Query Migrasi Data Apache Doris Menggunakan Apache Flink

> Dokumentasi ini berisi **query lengkap** untuk migrasi data dari **MariaDB (localhost)** ke **Apache Doris (server SSH)** menggunakan Apache Flink SQL Client. Termasuk pembuatan data dummy, persiapan tabel, dan 2 metode migrasi: **JDBC (batch)** & **CDC (real-time)**.

---

## 📋 Table of Contents

- [Info Koneksi](#info-koneksi)
- [Persiapan Data Dummy di MariaDB](#persiapan-data-dummy-di-mariadb)
- [Persiapan Tabel di Apache Doris](#persiapan-tabel-di-apache-doris)
- [Perbandingan Metode Migrasi: JDBC vs CDC](#perbandingan-metode-migrasi-jdbc-vs-cdc)
- [Metode 1: Migrasi JDBC (Batch / One-Time)](#metode-1-migrasi-jdbc-batch--one-time)
- [Metode 2: Migrasi CDC (Real-Time / Streaming)](#metode-2-migrasi-cdc-real-time--streaming)
- [Verifikasi Hasil Migrasi](#verifikasi-hasil-migrasi)

---

## Info Koneksi

| Service | Host | Port | Username | Password |
| ------- | ---- | ---- | -------- | -------- |
| **MariaDB** (source) | `127.0.0.1` (localhost) | `3306` | `root` | `password_kamu` |
| **Apache Doris FE HTTP** (sink) | `192.168.4.100` | `18030` | `root` | *(kosong)* |
| **Apache Doris FE MySQL** | `192.168.4.100` | `19030` | `root` | *(kosong)* |
| **Flink Web UI** | `localhost` | `8081` | — | — |

---

## Persiapan Data Dummy di MariaDB

Buat database dan data dummy di MariaDB localhost terlebih dahulu.

### Connect ke MariaDB

```bash
sudo mariadb -u root -p
```

### Buat Database & Tabel

```sql
-- 🗄️ Buat Database
CREATE DATABASE IF NOT EXISTS demo_db;
USE demo_db;

-- ========================================
-- 📊 Tabel 1: employees (Data Karyawan)
-- ========================================
CREATE TABLE IF NOT EXISTS employees (
    id             INT AUTO_INCREMENT PRIMARY KEY,
    name           VARCHAR(100) NOT NULL,
    email          VARCHAR(100),
    phone          VARCHAR(20),
    department     VARCHAR(50),
    position       VARCHAR(50),
    salary         DECIMAL(15, 2),
    hire_date      DATE,
    address        TEXT,
    created_at     TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at     TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

-- ========================================
-- 📊 Tabel 2: orders (Data Pesanan)
-- ========================================
CREATE TABLE IF NOT EXISTS orders (
    order_id       INT AUTO_INCREMENT PRIMARY KEY,
    customer_name  VARCHAR(100) NOT NULL,
    product_name   VARCHAR(100),
    quantity       INT,
    total_price    DECIMAL(15, 2),
    order_status   VARCHAR(20),
    order_date     DATE,
    shipping_addr  TEXT,
    created_at     TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at     TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

-- ========================================
-- 📊 Tabel 3: products (Data Produk)
-- ========================================
CREATE TABLE IF NOT EXISTS products (
    product_id     INT AUTO_INCREMENT PRIMARY KEY,
    product_name   VARCHAR(100) NOT NULL,
    category       VARCHAR(50),
    brand          VARCHAR(50),
    price          DECIMAL(15, 2),
    stock          INT,
    description    TEXT,
    created_at     TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at     TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);
```

### Insert Data Dummy

```sql
-- ========================================
-- 📝 Data Dummy: employees (20 rows)
-- ========================================
INSERT INTO employees (name, email, phone, department, position, salary, hire_date, address) VALUES
('Budi Santoso',      'budi.santoso@email.com',     '081234567801', 'Engineering',  'Software Engineer',    15000000.00, '2023-01-15', 'Jl. Merdeka No. 1, Jakarta'),
('Siti Rahayu',       'siti.rahayu@email.com',      '081234567802', 'Engineering',  'Senior Developer',     20000000.00, '2022-03-10', 'Jl. Sudirman No. 45, Jakarta'),
('Ahmad Hidayat',     'ahmad.hidayat@email.com',    '081234567803', 'Data',         'Data Engineer',        18000000.00, '2023-05-20', 'Jl. Gatot Subroto No. 12, Bandung'),
('Dewi Lestari',      'dewi.lestari@email.com',     '081234567804', 'Data',         'Data Analyst',         14000000.00, '2023-07-01', 'Jl. Asia Afrika No. 8, Bandung'),
('Rizky Pratama',     'rizky.pratama@email.com',    '081234567805', 'Engineering',  'DevOps Engineer',      17000000.00, '2022-11-20', 'Jl. Diponegoro No. 33, Surabaya'),
('Nurul Aini',        'nurul.aini@email.com',       '081234567806', 'HR',           'HR Manager',           22000000.00, '2021-06-15', 'Jl. Pemuda No. 77, Semarang'),
('Fajar Ramadhan',    'fajar.ramadhan@email.com',   '081234567807', 'Finance',      'Accountant',           13000000.00, '2024-01-10', 'Jl. Ahmad Yani No. 5, Yogyakarta'),
('Putri Wulandari',   'putri.wulandari@email.com',  '081234567808', 'Marketing',    'Marketing Manager',    21000000.00, '2022-08-25', 'Jl. Malioboro No. 99, Yogyakarta'),
('Hendra Gunawan',    'hendra.gunawan@email.com',   '081234567809', 'Engineering',  'Tech Lead',            25000000.00, '2021-02-01', 'Jl. Thamrin No. 10, Jakarta'),
('Maya Sari',         'maya.sari@email.com',        '081234567810', 'Data',         'BI Analyst',           16000000.00, '2023-09-14', 'Jl. Braga No. 21, Bandung'),
('Andi Wijaya',       'andi.wijaya@email.com',      '081234567811', 'Engineering',  'Backend Developer',    16000000.00, '2023-04-01', 'Jl. Pahlawan No. 15, Malang'),
('Rina Marlina',      'rina.marlina@email.com',     '081234567812', 'HR',           'Recruiter',            12000000.00, '2024-02-15', 'Jl. Veteran No. 30, Surabaya'),
('Dani Kurniawan',    'dani.kurniawan@email.com',   '081234567813', 'Finance',      'Finance Manager',      23000000.00, '2021-09-01', 'Jl. Gajah Mada No. 18, Jakarta'),
('Lina Permata',      'lina.permata@email.com',     '081234567814', 'Marketing',    'Content Writer',       11000000.00, '2024-03-10', 'Jl. Imam Bonjol No. 7, Medan'),
('Yoga Aditya',       'yoga.aditya@email.com',      '081234567815', 'Engineering',  'Frontend Developer',   15500000.00, '2023-06-20', 'Jl. Juanda No. 25, Bogor'),
('Fitri Handayani',   'fitri.handayani@email.com',  '081234567816', 'Data',         'Data Scientist',       22000000.00, '2022-12-01', 'Jl. Cendana No. 9, Depok'),
('Bayu Setiawan',     'bayu.setiawan@email.com',    '081234567817', 'Engineering',  'QA Engineer',          14500000.00, '2023-08-15', 'Jl. Kenanga No. 44, Tangerang'),
('Ayu Kartika',       'ayu.kartika@email.com',      '081234567818', 'HR',           'HR Admin',             10000000.00, '2024-04-01', 'Jl. Melati No. 11, Bekasi'),
('Reza Mahendra',     'reza.mahendra@email.com',    '081234567819', 'Engineering',  'System Architect',     28000000.00, '2020-10-15', 'Jl. Anggrek No. 6, Jakarta'),
('Indah Permatasari', 'indah.permatasari@email.com','081234567820', 'Marketing',    'Digital Marketing',    13500000.00, '2023-11-01', 'Jl. Mawar No. 52, Bandung');

-- ========================================
-- 📝 Data Dummy: orders (20 rows)
-- ========================================
INSERT INTO orders (customer_name, product_name, quantity, total_price, order_status, order_date, shipping_addr) VALUES
('Toko Maju Jaya',     'Laptop Asus ROG',       2,  45000000.00, 'COMPLETED',  '2025-01-05', 'Jl. Raya Bogor No. 10, Jakarta'),
('CV Berkah Abadi',    'Monitor LG 27"',         5,  17500000.00, 'COMPLETED',  '2025-01-08', 'Jl. Industri No. 22, Surabaya'),
('PT Sukses Mandiri',  'Keyboard Mechanical',    10, 8500000.00,  'SHIPPED',    '2025-01-12', 'Jl. Sudirman No. 88, Bandung'),
('Toko Komputer ABC',  'Mouse Wireless',         20, 3000000.00,  'COMPLETED',  '2025-01-15', 'Jl. Gatot Subroto No. 5, Semarang'),
('UD Elektronik 123',  'SSD Samsung 1TB',        8,  12000000.00, 'PROCESSING', '2025-01-20', 'Jl. Pemuda No. 33, Yogyakarta'),
('PT Indo Teknologi',  'RAM DDR5 32GB',          15, 22500000.00, 'COMPLETED',  '2025-02-01', 'Jl. Thamrin No. 71, Jakarta'),
('Toko Digital Plus',  'Webcam Logitech',        12, 7200000.00,  'SHIPPED',    '2025-02-05', 'Jl. Asia Afrika No. 19, Bandung'),
('CV Sinar Harapan',   'Headset Gaming',         6,  5400000.00,  'COMPLETED',  '2025-02-10', 'Jl. Diponegoro No. 50, Malang'),
('PT Mitra Solusi',    'Printer Epson L3210',     3,  7500000.00,  'CANCELLED',  '2025-02-14', 'Jl. Ahmad Yani No. 44, Medan'),
('Toko Serba Ada',     'UPS APC 1200VA',         4,  8000000.00,  'PROCESSING', '2025-02-18', 'Jl. Pahlawan No. 8, Palembang'),
('UD Jaya Makmur',     'Laptop Lenovo ThinkPad', 1,  18000000.00, 'COMPLETED',  '2025-03-01', 'Jl. Veteran No. 15, Makassar'),
('PT Karya Bangsa',    'Server Dell PowerEdge',  1,  85000000.00, 'SHIPPED',    '2025-03-05', 'Jl. Gajah Mada No. 90, Jakarta'),
('CV Sentosa Abadi',   'Switch Cisco 24 Port',   2,  12000000.00, 'COMPLETED',  '2025-03-10', 'Jl. Imam Bonjol No. 27, Surabaya'),
('Toko IT Solution',   'Router MikroTik',        5,  7500000.00,  'PROCESSING', '2025-03-15', 'Jl. Juanda No. 63, Bogor'),
('PT Global Tekno',    'NAS Synology 4Bay',      2,  16000000.00, 'COMPLETED',  '2025-03-20', 'Jl. Cendana No. 34, Depok'),
('UD Prima Comp',      'Kabel LAN Cat6 100m',    10, 5000000.00,  'COMPLETED',  '2025-04-01', 'Jl. Kenanga No. 18, Tangerang'),
('CV Tech Indonesia',  'GPU RTX 4070',           3,  33000000.00, 'SHIPPED',    '2025-04-05', 'Jl. Melati No. 72, Bekasi'),
('PT Data Nusantara',  'SSD NVMe 2TB',           5,  12500000.00, 'COMPLETED',  '2025-04-10', 'Jl. Anggrek No. 41, Jakarta'),
('Toko Mega Komputer', 'Monitor Samsung 32"',    4,  20000000.00, 'PROCESSING', '2025-04-15', 'Jl. Mawar No. 6, Bandung'),
('CV Bintang Jaya',    'Docking Station USB-C',  7,  10500000.00, 'COMPLETED',  '2025-04-20', 'Jl. Braga No. 55, Bandung');

-- ========================================
-- 📝 Data Dummy: products (20 rows)
-- ========================================
INSERT INTO products (product_name, category, brand, price, stock, description) VALUES
('Laptop Asus ROG Strix G16',    'Laptop',       'Asus',      22500000.00, 15,  'Gaming laptop with RTX 4060, i7-13650HX'),
('Laptop Lenovo ThinkPad X1',    'Laptop',       'Lenovo',    18000000.00, 10,  'Business laptop, i7-1365U, 16GB RAM'),
('Monitor LG 27UL850 27"',       'Monitor',      'LG',        3500000.00,  25,  'IPS 4K UHD, USB-C, HDR10'),
('Monitor Samsung Odyssey 32"',  'Monitor',      'Samsung',   5000000.00,  20,  'Curved gaming monitor, 165Hz, QHD'),
('Keyboard Rexus Daxa M84 Pro',  'Accessories',  'Rexus',     850000.00,   50,  'Mechanical keyboard, hot-swappable, RGB'),
('Mouse Logitech MX Master 3S',  'Accessories',  'Logitech',  1500000.00,  40,  'Wireless ergonomic mouse, 8000 DPI'),
('Webcam Logitech C920',         'Accessories',  'Logitech',  600000.00,   35,  'Full HD 1080p webcam, auto-focus'),
('Headset HyperX Cloud II',      'Audio',        'HyperX',    900000.00,   30,  '7.1 surround sound, detachable mic'),
('SSD Samsung 870 EVO 1TB',      'Storage',      'Samsung',   1500000.00,  45,  'SATA III, 560MB/s read, 530MB/s write'),
('SSD NVMe Samsung 980 Pro 2TB', 'Storage',      'Samsung',   2500000.00,  20,  'PCIe 4.0, 7000MB/s read'),
('RAM Corsair DDR5 32GB',        'Memory',       'Corsair',   1500000.00,  60,  'DDR5-5600MHz, CL36, dual channel kit'),
('Printer Epson EcoTank L3210',  'Printer',      'Epson',     2500000.00,  18,  'All-in-one ink tank printer'),
('UPS APC BX1200MI 1200VA',      'Power',        'APC',       2000000.00,  12,  'Line-interactive UPS, AVR, 600W'),
('Server Dell PowerEdge T150',   'Server',       'Dell',      85000000.00, 3,   'Xeon E-2314, 16GB ECC, 2TB HDD'),
('Switch Cisco SG350-28P',       'Networking',   'Cisco',     6000000.00,  8,   '28-port gigabit PoE managed switch'),
('Router MikroTik RB5009UG',     'Networking',   'MikroTik',  1500000.00,  22,  'RouterOS, 2.5G Ethernet, SFP+'),
('NAS Synology DS923+ 4Bay',     'Storage',      'Synology',  8000000.00,  5,   '4-bay NAS, AMD Ryzen R1600, 4GB RAM'),
('GPU Nvidia RTX 4070 12GB',     'GPU',          'Nvidia',    11000000.00, 7,   '12GB GDDR6X, DLSS 3, Ray Tracing'),
('Kabel LAN Cat6 UTP 100m',     'Networking',   'Belden',    500000.00,   100, 'Cat6 UTP, 1Gbps, pure copper'),
('Docking Station USB-C 12in1',  'Accessories',  'Ugreen',    1500000.00,  28,  'USB-C hub, HDMI 4K, PD 100W, Ethernet');
```

### Verifikasi Data Dummy

```sql
-- Cek jumlah data
SELECT 'employees' AS tabel, COUNT(*) AS jumlah FROM employees
UNION ALL
SELECT 'orders', COUNT(*) FROM orders
UNION ALL
SELECT 'products', COUNT(*) FROM products;

-- Output:
-- +-----------+--------+
-- | tabel     | jumlah |
-- +-----------+--------+
-- | employees |     20 |
-- | orders    |     20 |
-- | products  |     20 |
-- +-----------+--------+
```

---

## Persiapan Tabel di Apache Doris

Buat database dan tabel tujuan di Apache Doris (server SSH):

```bash
# Connect ke Doris di server SSH
mysql -h 192.168.4.100 -P 19030 -u root
```

```sql
-- 🗄️ Buat Database
CREATE DATABASE IF NOT EXISTS demo_db;
USE demo_db;

-- ========================================
-- 📊 Tabel 1: employees
-- ========================================
CREATE TABLE IF NOT EXISTS employees (
    id             INT NOT NULL,
    name           VARCHAR(100),
    email          VARCHAR(100),
    phone          VARCHAR(20),
    department     VARCHAR(50),
    position       VARCHAR(50),
    salary         DECIMAL(15, 2),
    hire_date      DATE,
    address        TEXT,
    created_at     DATETIME,
    updated_at     DATETIME
)
UNIQUE KEY(id)
DISTRIBUTED BY HASH(id) BUCKETS 10
PROPERTIES (
    "replication_num" = "1"
);

-- ========================================
-- 📊 Tabel 2: orders
-- ========================================
CREATE TABLE IF NOT EXISTS orders (
    order_id       INT NOT NULL,
    customer_name  VARCHAR(100),
    product_name   VARCHAR(100),
    quantity       INT,
    total_price    DECIMAL(15, 2),
    order_status   VARCHAR(20),
    order_date     DATE,
    shipping_addr  TEXT,
    created_at     DATETIME,
    updated_at     DATETIME
)
UNIQUE KEY(order_id)
DISTRIBUTED BY HASH(order_id) BUCKETS 10
PROPERTIES (
    "replication_num" = "1"
);

-- ========================================
-- 📊 Tabel 3: products
-- ========================================
CREATE TABLE IF NOT EXISTS products (
    product_id     INT NOT NULL,
    product_name   VARCHAR(100),
    category       VARCHAR(50),
    brand          VARCHAR(50),
    price          DECIMAL(15, 2),
    stock          INT,
    description    TEXT,
    created_at     DATETIME,
    updated_at     DATETIME
)
UNIQUE KEY(product_id)
DISTRIBUTED BY HASH(product_id) BUCKETS 10
PROPERTIES (
    "replication_num" = "1"
);
```

---

## Perbandingan Metode Migrasi: JDBC vs CDC

| Aspek | 📦 JDBC (Batch) | 🔄 CDC (Streaming) |
| ----- | --------------- | ------------------- |
| **Tujuan** | Migrasi data satu kali (one-time copy) | Sinkronisasi real-time terus-menerus |
| **Cara Kerja** | `SELECT *` dari MariaDB via JDBC | Membaca binlog MariaDB |
| **Execution Mode** | `batch` | `streaming` |
| **Job Selesai?** | ✅ Otomatis selesai setelah semua data tercopy | ❌ Berjalan terus (harus di-cancel manual) |
| **Menangkap UPDATE/DELETE?** | ❌ Tidak, hanya snapshot data saat ini | ✅ Ya, menangkap semua perubahan |
| **Butuh Binlog?** | ❌ Tidak perlu | ✅ Harus aktif (`binlog_format = ROW`) |
| **Cocok Untuk** | Migrasi awal, backup, data historis | Sinkronisasi real-time, data pipeline |
| **Connector** | `flink-connector-jdbc` | `flink-sql-connector-mysql-cdc` |

### Kapan Pakai Yang Mana?

```
🟢 Pakai JDBC jika:
   • Hanya butuh copy data sekali saja
   • Tidak perlu menangkap perubahan real-time
   • Tidak ingin mengaktifkan binlog di MariaDB
   • Migrasi data historis / arsip

🔵 Pakai CDC jika:
   • Butuh sinkronisasi data real-time
   • Data di MariaDB terus berubah (INSERT/UPDATE/DELETE)
   • Membangun data pipeline yang hidup terus-menerus
   • Ingin data di Doris selalu up-to-date dengan MariaDB
```

---

## Metode 1: Migrasi JDBC (Batch / One-Time)

> 📦 Metode ini membaca semua data dari MariaDB via `SELECT` query dan menulisnya ke Doris. **Job akan selesai otomatis** setelah semua data selesai dicopy.

### Alur JDBC

```
MariaDB localhost:3306
    │
    │  SELECT * FROM table
    ▼
┌──────────────┐
│  Flink SQL   │  execution.runtime-mode = 'batch'
│  JDBC Source  │
└──────┬───────┘
       │
       │  INSERT INTO doris_table
       ▼
┌──────────────┐
│  Flink SQL   │
│  Doris Sink  │
└──────┬───────┘
       │
       ▼
Apache Doris 192.168.4.100:18030
    ✅ Job FINISHED (selesai otomatis)
```

### Langkah 1: Set Execution Mode ke Batch

Masuk ke Flink SQL Client:

```bash
./bin/sql-client.sh
```

```sql
-- ⚠️ PENTING: Set mode ke BATCH untuk JDBC
SET 'execution.runtime-mode' = 'batch';

-- Set result mode
SET 'sql-client.execution.result-mode' = 'tableau';
```

### Langkah 2: Migrasi Tabel employees

```sql
-- ========================================
-- 📡 Source: MariaDB JDBC - employees
-- ========================================
CREATE TABLE jdbc_employees (
    id             INT,
    name           STRING,
    email          STRING,
    phone          STRING,
    department     STRING,
    `position`     STRING,
    salary         DECIMAL(15, 2),
    hire_date      DATE,
    address        STRING,
    created_at     TIMESTAMP(3),
    updated_at     TIMESTAMP(3),
    PRIMARY KEY (id) NOT ENFORCED
) WITH (
    'connector'    = 'jdbc',
    'url'          = 'jdbc:mysql://127.0.0.1:3306/demo_db',
    'table-name'   = 'employees',
    'username'     = 'root',
    'password'     = 'password_kamu',
    'driver'       = 'com.mysql.cj.jdbc.Driver'
);

-- ========================================
-- 🎯 Sink: Apache Doris - employees
-- ========================================
CREATE TABLE doris_employees (
    id             INT,
    name           STRING,
    email          STRING,
    phone          STRING,
    department     STRING,
    `position`     STRING,
    salary         DECIMAL(15, 2),
    hire_date      DATE,
    address        STRING,
    created_at     TIMESTAMP(3),
    updated_at     TIMESTAMP(3),
    PRIMARY KEY (id) NOT ENFORCED
) WITH (
    'connector'                          = 'doris',
    'fenodes'                            = '192.168.4.100:18030',
    'table.identifier'                   = 'demo_db.employees',
    'username'                           = 'root',
    'password'                           = '',
    'sink.properties.format'             = 'json',
    'sink.properties.read_json_by_line'  = 'true',
    'sink.label-prefix'                  = 'flink_jdbc_employees'
);

-- 🚀 Jalankan Migrasi (selesai otomatis)
INSERT INTO doris_employees
SELECT * FROM jdbc_employees;
```

### Langkah 3: Migrasi Tabel orders

```sql
-- ========================================
-- 📡 Source: MariaDB JDBC - orders
-- ========================================
CREATE TABLE jdbc_orders (
    order_id       INT,
    customer_name  STRING,
    product_name   STRING,
    quantity       INT,
    total_price    DECIMAL(15, 2),
    order_status   STRING,
    order_date     DATE,
    shipping_addr  STRING,
    created_at     TIMESTAMP(3),
    updated_at     TIMESTAMP(3),
    PRIMARY KEY (order_id) NOT ENFORCED
) WITH (
    'connector'    = 'jdbc',
    'url'          = 'jdbc:mysql://127.0.0.1:3306/demo_db',
    'table-name'   = 'orders',
    'username'     = 'flink',
    'password'     = 'flink123',
    'driver'       = 'com.mysql.cj.jdbc.Driver'
);

-- ========================================
-- 🎯 Sink: Apache Doris - orders
-- ========================================
CREATE TABLE doris_orders (
    order_id       INT,
    customer_name  STRING,
    product_name   STRING,
    quantity       INT,
    total_price    DECIMAL(15, 2),
    order_status   STRING,
    order_date     DATE,
    shipping_addr  STRING,
    created_at     TIMESTAMP(3),
    updated_at     TIMESTAMP(3),
    PRIMARY KEY (order_id) NOT ENFORCED
) WITH (
    'connector'                          = 'doris',
    'fenodes'                            = '192.168.4.100:18030',
    'table.identifier'                   = 'demo_db.orders',
    'username'                           = 'root',
    'password'                           = '',
    'sink.properties.format'             = 'json',
    'sink.properties.read_json_by_line'  = 'true',
    'sink.label-prefix'                  = 'flink_jdbc_orders'
);

-- 🚀 Jalankan Migrasi (selesai otomatis)
INSERT INTO doris_orders
SELECT * FROM jdbc_orders;
```

### Langkah 4: Migrasi Tabel products

```sql
-- ========================================
-- 📡 Source: MariaDB JDBC - products
-- ========================================
CREATE TABLE jdbc_products (
    product_id     INT,
    product_name   STRING,
    category       STRING,
    brand          STRING,
    price          DECIMAL(15, 2),
    stock          INT,
    description    STRING,
    created_at     TIMESTAMP(3),
    updated_at     TIMESTAMP(3),
    PRIMARY KEY (product_id) NOT ENFORCED
) WITH (
    'connector'    = 'jdbc',
    'url'          = 'jdbc:mysql://127.0.0.1:3306/demo_db',
    'table-name'   = 'products',
    'username'     = 'flink',
    'password'     = 'flink123',
    'driver'       = 'com.mysql.cj.jdbc.Driver'
);

-- ========================================
-- 🎯 Sink: Apache Doris - products
-- ========================================
CREATE TABLE doris_products (
    product_id     INT,
    product_name   STRING,
    category       STRING,
    brand          STRING,
    price          DECIMAL(15, 2),
    stock          INT,
    description    STRING,
    created_at     TIMESTAMP(3),
    updated_at     TIMESTAMP(3),
    PRIMARY KEY (product_id) NOT ENFORCED
) WITH (
    'connector'                          = 'doris',
    'fenodes'                            = '192.168.4.100:18030',
    'table.identifier'                   = 'demo_db.products',
    'username'                           = 'root',
    'password'                           = '',
    'sink.properties.format'             = 'json',
    'sink.properties.read_json_by_line'  = 'true',
    'sink.label-prefix'                  = 'flink_jdbc_products'
);

-- 🚀 Jalankan Migrasi (selesai otomatis)
INSERT INTO doris_products
SELECT * FROM jdbc_products;
```

### Migrasi JDBC dengan Filter & Transformasi

```sql
-- Migrasi hanya order yang COMPLETED
INSERT INTO doris_orders
SELECT * FROM jdbc_orders
WHERE order_status = 'COMPLETED';

-- Migrasi dengan transformasi kolom
INSERT INTO doris_employees
SELECT
    id, UPPER(name), email, phone,
    department, `position`, salary, hire_date,
    address, created_at, updated_at
FROM jdbc_employees
WHERE department = 'Engineering';
```

---

## Metode 2: Migrasi CDC (Real-Time / Streaming)

> 🔄 Metode ini membaca binlog MariaDB dan menangkap semua perubahan (INSERT/UPDATE/DELETE) secara real-time. **Job berjalan terus-menerus** sampai di-cancel manual.

### Alur CDC

```
MariaDB localhost:3306
    │
    │  1. Snapshot awal (SELECT * seluruh data)
    │  2. Baca binlog (perubahan real-time)
    ▼
┌──────────────┐
│  Flink SQL   │  execution.runtime-mode = 'streaming'
│  CDC Source   │
└──────┬───────┘
       │
       │  INSERT INTO doris_table
       ▼
┌──────────────┐
│  Flink SQL   │
│  Doris Sink  │
└──────┬───────┘
       │
       ▼
Apache Doris 192.168.4.100:18030
    🔄 Job RUNNING (berjalan terus, cancel manual)
```

### Langkah 1: Persiapan MariaDB (Aktifkan Binlog)

Pastikan MariaDB sudah mengaktifkan binary log:

```bash
# Cek status binlog di MariaDB
mysql -h 127.0.0.1 -P 3306 -u root -p -e "SHOW VARIABLES LIKE 'log_bin';"
#ATAU
SHOW VARIABLES LIKE 'log_bin';
# Harus: ON
```

Jika belum aktif, edit `/etc/mysql/mariadb.conf.d/50-server.cnf`:

```ini
[mysqld]
server-id         = 1
log_bin           = mysql-bin
binlog_format     = ROW
binlog_row_image  = FULL
expire_logs_days  = 3
```

Restart MariaDB:

```bash
sudo systemctl restart mariadb
```

Berikan privilege CDC ke user `flink` agar bisa membaca binlog:

```sql
-- Login sebagai root ke MariaDB
-- mysql -h 127.0.0.1 -P 3306 -u root -p

-- ========================================
-- 🔑 Grant Privilege CDC untuk user flink
-- ========================================
-- MariaDB 10.5+ menggunakan BINLOG MONITOR
-- (menggantikan REPLICATION CLIENT versi lama)
GRANT BINLOG MONITOR, REPLICATION SLAVE, SELECT ON *.* TO 'flink'@'%';
FLUSH PRIVILEGES;

-- ✅ Verifikasi privilege
SHOW GRANTS FOR 'flink'@'%';
```

> ⚠️ **Catatan:** Jika tanpa privilege `BINLOG MONITOR`, Flink CDC akan error:
> ```
> Access denied; you need (at least one of) the BINLOG MONITOR privilege(s) for this operation
> ```

### Langkah 2: Set Execution Mode ke Streaming

Masuk ke Flink SQL Client:

```bash
./bin/sql-client.sh
```

```sql
-- ⚠️ PENTING: Set mode ke STREAMING untuk CDC
SET 'execution.runtime-mode' = 'streaming';

-- Set checkpoint interval
SET 'execution.checkpointing.interval' = '10s';

-- Set result mode
SET 'sql-client.execution.result-mode' = 'changelog';
```

### Langkah 3: Migrasi CDC — employees

```sql
-- ========================================
-- 📡 Source: MariaDB CDC - employees
-- ========================================
CREATE TABLE cdc_employees (
    id             INT,
    name           STRING,
    email          STRING,
    phone          STRING,
    department     STRING,
    `position`     STRING,
    salary         DECIMAL(15, 2),
    hire_date      DATE,
    address        STRING,
    created_at     TIMESTAMP(3),
    updated_at     TIMESTAMP(3),
    PRIMARY KEY (id) NOT ENFORCED
) WITH (
    'connector'     = 'mysql-cdc',
    'hostname'      = '127.0.0.1',
    'port'          = '3306',
    'username'      = 'flink',
    'password'      = 'flink123',
    'database-name' = 'demo_db',
    'table-name'    = 'employees'
);

-- ========================================
-- 🎯 Sink: Apache Doris - employees
-- ========================================
CREATE TABLE doris_employees_cdc (
    id             INT,
    name           STRING,
    email          STRING,
    phone          STRING,
    department     STRING,
    `position`     STRING,
    salary         DECIMAL(15, 2),
    hire_date      DATE,
    address        STRING,
    created_at     TIMESTAMP(3),
    updated_at     TIMESTAMP(3),
    PRIMARY KEY (id) NOT ENFORCED
) WITH (
    'connector'                          = 'doris',
    'fenodes'                            = '192.168.4.100:18030',
    'table.identifier'                   = 'demo_db.employees',
    'username'                           = 'root',
    'password'                           = '',
    'sink.properties.format'             = 'json',
    'sink.properties.read_json_by_line'  = 'true',
    'sink.enable-2pc'                    = 'true',
    'sink.label-prefix'                  = 'flink_cdc_employees'
);

-- 🔄 Jalankan Migrasi CDC (berjalan terus)
INSERT INTO doris_employees_cdc
SELECT * FROM cdc_employees;
```

### Langkah 4: Migrasi CDC — orders

```sql
-- ========================================
-- 📡 Source: MariaDB CDC - orders
-- ========================================
CREATE TABLE cdc_orders (
    order_id       INT,
    customer_name  STRING,
    product_name   STRING,
    quantity       INT,
    total_price    DECIMAL(15, 2),
    order_status   STRING,
    order_date     DATE,
    shipping_addr  STRING,
    created_at     TIMESTAMP(3),
    updated_at     TIMESTAMP(3),
    PRIMARY KEY (order_id) NOT ENFORCED
) WITH (
    'connector'     = 'mysql-cdc',
    'hostname'      = '127.0.0.1',
    'port'          = '3306',
    'username'      = 'flink',
    'password'      = 'flink123',
    'database-name' = 'demo_db',
    'table-name'    = 'orders'
);

-- ========================================
-- 🎯 Sink: Apache Doris - orders
-- ========================================
CREATE TABLE doris_orders_cdc (
    order_id       INT,
    customer_name  STRING,
    product_name   STRING,
    quantity       INT,
    total_price    DECIMAL(15, 2),
    order_status   STRING,
    order_date     DATE,
    shipping_addr  STRING,
    created_at     TIMESTAMP(3),
    updated_at     TIMESTAMP(3),
    PRIMARY KEY (order_id) NOT ENFORCED
) WITH (
    'connector'                          = 'doris',
    'fenodes'                            = '192.168.4.100:18030',
    'table.identifier'                   = 'demo_db.orders',
    'username'                           = 'root',
    'password'                           = '',
    'sink.properties.format'             = 'json',
    'sink.properties.read_json_by_line'  = 'true',
    'sink.enable-2pc'                    = 'true',
    'sink.label-prefix'                  = 'flink_cdc_orders'
);

-- 🔄 Jalankan Migrasi CDC (berjalan terus)
INSERT INTO doris_orders_cdc
SELECT * FROM cdc_orders;
```

### Langkah 5: Migrasi CDC — products

```sql
-- ========================================
-- 📡 Source: MariaDB CDC - products
-- ========================================
CREATE TABLE cdc_products (
    product_id     INT,
    product_name   STRING,
    category       STRING,
    brand          STRING,
    price          DECIMAL(15, 2),
    stock          INT,
    description    STRING,
    created_at     TIMESTAMP(3),
    updated_at     TIMESTAMP(3),
    PRIMARY KEY (product_id) NOT ENFORCED
) WITH (
    'connector'     = 'mysql-cdc',
    'hostname'      = '127.0.0.1',
    'port'          = '3306',
    'username'      = 'flink',
    'password'      = 'flink123',
    'database-name' = 'demo_db',
    'table-name'    = 'products'
);

-- ========================================
-- 🎯 Sink: Apache Doris - products
-- ========================================
CREATE TABLE doris_products_cdc (
    product_id     INT,
    product_name   STRING,
    category       STRING,
    brand          STRING,
    price          DECIMAL(15, 2),
    stock          INT,
    description    STRING,
    created_at     TIMESTAMP(3),
    updated_at     TIMESTAMP(3),
    PRIMARY KEY (product_id) NOT ENFORCED
) WITH (
    'connector'                          = 'doris',
    'fenodes'                            = '192.168.4.100:18030',
    'table.identifier'                   = 'demo_db.products',
    'username'                           = 'root',
    'password'                           = '',
    'sink.properties.format'             = 'json',
    'sink.properties.read_json_by_line'  = 'true',
    'sink.enable-2pc'                    = 'true',
    'sink.label-prefix'                  = 'flink_cdc_products'
);

-- 🔄 Jalankan Migrasi CDC (berjalan terus)
INSERT INTO doris_products_cdc
SELECT * FROM cdc_products;
```

### Langkah 6: Tes CDC Real-Time

Setelah job CDC berjalan, coba INSERT/UPDATE/DELETE data di MariaDB dan lihat perubahannya langsung masuk ke Doris:

```sql
-- ========================================
-- Di MariaDB (localhost:3306) - Tes INSERT
-- ========================================
USE demo_db;

INSERT INTO employees (name, email, phone, department, position, salary, hire_date, address)
VALUES ('Test CDC User', 'test.cdc@email.com', '081299999999', 'Engineering', 'Tester', 10000000.00, '2026-02-11', 'Jl. Test No. 1, Jakarta');

-- ========================================
-- Di MariaDB - Tes UPDATE
-- ========================================
UPDATE employees SET salary = 12000000.00 WHERE name = 'Test CDC User';

-- ========================================
-- Di MariaDB - Tes DELETE
-- ========================================
DELETE FROM employees WHERE name = 'Test CDC User';
```

```sql
-- ========================================
-- Di Doris (192.168.4.100:19030) - Verifikasi
-- ========================================
-- Cek apakah perubahan langsung muncul di Doris
SELECT * FROM demo_db.employees ORDER BY id DESC LIMIT 5;
```

### Langkah 7: Monitor & Cancel Job CDC

```bash
# Cek running jobs via CLI
./bin/flink list --running

# Atau buka Web UI
# http://localhost:8081 → Running Jobs

# Cancel job (jika tidak butuh real-time CDC lagi)
./bin/flink cancel <job-id>

# Atau buat savepoint dulu sebelum stop (untuk resume nanti)
./bin/flink savepoint <job-id> /tmp/flink-savepoints/
./bin/flink cancel <job-id>
```

---

## Verifikasi Hasil Migrasi

### Connect ke Doris

```bash
mysql -h 192.168.4.100 -P 19030 -u root
```

### Cek Jumlah Data

```sql
USE demo_db;

-- Cek jumlah data (harus sama dengan MariaDB)
SELECT 'employees' AS tabel, COUNT(*) AS jumlah FROM employees
UNION ALL
SELECT 'orders', COUNT(*) FROM orders
UNION ALL
SELECT 'products', COUNT(*) FROM products;

-- Output yang diharapkan:
-- +-----------+--------+
-- | tabel     | jumlah |
-- +-----------+--------+
-- | employees |     20 |
-- | orders    |     20 |
-- | products  |     20 |
-- +-----------+--------+
```

### Lihat Sample Data

```sql
-- Lihat data employees
SELECT * FROM employees LIMIT 5;

-- Lihat data orders
SELECT * FROM orders LIMIT 5;

-- Lihat data products
SELECT * FROM products LIMIT 5;
```

### Contoh Query Analytics di Doris

```sql
-- 📊 Total gaji per department
SELECT department, COUNT(*) AS jumlah_karyawan, SUM(salary) AS total_gaji
FROM employees
GROUP BY department
ORDER BY total_gaji DESC;

-- 📊 Total revenue per status order
SELECT order_status, COUNT(*) AS jumlah_order, SUM(total_price) AS total_revenue
FROM orders
GROUP BY order_status
ORDER BY total_revenue DESC;

-- 📊 Produk dengan stok terbanyak per kategori
SELECT category, product_name, stock, price
FROM products
ORDER BY category, stock DESC;
```

---

## 📚 Referensi

- [Apache Flink 1.18 Documentation](https://nightlies.apache.org/flink/flink-docs-release-1.18/)
- [Flink JDBC Connector](https://nightlies.apache.org/flink/flink-docs-release-1.18/docs/connectors/table/jdbc/)
- [Flink CDC MySQL Connector](https://nightlies.apache.org/flink/flink-cdc-docs-release-3.0/docs/connectors/flink-sources/mysql-cdc/)
- [Flink Doris Connector](https://doris.apache.org/docs/ecosystem/flink-doris-connector)

---

> 📝 **Author**: Data Engineering Team
> 📅 **Last Updated**: 2026-02-11
> 🏷️ **Tags**: `apache-flink`, `jdbc`, `cdc`, `migrasi`, `mariadb`, `apache-doris`, `data-dummy`
