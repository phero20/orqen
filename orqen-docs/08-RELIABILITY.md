# 08 — Reliability

> **Document purpose:** Defines how Orqen handles failures — retries, heartbeat-based worker liveness, crash recovery, server restart, and duplicate/late result handling.  
> **Read before:** Implementing any failure detection, retry logic, or recovery mechanism.  
> **Reference when:** Debugging a stuck job, understanding how a crashed worker's jobs are recovered, or adding a new failure scenario.

---

## Overview

Reliability in Orqen is built on one principle:

> **PostgreSQL is the source of truth. No job is ever permanently lost because of a runtime failure.**

All reliability mechanisms ultimately read from and write to PostgreSQL. In-memory state (including CRS) is always rebuildable from the database.

### Failure Scenarios Covered

| Scenario | Detection Method | Recovery Mechanism |
|---|---|---|
| Worker function throws an error | Worker submits `POST /failure` | Retry with backoff |
| Job execution times out | Server-side timeout monitoring | Treated as failure, retry applies |
| Worker process crashes mid-job | Heartbeat timeout (45s) | Jobs recovered and requeued |
| Worker network partition | Heartbeat timeout | Same as crash |
| Late result from old attempt | `attempt_id` mismatch check | Result ignored, job state preserved |
| Duplicate result submission | `attempt_id` idempotency check | Second submission ignored |
| Server process crashes | — | Restart recovery from PostgreSQL |
| Server restart (planned) | — | Same as crash recovery |

---

## 1. Job Attempt Model

Every execution of a job is tracked as a separate **attempt** in the `job_attempts` table.

```
job_123
  │
  ├── attempt_1  (FAILED at 10:00 — "connection refused")
  ├── attempt_2  (FAILED at 10:05 — "timeout after 30s")
  └── attempt_3  (SUCCESS at 10:10 — result stored)
```

### Why attempts matter

The attempt model solves three problems:

**1. Retry tracking** — `attempt_count` on the job tells the system how many times it has been tried. When `attempt_count >= max_retries`, the job is terminally `FAILED`.

**2. Late result detection** — each attempt has a unique `attempt_id`. When a worker submits a result, the server checks if the submitted `attempt_id` matches the currently active attempt. If not, the result is from a superseded attempt and is silently ignored.

**3. Duplicate result detection** — if a worker submits the same `attempt_id` twice (network retry causing duplicate POST), the server checks the attempt's current status. If it is already `SUCCESS` or `FAILED`, the second submission is a no-op.

### Attempt Lifecycle

```
Job dispatched to worker
        │
        ▼
New job_attempts row created:
  attempt_id  = attempt_xxx
  attempt_num = job.attempt_count + 1
  status      = RUNNING
  worker_id   = dispatched worker
  started_at  = now()
        │
        ▼
Worker submits result or failure
        │
        ▼
Server validates attempt_id matches current active attempt
        │
  ┌─────┴──────────┐
 MATCH           NO MATCH
  │                │
  ▼                ▼
Process result   Ignore (late/duplicate)
Update attempt
row status
```

---

## 2. Retry Mechanism

### Retry Configuration (per job)

| Field | Type | Default | Description |
|---|---|---|---|
| `max_retries` | `integer` | `3` | Maximum number of retry attempts after the first failure |
| `backoff_type` | `string` | `exponential` | `fixed` or `exponential` |
| `backoff_seconds` | `integer` | `5` | Base backoff duration in seconds |
| `timeout_seconds` | `integer` | `null` | Max execution time per attempt; null = unlimited |

### Backoff Calculation

**Fixed backoff:** Every retry waits the same duration.
```
Attempt 1 fails → wait 5s → Attempt 2
Attempt 2 fails → wait 5s → Attempt 3
```

**Exponential backoff:** Each retry waits longer.
```
backoff_seconds = 5, base = 2

Attempt 1 fails → wait 5s  (5 * 2^0)  → Attempt 2
Attempt 2 fails → wait 10s (5 * 2^1)  → Attempt 3
Attempt 3 fails → wait 20s (5 * 2^2)  → Attempt 4
```

### Retry Flow

```
Worker submits POST /jobs/:id/failure
        │
        ▼
Validate attempt_id → matches current attempt
        │
        ▼
job_attempts row: status → FAILED, error stored
        │
        ▼
job.attempt_count++
        │
        ▼
job.attempt_count < job.max_retries?
   │                      │
  YES                      NO
   │                      │
   ▼                      ▼
job.status → RETRYING  job.status → FAILED (terminal)
Schedule re-queue       job.error = final error message
after backoff           DAG: propagate failure downstream
   │
   ▼
[backoff expires]
   │
   ▼
job.status → QUEUED
   │
   ▼
Dependency check (no deps or already satisfied)
   │
   ▼
job.status → READY
   │
   ▼
Scheduler dispatches (possibly to a different worker)
```

