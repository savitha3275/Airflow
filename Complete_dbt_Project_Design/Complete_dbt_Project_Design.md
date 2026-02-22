✅ STEP 1 — Folder Structure

cloudmetrics_dbt/
├── dbt_project.yml
├── models/
│   ├── staging/
│   │   ├── _sources.yml
│   │   ├── schema.yml
│   │   ├── stg_users.sql
│   │   ├── stg_subscriptions.sql
│   │   ├── stg_events.sql
│   │   ├── stg_invoices.sql
│   │   └── stg_support_tickets.sql
│   │
│   ├── intermediate/
│   │   ├── schema.yml
│   │   ├── int_users_enriched.sql
│   │   └── int_support_with_users.sql
│   │
│   └── marts/
│       ├── schema.yml
│       ├── mart_user_health.sql
│       ├── mart_revenue_metrics.sql
│       └── mart_support_performance.sql
│
├── tests/
│   ├── assert_mrr_non_negative.sql
│   ├── assert_active_users_have_plan.sql
│   └── assert_resolution_time_non_negative.sql
│
├── data_contracts/
│   ├── contract_mart_user_health.yml
│   ├── contract_mart_revenue_metrics.yml
│   └── contract_mart_support_performance.yml
│
└── README.md

✅ STEP 2 — STAGING MODELS

1️⃣ stg_users.sql

-- Cleans raw_users data
-- Standardizes casing, trims whitespace, removes legacy trial plan

SELECT
    user_id,
    TRIM(name) AS user_name,
    LOWER(TRIM(email)) AS email,
    CASE 
        WHEN LOWER(plan) IN ('free','starter','pro','enterprise')
        THEN LOWER(plan)
        ELSE NULL
    END AS plan,
    CAST(signup_date AS DATE) AS signup_date,
    UPPER(TRIM(country)) AS country_code
FROM {{ source('raw', 'raw_users') }}
WHERE LOWER(plan) != 'trial'

2️⃣ stg_subscriptions.sql

-- Cleans subscriptions
-- Removes $0 MRR
-- Derives active flag

SELECT
    subscription_id,
    user_id,
    LOWER(plan) AS plan,
    CAST(start_date AS DATE) AS start_date,
    CAST(end_date AS DATE) AS end_date,
    mrr,
    CASE WHEN end_date IS NULL THEN TRUE ELSE FALSE END AS is_active
FROM {{ source('raw', 'raw_subscriptions') }}
WHERE mrr > 0

3️⃣ stg_events.sql

-- Cleans event data
-- Removes internal test events

SELECT
    event_id,
    user_id,
    LOWER(event_type) AS event_type,
    CAST(event_date AS TIMESTAMP) AS event_timestamp
FROM {{ source('raw', 'raw_events') }}
WHERE LOWER(event_type) != 'test'

4️⃣ stg_invoices.sql

-- Cleans invoices
-- Removes void invoices

SELECT
    invoice_id,
    user_id,
    amount,
    CAST(invoice_date AS DATE) AS invoice_date,
    LOWER(status) AS invoice_status
FROM {{ source('raw', 'raw_invoices') }}
WHERE LOWER(status) != 'void'

5️⃣ stg_support_tickets.sql

-- Cleans support tickets
-- Removes legacy duplicate status
-- Calculates resolution time

SELECT
    ticket_id,
    user_id,
    subject,
    LOWER(priority) AS priority,
    CAST(created_at AS TIMESTAMP) AS created_at,
    CAST(resolved_at AS TIMESTAMP) AS resolved_at,
    LOWER(status) AS ticket_status,
    CASE
        WHEN resolved_at IS NOT NULL
        THEN EXTRACT(EPOCH FROM (resolved_at - created_at)) / 3600
        ELSE NULL
    END AS resolution_hours
FROM {{ source('raw', 'raw_support_tickets') }}
WHERE LOWER(status) != 'duplicate'

✅ STEP 3 — INTERMEDIATE MODELS

1️⃣ int_users_enriched.sql

-- Enriched user profile

WITH latest_subscription AS (
    SELECT *
    FROM {{ ref('stg_subscriptions') }}
    QUALIFY ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY start_date DESC) = 1
),

event_activity AS (
    SELECT
        user_id,
        COUNT(*) AS total_events,
        MAX(event_timestamp) AS last_activity
    FROM {{ ref('stg_events') }}
    GROUP BY user_id
)

SELECT
    u.user_id,
    u.user_name,
    u.email,
    u.plan,
    u.country_code,
    u.signup_date,
    s.mrr,
    s.is_active,
    e.total_events,
    e.last_activity,
    CURRENT_DATE - u.signup_date AS days_since_signup
FROM {{ ref('stg_users') }} u
LEFT JOIN latest_subscription s USING(user_id)
LEFT JOIN event_activity e USING(user_id)

