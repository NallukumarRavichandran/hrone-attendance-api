# DECISIONS.md

1. **Indexes.**
   I created a unique index on `employees.emp_code` to enforce business identity, and `{department: 1, emp_code: 1}` to support filtered employee listings. On `attendance_logs`, I created a compound unique index on `(emp_code, date)` to enforce R1 natural uniqueness and eliminate punch-in race conditions; `{date: -1, emp_code: 1}` to support sorted pagination; and `{emp_code: 1, punch_in: 1}` for quick unclosed punch-in lookups. I considered and rejected a standalone index on `status` due to low cardinality and poor selectivity.

2. **Punch-in race.**
   When two identical punch-in requests arrive simultaneously, both compute the R1 attendance date and attempt `insert_one`. MongoDB's storage engine enforces the compound unique index on `(emp_code, date)` atomically. The winning thread inserts the record and returns HTTP 201; the losing thread triggers `DuplicateKeyError`, which our exception handler intercepts to immediately return HTTP 409.

3. **Ties.**
   Using MongoDB's `$setWindowFields` with `$rank`, tied employees share identical ranks (e.g., 1, 2, 2, 4). The `limit` condition is evaluated against `rank <= limit`. If multiple employees tie at the cutoff rank, all of them are returned (yielding more than `limit` items), with tied rows deterministically ordered by `emp_code` ascending.

4. **Headcount.**
   The aggregation pipeline roots on the `employees` collection filtered by `joined_on <= month_end`, and pulls attendance logs via `$lookup`. By grouping from the employee collection rather than logs, employees who have zero attendance records in that month are still included in the department headcount.

5. **One thing you would change for 100x data.**
   At 100x scale (~10 million records), I would replace on-demand cross-collection aggregations with pre-aggregated monthly summary rollups maintained asynchronously via MongoDB Change Streams, and shard `attendance_logs` on `emp_code`.
