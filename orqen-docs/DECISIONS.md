# Architectural Decisions Log

> **Document purpose:** Records every significant architectural decision made for Orqen V1 — what was decided, why, what alternatives were considered, and what the accepted tradeoffs are.  
> **Read before:** Refactoring any core component or proposing an architectural change.  
> **Reference when:** An agent or contributor wants to change something fundamental — understand the reasoning first.

---

## How to Use This Document

Each decision is recorded with:
- **Status** — `ACCEPTED` (in effect), `SUPERSEDED` (replaced by a later decision), `PROPOSED` (under discussion)
- **Context** — the problem or question that required a decision
- **Decision** — what was decided
- **Alternatives considered** — what else was evaluated
- **Consequences** — tradeoffs and implications
- **Do not reverse without** — what must be true before reversing this decision

---

## Decision Index

| ID | Title | Status |
|---|---|---|
| D-001 | PostgreSQL as the job queue (no Redis) | ACCEPTED |
| D-002 | Long polling as the worker protocol (no WebSockets/gRPC) | ACCEPTED |
| D-003 | One CRS instance per (project_id, capability) | ACCEPTED |
| D-004 | Exclusive acquire policy in CRS | ACCEPTED |
| D-005 | Logical project isolation (not physical/schema-per-project) | ACCEPTED |
| D-006 | Go as the backend language | ACCEPTED |
| D-007 | At-least-once job delivery (not exactly-once) | ACCEPTED |
| D-008 | Heartbeat-based worker liveness detection | ACCEPTED |
| D-009 | Application-side ID generation (not PostgreSQL sequences) | ACCEPTED |
| D-010 | Failure propagation policy: block (V1 default) | ACCEPTED |

---

## D-001 — PostgreSQL as the Job Queue

**Status:** ACCEPTED

**Context:**
Orqen needs a durable job store. Most existing job systems use Redis as the queue layer (BullMQ, Sidekiq). We needed to decide whether to use Redis, PostgreSQL, or both.

**Decision:**
Use PostgreSQL exclusively as the job store and queue. No Redis dependency.

**Alternatives considered:**
- **Redis + PostgreSQL** — Redis as the fast queue, PostgreSQL as the audit log. Adds operational complexity (two systems), and Redis data loss under failure requires careful AOF/RDB configuration.
- **Redis only** — Not durable enough. State loss on restart is unacceptable for Orqen's reliability guarantees.
- **PostgreSQL only** — Chosen. Durable by default, supports complex dependency queries via JOINs and transactions, single system to operate.

**Consequences:**
- Lower raw throughput than Redis at extreme scale (millions of jobs/second)
- Requires careful indexing to avoid PostgreSQL polling bottlenecks at high volume
- Simpler operational model — one fewer system to deploy, monitor, and back up
- All state survives server restarts with zero extra configuration
- Complex DAG dependency queries are natural SQL (no custom data structures needed)

**Do not reverse without:**
A demonstrated, measured throughput ceiling that PostgreSQL cannot meet for the target workload. Redis must not be added "just in case" — it must solve a proven problem.

---

## D-002 — Long Polling as the Worker Protocol

**Status:** ACCEPTED

**Context:**
Workers need to receive jobs from the server. Options include: push (server initiates), WebSockets (bidirectional persistent), SSE (server-sent events), gRPC streaming, or long polling (worker-initiated HTTP).

**Decision:**
Long polling over HTTP. Workers call `GET /worker/poll` and the server holds the connection open until a job is available.

**Alternatives considered:**
- **WebSockets** — Persistent bidirectional connection. More efficient but requires every language SDK to implement WebSocket handling. Not all environments support WebSockets cleanly (some proxies, firewalls block them).
- **gRPC streaming** — Efficient and typed but requires protobuf tooling in every SDK language. High implementation burden per language.
- **Server-Sent Events (SSE)** — Server can push but connection management under load is complex.
- **Long polling** — Chosen. Every language has an HTTP client. Firewall-friendly (outbound HTTP from worker). Simple to implement in any SDK. Slightly higher latency than push but acceptable for the use case.

**Consequences:**
- Any language with an HTTP client can implement a worker
- Slightly higher latency (one round-trip to receive a job) vs push
- More open connections under high worker counts
- Simple reconnection — just re-poll

**Do not reverse without:**
A demonstrated latency or connection scalability problem that long polling cannot address, AND a plan for maintaining SDK simplicity across all target languages.

---

## D-003 — One CRS Instance Per (project_id, capability)

**Status:** ACCEPTED

**Context:**
Orqen uses the `concurrent-resource-scheduler` Go library (CRS) for worker selection. We needed to decide how to map Orqen's concepts (projects, capabilities, workers) onto CRS scheduler instances.

**Proposal considered and rejected:** One CRS instance per project, with one shard per capability. This would mean shards represent different resource categories.

**Decision:**
One `*crs.Scheduler` instance per `(project_id, capability)` pair. Instances are lazily initialized. Workers with multiple capabilities are registered in multiple instances.

