Scenario 1 — Linear DAG

E-Commerce Daily Pipeline
1️⃣ Task List:

| Task ID               | Description                                   | Operator Type  |
| --------------------- | --------------------------------------------- | -------------- |
| extract_orders        | Pull today's orders from transactional DB     | PythonOperator |
| clean_orders          | Remove duplicates, fix nulls, standardize     | PythonOperator |
| enrich_with_customers | Join orders with customer dimension           | PythonOperator |
| load_to_warehouse     | Insert enriched data into analytics warehouse | PythonOperator |
| send_report_email     | Email daily order summary                     | EmailOperator  |

2️⃣ Dependency Diagram:

[extract_orders] 
      ↓
[clean_orders] 
      ↓
[enrich_with_customers] 
      ↓
[load_to_warehouse] 
      ↓
[send_report_email]

Linear chain — each task depends on the previous one.

3️⃣ Airflow-Style Pseudocode:

Using Apache Airflow style dependency syntax:

extract_orders >> clean_orders >> enrich_with_customers >> load_to_warehouse >> send_report_email

4️⃣ Failure Propagation Analysis:

| Failed Task       | Impact                                            |
| ----------------- | ------------------------------------------------- |
| extract_orders    | Entire pipeline stops; no downstream tasks run    |
| load_to_warehouse | Email skipped; data not loaded                    |
| send_report_email | Data loaded successfully; only notification fails |

Default behavior: downstream tasks are skipped if upstream fails.

5️⃣ Parallelism:

❌ None.

Strict sequential dependency — every step requires output of the previous step.

Scenario 2 — Parallel Fan-In DAG
Multi-Source Analytics
1️⃣ Task List:

| Task ID           | Description                        | Operator Type  |
| ----------------- | ---------------------------------- | -------------- |
| extract_crm       | Pull data from CRM API             | PythonOperator |
| extract_billing   | Pull invoice data from billing API | PythonOperator |
| extract_support   | Pull ticket data from support API  | PythonOperator |
| merge_data        | Join all three datasets            | PythonOperator |
| aggregate_metrics | Compute KPIs                       | PythonOperator |
| load_to_warehouse | Load aggregated data               | PythonOperator |


2️⃣ Dependency Diagram:

[extract_crm]     ──┐
                    ├──> [merge_data] → [aggregate_metrics] → [load_to_warehouse]
[extract_billing] ──┤
                    │
[extract_support] ──┘

Fan-in pattern — multiple branches converge into one task.

3️⃣ Airflow-Style Pseudocode:
[extract_crm, extract_billing, extract_support] >> merge_data >> aggregate_metrics >> load_to_warehouse

Square brackets indicate parallel execution.

4️⃣ Failure Propagation Analysis

| Failed Task      | Impact                                 |
| ---------------- | -------------------------------------- |
| extract_billing  | merge_data skipped → no final dataset  |
| merge_data       | aggregation and load skipped           |
| extract_crm slow | other extracts unaffected; merge waits |


All upstream tasks must succeed before merge runs.

5️⃣ Parallelism

✅ Parallel tasks:

extract_crm
extract_billing
extract_support

These run simultaneously.

Time optimization example:
5 min each sequential = 15 min
5 min parallel = ~5 min

Scenario 3 — Conditional Branching DAG
Data Quality Pipeline
Uses branching logic in Apache Airflow.
1️⃣ Task List:

| Task ID           | Description          | Operator Type        |
| ----------------- | -------------------- | -------------------- |
| extract_data      | Pull raw data        | PythonOperator       |
| validate_data     | Run quality checks   | PythonOperator       |
| branch_on_quality | Decide next path     | BranchPythonOperator |
| transform_data    | Transform valid data | PythonOperator       |
| load_to_warehouse | Load valid data      | PythonOperator       |
| quarantine_data   | Store invalid data   | PythonOperator       |
| send_alert        | Notify team          | EmailOperator        |

2️⃣ Dependency Diagram:
                        (valid)
                          ↓
[extract_data] → [validate_data] → [branch_on_quality]
                          ↓
                        (invalid)

                              ┌──> [transform_data] → [load_to_warehouse]
[extract_data] → [validate_data] → [branch_on_quality]
                              └──> [quarantine_data] →[send_alert]                    
Only ONE path executes.

3️⃣ Airflow-Style Pseudocode:
extract_data >> validate_data >> branch_on_quality

branch_on_quality >> transform_data >> load_to_warehouse
branch_on_quality >> quarantine_data >> send_alert

Branch operator returns only one downstream task ID.

4️⃣ Failure Propagation Analysis:

| Scenario              | Impact                       |
| --------------------- | ---------------------------- |
| extract_data fails    | Entire DAG stops             |
| validate_data crashes | Branch never executes        |
| data invalid          | Valid path skipped           |
| quarantine_data fails | Alert skipped → bad scenario |

Important distinction:
Validation “fails data” ≠ task failure
Task crash prevents branching

5️⃣ Parallelism:

❌ No parallelism in current design
(Branches are mutually exclusive, not concurrent)

Possible improvement:
Run multiple validation checks in parallel before branching

Final Comparison Table:
| Aspect               | Linear         | Fan-In         | Branching         |
| -------------------- | -------------- | -------------- | ----------------- |
| Pattern              | Sequential     | Parallel merge | Conditional path  |
| Total tasks          | 5              | 6              | 7                 |
| Parallel tasks       | 0              | 3              | 0                 |
| Branch points        | 0              | 0              | 1                 |
| Single failure point | extract_orders | merge_data     | branch_on_quality |
| Critical path length | 5              | 4              | 4                 |
