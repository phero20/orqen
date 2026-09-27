# Orqen — Documentation Index

> **This is the entry point for all agents and contributors.**  
> Read this file first. It tells you what every document contains, in what order to read them, and where to find specific information.

---

## What Is Orqen?

Orqen is a **self-hosted, multi-project, language-independent job processing and workflow platform** written in Go.

It allows engineering teams to deploy a single Orqen instance and use it to power background job processing across multiple independent applications. Workers can be written in any language. Jobs are stored durably in PostgreSQL. Scheduling is powered by the [concurrent-resource-scheduler](https://github.com/phero20/concurrent-resource-scheduler) (CRS) Go library.

**The backend is Go. SDKs are multi-language.**

---

## Reading Order

If you are an agent or new contributor, read the documents in this order:

| Order | File | What It Answers |
|---|---|---|
| 1 | `01-PROJECT.md` | What is Orqen? Why does it exist? What is V1 scope? |
| 2 | `02-REQUIREMENTS.md` | What must the system do? What are the constraints? |
| 3 | `03-ARCHITECTURE.md` | How is the system structured? What are the layers? |
| 4 | `04-DATA-MODEL.md` | What is stored in PostgreSQL? What does the schema look like? |
| 5 | `05-WORKER-PROTOCOL.md` | How do workers communicate with the server? |
| 6 | `06-JOB-SCHEDULING.md` | How does a job go from READY to executed? |
| 7 | `07-CRS-INTEGRATION.md` | How does Orqen use the CRS library specifically? |
| 8 | `08-RELIABILITY.md` | How are failures, retries, and crashes handled? |
| 9 | `09-API-REFERENCE.md` | What are all the HTTP endpoints and their contracts? |
| 10 | `10-SDK-SPEC.md` | What must every language SDK implement? |
| 11 | `11-PROJECT-ISOLATION.md` | How is multi-project isolation enforced? |
| 12 | `12-DASHBOARD.md` | What does the web dashboard show and do? |
| 13 | `13-DEVELOPER-GUIDE.md` | How do you run and build the project locally? |

---

## File Descriptions

### `01-PROJECT.md`
The product overview. Covers what Orqen is, the problem it solves, the V1 vision and goals, explicit non-goals, and the complete customer journey from deployment to running jobs. Start here to understand the product context before touching any code.

### `02-REQUIREMENTS.md`
All functional and non-functional requirements for V1. Covers what the system must do (job processing, worker management, DAGs, multi-project, dashboard) and what it must NOT do. Also covers performance, reliability, and security requirements. Reference this when making any scope decision.

### `03-ARCHITECTURE.md`
The full system architecture. Covers the 7 layers (Project & Control, Job System, Worker System, Worker Communication, Scheduler + CRS, DAG Engine, Reliability), how they connect, component responsibilities, and the complete job execution flow end-to-end. The authoritative source for "how does the system fit together."

### `04-DATA-MODEL.md`
The PostgreSQL schema. Every table, every column, data types, indexes, foreign keys, and the relationships between entities. Also covers the job state machine transitions as they map to DB state. Reference this for any database query or migration work.

### `05-WORKER-PROTOCOL.md`
The worker communication protocol. Covers long polling mechanics, all protocol operations (register, heartbeat, poll, acknowledge, submit success/failure, reconnect, shutdown), request/response formats, and how language independence is achieved. Reference this when implementing any SDK or the worker-facing API.

### `06-JOB-SCHEDULING.md`
The job scheduling subsystem. Covers the full job lifecycle state machine, the DAG/dependency engine (how dependencies are tracked and released), the difference between WAITING and READY, and the complete scheduling flow from job becoming READY to worker receiving it. Reference this for anything touching the scheduler or DAG engine.

### `07-CRS-INTEGRATION.md`
How Orqen specifically uses the [concurrent-resource-scheduler](https://github.com/phero20/concurrent-resource-scheduler) library. Covers the registry map design (`(project_id, capability) → *crs.Scheduler`), lazy initialization, which CRS APIs are called and when (`Add`, `Remove`, `Exclude`, `Include`, `BatchAdd`, `Acquire`, `Release`), the Comparator definition for workers, `Exclusive` acquire policy rationale, and how CRS state is rebuilt from PostgreSQL on restart. **Read this before touching anything in the scheduler layer.**

### `08-RELIABILITY.md`
The reliability subsystem. Covers the job attempt model, retry configuration (`maxRetries`, `backoff`, `timeout`), heartbeat-based worker liveness detection, worker crash recovery flow, server restart recovery, and duplicate/late result handling. Reference this for any failure handling work.

### `09-API-REFERENCE.md`
The complete HTTP API. Every endpoint grouped by domain (Projects, API Keys, Jobs, Workers, Workflows), HTTP method, path, auth requirements, request body, response body, status codes, and error formats. Reference this when implementing any API handler or SDK method.

### `10-SDK-SPEC.md`
The language-neutral SDK specification. Defines what every SDK (Node.js, Python, Go, etc.) must implement: initialization, worker registration, capability registration, job enqueueing, the long-poll loop, result submission, error handling, and reconnection behavior. Implement each language SDK from this spec — do not deviate from the contract it defines.

### `11-PROJECT-ISOLATION.md`
The multi-project isolation model. Covers the isolation rules at every layer (API auth, database queries, CRS pools, scheduler, dashboard), what Project A is and is not allowed to access, and a checklist for verifying isolation in any new feature. **Any feature that touches multiple projects must be reviewed against this document.**

### `12-DASHBOARD.md`
The web dashboard specification. Covers page structure, each page's purpose, what data is displayed, what actions are available, and the visual DAG representation for workflows. Reference this for any frontend work.

### `13-DEVELOPER-GUIDE.md`
The local development guide. Covers prerequisites, environment variables, how to run the server, how to run tests, how to run a local worker, project structure, and contribution guidelines.

---

## Key Architectural Decisions (Do Not Reverse Without Reading)

These decisions were made deliberately. Agents must not refactor away from them without understanding the rationale documented in `03-ARCHITECTURE.md` and `07-CRS-INTEGRATION.md`.

| Decision | Rationale Summary |
|---|---|
| PostgreSQL as job queue (not Redis) | Durability, crash recovery, dependency queries via JOINs |
| Long polling (not WebSockets or push) | Language independence — every language has an HTTP client |
| One CRS instance per `(project_id, capability)` | Workers have multiple capabilities; cross-capability coordination would require global locks, defeating CRS's purpose |
| `Exclusive` CRS acquire policy | Worker slots are finite; popping on acquire prevents over-assignment |
| Concurrency enforced via `Exclude/Include` | When a worker hits its concurrency limit, Orqen excludes it from all its capability pools |
| CRS state rebuilt from PostgreSQL on restart | CRS is purely in-memory; PostgreSQL is the source of truth |
| Project isolation enforced at query level | `project_id` filter on every DB query; API key resolves project server-side |

---

## Technology Stack

| Component | Technology |
|---|---|
| Backend server | Go |
| Database | PostgreSQL |
| Worker selection | [concurrent-resource-scheduler](https://github.com/phero20/concurrent-resource-scheduler) (Go library) |
| Worker communication | HTTP long polling |
| SDKs | Node.js, Python, Go (language-neutral spec in `10-SDK-SPEC.md`) |
| Dashboard | TBD |

---

## Glossary Quick Reference

| Term | Meaning |
|---|---|
| Project | An isolated workspace inside one Orqen deployment |
| Job | One unit of work: type + payload + metadata |
| Job Type | String identifier mapping to a worker capability (e.g. `send_email`) |
| Worker | Customer-operated process that executes job functions |
| Capability | A job type a worker has declared it can handle |
| CRS | Concurrent Resource Scheduler — Go library for worker pool management |
| CRS Registry | In-memory map of `(project_id, capability) → *crs.Scheduler` |
| DAG | Directed Acyclic Graph — a workflow of jobs with dependencies |
| Long Polling | HTTP connection held open until a job is available |
| Heartbeat | Periodic signal from worker proving it is alive |
| Attempt | One execution try of a job |
| WAITING | Job has unresolved dependencies; not eligible for scheduling |
| READY | Job is eligible and can be dispatched to a worker |

---

*Last updated: V1 pre-implementation. Update this index whenever a new document is added or a key decision changes.*