**Why the alternative was rejected:**
Workers have multiple capabilities (e.g. `["send_email", "generate_pdf"]`). CRS resources have a single identity — one worker cannot exist in multiple shards simultaneously within one instance. If Worker A is acquired for `send_email` (exclusive policy), the `generate_pdf` shard in the same instance would have no way to know Worker A is now busy. This would require cross-shard coordination — which defeats the entire purpose of CRS's sharded design (avoiding global locks).

**Consequences:**
- A worker with N capabilities is registered in N CRS instances
- When a worker hits its concurrency limit, Orqen must call `Exclude` on all N instances
- Memory overhead is minimal — a CRS instance is a few structs and heap arrays
- 500 capabilities across a project = 500 lightweight in-memory structs (negligible)
- Clean separation — each instance is a homogeneous pool of equivalent workers

**Do not reverse without:**
Demonstrated memory or initialization overhead that is measurable and problematic. The overhead of 500+ instances must be benchmarked before concluding it is a problem. The cross-shard coordination problem from the alternative is a hard architectural constraint — not a preference.

---

## D-004 — Exclusive Acquire Policy in CRS

**Status:** ACCEPTED

**Context:**
CRS supports two acquire policies: `Shared` (worker stays in heap after acquisition) and `Exclusive` (worker is popped from heap on acquisition, returned via `Release`).

**Decision:**
Use `Exclusive` policy. When a worker is acquired, it is popped from the heap. Orqen then checks if the worker has remaining concurrency slots — if yes, it calls `Include` to return it to the heap; if at capacity, it stays excluded until a slot frees.

**Alternatives considered:**
- **Shared policy** — Worker stays in heap. O(1) peek. But with Shared, there is no built-in mechanism to prevent over-assignment. Orqen would need to track concurrent dispatch counts separately and deal with race conditions between the CRS acquisition and the concurrency check.
- **Exclusive policy** — Chosen. The pop-and-reinsert pattern gives Orqen explicit control over when a worker is available. The slot check and `Include` happen atomically from the scheduler's perspective. Cleaner than Shared + external tracking.

**Consequences:**
- O(log N) acquire operations (heap mutation) vs O(1) for Shared
- Explicit `Include`/`Exclude` calls required when concurrency state changes
- Eliminates any race condition between "worker acquired" and "concurrency check"

**Do not reverse without:**
A measured performance bottleneck in acquire operations at the target scale. The O(log N) cost of Exclusive is well within acceptable bounds for job scheduling workloads.

---

## D-005 — Logical Project Isolation (Not Physical)

**Status:** ACCEPTED

**Context:**
Orqen supports multiple projects in one deployment. We needed to decide the isolation model: logical (shared DB, `project_id` filter everywhere) or physical (separate schema or database per project).

**Decision:**
Logical isolation via `project_id` scoping on all queries and authorization checks. All projects share one PostgreSQL database and one server process.

**Alternatives considered:**
- **Schema-per-project** — Each project gets its own PostgreSQL schema. Strong isolation but complex migration management (N schemas to migrate), high overhead for deployments with many projects.
- **Database-per-project** — Maximum isolation but operationally prohibitive for self-hosted deployments.
- **Logical isolation** — Chosen. Simpler to implement, simpler to operate. Sufficient for V1's self-hosted, trusted-operator model.

**Consequences:**
- A database-level breach exposes all projects
- A server bug could theoretically cause data leakage between projects
- No per-project resource quotas in V1
- Much simpler migrations, backups, and operations
- Adequate for self-hosted deployments where the operator is trusted

**Do not reverse without:**
A V2 requirement for SaaS-style multi-tenancy with untrusted tenants, or a security audit finding that logical isolation is insufficient for the target deployment model.

---

## D-006 — Go as the Backend Language

**Status:** ACCEPTED

**Context:**
The CRS library is written in Go. We needed to choose the backend language for the Orqen server.

**Decision:**
Go. The CRS library is integrated as a native Go dependency. Orqen's server is written in Go.

**Alternatives considered:**
- **Node.js** — Would require CRS to be wrapped in a sidecar or reimplemented. Not viable given direct CRS integration.
- **Python** — Same problem as Node.js. CRS FFI from Python is complex and fragile.
- **Rust** — High performance but steep learning curve and CRS FFI complexity.
- **Go** — Chosen. CRS is a native Go library. Direct integration, no FFI, no sidecar. Excellent concurrency model for the polling and scheduling workloads.

**Consequences:**
- All backend developers must know Go
- SDKs are separate per-language implementations (not a port of the server)
- Direct CRS integration — no serialization, no network hop for scheduling decisions

**Do not reverse without:**
This is a foundational decision tied to CRS. Reversing it means either reimplementing CRS in another language or wrapping it in a network sidecar — both are significant undertakings.

---

## D-007 — At-Least-Once Job Delivery

**Status:** ACCEPTED

**Context:**
Distributed systems cannot guarantee exactly-once delivery without significant complexity (distributed transactions, two-phase commit, idempotency keys at every layer). We needed to define the delivery guarantee for Orqen.

