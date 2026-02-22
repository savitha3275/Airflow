Step 2 — Scenario 1: FileSensor for Partner Data

Step 2a — Requirements Analysis:
## Scenario 1: Partner File Upload

### Requirement Analysis

**What are we waiting for?**  
We are waiting for a file (`transactions.csv`) to appear in a specific directory.

**Is the wait short or long?**  
Potentially long (up to several hours). The partner uploads sometime between midnight and 5 AM.

**What happens if condition is never met?**  
The pipeline must fail and alert the team.

**How frequently should we check?**  
Every 5 minutes is reasonable. The file arrival window is hours, so checking every few seconds is unnecessary.

Step 2b — Sensor Type Selection:
### Sensor Selection

Chosen sensor: **FileSensor**

Reason:
- We are waiting for a file on a filesystem.
- FileSensor is purpose-built to monitor file existence.
- It supports timeout handling and worker slot management.

Step 2c — Sensor Configuration:
# Scenario 1: Wait for Partner File Upload
wait_for_partner_file = FileSensor(
    task_id="wait_for_partner_file",
    filepath="/data/partner/{{ ds }}/transactions.csv",
    poke_interval=300,
    timeout=7200,
    mode="reschedule",
    soft_fail=False,
    dag=dag,
)

process_partner_data = PythonOperator(
    task_id="process_partner_data",
    python_callable=process_transactions,
    dag=dag,
)

wait_for_partner_file >> process_partner_data

Explanation of the Code:
### Code Explanation

- `FileSensor` waits until the specified file exists.
- `filepath` uses `{{ ds }}` which is an Airflow execution date template.
- `poke_interval=300` means the sensor checks every 5 minutes.
- `timeout=7200` means fail after 2 hours if file does not appear.
- `mode="reschedule"` releases the worker slot between checks.
- `soft_fail=False` means this is a hard dependency.
- The downstream task (`process_partner_data`) runs only after the file exists.
- The `>>` operator defines task dependency.

Step 2d — Parameter Justification:
### Scenario 1: Partner File Upload — Parameter Justification

| Parameter | Value | Justification |
|-----------|-------|---------------|
| `filepath` | `/data/partner/{{ ds }}/transactions.csv` | Uses Airflow template `{{ ds }}` to resolve to the execution date (e.g., 2025-06-15). This ensures each DAG run waits for the correct day's file. |
| `poke_interval` | 300 (5 min) | The file arrives "sometime" in a multi-hour window. Checking every 5 minutes balances responsiveness (notice file within 5 min of arrival) against unnecessary I/O load on the file system. |
| `timeout` | 7200 (2 hours) | Partner SLA is "before 5 AM". If the DAG starts at 3 AM and the file hasn't arrived by 5 AM, something is wrong. 2 hours gives enough buffer. |
| `mode` | `reschedule` | The wait could last hours. In `poke` mode, the sensor would hold a worker slot doing nothing for up to 2 hours. `reschedule` releases the slot between checks, freeing compute for other tasks. |
| `soft_fail` | `False` | This file is required. If it doesn't arrive, downstream tasks have no data to process. A hard failure triggers alerts so the team can investigate. |

Step 2e — Failure Behavior:
### Failure Behavior Explanation

- If the file arrives within timeout → sensor succeeds and downstream executes.
- If file never arrives → AirflowSensorTimeout is raised and DAG fails.
- If file is empty → FileSensor does not validate contents. A downstream validation task must check file size/schema.
- If path is incorrect → Sensor waits until timeout. Logs help debug path issues.

Step 3 — Scenario 2: ExternalTaskSensor
Step 3a — Requirements Analysis:
## Scenario 2: Cross-DAG Dependency

### Requirement Analysis

**What condition must be met?**  
The `final_load` task in the `warehouse_refresh` DAG must succeed.

**Do both DAGs run on same schedule?**  
Yes — both run daily at midnight.

**What happens if warehouse_refresh fails?**  
The ML pipeline should fail, not skip. It depends on valid refreshed data.

Step 3b — Sensor Configuration:
wait_for_warehouse = ExternalTaskSensor(
    task_id="wait_for_warehouse_refresh",
    external_dag_id="warehouse_refresh",
    external_task_id="final_load",
    execution_delta=timedelta(0),
    poke_interval=120,
    timeout=10800,
    mode="reschedule",
    soft_fail=False,
    dag=dag,
)

build_features = PythonOperator(
    task_id="build_ml_features",
    python_callable=build_feature_tables,
    dag=dag,
)

wait_for_warehouse >> build_features

Code Explanation:
### Code Explanation

- `ExternalTaskSensor` waits for a task in another DAG.
- `external_dag_id` identifies the upstream DAG.
- `external_task_id` ensures we wait for the final task only.
- `execution_delta=timedelta(0)` means same execution date.
- `poke_interval=120` checks every 2 minutes.
- `timeout=10800` fails after 3 hours.
- `mode="reschedule"` avoids holding a worker slot.

Step 3c: Explain your parameter choices:
| Parameter        | Value             | Justification                                            |
| ---------------- | ----------------- | -------------------------------------------------------- |
| external_dag_id  | warehouse_refresh | Specifies dependent DAG.                                 |
| external_task_id | final_load        | Ensures full completion before proceeding.               |
| execution_delta  | timedelta(0)      | Same execution date alignment.                           |
| poke_interval    | 120 seconds       | Detect completion quickly without overloading scheduler. |
| timeout          | 10800 seconds     | If not complete within 3 hours, likely upstream issue.   |
| mode             | reschedule        | Long wait expected; conserve resources.                  |

