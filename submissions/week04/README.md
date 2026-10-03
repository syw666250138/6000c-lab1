***migration-related observation***
1. **Foreign key enforcing case–job relationship**  
   `jobs.case_id` → `cases.id`.  
   In the model:  
   ```python
   ForeignKey("cases.id", ondelete="CASCADE")
   ```  
   In the migration:  
   ```python
   sa.ForeignKeyConstraint(["case_id"], ["cases.id"], ondelete="CASCADE")
   ```  
   This makes it one-to-many: one `Case` can have many `Job` rows. The SQLAlchemy `relationship()`/`back_populates` only maps the ORM side; the database-level FK enforces the link.

2. **Columns supporting auditability or diagnosis**  
   - **Audit/lifecycle timestamps:** `created_at`, `updated_at` on both `cases` and `jobs`; `jobs.claimed_at`, `jobs.completed_at`.  
   - **Retry/failure diagnosis:** `jobs.attempts`, `jobs.error`.  
   - **AI/triage diagnosis:** `cases.ai_label`, `cases.ai_summary`, `cases.ai_confidence`.  
   - `jobs.job_type` can also help diagnose what work was attempted.  
   - Caveat: this schema does **not** keep full status history. `status` is overwritten, so true audit of transitions would need an audit/event table.

3. **Columns representing current state**  
   - `cases.status`  
   - `cases.ai_label`, `cases.ai_summary`, `cases.ai_confidence` — latest AI triage result.  
   - `jobs.status`  
   - `jobs.attempts`  
   - `jobs.error`  
   - `jobs.claimed_at`, `jobs.completed_at` — timestamps for when the current/terminal state was reached.  
   - `updated_at` reflects the last modification time.

4. **Future project change that would require a migration**  
   Any database schema change, for example:  
   - Add columns such as `cases.priority`, `cases.assignee_id`, `cases.resolved_at`, `jobs.next_run_at`, `jobs.locked_by`.  
   - Add tables such as `audit_log`, `job_attempts`, `users`, or `comments`.  
   - Add/change constraints: check constraints, unique constraints, foreign keys, indexes.  
   - Change column types/lengths, e.g. `status` from `String(20)` to a longer value or native enum.  
   - Rename or drop columns/tables.  
   - Add DB-level enum/check constraints for `CaseStatus`, `JobStatus`, or `JobType`.  
   Adding a new `StrEnum` value in Python may **not** require a migration if the DB column is still a free-form string and long enough, but it would if there is a DB constraint/enum or length limit.

5. **How existing data might be affected**  
   - Adding a `NOT NULL` column without a default or backfill will fail on existing rows.  
   - Adding unique/check/FK constraints can fail if existing rows violate them.  
   - Changing column type/length can truncate or error.  
   - Dropping/renaming columns can lose data or break running code.  
   - Creating indexes/constraints can lock or be slow on large tables.  
   - Backfills update many rows and may need batching.  
   - Adding new status/job-type values leaves old rows unchanged; application code must handle both old and new values.  
   - Changing `ondelete="CASCADE"` affects future deletes, not existing rows, but still needs a migration if altered.

***Schema Diagram***


```text
CASE
────────────────────────────
id              PK UUID
title           VARCHAR
input_text      TEXT
status          VARCHAR
created_at      TIMESTAMPTZ
updated_at      TIMESTAMPTZ


        
        │
        │
        │ 
        ▼

JOB
────────────────────────────
id              PK UUID
case_id         FK → case.id
status          VARCHAR
attempts        INTEGER
claimed_at      TIMESTAMPTZ
started_at      TIMESTAMPTZ
completed_at    TIMESTAMPTZ
error_message   TEXT
created_at      TIMESTAMPTZ
updated_at      TIMESTAMPTZ


        
        │
        │
        │ 
        ▼

RESULT
────────────────────────────
id              PK UUID
job_id          FK → job.id
output_text     TEXT
model_name      VARCHAR
status          VARCHAR
created_at      TIMESTAMPTZ
completed_at    TIMESTAMPTZ
```

