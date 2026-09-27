# 06 — Job Scheduling

> **Document purpose:** Defines the complete job scheduling subsystem — job lifecycle, the DAG/dependency engine, how jobs move from READY to dispatched, and all scheduling flows.  
> **Read before:** Implementing the scheduler, DAG engine, or any job state transition logic.  
> **Reference when:** Debugging scheduling behavior, adding a new job state, or changing dependency resolution logic.

---

## Overview

The scheduling subsystem answers two questions:

```
1. CAN this job run yet?       → Dependency / DAG Engine
2. WHO should run this job?    → Scheduler + CRS
```

These are two separate concerns handled by two separate components. A job must pass through the dependency engine before it ever reaches the scheduler.

---

## Part 1 — Job Lifecycle

### State Machine (Full)

```
                    ┌──────────┐
                    │ CREATED  │
                    └────┬─────┘
                         │  stored in PostgreSQL
                    ┌────▼─────┐
                    │  QUEUED  │◄─────────────────────────┐
                    └────┬─────┘                          │
                         │                                │
          ┌──────────────▼───────────────┐                │
          │                              │                │
    ┌─────▼────┐              ┌──────────▼─────────────┐  │
    │  READY   │              │  WAITING_FOR_DEPENDENCY │  │
    └─────┬────┘              └─────────────────────────┘  │
          │                           │                    │
          │                  all deps │                    │
          │                 satisfied │                    │
          │                   ┌───────┘                    │
          │                   │                            │
    ┌─────▼──────────────────▼┐                           │
    │          READY          │                           │
    └─────────────┬───────────┘                           │
                  │  scheduler dispatches                  │
            ┌─────▼────┐                                  │
            │ RUNNING  │                                  │
            └──┬─────┬─┘                                  │
               │     │                                    │
         ┌─────▼──┐  └──────────────┐                    │
         │COMPLETED│         ┌──────▼───┐                 │
         └─────────┘         │  FAILED  │                 │
                             └─────┬────┘                 │
                                   │ retries remaining     │
                            ┌──────▼────┐                 │
                            │ RETRYING  ├─────────────────┘
                            └───────────┘
                                   │ retries exhausted
                            stays FAILED (terminal)

         ┌──────────┐
         │CANCELLED │ ← from QUEUED or READY only (terminal)
         └──────────┘
```

### State Definitions

| Status | Terminal? | Description |
|---|---|---|
| `CREATED` | No | Job accepted and stored in PostgreSQL |
| `QUEUED` | No | Waiting for dependency check and scheduling |
| `WAITING_FOR_DEPENDENCY` | No | Has unresolved upstream jobs; blocked from scheduling |
| `READY` | No | All dependencies satisfied; eligible for scheduling |
| `RUNNING` | No | Dispatched to a worker; execution in progress |
| `COMPLETED` | **Yes** | Worker returned a successful result |
| `FAILED` | **Yes** | All retry attempts exhausted with failures |
| `RETRYING` | No | Failed attempt; waiting for backoff period before re-queuing |
| `CANCELLED` | **Yes** | Explicitly cancelled before execution began |

### Terminal States
`COMPLETED`, `FAILED`, and `CANCELLED` are terminal — a job in one of these states never transitions to any other state.

### Transition Rules

| From | To | Condition |
|---|---|---|
| `CREATED` | `QUEUED` | Always — immediately on store |
| `QUEUED` | `READY` | No dependencies declared |
| `QUEUED` | `WAITING_FOR_DEPENDENCY` | Has one or more unresolved dependency edges |
| `QUEUED` | `CANCELLED` | Explicit cancel request |
| `READY` | `CANCELLED` | Explicit cancel request |
| `READY` | `RUNNING` | Scheduler acquires a worker and dispatches |
| `RUNNING` | `COMPLETED` | Worker submits `POST /jobs/:id/result` |
| `RUNNING` | `FAILED` | Worker submits failure AND `attempt_count >= max_retries` |
| `RUNNING` | `RETRYING` | Worker submits failure AND `attempt_count < max_retries` |
| `RETRYING` | `QUEUED` | Backoff period expires |
| `WAITING_FOR_DEPENDENCY` | `READY` | All upstream dependency edges are satisfied |

---

## Part 2 — Dependency & DAG Engine

### What Is a Dependency?

A dependency is a declared edge between two jobs:

```
upstream_job_id  →  downstream_job_id
```

It means: **the downstream job cannot become `READY` until the upstream job is `COMPLETED`.**

Dependencies are stored in the `job_dependencies` table as individual edges. A job can have multiple upstream dependencies.

### Supported DAG Patterns

**Linear chain:**
```
A → B → C → D
```
Each job waits for the previous one to complete.

**Fan-out (parallel execution):**
```
     A
    / \
   B   C
```
B and C both depend on A. After A completes, B and C become `READY` simultaneously and execute in parallel.