**Decision:**
At-least-once delivery. A job may be executed more than once in failure scenarios (server crash during dispatch, worker crash + recovery + re-dispatch). The attempt model tracks executions. Workers are expected to implement idempotent functions.

**Alternatives considered:**
- **Exactly-once** — Requires distributed transactions between job dispatch and worker acknowledgment. Extreme complexity. Not justified for V1.
- **At-most-once** — Jobs could be lost. Unacceptable for a durable job platform.
- **At-least-once** — Chosen. Durable, recoverable, and the standard guarantee for job processing systems.

**Consequences:**
- Customer worker functions must be designed to be idempotent
- This must be clearly documented in the SDK and customer-facing docs
- The attempt model (`job_attempts` table) provides full audit trail of all executions

**Do not reverse without:**
N/A. This is a fundamental distributed systems tradeoff. Exactly-once cannot be added without a complete rearchitecture of the dispatch and acknowledgment protocol.

---

## D-008 — Heartbeat-Based Worker Liveness Detection

**Status:** ACCEPTED

**Context:**
Workers are external processes. The server needs to detect when a worker has died so it can recover in-flight jobs.

**Decision:**
Workers send periodic heartbeats (`POST /worker/heartbeat`) every 15 seconds. A background monitor on the server checks for workers whose `last_heartbeat_at` is older than 45 seconds and marks them `OFFLINE`, triggering job recovery.

**Alternatives considered:**
- **TCP keepalive / persistent connection** — Would require a persistent connection per worker. Breaks the stateless HTTP model and complicates SDK implementation.
- **Polling from server to worker** — Requires the server to know the worker's address. Not viable for workers behind NAT/firewalls.
- **Heartbeat** — Chosen. Simple, language-independent, works with the stateless HTTP protocol, no address exposure required.

**Consequences:**
- Up to 45 seconds of delay before a dead worker's jobs are recovered
- Workers must implement a background heartbeat loop
- The 3x multiplier (timeout = 3 × interval) provides tolerance for transient network hiccups without false positives

**Do not reverse without:**
A demonstrated need for faster liveness detection. The 45-second window can be reduced by decreasing `ORQEN_HEARTBEAT_TIMEOUT_SECONDS` — no architecture change needed for tuning.

---

## D-009 — Application-Side ID Generation

**Status:** ACCEPTED

**Context:**
Every entity in Orqen needs a unique ID. Options: PostgreSQL `SERIAL`/`BIGSERIAL` sequences, PostgreSQL `gen_random_uuid()`, or application-generated IDs.

**Decision:**
Application-side generation of prefixed random string IDs (e.g. `job_k8x2mn4p`). IDs are generated using a cryptographically random suffix.

**Alternatives considered:**
- **PostgreSQL SERIAL** — Sequential integers. Enumerable by clients, leaks record count information.
- **UUID v4** — Random but no semantic information in the ID. Harder to debug.
- **Prefixed random strings** — Chosen. IDs carry semantic meaning (`job_`, `worker_`, etc.), are non-sequential, are non-enumerable, and are consistent across environments (no dependency on DB sequence state for testing).

**Consequences:**
- IDs are generated before the DB insert — IDs are known before the row is committed
- No auto-increment dependency — easier to test and generate test fixtures
- Slightly longer IDs than integers but trivially small storage cost

**Do not reverse without:**
N/A — this is a low-stakes convention. If a specific use case requires sequential IDs, they can be added as a separate field.

---

## D-010 — Failure Propagation Policy: Block (V1 Default)

**Status:** ACCEPTED

**Context:**
When an upstream job in a DAG fails terminally (all retries exhausted), downstream jobs that were `WAITING_FOR_DEPENDENCY` on it need a defined behavior.

**Decision:**
V1 default policy is **block** — when an upstream job fails, all downstream jobs that depend on it are also transitioned to `FAILED` with the error `"upstream job {id} failed"`. The failure propagates through the entire downstream DAG chain.

**Alternatives considered:**
- **Continue** — Downstream job runs regardless of upstream failure. Makes sense for optional dependencies but requires a way for the downstream to know the upstream failed.
- **Skip** — Downstream job is `CANCELLED` instead of `FAILED`. Cleaner than `FAILED` for optional steps.
- **Configurable per-edge** — Most flexible but complex to implement and configure.
- **Block (propagate failure)** — Chosen for V1. Simple, predictable, safe default. If a workflow fails, the whole DAG is marked failed. No partial execution of dependent steps.

**Consequences:**
- A single job failure can cancel an entire workflow
- This is the safest default — prevents downstream jobs from operating on missing upstream results
- Future versions may support per-edge policies (`continue`, `skip`, `custom`)

**Do not reverse without:**
A customer requirement for non-blocking dependency failure handling. This can be implemented as a per-edge policy without changing the default.

---

*Add new decisions to this log whenever a significant architectural choice is made. Record rejected alternatives — they are as important as the accepted decision.*
