# Orqen V1 — Comprehensive Project Specification

> **Version:** 1.0.0  
> **Status:** Pre-Implementation  
> **Type:** Self-Hosted Job Processing & Workflow Platform  

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Problem Statement](#2-problem-statement)
3. [Product Vision & Goals](#3-product-vision--goals)
4. [Scope — What Orqen Is and Is Not](#4-scope--what-orqen-is-and-is-not)
5. [System Architecture Overview](#5-system-architecture-overview)
6. [Deployment Model](#6-deployment-model)
7. [Core Mental Model](#7-core-mental-model)
8. [Layer 1 — Project & Control Layer](#8-layer-1--project--control-layer)
9. [Layer 2 — Job System](#9-layer-2--job-system)
10. [Layer 3 — Worker System](#10-layer-3--worker-system)
11. [Layer 4 — Worker Communication Protocol](#11-layer-4--worker-communication-protocol)
12. [Layer 5 — Scheduler & CRS](#12-layer-5--scheduler--crs)
13. [Layer 6 — Dependency & DAG Engine](#13-layer-6--dependency--dag-engine)
14. [Layer 7 — Reliability & Failure Handling](#14-layer-7--reliability--failure-handling)
15. [Data Layer — PostgreSQL](#15-data-layer--postgresql)
16. [Project Isolation Model](#16-project-isolation-model)
17. [Complete Job Execution Flow](#17-complete-job-execution-flow)
18. [Complete DAG Execution Flow](#18-complete-dag-execution-flow)
19. [Dashboard](#19-dashboard)
20. [Customer Experience — End-to-End](#20-customer-experience--end-to-end)
21. [Key Design Decisions & Tradeoffs](#21-key-design-decisions--tradeoffs)
22. [Glossary](#22-glossary)

---

## 1. Executive Summary

Orqen is a **self-hostable, multi-project, language-independent job processing and workflow platform**.

It allows engineering teams to deploy a single Orqen instance on their own infrastructure and use it to power background job processing across multiple independent applications — each isolated from the other — without being locked into a specific programming language or runtime.

Orqen is conceptually similar to platforms like BullMQ or Kafka-based job systems, but its differentiating characteristics are:

| Capability | Description |
|---|---|
| **Language Independence** | Workers can be written in Node.js, Python, Go, Java, or any language that can speak HTTP |
| **Multi-Project Isolation** | Multiple applications share one deployment without seeing each other's data |
| **Durable PostgreSQL-Backed Jobs** | No Redis dependency; jobs survive server restarts |
| **Long-Poll Worker Communication** | Workers pull jobs over persistent HTTP connections |
| **Dependency-Based Execution** | Jobs can declare dependencies on other jobs before becoming eligible |
| **DAG/Workflow Support** | Complex multi-step workflows with branching and merging |
| **CRS Scheduling** | Powered by [concurrent-resource-scheduler](https://github.com/phero20/concurrent-resource-scheduler) — an external Go library (also authored by this project) providing sharded priority heaps, pluggable acquire strategies, and affinity routing for high-throughput, low-contention worker selection |
| **Built-in Dashboard** | Web UI for managing all projects, jobs, workers, and workflows |

---

## 2. Problem Statement

Most background job processing systems have at least one of these limitations:

- **Language lock-in** — BullMQ requires Node.js; Celery requires Python. Teams using polyglot stacks have no unified option.
- **No multi-tenancy** — Running separate processing infrastructure per application is expensive and operationally heavy.
- **Volatile state** — Redis-backed queues lose state under failure without additional safeguards.
- **No dependency modeling** — Simple FIFO queues cannot express "Job B should only run after Job A succeeds."
- **No built-in workflow orchestration** — DAG-based workflows require plugging in separate tools like Airflow or Temporal.

Orqen solves all five in a single self-hosted deployment.

---

## 3. Product Vision & Goals

### Vision

> Enable any engineering team — regardless of language or infrastructure — to manage durable, observable, dependency-aware background jobs through a single self-hosted platform.

### V1 Goals

- **G1** — Support multiple logically isolated projects inside one deployment
- **G2** — Workers can be written in any language
- **G3** — All job state is durably stored in PostgreSQL
- **G4** — Jobs can declare dependencies on other jobs (DAG support)
- **G5** — A built-in scheduler selects the best worker using CRS
- **G6** — Workers communicate with the server over long-polling
- **G7** — Failures are detected and jobs are retried automatically
- **G8** — A web dashboard provides real-time observability
- **G9** — The customer never exposes their function code to Orqen

### Non-Goals for V1

- No CPU/GPU resource-aware scheduling
- No external trigger types (webhooks, cron, events)
- No plugin ecosystem
- No multi-region replication
- No built-in secret management
- No function code execution inside the Orqen server

---

## 4. Scope — What Orqen Is and Is Not

### What Orqen Does

```
✅ Store jobs durably in PostgreSQL
✅ Manage job state and lifecycle
✅ Manage job dependencies
✅ Find eligible workers for a job
✅ Schedule jobs to the best available worker (via CRS)
✅ Deliver jobs to workers over long polling
✅ Receive results from workers
✅ Handle retries and failure recovery
✅ Manage projects and credentials
✅ Provide a web dashboard
```

### What Orqen Does NOT Do

```
❌ Execute customer functions itself
❌ Require workers to be written in one language
❌ Require customer applications to run inside Orqen
❌ Expose customer function source code
❌ Operate as a general-purpose event streaming platform
❌ Perform CPU/GPU resource-aware scheduling (V1)
❌ Provide a plugin ecosystem (V1)
```

---

## 5. System Architecture Overview

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

### Architectural Layers

| Layer | Name | Responsibility |
|---|---|---|
| Layer 1 | Project & Control | Project creation, API keys, project isolation |
| Layer 2 | Job System | Job storage, state management, job lifecycle |
| Layer 3 | Worker System | Worker registration, heartbeats, concurrency tracking |
| Layer 4 | Worker Communication | Long-poll protocol, job delivery, result submission |
| Layer 5 | Scheduler + CRS | Worker pool management, best-worker selection via the [concurrent-resource-scheduler](https://github.com/phero20/concurrent-resource-scheduler) Go library |
| Layer 6 | Dependency / DAG Engine | Dependency tracking, DAG validation, job release |
| Layer 7 | Reliability | Retries, failure detection, worker crash recovery |
| Data | PostgreSQL | Durable source of truth for all persistent state |
| UI | Dashboard | Web UI for observability and management |

---

## 6. Deployment Model

Orqen is entirely **self-hosted**. The customer deploys the entire platform on their own infrastructure.

```
Customer Server

Orqen
├── API Server
├── Scheduler
├── Worker Management
├── DAG Engine
├── Dashboard
├── PostgreSQL
└── CRS
```

Inside a single deployment, the customer creates multiple **projects**:

```
One Orqen Deployment
│
├── Project: E-Commerce
├── Project: SaaS Application
├── Project: Mobile Backend
└── Project: Internal Tools
```

Each project is a logically isolated workspace. Projects cannot access each other's data, workers, jobs, API keys, or workflows.

---

## 7. Core Mental Model

The entire system can be understood through five sequential questions:

```
┌─────────────────────────────────────────────────────────┐
│                                                         │
│  1. WHAT PROJECT DOES THIS BELONG TO?                   │
│     → Project Layer + API Key Authentication            │
│                                                         │
│  2. WHAT NEEDS TO HAPPEN?                               │
│     → Job (type + payload + priority + metadata)        │
│                                                         │
│  3. CAN IT RUN YET?                                     │
│     → Dependency Engine / DAG (WAITING or READY?)       │
│                                                         │
│  4. WHO SHOULD RUN IT?                                  │
│     → Scheduler + CRS (find best eligible worker)       │
│                                                         │
│  5. HOW DOES IT ACTUALLY RUN?                           │
│     → Worker + Long Polling + Customer Function         │
│                                                         │
└─────────────────────────────────────────────────────────┘

PostgreSQL holds the durable state of the entire system.
```

---

## 8. Layer 1 — Project & Control Layer

### Purpose

This layer manages the top-level organizational unit: the **Project**. It handles who owns resources and which project a request belongs to.

### Project Model

A project is an isolated workspace inside one Orqen deployment.

```
Project
├── Project ID         (e.g., proj_123)
├── Name               (e.g., "My E-Commerce")
├── API Keys
├── Workers
├── Worker Capabilities
├── Jobs
├── Job Attempts
├── Dependencies
└── Workflows / DAGs
```

### API Keys

When a project is created, Orqen generates a project-scoped API key.

```
Project ID:  proj_123
API Key:     orqen_live_xxxxxxxxxxxxxxxxxx
```

The customer uses the key to initialize their SDK:

```js
const orqen = new Orqen({
    apiKey: "orqen_live_xxxxx"
});
```

The API key is the authentication boundary. Every inbound request is authenticated against its API key, which resolves to a project. **The server never trusts a project ID supplied directly by the client** — the project context is always derived from the authenticated key.

### Responsibilities

| Responsibility | Description |
|---|---|
| Create project | Initialize a new isolated project |
| Delete project | Remove a project and all its data |
| Project configuration | Update project settings |
| API key generation | Issue new keys for a project |
| API key revocation | Invalidate a key immediately |
| API key rotation | Replace an existing key with a new one |
| Project-level authorization | Ensure all requests are scoped to the correct project |
| Project isolation | Enforce that Project A cannot access Project B resources |

### What This Layer Does NOT Do

It does not execute jobs. It only establishes:

```
WHO owns the resources?
WHICH project does the resource belong to?
```

---

## 9. Layer 2 — Job System

### What Is a Job?

A **Job** represents one unit of work that needs to be executed.

```
Job Type:  send_email
Payload:
{
    "to": "user@example.com",
    "template": "welcome"
}
```

### Job Data Model

| Field | Type | Description |
|---|---|---|
| `job_id` | string | Unique identifier |
| `project_id` | string | Owning project |
| `job_type` | string | The kind of work (maps to a worker capability) |
| `payload` | JSON | Input data for the worker function |
| `priority` | integer | Scheduling priority |
| `status` | enum | Current state in the lifecycle |
| `attempts` | integer | How many times execution has been attempted |
| `max_retries` | integer | Maximum number of retries allowed |
| `created_at` | timestamp | When the job was enqueued |
| `updated_at` | timestamp | Last state change |
| `result` | JSON | Output from a successful execution |
| `error` | string | Error message from a failed execution |

### Job Lifecycle — State Machine

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
    │  READY   │             │ WAITING_FOR_         │  │
    └─────┬────┘             │ DEPENDENCY          │  │
          │                  └─────────────────────┘  │
    ┌─────▼────┐                                       │
    │ RUNNING  │                                       │
    └──┬────┬──┘                                       │
       │    │                                          │
  ┌────▼─┐  └──────────────┐                          │
  │ COM- │           ┌─────▼────┐                      │
  │PLETED│           │  FAILED  │                      │
  └──────┘           └─────┬────┘                      │
                           │                           │
                    ┌──────▼────┐                      │
                    │ RETRYING  ├──────────────────────┘
                    └───────────┘

                    ┌──────────┐
                    │CANCELLED │ ← from QUEUED or READY
                    └──────────┘
```

### Job Status Definitions

| Status | Meaning |
|---|---|
| `CREATED` | Job has been received by the system |
| `QUEUED` | Job is in the queue, waiting for a free worker |
| `WAITING_FOR_DEPENDENCY` | Job has unresolved dependencies; not yet eligible |
| `READY` | Job is eligible for scheduling |
| `RUNNING` | Job has been dispatched to a worker and is executing |
| `COMPLETED` | Worker returned a successful result |
| `FAILED` | Worker returned an error or timed out (retries exhausted) |
| `RETRYING` | Job failed but is eligible for re-attempt |
| `CANCELLED` | Job was explicitly cancelled before execution |

### Job Queue

PostgreSQL is the durable source of truth. The queue is a logical view over persistent job state — not a separate in-memory structure.

```
PostgreSQL
    ↓
Durable source of truth
    ↓
Queue = view of READY / QUEUED jobs
```

Only jobs that are **actually eligible** (status = `READY`) should reach the scheduler.

### Job Types

A `job_type` identifies the function that needs to execute:

```
send_email
generate_pdf
resize_image
send_sms
process_invoice
```

Every job type maps to a worker capability. Workers advertise which job types they can process.

---

## 10. Layer 3 — Worker System

### What Is a Worker?

A **Worker** is a running process owned and operated by the customer. It connects to Orqen, receives jobs, executes the customer's own functions, and returns results.

```
Customer Application
│
└── Worker
    ├── send_email()
    ├── send_sms()
    └── generate_pdf()
```

Orqen never receives or executes the customer's function code. It only delivers:

```
Job Type
Payload
Job Metadata
```

### Worker Registration

The customer initializes the SDK with the project's API key, then registers the worker:

```
Worker
├── ID:              worker_abc123
├── Capabilities:    ["send_email", "generate_pdf"]
└── Concurrency:     10
```

Workers are scoped to a project:

```
Project 1
│
├── Worker 1  (capabilities: send_email, generate_pdf)
├── Worker 2  (capabilities: generate_pdf)
└── Worker 3  (capabilities: send_email, send_sms)

Project 2
│
├── Worker 4  (capabilities: send_email)
└── Worker 5  (capabilities: process_invoice)
```

### Worker Capabilities

Capabilities are the bridge between jobs and workers:

```
Worker 1: ["send_email", "send_sms"]
Worker 2: ["generate_pdf"]
Worker 3: ["send_email", "generate_pdf"]
```

This creates the mapping:

```
Job Type: generate_pdf
    ↓
Eligible Workers: Worker 2, Worker 3
```

### Worker Concurrency

Each worker declares its concurrency limit — the number of jobs it can execute in parallel:

```
Worker 1
Concurrency: 10

Current jobs: 10  →  Worker is at capacity
Current jobs: 7   →  Worker has 3 available slots
```

The scheduler must not assign additional jobs to a worker that has reached its concurrency limit.

### Worker Data Model

| Field | Description |
|---|---|
| `worker_id` | Unique identifier |
| `project_id` | Owning project |
| `status` | `IDLE`, `BUSY`, `OFFLINE` |
| `capabilities` | Array of job types this worker handles |
| `concurrency` | Maximum parallel job slots |
| `current_job_count` | Currently assigned jobs |
| `last_heartbeat` | Timestamp of last heartbeat |
| `registered_at` | When the worker first registered |

---

## 11. Layer 4 — Worker Communication Protocol

### Transport Mechanism — Long Polling

Orqen uses **long polling** as its worker communication mechanism. This enables language-independent worker implementations — any language that can make HTTP requests can participate.

```
Customer Worker
      │
      │  GET /worker/jobs/poll
      │  (connection stays open)
      │
      ↓
   Orqen

   [No job available]
      │
      │  connection remains open...
      │
   [Job becomes available]
      │
      ↓
   Return job to worker
      │
      ↓
Customer Worker executes job
      │
      ↓
Customer Worker submits result
      │  POST /worker/jobs/{job_id}/result
      ↓
   Orqen
```

### Protocol Operations

The protocol must support at minimum the following operations:

| Operation | Direction | Description |
|---|---|---|
| `Register Worker` | Worker → Server | Declare existence, capabilities, concurrency |
| `Heartbeat` | Worker → Server | Signal that the worker is still alive |
| `Poll for Job` | Worker → Server | Long-poll to receive the next assigned job |
| `Receive Job` | Server → Worker | Deliver job type, payload, and metadata |
| `Acknowledge Job` | Worker → Server | Confirm receipt of the job before execution |
| `Submit Success` | Worker → Server | Return result of successful execution |
| `Submit Failure` | Worker → Server | Return error from a failed execution |
| `Reconnect` | Worker → Server | Re-establish connection after network interruption |
| `Graceful Shutdown` | Worker → Server | Signal the worker is shutting down gracefully |

### Language Independence

Because the protocol is HTTP-based, workers in any language can participate:

```
Node.js Worker  ─┐
Python Worker   ─┤──►  Orqen API
Go Worker       ─┤
Java Worker     ─┘
```

---

## 12. Layer 5 — Scheduler & CRS

### Scheduler Responsibilities

The scheduler answers one question:

> **Which eligible worker should receive this job?**

When a job becomes `READY`, the scheduler:

1. Identifies the required capability (job type)
2. Looks up the worker pool for that capability within the correct project
3. Passes the eligible pool to CRS
4. CRS selects the best available worker
5. The job is dispatched to that worker

### CRS — Concurrent Resource Scheduler

> **Library:** [`github.com/phero20/concurrent-resource-scheduler`](https://github.com/phero20/concurrent-resource-scheduler)  
> CRS is a separate, open-source Go library (also authored by this project). Orqen integrates it as a dependency — it is **not** reimplemented internally. It is a high-performance, domain-agnostic resource scheduler built on sharded priority heaps for low-contention concurrent scheduling.

CRS is the engine Orqen uses for **worker selection**. Each project's per-capability worker pool is managed as a CRS scheduler instance.

#### Worker Pools

When a worker registers a capability, it is registered into CRS via `sched.Add()`:

```
Project 1

generate_pdf → CRS scheduler instance
               (workers distributed across N shards)

Shard 0          Shard 1          Shard 2
┌──────────┐     ┌──────────┐     ┌──────────┐
│ Worker 2 │     │ Worker 5 │     │ Worker 1 │
└──────────┘     └──────────┘     └──────────┘

send_email → separate CRS scheduler instance
```

Workers are spread across independently-locked shards (configurable via `HeapCount`). There is no single global heap — multiple goroutines can acquire workers simultaneously without blocking each other.

#### How CRS Selects a Worker

CRS separates three distinct concerns:

**1. Priority (Comparator)**  
A `Comparator` function defines which worker is "best" within a shard — for example, the worker with the most available concurrency slots. This is pure domain logic owned by Orqen.

```go
// Example: prefer workers with more free slots
Comparator: func(a, b *Worker) int {
    return b.FreeSlots - a.FreeSlots
}
```

**2. Acquire Strategy (Shard Selection)**  
Separately, an `AcquireStrategy` decides *which shard to query first*. CRS ships three built-in strategies:

| Strategy | Behaviour |
|---|---|
| Round Robin *(default)* | Cycles through shards evenly using an atomic counter |
| Weighted | Probabilistic selection weighted by shard capacity |
| Adaptive | Routes to the least-congested shard using a non-blocking `ShardView` |

**3. Acquire Policy**  
Controls what happens to a worker when it is selected:

| Policy | Behaviour | When to use in Orqen |
|---|---|---|
| `Exclusive` | Worker is **popped** from the heap (removed from active rotation) until `Release()` is called | When a worker slot is fully occupied by a job |
| `Shared` | Worker **stays** in the heap; O(1) peek with no mutation | When tracking workers that can serve multiple concurrent jobs via comparator |

**4. Affinity Routing**  
`sched.AcquireByAffinity(key)` uses a Consistent Hash Ring to deterministically route to the same shard for a given key — useful for session or tenant stickiness.

**5. Cooldown (Extension)**  
The `extensions/cooldown` package is an async Observer that intercepts `Release` events and temporarily excludes a worker from the pool for a configured duration before re-including it. It is not a built-in scheduler mechanism — it runs asynchronously after the hot path.

#### CRS Boundary

CRS is responsible **only** for worker selection. It is explicitly not responsible for:

```
❌ PostgreSQL persistence
❌ API authentication
❌ DAG management
❌ Customer function execution
❌ Project management
```

#### CRS vs PostgreSQL

| System | Answers |
|---|---|
| PostgreSQL | What exists? What happened? What is the current state? |
| CRS | Which available worker should receive this job right now? |

CRS holds **only in-memory state**. It has no persistence of its own. If CRS state is lost, Orqen queries PostgreSQL for all active workers and repopulates each CRS scheduler instance via `sched.BatchAdd()` on startup.

### Scheduling Flow

```
Job becomes READY
        │
        ▼
Identify required capability
        │
        ▼
Look up Project + Capability Worker Pool
        │
        ▼
        CRS
        │
        ▼
Select best eligible worker
        │
        ▼
Dispatch job to worker via long poll
```

---

## 13. Layer 6 — Dependency & DAG Engine

### Simple Dependencies

Jobs can declare dependencies on other jobs:

```
Job A  →  Job B  →  Job C
```

Job B cannot become `READY` until Job A has `COMPLETED`.  
Job C cannot become `READY` until Job B has `COMPLETED`.

### DAG — Directed Acyclic Graph

More complex workflows are expressed as DAGs:

```
        A
       / \
      B   C
       \ /
        D
```

- B and C can execute in parallel after A completes
- D requires both B and C to complete before it becomes eligible

### DAG Engine Responsibilities

| Responsibility | Description |
|---|---|
| Create dependencies | Record that Job X must precede Job Y |
| Validate dependencies | Ensure the declared graph is valid |
| Detect cyclic graphs | Reject any dependency graph that contains a cycle |
| Track dependency state | Monitor which upstream jobs have completed |
| Identify ready jobs | Determine when all dependencies are satisfied |
| Release dependent jobs | Transition jobs from `WAITING_FOR_DEPENDENCY` → `READY` |
| Handle dependency failures | Define behavior when an upstream job fails |
| Track workflow completion | Know when an entire DAG has finished |

### Waiting vs Ready

This distinction is critical for correct scheduling:

```
Job A → COMPLETED       (upstream finished)
Job B → READY           (no dependencies, or all satisfied)
Job C → WAITING_FOR_DEPENDENCY  (upstream not yet done)
```

Job C **must not** enter the scheduler until its dependencies are resolved.

```
WAITING_FOR_DEPENDENCY
          │
    [upstream completes]
          │
          ▼
        READY
          │
          ▼
      Scheduler
          │
          ▼
         CRS
          │
          ▼
        Worker
```

The DAG system does not need its own worker execution mechanism. Once a job becomes `READY`, it follows the standard scheduling pipeline.

---

## 14. Layer 7 — Reliability & Failure Handling

### Retry Mechanism

Jobs that fail are automatically retried up to a configured maximum:

```
Job
 │
 ▼
Attempt 1  →  FAILED
 │
 ▼
Attempt 2  →  FAILED
 │
 ▼
Attempt 3  →  SUCCESS
```

#### Retry Configuration

| Parameter | Description |
|---|---|
| `maxRetries` | Maximum number of retry attempts |
| `backoff` | Delay strategy between attempts (fixed, exponential) |
| `timeout` | Maximum execution time per attempt |

### Job Attempt Model

To handle retries and duplicate execution, every job maintains an explicit **attempt log**:

```
Job 123
│
├── Attempt 1  (FAILED at 10:00)
└── Attempt 2  (SUCCESS at 10:05)
```

This makes it possible to distinguish:
- A legitimately new attempt
- A late result arriving from an old attempt
- A duplicate result from the same attempt

### Worker Failure Detection

If a worker crashes mid-execution:

```
Worker 1
   │
   │  receives Job 123
   │
   │  CRASH
   │
   ↓  [heartbeat stops]

Orqen detects missing heartbeat
   │
   ▼
Recover Job 123
   │
   ▼
Requeue / Retry
   │
   ▼
Scheduler assigns to another worker
```

The heartbeat mechanism is the primary signal for worker liveness. A worker that stops sending heartbeats within the expected interval is considered dead, and its in-flight jobs are recovered.

### Server Restart Recovery

PostgreSQL is the durable source of truth. Orqen can crash and fully recover:

```
Orqen crashes
      │
      ▼
PostgreSQL still holds all job state
      │
      ▼
Orqen restarts
      │
      ▼
Loads persistent state
      │
      ▼
Finds unfinished / in-flight jobs
      │
      ▼
Reconstructs CRS scheduling state
      │
      ▼
Resumes processing
```

Jobs must not depend solely on in-memory state.

### Duplicate Execution

Distributed systems can deliver a job more than once (network failures, retries, etc.).

```
Worker executes Job 123
      │
      ▼
Worker sends result
      │
      ▼  [network failure]
Server does not receive result
      │
      ▼
Job is retried → potentially executed again
```

The attempt model and result handling logic must explicitly account for:
- **Late results** — a result arriving for an attempt that has already been superseded
- **Duplicate results** — the same attempt submitting a result more than once

The system must handle both cases without corrupting job state.

---

## 15. Data Layer — PostgreSQL

### Role

PostgreSQL is the **single source of truth** for all persistent state in Orqen.

### Entities Stored

| Entity | Description |
|---|---|
| `projects` | All projects in the deployment |
| `api_keys` | Keys scoped to projects |
| `workers` | Registered worker instances |
| `worker_capabilities` | Which job types each worker handles |
| `jobs` | All job records with full metadata |
| `job_attempts` | Individual execution attempts per job |
| `dependencies` | Job-to-job dependency edges |
| `workflows` | DAG/workflow definitions and metadata |

### CRS vs PostgreSQL — Responsibility Split

```
PostgreSQL answers:
  "What exists?"
  "What happened?"
  "What is the current state of Job 123?"

CRS answers:
  "Right now, which worker should receive this job?"
```

CRS is a **purely in-memory** library with no persistence of its own. On startup or restart, Orqen queries PostgreSQL for all registered active workers and repopulates each per-project, per-capability CRS scheduler instance by calling `sched.BatchAdd()`. CRS state is always fully derivable from PostgreSQL.

---

## 16. Project Isolation Model

Every resource in Orqen belongs to exactly one project. The project boundary is enforced at every layer.

```
Project A                    Project B
├── Worker A1                ├── Worker B1
├── Worker A2                ├── Worker B2
├── Job A1                   ├── Job B1
└── DAG A1                   └── DAG B1
```

A request authenticated with Project A credentials must **only** be able to access Project A resources. This applies to:

```
✅  API layer — requests scoped by API key → project
✅  Database queries — all queries include project_id filter
✅  Worker assignment — workers only receive jobs from their project
✅  Scheduler — CRS pools are per-project
✅  Jobs — jobs are only visible to their owning project
✅  DAGs — workflows are project-scoped
✅  Dashboard — only shows data for the authenticated project
```

---

## 17. Complete Job Execution Flow

```
Customer Application
        │
        │  enqueue("generate_pdf", payload)
        ▼
     Orqen API
        │
        ▼
   PostgreSQL
        │  (Job stored as QUEUED)
        ▼
Dependency Check
        │
        │  No dependencies / all dependencies satisfied
        ▼
Job status → READY
        │
        ▼
    Scheduler
        │  Identifies required capability: generate_pdf
        ▼
Project 1 / generate_pdf Worker Pool
        │
        ▼
       CRS
        │  Selects best eligible worker
        ▼
     Worker 2
        │  (currently waiting on long-poll connection)
        ▼
 Job delivered to Worker 2
        │
        ▼
 generate_pdf(payload)  ← Customer function executes
        │
        ▼
     Result
        │
        ▼
  Worker 2 POSTs result to Orqen
        │
        ▼
   PostgreSQL
        │
        ▼
    Job → COMPLETED
```

---

## 18. Complete DAG Execution Flow

### Example DAG

```
A → B → C
```

### Initial State

```
Job A  →  READY
Job B  →  WAITING_FOR_DEPENDENCY
Job C  →  WAITING_FOR_DEPENDENCY
```

### Execution

**Step 1: A executes**

```
A → Scheduler → CRS → Worker → SUCCESS
```

**Step 2: DAG engine detects A completed**

```
A COMPLETED
    │
    ▼
B's dependency satisfied
    │
    ▼
B → READY
```

**Step 3: B executes**

```
B → Scheduler → CRS → Worker → SUCCESS
```

**Step 4: DAG engine detects B completed**

```
B COMPLETED
    │
    ▼
C's dependency satisfied
    │
    ▼
C → READY
```

**Step 5: C executes**

```
C → Scheduler → CRS → Worker → SUCCESS
```

**Step 6: DAG engine detects all nodes completed**

```
Workflow → COMPLETED
```

No separate execution architecture is needed. Once a job becomes `READY`, it flows through the standard scheduling pipeline.

### Parallel DAG Example

```
        A
       / \
      B   C
       \ /
        D
```

After A completes → B and C both become `READY` and execute in parallel.  
After both B and C complete → D becomes `READY`.

---

## 19. Dashboard

### Structure

```
Orqen Dashboard
│
├── Projects List
│
└── [Selected Project]
    │
    ├── Overview
    ├── Jobs
    ├── Workers
    ├── Workflows
    └── API Keys
```

### Jobs Page

| Column | Description |
|---|---|
| Job ID | Unique identifier |
| Type | Job type / capability |
| Status | Current state |
| Priority | Scheduling priority |
| Attempts | Number of execution attempts |
| Created | Creation timestamp |
| Updated | Last status change |

### Workers Page

| Column | Description |
|---|---|
| Worker ID | Unique identifier |
| Status | `IDLE`, `BUSY`, `OFFLINE` |
| Capabilities | List of supported job types |
| Concurrency | Max parallel job slots |
| Current Jobs | Number of active jobs |
| Last Heartbeat | Most recent heartbeat timestamp |

### Workflows Page

Displays DAGs visually:

```
        A
       / \
      B   C
       \ /
        D
```

Each node shows its current status (`WAITING`, `READY`, `RUNNING`, `COMPLETED`, `FAILED`).

---

## 20. Customer Experience — End-to-End

The complete customer journey from deployment to running jobs:

```
Step 1  →  Deploy Orqen on your server

Step 2  →  Open Dashboard in a browser

Step 3  →  Create a Project (e.g., "My E-Commerce App")

Step 4  →  Receive Project ID + API Key
              proj_123
              orqen_live_xxxxxxxxxxxxxxxxxx

Step 5  →  Install the Orqen SDK in your application

Step 6  →  Initialize the SDK with your API key
              const orqen = new Orqen({ apiKey: "orqen_live_xxx" })

Step 7  →  Register a Worker
              const worker = orqen.worker({ concurrency: 10 })

Step 8  →  Register worker capabilities (your functions)
              worker.register("send_email", async (payload) => { ... })
              worker.register("generate_pdf", async (payload) => { ... })

Step 9  →  Enqueue Jobs from your application
              await orqen.enqueue("send_email", { to: "user@example.com" })

Step 10 →  Optionally define dependencies
              await orqen.enqueue("generate_pdf", { ... }, { dependsOn: jobA.id })

Step 11 →  Orqen schedules the job automatically

Step 12 →  Worker receives the job via long polling

Step 13 →  Your function executes locally inside your worker

Step 14 →  Worker returns the result to Orqen

Step 15 →  Orqen updates job state in PostgreSQL

Step 16 →  Dashboard reflects the completed job
```

---

## 21. Key Design Decisions & Tradeoffs

### PostgreSQL as Queue

**Decision:** Use PostgreSQL as the job queue rather than Redis or an in-memory queue.

**Rationale:**
- Durability without additional infrastructure
- Supports complex dependency queries natively (JOINs, transactions)
- Recovery from crash is built-in
- Simpler operational model (one fewer system to manage)

**Tradeoff:**
- Lower raw throughput vs Redis at extreme scale
- Requires careful indexing and query optimization to avoid polling bottlenecks

---

### Long Polling vs Push / WebSockets

**Decision:** Workers use long polling to receive jobs.

**Rationale:**
- Works with any HTTP client (every language has one)
- No persistent connection management required by the server
- Firewall-friendly (outbound HTTP from worker)
- Simple to implement in any SDK

**Tradeoff:**
- Slightly higher latency than a push model
- More connections under high worker counts

---

### CRS as a Separate Component

**Decision:** Worker selection is extracted into a dedicated CRS layer with sharded heap structures.

**Rationale:**
- O(log n) selection vs O(n) scanning all workers
- CRS can evolve independently of the scheduler and job system
- Clear separation of concerns

**Tradeoff:**
- CRS state must be reconstructed on restart from PostgreSQL
- Added complexity vs naive worker scanning

---

### Project-Level Isolation (Not Tenant-Level Encryption)

**Decision:** Isolation is enforced via `project_id` scoping on all queries and authorization checks.

**Rationale:**
- Sufficient for V1 use cases (single deployment, trusted operators)
- Simple to implement and audit

**Tradeoff:**
- Data is not physically separated — a database breach would expose all projects
- For V1, this is an accepted tradeoff

---

## 22. Glossary

| Term | Definition |
|---|---|
| **Job** | One unit of work: a type, a payload, and metadata |
| **Job Type** | A string identifier for the kind of work (e.g., `send_email`) |
| **Worker** | A customer-operated process that executes job functions |
| **Capability** | A job type that a worker has declared it can handle |
| **Worker Pool** | The set of workers registered for a specific capability within a project |
| **CRS** | Concurrent Resource Scheduler — an open-source Go library (`github.com/phero20/concurrent-resource-scheduler`) that Orqen integrates for worker selection. It uses sharded priority heaps, a pluggable `Comparator` for priority logic, configurable `AcquireStrategy` for shard selection (Round Robin, Weighted, Adaptive), `AcquirePolicy` (Shared/Exclusive), and affinity routing via a Consistent Hash Ring |
| **DAG** | Directed Acyclic Graph — a workflow composed of jobs with dependencies |
| **Dependency** | A declared constraint that Job B cannot run until Job A completes |
| **Long Polling** | An HTTP technique where the server holds the connection open until a job is available |
| **Project** | An isolated workspace inside one Orqen deployment |
| **API Key** | A project-scoped credential used to authenticate SDK and worker connections |
| **Attempt** | A single execution try of a job (a job can have multiple attempts) |
| **Heartbeat** | A periodic signal from a worker to confirm it is still alive |
| **Sharded Heap** | A data structure used by CRS to efficiently rank and select workers |
| **PostgreSQL** | The durable data store that serves as the system's source of truth |
| **Scheduler** | The component that identifies ready jobs and routes them to CRS |
| **WAITING_FOR_DEPENDENCY** | A job status indicating upstream jobs have not yet completed |
| **READY** | A job status indicating a job is eligible to be scheduled |
| **Concurrency** | The maximum number of jobs a worker can execute simultaneously |
| **Self-hosted** | The customer deploys and operates Orqen on their own infrastructure |

---

*This document describes the complete V1 scope of Orqen. It is intended as the canonical reference for all design and implementation decisions.*
