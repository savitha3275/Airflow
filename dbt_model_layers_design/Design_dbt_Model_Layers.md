📦 CartWave dbt Model Structure Design
✅ Step 1: Staging Models

Staging models:
Map 1:1 with raw source tables
Use {{ source() }} only
Clean, rename, cast, standardize
Filter invalid records
No joins
No aggregation

1️⃣ stg_orders.sql

-- models/staging/stg_orders.sql
-- Cleans raw orders: casts order_date to DATE, filters out cancelled orders

SELECT
    order_id,
    customer_id,
    product_id,
    CAST(order_date AS DATE)    AS order_date,
    quantity,
    status                      AS order_status

FROM {{ source('raw', 'raw_orders') }}

WHERE status != 'cancelled'

What this does:
CAST(order_date AS DATE) → removes time component
Renames status → order_status (avoids conflict later)
Filters cancelled orders (not relevant for revenue analytics)

2️⃣ stg_customers.sql

-- models/staging/stg_customers.sql
-- Cleans raw customers: trims names, validates email format

SELECT
    customer_id,
    TRIM(name)                  AS customer_name,
    LOWER(TRIM(email))          AS email,
    UPPER(country)              AS country_code,
    CAST(created_at AS DATE)    AS customer_created_date

FROM {{ source('raw', 'raw_customers') }}

WHERE email LIKE '%@%.%'

What this does:
Removes whitespace from names
Normalizes email to lowercase
Standardizes country codes to uppercase
Filters invalid emails
Casts created_at to DATE

3️⃣ stg_products.sql

-- models/staging/stg_products.sql
-- Cleans raw products: standardizes category casing

SELECT
    product_id,
    product_name,
    LOWER(TRIM(category))       AS product_category,
    unit_price

FROM {{ source('raw', 'raw_products') }}

WHERE unit_price > 0

What this does:
Standardizes category casing
Removes whitespace
Filters invalid pricing records

4️⃣ stg_payments.sql

-- models/staging/stg_payments.sql
-- Cleans raw payments: keeps only completed payments

SELECT
    payment_id,
    order_id,
    amount                      AS payment_amount,
    payment_method,
    CAST(payment_date AS DATE)  AS payment_date,
    status                      AS payment_status

FROM {{ source('raw', 'raw_payments') }}

WHERE status = 'completed'

What this does:
Renames amount → payment_amount
Renames status → payment_status
Casts payment_date to DATE
Excludes failed/pending payments

✅ Step 2: Intermediate Model

Intermediate models:
Use {{ ref() }}
Join staging models
No aggregation
Compute business logic

5️⃣ int_orders_enriched.sql

-- models/intermediate/int_orders_enriched.sql
-- Joins orders with customers, products, and payments

SELECT
    -- Order fields
    o.order_id,
    o.order_date,
    o.quantity,
    o.order_status,

    -- Customer fields
    c.customer_id,
    c.customer_name,
    c.email,
    c.country_code,
    c.customer_created_date,

    -- Product fields
    p.product_id,
    p.product_name,
    p.product_category,
    p.unit_price,

    -- Payment fields
    pay.payment_id,
    pay.payment_amount,
    pay.payment_method,
    pay.payment_date,

    -- Derived metric
    (o.quantity * p.unit_price) AS line_total

FROM {{ ref('stg_orders') }} AS o

LEFT JOIN {{ ref('stg_customers') }} AS c
    ON o.customer_id = c.customer_id

LEFT JOIN {{ ref('stg_products') }} AS p
    ON o.product_id = p.product_id

LEFT JOIN {{ ref('stg_payments') }} AS pay
    ON o.order_id = pay.order_id

Why LEFT JOIN?
Preserve all orders
Orders may not yet have payment
Customer could have been filtered (invalid email)
Maintains debugging visibility

✅ Step 3: Mart Models

Mart models:
Aggregate
Business-friendly
Use {{ ref() }}
Materialized as tables

6️⃣ mart_revenue_by_customer.sql

-- models/marts/mart_revenue_by_customer.sql

SELECT
    customer_id,
    customer_name,
    email,
    country_code,
    customer_created_date,

    COUNT(DISTINCT order_id)             AS total_orders,
    SUM(quantity)                        AS total_units_purchased,
    SUM(payment_amount)                  AS total_revenue,

    ROUND(
        SUM(payment_amount) /
        NULLIF(COUNT(DISTINCT order_id), 0),
        2
    )                                    AS avg_order_value,

    MIN(order_date)                      AS first_order_date,
    MAX(order_date)                      AS last_order_date

