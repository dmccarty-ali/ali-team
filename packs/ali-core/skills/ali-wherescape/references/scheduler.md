# ali-wherescape - Scheduler Reference

This reference provides detailed guidance on WhereScape's built-in scheduler configuration and orchestration.

---

## Job Types

| Type | Purpose | Example |
|------|---------|---------|
| **Load Job** | Execute Load table | Load_Customer |
| **Update Job** | Execute transformation | Update_Stage_Customer |
| **Procedure Job** | Run custom stored procedure | Rebuild_Indexes |
| **Script Job** | Execute shell script | Archive_Files.sh |

---

## Dependency Configuration

```
Job: Update_DIM_CUSTOMER
Dependencies:
  - Update_STG_CUSTOMER (must complete successfully)
  - Update_HUB_CUSTOMER (must complete successfully)
  - Update_SAT_CUSTOMER (must complete successfully)

Schedule:
  - Frequency: Daily
  - Start Time: 02:00 AM
  - Retry on Failure: 3 attempts
  - Retry Delay: 5 minutes
```

---

## Error Handling

```sql
-- Generated error handling in WhereScape jobs
BEGIN TRY
    -- Load data
    EXEC Load_Customer;

    -- Update control table
    UPDATE dss_control.dss_load_status
    SET last_load_timestamp = GETDATE(),
        last_load_status = 'SUCCESS',
        rows_loaded = @@ROWCOUNT
    WHERE table_name = 'LOAD_CUSTOMER';

END TRY
BEGIN CATCH
    -- Log error
    INSERT INTO dss_control.dss_error_log (
        job_name,
        error_timestamp,
        error_message,
        error_severity
    )
    VALUES (
        'Load_Customer',
        GETDATE(),
        ERROR_MESSAGE(),
        ERROR_SEVERITY()
    );

    -- Update control table
    UPDATE dss_control.dss_load_status
    SET last_load_status = 'FAILED',
        last_error_message = ERROR_MESSAGE()
    WHERE table_name = 'LOAD_CUSTOMER';

    -- Re-raise error
    THROW;
END CATCH;
```

---

## Integration with External Orchestrators

WhereScape can be orchestrated externally:

| Orchestrator | Integration Method |
|--------------|-------------------|
| **Airflow** | BashOperator calling WhereScape CLI |
| **Control-M** | Job definition calls WhereScape scheduler API |
| **Azure Data Factory** | Web Activity calling WhereScape REST API |
| **AWS Step Functions** | Lambda function invoking WhereScape jobs |

### Example: Airflow DAG

```python
from airflow import DAG
from airflow.operators.bash import BashOperator
from datetime import datetime

dag = DAG('wherescape_daily_load', start_date=datetime(2026, 1, 1), schedule='0 2 * * *')

load_customer = BashOperator(
    task_id='load_customer',
    bash_command='ws_execute -job Load_Customer -environment PROD',
    dag=dag
)

stage_customer = BashOperator(
    task_id='stage_customer',
    bash_command='ws_execute -job Update_Stage_Customer -environment PROD',
    dag=dag
)

load_customer >> stage_customer
```

---

## Scheduler Job Status

| Status | Meaning |
|--------|---------|
| SUCCESS | Job completed successfully |
| FAILED | Job failed with error |
| RUNNING | Job currently executing |
| PENDING | Job waiting for dependencies |
| CANCELLED | Job cancelled by user |
| SKIPPED | Job skipped (condition not met) |

---

**Document Version:** 1.0
**Last Updated:** 2026-02-16
**Maintained By:** ALI AI Team
