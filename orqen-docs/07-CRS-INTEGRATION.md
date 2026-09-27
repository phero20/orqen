# 07 — CRS Integration

> **Document purpose:** Defines exactly how Orqen integrates the `concurrent-resource-scheduler` Go library — the registry design, every CRS API call and when it is made, the Comparator, acquire policy, and startup rebuild.  
> **Read before:** Implementing anything in the scheduler layer or worker management layer.  
> **Reference when:** Adding a new CRS operation, debugging scheduling behavior, or understanding why a worker was or wasn't selected.

---

## What Is CRS?

CRS (`github.com/phero20/concurrent-resource-scheduler`) is an **external open-source Go library** authored by this project. It is a high-performance, domain-agnostic resource scheduler built on sharded priority heaps.

Orqen integrates CRS as a Go dependency. It is **not** reimplemented internally. The library is responsible for one thing only:

> Given a pool of workers for a specific capability, return the best available worker right now.

Everything else — routing to the right pool, concurrency enforcement, PostgreSQL persistence — is Orqen's responsibility.

**Library import path:**
```go
import (
    "github.com/phero20/concurrent-resource-scheduler/config"
    "github.com/phero20/concurrent-resource-scheduler/scheduler"
)
```

---

## Core Concept: One Instance Per (project_id, capability)

Orqen creates one `*scheduler.Scheduler` instance per unique `(project_id, capability)` pair.

```
proj_abc123 + "send_email"    → *scheduler.Scheduler  (contains: Worker1, Worker3, Worker8)
proj_abc123 + "generate_pdf"  → *scheduler.Scheduler  (contains: Worker1, Worker2, Worker5)
proj_abc123 + "resize_image"  → *scheduler.Scheduler  (contains: Worker4)

proj_xyz789 + "send_email"    → *scheduler.Scheduler  (contains: Worker9)
proj_xyz789 + "process_invoice" → *scheduler.Scheduler (contains: Worker10, Worker11)
```

Each instance is completely independent — different projects and different capabilities never share a CRS instance.

**Why not one instance per project with shards per capability?**
Workers can have multiple capabilities. A worker registered for `["send_email", "generate_pdf"]` must exist in both pools. With shards-as-capabilities, the same worker would need to live in multiple shards simultaneously — CRS does not support this (one resource, one identity, one shard). The per-`(project, capability)` model avoids this entirely. See `00-INDEX.md` — Key Architectural Decisions.

---

## The CRS Registry

Orqen maintains an in-memory registry map:

```go
type CRSRegistry struct {
    mu        sync.RWMutex
    instances map[registryKey]*scheduler.Scheduler[*WorkerNode, string]
}

type registryKey struct {
    ProjectID  string
    Capability string
}
```

### Registry Operations

| Operation | Method | When |
|---|---|---|
| Get or create instance | `registry.GetOrCreate(projectID, capability)` | Worker registers a new capability |
| Get existing instance | `registry.Get(projectID, capability)` | Job dispatch, worker exclude/include |
| Delete instance | `registry.Delete(projectID, capability)` | Last worker for a capability goes offline |

The registry is protected by a `sync.RWMutex`. Instance lookup is a read lock; creation is a write lock.

---

## WorkerNode — The CRS Resource Type

CRS is generic (`T any`). Orqen defines `WorkerNode` as the resource type:

```go
type WorkerNode struct {
    WorkerID        string
    ProjectID       string
    Concurrency     int
    CurrentJobCount int
}
```

CRS stores pointers to `WorkerNode`. Orqen updates `CurrentJobCount` in the node before calling `Update()` so the heap re-sorts correctly.

---

## CRS Configuration

Every CRS instance is created with the same configuration:

```go
cfg := config.Config[*WorkerNode, string]{
    HeapCount:     8,              // 8 shards — tune based on concurrency profile
    AcquirePolicy: config.Exclusive,
    Comparator:    workerComparator,
    KeyFunc:       func(w *WorkerNode) string { return w.WorkerID },
}

sched, err := scheduler.New(cfg)
```

### HeapCount
`HeapCount: 8` means 8 independent shard heaps per instance. Multiple goroutines can acquire workers simultaneously without blocking. Tune this value based on expected concurrent scheduling throughput per capability pool.

### AcquirePolicy: Exclusive
When `sched.Acquire()` is called, the worker is **popped** from its shard heap and moved to the inactive store. It is not available for another acquisition until `sched.Release()` or `sched.Include()` returns it to the heap.

This prevents double-dispatching the same worker beyond its concurrency limit. When a worker is acquired, Orqen checks if it has remaining slots — if it does, it is immediately re-included via `sched.Include()` so it remains available for more jobs. If it is at capacity, it stays excluded.

### KeyFunc
Returns `WorkerID` — the unique string identity of each worker within a CRS instance.

---

## The Comparator

The Comparator defines worker priority within a shard. Workers with more available concurrency slots are ranked higher (preferred first):

