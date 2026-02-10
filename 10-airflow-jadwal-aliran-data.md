# ⏰ Cara Menggunakan Apache Airflow untuk Mengatur Jadwal dan Aliran Data

> Panduan lengkap menggunakan **Apache Airflow** sebagai orchestrator untuk mengatur jadwal dan aliran data pipeline — mulai dari CDC monitoring, DBT transformations, data quality checks, hingga alerting.

---

## 📋 Table of Contents

- [Arsitektur Orchestration](#arsitektur-orchestration)
- [DAG 1: DBT Transformation Pipeline](#dag-1-dbt-transformation-pipeline)
- [DAG 2: Data Quality Monitoring](#dag-2-data-quality-monitoring)
- [DAG 3: CDC Health Check](#dag-3-cdc-health-check)
- [DAG 4: Full ELT Pipeline (End-to-End)](#dag-4-full-elt-pipeline-end-to-end)
- [DAG 5: Scheduled Report Generator](#dag-5-scheduled-report-generator)
- [Scheduling Patterns](#scheduling-patterns)
- [Alerting & Notification](#alerting--notification)
- [Best Practices](#best-practices)
- [Troubleshooting](#troubleshooting)

---

## Arsitektur Orchestration

```
┌──────────────────────────────────────────────────────────────┐
│              Apache Airflow - Orchestration Layer            │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │                  DAG Scheduler                       │    │
│  │                                                      │    │
│  │  🕐 02:00 ──▶ dag_cdc_health_check (hourly)         │    │
│  │  🕐 03:00 ──▶ dag_dbt_transform (daily)             │    │
│  │  🕐 04:00 ──▶ dag_data_quality (daily)              │    │
│  │  🕐 06:00 ──▶ dag_full_elt_pipeline (daily)         │    │
│  │  🕐 08:00 ──▶ dag_report_generator (daily)          │    │
│  └─────────────────────────────────────────────────────┘    │
│       │         │         │         │         │              │
│       ▼         ▼         ▼         ▼         ▼              │
│  ┌────────┐ ┌───────┐ ┌───────┐ ┌───────┐ ┌────────────┐   │
│  │ Flink  │ │  DBT  │ │ Doris │ │ Kafka │ │ Slack/Email │   │
│  │ Jobs   │ │ Run   │ │ Query │ │ Check │ │ Alert       │   │
│  └────────┘ └───────┘ └───────┘ └───────┘ └────────────┘   │
└──────────────────────────────────────────────────────────────┘
```

---

## DAG 1: DBT Transformation Pipeline

```python
# dags/dag_dbt_transform.py

from datetime import datetime, timedelta
from airflow import DAG
from airflow.operators.bash import BashOperator
from airflow.operators.python import PythonOperator
from airflow.operators.empty import EmptyOperator
from airflow.utils.task_group import TaskGroup
from airflow.utils.trigger_rule import TriggerRule

# ============================================
# 📋 Config
# ============================================
DBT_PROJECT_DIR = '/opt/airflow/dbt_project'
DBT_PROFILES_DIR = '/opt/airflow/.dbt'

default_args = {
    'owner': 'data-engineering',
    'depends_on_past': False,
    'retries': 2,
    'retry_delay': timedelta(minutes=5),
    'email': ['data-team@example.com'],
    'email_on_failure': True,
}

with DAG(
    dag_id='dag_dbt_transform',
    default_args=default_args,
    description='DBT Transformation Pipeline - ODS → DWD → DWS',
    schedule_interval='0 3 * * *',  # Setiap hari jam 3 pagi
    start_date=datetime(2026, 1, 1),
    catchup=False,
    tags=['dbt', 'transformation', 'daily'],
    max_active_runs=1,
) as dag:

    start = EmptyOperator(task_id='start')

    # ========================================
    # 🔍 Pre-check: pastikan sumber data ready
    # ========================================
    check_doris = BashOperator(
        task_id='check_doris_connection',
        bash_command='''
            mysql -h doris-fe -P 9030 -u root -e "
                SELECT COUNT(*) as total FROM ods_ecommerce.ods_users;
                SELECT COUNT(*) as total FROM ods_ecommerce.ods_orders;
            " && echo "✅ Doris is ready"
        ''',
    )

    # ========================================
    # 📦 DBT: Install dependencies
    # ========================================
    dbt_deps = BashOperator(
        task_id='dbt_deps',
        bash_command=f'cd {DBT_PROJECT_DIR} && dbt deps --profiles-dir {DBT_PROFILES_DIR}',
    )

    # ========================================
    # 🏗️ DBT: Run Staging Layer
    # ========================================
    with TaskGroup(group_id='staging_layer') as staging:
        stg_users = BashOperator(
            task_id='stg_users',
            bash_command=f'''cd {DBT_PROJECT_DIR} && \
                dbt run --select stg_users --profiles-dir {DBT_PROFILES_DIR}''',
        )
        stg_orders = BashOperator(
            task_id='stg_orders',
            bash_command=f'''cd {DBT_PROJECT_DIR} && \
                dbt run --select stg_orders --profiles-dir {DBT_PROFILES_DIR}''',
        )
        stg_products = BashOperator(
            task_id='stg_products',
            bash_command=f'''cd {DBT_PROJECT_DIR} && \
                dbt run --select stg_products --profiles-dir {DBT_PROFILES_DIR}''',
        )

    # ========================================
    # 🧪 DBT: Test Staging
    # ========================================
    test_staging = BashOperator(
        task_id='test_staging',
        bash_command=f'''cd {DBT_PROJECT_DIR} && \
            dbt test --select staging.* --profiles-dir {DBT_PROFILES_DIR}''',
    )

    # ========================================
    # 🏗️ DBT: Run Marts Layer
    # ========================================
    with TaskGroup(group_id='marts_layer') as marts:
        dim_users = BashOperator(
            task_id='dim_users',
            bash_command=f'''cd {DBT_PROJECT_DIR} && \
                dbt run --select dim_users --profiles-dir {DBT_PROFILES_DIR}''',
        )
        fct_revenue = BashOperator(
            task_id='fct_daily_revenue',
            bash_command=f'''cd {DBT_PROJECT_DIR} && \
                dbt run --select fct_daily_revenue --profiles-dir {DBT_PROFILES_DIR}''',
        )
        fct_products = BashOperator(
            task_id='fct_product_performance',
            bash_command=f'''cd {DBT_PROJECT_DIR} && \
                dbt run --select fct_product_performance --profiles-dir {DBT_PROFILES_DIR}''',
        )

    # ========================================
    # 🧪 DBT: Test Marts
    # ========================================
    test_marts = BashOperator(
        task_id='test_marts',
        bash_command=f'''cd {DBT_PROJECT_DIR} && \
            dbt test --select marts.* --profiles-dir {DBT_PROFILES_DIR}''',
    )

    # ========================================
    # 📄 DBT: Generate Docs
    # ========================================
    dbt_docs = BashOperator(
        task_id='dbt_generate_docs',
        bash_command=f'''cd {DBT_PROJECT_DIR} && \
            dbt docs generate --profiles-dir {DBT_PROFILES_DIR}''',
    )

    # ========================================
    # 📊 Log summary
    # ========================================
    def log_summary(**context):
        print("=" * 50)
        print("📊 DBT Transform Pipeline Complete!")
        print(f"📅 Execution Date: {context['ds']}")
        print(f"⏰ Completed at: {datetime.now()}")
        print("=" * 50)

    summary = PythonOperator(
        task_id='log_summary',
        python_callable=log_summary,
        trigger_rule=TriggerRule.ALL_SUCCESS,
    )

    end = EmptyOperator(task_id='end')

    # ========================================
    # 📊 DAG Flow
    # ========================================
    start >> check_doris >> dbt_deps >> staging >> test_staging
    test_staging >> marts >> test_marts >> dbt_docs >> summary >> end
```

---

## DAG 2: Data Quality Monitoring

```python
# dags/dag_data_quality.py

from datetime import datetime, timedelta
from airflow import DAG
from airflow.operators.python import PythonOperator, BranchPythonOperator
from airflow.operators.empty import EmptyOperator
from airflow.utils.trigger_rule import TriggerRule
import mysql.connector
import json

default_args = {
    'owner': 'data-engineering',
    'retries': 1,
    'retry_delay': timedelta(minutes=3),
}

def query_doris(sql):
    """Helper: execute query di Doris"""
    conn = mysql.connector.connect(
        host='doris-fe', port=9030, user='root', password=''
    )
    cursor = conn.cursor(dictionary=True)
    cursor.execute(sql)
    result = cursor.fetchall()
    conn.close()
    return result

with DAG(
    dag_id='dag_data_quality',
    default_args=default_args,
    description='Data Quality Checks pada Doris',
    schedule_interval='0 4 * * *',
    start_date=datetime(2026, 1, 1),
    catchup=False,
    tags=['quality', 'monitoring', 'daily'],
) as dag:

    # ========================================
    # 📊 Check 1: Row Count Validation
    # ========================================
    def check_row_counts(**context):
        """Validasi jumlah row antar layer"""
        checks = query_doris("""
            SELECT 'ods_users' AS tbl, COUNT(*) AS cnt FROM ods_ecommerce.ods_users
            UNION ALL
            SELECT 'dim_users', COUNT(*) FROM marts.dim_users
            UNION ALL
            SELECT 'ods_orders', COUNT(*) FROM ods_ecommerce.ods_orders
        """)
        
        results = {r['tbl']: r['cnt'] for r in checks}
        context['ti'].xcom_push(key='row_counts', value=results)
        
        print(f"📊 Row Counts: {json.dumps(results, indent=2)}")
        
        # Alert jika dim_users < ods_users
        if results.get('dim_users', 0) < results.get('ods_users', 0) * 0.9:
            raise Exception("⚠️ dim_users has significantly fewer rows than ods_users!")

    check_rows = PythonOperator(
        task_id='check_row_counts',
        python_callable=check_row_counts,
    )

    # ========================================
    # 📊 Check 2: Null Values
    # ========================================
    def check_null_values(**context):
        """Cek critical columns untuk NULL values"""
        null_checks = query_doris("""
            SELECT 
                'dim_users.user_id' AS col, 
                COUNT(*) AS null_count
            FROM marts.dim_users WHERE user_id IS NULL
            UNION ALL
            SELECT 'dim_users.email', COUNT(*) 
            FROM marts.dim_users WHERE email IS NULL
            UNION ALL
            SELECT 'fct_revenue.revenue_date', COUNT(*)
            FROM marts.fct_daily_revenue WHERE revenue_date IS NULL
        """)
        
        has_nulls = False
        for check in null_checks:
            if check['null_count'] > 0:
                print(f"❌ NULL found in {check['col']}: {check['null_count']} rows")
                has_nulls = True
            else:
                print(f"✅ {check['col']}: No NULLs")
        
        if has_nulls:
            raise Exception("Data quality check failed: NULL values detected!")

    check_nulls = PythonOperator(
        task_id='check_null_values',
        python_callable=check_null_values,
    )

    # ========================================
    # 📊 Check 3: Data Freshness
    # ========================================
    def check_freshness(**context):
        """Cek apakah data masih fresh (updated within 24 hours)"""
        freshness = query_doris("""
            SELECT 
                'ods_orders' AS tbl,
                MAX(updated_at) AS last_update,
                TIMESTAMPDIFF(HOUR, MAX(updated_at), NOW()) AS hours_since_update
            FROM ods_ecommerce.ods_orders
        """)
        
        for check in freshness:
            hours = check['hours_since_update'] or 999
            tbl = check['tbl']
            if hours > 24:
                print(f"⚠️ {tbl}: Data stale! Last update {hours}h ago")
            else:
                print(f"✅ {tbl}: Fresh (updated {hours}h ago)")

    check_fresh = PythonOperator(
        task_id='check_freshness',
        python_callable=check_freshness,
    )

    # ========================================
    # 📊 Summary
    # ========================================
    def quality_summary(**context):
        print("📋 Data Quality Check Complete!")
        print(f"📅 Date: {context['ds']}")

    summary = PythonOperator(
        task_id='quality_summary',
        python_callable=quality_summary,
        trigger_rule=TriggerRule.ALL_DONE,
    )

    [check_rows, check_nulls, check_fresh] >> summary
```

---

## DAG 3: CDC Health Check

```python
# dags/dag_cdc_health_check.py

from datetime import datetime, timedelta
from airflow import DAG
from airflow.operators.python import PythonOperator
import requests

default_args = {
    'owner': 'data-engineering',
    'retries': 1,
    'retry_delay': timedelta(minutes=2),
}

with DAG(
    dag_id='dag_cdc_health_check',
    default_args=default_args,
    description='Monitor CDC pipeline health (Flink + Kafka + Debezium)',
    schedule_interval='0 * * * *',  # Setiap jam
    start_date=datetime(2026, 1, 1),
    catchup=False,
    tags=['monitoring', 'cdc', 'hourly'],
) as dag:

    def check_flink_jobs(**context):
        """Cek status Flink jobs"""
        try:
            resp = requests.get('http://flink-jobmanager:8081/jobs/overview', timeout=10)
            jobs = resp.json().get('jobs', [])
            
            running = [j for j in jobs if j['state'] == 'RUNNING']
            failed = [j for j in jobs if j['state'] == 'FAILED']
            
            print(f"✅ Running jobs: {len(running)}")
            if failed:
                print(f"❌ Failed jobs: {len(failed)}")
                for f in failed:
                    print(f"   - {f['jid']}: {f.get('name', 'unknown')}")
                raise Exception(f"{len(failed)} Flink job(s) FAILED!")
        except requests.ConnectionError:
            raise Exception("❌ Cannot connect to Flink JobManager!")

    def check_kafka_connect(**context):
        """Cek status Debezium connectors"""
        try:
            resp = requests.get('http://kafka-connect:8083/connectors', timeout=10)
            connectors = resp.json()
            
            for name in connectors:
                status = requests.get(
                    f'http://kafka-connect:8083/connectors/{name}/status',
                    timeout=10
                ).json()
                
                connector_state = status['connector']['state']
                task_states = [t['state'] for t in status.get('tasks', [])]
                
                if connector_state != 'RUNNING' or 'FAILED' in task_states:
                    print(f"❌ {name}: connector={connector_state}, tasks={task_states}")
                    raise Exception(f"Connector {name} unhealthy!")
                else:
                    print(f"✅ {name}: RUNNING")
        except requests.ConnectionError:
            raise Exception("❌ Cannot connect to Kafka Connect!")

    def check_consumer_lag(**context):
        """Cek Kafka consumer lag"""
        import subprocess
        result = subprocess.run([
            'docker', 'exec', 'kafka',
            'kafka-consumer-groups',
            '--bootstrap-server', 'localhost:9092',
            '--describe', '--all-groups'
        ], capture_output=True, text=True, timeout=30)
        
        print(result.stdout)
        
        # Parse lag
        for line in result.stdout.strip().split('\n'):
            parts = line.split()
            if len(parts) >= 6 and parts[-1].isdigit():
                lag = int(parts[-1])
                if lag > 10000:
                    print(f"⚠️ High lag detected: {line}")

    check_flink = PythonOperator(task_id='check_flink_jobs', python_callable=check_flink_jobs)
    check_kafka = PythonOperator(task_id='check_kafka_connect', python_callable=check_kafka_connect)
    check_lag = PythonOperator(task_id='check_consumer_lag', python_callable=check_consumer_lag)

    [check_flink, check_kafka, check_lag]
```

---

## DAG 4: Full ELT Pipeline (End-to-End)

```python
# dags/dag_full_elt_pipeline.py

from datetime import datetime, timedelta
from airflow import DAG
from airflow.operators.bash import BashOperator
from airflow.operators.python import PythonOperator
from airflow.operators.trigger_dagrun import TriggerDagRunOperator
from airflow.sensors.external_task import ExternalTaskSensor
from airflow.utils.task_group import TaskGroup

default_args = {
    'owner': 'data-engineering',
    'retries': 2,
    'retry_delay': timedelta(minutes=5),
    'email_on_failure': True,
    'email': ['data-team@example.com'],
}

with DAG(
    dag_id='dag_full_elt_pipeline',
    default_args=default_args,
    description='Full ELT: CDC Check → DBT → Quality → Report',
    schedule_interval='0 6 * * *',
    start_date=datetime(2026, 1, 1),
    catchup=False,
    tags=['elt', 'production', 'daily'],
    max_active_runs=1,
) as dag:

    # Step 1: Trigger CDC health check first
    trigger_cdc_check = TriggerDagRunOperator(
        task_id='trigger_cdc_health_check',
        trigger_dag_id='dag_cdc_health_check',
        wait_for_completion=True,
        poke_interval=30,
    )

    # Step 2: Trigger DBT transform
    trigger_dbt = TriggerDagRunOperator(
        task_id='trigger_dbt_transform',
        trigger_dag_id='dag_dbt_transform',
        wait_for_completion=True,
        poke_interval=60,
    )

    # Step 3: Trigger data quality
    trigger_quality = TriggerDagRunOperator(
        task_id='trigger_data_quality',
        trigger_dag_id='dag_data_quality',
        wait_for_completion=True,
        poke_interval=30,
    )

    # Step 4: Generate metrics
    def generate_daily_metrics(**context):
        """Generate ringkasan harian"""
        import mysql.connector
        conn = mysql.connector.connect(host='doris-fe', port=9030, user='root')
        cursor = conn.cursor(dictionary=True)
        
        cursor.execute("""
            SELECT 
                revenue_date, gross_revenue, total_orders,
                unique_customers, revenue_growth_pct
            FROM marts.fct_daily_revenue
            ORDER BY revenue_date DESC LIMIT 1
        """)
        latest = cursor.fetchone()
        conn.close()
        
        report = f"""
📊 Daily Pipeline Report - {context['ds']}
{'='*50}
📅 Revenue Date: {latest['revenue_date']}
💰 Gross Revenue: Rp {latest['gross_revenue']:,.0f}
📦 Total Orders: {latest['total_orders']}
👥 Unique Customers: {latest['unique_customers']}
📈 Growth: {latest['revenue_growth_pct']}%
{'='*50}
        """
        print(report)
        context['ti'].xcom_push(key='daily_report', value=report)

    metrics = PythonOperator(
        task_id='generate_daily_metrics',
        python_callable=generate_daily_metrics,
    )

    trigger_cdc_check >> trigger_dbt >> trigger_quality >> metrics
```

---

## DAG 5: Scheduled Report Generator

```python
# dags/dag_report_generator.py

from datetime import datetime, timedelta
from airflow import DAG
from airflow.operators.python import PythonOperator

default_args = {
    'owner': 'data-engineering',
    'retries': 1,
}

with DAG(
    dag_id='dag_report_generator',
    default_args=default_args,
    description='Generate & kirim laporan harian',
    schedule_interval='0 8 * * *',  # Jam 8 pagi
    start_date=datetime(2026, 1, 1),
    catchup=False,
    tags=['report', 'daily'],
) as dag:

    def generate_and_send_report(**context):
        """Generate report dan kirim via webhook/email"""
        import mysql.connector
        import requests
        import json
        
        conn = mysql.connector.connect(host='doris-fe', port=9030, user='root')
        cursor = conn.cursor(dictionary=True)
        
        # Query metrics
        cursor.execute("""
            SELECT * FROM marts.fct_daily_revenue 
            ORDER BY revenue_date DESC LIMIT 7
        """)
        weekly_data = cursor.fetchall()
        
        cursor.execute("""
            SELECT customer_segment, COUNT(*) as cnt 
            FROM marts.dim_users 
            GROUP BY customer_segment
        """)
        segments = cursor.fetchall()
        conn.close()
        
        # Format report
        lines = ["📊 *Weekly Data Pipeline Report*", ""]
        for row in weekly_data:
            lines.append(
                f"📅 {row['revenue_date']} | "
                f"💰 Rp {row['gross_revenue']:,.0f} | "
                f"📦 {row['total_orders']} orders"
            )
        
        lines.append("\n👥 *Customer Segments:*")
        for seg in segments:
            lines.append(f"  • {seg['customer_segment']}: {seg['cnt']}")
        
        report = "\n".join(lines)
        print(report)
        
        # Kirim ke Slack (opsional)
        # webhook_url = Variable.get('slack_webhook')
        # requests.post(webhook_url, json={'text': report})

    PythonOperator(
        task_id='generate_report',
        python_callable=generate_and_send_report,
    )
```

---

## Scheduling Patterns

```python
# ============================================
# ⏰ Scheduling Cheat Sheet
# ============================================

# Preset
schedule_interval='@once'       # Sekali saja
schedule_interval='@hourly'     # Setiap jam
schedule_interval='@daily'      # Setiap hari 00:00
schedule_interval='@weekly'     # Setiap Senin 00:00
schedule_interval='@monthly'    # Setiap tanggal 1

# Cron Expression: minute hour day month weekday
schedule_interval='0 3 * * *'     # Jam 3 pagi setiap hari
schedule_interval='0 */6 * * *'   # Setiap 6 jam
schedule_interval='30 2 * * 1'    # Senin jam 2:30
schedule_interval='0 8 1 * *'     # Tanggal 1 setiap bulan jam 8
schedule_interval='0 0 * * 1-5'   # Weekdays saja
schedule_interval='*/15 * * * *'  # Setiap 15 menit

# Timedelta
schedule_interval=timedelta(hours=2)   # Setiap 2 jam
schedule_interval=timedelta(minutes=30) # Setiap 30 menit
```

---

## Alerting & Notification

```python
# Callback functions untuk alerting
def on_failure_callback(context):
    """Kirim alert ketika task gagal"""
    import requests
    dag_id = context['dag'].dag_id
    task_id = context['task_instance'].task_id
    execution_date = context['ds']
    log_url = context['task_instance'].log_url
    
    message = (
        f"🚨 *Airflow Task Failed!*\n"
        f"• DAG: `{dag_id}`\n"
        f"• Task: `{task_id}`\n"
        f"• Date: {execution_date}\n"
        f"• [View Log]({log_url})"
    )
    
    # Slack webhook
    # requests.post(SLACK_WEBHOOK, json={'text': message})
    print(message)

def on_success_callback(context):
    """Notifikasi sukses"""
    print(f"✅ {context['dag'].dag_id} completed successfully!")

# Penggunaan di DAG
with DAG(
    dag_id='my_dag',
    on_failure_callback=on_failure_callback,
    on_success_callback=on_success_callback,
    # ...
) as dag:
    pass
```

---

## Best Practices

```
✅ DO:
├── Gunakan task_group untuk organisasi tasks
├── Set max_active_runs=1 untuk pipeline yang berat
├── Gunakan TriggerDagRunOperator untuk chain DAGs
├── Implement on_failure_callback untuk alerting
├── Set catchup=False kecuali butuh backfill
├── Gunakan Jinja template untuk dynamic parameters
└── Simpan credentials di Connections/Variables, bukan hardcode

❌ DON'T:
├── Jangan run heavy logic di sensor/operator langsung
├── Jangan pass data besar via XCom (max ~48KB)
├── Jangan buat DAG yang terlalu banyak tasks (>100)
├── Jangan hardcode connection strings
└── Jangan set schedule terlalu sering tanpa alasan
```

---

## Troubleshooting

| Problem | Solusi |
|---------|--------|
| DAG not appearing | `python dags/my_dag.py` untuk cek syntax error |
| Task stuck queued | Cek scheduler running, cek `parallelism` setting |
| Import error | Tambah package di `_PIP_ADDITIONAL_REQUIREMENTS` |
| TriggerDagRun timeout | Increase `poke_interval` dan DAG timeout |
| XCom too large | Gunakan file storage, bukan XCom |
| Timezone mismatch | Set `default_timezone: Asia/Jakarta` di config |

```bash
# Debug commands
docker exec airflow-webserver airflow dags list-import-errors
docker exec airflow-webserver airflow tasks test dag_dbt_transform dbt_deps 2026-02-10
docker exec airflow-webserver airflow dags trigger dag_full_elt_pipeline
```

---

## 📚 Referensi

- [Apache Airflow Documentation](https://airflow.apache.org/docs/)
- [Airflow Best Practices](https://airflow.apache.org/docs/apache-airflow/stable/best-practices.html)
- [Airflow TaskFlow API](https://airflow.apache.org/docs/apache-airflow/stable/tutorial/taskflow.html)

---

> 📝 **Author**: Data Engineering Team  
> 📅 **Last Updated**: 2026-02-10  
> 🏷️ **Tags**: `airflow`, `orchestration`, `scheduling`, `dbt`, `pipeline`
