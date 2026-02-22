📦 CartWave dbt — Complete Testing & Documentation Strategy
✅ 1️⃣ STAGING LAYER — schema.yml

Location: models/staging/schema.yml

version: 2

models:

  - name: stg_orders
    description: >
      Cleaned orders from raw_orders. Casts order_date to DATE,
      renames status to order_status, and filters out cancelled orders.
      One row per non-cancelled order.
    columns:
      - name: order_id
        description: "Primary key — unique identifier for each order"
        tests:
          - not_null
          - unique

      - name: customer_id
        description: "Foreign key to stg_customers — customer who placed the order"
        tests:
          - not_null

      - name: product_id
        description: "Foreign key to stg_products — product ordered"
        tests:
          - not_null

      - name: order_date
        description: "Date the order was placed (cast from timestamp)"
        tests:
          - not_null

      - name: quantity
        description: "Number of units ordered"
        tests:
          - not_null

      - name: order_status
        description: "Order status — only completed or pending allowed"
        tests:
          - not_null
          - accepted_values:
              values: ['completed', 'pending']


  - name: stg_customers
    description: >
      Cleaned customers from raw_customers. Trims names,
      lowercases email, uppercases country code,
      and filters invalid emails.
      One row per valid customer.
    columns:
      - name: customer_id
        description: "Primary key — unique customer identifier"
        tests:
          - not_null
          - unique

      - name: customer_name
        description: "Customer full name (trimmed)"
        tests:
          - not_null

      - name: email
        description: "Customer email address (lowercase)"
        tests:
          - not_null
          - unique

      - name: country_code
        description: "Two-letter uppercase country code"
        tests:
          - not_null

      - name: customer_created_date
        description: "Date customer account was created"
        tests:
          - not_null


  - name: stg_products
    description: >
      Cleaned products from raw_products.
      Standardizes category casing and filters invalid prices.
      One row per valid product.
    columns:
      - name: product_id
        description: "Primary key — unique product identifier"
        tests:
          - not_null
          - unique

      - name: product_name
        description: "Product display name"
        tests:
          - not_null

      - name: product_category
        description: "Standardized lowercase category"
        tests:
          - not_null
          - accepted_values:
              values:
                - 'electronics'
                - 'clothing'
                - 'home'
                - 'sports'
                - 'books'
                - 'toys'

      - name: unit_price
        description: "Price per unit in USD (positive values only)"
        tests:
          - not_null


  - name: stg_payments
    description: >
      Cleaned payments from raw_payments.
      Keeps only completed payments.
      One row per completed payment.
    columns:
      - name: payment_id
        description: "Primary key — unique payment identifier"
        tests:
          - not_null
          - unique

      - name: order_id
        description: "Foreign key to stg_orders"
        tests:
          - not_null

      - name: payment_amount
        description: "Payment amount in USD"
        tests:
          - not_null

      - name: payment_method
        description: "Payment method used"
        tests:
          - not_null
          - accepted_values:
              values:
                - 'credit_card'
                - 'paypal'
                - 'bank_transfer'

      - name: payment_date
        description: "Date payment was processed"
        tests:
          - not_null

      - name: payment_status
        description: "Payment status — must be completed"
        tests:
          - not_null
          - accepted_values:
              values: ['completed']


✅ 2️⃣ INTERMEDIATE LAYER — schema.yml

Location: models/intermediate/schema.yml

version: 2

models:
  - name: int_orders_enriched
    description: >
      Enriched order-level dataset joining orders,
      customers, products, and payments.
      Contains one row per order with derived line_total.
      Uses LEFT JOINs to preserve all orders.
    columns:
      - name: order_id
        description: "Primary key — unique order identifier"
        tests:
          - not_null
          - unique

      - name: customer_id
        description: "Foreign key to stg_customers"
        tests:
          - relationships:
              to: ref('stg_customers')
              field: customer_id

      - name: product_id
        description: "Foreign key to stg_products"
        tests:
          - relationships:
              to: ref('stg_products')
              field: product_id

      - name: order_date
        description: "Date order was placed"
        tests:
          - not_null

      - name: line_total
        description: "Computed value: quantity × unit_price"

✅ 3️⃣ MART LAYER — schema.yml

Location: models/marts/schema.yml

version: 2