### Maximum Retries Exhausted

When `attempt_count >= max_retries` and the latest attempt fails:
- `jobs.status → FAILED`
- `jobs.error` is set to the last error message
- `jobs.completed_at` is set
- DAG engine is notified to propagate failure to downstream jobs per failure policy (V1: block — downstream jobs also fail)

---

## 3. Heartbeat & Worker Liveness Detection

### Heartbeat Protocol

- Workers send `POST /worker/heartbeat` every **15 seconds**
- Server updates `workers.last_heartbeat_at` on every heartbeat
- A background monitor checks for workers whose `last_heartbeat_at` is older than **45 seconds**

### Heartbeat Monitor

The heartbeat monitor runs as a background goroutine on the Orqen server:

```
Every 15 seconds (configurable):
        │
        ▼
Query PostgreSQL:
  SELECT id FROM workers
  WHERE status != 'OFFLINE'
    AND last_heartbeat_at < now() - interval '45 seconds'
        │
        ▼
For each timed-out worker:
  MarkWorkerOffline(workerID)
```

### MarkWorkerOffline Flow

```
Worker X heartbeat timed out
        │
        ▼
workers.status → OFFLINE
        │
        ▼
Remove Worker X from all CRS capability instances:
  for each capability in worker_capabilities WHERE worker_id = X:
    registry[(projectID, capability)].Remove(X)
        │
        ▼
Find all RUNNING jobs assigned to Worker X:
  SELECT id FROM jobs WHERE worker_id = X AND status = 'RUNNING'
        │
        ▼
For each job: RecoverJob(jobID)
```

### RecoverJob Flow

```
Job J was RUNNING on crashed Worker X
        │
        ▼
Current job_attempts row for J: status → FAILED
  error = "worker offline — heartbeat timeout"
  completed_at = now()
        │
        ▼
job.attempt_count++
        │
        ▼
job.attempt_count < job.max_retries?
   │                      │
  YES                      NO
   │                      │
   ▼                      ▼
job.status → RETRYING  job.status → FAILED (terminal)
job.worker_id = NULL
Schedule re-queue after backoff
   │
[backoff expires]
   │
job.status → QUEUED → READY → dispatched to another worker
```

### Configuration

| Parameter | Default | Description |
|---|---|---|
| `HEARTBEAT_INTERVAL_SECONDS` | `15` | How often workers must heartbeat |
| `HEARTBEAT_TIMEOUT_SECONDS` | `45` | How long before a worker is considered dead |
| `MONITOR_INTERVAL_SECONDS` | `15` | How often the server checks for dead workers |

All configurable via environment variables.

---

## 4. Late Result Handling

A late result occurs when a worker submits a result for a job attempt that has already been superseded.

### Scenario

```
Job J dispatched → Attempt 1 (attempt_id: att_001)
        │
        ▼
Worker crashes → heartbeat timeout
        │
        ▼
Attempt 1 marked FAILED, job recovered
        │
        ▼
Job J dispatched again → Attempt 2 (attempt_id: att_002)
        │
        ▼
[Network comes back — Worker submits result for att_001]
        │
        ▼
Server checks: active attempt_id for Job J = att_002
att_001 != att_002
        │
        ▼
Return 409 ATTEMPT_MISMATCH — result ignored
Job J continues on Attempt 2
```

### Rule
The `attempt_id` submitted by the worker must exactly match the `attempt_id` of the currently `RUNNING` attempt for that job. Any mismatch is silently dropped — the job state is not affected.

---

## 5. Duplicate Result Handling

A duplicate result occurs when the same attempt submits a result more than once (e.g. network retry causes the worker to POST twice).

### Scenario

```
Worker submits POST /jobs/:id/result with attempt_id: att_002
        │
        ▼
Server processes → job_attempts[att_002].status = SUCCESS
        │
        ▼
[Network failure — worker doesn't receive 200 response]
        │
        ▼
Worker retries → POST /jobs/:id/result with attempt_id: att_002 again
        │
        ▼
Server checks: attempt_id att_002 exists AND status = SUCCESS
        │
        ▼
Return 200 OK (idempotent) — no state change
```

### Rule
Before processing any result submission, the server checks the `job_attempts` row for the given `attempt_id`. If the status is already `SUCCESS` or `FAILED`, the submission is a no-op and returns `200 OK`. This makes result submission **idempotent**.

---

## 6. Job Execution Timeout

If a worker accepts a job but does not submit a result within `timeout_seconds`:

```
Job dispatched at 10:00:00
timeout_seconds = 120

At 10:02:00 — no result submitted
        │
        ▼
Server timeout monitor detects:
  job.status = RUNNING
  job.started_at + timeout_seconds < now()
        │
        ▼
Treated as a failure:
  Current attempt: status → TIMEOUT
  Same retry logic applies (attempt_count vs max_retries)
```