```go
var workerComparator = func(a, b *WorkerNode) int {
    aFree := a.Concurrency - a.CurrentJobCount
    bFree := b.Concurrency - b.CurrentJobCount
    if aFree > bFree {
        return -1  // a is better candidate
    }
    if aFree < bFree {
        return 1   // b is better candidate
    }
    return 0       // equal priority
}
```

A worker with `concurrency=10, currentJobCount=2` (8 free slots) is ranked higher than a worker with `concurrency=10, currentJobCount=8` (2 free slots). This distributes load toward the least-busy workers.

---

## Every CRS API Call Orqen Makes

### `sched.Add(worker)`

**When:** A worker registers a capability for the first time, or reconnects after going offline.

**What happens:**
1. Get or create the CRS instance for `(projectID, capability)`
2. Call `sched.Add(&WorkerNode{...})`
3. Worker is now in the heap and eligible for acquisition

**Error handling:** If `ErrDuplicateKey` is returned (worker already exists in this instance), call `sched.Update()` instead to refresh the node.

---

### `sched.BatchAdd(workers)`

**When:** Server startup/restart — rebuilding CRS state from PostgreSQL.

**What happens:**
1. Query PostgreSQL for all non-OFFLINE workers and their capabilities
2. Group by `(project_id, capability)`
3. For each group, create the CRS instance and call `sched.BatchAdd(workerSlice)`

`BatchAdd` performs atomic two-phase insertion — either all workers are added or none, preventing partial state.

```go
// Startup rebuild pseudocode
rows := db.Query(`
    SELECT w.id, w.concurrency, w.current_job_count, wc.capability, wc.project_id
    FROM workers w
    JOIN worker_capabilities wc ON wc.worker_id = w.id
    WHERE w.status != 'OFFLINE'
`)

grouped := groupByProjectCapability(rows)

for key, workers := range grouped {
    instance := registry.GetOrCreate(key.ProjectID, key.Capability)
    instance.BatchAdd(workers)
}
```

---

### `sched.Acquire()`

**When:** A `READY` job needs to be dispatched — the scheduler has looked up the correct CRS instance and calls `Acquire()`.

**What happens (Exclusive policy):**
1. AcquireStrategy selects a shard (default: Round Robin)
2. Best worker is popped from the shard heap
3. Worker moved to inactive store
4. Worker returned to Orqen

**After Acquire:**
```
worker returned
    │
    ▼
Orqen increments worker.CurrentJobCount
    │
    ▼
worker.CurrentJobCount < worker.Concurrency?
   │                          │
  YES                         NO
   │                          │
   ▼                          ▼
sched.Include(workerID)   Worker stays excluded
(worker stays available    (at capacity — will be
for more jobs)             included when slot frees)
```

**Error handling:** If `ErrNoResourceAvailable` is returned, all workers for this capability are either at capacity or offline. Job stays `READY` and will be dispatched when a worker becomes available.

---

### `sched.Release(workerID)`

**When:** The Exclusive policy requires `Release()` to move a worker from inactive back to active. Orqen calls this as part of the `Include` flow.

In practice, Orqen uses `sched.Include()` directly for concurrency-based re-admission rather than `Release()`, since `Include` gives explicit control over timing.

---

### `sched.Include(workerID)`

**When:** A worker frees a job slot (job completes or fails) AND the worker is currently excluded from CRS (was at capacity).

**What happens:**
1. Worker is moved from inactive store back to its shard heap
2. Worker is available for acquisition again

**Called on ALL capability instances for this worker:**
```go
for _, capability := range worker.Capabilities {
    instance := registry.Get(worker.ProjectID, capability)
    if instance != nil {
        instance.Include(worker.WorkerID)
    }
}
```

---

### `sched.Exclude(workerID)`

**When:** A worker reaches its concurrency limit (`CurrentJobCount == Concurrency`) after a job is dispatched.

**What happens:**
1. Worker is moved from its shard heap to the inactive store
2. Worker will not be returned by `Acquire()` until `Include()` is called

**Called on ALL capability instances for this worker:**
```go
for _, capability := range worker.Capabilities {
    instance := registry.Get(worker.ProjectID, capability)
    if instance != nil {
        instance.Exclude(worker.WorkerID)
    }
}
```

---

### `sched.Update(worker)`

**When:** A worker's `CurrentJobCount` changes (job dispatched or completed) and the worker is still active in the heap. This re-sorts the heap to reflect the new priority.

**When NOT to call:** If the worker is currently in the inactive store (excluded), `Update` is not needed — the updated node will be re-ranked when `Include` pushes it back to the heap.

---

### `sched.Remove(workerID)`

**When:** A worker goes permanently `OFFLINE` (heartbeat timeout, or explicit shutdown after all in-flight jobs complete).

**Called on ALL capability instances for this worker:**
```go
for _, capability := range worker.Capabilities {
    instance := registry.Get(worker.ProjectID, capability)
    if instance != nil {
        instance.Remove(worker.WorkerID)
        // If instance is now empty, optionally delete it from registry
    }
}
```

---

## Acquire Strategies

CRS ships three built-in acquire strategies. Orqen uses **Round Robin** (the default) in V1.

