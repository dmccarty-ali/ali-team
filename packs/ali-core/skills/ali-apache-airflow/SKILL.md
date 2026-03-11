---
name: ali-apache-airflow
description: |
  Apache Airflow DAG development and operations. Use when:

  PLANNING: Designing DAG structure, planning task dependencies, choosing
  operators, architecting data pipelines, evaluating scheduling strategies

  IMPLEMENTATION: Writing DAGs, implementing operators, configuring sensors,
  using XCom, dynamic task mapping, setting up connections and variables

  GUIDANCE: Asking about Airflow best practices, DAG design patterns, how to
  handle failures, Astro CLI usage, TaskFlow API vs traditional operators

  REVIEW: Reviewing DAG structure, checking for anti-patterns, validating
  error handling, auditing task dependencies and scheduling
---

# Apache Airflow

## When I Speak Up

This skill activates when the conversation involves:

**Planning/Design:**
- Designing DAG structure and task dependencies
- Choosing between operators for a task
- Planning scheduling and retry strategies
- Architecting multi-DAG workflows

**Implementation:**
- Writing DAG files
- Implementing custom operators or sensors
- Using XCom for task communication
- Configuring connections and variables
- Dynamic task mapping

**Guidance/Best Practices:**
- Asking about DAG design patterns
- Asking about error handling and retries
- Asking about Astro CLI commands
- Asking about TaskFlow API

**Review/Validation:**
- Reviewing DAGs for anti-patterns
- Checking task dependencies
- Validating error handling
- Auditing performance issues

---

## Key Principles

- **DAGs are configuration, not code**: Keep business logic in separate modules
- **Idempotent tasks**: Tasks can be re-run safely without side effects
- **Atomic tasks**: Each task does one thing; avoid mega-tasks
- **Explicit dependencies**: All dependencies declared, no implicit ordering
- **Fail fast**: Detect errors early with sensors and validation
- **XCom sparingly**: Small metadata only; not for large data transfer
- **Test locally first**: Use `dag.test()` before deploying

---

## DAG Design Patterns

### Factory Pattern

```python
# dags/factories/edi_dag_factory.py
from airflow import DAG
from airflow.operators.python import PythonOperator
from datetime import datetime, timedelta

def create_edi_dag(
    dag_id: str,
    transaction_type: str,  # "835" or "837I"
    schedule: str = "@daily"
) -> DAG:
    """Factory for EDI processing DAGs."""

    default_args = {
        'owner': 'data-team',
        'depends_on_past': False,
        'retries': 3,
        'retry_delay': timedelta(minutes=5),
        'email_on_failure': True,
    }

    with DAG(
        dag_id=dag_id,
        default_args=default_args,
        schedule=schedule,
        start_date=datetime(2024, 1, 1),
        catchup=False,
        tags=['edi', transaction_type],
    ) as dag:

        discover = PythonOperator(
            task_id='discover_files',
            python_callable=discover_edi_files,
            op_kwargs={'transaction_type': transaction_type},
        )

        parse = PythonOperator(
            task_id='parse_files',
            python_callable=parse_edi_files,
        )

        load = PythonOperator(
            task_id='load_to_snowflake',
            python_callable=load_to_snowflake,
        )

        discover >> parse >> load

    return dag

# dags/edi_835_dag.py
from factories.edi_dag_factory import create_edi_dag
dag = create_edi_dag('edi_835_ingestion', '835')

# dags/edi_837i_dag.py
from factories.edi_dag_factory import create_edi_dag
dag = create_edi_dag('edi_837i_ingestion', '837I')
```

### Thin DAG Wrapper Pattern

```python
# dags/claims_processing.py
from airflow import DAG
from airflow.operators.python import PythonOperator
from datetime import datetime

# Business logic lives OUTSIDE the DAG file
from pipelines.claims import (
    extract_claims,
    validate_claims,
    load_claims,
)

with DAG(
    'claims_processing',
    schedule='@daily',
    start_date=datetime(2024, 1, 1),
    catchup=False,
) as dag:

    # DAG file only defines structure, not logic
    extract = PythonOperator(
        task_id='extract',
        python_callable=extract_claims,
    )

    validate = PythonOperator(
        task_id='validate',
        python_callable=validate_claims,
    )

    load = PythonOperator(
        task_id='load',
        python_callable=load_claims,
    )

    extract >> validate >> load
```