**Fan-in (join):**
```
  B
   \
    D
   /
  C
```
D depends on both B and C. D only becomes `READY` after both B and C complete.

**Full DAG (fan-out + fan-in):**
```
      A
     / \
    B   C
     \ /
      D
```
A completes → B and C become READY and run in parallel → both complete → D becomes READY.

### Dependency Creation Rules

When a customer declares a dependency, the DAG engine must:

1. Validate that both `upstream_job_id` and `downstream_job_id` exist in the same project
2. Run **cycle detection** — traverse the existing graph to ensure adding this edge does not create a cycle
3. If cyclic: reject with `400 Bad Request`
4. If valid: insert the `job_dependencies` row and set downstream job status to `WAITING_FOR_DEPENDENCY`

### Cycle Detection Algorithm

On each new dependency edge `(U → D)`:
- Perform a depth-first traversal starting from `D`
- Follow existing downstream edges recursively
- If `U` is encountered during traversal: the new edge would create a cycle — reject it
- If traversal completes without encountering `U`: the edge is safe — insert it

This is evaluated at write time, not at execution time.

### Dependency Resolution Flow

When any job transitions to `COMPLETED`:

```
Job X → COMPLETED
        │
        ▼
DAG Engine: find all dependency edges WHERE upstream_job_id = X.id
        │
        ▼
For each edge, mark is_satisfied = true, satisfied_at = now()
        │
        ▼
For each affected downstream_job_id:
        │
        ▼
COUNT unsatisfied edges WHERE downstream_job_id = affected AND is_satisfied = false
        │
        ▼
Count = 0?
   │           │
  YES          NO
   │           │
   ▼           ▼
Downstream   Downstream stays
job → READY  WAITING_FOR_DEPENDENCY
   │
   ▼
Enters normal scheduling pipeline
(Scheduler → CRS → Worker)
```

### Failure Propagation Policy

When an upstream job reaches `FAILED` (terminal), downstream jobs that were `WAITING_FOR_DEPENDENCY` on it must be handled. V1 policy:

| Policy | Behavior |
|---|---|
| **Block** *(V1 default)* | Downstream job transitions to `FAILED` with error `"upstream job {id} failed"`. The failure propagates down the entire DAG chain. |

This policy is applied by the DAG engine when a job transitions to terminal `FAILED` state. Future versions may support configurable policies (e.g. skip, continue).

### Workflow Completion Tracking

A workflow (DAG) is considered `COMPLETED` when all jobs associated with the `workflow_id` have reached a terminal state (`COMPLETED`, `FAILED`, or `CANCELLED`). The `workflows.status` is updated accordingly:

- All jobs `COMPLETED` → workflow `COMPLETED`
- Any job `FAILED` → workflow `FAILED`
- All jobs `CANCELLED` → workflow `CANCELLED`

---

## Part 3 — Scheduler

### Scheduler Responsibility

The scheduler's only job is:

> Find a `READY` job and dispatch it to the best available worker via CRS.

It does not decide if a job *can* run — that is the DAG engine's responsibility. It only handles jobs that are already `READY`.

### Scheduler Trigger Points

The scheduler is triggered in three scenarios:

| Trigger | Description |
|---|---|
| Job transitions to `READY` | Immediately attempt dispatch |
| Worker comes online (register or reconnect) | Check for pending `READY` jobs for this worker's capabilities |
| Worker frees a slot (`Include` in CRS) | Check for pending `READY` jobs for this worker's capabilities |

### Scheduling Flow (Detailed)

```
Job status → READY
        │
        ▼
Scheduler picks up job (job_type, project_id, priority)
        │
        ▼
Look up CRS registry:
  key = (project_id, job_type)
  instance = registry[key]
        │
        ▼
instance exists?
   │           │
  YES          NO
   │           │
   ▼           ▼
sched.Acquire()  No workers registered
   │             for this capability →
   │             job stays READY, will
   │             retry when a worker
   │             registers for this type
   │
   ▼
Worker acquired?
   │              │
  YES             NO (ErrNoResourceAvailable)
   │              │
   ▼              ▼
Dispatch job   Job stays READY
to worker      Will be dispatched when
               a worker becomes available
   │
   ▼
Job status → RUNNING
worker_id set on job
new job_attempt row created
worker.current_job_count incremented
   │
   ▼
If worker.current_job_count == worker.concurrency:
  sched.Exclude(workerID) on ALL capability instances
   │
   ▼
Job delivered via long-poll response
```

### No-Worker Scenario

If `sched.Acquire()` returns `ErrNoResourceAvailable` (all workers at capacity or no workers registered), the job remains `READY` in PostgreSQL. It will be picked up when:
- A worker frees a slot and `Include` is called → scheduler re-checks pending READY jobs
- A new worker registers for the required capability → scheduler checks READY jobs for that capability