Failure Behavior:

warehouse_refresh succeeds: Sensor succeeds; ML pipeline starts.
warehouse_refresh fails: Sensor eventually times out; ML DAG fails.
warehouse_refresh delayed but succeeds: Sensor continues polling and then proceeds.
execution_delta misconfigured: Sensor waits for incorrect execution date and eventually times out.

Scenario 3 — HttpSensor for API Health Check:
Requirement Analysis

What must be true before pulling data?
The API must return HTTP 200 AND JSON body { "status": "healthy" }.

How often should we check?
Every 1 minute.

Maximum wait time?
30 minutes.

Sensor Configuration:
wait_for_api = HttpSensor(
    task_id="wait_for_vendor_api",
    http_conn_id="vendor_api",
    endpoint="/health",
    method="GET",
    response_check=lambda response: response.json().get("status") == "healthy",
    poke_interval=60,
    timeout=1800,
    mode="poke",
    soft_fail=False,
    dag=dag,
)

pull_catalog_data = PythonOperator(
    task_id="pull_catalog_data",
    python_callable=fetch_vendor_catalog,
    dag=dag,
)

wait_for_api >> pull_catalog_data

Code Explanation:
HttpSensor polls an HTTP endpoint.
http_conn_id references connection credentials stored in Airflow.
response_check validates response body content.
poke_interval=60 checks every minute.
timeout=1800 fails after 30 minutes.
mode="poke" is acceptable since wait is expected to be short.

Parameter Justification:
| Parameter      | Value                  | Justification                                   |
| -------------- | ---------------------- | ----------------------------------------------- |
| http_conn_id   | vendor_api             | Keeps credentials out of code.                  |
| endpoint       | /health                | Lightweight health endpoint.                    |
| method         | GET                    | Standard health check method.                   |
| response_check | JSON status validation | Ensures API truly healthy, not just HTTP 200.   |
| poke_interval  | 60 seconds             | Quick recovery detection after maintenance.     |
| timeout        | 1800 seconds           | Long outage likely major issue; fail and alert. |
| mode           | poke                   | Expected wait short; minimal overhead.          |


Failure Behavior

API returns 200 immediately: Sensor succeeds instantly.
API returns 503 for 10 minutes: Sensor keeps retrying; succeeds once healthy.
API down >30 minutes: Sensor times out; DAG fails.
API returns 200 but degraded status: Sensor continues polling.
Network error: Sensor retries until timeout.

Scenario 4 — TimeSensor for Business Time Gate:
Requirement Analysis

What condition must be met?
Current time must be 6:00 AM or later.

Is this hard or soft rule?
Hard business rule.

Timezone consideration?
Must align with business timezone configuration.

Sensor Configuration:
from datetime import time as dt_time

wait_for_business_hours = TimeSensor(
    task_id="wait_for_6am",
    target_time=dt_time(6, 0, 0),
    poke_interval=60,
    mode="reschedule",
    timeout=14400,
    dag=dag,
)

generate_revenue_report = PythonOperator(
    task_id="generate_revenue_report",
    python_callable=build_revenue_report,
    dag=dag,
)

wait_for_business_hours >> generate_revenue_report

Code Explanation:
TimeSensor waits until system clock reaches target time.
target_time=6:00 AM enforces business rule.
poke_interval=60 ensures report starts within 1 minute of 6 AM.
mode="reschedule" prevents worker slot blocking.
timeout acts as safety guard.

Parameter Justification:
| Parameter     | Value         | Justification                              |
| ------------- | ------------- | ------------------------------------------ |
| target_time   | 6:00 AM       | Enforces business rule.                    |
| poke_interval | 60 seconds    | Ensures near-immediate start.              |
| mode          | reschedule    | Long wait possible; conserve resources.    |
| timeout       | 14400 seconds | Safety mechanism against misconfiguration. |


Failure Behavior

DAG starts at 5 AM: Waits 1 hour; succeeds at 6 AM.
DAG starts at 7 AM: Succeeds immediately.
Timezone misalignment: Reports generated at incorrect business time.

Sensor Design Summary:
| # | Scenario             | Sensor Type        | poke_interval | timeout | mode       | soft_fail | Key Risk            |
| - | -------------------- | ------------------ | ------------- | ------- | ---------- | --------- | ------------------- |
| 1 | Partner file upload  | FileSensor         | 300s          | 7200s   | reschedule | False     | File never arrives  |
| 2 | Cross-DAG dependency | ExternalTaskSensor | 120s          | 10800s  | reschedule | False     | Upstream DAG fails  |
| 3 | API health check     | HttpSensor         | 60s           | 1800s   | poke       | False     | Extended API outage |
| 4 | Business time gate   | TimeSensor         | 60s           | 14400s  | reschedule | False     | Timezone mismatch   |


Choosing Between poke and reschedule Mode

I use reschedule mode when expected wait time is long (tens of minutes to hours). This releases worker slots between checks and improves cluster efficiency.

I use poke mode when the expected wait is short (seconds to a few minutes). In short waits, rescheduling overhead may not justify its use.

Principle:
Long waits → reschedule
Short waits → poke