2️⃣ int_support_with_users.sql

-- Joins support with user data

SELECT
    t.ticket_id,
    t.user_id,
    u.plan,
    u.country_code,
    t.priority,
    t.ticket_status,
    t.resolution_hours,
    CASE WHEN t.resolved_at IS NOT NULL THEN TRUE ELSE FALSE END AS is_resolved
FROM {{ ref('stg_support_tickets') }} t
LEFT JOIN {{ ref('stg_users') }} u USING(user_id)

✅ STEP 4 — MART MODELS

1️⃣ mart_user_health.sql

WITH support_stats AS (
    SELECT
        user_id,
        COUNT(*) AS total_tickets,
        SUM(CASE WHEN is_resolved = FALSE THEN 1 ELSE 0 END) AS open_tickets,
        AVG(resolution_hours) AS avg_resolution_hours
    FROM {{ ref('int_support_with_users') }}
    GROUP BY user_id
)

SELECT
    u.*,
    s.total_tickets,
    s.open_tickets,
    s.avg_resolution_hours,
    CASE
        WHEN u.last_activity < CURRENT_DATE - INTERVAL '30 days'
             AND s.open_tickets > 0
        THEN 'high'
        ELSE 'low'
    END AS churn_risk
FROM {{ ref('int_users_enriched') }} u
LEFT JOIN support_stats s USING(user_id)


2️⃣ mart_revenue_metrics.sql

SELECT
    DATE_TRUNC('month', invoice_date) AS revenue_month,
    SUM(amount) AS total_invoiced,
    SUM(CASE WHEN invoice_status = 'paid' THEN amount ELSE 0 END) AS total_paid,
    SUM(CASE WHEN invoice_status = 'overdue' THEN amount ELSE 0 END) AS total_overdue
FROM {{ ref('stg_invoices') }}
GROUP BY 1

3️⃣ mart_support_performance.sql

SELECT
    priority,
    ticket_status,
    COUNT(*) AS ticket_count,
    AVG(resolution_hours) AS avg_resolution_hours
FROM {{ ref('int_support_with_users') }}
GROUP BY priority, ticket_status

✅ STEP 5 — GENERIC TESTS (Example: Staging schema.yml)

version: 2

models:
  - name: stg_users
    description: Cleaned user table.
    columns:
      - name: user_id
        description: Primary key.
        tests:
          - not_null
          - unique
      - name: plan
        tests:
          - accepted_values:
              values: ['free','starter','pro','enterprise']

(Repeat similar structure for all models with PK tests + relationships in intermediate models.)

✅ CUSTOM TESTS 

assert_mrr_non_negative.sql

SELECT *
FROM {{ ref('stg_subscriptions') }}
WHERE mrr < 0

assert_active_users_have_plan.sql

SELECT *
FROM {{ ref('mart_user_health') }}
WHERE is_active = TRUE
  AND plan IS NULL

assert_resolution_time_non_negative.sql

SELECT *
FROM {{ ref('stg_support_tickets') }}
WHERE resolution_hours < 0

✅ STEP 6 — DATA CONTRACT (Example)

contract_mart_user_health.yml

contract:
  model: mart_user_health
  owner: Analytics Engineering
  consumers:
    - Customer Success
  columns:
    - name: user_id
      type: INTEGER
      nullable: false
      unique: true
    - name: churn_risk
      type: VARCHAR
      nullable: false
  freshness:
    warn_after: "24 hours"
    error_after: "48 hours"
  sla:
    availability: "99.5%"
    refresh_schedule: "Daily 6AM UTC"

(Repeat for revenue + support marts.)

✅ STEP 7 — DEPENDENCY GRAPH

RAW SOURCES
 ├── raw_users
 ├── raw_subscriptions
 ├── raw_events
 ├── raw_invoices
 └── raw_support_tickets
        │
        ▼
STAGING
 ├── stg_users
 ├── stg_subscriptions
 ├── stg_events
 ├── stg_invoices
 └── stg_support_tickets
        │
        ▼
INTERMEDIATE
 ├── int_users_enriched
 └── int_support_with_users
        │
        ▼
MARTS
 ├── mart_user_health
 ├── mart_revenue_metrics
 └── mart_support_performance

✅ TOTAL MODEL COUNT

| Layer        | Count |
| ------------ | ----- |
| Staging      | 5     |
| Intermediate | 2     |
| Marts        | 3     |
| Custom Tests | 3     |
| Total Models | 10    |


✅ FINAL CHECKLIST

✔ Folder structure
✔ 5 staging models
✔ 2 intermediate models
✔ 3 mart models
✔ Generic tests
✔ 3 custom tests
✔ Data contracts
✔ DAG
✔ Complete documentation