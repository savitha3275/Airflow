✅ Part 1: Failure Scenario Analysis
### Scenario 1: API Rate Limit (HTTP 429)

Classification: Transient
Reasoning:
Rate limits are temporary and reset automatically. The API explicitly provides Retry-After: 60. No human intervention required unless persistent.

Retry Configuration:
retries: 3
retry_delay: timedelta(minutes=2)
retry_exponential_backoff: True
max_retry_delay: timedelta(minutes=10)

Why?
Allows 3 retry windows (2m, 4m, 8m)
Exponential backoff prevents hammering API
Max wait ~14 minutes — acceptable for nightly job

Alert Configuration:
Severity: P3
Channel: Slack #data-alerts

When to alert: After all retries exhausted

Message template:
"extract_product_catalog failed after 3 retries due to HTTP 429. Vendor API rate limit exceeded. Check concurrency and vendor status."

Runbook:
Check task logs for Retry-After header
Verify vendor API status page
Check request concurrency in DAG
Manually re-trigger task
If recurring, request rate limit increase from vendor

### Scenario 2: Database Connection Timeout (Redshift)

Classification: Transient
Reasoning:
Cluster overloaded temporarily. Auto-recovers within 5–10 minutes. Happens occasionally.
Retry Configuration:
retries: 4
retry_delay: timedelta(minutes=3)
retry_exponential_backoff: True
max_retry_delay: timedelta(minutes=15)

Why?
Covers 3m, 6m, 12m windows
Aligns with known 5–10 minute recovery time
Avoids alerting for short maintenance windows

Alert Configuration:
Severity: P2
Channel: PagerDuty (push) + Slack #data-alerts

When to alert: After all retries exhausted
Message template:
"load_to_warehouse failed after retries. Unable to connect to Redshift cluster. Possible prolonged outage or capacity issue."

Runbook:
Check Airflow logs for timeout pattern
Verify Redshift cluster status in AWS Console
Check ongoing maintenance events
Confirm cluster CPU/memory metrics
Manually retry once cluster healthy
If frequent, evaluate WLM/concurrency scaling

### Scenario 3: Invalid Data Schema (Missing Column)

Classification: Permanent
Reasoning:
Retrying will NOT create missing column. External partner changed schema.

Retry Configuration:
retries: 0
retry_delay: N/A
retry_exponential_backoff: False
max_retry_delay: N/A

Why?
Retrying wastes compute and delays alerting.
Alert Configuration:
Severity: P1
Channel: PagerDuty (call) + Slack #data-incidents

When to alert: Immediately
Message template:
"validate_partner_data failed: Missing column 'transaction_amount'. Financial reporting blocked. Immediate partner coordination required."

Runbook:
Confirm missing column in file
Compare with expected schema contract
Contact partner immediately
Roll back to last valid dataset if required
Block downstream financial jobs
Once corrected, re-run validation + downstream DAGs

### Scenario 4: S3 Permission Denied (IAM)

Classification: Permanent
Reasoning:
IAM permission revoked. Will fail until policy restored.

Retry Configuration:
retries: 0
retry_delay: N/A
retry_exponential_backoff: False
max_retry_delay: N/A

Alert Configuration:
Severity: P2
Channel: PagerDuty (push) + Slack #data-alerts

When to alert: Immediately
Message template:
"archive_processed_files failed due to S3 AccessDenied. IAM role likely missing archive bucket permissions."

Runbook:
Confirm AccessDenied error in logs
Verify IAM role attached to Airflow worker
Check bucket policy and recent IAM changes
Coordinate with security team
Restore PutObject permission
Re-run archive task
Confirm file exists in archive bucket


### Scenario 5: Network Partition (Intermittent Connectivity)

Classification: Transient
Reasoning:
Network instability resolved in ~15 minutes. Upload is idempotent.

Retry Configuration:
retries: 5
retry_delay: timedelta(minutes=2)
retry_exponential_backoff: True
max_retry_delay: timedelta(minutes=20)

Why?
Covers up to ~30 minutes of instability. Safe because idempotent upload.
Alert Configuration:
Severity: P3
Channel: Slack #data-alerts

When to alert: After all retries exhausted
Message template:
"sync_to_datalake failed after retries due to network errors. Check connectivity between on-prem and AWS."

