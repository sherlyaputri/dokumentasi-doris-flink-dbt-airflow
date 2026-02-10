# 🌊 Cara Instalasi Apache Airflow

> **Apache Airflow** adalah platform open-source untuk membuat, menjadwalkan, dan memonitoring workflow (DAG - Directed Acyclic Graph). Airflow sangat populer untuk orchestrating data pipelines, ETL jobs, dan automasi tugas-tugas data engineering.

---

## 📋 Table of Contents

- [Prerequisites](#prerequisites)
- [Arsitektur Apache Airflow](#arsitektur-apache-airflow)
- [Instalasi Apache Airflow](#instalasi-apache-airflow)
  - [Metode 1: Docker Compose (Recommended)](#metode-1-docker-compose-recommended)
  - [Metode 2: pip Install (Standalone)](#metode-2-pip-install-standalone)
- [Konfigurasi Airflow](#konfigurasi-airflow)
- [Airflow Web UI](#airflow-web-ui)
- [Membuat DAG Pertama](#membuat-dag-pertama)
- [Connections & Variables](#connections--variables)
- [Operators Penting](#operators-penting)
- [Monitoring & Logging](#monitoring--logging)
- [Troubleshooting](#troubleshooting)

---

## Prerequisites

| Komponen         | Minimum Requirement          |
|------------------|------------------------------|
| **OS**           | Ubuntu 20.04+ / macOS       |
| **CPU**          | 4 Cores                     |
| **RAM**          | 8 GB (16+ recommended)      |
| **Disk**         | 50 GB SSD                   |
| **Python**       | 3.8 - 3.11                  |
| **Docker**       | 20.10+ (untuk Docker method)|
| **Docker Compose** | v2.0+                     |

---

## Arsitektur Apache Airflow

```
┌──────────────────────────────────────────────────────────────────┐
│                     Apache Airflow Architecture                  │
│                                                                  │
│  ┌──────────────┐   ┌──────────────┐   ┌──────────────────────┐ │
│  │  Web Server  │   │  Scheduler   │   │  Executor            │ │
│  │              │   │              │   │                      │ │
│  │ • Flask App  │   │ • Parse DAGs │   │ • LocalExecutor      │ │
│  │ • REST API   │   │ • Schedule   │   │ • CeleryExecutor     │ │
│  │ • UI (:8080) │   │   Tasks      │   │ • KubernetesExecutor │ │
│  │ • Auth/RBAC  │   │ • Trigger    │   │                      │ │
│  └──────┬───────┘   └──────┬───────┘   └──────────┬───────────┘ │
│         │                  │                       │             │
│         ▼                  ▼                       ▼             │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │                   Metadata Database                        │ │
│  │              (PostgreSQL / MySQL / SQLite)                  │ │
│  │                                                            │ │
│  │  • DAG definitions    • Task instances                     │ │
│  │  • Run history        • Variables & Connections            │ │
│  │  • XCom data          • Logs metadata                      │ │
│  └─────────────────────────────────────────────────────────────┘ │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │                     DAGs Directory                          │ │
│  │                    /opt/airflow/dags/                       │ │
│  │                                                            │ │
│  │  📄 dag_etl_daily.py     📄 dag_dbt_run.py                │ │
│  │  📄 dag_kafka_monitor.py 📄 dag_data_quality.py           │ │
│  └─────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────┘
```

---

## Instalasi Apache Airflow

### Metode 1: Docker Compose (Recommended)

```bash
# 📁 Buat project directory
mkdir -p ~/airflow-project && cd ~/airflow-project

# Download official docker-compose dari Apache Airflow
curl -LfO 'https://airflow.apache.org/docs/apache-airflow/2.8.1/docker-compose.yaml'

# Buat folder yang diperlukan
mkdir -p ./dags ./logs ./plugins ./config

# Set Airflow UID (penting untuk Linux)
echo -e "AIRFLOW_UID=$(id -u)" > .env
```

Atau buat custom `docker-compose.yml`:

```yaml
# docker-compose.yml - Apache Airflow Custom Setup
version: "3.8"

x-airflow-common: &airflow-common
  image: apache/airflow:2.8.1-python3.11
  environment: &airflow-common-env
    AIRFLOW__CORE__EXECUTOR: LocalExecutor
    AIRFLOW__DATABASE__SQL_ALCHEMY_CONN: postgresql+psycopg2://airflow:airflow@postgres/airflow
    AIRFLOW__CORE__FERNET_KEY: ''
    AIRFLOW__CORE__DAGS_ARE_PAUSED_AT_CREATION: 'true'
    AIRFLOW__CORE__LOAD_EXAMPLES: 'false'
    AIRFLOW__CORE__DEFAULT_TIMEZONE: 'Asia/Jakarta'
    AIRFLOW__WEBSERVER__DEFAULT_UI_TIMEZONE: 'Asia/Jakarta'
    AIRFLOW__API__AUTH_BACKENDS: 'airflow.api.auth.backend.basic_auth,airflow.api.auth.backend.session'
    AIRFLOW__SCHEDULER__ENABLE_HEALTH_CHECK: 'true'
    AIRFLOW__SCHEDULER__DAG_DIR_LIST_INTERVAL: 30
    AIRFLOW__CORE__PARALLELISM: 32
    AIRFLOW__CORE__MAX_ACTIVE_TASKS_PER_DAG: 16
    AIRFLOW__CORE__MAX_ACTIVE_RUNS_PER_DAG: 3
    # Email Configuration (opsional)
    AIRFLOW__SMTP__SMTP_HOST: 'smtp.gmail.com'
    AIRFLOW__SMTP__SMTP_PORT: 587
    AIRFLOW__SMTP__SMTP_STARTTLS: 'true'
    AIRFLOW__SMTP__SMTP_SSL: 'false'
    AIRFLOW__SMTP__SMTP_USER: 'your-email@gmail.com'
    AIRFLOW__SMTP__SMTP_PASSWORD: 'your-app-password'
    AIRFLOW__SMTP__SMTP_MAIL_FROM: 'airflow@example.com'
    # Tambahan Python packages
    _PIP_ADDITIONAL_REQUIREMENTS: >-
      apache-airflow-providers-apache-flink
      apache-airflow-providers-mysql
      apache-airflow-providers-postgres
      apache-airflow-providers-http
      apache-airflow-providers-ssh
      dbt-core
      dbt-doris
      requests
      pandas
  volumes:
    - ./dags:/opt/airflow/dags
    - ./logs:/opt/airflow/logs
    - ./plugins:/opt/airflow/plugins
    - ./config:/opt/airflow/config
    - ./data:/opt/airflow/data
    - ./dbt_project:/opt/airflow/dbt_project
  user: "${AIRFLOW_UID:-50000}:0"
  networks:
    - airflow-network
  depends_on: &airflow-common-depends-on
    postgres:
      condition: service_healthy

services:
  # ===========================
  # PostgreSQL (Metadata DB)
  # ===========================
  postgres:
    image: postgres:15
    container_name: airflow-postgres
    environment:
      POSTGRES_USER: airflow
      POSTGRES_PASSWORD: airflow
      POSTGRES_DB: airflow
    volumes:
      - postgres-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD", "pg_isready", "-U", "airflow"]
      interval: 10s
      retries: 5
      start_period: 5s
    networks:
      - airflow-network
    restart: unless-stopped

  # ===========================
  # Airflow Init (First Time)
  # ===========================
  airflow-init:
    <<: *airflow-common
    container_name: airflow-init
    entrypoint: /bin/bash
    command:
      - -c
      - |
        mkdir -p /sources/logs /sources/dags /sources/plugins
        exec /entrypoint airflow version
    environment:
      <<: *airflow-common-env
      _AIRFLOW_DB_MIGRATE: 'true'
      _AIRFLOW_WWW_USER_CREATE: 'true'
      _AIRFLOW_WWW_USER_USERNAME: ${_AIRFLOW_WWW_USER_USERNAME:-admin}
      _AIRFLOW_WWW_USER_PASSWORD: ${_AIRFLOW_WWW_USER_PASSWORD:-admin}
      _AIRFLOW_WWW_USER_FIRSTNAME: Admin
      _AIRFLOW_WWW_USER_LASTNAME: User
      _AIRFLOW_WWW_USER_EMAIL: admin@example.com
      _AIRFLOW_WWW_USER_ROLE: Admin
    user: "0:0"

  # ===========================
  # Airflow Webserver
  # ===========================
  airflow-webserver:
    <<: *airflow-common
    container_name: airflow-webserver
    command: webserver
    ports:
      - "8080:8080"
    healthcheck:
      test: ["CMD", "curl", "--fail", "http://localhost:8080/health"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 30s
    restart: unless-stopped

  # ===========================
  # Airflow Scheduler
  # ===========================
  airflow-scheduler:
    <<: *airflow-common
    container_name: airflow-scheduler
    command: scheduler
    healthcheck:
      test: ["CMD", "airflow", "jobs", "check", "--job-type", "SchedulerJob", "--hostname", "$${HOSTNAME}"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 30s
    restart: unless-stopped

  # ===========================
  # Airflow Triggerer
  # ===========================
  airflow-triggerer:
    <<: *airflow-common
    container_name: airflow-triggerer
    command: triggerer
    healthcheck:
      test: ["CMD-SHELL", 'airflow jobs check --job-type TriggererJob --hostname "$${HOSTNAME}"']
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 30s
    restart: unless-stopped

  # ===========================
  # Airflow CLI (helper)
  # ===========================
  airflow-cli:
    <<: *airflow-common
    container_name: airflow-cli
    profiles:
      - debug
    environment:
      <<: *airflow-common-env
      CONNECTION_CHECK_MAX_COUNT: "0"
    command:
      - bash
      - -c
      - airflow

volumes:
  postgres-data:

networks:
  airflow-network:
    driver: bridge
```

```bash
# 🚀 Inisialisasi Airflow (pertama kali saja)
docker compose up airflow-init

# Jalankan Airflow
docker compose up -d

# Cek status
docker compose ps

# 🌐 Akses Web UI
# URL: http://localhost:8080
# Username: admin
# Password: admin

# Cek logs
docker compose logs -f airflow-scheduler
docker compose logs -f airflow-webserver
```

### Metode 2: pip Install (Standalone)

```bash
# Buat virtual environment
python3 -m venv ~/airflow-env
source ~/airflow-env/bin/activate

# Set AIRFLOW_HOME
export AIRFLOW_HOME=~/airflow
echo 'export AIRFLOW_HOME=~/airflow' >> ~/.bashrc

# Install Airflow (PENTING: gunakan constraint file)
AIRFLOW_VERSION=2.8.1
PYTHON_VERSION="$(python3 --version | cut -d " " -f 2 | cut -d "." -f 1-2)"
CONSTRAINT_URL="https://raw.githubusercontent.com/apache/airflow/constraints-${AIRFLOW_VERSION}/constraints-${PYTHON_VERSION}.txt"

pip install "apache-airflow==${AIRFLOW_VERSION}" --constraint "${CONSTRAINT_URL}"

# Install providers
pip install apache-airflow-providers-postgres \
            apache-airflow-providers-mysql \
            apache-airflow-providers-http \
            apache-airflow-providers-ssh

# Inisialisasi database
airflow db migrate

# Buat admin user
airflow users create \
    --username admin \
    --firstname Admin \
    --lastname User \
    --role Admin \
    --email admin@example.com \
    --password admin

# Start Airflow (2 terminal berbeda)
# Terminal 1: Webserver
airflow webserver --port 8080

# Terminal 2: Scheduler
airflow scheduler
```

---

## Konfigurasi Airflow

### airflow.cfg (Key Settings)

```ini
# ~/airflow/airflow.cfg atau via environment variables

[core]
# Executor type
executor = LocalExecutor

# DAGs folder
dags_folder = /opt/airflow/dags

# Parallelism
parallelism = 32
max_active_tasks_per_dag = 16
max_active_runs_per_dag = 3

# Default timezone
default_timezone = Asia/Jakarta

# Don't load example DAGs
load_examples = False

[database]
# Connection string (PostgreSQL recommended untuk production)
sql_alchemy_conn = postgresql+psycopg2://airflow:airflow@localhost:5432/airflow

[webserver]
# Web UI settings
web_server_port = 8080
default_ui_timezone = Asia/Jakarta
expose_config = True
rbac = True

# Session timeout
session_lifetime_minutes = 1440

[scheduler]
# How often to scan DAGs folder
dag_dir_list_interval = 30

# Min interval antara DAG runs
min_file_process_interval = 30

# Parsing timeout
dagbag_import_timeout = 120

[logging]
# Log format
log_format = [%%(asctime)s] {%%(filename)s:%%(lineno)d} %%(levelname)s - %%(message)s

# Remote logging (S3/GCS)
# remote_logging = True
# remote_base_log_folder = s3://my-bucket/airflow/logs
```

---

## Airflow Web UI

```
🌐 http://localhost:8080
👤 Username: admin
🔑 Password: admin
```

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

### Hello World DAG

```python
# dags/01_hello_world.py

from datetime import datetime, timedelta
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.operators.bash import BashOperator
from airflow.operators.empty import EmptyOperator

# ============================================
# 📋 Default Arguments
# ============================================
default_args = {
    'owner': 'data-engineering',
    'depends_on_past': False,
    'email': ['alert@example.com'],
    'email_on_failure': True,
    'email_on_retry': False,
    'retries': 2,
    'retry_delay': timedelta(minutes=5),
    'execution_timeout': timedelta(hours=1),
}

# ============================================
# 🔄 DAG Definition
# ============================================
with DAG(
    dag_id='01_hello_world',
    default_args=default_args,
    description='DAG pertama - Hello World example',
    schedule_interval='@daily',  # Cron: '0 0 * * *'
    start_date=datetime(2026, 1, 1),
    end_date=None,
    catchup=False,
    tags=['example', 'hello-world'],
    max_active_runs=1,
    doc_md="""
    ## Hello World DAG
    
    DAG sederhana untuk demonstrasi Apache Airflow.
    
    ### Tasks:
    1. **start** - Empty operator (marker)
    2. **print_date** - Print tanggal hari ini
    3. **hello_python** - Python function
    4. **hello_bash** - Bash command
    5. **end** - Empty operator (marker)
    """,
) as dag:

    # Task 1: Start marker
    start = EmptyOperator(
        task_id='start',
    )

    # Task 2: Bash operator
    print_date = BashOperator(
        task_id='print_date',
        bash_command='echo "Execution date: {{ ds }} | Logical date: {{ logical_date }}"',
    )

    # Task 3: Python operator
    def greet(name, **context):
        """Python function yang dipanggil oleh PythonOperator"""
        execution_date = context['ds']
        task_instance = context['ti']
        
        message = f"Hello {name}! Today is {execution_date}"
        print(message)
        
        # Push data ke XCom (cross-task communication)
        task_instance.xcom_push(key='greeting', value=message)
        return message

    hello_python = PythonOperator(
        task_id='hello_python',
        python_callable=greet,
        op_kwargs={'name': 'Data Engineer'},
    )

    # Task 4: Bash operator
    hello_bash = BashOperator(
        task_id='hello_bash',
        bash_command='''
            echo "===================="
            echo "🚀 Airflow is running!"
            echo "Host: $(hostname)"
            echo "Date: $(date)"
            echo "Python: $(python3 --version)"
            echo "===================="
        ''',
    )

    # Task 5: Read XCom
    def read_xcom(**context):
        ti = context['ti']
        greeting = ti.xcom_pull(task_ids='hello_python', key='greeting')
        print(f"📨 Received from XCom: {greeting}")

    read_result = PythonOperator(
        task_id='read_xcom_result',
        python_callable=read_xcom,
    )

    # Task 6: End marker
    end = EmptyOperator(
        task_id='end',
        trigger_rule='all_success',
    )

    # ============================================
    # 📊 Task Dependencies (DAG Flow)
    # ============================================
    start >> [print_date, hello_python, hello_bash]
    hello_python >> read_result
    [print_date, read_result, hello_bash] >> end
```

### ETL Pipeline DAG

```python
# dags/02_etl_pipeline.py

from datetime import datetime, timedelta
from airflow import DAG
from airflow.operators.python import PythonOperator, BranchPythonOperator
from airflow.operators.bash import BashOperator
from airflow.operators.empty import EmptyOperator
from airflow.providers.mysql.operators.mysql import MySqlOperator
from airflow.providers.http.sensors.http import HttpSensor
from airflow.utils.trigger_rule import TriggerRule
import json
import requests

default_args = {
    'owner': 'data-engineering',
    'depends_on_past': False,
    'retries': 3,
    'retry_delay': timedelta(minutes=5),
}

with DAG(
    dag_id='02_etl_pipeline',
    default_args=default_args,
    description='ETL Pipeline - Extract, Transform, Load ke Doris',
    schedule_interval='0 2 * * *',  # Setiap jam 2 pagi
    start_date=datetime(2026, 1, 1),
    catchup=False,
    tags=['etl', 'production', 'daily'],
    max_active_runs=1,
) as dag:

    # ========================================
    # 📡 EXTRACT - Ambil data dari API
    # ========================================
    
    # Sensor: cek apakah API available
    check_api = HttpSensor(
        task_id='check_api_available',
        http_conn_id='api_source',
        endpoint='/health',
        request_params={},
        response_check=lambda response: response.status_code == 200,
        poke_interval=30,
        timeout=300,
        mode='poke',
    )

    def extract_data(**context):
        """Extract data dari source API"""
        import requests
        
        execution_date = context['ds']
        print(f"📥 Extracting data for date: {execution_date}")
        
        # Simulasi extract dari API
        response = requests.get(
            'https://api.example.com/data',
            params={'date': execution_date},
            timeout=60
        )
        
        if response.status_code == 200:
            data = response.json()
            # Simpan ke XCom atau file
            context['ti'].xcom_push(key='raw_data', value=data)
            context['ti'].xcom_push(key='record_count', value=len(data))
            print(f"✅ Extracted {len(data)} records")
        else:
            raise Exception(f"API Error: {response.status_code}")

    extract = PythonOperator(
        task_id='extract_data',
        python_callable=extract_data,
    )

    # ========================================
    # 🔄 TRANSFORM - Bersihkan & transformasi data
    # ========================================
    
    def transform_data(**context):
        """Transform dan clean data"""
        ti = context['ti']
        raw_data = ti.xcom_pull(task_ids='extract_data', key='raw_data')
        
        print(f"🔄 Transforming {len(raw_data)} records...")
        
        transformed = []
        for record in raw_data:
            transformed.append({
                'id': record['id'],
                'name': record.get('name', '').strip().title(),
                'email': record.get('email', '').lower(),
                'amount': round(float(record.get('amount', 0)), 2),
                'status': record.get('status', 'unknown').lower(),
                'processed_at': datetime.now().isoformat(),
            })
        
        # Filter invalid records
        valid_records = [r for r in transformed if r['amount'] > 0]
        invalid_count = len(transformed) - len(valid_records)
        
        ti.xcom_push(key='transformed_data', value=valid_records)
        ti.xcom_push(key='invalid_count', value=invalid_count)
        
        print(f"✅ Valid: {len(valid_records)}, Invalid: {invalid_count}")

    transform = PythonOperator(
        task_id='transform_data',
        python_callable=transform_data,
    )

    # ========================================
    # 🔀 BRANCH - Cek apakah ada data
    # ========================================
    
    def check_data_quality(**context):
        """Branch logic: cek apakah data cukup untuk di-load"""
        ti = context['ti']
        record_count = ti.xcom_pull(task_ids='extract_data', key='record_count')
        invalid_count = ti.xcom_pull(task_ids='transform_data', key='invalid_count')
        
        error_rate = invalid_count / record_count if record_count > 0 else 1
        
        if record_count == 0:
            return 'no_data_alert'
        elif error_rate > 0.5:
            return 'high_error_alert'
        else:
            return 'load_to_doris'

    branch = BranchPythonOperator(
        task_id='check_data_quality',
        python_callable=check_data_quality,
    )

    # ========================================
    # 📤 LOAD - Masukkan ke Apache Doris
    # ========================================
    
    def load_to_doris(**context):
        """Load transformed data ke Apache Doris via Stream Load"""
        import requests
        import json
        
        ti = context['ti']
        data = ti.xcom_pull(task_ids='transform_data', key='transformed_data')
        
        print(f"📤 Loading {len(data)} records to Doris...")
        
        # Stream Load ke Doris
        url = 'http://doris-fe:8030/api/demo_db/processed_data/_stream_load'
        headers = {
            'label': f'airflow_load_{context["ds_nodash"]}_{context["ts_nodash"]}',
            'format': 'json',
            'strip_outer_array': 'true',
            'Content-Type': 'application/json',
        }
        
        response = requests.put(
            url,
            headers=headers,
            data=json.dumps(data),
            auth=('root', ''),
            timeout=300
        )
        
        result = response.json()
        if result.get('Status') == 'Success':
            print(f"✅ Loaded successfully: {result}")
        else:
            raise Exception(f"❌ Load failed: {result}")

    load = PythonOperator(
        task_id='load_to_doris',
        python_callable=load_to_doris,
    )

    # Alert tasks
    no_data = PythonOperator(
        task_id='no_data_alert',
        python_callable=lambda: print("⚠️ ALERT: No data extracted!"),
    )

    high_error = PythonOperator(
        task_id='high_error_alert',
        python_callable=lambda: print("🚨 ALERT: High error rate detected!"),
    )

    # ========================================
    # ✅ END
    # ========================================
    end = EmptyOperator(
        task_id='end',
        trigger_rule=TriggerRule.NONE_FAILED_MIN_ONE_SUCCESS,
    )

    # ========================================
    # 📊 DAG Flow
    # ========================================
    check_api >> extract >> transform >> branch
    branch >> [load, no_data, high_error]
    [load, no_data, high_error] >> end
```

---

## Connections & Variables

### Buat Connections via CLI

```bash
# Apache Doris connection
docker exec airflow-webserver airflow connections add 'doris_default' \
    --conn-type 'mysql' \
    --conn-host 'doris-fe' \
    --conn-port '9030' \
    --conn-login 'root' \
    --conn-password '' \
    --conn-schema 'demo_db'

# PostgreSQL connection
docker exec airflow-webserver airflow connections add 'postgres_source' \
    --conn-type 'postgres' \
    --conn-host 'postgres-host' \
    --conn-port '5432' \
    --conn-login 'airflow' \
    --conn-password 'airflow' \
    --conn-schema 'source_db'

# HTTP API connection
docker exec airflow-webserver airflow connections add 'api_source' \
    --conn-type 'http' \
    --conn-host 'https://api.example.com' \
    --conn-extra '{"Authorization": "Bearer token123"}'

# SSH connection
docker exec airflow-webserver airflow connections add 'ssh_server' \
    --conn-type 'ssh' \
    --conn-host '192.168.1.100' \
    --conn-port '22' \
    --conn-login 'ubuntu' \
    --conn-extra '{"key_file": "/opt/airflow/keys/id_rsa"}'

# List connections
docker exec airflow-webserver airflow connections list
```

### Buat Variables

```bash
# Set variables via CLI
docker exec airflow-webserver airflow variables set 'env' 'production'
docker exec airflow-webserver airflow variables set 'doris_fe_host' 'doris-fe'
docker exec airflow-webserver airflow variables set 'slack_webhook' 'https://hooks.slack.com/xxx'
docker exec airflow-webserver airflow variables set 'email_recipients' '["admin@example.com","dev@example.com"]'

# Set JSON variable
docker exec airflow-webserver airflow variables set 'db_config' \
    '{"host": "doris-fe", "port": 9030, "database": "demo_db"}'

# Get variable
docker exec airflow-webserver airflow variables get 'env'

# Pakai di DAG
from airflow.models import Variable

env = Variable.get("env")
db_config = Variable.get("db_config", deserialize_json=True)
recipients = Variable.get("email_recipients", deserialize_json=True)
```

---

## Operators Penting

### Cheat Sheet

```python
from airflow.operators.python import PythonOperator       # Run Python function
from airflow.operators.bash import BashOperator             # Run bash command
from airflow.operators.empty import EmptyOperator           # Placeholder / marker
from airflow.operators.python import BranchPythonOperator   # Conditional branching
from airflow.operators.trigger_dagrun import TriggerDagRunOperator  # Trigger DAG lain
from airflow.operators.email import EmailOperator           # Send email

# Sensors (menunggu kondisi terpenuhi)
from airflow.sensors.filesystem import FileSensor           # Tunggu file ada
from airflow.providers.http.sensors.http import HttpSensor  # Tunggu API ready
from airflow.sensors.external_task import ExternalTaskSensor  # Tunggu task DAG lain

# Database Operators
from airflow.providers.mysql.operators.mysql import MySqlOperator
from airflow.providers.postgres.operators.postgres import PostgresOperator

# Hooks (koneksi ke external systems)
from airflow.providers.mysql.hooks.mysql import MySqlHook
from airflow.providers.postgres.hooks.postgres import PostgresHook
from airflow.providers.http.hooks.http import HttpHook

# Task Groups (sub-DAG replacement)
from airflow.utils.task_group import TaskGroup

# Decorators (Taskflow API - modern style)
from airflow.decorators import dag, task
```

### TaskFlow API (Modern Style)

```python
# dags/03_taskflow_example.py

from datetime import datetime
from airflow.decorators import dag, task

@dag(
    dag_id='03_taskflow_api',
    schedule_interval='@daily',
    start_date=datetime(2026, 1, 1),
    catchup=False,
    tags=['example', 'taskflow'],
)
def taskflow_etl():
    """ETL menggunakan TaskFlow API (modern & clean)"""

    @task()
    def extract():
        """Extract data"""
        data = {
            'users': [
                {'id': 1, 'name': 'Budi', 'spend': 5000000},
                {'id': 2, 'name': 'Siti', 'spend': 12000000},
                {'id': 3, 'name': 'Andi', 'spend': 800000},
            ]
        }
        return data  # Otomatis ke XCom

    @task()
    def transform(raw_data: dict):
        """Transform data"""
        users = raw_data['users']
        for user in users:
            user['segment'] = 'VIP' if user['spend'] >= 10000000 else 'Regular'
            user['spend_formatted'] = f"Rp {user['spend']:,}"
        return users

    @task()
    def load(transformed_data: list):
        """Load data"""
        for user in transformed_data:
            print(f"📤 Loading: {user['name']} ({user['segment']}) - {user['spend_formatted']}")
        print(f"✅ Loaded {len(transformed_data)} records")

    # DAG Flow (sangat clean!)
    raw = extract()
    transformed = transform(raw)
    load(transformed)

# Instantiate DAG
taskflow_etl()
```

---

## Monitoring & Logging

### Health Check

```bash
# Cek health via API
curl http://localhost:8080/health
# Response: {"metadatabase":{"status":"healthy"},"scheduler":{"status":"healthy",...}}

# Cek DAG status
docker exec airflow-webserver airflow dags list

# Trigger DAG manual
docker exec airflow-webserver airflow dags trigger 01_hello_world

# Cek task status
docker exec airflow-webserver airflow tasks list 01_hello_world
docker exec airflow-webserver airflow tasks test 01_hello_world hello_python 2026-02-10

# Cek failed tasks
docker exec airflow-webserver airflow tasks states-for-dag-run 02_etl_pipeline 2026-02-10T02:00:00+00:00
```

### Airflow REST API

```bash
# List all DAGs
curl -u admin:admin http://localhost:8080/api/v1/dags | python3 -m json.tool

# Get specific DAG
curl -u admin:admin http://localhost:8080/api/v1/dags/01_hello_world | python3 -m json.tool

# Trigger DAG via API
curl -X POST -u admin:admin \
  -H "Content-Type: application/json" \
  -d '{"conf": {"key": "value"}}' \
  http://localhost:8080/api/v1/dags/01_hello_world/dagRuns

# Get DAG runs
curl -u admin:admin \
  http://localhost:8080/api/v1/dags/01_hello_world/dagRuns | python3 -m json.tool

# Pause/Unpause DAG
curl -X PATCH -u admin:admin \
  -H "Content-Type: application/json" \
  -d '{"is_paused": true}' \
  http://localhost:8080/api/v1/dags/01_hello_world
```

---

## Troubleshooting

| Problem | Solusi |
|---------|--------|
| `DAG not appearing` | Cek syntax error: `docker exec airflow-webserver python /opt/airflow/dags/my_dag.py` |
| `Broken DAG` | Cek import error di Web UI > DAGs atau `airflow dags list-import-errors` |
| `Task stuck in queued` | Cek executor, pastikan scheduler running |
| `Permission denied` | Set `AIRFLOW_UID=$(id -u)` di `.env` |
| `Connection refused DB` | Cek PostgreSQL container: `docker compose logs postgres` |
| `No module named 'xxx'` | Tambah di `_PIP_ADDITIONAL_REQUIREMENTS` atau rebuild image |
| `XCom too large` | Gunakan file atau external storage, bukan XCom untuk data besar |

```bash
# Debug DAG parsing
docker exec airflow-webserver python -c "
from airflow.models import DagBag
bag = DagBag()
if bag.import_errors:
    for path, error in bag.import_errors.items():
        print(f'❌ {path}: {error}')
else:
    print('✅ All DAGs parsed successfully')
    for dag_id in bag.dags:
        print(f'  📄 {dag_id}')
"
```

---

## 📚 Referensi

- [Apache Airflow Documentation](https://airflow.apache.org/docs/)
- [Airflow Docker Setup](https://airflow.apache.org/docs/apache-airflow/stable/howto/docker-compose/index.html)
- [Airflow Best Practices](https://airflow.apache.org/docs/apache-airflow/stable/best-practices.html)
- [Airflow Providers](https://airflow.apache.org/docs/apache-airflow-providers/packages-ref.html)
- [Astronomer Guides](https://docs.astronomer.io/learn/)

---

> 📝 **Author**: Data Engineering Team  
> 📅 **Last Updated**: 2026-02-10  
> 🏷️ **Tags**: `airflow`, `orchestration`, `dag`, `scheduling`, `data-pipeline`