---

## Operators

### PythonOperator

```python
from airflow.operators.python import PythonOperator

def process_data(ds, **kwargs):
    """Task function receives context via kwargs."""
    execution_date = kwargs['execution_date']
    ti = kwargs['ti']

    # Get XCom from upstream task
    file_count = ti.xcom_pull(task_ids='discover_files', key='file_count')

    # Push XCom for downstream tasks
    ti.xcom_push(key='processed_count', value=100)

    return {'status': 'success', 'count': 100}

task = PythonOperator(
    task_id='process_data',
    python_callable=process_data,
    provide_context=True,  # Deprecated in 2.0+; context always provided
)
```

### BashOperator

```python
from airflow.operators.bash import BashOperator

run_parser = BashOperator(
    task_id='run_edi_parser',
    bash_command='java -jar /opt/parsers/edi-parser.jar {{ ds }}',
    env={
        'SNOWFLAKE_ACCOUNT': '{{ var.value.snowflake_account }}',
        'SNOWFLAKE_DATABASE': '{{ var.value.snowflake_database }}',
    },
)
```

### Snowflake Operator

```python
from airflow.providers.snowflake.operators.snowflake import SnowflakeOperator

load_data = SnowflakeOperator(
    task_id='load_staging',
    snowflake_conn_id='snowflake_default',
    sql="""
        COPY INTO STG_MEDEDI.CLAIM
        FROM @LAKE_MEDEDI.EDI_STAGE/{{ ds }}/
        FILE_FORMAT = (TYPE = PARQUET)
    """,
)
```

---

## TaskGroups

```python
from airflow.utils.task_group import TaskGroup

with DAG('edi_pipeline', ...) as dag:

    with TaskGroup('extraction') as extract_group:
        discover = PythonOperator(task_id='discover', ...)
        download = PythonOperator(task_id='download', ...)
        discover >> download

    with TaskGroup('transformation') as transform_group:
        parse = PythonOperator(task_id='parse', ...)
        validate = PythonOperator(task_id='validate', ...)
        parse >> validate

    with TaskGroup('loading') as load_group:
        stage = PythonOperator(task_id='stage', ...)
        copy = PythonOperator(task_id='copy', ...)
        stage >> copy

    extract_group >> transform_group >> load_group
```

---

## Sensors

### S3 Sensor

```python
from airflow.providers.amazon.aws.sensors.s3 import S3KeySensor

wait_for_file = S3KeySensor(
    task_id='wait_for_edi_file',
    bucket_name='aliunde-prod-edi-files',
    bucket_key='incoming/{{ ds }}/*.edi',
    wildcard_match=True,
    aws_conn_id='aws_default',
    poke_interval=60,  # Check every 60 seconds
    timeout=3600,      # Fail after 1 hour
    mode='poke',       # 'poke' or 'reschedule'
)
```

### External Task Sensor

```python
from airflow.sensors.external_task import ExternalTaskSensor

wait_for_upstream = ExternalTaskSensor(
    task_id='wait_for_claims_dag',
    external_dag_id='claims_processing',
    external_task_id='load',  # Wait for specific task, or None for whole DAG
    execution_delta=timedelta(hours=0),  # Same execution date
    timeout=7200,
    mode='reschedule',  # Free up worker while waiting
)
```

---

## TriggerDagRunOperator

Chain DAGs together for multi-stage pipelines:

```python
from airflow.operators.trigger_dagrun import TriggerDagRunOperator

# Trigger another DAG after this one completes
trigger_downstream = TriggerDagRunOperator(
    task_id='trigger_edw_merge',
    trigger_dag_id='edw_merge_dag',  # DAG to trigger
    wait_for_completion=True,        # Wait for triggered DAG
    poke_interval=60,                # Check every 60 seconds
    reset_dag_run=True,              # Allow re-triggering same date
    conf={'source': 'edi_835'},      # Pass configuration to triggered DAG
)

# In triggered DAG, access conf:
def process_with_conf(**context):
    dag_conf = context['dag_run'].conf
    source = dag_conf.get('source', 'unknown')
    print(f"Processing for source: {source}")
```

