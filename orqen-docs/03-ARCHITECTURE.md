# 03 — Architecture

> **Document purpose:** Defines the full system architecture — all layers, components, their responsibilities, boundaries, and how they connect.  
> **Read before:** Any implementation work on any layer.  
> **Reference when:** Adding a new component, understanding where a piece of logic lives, or reviewing a cross-layer change.

---

## System Overview

```
                          ORQEN DEPLOYMENT
                                 |
                     +-----------+-----------+
                     |                       |
                Project 1                Project 2
                     |                       |
           +---------+---------+     +-------+-------+
           |         |         |     |       |       |
         Jobs     Workers     DAGs   Jobs  Workers  DAGs
           |         |         |     |       |       |
           +---------+---------+     +-------+-------+
                     |                       |
                     +-----------+-----------+
                                 |
                            Orqen Core
                                 |
                  +--------------+--------------+
                  |              |              |
             Job System      Scheduler        DAG
                  |              |              |
                  |             CRS             |
                  |              |              |
                  +--------------+--------------+
                                 |
                            Worker Protocol
                                 |
                     +-----------+-----------+
                     |           |           |
                  Node.js     Python       Go
                   Worker      Worker      Worker
                     |           |           |
                     +-----------+-----------+
                                 |
                        Customer Functions
```

---

## Architectural Layers

Orqen is composed of 7 logical layers plus the data layer and the dashboard.

| Layer | Name | Primary Responsibility |
|---|---|---|
| Layer 1 | Project & Control | Project lifecycle, API keys, authentication, isolation boundary |
| Layer 2 | Job System | Job storage, state machine, attempt tracking |
| Layer 3 | Worker System | Worker registration, heartbeat, concurrency tracking |
| Layer 4 | Worker Communication | Long-poll protocol, job delivery, result submission |
| Layer 5 | Scheduler + CRS | Worker pool management, best-worker selection |
| Layer 6 | Dependency / DAG Engine | Dependency tracking, DAG validation, job release |
| Layer 7 | Reliability | Retry logic, failure detection, crash recovery |
| Data | PostgreSQL | Durable source of truth for all persistent state |
| UI | Dashboard | Web UI — observability and management |

---

## Layer 1 — Project & Control

### Responsibility
Manages the top-level isolation unit: the **Project**. Handles authentication and establishes the project context for every inbound request.

### What it does
- Create, configure, and delete projects
- Generate, revoke, and rotate API keys
- Authenticate every API request via API key
- Resolve project context from API key (never from client-supplied project ID)
- Enforce that all downstream operations are scoped to the authenticated project

### What it does NOT do
- Execute jobs
- Manage workers
- Touch the scheduler

### Key rule
> The server derives project context from the authenticated API key. A client-supplied `project_id` is never trusted.

### Components
```
API Key Store (PostgreSQL)
    ↓
Auth Middleware
    ↓
Project Context (injected into every request)
```

---

## Layer 2 — Job System

### Responsibility
Manages the lifecycle of every job from enqueue to completion.

### What it does
- Accept job enqueue requests
- Store jobs durably in PostgreSQL
- Manage job status transitions through the formal state machine
- Track per-job attempt logs
- Support job cancellation
- Store results and error information

### Job State Machine

```
                    ┌──────────┐
                    │ CREATED  │
                    └────┬─────┘
                         │
                    ┌────▼─────┐
                    │  QUEUED  │◄─────────────────────┐
                    └────┬─────┘                      │
                         │                            │
          ┌──────────────▼──────────────┐             │
          │                             │             │
    ┌─────▼────┐             ┌──────────▼──────────┐  │
    │  READY   │             │ WAITING_FOR_        │  │
    └─────┬────┘             │ DEPENDENCY          │  │
          │                  └─────────────────────┘  │
    ┌─────▼────┐                                       │
    │ RUNNING  │                                       │
    └──┬────┬──┘                                       │
       │    │                                          │
  ┌────▼──┐ └──────────────┐                          │
  │COMPLE-│         ┌──────▼───┐                      │
  │ TED   │         │  FAILED  │                      │
  └───────┘         └─────┬────┘                      │
                          │                           │
                   ┌──────▼────┐                      │
                   │ RETRYING  ├──────────────────────┘
                   └───────────┘

              ┌──────────┐
              │CANCELLED │ ← from QUEUED or READY only
              └──────────┘
```