models:

  - name: mart_revenue_by_customer
    description: >
      Customer-level revenue aggregation used by Marketing and Executive dashboards.
      Contains total orders, revenue, AOV, and lifecycle dates.
      Includes only customers with completed payments.
    columns:
      - name: customer_id
        description: "Primary key — unique customer identifier"
        tests:
          - not_null
          - unique

      - name: email
        description: "Customer email"
        tests:
          - not_null
          - unique

      - name: total_orders
        description: "Total distinct orders"
        tests:
          - not_null

      - name: total_units_purchased
        description: "Total units purchased"
        tests:
          - not_null

      - name: total_revenue
        description: "Total revenue in USD"
        tests:
          - not_null

      - name: avg_order_value
        description: "Average revenue per order"
        tests:
          - not_null

      - name: first_order_date
        description: "Earliest order date"
        tests:
          - not_null

      - name: last_order_date
        description: "Most recent order date"
        tests:
          - not_null


  - name: mart_revenue_by_product
    description: >
      Product-level revenue aggregation used by Product team.
      Contains units sold, revenue, and unique customer counts.
    columns:
      - name: product_id
        description: "Primary key — unique product identifier"
        tests:
          - not_null
          - unique

      - name: total_orders
        description: "Total distinct orders"
        tests:
          - not_null

      - name: total_units_sold
        description: "Total units sold"
        tests:
          - not_null

      - name: total_revenue
        description: "Total revenue in USD"
        tests:
          - not_null

      - name: revenue_per_unit
        description: "Average revenue per unit"
        tests:
          - not_null

      - name: unique_customers
        description: "Distinct customers purchasing this product"
        tests:
          - not_null


✅ 4️⃣ CUSTOM BUSINESS RULE TESTS (Singular Tests)

Location: tests/

1️⃣ assert_revenue_by_customer_is_positive.sql

SELECT *
FROM {{ ref('mart_revenue_by_customer') }}
WHERE total_revenue <= 0

2️⃣ assert_customer_has_at_least_one_order.sql

SELECT *
FROM {{ ref('mart_revenue_by_customer') }}
WHERE total_orders <= 0

3️⃣ assert_first_order_before_last_order.sql

SELECT *
FROM {{ ref('mart_revenue_by_customer') }}
WHERE first_order_date > last_order_date

4️⃣ assert_product_has_units_sold.sql

SELECT *
FROM {{ ref('mart_revenue_by_product') }}
WHERE total_units_sold <= 0

5️⃣ assert_product_revenue_per_unit_is_reasonable.sql

SELECT *
FROM {{ ref('mart_revenue_by_product') }}
WHERE revenue_per_unit > (unit_price * 10)

✔ All custom tests return violating rows only
✔ If query returns 0 rows → test passes

✅ 5️⃣ DATA CONTRACTS

contract_mart_revenue_by_customer.yml

contract:
  model: mart_revenue_by_customer
  owner: Analytics Engineering
  version: 1.0

  columns:
    - name: customer_id
      type: INTEGER
      nullable: false
      unique: true

    - name: total_orders
      type: INTEGER
      nullable: false
      constraints:
        - "total_orders > 0"

    - name: total_revenue
      type: NUMERIC(12,2)
      nullable: false
      constraints:
        - "total_revenue > 0"

    - name: avg_order_value
      type: NUMERIC(10,2)
      nullable: false

  freshness:
    warn_after: "24 hours"
    error_after: "48 hours"

  sla:
    availability: "99.5%"
    refresh_schedule: "Daily by 06:00 UTC"



contract_mart_revenue_by_product.yml

contract:
  model: mart_revenue_by_product
  owner: Analytics Engineering
  version: 1.0

  columns:
    - name: product_id
      type: INTEGER
      nullable: false
      unique: true

    - name: total_units_sold
      type: INTEGER
      nullable: false
      constraints:
        - "total_units_sold > 0"

    - name: total_revenue
      type: NUMERIC(12,2)
      nullable: false
      constraints:
        - "total_revenue > 0"

    - name: revenue_per_unit
      type: NUMERIC(10,2)
      nullable: false

  freshness:
    warn_after: "24 hours"
    error_after: "48 hours"

  sla:
    availability: "99.5%"
    refresh_schedule: "Daily by 06:00 UTC"


✅ 6️⃣ Test Inventory Summary

40 generic tests
5 custom business rule tests
45 total tests
7 fully documented models
2 formal data contracts

✅ 7️⃣ Final Folder Structure:

cartwave_dbt/
├── models/
│   ├── staging/
│   │   ├── schema.yml
│   ├── intermediate/
│   │   ├── schema.yml
│   ├── marts/
│   │   ├── schema.yml
│
├── tests/
│   ├── assert_revenue_by_customer_is_positive.sql
│   ├── assert_customer_has_at_least_one_order.sql
│   ├── assert_first_order_before_last_order.sql
│   ├── assert_product_has_units_sold.sql
│   └── assert_product_revenue_per_unit_is_reasonable.sql
│
├── data_contracts/
│   ├── contract_mart_revenue_by_customer.yml
│   └── contract_mart_revenue_by_product.yml
│
└── README.md