### Full Pipeline Orchestration Pattern

```python
# dags/full_pipeline_dag.py
from airflow import DAG
from airflow.operators.trigger_dagrun import TriggerDagRunOperator
from airflow.operators.empty import EmptyOperator

with DAG('full_pipeline', schedule='@daily', ...) as dag:
    start = EmptyOperator(task_id='start')

    # Stage 1: Ingest EDI files
    trigger_835 = TriggerDagRunOperator(
        task_id='trigger_edi_835',
        trigger_dag_id='edi_835_ingestion',
        wait_for_completion=True,
    )

    trigger_837i = TriggerDagRunOperator(
        task_id='trigger_edi_837i',
        trigger_dag_id='edi_837i_ingestion',
        wait_for_completion=True,
    )

    # Stage 2: Create staging tables (after both ingestions)
    trigger_stg = TriggerDagRunOperator(
        task_id='trigger_stg_creation',
        trigger_dag_id='stg_creation_dag',
        wait_for_completion=True,
    )

    # Stage 3: Merge to EDW
    trigger_edw = TriggerDagRunOperator(
        task_id='trigger_edw_merge',
        trigger_dag_id='edw_merge_dag',
        wait_for_completion=True,
    )

    end = EmptyOperator(task_id='end')

    # Parallel ingestion, then sequential processing
    start >> [trigger_835, trigger_837i] >> trigger_stg >> trigger_edw >> end
```

---

## XCom Patterns

### When to Use XCom

| Use Case | XCom? | Alternative |
|----------|-------|-------------|
| Pass file count between tasks | Yes | - |
| Pass list of file paths (<100) | Yes | - |
| Pass large dataframe | No | Write to S3/database |
| Pass status/metadata | Yes | - |
| Pass credentials | No | Use Connections |

### XCom Example

```python
def discover_files(**kwargs):
    ti = kwargs['ti']
    files = list_s3_files('bucket', 'prefix/')

    # Push to XCom (small data only!)
    ti.xcom_push(key='file_paths', value=files[:100])
    ti.xcom_push(key='file_count', value=len(files))

def process_files(**kwargs):
    ti = kwargs['ti']
    files = ti.xcom_pull(task_ids='discover_files', key='file_paths')

    for f in files:
        process(f)
```

---

## Dynamic Task Mapping

```python
from airflow.decorators import task

@task
def get_file_batches() -> list[dict]:
    """Returns list of batches to process."""
    files = list_files()
    return [{'batch_id': i, 'files': batch} for i, batch in enumerate(chunked(files, 100))]

@task
def process_batch(batch: dict):
    """Process one batch - called once per batch."""
    for f in batch['files']:
        process(f)

with DAG('dynamic_processing', ...) as dag:
    batches = get_file_batches()
    process_batch.expand(batch=batches)  # Creates N tasks dynamically
```

---

## Branching

```python
from airflow.operators.python import BranchPythonOperator
from airflow.operators.empty import EmptyOperator

def choose_branch(**kwargs):
    ti = kwargs['ti']
    file_count = ti.xcom_pull(task_ids='discover', key='file_count')

    if file_count == 0:
        return 'no_files_to_process'
    elif file_count < 100:
        return 'process_small_batch'
    else:
        return 'process_large_batch'

branch = BranchPythonOperator(
    task_id='branch',
    python_callable=choose_branch,
)

no_files = EmptyOperator(task_id='no_files_to_process')
small_batch = PythonOperator(task_id='process_small_batch', ...)
large_batch = PythonOperator(task_id='process_large_batch', ...)

branch >> [no_files, small_batch, large_batch]
```

---

## Error Handling

### Retries

```python
default_args = {
    'retries': 3,
    'retry_delay': timedelta(minutes=5),
    'retry_exponential_backoff': True,
    'max_retry_delay': timedelta(minutes=30),
}
```

### Callbacks