Runbook:
Check task logs for connection reset
Verify network monitoring dashboards
Confirm AWS S3 availability
Check corporate firewall/VPN routing
Re-run upload
If recurring, escalate to network team

### Scenario 6: Data Quality Failure (Excessive Nulls)

Classification: Permanent (Data issue)
Reasoning:
Pipeline worked. Data itself is invalid. Retrying won't fix nulls.

Retry Configuration:
retries: 0
retry_delay: N/A
retry_exponential_backoff: False
max_retry_delay: N/A

Alert Configuration:
Severity: P1
Channel: PagerDuty (call) + Slack #data-incidents

When to alert: Immediately
Message template:
"Data quality check failed: email null rate 35.2% (>10%). Marketing campaigns impacted. Source system bug suspected."

Runbook:
Verify null rate calculation
Compare with previous 7 days
Identify source system changes
Notify marketing + source team
Decide whether to halt downstream jobs
Backfill once source system fixed

✅ Part 2: Company Default Retry Policy
Company Default Retry Policy
default_args Configuration
from datetime import timedelta

default_args = {
    "retries": 3,
    "retry_delay": timedelta(minutes=2),
    "retry_exponential_backoff": True,
    "max_retry_delay": timedelta(minutes=15),
}
What This Code Does

retries: number of automatic retry attempts
retry_delay: initial wait before first retry
retry_exponential_backoff: doubles delay each retry
max_retry_delay: prevents runaway long waits

Justification:
| Parameter                 | Value  | Why                                                          |
| ------------------------- | ------ | ------------------------------------------------------------ |
| retries                   | 3      | Covers most transient failures without hiding permanent ones |
| retry_delay               | 2 min  | Fast recovery for short network blips                        |
| retry_exponential_backoff | True   | Prevents retry storms                                        |
| max_retry_delay           | 15 min | Prevents excessive pipeline delay                            |

Total max wait ≈ 2 + 4 + 8 = 14 minutes
Balanced between resilience and fast alerting.

Override Guidelines:
| Scenario                 | Override                         | Reason                |
| ------------------------ | -------------------------------- | --------------------- |
| Known permanent failures | retries=0                        | Immediate alert       |
| API rate limits          | retries=3, retry_delay=2min      | Respect reset window  |
| Critical path tasks      | retries=5, max_retry_delay=60min | Extra resilience      |
| External dependencies    | retries=4                        | Allow recovery window |

✅ Part 3: Alert Priority Matrix
Alert Priority Matrix:
| Priority | Name     | Criteria                                        | Response Time      | Channel                                  | Escalation                                     | Examples                        |
| -------- | -------- | ----------------------------------------------- | ------------------ | ---------------------------------------- | ---------------------------------------------- | ------------------------------- |
| **P1**   | Critical | Revenue/compliance impact, data corruption risk | <15 min            | PagerDuty (call) + Slack #data-incidents | If unacknowledged in 15 min → page Eng Manager | Schema break, severe DQ failure |
| **P2**   | High     | SLA breach, full pipeline failure               | <1 hour            | PagerDuty (push) + Slack #data-alerts    | If unresolved in 2h → Team Lead                | Redshift outage, IAM issue      |
| **P3**   | Medium   | Transient failures, non-critical delays         | 4 hours (business) | Slack #data-alerts                       | If 3+ times/week → escalate to P2              | API rate limits, network blips  |
| **P4**   | Low      | Informational, auto-recovered                   | Next business day  | Email digest                             | None                                           | Successful retries              |

Scenario-to-Priority Mapping:

| Scenario                          | Priority | Justification                       |
| --------------------------------- | -------- | ----------------------------------- |
| 1. API Rate Limit (429)           | P3       | Temporary, auto-recovers            |
| 2. Database Connection Timeout    | P2       | Internal SLA risk if prolonged      |
| 3. Invalid Data Schema            | P1       | Financial reporting blocked         |
| 4. S3 Permission Denied           | P2       | Archiving blocked, needs manual fix |
| 5. Network Partition              | P3       | Transient connectivity              |
| 6. Data Quality — Excessive Nulls | P1       | Marketing + business impact         |
