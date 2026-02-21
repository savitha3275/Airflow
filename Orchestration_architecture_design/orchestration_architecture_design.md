ShopStream Orchestration Architecture Design

Step 1 — Dependency Graph
Pipeline Dependencies
orders_etl → (none)
inventory_sync → (none)

customer_360 → orders_etl
product_catalog → inventory_sync

daily_reports → orders_etl + customer_360
data_quality_checks → orders_etl + customer_360 + product_catalog
ml_feature_pipeline → customer_360
weekly_analytics → daily_reports

Full Dependency Graph:
[orders_etl] ──────────┬──────────────────> [daily_reports] ───> [weekly_analytics]
       │                │
       v                │
[customer_360] ────────┼───────────────┐
       │                │               │
       v                v               │
[ml_feature_pipeline]  [data_quality_checks] <── [product_catalog] <── [inventory_sync]

Independent Chains

Chain A (Revenue domain)
orders_etl → customer_360 → (ml_features, daily_reports, DQ)

Chain B (Product domain)
inventory_sync → product_catalog → DQ

Merge points:
daily_reports
data_quality_checks

Step 2 — DAG Grouping Decisions

We group by:

Schedule alignment
Tight coupling
Failure blast radius
Operational clarity

DAG 1: daily_revenue_pipeline
Pipelines Included

orders_etl
customer_360
ml_feature_pipeline
daily_reports

Schedule:
schedule_interval = "0 2 * * *" (2:00 AM)
start_date = datetime(2024, 1, 1)
catchup = False

Why Grouped Together?
Strong sequential dependency chain
Same daily cadence
Same data domain (revenue & customers)
Prevents cron race conditions (current issue)

Task Flow (within DAG)
orders_etl
   >> customer_360
        >> [ml_feature_pipeline, daily_reports]
Why Not Separate DAGs?

If separate:
Need ExternalTaskSensors
More complexity
Harder SLA guarantees for 7AM report
These are tightly coupled — they belong together.

DAG 2: inventory_catalog_pipeline
Pipelines Included

inventory_sync
product_catalog

Schedule:
inventory_sync: every 4 hours
product_catalog: daily 3AM

Because schedules differ, we split:
DAG 2A: inventory_sync_dag

schedule_interval = "0 */4 * * *"
catchup = False
Only includes:
inventory_sync

DAG 2B: product_catalog_dag

schedule_interval = "0 3 * * *"
Depends on latest successful inventory_sync
Cross-DAG Dependency

Mechanism: Dataset-based scheduling (Airflow 2.4+)

inventory_sync updates dataset:
dataset://warehouse/inventory_table
product_catalog listens to dataset.

Why Dataset?
Data-driven
Avoids time-based race conditions
More modern than sensors

DAG 3: data_quality_dag
Schedule

schedule_interval = "0 5 * * *"

Why Separate?

Depends on multiple domains
Acts as quality gate
Easier to monitor independently
High criticality
Cross-DAG Dependencies

Depends on:
daily_revenue_pipeline
product_catalog_dag

Mechanism: Dataset schedulin
DQ runs when all required datasets are updated.

DAG 4: weekly_analytics_dag
Schedule

"0 8 * * 1" (Monday 8AM)
catchup=False
Why Separate?
Different cadence (weekly)
Heavy workload
Low criticality
Should not block daily pipelines

Cross-DAG Dependency

Depends on:
daily_reports
Mechanism: Dataset trigger on report completion.

Final DAG Structure:
| DAG                    | Pipelines                                                    |
| ---------------------- | ------------------------------------------------------------ |
| daily_revenue_pipeline | orders_etl, customer_360, ml_feature_pipeline, daily_reports |
| inventory_sync_dag     | inventory_sync                                               |
| product_catalog_dag    | product_catalog                                              |
| data_quality_dag       | data_quality_checks                                          |
| weekly_analytics_dag   | weekly_analytics                                             |


Step 3 — Cross-DAG Dependencies
inventory_sync → product_catalog

Mechanism: Dataset

If upstream late:
product_catalog waits
No stale inventory used

If upstream fails:
product_catalog does not run
Slack alert to data team

daily_revenue_pipeline → data_quality_dag

Mechanism: Dataset

If revenue DAG late:
DQ waits
Alert if > 45 min delay

If failure:
DQ skipped
Escalation alert

product_catalog → data_quality_dag

Same Dataset mechanism.

daily_reports → weekly_analytics

Mechanism: Dataset

If weekly heavy:
Set max_active_runs=1
Use separate queue

Step 4 — Executor Decision
Workload Profile