| Strategy | Behavior | When to consider |
|---|---|---|
| **Round Robin** *(V1 default)* | Cycles shards via atomic counter; even distribution | General use, good default |
| **Weighted** | Probabilistic selection by shard capacity weight | When shards have very uneven worker counts |
| **Adaptive** | Routes to least-congested shard using `ShardView` | Under extremely high concurrent scheduling throughput |

To change strategy, pass it in the config:
```go
cfg := config.Config[*WorkerNode, string]{
    // ...
    AcquireStrategy: acquire.NewAdaptiveStrategy[*WorkerNode, string](),
}
```

---

## Affinity Routing

CRS supports `sched.AcquireByAffinity(key)` which uses a Consistent Hash Ring to deterministically route to the same shard for a given key.

In V1, Orqen does not use affinity routing. It is available for future use cases such as:
- Routing jobs from the same customer/tenant to the same worker for cache locality
- Session-scoped jobs that benefit from same-worker execution

---

## Cooldown Extension

CRS ships an `extensions/cooldown` package that intercepts `Release` events and temporarily excludes a worker before re-including it. Orqen does **not** use the cooldown extension in V1 — concurrency-based exclusion via `Exclude`/`Include` is sufficient.

---

## CRS State and PostgreSQL

CRS holds **only in-memory state**. It has no database, no file, no persistence of any kind.

```
PostgreSQL = source of truth (what workers exist, their state)
CRS        = runtime index   (which worker to pick right now)
```

If the Orqen server crashes and restarts:
1. CRS instances are gone
2. Orqen queries PostgreSQL for all active workers + capabilities
3. CRS instances are rebuilt via `BatchAdd`
4. Scheduling resumes

CRS never needs to know this happened. From its perspective, it was just initialized.

---

## Concurrency Flow — Full Example

```
Worker 1 registers: capabilities=[send_email, generate_pdf], concurrency=3

registry[(proj1, send_email)].Add(Worker1{Concurrency:3, CurrentJobCount:0})
registry[(proj1, generate_pdf)].Add(Worker1{Concurrency:3, CurrentJobCount:0})

── Job A (send_email) arrives ──────────────────────────────
registry[(proj1, send_email)].Acquire() → Worker1
Worker1.CurrentJobCount = 1  (2 slots remain)
Worker1.CurrentJobCount < Concurrency → registry[(proj1, send_email)].Include(Worker1)
                                         registry[(proj1, generate_pdf)].Include(Worker1)
Job A: RUNNING on Worker1

── Job B (generate_pdf) arrives ────────────────────────────
registry[(proj1, generate_pdf)].Acquire() → Worker1
Worker1.CurrentJobCount = 2  (1 slot remains)
Still under limit → Include on all instances

── Job C (send_email) arrives ──────────────────────────────
registry[(proj1, send_email)].Acquire() → Worker1
Worker1.CurrentJobCount = 3  (0 slots remain — AT CAPACITY)
registry[(proj1, send_email)].Exclude(Worker1)
registry[(proj1, generate_pdf)].Exclude(Worker1)
Worker1 excluded from all pools

── Job D (send_email) arrives ──────────────────────────────
registry[(proj1, send_email)].Acquire() → ErrNoResourceAvailable
Job D stays READY

── Job A completes ─────────────────────────────────────────
Worker1.CurrentJobCount = 2  (below capacity)
registry[(proj1, send_email)].Include(Worker1)
registry[(proj1, generate_pdf)].Include(Worker1)
Scheduler re-checks READY jobs → dispatches Job D to Worker1
```

---

## Summary: CRS API Call Reference

| CRS Method | Called By | When |
|---|---|---|
| `scheduler.New(cfg)` | Registry | First time a capability is seen for a project |
| `sched.Add(worker)` | Worker registration | New worker joins a capability pool |
| `sched.BatchAdd(workers)` | Startup | Rebuilding all pools from PostgreSQL |
| `sched.Acquire()` | Scheduler | Dispatching a READY job |
| `sched.Include(id)` | Worker system | Worker frees a slot; re-enter pools |
| `sched.Exclude(id)` | Worker system | Worker hits concurrency limit; leave pools |
| `sched.Update(worker)` | Worker system | Worker's job count changed while in heap |
| `sched.Remove(id)` | Worker system | Worker goes OFFLINE permanently |
| `sched.Shutdown()` | Server shutdown | Clean up the instance on server stop |

---

## Related Documents

| Question | Document |
|---|---|
| Where does CRS fit in the overall architecture? | `03-ARCHITECTURE.md` — Layer 5 |
| How does the scheduler use CRS to dispatch jobs? | `06-JOB-SCHEDULING.md` — Part 3 & 4 |
| What triggers Include/Exclude in the worker lifecycle? | `03-ARCHITECTURE.md` — Layer 3 |
| How is CRS rebuilt after a server restart? | `08-RELIABILITY.md` — Server Restart Recovery |
| What are the CRS requirements? | `02-REQUIREMENTS.md` — FR-042 to FR-049 |