The timeout monitor runs alongside the heartbeat monitor. It queries:

```sql
SELECT id FROM jobs
WHERE status = 'RUNNING'
  AND timeout_seconds IS NOT NULL
  AND started_at + (timeout_seconds * interval '1 second') < now()
```

Note: A worker crash will also be caught by the heartbeat monitor. The timeout monitor is a safety net for workers that are alive but stuck (infinite loop, deadlock, etc.).

---

## 7. Server Restart Recovery

PostgreSQL is the durable source of truth. Orqen can crash and restart without losing any jobs.

### Recovery Flow

```
Orqen server starts
        │
        ▼
Step 1: Load project data from PostgreSQL
        │
        ▼
Step 2: Rebuild CRS registry
  Query: all non-OFFLINE workers + capabilities
  Call sched.BatchAdd() for each (project, capability) pool
        │
        ▼
Step 3: Recover in-flight jobs
  Query: all jobs WHERE status = 'RUNNING'
  For each: check if assigned worker is still valid
    Worker still ONLINE (heartbeat recent)? → Leave as RUNNING
    Worker OFFLINE or unknown?              → RecoverJob()
        │
        ▼
Step 4: Re-queue RETRYING jobs whose backoff has expired
  Query: jobs WHERE status = 'RETRYING'
         AND scheduled_retry_at <= now()
  For each: status → QUEUED → READY → enter scheduling pipeline
        │
        ▼
Step 5: Resume heartbeat monitor and timeout monitor
        │
        ▼
Server fully operational
```

### What Cannot Be Lost

| State | Durability |
|---|---|
| Job existence and payload | PostgreSQL — survives any restart |
| Job status | PostgreSQL — exact state at time of crash is preserved |
| Attempt history | PostgreSQL — full audit trail survives |
| Worker registrations | PostgreSQL — rebuilt into CRS on restart |
| Dependencies | PostgreSQL — fully preserved |
| Workflow state | PostgreSQL — fully preserved |
| CRS in-memory state | Rebuilt from PostgreSQL on startup — not persisted |

### Edge Case: Server Crashes Mid-Transition

If the server crashes while updating job state (e.g. between setting `status = RUNNING` and incrementing `attempt_count`), the recovery step catches this:

- Job in `RUNNING` state with a worker that is offline → `RecoverJob()` is called
- The attempt is marked `FAILED` and a new attempt is created
- This may cause a job to be executed one extra time in rare crash scenarios — the attempt model tracks this

This is the inherent at-least-once delivery guarantee. Workers should implement idempotent functions where possible.

---

## 8. At-Least-Once Delivery

Orqen provides **at-least-once** job delivery. A job may be executed more than once in failure scenarios (server crash mid-dispatch, network partition, etc.).

**Implication for customers:** Worker functions should be designed to be **idempotent** where possible — executing the same job twice should produce the same outcome without side effects.

This is documented in the SDK spec (`10-SDK-SPEC.md`) and should be clearly communicated to customers.

**Orqen will NOT provide exactly-once delivery in V1.** This is a documented, intentional tradeoff in exchange for simplicity and durability.

---

## 9. Reliability Configuration Reference

All reliability parameters are configurable via environment variables:

| Environment Variable | Default | Description |
|---|---|---|
| `ORQEN_HEARTBEAT_INTERVAL_SECONDS` | `15` | Expected worker heartbeat frequency |
| `ORQEN_HEARTBEAT_TIMEOUT_SECONDS` | `45` | Time before worker is considered dead |
| `ORQEN_MONITOR_INTERVAL_SECONDS` | `15` | How often liveness + timeout monitors run |
| `ORQEN_DEFAULT_MAX_RETRIES` | `3` | Default max retries if not specified per job |
| `ORQEN_DEFAULT_BACKOFF_TYPE` | `exponential` | Default backoff type |
| `ORQEN_DEFAULT_BACKOFF_SECONDS` | `5` | Default base backoff in seconds |

Per-job values always override these defaults.

---

## Related Documents

| Question | Document |
|---|---|
| What are the reliability requirements? | `02-REQUIREMENTS.md` — FR-059 to FR-067, NFR-006 to NFR-009 |
| How does CRS get rebuilt on restart? | `07-CRS-INTEGRATION.md` — BatchAdd on startup |
| What does the attempt schema look like? | `04-DATA-MODEL.md` — job_attempts table |
| How does the worker submit failures? | `05-WORKER-PROTOCOL.md` — Submit Failure |
| How does DAG failure propagation work? | `06-JOB-SCHEDULING.md` — Failure Propagation Policy |
