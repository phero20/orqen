# 01 — Project Overview

> **Document purpose:** Defines what Orqen is, why it exists, what V1 delivers, and the complete customer journey.  
> **Read before:** Everything else. This is the product foundation.  
> **Reference when:** Making any scope decision or explaining the product to a new agent/contributor.

---

## What Is Orqen?

Orqen is a **self-hosted, multi-project, language-independent job processing and workflow platform**.

A single Orqen deployment lets engineering teams run background job processing across multiple independent applications — each fully isolated from the others — without being locked into a specific programming language or runtime.

The customer deploys Orqen on their own infrastructure. They create projects inside it. Each project gets credentials. Their workers (written in any language) connect to Orqen, register what they can do, and receive jobs via long polling. Orqen handles everything in between: storing jobs, managing state, resolving dependencies, scheduling, delivering, and recovering from failures.

---

## The Problem Orqen Solves

Most background job systems have at least one of these limitations:

**1. Language lock-in**  
BullMQ requires Node.js. Celery requires Python. Teams with polyglot stacks have no unified option. They end up running multiple job systems for different services.

**2. No multi-tenancy**  
Running separate job infrastructure per application is expensive and operationally heavy. There is no good self-hosted option for running multiple isolated job pipelines in one deployment.

**3. Volatile state**  
Redis-backed queues lose state under failure without additional safeguards. Crashed jobs disappear. Recovery is manual.

**4. No dependency modeling**  
Simple FIFO queues cannot express "Job B should only run after Job A succeeds." Teams bolt on external orchestration tools.

**5. No built-in workflow orchestration**  
DAG-based workflows require plugging in separate tools like Airflow or Temporal. The operational cost is high for teams that just need basic dependency chains.

Orqen solves all five in a single self-hosted deployment.

---

## V1 Vision

> Enable any engineering team — regardless of language or infrastructure — to run durable, observable, dependency-aware background jobs through a single self-hosted platform.

---

## V1 Goals

| ID | Goal |
|---|---|
| G1 | Multiple logically isolated projects inside one deployment |
| G2 | Workers can be written in any language |
| G3 | All job state is durably stored in PostgreSQL |
| G4 | Jobs can declare dependencies on other jobs (DAG support) |
| G5 | A built-in scheduler selects the best worker using CRS |
| G6 | Workers communicate with the server over long polling |
| G7 | Failures are detected and jobs are retried automatically |
| G8 | A web dashboard provides real-time observability |
| G9 | The customer never exposes their function code to Orqen |

---

## V1 Non-Goals

These are explicitly out of scope for V1. Do not implement or design for them.

| Non-Goal | Reason |
|---|---|
| CPU / GPU resource-aware scheduling | Adds complexity not needed for V1 worker model |
| External trigger types (webhooks, cron, events) | Out of scope; jobs are enqueued programmatically |
| Plugin ecosystem | Premature for V1 |
| Multi-region replication | Infrastructure concern beyond V1 |
| Built-in secret management | Customers manage their own secrets |
| Function code execution inside Orqen server | Orqen never runs customer code |
| General-purpose event streaming | Orqen is a job platform, not an event bus |

---

## What Orqen Does vs Does Not Do

### Orqen Does

```
✅ Store jobs durably in PostgreSQL
✅ Manage job state and lifecycle
✅ Manage job dependencies and DAGs
✅ Find eligible workers for a job
✅ Schedule jobs to the best available worker (via CRS)
✅ Deliver jobs to workers over long polling
✅ Receive results from workers
✅ Handle retries and failure recovery
✅ Manage projects and credentials
✅ Enforce project isolation
✅ Provide a web dashboard
```

### Orqen Does NOT Do

```
❌ Execute customer functions itself
❌ Require workers to be written in one language
❌ Require customer applications to run inside Orqen
❌ Expose or store customer function source code
❌ Operate as a general-purpose event streaming platform
❌ Perform CPU/GPU resource-aware scheduling (V1)
❌ Provide a plugin ecosystem (V1)
```

