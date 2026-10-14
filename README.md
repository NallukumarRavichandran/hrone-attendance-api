# Employee Attendance & Analytics API

FastAPI and MongoDB backend service for employee attendance tracking and analytics, implemented for the HROne engineering assessment.

## How to Run

1. **Install dependencies:**
   ```bash
   pip install -r requirements.txt
   ```

2. **Configure environment (optional):**
   Set `MONGO_URI` and `MONGO_DB` via environment variables (or a local `.env`). Real environment variables take precedence:
   - `MONGO_URI`: MongoDB connection string (defaults to `mongodb://localhost:27017`)
   - `MONGO_DB`: Target database name (defaults to `attendance_db`)

3. **Start the API server:**
   ```bash
   uvicorn app.main:app --port 8000
   ```

The application initializes all required unique and compound indexes idempotently on startup.

## Architectural & Implementation Notes

- **Single File:** All API application code is consolidated in `app/main.py`.
- **Business Rules:** Strictly adheres to rules R1 - R10 defined in `openapi.yaml` and `PROBLEM_STATEMENT.pdf` (IST calendar dates, overnight shift boundaries, subsecond truncation, Decimal half-up rounding, 10-minute grace window, and standard competition ranking).
- **Concurrency & Races:** Punch-in and punch-out operations are guaranteed race-safe using MongoDB unique compound indexes and atomic `find_one_and_update` operations.
- **Audit Trail:** Regularizations (`PATCH /attendance/{emp_code}/{date}`) append immutable change diffs to `history` without silent data loss.
- **Zero In-Memory Analytics:** All analytics and explain plans leverage database-level aggregation pipelines and index scans (`IXSCAN`).

## Documentation Files

- `REVIEW.md`: Comprehensive audit catalog of defects identified and resolved from the starter code.
- `DECISIONS.md`: Architectural rationale covering index choices, race condition handling, ranking ties, zero-log headcount, and scaling strategies.
