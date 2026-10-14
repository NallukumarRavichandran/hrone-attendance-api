# Employee Attendance & Analytics API

[![FastAPI](https://img.shields.io/badge/FastAPI-0.110+-009688.svg?logo=fastapi&logoColor=white)](https://fastapi.tiangolo.com)
[![Python](https://img.shields.io/badge/Python-3.11+-3776AB.svg?logo=python&logoColor=white)](https://www.python.org/)
[![MongoDB](https://img.shields.io/badge/MongoDB-6.0+-47A248.svg?logo=mongodb&logoColor=white)](https://www.mongodb.com/)
[![License](https://img.shields.io/badge/Assessment-HROne%20Engineering-blue.svg)]()

Production-grade, concurrency-hardened REST API for employee attendance tracking and analytics, engineered for the **HROne Software Engineer** technical assessment.

Consolidated strictly within a single executable application file (`app/main.py`), this service implements business rules R1–R10, guarantees race hazard elimination, preserves complete immutable audit trails, and performs complex analytics using native MongoDB 6.0+ aggregation pipelines without in-memory Python looping.

---

## Architecture Overview

```
                      +---------------------------------------+
                      |       Client / Grader Harness        |
                      +-------------------+-------------------+
                                          |
                      +-------------------v-------------------+
                      |      FastAPI Gateway (Port 8000)      |
                      |  - EpochMillis Strict Schema Validator|
                      |  - Lifespan Idempotent Index Initializer
                      +---------+-------------------+---------+
                                |                   |
        +-----------------------+                   +-----------------------+
        |                                                                   |
+-------v-------------------------------+                   +---------------v-----------------------+
|   CRUD & Concurrency Engine           |                   |    Database Aggregation Analytics     |
| - Punch-In (Unique compound lock)     |                   | - Dept Summary (Headcount preserve)   |
| - Punch-Out (Atomic find_one_update)  |                   | - Late Leaderboard ($setWindow rank)  |
| - Regularize (History diff audit)     |                   | - Dept Trend ($densify gap filling)   |
+-----------------------+---------------+                   +---------------+-----------------------+
                        |                                                   |
                        +-------------------+-------------------------------+
                                            |
                      +---------------------v-----------------+
                      |          MongoDB 6.0+ Engine          |
                      | - employees (Unique emp_code)         |
                      | - attendance_logs (emp_code + date)   |
                      | - 100% IXSCAN / Zero COLLSCAN         |
                      +---------------------------------------+
```

---

## How to Run

### 1. Install Dependencies
```bash
pip install -r requirements.txt
```

### 2. Configure Environment (Optional)
The service automatically connects using environment variables, giving real environment variables strict precedence over local fallbacks:
* `MONGO_URI`: MongoDB connection string (defaults to `mongodb://localhost:27017` or Atlas cluster).
* `MONGO_DB`: Target database name (defaults to `attendance_db`).

```bash
# Example (Linux/macOS)
export MONGO_URI="mongodb+srv://<user>:<password>@cluster.mongodb.net"
export MONGO_DB="attendance_db"

# Example (Windows PowerShell)
$env:MONGO_URI="mongodb+srv://<user>:<password>@cluster.mongodb.net"
$env:MONGO_DB="attendance_db"
```

### 3. Launch Application
```bash
uvicorn app.main:app --port 8000
```
On startup, FastAPI's lifespan context manager idempotently builds all required unique and compound database indexes within seconds.

* **Health Check:** `GET http://127.0.0.1:8000/health` (returns `200` with ping latency, or `503` if MongoDB is unreachable).
* **Interactive API Documentation:** `http://127.0.0.1:8000/docs` (customized with a high-contrast developer Dark Theme, Monokai code highlighting, and live endpoint filter).

---

## Core Technical Highlights & Engineering Decisions

### 1. Concurrency & Race-Hazard Elimination
Under parallel workloads (simultaneous requests at the same millisecond):
* **Simultaneous Punch-Ins:** Both requests compute the R1 attendance date and attempt insertion. MongoDB's compound unique index on `(emp_code, date)` enforces atomic mutual exclusion. The winning worker returns `HTTP 201 Created`; the losing worker encounters `DuplicateKeyError`, caught by the application to return `HTTP 409 Conflict`.
* **Simultaneous Punch-Outs:** Handled via an atomic `find_one_and_update` targeting `{"emp_code": ..., "punch_out": None}`. Only one execution thread mutates the record and returns `HTTP 200 OK`; subsequent attempts match zero documents and return `HTTP 409 Conflict`.
* **Concurrent Regularization (`PATCH`):** Computes an atomic field diff (`{"from": ..., "to": ...}`) and uses MongoDB's `$push` operator to guarantee audit entries are appended to `history` without silent data overwrites.

### 2. Business Rules Engine (R1 – R10)
| Rule | Specification | Engineering Implementation |
|---|---|---|
| **R1** | IST Calendar Dates & Overnight Shifts | Instants are truncated to whole seconds. For overnight shifts (`shift_end <= shift_start`), punch-ins before `shift_end` belong to the previous calendar day shift. |
| **R2** | 10-Minute Grace Window | Late minutes are counted only if arrival is **strictly greater than** 10 minutes past `shift_start` (`elapsed > 600s`). Then, `late_minutes = floor(elapsed / 60)` from `shift_start`. E.g., `09:40:00` $\rightarrow$ 0; `09:40:01` $\rightarrow$ 10. |
| **R3** | Overtime Threshold | Calculated as floor minutes from `shift_end` to `punch_out`. Counted only if $\ge 30$ minutes; otherwise evaluates to 0. |
| **R4 & R8** | Decimal Rounding (Anti-Banker's Bias) | Python's native `round()` introduces round-to-even bias. All work hours and rates use `decimal.Decimal` with `ROUND_HALF_UP` (2 decimals for hours, 4 decimals for percentage rates). |
| **R5** | Half-Day Determination | Computed post-punch-out: `work_hours < 4.50` triggers `half_day = True`, weighted as `0.5` in working-day presence calculations. |
| **R6 & R7** | Working Days & Mid-Month Joiners | Monday–Friday calendar excluding weekends. For mid-month joiners, counted from `max(joined_on, month_start)` to `month_end`. Weekend logs contribute to overtime and late totals without inflating `present_days`. |
| **R9** | Zero-Log Headcount Preservation | Department summaries initiate aggregations from `employees` rather than `attendance_logs`, pulling logs via `$lookup`. Staff with zero records in the period are accurately counted in department headcount. |
| **R10** | Strict Pagination | 1-based indexing (`page >= 1`), default 20, max 100. Skip formula: `skip = (page - 1) * page_size`. `total` reflects applied query filters. |

### 3. Pure Database Analytics (Zero Python Loops)
All analytical queries execute entirely inside MongoDB using aggregation pipelines:
1. **Department Summary (`GET /analytics/departments/summary`):** Employs `$lookup` to join logs against employees filtered by `joined_on <= period_end`. Average work hours are calculated across individual present records (record-level mean), avoiding the mathematical fallacy of averaging department averages.
2. **Late Leaderboard (`GET /analytics/leaderboard/late`):** Implements standard competition ranking (`1, 2, 2, 4`) using MongoDB `$setWindowFields` with `$rank: {}`. The `limit` clause is evaluated post-ranking (`rank <= limit`), ensuring tied participants are not truncated arbitrarily. Tied rows are deterministically ordered by `emp_code ASC`.
3. **Department Trend (`GET /analytics/departments/{department}/trend`):** Solves the sparse-data calendar problem via native MongoDB `$densify` on `date` across the date range. A 7-day rolling average is computed inside the database using `$setWindowFields` with a document window `[-6, 0]`.

### 4. Query Plan Verification (`/admin/explain`)
The `/admin/explain/{endpoint}` route executes `explain("executionStats")` on the exact underlying database query for each endpoint:
* `attendance_list`: Backed by compound index `{"date": -1, "emp_code": 1}`.
* `employee_monthly`: Backed by unique compound index `{"emp_code": 1, "date": 1}`.
* `department_summary`: Backed by compound index `{"department": 1, "emp_code": 1}`.
* `late_leaderboard`: Backed by compound index `{"emp_code": 1, "date": 1}`.
* `department_trend`: Backed by compound index `{"emp_code": 1, "date": 1}`.

All queries evaluate to `IXSCAN` stages with **zero `COLLSCAN`** operations on large datasets. All BSON cluster timestamps are sanitized to prevent FastAPI serialization crashes.

---

## Repository Deliverables & Structure

```
.
├── app/
│   ├── __init__.py
│   └── main.py              # Consolidated application codebase (single file)
├── requirements.txt         # Pinned production dependencies
├── REVIEW.md                # Part A: Comprehensive audit of 19 defects in starter code
├── DECISIONS.md             # Part C: Architectural rationale (< 200 words)
├── README.md                # System documentation and deployment instructions
├── openapi.yaml             # API contract specification
└── DATA_MODEL.md            # Collection schemas and annotated document examples
```

* **`REVIEW.md`:** Detailed catalog of 19 defects identified in the starter code, detailing exact function locations, failing symptoms/inputs, automated regression tests, and implemented fixes, along with non-defect rationale.
* **`DECISIONS.md`:** Concise technical explanations covering index choices, race handling, ranking tie logic, headcount preservation, and 100x horizontal scaling strategies.