### Valid State Transitions

| From | To | Trigger |
|---|---|---|
| `CREATED` | `QUEUED` | Job accepted and stored |
| `QUEUED` | `READY` | No dependencies, or all dependencies satisfied |
| `QUEUED` | `WAITING_FOR_DEPENDENCY` | Has unresolved upstream dependencies |
| `QUEUED` | `CANCELLED` | Explicit cancellation request |
| `READY` | `CANCELLED` | Explicit cancellation request |
| `READY` | `RUNNING` | Scheduler dispatches to a worker |
| `RUNNING` | `COMPLETED` | Worker submits success result |
| `RUNNING` | `FAILED` | Worker submits failure, retries exhausted |
| `RUNNING` | `RETRYING` | Worker submits failure, retries remaining |
| `RETRYING` | `QUEUED` | Backoff period elapsed, re-enters queue |
| `WAITING_FOR_DEPENDENCY` | `READY` | All upstream dependencies completed |

---

## Layer 3 — Worker System

### Responsibility
Manages the lifecycle of workers — registration, liveness, and concurrency state.

### What it does
- Store worker registrations in PostgreSQL
- Track worker capabilities per project
- Track current job count vs concurrency limit
- Maintain worker status: `IDLE`, `BUSY`, `OFFLINE`
- Detect dead workers via missed heartbeats
- Trigger job recovery when a worker is marked dead
- Maintain CRS registry — add/remove workers from the correct CRS instances when they register or go offline

### Worker Status Transitions

```
Register
   ↓
IDLE  ←────────────────────────────────┐
   │                                   │
   │  job dispatched                   │  job completed / slot freed
   ▼                                   │
BUSY ──────────────────────────────────┘
   │
   │  all slots occupied → Exclude from CRS pools
   │  slot freed         → Include in CRS pools
   │
   │  heartbeat missed
   ▼
OFFLINE → jobs recovered → worker removed from CRS
```

### CRS Interaction
- On worker register: `sched.Add(worker)` on each capability's CRS instance
- On worker offline: `sched.Remove(workerID)` on each capability's CRS instance
- On concurrency limit reached: `sched.Exclude(workerID)` on ALL capability instances for that worker
- On slot freed: `sched.Include(workerID)` on ALL capability instances for that worker

---

## Layer 4 — Worker Communication

### Responsibility
Handles the language-independent HTTP protocol between Orqen and customer workers.

### Transport
HTTP long polling. The worker sends a `GET /worker/jobs/poll` request. The server holds the connection open until a job is available or a timeout is reached, then returns the job. This works with any language that supports HTTP.

### Protocol Flow

```
Worker                           Orqen Server
  │                                   │
  │── GET /worker/poll ───────────────►│
  │                                   │  (holds connection)
  │                                   │  (job becomes available)
  │◄── 200 { job } ───────────────────│
  │                                   │
  │── POST /worker/jobs/:id/ack ──────►│
  │                                   │
  │  [executes customer function]     │
  │                                   │
  │── POST /worker/jobs/:id/result ───►│
  │                                   │
  │── GET /worker/poll ───────────────►│  (immediately re-polls)
  │                                   │
```

### Protocol Operations

| Operation | Method | Path | Direction |
|---|---|---|---|
| Register Worker | `POST` | `/worker/register` | Worker → Server |
| Heartbeat | `POST` | `/worker/heartbeat` | Worker → Server |
| Poll for Job | `GET` | `/worker/poll` | Worker → Server |
| Acknowledge Job | `POST` | `/worker/jobs/:id/ack` | Worker → Server |
| Submit Success | `POST` | `/worker/jobs/:id/result` | Worker → Server |
| Submit Failure | `POST` | `/worker/jobs/:id/failure` | Worker → Server |
| Graceful Shutdown | `POST` | `/worker/shutdown` | Worker → Server |