```python
def on_failure(context):
    """Called when task fails."""
    task_id = context['task_instance'].task_id
    dag_id = context['dag'].dag_id
    execution_date = context['execution_date']

    send_slack_alert(f"Task {dag_id}.{task_id} failed at {execution_date}")

def on_success(context):
    """Called when task succeeds."""
    log_success_metric(context)

task = PythonOperator(
    task_id='critical_task',
    python_callable=process,
    on_failure_callback=on_failure,
    on_success_callback=on_success,
)
```

### SLA

```python
with DAG(
    'time_sensitive_dag',
    sla_miss_callback=notify_sla_miss,
    ...
) as dag:

    task = PythonOperator(
        task_id='must_finish_quickly',
        python_callable=process,
        sla=timedelta(hours=1),  # Alert if not complete within 1 hour
    )
```

---

## Testing

### Local Testing with dag.test()

```bash
# Test entire DAG
airflow dags test edi_835_ingestion 2024-01-15

# Test single task
airflow tasks test edi_835_ingestion parse_files 2024-01-15
```

### Pytest

```python
# tests/test_edi_dag.py
import pytest
from airflow.models import DagBag

@pytest.fixture
def dagbag():
    return DagBag(dag_folder='dags/', include_examples=False)

def test_dag_loads(dagbag):
    assert dagbag.import_errors == {}
    assert 'edi_835_ingestion' in dagbag.dags

def test_dag_structure(dagbag):
    dag = dagbag.get_dag('edi_835_ingestion')
    assert len(dag.tasks) >= 3
    assert dag.has_task('discover_files')
    assert dag.has_task('parse_files')
    assert dag.has_task('load_to_snowflake')

def test_task_dependencies(dagbag):
    dag = dagbag.get_dag('edi_835_ingestion')
    discover = dag.get_task('discover_files')
    parse = dag.get_task('parse_files')

    assert parse in discover.downstream_list
```

---

## Astro CLI

```bash
# Initialize project
astro dev init

# Start local Airflow
astro dev start

# Stop local Airflow
astro dev stop

# View logs
astro dev logs --follow

# Run a DAG test
astro dev run dags test my_dag 2024-01-15

# Deploy to Astronomer
astro deploy

# List deployments
astro deployment list
```

---

## Connections and Variables

### Setting via CLI

```bash
# Connection
airflow connections add 'snowflake_default' \
    --conn-type 'snowflake' \
    --conn-login 'user' \
    --conn-password 'password' \
    --conn-schema 'LAKE_MEDEDI' \
    --conn-extra '{"account": "abc123", "warehouse": "COMPUTE_WH"}'

# Variable
airflow variables set snowflake_account abc123
```

### Using in DAGs

```python
from airflow.models import Variable
from airflow.hooks.base import BaseHook

# Get variable
account = Variable.get('snowflake_account')

# Get connection
conn = BaseHook.get_connection('snowflake_default')
password = conn.password
```

---

## Performance

### Pools

```python
# Limit concurrent Snowflake connections
task = SnowflakeOperator(
    task_id='load_data',
    pool='snowflake_pool',  # Max 5 concurrent
    ...
)
```

### Priority

```python
# Higher priority runs first when resources limited
urgent_task = PythonOperator(
    task_id='urgent',
    priority_weight=10,  # Default is 1
    ...
)
```

---

## Custom Operators

Create project-specific operators for reusable patterns:

```python
# plugins/operators/snowflake_merge.py
from airflow.models import BaseOperator
from airflow.providers.snowflake.hooks.snowflake import SnowflakeHook

class SnowflakeMergeOperator(BaseOperator):
    """Custom operator for MERGE operations with audit logging."""

    template_fields = ('target_table', 'source_table', 'merge_keys')

    def __init__(
        self,
        target_table: str,
        source_table: str,
        merge_keys: list[str],
        snowflake_conn_id: str = 'snowflake_default',
        **kwargs
    ):
        super().__init__(**kwargs)
        self.target_table = target_table
        self.source_table = source_table
        self.merge_keys = merge_keys
        self.snowflake_conn_id = snowflake_conn_id

    def execute(self, context):
        hook = SnowflakeHook(snowflake_conn_id=self.snowflake_conn_id)

        # Build MERGE statement
        key_conditions = ' AND '.join(
            f'tgt.{k} = src.{k}' for k in self.merge_keys
        )

        sql = f"""
            MERGE INTO {self.target_table} tgt
            USING {self.source_table} src
            ON {key_conditions}
            WHEN MATCHED THEN UPDATE SET ...
            WHEN NOT MATCHED THEN INSERT ...
        """

        rows_affected = hook.run(sql)
        self.log.info(f"Merged {rows_affected} rows into {self.target_table}")

        # Push to XCom for downstream tasks
        return {'rows_affected': rows_affected}

# Usage in DAG
from operators.snowflake_merge import SnowflakeMergeOperator

merge_claims = SnowflakeMergeOperator(
    task_id='merge_claims',
    target_table='EDW_MEDEDI.FACT_CLAIM',
    source_table='STG_MEDEDI.CLAIM',
    merge_keys=['claim_id'],
)
```

---

## Deferrable Operators

Free up worker slots during long waits (Airflow 2.2+):

```python
from airflow.triggers.temporal import TimeDeltaTrigger
from airflow.sensors.base import BaseSensorOperator
from datetime import timedelta

class DeferrableS3Sensor(BaseSensorOperator):
    """S3 sensor that defers instead of blocking a worker."""

    def __init__(self, bucket: str, key: str, **kwargs):
        super().__init__(**kwargs)
        self.bucket = bucket
        self.key = key

    def execute(self, context):
        # Check immediately first
        if self._check_for_file():
            return True

        # Defer to trigger, freeing up the worker
        self.defer(
            trigger=S3KeyTrigger(bucket=self.bucket, key=self.key),
            method_name='execute_complete'
        )

    def execute_complete(self, context, event=None):
        # Called when trigger fires
        if event['status'] == 'success':
            return True
        raise AirflowException(f"File not found: {self.key}")

    def _check_for_file(self) -> bool:
        from airflow.providers.amazon.aws.hooks.s3 import S3Hook
        hook = S3Hook()
        return hook.check_for_key(self.key, self.bucket)
```

### Using Built-in Deferrable Operators

```python
from airflow.providers.amazon.aws.sensors.s3 import S3KeySensor

# Use deferrable mode for long waits
wait_for_file = S3KeySensor(
    task_id='wait_for_edi_file',
    bucket_name='aliunde-prod-edi-files',
    bucket_key='incoming/{{ ds }}/*.edi',
    deferrable=True,  # Free up worker while waiting
    poke_interval=60,
    timeout=3600,
)
```

---

## Data Quality Integration

Integrate data quality checks into DAG workflows:

```python
from airflow.operators.python import PythonOperator, BranchPythonOperator

def run_data_quality_checks(**context):
    """Run data quality validations."""
    from great_expectations.checkpoint import Checkpoint

    # Run Great Expectations checkpoint
    result = checkpoint.run()

    if not result.success:
        failed_expectations = [
            r.expectation_config.expectation_type
            for r in result.run_results.values()
            if not r.success
        ]
        raise AirflowException(f"Data quality failed: {failed_expectations}")

    return {'validation_passed': True, 'records_validated': result.statistics['evaluated_expectations']}

def check_row_counts(**context):
    """Validate row counts match between source and target."""
    ti = context['ti']
    source_count = ti.xcom_pull(task_ids='extract', key='row_count')
    target_count = ti.xcom_pull(task_ids='load', key='row_count')

    if source_count != target_count:
        raise AirflowException(
            f"Row count mismatch: source={source_count}, target={target_count}"
        )

    return source_count

# DAG with data quality gates
with DAG('etl_with_quality', ...) as dag:
    extract = PythonOperator(task_id='extract', ...)
    transform = PythonOperator(task_id='transform', ...)
    load = PythonOperator(task_id='load', ...)

    quality_check = PythonOperator(
        task_id='data_quality_check',
        python_callable=run_data_quality_checks,
    )

    reconcile = PythonOperator(
        task_id='reconcile_counts',
        python_callable=check_row_counts,
    )

    extract >> transform >> load >> [quality_check, reconcile]
```