## Tables

### 1. `cases`

| Field        | Type        | Constraint  |
| ------------ | ----------- | ----------- |
| `id`         | UUID        | Primary Key |
| `title`      | VARCHAR     | NOT NULL    |
| `input_text` | TEXT        | NOT NULL    |
| `status`     | VARCHAR     | NOT NULL    |
| `created_at` | TIMESTAMPTZ | NOT NULL    |
| `updated_at` | TIMESTAMPTZ | NOT NULL    |

**Status lifecycle:**

`created` → `processing` → `completed` / `failed`

---

### 2. `jobs`

| Field           | Type        | Constraint      |
| --------------- | ----------- | --------------- |
| `id`            | UUID        | Primary Key     |
| `case_id`       | UUID        | FK → `cases.id` |
| `status`        | VARCHAR     | NOT NULL        |
| `attempts`      | INTEGER     | NOT NULL        |
| `claimed_at`    | TIMESTAMPTZ | NULL            |
| `started_at`    | TIMESTAMPTZ | NULL            |
| `completed_at`  | TIMESTAMPTZ | NULL            |
| `error_message` | TEXT        | NULL            |
| `created_at`    | TIMESTAMPTZ | NOT NULL        |
| `updated_at`    | TIMESTAMPTZ | NOT NULL        |

**Status lifecycle:**

`queued` → `running` → `succeeded` / `failed`

Recommended index:

```sql
INDEX jobs(status, created_at)
```

---

### 3. `results`

| Field          | Type        | Constraint     |
| -------------- | ----------- | -------------- |
| `id`           | UUID        | Primary Key    |
| `job_id`       | UUID        | FK → `jobs.id` |
| `output_text`  | TEXT        | NOT NULL       |
| `model_name`   | VARCHAR     | NOT NULL       |
| `status`       | VARCHAR     | NOT NULL       |
| `created_at`   | TIMESTAMPTZ | NOT NULL       |
| `completed_at` | TIMESTAMPTZ | NULL           |

**Status lifecycle:**

`created` → `available` / `failed`

Recommended constraint:

```sql
UNIQUE(results.job_id)
```

This ensures that a job produces at most one persisted result.

## Relationships

```text
cases 1 ────────────< jobs
                      │
                      │ 1
                      │
                      └────────── 0..1 results
```

The workflow is:

```text
Case Created
     │
     ▼
Job Queued
     │
     ▼
Worker Processes Job
     │
     ▼
AI Service Called
     │
     ▼
Result Persisted
     │
     ▼
API Retrieves Result
```

## Design Checks

| Requirement                       | Design                                                                     |
| --------------------------------- | -------------------------------------------------------------------------- |
| One fact in one appropriate place | Case data → `cases`; execution state → `jobs`; AI output → `results`       |
| No vague "everything JSON" table  | Core domain fields use typed relational columns                            |
| Stable identifiers                | UUID primary keys                                                          |
| Required timestamps               | `created_at` and `updated_at`; execution timestamps for jobs/results       |
| Clear status meanings             | Case, job, and result have separate lifecycle states                       |
| Relationships match real workflow | `case → job → result`                                                      |
| Async processing                  | `jobs` represents background work                                          |
| Retry support                     | `attempts` belongs to `jobs`                                               |
| Failure tracking                  | `error_message` belongs to the job execution                               |
| Result uniqueness                 | One result per job enforced with `UNIQUE(job_id)`                          |
| Normalization                     | Result does not duplicate `case_id`; relationship is `result → job → case` |

## Normalized Final Schema

```text
cases
  │
  │ 1:N
  ▼
jobs
  │
  │ 1:0..1
  ▼
results
```

The core relational model is therefore:

**`cases → jobs → results`**