FROM {{ ref('int_orders_enriched') }}

WHERE payment_amount IS NOT NULL

GROUP BY
    customer_id,
    customer_name,
    email,
    country_code,
    customer_created_date

7️⃣ mart_revenue_by_product.sql

-- models/marts/mart_revenue_by_product.sql

SELECT
    product_id,
    product_name,
    product_category,
    unit_price,

    COUNT(DISTINCT order_id)             AS total_orders,
    SUM(quantity)                        AS total_units_sold,
    SUM(payment_amount)                  AS total_revenue,

    ROUND(
        SUM(payment_amount) /
        NULLIF(SUM(quantity), 0),
        2
    )                                    AS revenue_per_unit,

    COUNT(DISTINCT customer_id)          AS unique_customers

FROM {{ ref('int_orders_enriched') }}

WHERE payment_amount IS NOT NULL

GROUP BY
    product_id,
    product_name,
    product_category,
    unit_price

✅ Step 4: Dependency Graph (DAG)

                    RAW SOURCE TABLES
                    =================
    [raw_orders]    [raw_customers]    [raw_products]    [raw_payments]
         |                |                 |                  |
         v                v                 v                  v
    ┌─────────┐     ┌─────────────┐   ┌────────────┐   ┌────────────┐
    │stg_orders│     │stg_customers│   │stg_products│   │stg_payments│
    └────┬────┘     └─────┬───────┘   └─────┬──────┘   └─────┬──────┘
         │                │                 │                 │
         └────────┬───────┴────────┬────────┴─────────────────┘
                  │
                  v
         ┌─────────────────────────────┐
         │     int_orders_enriched      │
         └──────────┬──────────────────┘
                    │
           ┌────────┴────────┐
           │                 │
           v                 v
┌─────────────────────┐  ┌─────────────────────┐
│mart_revenue_by_     │  │mart_revenue_by_     │
│      customer       │  │      product        │
└─────────────────────┘  └─────────────────────┘


Explicit Dependencies:
| Model                    | Depends On          | Function |
| ------------------------ | ------------------- | -------- |
| stg_orders               | raw_orders          | source() |
| stg_customers            | raw_customers       | source() |
| stg_products             | raw_products        | source() |
| stg_payments             | raw_payments        | source() |
| int_orders_enriched      | all 4 staging       | ref()    |
| mart_revenue_by_customer | int_orders_enriched | ref()    |
| mart_revenue_by_product  | int_orders_enriched | ref()    |

✔ No cycles
✔ Clean layering
✔ 7 total models

✅ Step 5: Folder Structure

cartwave_dbt/
├── dbt_project.yml
├── models/
│   ├── staging/
│   │   ├── _sources.yml
│   │   ├── stg_orders.sql
│   │   ├── stg_customers.sql
│   │   ├── stg_products.sql
│   │   └── stg_payments.sql
│   │
│   ├── intermediate/
│   │   └── int_orders_enriched.sql
│   │
│   └── marts/
│       ├── mart_revenue_by_customer.sql
│       └── mart_revenue_by_product.sql
│
└── README.md

_sources.yml:

version: 2

sources:
  - name: raw
    description: "Raw source tables from CartWave transactional DB"
    schema: raw_data

    tables:
      - name: raw_orders
      - name: raw_customers
      - name: raw_products
      - name: raw_payments

dbt_project.yml:

name: 'cartwave_dbt'
version: '1.0.0'
config-version: 2

profile: 'cartwave'

model-paths: ["models"]
test-paths: ["tests"]

models:
  cartwave_dbt:
    staging:
      +materialized: view
    intermediate:
      +materialized: view
    marts:
      +materialized: table


Why materializations differ?
Staging → views (lightweight)
Intermediate → views (always fresh)
Marts → tables (optimized for BI performance)


✅ Final Checklist (Success Criteria)

✔ 4 staging models using source()
✔ 1 intermediate model using ref()
✔ 2 mart models aggregating from intermediate
✔ Clean DAG
✔ Proper folder structure
✔ dbt best practices followed
✔ SQL syntactically valid