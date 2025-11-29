# Apache Airflow Production Interview Questions (2024-2025)

> Comprehensive guide covering beginner to senior-level questions for Data Engineer interviews.

---

## Table of Contents

1. [Core Concepts](#core-concepts-beginner-to-intermediate)
2. [Architecture & Components](#architecture--components-intermediate)
3. [Executors](#executors-advanced)
4. [XComs & Data Passing](#xcoms--data-passing-intermediate-to-advanced)
5. [Scheduling & Triggers](#scheduling--triggers-intermediate)
6. [Production Deployment](#production-deployment-advanced)
7. [Performance & Optimization](#performance--optimization-advanced)
8. [Airflow 3.0 New Features](#airflow-30-new-features-2025)
9. [Troubleshooting & Debugging](#troubleshooting--debugging-advanced)
10. [Best Practices & Design Patterns](#best-practices--design-patterns-advanced)
11. [Scenario-Based Questions](#scenario-based-questions-senior-level)
12. [Security Questions](#security-questions)

---

## Core Concepts (Beginner to Intermediate)

### 1. What is Apache Airflow and why is it used?

**Expected Answer:**

Apache Airflow is an open-source platform to programmatically author, schedule, and monitor workflows. It's used for orchestrating complex data pipelines, ETL processes, and ML workflows.

**Key benefits:**
- DAGs as Python code (version control, testing)
- Rich UI for monitoring
- Extensible with operators, hooks, and plugins
- Built-in retry and alerting mechanisms

---

### 2. What is a DAG and why must it be "acyclic"?

**Expected Answer:**

A DAG (Directed Acyclic Graph) is a collection of tasks with dependencies. It must be acyclic because:

- Prevents infinite loops in execution
- Ensures deterministic execution order
- Allows the scheduler to calculate task dependencies and execution sequence

---

### 3. Explain the difference between Operators, Sensors, and Hooks.

**Expected Answer:**

| Component | Purpose | Example |
|-----------|---------|---------|
| **Operator** | Defines a single task/unit of work | `BashOperator`, `PythonOperator` |
| **Sensor** | Waits for a condition to be met | `FileSensor`, `S3KeySensor` |
| **Hook** | Interface to external systems | `PostgresHook`, `S3Hook` |

---

### 4. What is the difference between `execution_date` and `logical_date`?

**Expected Answer:**

- `execution_date` (deprecated in Airflow 2.2+) refers to the start of the data interval
- `logical_date` is the replacement term in newer versions
- Both represent when the DAG run is logically scheduled, **NOT** when it actually runs

**Example:** A daily DAG scheduled for 2024-01-01 runs at 2024-01-02 00:00 but has `logical_date` of 2024-01-01

---

## Architecture & Components (Intermediate)

### 5. Describe Airflow's architecture components and their roles.

**Expected Answer:**

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│  Webserver  │    │  Scheduler  │    │  Executor   │
│   (UI/API)  │    │ (Orchestr.) │    │  (Workers)  │
└──────┬──────┘    └──────┬──────┘    └──────┬──────┘
       │                  │                  │
       └────────────┬─────┴─────────────────┘
                    │
              ┌─────▼─────┐
              │ Metadata  │
              │ Database  │
              └───────────┘
```

| Component | Role |
|-----------|------|
| **Scheduler** | Monitors DAGs, triggers task instances, manages dependencies |
| **Webserver** | Provides UI and REST API |
| **Executor** | Determines how tasks are executed (local, distributed) |
| **Metadata Database** | Stores state, DAG info, connections, variables |
| **DAG Processor** (Airflow 3.0) | Parses DAG files separately from scheduler |

---

### 6. What is stored in the Airflow metadata database?

**Expected Answer:**

- DAG definitions and task state
- Task instance history and logs references
- XCom values
- Variables and Connections
- User information and permissions
- Pool and slot configurations
- Job heartbeats

---

### 7. What happens when the scheduler parses a DAG file?

**Expected Answer:**

1. Scheduler scans `dags_folder` at `dag_dir_list_interval`
2. Python files are parsed to discover DAG objects
3. DAG structure is serialized and stored in metadata DB
4. Task dependencies are calculated
5. DagRuns are created based on schedule
6. Tasks are queued when dependencies are met

---

## Executors (Advanced)

### 8. Compare LocalExecutor, CeleryExecutor, and KubernetesExecutor.

**Expected Answer:**

| Executor | Use Case | Pros | Cons |
|----------|----------|------|------|
| **LocalExecutor** | Single machine, moderate workload | Simple setup, no external dependencies | Limited by single machine resources |
| **CeleryExecutor** | Distributed, high throughput | Mature, fast task startup, horizontally scalable | Requires Redis/RabbitMQ, static workers |
| **KubernetesExecutor** | Cloud-native, burstable workloads | Dynamic scaling, task isolation, cost-efficient | Cold start latency, K8s complexity |

---

### 9. When would you use CeleryKubernetesExecutor?

**Expected Answer:**

Use when you have:
- **Mixed workloads**: fast, frequent tasks (Celery) + resource-intensive tasks (K8s)
- Need quick startup for most tasks but isolation for specific ones

**Configuration:**
```ini
[core]
executor = CeleryKubernetesExecutor
```

```python
# Use queue='kubernetes' for K8s tasks
@task(queue='kubernetes')
def heavy_task():
    pass
```

---

### 10. How does KubernetesExecutor handle task execution?

**Expected Answer:**

1. Scheduler creates task instance
2. Executor spawns a new K8s pod for each task
3. Pod runs Airflow worker with the specific task
4. Task executes, results stored in DB
5. Pod terminates after completion
6. Resources are released back to cluster

---

## XComs & Data Passing (Intermediate to Advanced)

### 11. What are XComs and their limitations?

**Expected Answer:**

**XComs** (Cross-Communications) allow tasks to exchange small amounts of data.

**Limitations:**
- Only JSON-serializable values by default
- Stored in metadata DB (size limits ~1GB, varies by DB)
- No encryption or authorization
- Not suitable for large data (DataFrames, files)
- Performance impact with large payloads

---

### 12. How do you handle large data transfer between tasks?

**Expected Answer:**

```python
# ❌ BAD: Passing large DataFrame via XCom
@task
def extract():
    return huge_dataframe  # Don't do this!

# ✅ GOOD: Store in external storage, pass reference
@task
def extract():
    df = process_data()
    path = "s3://bucket/data/output.parquet"
    df.to_parquet(path)
    return path  # Only pass reference

@task
def transform(path: str):
    df = pd.read_parquet(path)
    # Process...
```

---

### 13. What is the TaskFlow API and how does it improve XCom handling?

**Expected Answer:**

TaskFlow API (Airflow 2.0+) simplifies DAG authoring with `@task` decorator:

```python
# Traditional approach
def extract(**context):
    data = get_data()
    context['ti'].xcom_push(key='data', value=data)

def transform(**context):
    data = context['ti'].xcom_pull(task_ids='extract', key='data')

# TaskFlow API - 28% less code
@task
def extract():
    return get_data()  # Auto-pushed to XCom

@task
def transform(data):  # Auto-pulled from XCom
    return process(data)

# Dependencies auto-inferred
transform(extract())
```

---

## Scheduling & Triggers (Intermediate)

### 14. Explain `start_date`, `schedule_interval`, and `catchup`.

**Expected Answer:**

| Parameter | Description |
|-----------|-------------|
| **start_date** | Earliest date the DAG can be scheduled (not when it was created) |
| **schedule_interval** | How often DAG runs (`@daily`, `@hourly`, cron, timedelta) |
| **catchup** | If `True`, backfills all missed DAG runs since `start_date` |

```python
DAG(
    start_date=datetime(2024, 1, 1),
    schedule_interval="@daily",
    catchup=False  # Don't run historical dates
)
```

---

### 15. What is the difference between `schedule_interval` and `timetable`?

**Expected Answer:**

| Feature | schedule_interval | Timetable (Airflow 2.2+) |
|---------|-------------------|--------------------------|
| Flexibility | Simple cron or preset | Custom Python class |
| Use Cases | Regular intervals | Business days, market hours, irregular |
| Data Intervals | Limited control | Full control |

---

### 16. How do you trigger a DAG externally?

**Expected Answer:**

```bash
# CLI
airflow dags trigger my_dag --conf '{"key": "value"}'
```

```bash
# REST API
curl -X POST "http://localhost:8080/api/v1/dags/my_dag/dagRuns" \
  -H "Content-Type: application/json" \
  -d '{"conf": {"key": "value"}}'
```

```python
# Python (from another DAG)
TriggerDagRunOperator(
    task_id='trigger',
    trigger_dag_id='target_dag',
    conf={'key': 'value'}
)
```

---

## Production Deployment (Advanced)

### 17. What database should you use in production and why?

**Expected Answer:**

| Database | Recommendation |
|----------|----------------|
| **SQLite** | ❌ Never in production - doesn't support concurrent writes |
| **PostgreSQL** | ✅ Recommended - robust, supports PGBouncer for connection pooling |
| **MySQL** | ✅ Supported, but PostgreSQL preferred |
| **MS SQL Server** | ❌ Dropped in Airflow 3.0 |

---

### 18. How do you ensure high availability for Airflow?

**Expected Answer:**

1. **Multiple Schedulers**: Airflow 2.0+ supports HA scheduler
2. **Load-balanced Webservers**: Behind Nginx/ALB
3. **Database HA**: PostgreSQL with replication, managed DB services
4. **Distributed Executors**: Celery/Kubernetes for worker redundancy
5. **Persistent Storage**: Shared NFS/S3 for DAGs and logs
6. **Health Checks**: Configure `AIRFLOW__SCHEDULER__ENABLE_HEALTH_CHECK`

---

### 19. How do you handle secrets and sensitive data?

**Expected Answer:**

```python
# ❌ BAD: Hardcoded credentials
conn = psycopg2.connect(password="secret123")

# ✅ GOOD: Use Connections
from airflow.hooks.base import BaseHook
conn = BaseHook.get_connection('my_postgres')

# ✅ GOOD: Use Variables with secrets backend
from airflow.models import Variable
api_key = Variable.get("api_key")

# ✅ BEST: External secrets backend
# airflow.cfg
[secrets]
backend = airflow.providers.amazon.aws.secrets.secrets_manager.SecretsManagerBackend
```

---

### 20. What are the key configurations for production Airflow?

**Expected Answer:**

```ini
[core]
executor = CeleryExecutor  # or KubernetesExecutor
dags_are_paused_at_creation = True
load_examples = False

[scheduler]
dag_dir_list_interval = 300  # Reduce parsing frequency
min_file_process_interval = 60

[webserver]
expose_config = False  # Security
rbac = True

[celery]
worker_concurrency = 16

# Resource limits
parallelism = 32              # Max tasks across all DAGs
dag_concurrency = 16          # Max tasks per DAG
max_active_runs_per_dag = 3
```

---

## Performance & Optimization (Advanced)

### 21. How do you optimize slow DAG parsing?

**Expected Answer:**

1. **Reduce top-level code**: Move imports inside functions
2. **Use `.airflowignore`**: Exclude non-DAG files
3. **Avoid dynamic DAG generation** at parse time
4. **Increase `min_file_process_interval`**
5. **Use DAG serialization** (default in 2.0+)

```python
# ❌ BAD: Heavy import at top level
import pandas as pd  # Loaded every parse

# ✅ GOOD: Import inside task
@task
def process():
    import pandas as pd
    # ...
```

---

### 22. How do you manage task concurrency?

**Expected Answer:**

```python
# DAG level
DAG(
    max_active_runs=3,
    max_active_tasks=10,
)

# Task level
@task(max_active_tis_per_dag=5)
def my_task():
    pass

# Pools - limit across DAGs
with DAG(...) as dag:
    task = BashOperator(
        task_id='limited_task',
        pool='limited_pool',  # Pool with 5 slots
        pool_slots=2
    )
```

---

### 23. What are Pools and when would you use them?

**Expected Answer:**

Pools limit parallel task execution across DAGs.

**Use cases:**
- Limit database connections
- Throttle API calls to avoid rate limits
- Control resource usage for expensive tasks

```bash
# Create pool in CLI
airflow pools set api_pool 5 "API rate limit pool"
```

```python
# Use in task
@task(pool='api_pool', pool_slots=1)
def call_api():
    pass
```

---

## Airflow 3.0 New Features (2025)

### 24. What are the major changes in Airflow 3.0?

**Expected Answer:**

| Feature | Description |
|---------|-------------|
| **Task Execution API (AIP-72)** | Service-oriented architecture, tasks execute remotely |
| **Task SDK** | Lightweight runtime for external task execution |
| **DAG Versioning (AIP-66)** | Native version tracking of DAG changes |
| **Edge Executor (AIP-69)** | Execute tasks in distributed/edge environments |
| **Scheduler-Managed Backfills (AIP-78)** | UI/API support for backfills |
| **React UI Rewrite** | Modern, responsive interface |
| **FastAPI Backend** | Replaces Flask |
| **Conditional Decorators** | `@skip_if`, `@run_if` |
| **Non-unique execution dates** | Better ML/AI workflow support |

---

### 25. What is the Task SDK in Airflow 3.0?

**Expected Answer:**

The Task SDK is a lightweight runtime enabling:
- Task execution outside Airflow's traditional environment
- Running tasks in containers, edge devices, or other runtimes
- Foundation for language-agnostic execution (beyond Python)
- Improved isolation and portability

```python
# Airflow 3.0 asset example
from airflow.sdk import asset, Asset

@asset(schedule="@daily", uri="https://api.example.com/data")
def fetch_data(self) -> dict:
    return requests.get(self.uri).json()
```

---

### 26. What is DAG Versioning and why is it important?

**Expected Answer:**

DAG Versioning (AIP-66) tracks changes to DAG definitions:

- View historical DAG versions in UI
- Understand what version ran for each DAG run
- Debug issues by comparing versions
- No more third-party workarounds needed
- Critical for audit/compliance requirements

---

## Troubleshooting & Debugging (Advanced)

### 27. A DAG is not appearing in the UI. How do you debug?

**Expected Answer:**

```bash
# 1. Check for syntax errors
python /path/to/dag.py

# 2. List DAGs from CLI
airflow dags list

# 3. Check import errors in UI or CLI
airflow dags list-import-errors

# 4. Check scheduler logs
tail -f $AIRFLOW_HOME/logs/scheduler/latest/*.log

# 5. Verify DAG is in correct folder
ls $AIRFLOW_HOME/dags/

# 6. Check .airflowignore isn't excluding it
cat $AIRFLOW_HOME/dags/.airflowignore

# 7. Ensure DAG has valid schedule
# start_date must be in the past
```

---

### 28. Tasks are stuck in "queued" state. What do you check?

**Expected Answer:**

| Check | Command/Location |
|-------|------------------|
| Executor workers running? | `airflow celery worker` or Flower UI |
| Pool slots available? | UI → Admin → Pools |
| Parallelism limits reached? | Check `parallelism`, `dag_concurrency` |
| Scheduler healthy? | Check heartbeat in logs |
| Database connections exhausted? | Check connection pooling |
| Resource constraints? | Memory/CPU on workers |

---

### 29. How do you handle failed tasks and retries?

**Expected Answer:**

```python
from airflow.decorators import task
from datetime import timedelta

@task(
    retries=3,
    retry_delay=timedelta(minutes=5),
    retry_exponential_backoff=True,
    max_retry_delay=timedelta(minutes=30),
    on_failure_callback=alert_on_failure,
    email_on_failure=True,
    email=['team@company.com']
)
def unreliable_task():
    # Task that might fail
    pass

# DAG-level defaults
default_args = {
    'retries': 2,
    'retry_delay': timedelta(minutes=5),
    'on_failure_callback': slack_alert
}
```

---

## Best Practices & Design Patterns (Advanced)

### 30. What are best practices for production DAGs?

**Expected Answer:**

1. **Idempotency**: Tasks should produce same result if run multiple times
2. **Atomicity**: Tasks should be all-or-nothing
3. **Don't delete tasks**: Archive instead to preserve history
4. **Parameterize everything**: Use Variables, Connections, Jinja templates
5. **Test DAGs**: Unit tests for task logic, integration tests for DAGs
6. **Modular design**: Reusable tasks and sub-DAGs
7. **Use TaskGroups** instead of SubDAGs (deprecated)
8. **Document**: Add `doc_md` to DAGs and tasks

```python
with DAG(
    dag_id='production_dag',
    doc_md="""
    ## Production ETL Pipeline
    Extracts from source, transforms, loads to warehouse.
    Owner: data-team@company.com
    """,
    tags=['production', 'etl'],
    default_args=default_args,
) as dag:
    pass
```

---

### 31. How do you structure a large Airflow project?

**Expected Answer:**

```
airflow_project/
├── dags/
│   ├── __init__.py
│   ├── etl/
│   │   ├── __init__.py
│   │   ├── sales_pipeline.py
│   │   └── inventory_pipeline.py
│   └── ml/
│       ├── __init__.py
│       └── training_pipeline.py
├── plugins/
│   ├── operators/
│   │   └── custom_operator.py
│   └── hooks/
│       └── custom_hook.py
├── tests/
│   ├── dags/
│   │   └── test_sales_pipeline.py
│   └── plugins/
├── config/
│   └── airflow.cfg
├── docker-compose.yaml
└── requirements.txt
```

---

### 32. What is the Dynamic Task Mapping feature?

**Expected Answer:**

Dynamic Task Mapping (Airflow 2.3+) creates tasks at runtime:

```python
@task
def get_files():
    return ['file1.csv', 'file2.csv', 'file3.csv']

@task
def process_file(filename: str):
    # Process each file
    return f"processed_{filename}"

# Dynamically creates 3 parallel tasks
files = get_files()
process_file.expand(filename=files)
```

**Benefits:**
- No need to know task count at parse time
- Parallel processing of variable-length inputs
- Replaces need for many SubDAG patterns

---

## Scenario-Based Questions (Senior Level)

### 33. Design a data pipeline that processes files from S3, transforms with Spark, and loads to Snowflake.

**Expected Answer:**

```python
from airflow.decorators import dag, task, task_group
from airflow.providers.amazon.aws.sensors.s3 import S3KeySensor
from airflow.providers.amazon.aws.operators.emr import EmrAddStepsOperator
from airflow.providers.snowflake.operators.snowflake import SnowflakeOperator

@dag(schedule='@hourly', catchup=False)
def s3_to_snowflake():

    # Wait for file
    wait_for_file = S3KeySensor(
        task_id='wait_for_file',
        bucket_name='raw-data',
        bucket_key='incoming/*.parquet',
        poke_interval=60,
        timeout=3600,
        mode='reschedule'  # Release worker while waiting
    )

    @task_group
    def transform():
        # Submit Spark job
        spark_step = EmrAddStepsOperator(...)
        # Wait for completion
        wait_spark = EmrStepSensor(...)
        spark_step >> wait_spark

    @task
    def load_to_snowflake():
        # COPY INTO from S3
        pass

    @task
    def validate():
        # Data quality checks
        pass

    wait_for_file >> transform() >> load_to_snowflake() >> validate()
```

---

### 34. How would you migrate from Airflow 2.x to 3.0?

**Expected Answer:**

1. **Review breaking changes**: Check release notes
2. **Update dependencies**: New providers may be required
3. **Test in staging**: Run full DAG suite
4. **Update deprecated features**:
   - `execution_date` → `logical_date`
   - SubDAGs → TaskGroups
   - Old UI customizations → React plugins
5. **Database migration**: `airflow db migrate`
6. **Update executor config**: New Task Execution API
7. **Gradual rollout**: Use feature flags if available

---

### 35. Your DAGs are taking too long to parse (>30 seconds). How do you fix this?

**Expected Answer:**

1. **Profile parsing**:
   ```bash
   time python dags/slow_dag.py
   ```
2. **Check top-level imports**: Move heavy imports inside tasks
3. **Reduce Jinja rendering**: Pre-compute values
4. **Use DAG factories carefully**: Avoid loops creating many DAGs
5. **Enable DAG serialization**: Already default in 2.0+
6. **Increase `min_file_process_interval`**
7. **Split into multiple files**: Smaller, focused DAGs
8. **Use `.airflowignore`**: Exclude test/utility files

---

## Security Questions

### 36. How do you implement RBAC in Airflow?

**Expected Answer:**

```bash
# Enable RBAC (default in 2.0+)
# airflow.cfg
[webserver]
rbac = True

# Create custom roles via CLI
airflow roles create DataEngineer

# Assign permissions
airflow roles add-perms DataEngineer \
    --action can_read --resource DAG:sales_pipeline

# Create users
airflow users create \
    --username john \
    --role DataEngineer \
    --email john@company.com
```

---

### 37. How do you secure Airflow in production?

**Expected Answer:**

| Security Measure | Implementation |
|------------------|----------------|
| Use secrets backend | AWS Secrets Manager, HashiCorp Vault |
| Enable RBAC | Role-based access control |
| Use HTTPS | TLS for webserver |
| Network isolation | Private subnets, security groups |
| Disable config exposure | `expose_config = False` |
| Audit logging | Track user actions |
| Fernet key rotation | Encrypt connections |
| Service accounts | Limited permissions for workers |

---

## Quick Reference: Common Context Variables

| Variable | Description |
|----------|-------------|
| `ti` / `task_instance` | Current TaskInstance object |
| `ds` | Execution date as string (YYYY-MM-DD) |
| `ds_nodash` | Execution date without dashes |
| `ts` | Execution timestamp (ISO format) |
| `dag` | The DAG object |
| `dag_run` | The current DagRun object |
| `params` | User-defined params dict |
| `var` | Access to Airflow Variables |
| `conn` | Access to Airflow Connections |

---

## Quick Reference: Common Cron Presets

| Preset | Cron Equivalent | Description |
|--------|-----------------|-------------|
| `@once` | - | Run once and only once |
| `@hourly` | `0 * * * *` | Run at the start of every hour |
| `@daily` | `0 0 * * *` | Run at midnight every day |
| `@weekly` | `0 0 * * 0` | Run at midnight every Sunday |
| `@monthly` | `0 0 1 * *` | Run at midnight on the 1st of every month |
| `@yearly` | `0 0 1 1 *` | Run at midnight on January 1st |

---

## Sources

- [DataCamp - Top 21 Airflow Interview Questions](https://www.datacamp.com/blog/top-airflow-interview-questions)
- [ProjectPro - 50 Apache Airflow Interview Questions](https://www.projectpro.io/article/airflow-interview-questions-and-answers/685)
- [Apache Airflow 3.0 Release Blog](https://airflow.apache.org/blog/airflow-three-point-oh-is-here/)
- [Airflow Best Practices Documentation](https://airflow.apache.org/docs/apache-airflow/stable/best-practices.html)
- [Airflow Production Deployment Guide](https://airflow.apache.org/docs/apache-airflow/stable/administration-and-deployment/production-deployment.html)
- [Astronomer - Airflow Executors Explained](https://www.astronomer.io/docs/learn/airflow-executors-explained)
- [Astronomer - TaskFlow API vs Traditional Operators](https://www.astronomer.io/blog/apache-airflow-taskflow-api-vs-traditional-operators/)
- [XComs Documentation](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/xcoms.html)
- [DataCamp - Apache Airflow 3.0](https://www.datacamp.com/blog/apache-airflow-3-0)

---

*Last updated: November 2025*