This is a pull-based, event-driven approach — no busy polling loop needed.

### Priority Handling

Within a capability pool, the CRS `Comparator` determines worker priority. For job-side priority, the scheduler processes `READY` jobs ordered by:

```sql
ORDER BY priority DESC, created_at ASC
```

Higher priority jobs are dispatched first. Ties are broken by creation time (FIFO).

### Multiple Workers, Same Capability

```
READY job: generate_pdf
        │
        ▼
CRS instance: (proj_1, generate_pdf)
  Worker 2 — 3 free slots   ← highest priority (most free slots)
  Worker 5 — 1 free slot
  Worker 1 — 2 free slots
        │
        ▼
sched.Acquire() → Worker 2 (Comparator selects most available)
```

---

## Part 4 — CRS Registry Management

The scheduler maintains a registry map in memory:

```
registry: map[(project_id, capability)] → *crs.Scheduler
```

### Registry Operations

| Event | Registry Action |
|---|---|
| Worker registers with capability X | `registry[(proj, X)].Add(worker)` — create instance if not exists |
| Worker goes OFFLINE | `registry[(proj, X)].Remove(workerID)` for each capability X |
| Worker reaches concurrency limit | `registry[(proj, X)].Exclude(workerID)` for ALL capabilities |
| Worker frees a slot | `registry[(proj, X)].Include(workerID)` for ALL capabilities |
| Server restarts | Rebuild all instances from PostgreSQL via `BatchAdd` |
| Job needs dispatch | `registry[(proj, job_type)].Acquire()` |

### CRS Comparator for Workers

The Comparator is defined by Orqen. Workers are ranked by available concurrency slots:

```go
Comparator: func(a, b *WorkerNode) int {
    aFree := a.Concurrency - a.CurrentJobCount
    bFree := b.Concurrency - b.CurrentJobCount
    if aFree > bFree { return -1 }  // a is better
    if aFree < bFree { return 1 }   // b is better
    return 0
},
```

Worker with the most free slots is always at the top of the heap within its shard.

### Acquire Policy

`Exclusive` — when a worker is acquired for a job, it is popped from the heap and placed in the inactive store. It is returned to the heap (via `Release` / `Include`) when it completes a job or frees a slot.

This prevents the same worker from being double-dispatched beyond its concurrency limit.

> Full CRS integration details in `07-CRS-INTEGRATION.md`.

---

## Part 5 — Complete End-to-End Scheduling Flow

### Simple Job (No Dependencies)

```
POST /jobs/enqueue { type: "send_email", payload: {...} }
        │
        ▼
Job stored: status = QUEUED
        │
        ▼
Dependency check: no dependencies declared
        │
        ▼
Job status → READY
        │
        ▼
Scheduler triggered
        │
        ▼
CRS[(proj_1, "send_email")].Acquire() → Worker 3
        │
        ▼
Long-poll response delivers job to Worker 3
        │
        ▼
Job status → RUNNING
        │
        ▼
Worker 3 executes send_email(payload)
        │
        ▼
POST /worker/jobs/:id/result
        │
        ▼
Job status → COMPLETED
Worker 3 slot freed → sched.Include if was excluded
DAG engine checks for downstream jobs
```

### DAG Job (With Dependencies)

```
Enqueue A (no deps)     → status: READY
Enqueue B (depends: A)  → status: WAITING_FOR_DEPENDENCY
Enqueue C (depends: A)  → status: WAITING_FOR_DEPENDENCY
Enqueue D (depends: B, C) → status: WAITING_FOR_DEPENDENCY

Scheduler dispatches A → Worker 1
A completes
        │
        ▼
DAG engine: A's edges satisfied
  B: all deps satisfied? YES → B → READY
  C: all deps satisfied? YES → C → READY
        │
        ▼
Scheduler dispatches B → Worker 2
Scheduler dispatches C → Worker 3  (parallel)
        │
        ▼
B completes
DAG engine: B's edges for D: is_satisfied = true
D: all deps satisfied? NO (C still running) → stays WAITING

C completes
DAG engine: C's edges for D: is_satisfied = true
D: all deps satisfied? YES → D → READY
        │
        ▼
Scheduler dispatches D → Worker 1
D completes
        │
        ▼
Workflow → COMPLETED
```

---

## Related Documents

| Question | Document |
|---|---|
| What are the requirements this subsystem satisfies? | `02-REQUIREMENTS.md` — FR-039 to FR-058 |
| How does CRS worker selection work in detail? | `07-CRS-INTEGRATION.md` |
| What does the job and dependency schema look like? | `04-DATA-MODEL.md` |
| How does the worker receive the dispatched job? | `05-WORKER-PROTOCOL.md` — Poll for Job |
| How does failure handling interact with scheduling? | `08-RELIABILITY.md` |