### SQL-Based Data Quality

```python
from airflow.providers.snowflake.operators.snowflake import SnowflakeOperator

# Check for nulls in required columns
null_check = SnowflakeOperator(
    task_id='null_check',
    sql="""
        SELECT CASE
            WHEN COUNT(*) > 0 THEN
                RAISE_ERROR('Found ' || COUNT(*) || ' null claim_ids')
            ELSE 'OK'
        END
        FROM STG_MEDEDI.CLAIM
        WHERE claim_id IS NULL AND load_date = '{{ ds }}'
    """,
)

# Check for duplicate keys
dupe_check = SnowflakeOperator(
    task_id='duplicate_check',
    sql="""
        SELECT CASE
            WHEN COUNT(*) > 0 THEN
                RAISE_ERROR('Found ' || COUNT(*) || ' duplicate claims')
            ELSE 'OK'
        END
        FROM (
            SELECT claim_id, COUNT(*) as cnt
            FROM STG_MEDEDI.CLAIM
            WHERE load_date = '{{ ds }}'
            GROUP BY claim_id
            HAVING COUNT(*) > 1
        )
    """,
)
```

---

## Secrets Management

Access secrets securely from DAGs:

```python
from airflow.hooks.base import BaseHook
from airflow.models import Variable

# Method 1: Airflow Connections (preferred for external services)
conn = BaseHook.get_connection('snowflake_default')
password = conn.password  # Encrypted in Airflow metadata DB

# Method 2: Airflow Variables (for configuration)
api_endpoint = Variable.get('external_api_url')

# Method 3: AWS Secrets Manager (recommended for sensitive credentials)
def get_secret_from_aws(secret_name: str) -> dict:
    import boto3
    import json

    client = boto3.client('secretsmanager')
    response = client.get_secret_value(SecretId=secret_name)
    return json.loads(response['SecretString'])

# Use in task
def process_with_secrets(**context):
    creds = get_secret_from_aws('prod/database/credentials')
    # Use creds['username'], creds['password']
```

### Secrets Backend Configuration

```python
# airflow.cfg or environment variables
# Use AWS Secrets Manager as backend
[secrets]
backend = airflow.providers.amazon.aws.secrets.secrets_manager.SecretsManagerBackend
backend_kwargs = {"connections_prefix": "airflow/connections", "variables_prefix": "airflow/variables"}

# Connections stored in Secrets Manager:
# airflow/connections/snowflake_default = {"conn_type": "snowflake", "login": "...", "password": "..."}
```

---

## Anti-Patterns to Avoid

| Anti-Pattern | Why It's Bad | Do This Instead |
|--------------|--------------|-----------------|
| Business logic in DAG file | Hard to test, clutters DAG | Import from separate modules |
| Large XCom payloads | Metadata DB bloat, slow | Write to S3/database |
| Top-level code in DAG | Runs on every scheduler parse | Put in functions/tasks |
| `depends_on_past=True` everywhere | Blocks on any historical failure | Use only when truly needed |
| Ignoring task failures | Silent data quality issues | Use callbacks, alerting |
| Hardcoded values | Can't change without deploy | Use Variables or env vars |
| Blocking sensors | Wastes worker slots | Use deferrable=True |
| No data quality checks | Silent data corruption | Add validation tasks |
| Secrets in DAG code | Security risk | Use Connections or Secrets Manager |

---

## Your Codebase

| Purpose | Location |
|---------|----------|
| DAG files | `airflow/dags/` |
| DAG factory | `airflow/dags/dag_factory.py` |
| Shared utilities | `airflow/dags/dag_utils.py` |
| Connections config | `airflow/connections.yaml` |
| Astro project | `airflow/` |

---

## References

- [Airflow Documentation](https://airflow.apache.org/docs/)
- [Astronomer Guides](https://docs.astronomer.io/learn)
- [Airflow Best Practices](https://airflow.apache.org/docs/apache-airflow/stable/best-practices.html)
- [TaskFlow API](https://airflow.apache.org/docs/apache-airflow/stable/tutorial/taskflow.html)