> Full request/response shapes are defined in `05-WORKER-PROTOCOL.md`.

---

## Layer 5 — Scheduler & CRS

### Responsibility
Selects the best eligible worker for a ready job.

### What it does
- Maintain the CRS registry: `map[(project_id, capability)] → *crs.Scheduler`
- When a job becomes `READY`: look up the CRS instance for `(job.project_id, job.job_type)`
- Call `sched.Acquire()` to get the best available worker
- Dispatch the job to that worker via the long-poll connection
- After dispatch: update worker's current job count, call `Exclude` if at capacity

### CRS Library
Worker selection is powered by `github.com/phero20/concurrent-resource-scheduler`.
- One `*crs.Scheduler` instance per `(project_id, capability)`
- Instances are lazily initialized when a capability is first registered
- Workers with multiple capabilities are registered in multiple instances
- `Exclusive` acquire policy — worker is popped from heap on acquire, pushed back on release
- `Comparator` defined by Orqen — sorts workers by available concurrency slots (most free slots = highest priority)

> Full CRS integration details in `07-CRS-INTEGRATION.md`.

### Scheduling Flow

```
Job status → READY
       │
       ▼
Scheduler picks up job
       │
       ▼
Look up CRS instance for (project_id, job_type)
       │
       ▼
sched.Acquire() → best available worker
       │
       ▼
Dispatch job to worker via long poll
       │
       ▼
Update worker job count in PostgreSQL
       │
       ▼
If worker at concurrency limit:
  sched.Exclude(workerID) on ALL capability instances
```

---

## Layer 6 — Dependency / DAG Engine

### Responsibility
Manages job dependencies and DAG-based workflows.

### What it does
- Accept dependency declarations at job creation time
- Validate the dependency graph — reject cyclic graphs immediately
- Set jobs with unresolved dependencies to `WAITING_FOR_DEPENDENCY`
- Monitor upstream job completions
- When all upstreams complete: transition downstream job to `READY`
- Track workflow-level completion state
- Enforce failure propagation policy

### DAG Patterns Supported

**Linear chain:**
```
A → B → C
```

**Fan-out / Fan-in:**
```
    A
   / \
  B   C
   \ /
    D
```

### Dependency Resolution Flow

```
Upstream job COMPLETED
        │
        ▼
DAG engine checks all dependency edges for downstream job
        │
        ▼
All edges satisfied?
   │              │
  YES             NO
   │              │
   ▼              ▼
Job → READY    Job stays WAITING_FOR_DEPENDENCY
   │
   ▼
Enters normal scheduling pipeline
(Scheduler → CRS → Worker)
```

### Key Rule
The DAG engine does not have its own execution path. Once a job transitions to `READY`, it flows through the standard scheduling pipeline.

---

## Layer 7 — Reliability

### Responsibility
Ensures jobs are not permanently lost due to worker failures, server crashes, or network issues.

### What it does
- Retry failed jobs up to `maxRetries` with configurable `backoff`
- Detect dead workers via heartbeat timeout
- Recover in-flight jobs from dead workers and requeue them
- Handle late results (ignore results for superseded attempts)
- Handle duplicate results (idempotent result processing)
- On server restart: reload job state from PostgreSQL, rebuild CRS from active workers

### Failure Scenarios

| Scenario | Detection | Recovery |
|---|---|---|
| Worker crashes mid-job | Heartbeat timeout | Mark worker OFFLINE, recover jobs, requeue |
| Worker submits failure | Explicit failure submission | Increment attempt count, retry if under limit |
| Job times out | `timeout` field exceeded | Treated as failure, retry logic applies |
| Server crashes | — | On restart, load PostgreSQL state, rebuild CRS |
| Network failure causes duplicate result | Attempt ID mismatch | Ignore result for non-current attempt |
| Worker at capacity | Real-time concurrency check | Exclude from CRS pools, re-include when slot frees |

> Full reliability design in `08-RELIABILITY.md`.

---

## Data Layer — PostgreSQL

### Responsibility
Single durable source of truth for all persistent state.

### What is stored

