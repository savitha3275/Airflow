Scheduling Semantics Lab
🟢 Part 1 — Fix 5 Broken DAG Configurations
🔴 Broken Config 1: The DAG That Never Stops Changing
Diagnosis

What is wrong:
start_date=datetime.now() changes every time the DAG file is parsed.
Airflow parses DAG files continuously.
Each parse creates a new start_date.
The scheduler sees a moving target and never stabilizes.

Why it matters:
start_date defines the beginning of the first data interval.
It must be STATIC.
A dynamic start_date breaks scheduling logic and catchup behavior.

✅ Fix
with DAG(
    dag_id="daily_sales_report",
    start_date=datetime(2024, 1, 1),  # Static date
    schedule_interval="@daily",
    catchup=False,
) as dag:

Explanation:
Now the first data interval is fixed and predictable.


Broken Config 2: The DAG That Spawns 180 Runs
Diagnosis

What is wrong:
start_date is 6 months in the past.
catchup=True (default behavior).
Airflow creates one DAG run per missed interval.
That means ~180 daily runs are immediately scheduled.

Why it matters:
Can overload scheduler and workers.
Can overwhelm APIs.
Backfill should be intentional.

✅ Fix
with DAG(
    dag_id="customer_sync",
    start_date=datetime(2024, 1, 1),
    schedule_interval="@daily",
    catchup=False,
) as dag:

Alternative (If Backfill Intended)
with DAG(
    dag_id="customer_sync",
    start_date=datetime(2024, 1, 1),
    schedule_interval="@daily",
    catchup=True,
    max_active_runs=3,  # Limits concurrency
) as dag:

Explanation:
Controls resource usage during backfills.

Broken Config 3: The Non-Deterministic Task
Diagnosis

What is wrong:
Uses datetime.now() inside task.
Result depends on wall-clock time.
Backfills break.
Retries may process different data.

Why it matters:

Airflow tasks must be:
Deterministic
Idempotent
Based on execution_date, not system time

✅ Fix
def process_daily_data(**context):
    target_date = context["ds"]  # execution_date
    query = f"SELECT * FROM orders WHERE order_date = '{target_date}'"
process = PythonOperator(
    task_id="process_daily_data",
    python_callable=process_daily_data,
    provide_context=True,
)

Explanation:
Now the task always processes the correct data interval


Broken Config 4: The DAG That Never Triggers
Diagnosis

What is wrong:

start_date=datetime(2099, 1, 1)
First interval hasn't ended.
Airflow only triggers at END of interval.
No runs appear.

Why it matters:
start_date must be in the past.
Otherwise scheduler has nothing to trigger.

✅ Fix
with DAG(
    dag_id="new_feature_pipeline",
    start_date=datetime(2024, 6, 1),
    schedule_interval="@daily",
    catchup=False,
) as dag:

Broken Config 5: Template-Unaware Midnight Task
Diagnosis

What is wrong:
Uses datetime.today() - timedelta(1)
That is wall-clock logic.
Airflow already provides execution_date.
Breaks retries and backfills.

Why it matters:
For @daily:
execution_date already represents "yesterday’s data".

✅ Fix
def process_yesterdays_data(**context):
    target_date = context["ds"]
    file_path = f"/data/raw/{target_date}/events.csv"
✅ Even Better (Jinja Template)
bash_command="python process.py --date {{ ds }}"

or
/data/raw/{{ ds_nodash }}/events.csv

Part 2 — Timeline Exercises
🟣 Exercise 1 — Daily Schedule

Config:

start_date = 2024-01-01
schedule = @daily
catchup = True

| Run # | Data Interval Start | Data Interval End | Triggers At | execution_date |
| ----- | ------------------- | ----------------- | ----------- | -------------- |
| 1     | Jan 1 00:00         | Jan 1 23:59       | Jan 2 00:00 | 2024-01-01     |
| 2     | Jan 2 00:00         | Jan 2 23:59       | Jan 3 00:00 | 2024-01-02     |
| 3     | Jan 3 00:00         | Jan 3 23:59       | Jan 4 00:00 | 2024-01-03     |

Key Insight

The DAG does NOT run on Jan 1.
It runs Jan 2 processing Jan 1 data.

Exercise 2 — Hourly Schedule

Config:

start_date = 2024-01-01 00:00
schedule = @hourly

| Run # | Interval Start | Interval End | Triggers At | execution_date |
| ----- | -------------- | ------------ | ----------- | -------------- |
| 1     | 00:00          | 00:59        | 01:00       | 00:00          |
| 2     | 01:00          | 01:59        | 02:00       | 01:00          |
| 3     | 02:00          | 02:59        | 03:00       | 02:00          |

Key Insight
execution_date includes time component.

Exercise 3 — Weekly Schedule

Config:

start_date = 2024-06-01 (Saturday)
cron = 0 6 * * 1 (Monday 06:00)
First matching cron after start_date:
→ June 3, 2024 at 06:00

| Run # | Interval Start | Interval End | Triggers At  | execution_date   |
| ----- | -------------- | ------------ | ------------ | ---------------- |
| 1     | Jun 3 06:00    | Jun 10 05:59 | Jun 10 06:00 | 2024-06-03T06:00 |
| 2     | Jun 10 06:00   | Jun 17 05:59 | Jun 17 06:00 | 2024-06-10T06:00 |
| 3     | Jun 17 06:00   | Jun 24 05:59 | Jun 24 06:00 | 2024-06-17T06:00 |


Tricky Question

If today is June 5:

👉 No runs yet.
First interval (June 3–10) has not ended.

Part 3 — Template Exercises
Template 1 — Date String
❌ Wrong
datetime.today()
✅ Correct
date_str = context["ds"]

or

sql="SELECT * FROM events WHERE event_date='{{ ds }}'"
Template 2 — No-Dash Date
❌ Wrong
strftime('%Y%m%d')
✅ Correct
date_nodash = context["ds_nodash"]

or

/data/{{ ds_nodash }}/output.csv
Template 3 — Partition Path
✅ Deterministic Version
ds = context["ds"]
year, month, day = ds.split("-")

or cleaner:

/data/year={{ execution_date.year }}/month={{ execution_date.strftime('%m') }}/day={{ execution_date.strftime('%d') }}/
Template 4 — Time Range Query
❌ Wrong

Manual timedelta logic.

✅ Correct
start = context["data_interval_start"]
end = context["data_interval_end"]

No arithmetic required. Fully deterministic

Scheduling Cheat Sheet:
| Concept        | ❌ Wrong          | ✅ Right                 |
| -------------- | ---------------- | ----------------------- |
| start_date     | datetime.now()   | datetime(2024,1,1)      |
| Task date      | datetime.today() | context["ds"]           |
| No-dash date   | strftime         | {{ ds_nodash }}         |
| Data range     | timedelta math   | data_interval_start/end |
| Avoid backfill | catchup=True     | catchup=False           |


Core Rules:
start_date must be STATIC
execution_date = START of data interval
DAG triggers at END of interval
Tasks must be deterministic
Avoid unintended catchup