---

## Deployment Model

Orqen is entirely self-hosted. The customer deploys the full platform on their own infrastructure.

```
Customer Server

Orqen
├── API Server
├── Scheduler
├── Worker Management
├── DAG Engine
├── Dashboard
├── PostgreSQL
└── CRS (in-process Go library)
```

Inside one deployment, the customer creates multiple projects:

```
One Orqen Deployment
│
├── Project: E-Commerce
├── Project: SaaS Application
├── Project: Mobile Backend
└── Project: Internal Tools
```

Projects are logically isolated. Project A cannot access Project B's jobs, workers, API keys, or workflows.

---

## Project Model

A project is the top-level isolation unit inside Orqen.

```
Project
├── Project ID          (e.g. proj_abc123)
├── Name                (e.g. "My E-Commerce App")
├── API Keys
├── Workers
├── Worker Capabilities
├── Jobs
├── Job Attempts
├── Dependencies
└── Workflows / DAGs
```

When a project is created, Orqen generates a project-scoped API key:

```
Project ID:  proj_abc123
API Key:     orqen_live_xxxxxxxxxxxxxxxxxx
```

The customer uses the key to initialize their SDK. The key is the authentication boundary — the server resolves the project from the key, never from a client-supplied project ID.

---

## Technology Stack

| Component | Technology |
|---|---|
| Backend server | Go |
| Database | PostgreSQL |
| Worker selection | CRS — `github.com/phero20/concurrent-resource-scheduler` |
| Worker communication | HTTP long polling |
| SDKs | Node.js, Python, Go (and any language with an HTTP client) |

---

## Complete Customer Journey

The full experience from deployment to running jobs:

```
Step 1   Deploy Orqen on your server

Step 2   Open the Dashboard in a browser

Step 3   Create a Project
           → Name: "My E-Commerce App"

Step 4   Receive credentials
           Project ID:  proj_abc123
           API Key:     orqen_live_xxxxxxxxxxxxxxxxxx

Step 5   Install the Orqen SDK in your application

Step 6   Initialize the SDK
           const orqen = new Orqen({ apiKey: "orqen_live_xxx" })

Step 7   Register a Worker
           const worker = orqen.worker({ concurrency: 10 })

Step 8   Register worker capabilities (your functions)
           worker.register("send_email", async (payload) => { ... })
           worker.register("generate_pdf", async (payload) => { ... })

Step 9   Enqueue jobs from your application
           await orqen.enqueue("send_email", { to: "user@example.com" })

Step 10  Optionally define dependencies
           await orqen.enqueue("generate_pdf", { ... }, { dependsOn: jobA.id })

Step 11  Orqen schedules the job automatically

Step 12  Worker receives the job via long polling

Step 13  Your function executes locally inside your worker process

Step 14  Worker returns the result to Orqen

Step 15  Orqen updates job state in PostgreSQL

Step 16  Dashboard reflects the completed job
```

---

## Core Mental Model

Five questions describe the entire system:

```
1. WHAT PROJECT DOES THIS BELONG TO?
   → Project + API Key authentication

2. WHAT NEEDS TO HAPPEN?
   → Job (type + payload + priority + metadata)

3. CAN IT RUN YET?
   → Dependency Engine / DAG (WAITING or READY?)

4. WHO SHOULD RUN IT?
   → Scheduler + CRS (find best eligible worker)

5. HOW DOES IT ACTUALLY RUN?
   → Worker + Long Polling + Customer Function
```

PostgreSQL holds the durable state of the entire system.

---

## Related Documents

| Question | Document |
|---|---|
| How is the system structured? | `03-ARCHITECTURE.md` |
| What are the exact requirements? | `02-REQUIREMENTS.md` |
| What does the schema look like? | `04-DATA-MODEL.md` |
| How do workers connect? | `05-WORKER-PROTOCOL.md` |
| How is isolation enforced? | `11-PROJECT-ISOLATION.md` |