8 pipelines

Peak concurrency: ~6–8 tasks
Heaviest job: weekly_analytics (60 min heavy queries)
Team: 6 engineers
Growth: moderate

Recommended Executor: CeleryExecutor

Why?

1️⃣ Parallelism

Multiple daily + 4-hourly jobs overlap.

2️⃣ Fault Tolerance

If one worker crashes → others continue.

3️⃣ Growth Ready

Can scale workers horizontally.

Why Not Others?

SequentialExecutor → Dev only.

LocalExecutor → Single machine bottleneck.

KubernetesExecutor → Overkill for current size; operational complexity high.

Celery gives best balance.

Infra Design

1 Scheduler

2–3 Celery workers

Redis (broker)

PostgreSQL (metadata DB)

Step 5 — Failure Handling Strategy
daily_revenue_pipeline (HIGH Criticality)
Retry Policy

retries: 3
retry_delay: 5 min
exponential_backoff=True
Only for extraction/API tasks
Alerting
Slack on first failure
PagerDuty if > 30 min late

SLA: Reports ready by 7AM
Scenario: orders_etl fails at 2:15 AM
Flow:
Auto-retry
customer_360 blocked
daily_reports blocked
Alert sent
On recovery → downstream auto-resumes
Idempotency required.

inventory_sync (API unstable)
retries: 5
retry_delay: 3 min
exponential_backoff=True
Use timeout control
To avoid alert fatigue:
Alert only after all retries fail
weekly_analytics
retries: 1

Dedicated Celery queue
Lower priority
max_active_runs=1
Prevents starving daily pipelines.
data_quality_dag
Critical gate.

If DQ fails:
Option chosen:
Block downstream report publishing
Alert immediately
Manual approval override allowed

Step 6 — Summary Architecture Table:
| Pipeline            | DAG             | Schedule  | Criticality | Retry | Alert             | Cross-DAG      |
| ------------------- | --------------- | --------- | ----------- | ----- | ----------------- | -------------- |
| orders_etl          | daily_revenue   | 2AM       | HIGH        | 3     | Slack + PagerDuty | —              |
| customer_360        | daily_revenue   | 2AM chain | MED         | 2     | Slack             | —              |
| ml_feature_pipeline | daily_revenue   | chain     | MED         | 2     | Slack             | —              |
| daily_reports       | daily_revenue   | chain     | HIGH        | 2     | PagerDuty         | feeds weekly   |
| inventory_sync      | inventory_sync  | 4hr       | MED         | 5     | Slack             | dataset        |
| product_catalog     | product_catalog | 3AM       | MED         | 2     | Slack             | waits dataset  |
| data_quality_checks | data_quality    | 5AM       | HIGH        | 1     | Slack             | waits datasets |
| weekly_analytics    | weekly          | Mon 8AM   | LOW         | 1     | Slack             | waits reports  |


Step 7 — Architecture Diagram:
AIRFLOW (CeleryExecutor)

DAG: daily_revenue_pipeline
orders_etl
    ↓
customer_360
    ↓
 ┌───────────────┬───────────────┐
 ↓               ↓
ml_feature     daily_reports
                    ↓
             (Dataset update)

DAG: weekly_analytics_dag
weekly_analytics

DAG: inventory_sync_dag
inventory_sync
   ↓ (Dataset update)

DAG: product_catalog_dag
product_catalog
   ↓ (Dataset update)

DAG: data_quality_dag
data_quality_checks

Executor: CeleryExecutor
Workers: 3
Broker: Redis
Metadata DB: PostgreSQL

Reflection Answers
Hardest Grouping Decision?

Whether to separate daily_reports from revenue DAG. It could be standalone for isolation, but tight SLA (7AM) and dependency chain justify keeping it inside revenue DAG for deterministic execution.

If Team Grows to 20?

Move to KubernetesExecutor for better workload isolation and dynamic scaling. Also enforce DAG ownership per domain (Revenue, Product, ML).

Biggest Risk?

orders_etl is single point of failure. If it fails, entire revenue chain collapses. Mitigation:

Strong retries

DB connection pooling

Replica read DB

Early alerting

Should DQ Be Standalone?

Yes. Centralized DQ improves visibility and prevents duplication. Tradeoff: cross-DAG complexity. Inline checks simplify dependency but reduce governance consistency.

🎯 Final Architectural decision making:

Group tightly coupled pipelines
Separate by cadence
Use Dataset scheduling over sensors
Use Celery for scalable parallelism
Gate executive reports with quality checks
Isolate heavy weekly jobs