| Entity | Description |
|---|---|
| `projects` | All projects in the deployment |
| `api_keys` | Hashed API keys scoped to projects |
| `workers` | Registered worker instances |
| `worker_capabilities` | Which job types each worker handles |
| `jobs` | All job records with full metadata |
| `job_attempts` | Individual execution attempts per job |
| `dependencies` | Job-to-job dependency edges |
| `workflows` | DAG/workflow definitions and state |

### CRS vs PostgreSQL

| System | Answers |
|---|---|
| PostgreSQL | What exists? What happened? What is the current state? |
| CRS | Which available worker should receive this job right now? |

CRS is purely in-memory. PostgreSQL is the source of truth. On restart, Orqen rebuilds CRS state from PostgreSQL.

> Full schema in `04-DATA-MODEL.md`.

---

## Complete Job Execution Flow

End-to-end from enqueue to completion:

```
Customer Application
        │
        │  POST /jobs/enqueue { type: "generate_pdf", payload: {...} }
        ▼
Layer 1: Auth Middleware
        │  API key → project_id resolved
        ▼
Layer 2: Job System
        │  Job stored in PostgreSQL (status: QUEUED)
        ▼
Layer 6: DAG Engine
        │  No dependencies? → status: READY
        │  Has dependencies? → status: WAITING_FOR_DEPENDENCY
        ▼
Layer 5: Scheduler
        │  Picks up READY job
        │  Looks up CRS[(project_id, "generate_pdf")]
        │  sched.Acquire() → Worker 2
        ▼
Layer 4: Worker Communication
        │  Job delivered to Worker 2 via long poll response
        ▼
Customer Worker (Worker 2)
        │  generate_pdf(payload) executes locally
        │  POST /worker/jobs/:id/result { result: {...} }
        ▼
Layer 2: Job System
        │  Job status → COMPLETED
        │  Result stored in PostgreSQL
        ▼
Layer 6: DAG Engine
        │  Check if any downstream jobs were waiting on this job
        │  If yes → transition them to READY
        ▼
Layer 3: Worker System
        │  Worker 2 job count decremented
        │  If below concurrency limit → sched.Include(worker2)
        ▼
PostgreSQL updated. Dashboard reflects COMPLETED.
```

---

## Component Boundaries — What Each Layer Must NOT Do

| Layer | Must NOT |
|---|---|
| Layer 1 (Project) | Execute jobs, touch scheduler, manage workers |
| Layer 2 (Job System) | Select workers, manage dependencies beyond state transitions |
| Layer 3 (Worker System) | Execute jobs, make routing decisions |
| Layer 4 (Protocol) | Store state, make scheduling decisions |
| Layer 5 (Scheduler/CRS) | Persist to PostgreSQL, authenticate requests, manage DAGs |
| Layer 6 (DAG Engine) | Execute jobs, select workers |
| Layer 7 (Reliability) | Make scheduling decisions, manage projects |
| CRS library | Know about PostgreSQL, job types, project isolation |

---

## Project Isolation Enforcement Points

Isolation is enforced at every layer — not just at the API boundary.

```
API Layer          → API key resolves project_id; all routes are project-scoped
Database Layer     → Every query includes WHERE project_id = $1
Worker Layer       → Workers are registered under a project; only receive that project's jobs
Scheduler Layer    → CRS instances are keyed by (project_id, capability)
Dashboard Layer    → Session scoped to a project; cross-project data never returned
```

> Full isolation rules in `11-PROJECT-ISOLATION.md`.

---

## Related Documents

| Question | Document |
|---|---|
| What are the requirements each layer must satisfy? | `02-REQUIREMENTS.md` |
| What does the database schema look like? | `04-DATA-MODEL.md` |
| How does the worker protocol work in detail? | `05-WORKER-PROTOCOL.md` |
| How does scheduling and CRS work in detail? | `06-JOB-SCHEDULING.md` + `07-CRS-INTEGRATION.md` |
| How does reliability work in detail? | `08-RELIABILITY.md` |
| How is isolation enforced at every layer? | `11-PROJECT-ISOLATION.md` |
