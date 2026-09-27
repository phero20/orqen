# 04 — Data Model

> **Document purpose:** Defines the complete PostgreSQL schema — every table, column, type, index, constraint, and relationship.  
> **Read before:** Any database query, migration, or ORM model work.  
> **Reference when:** Writing SQL, adding a new field, or understanding entity relationships.

---

## Overview

PostgreSQL is the **sole durable source of truth** for all persistent state in Orqen. There is no Redis, no secondary store. CRS is in-memory only and is rebuilt from this schema on startup.

### Entity Relationship Summary

```
projects
    │
    ├── api_keys           (one project → many keys)
    │
    ├── workers            (one project → many workers)
    │   └── worker_capabilities  (one worker → many capabilities)
    │
    └── jobs               (one project → many jobs)
        ├── job_attempts   (one job → many attempts)
        ├── job_dependencies  (one job → many dependency edges)
        └── workflows      (one workflow → many jobs)
```

---

## Tables

---

### `projects`

Stores every project in the deployment.

```sql
CREATE TABLE projects (
    id           TEXT        PRIMARY KEY,           -- e.g. proj_abc123
    name         TEXT        NOT NULL,
    created_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at   TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

| Column | Type | Description |
|---|---|---|
| `id` | `TEXT` | Unique project identifier, prefixed `proj_` |
| `name` | `TEXT` | Human-readable project name |
| `created_at` | `TIMESTAMPTZ` | When the project was created |
| `updated_at` | `TIMESTAMPTZ` | Last time the project was modified |

**Indexes:**
```sql
-- Primary key index on id (implicit)
```

---

### `api_keys`

Stores hashed API keys scoped to projects. Plaintext keys are never stored.

```sql
CREATE TABLE api_keys (
    id           TEXT        PRIMARY KEY,           -- e.g. key_xyz789
    project_id   TEXT        NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    key_hash     TEXT        NOT NULL UNIQUE,       -- bcrypt/sha256 hash of the key
    key_prefix   TEXT        NOT NULL,              -- first 8 chars for display (e.g. orqen_li)
    name         TEXT,                              -- optional human label
    is_active    BOOLEAN     NOT NULL DEFAULT true,
    expires_at   TIMESTAMPTZ,                       -- NULL = never expires
    last_used_at TIMESTAMPTZ,
    created_at   TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

| Column | Type | Description |
|---|---|---|
| `id` | `TEXT` | Unique key record identifier |
| `project_id` | `TEXT` | Owning project (FK → projects) |
| `key_hash` | `TEXT` | Hashed value of the API key — never the plaintext |
| `key_prefix` | `TEXT` | First few characters for display in the dashboard |
| `name` | `TEXT` | Optional label (e.g. "Production key") |
| `is_active` | `BOOLEAN` | False = revoked |
| `expires_at` | `TIMESTAMPTZ` | Optional expiry; NULL = no expiry |
| `last_used_at` | `TIMESTAMPTZ` | Updated on every successful auth |
| `created_at` | `TIMESTAMPTZ` | When the key was generated |

**Indexes:**
```sql
CREATE INDEX idx_api_keys_project_id ON api_keys(project_id);
CREATE UNIQUE INDEX idx_api_keys_key_hash ON api_keys(key_hash);
```

**Rules:**
- The plaintext key is returned once at creation time and never stored
- Revoking sets `is_active = false`
- Auth lookup: hash the incoming key, query `WHERE key_hash = $1 AND is_active = true`

---

### `workers`

Stores registered worker instances.

```sql
CREATE TABLE workers (
    id                TEXT        PRIMARY KEY,      -- e.g. worker_abc123
    project_id        TEXT        NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    status            TEXT        NOT NULL DEFAULT 'IDLE',  -- IDLE | BUSY | OFFLINE
    concurrency       INTEGER     NOT NULL DEFAULT 1,
    current_job_count INTEGER     NOT NULL DEFAULT 0,
    last_heartbeat_at TIMESTAMPTZ,
    registered_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

| Column | Type | Description |
|---|---|---|
| `id` | `TEXT` | Unique worker identifier |
| `project_id` | `TEXT` | Owning project (FK → projects) |
| `status` | `TEXT` | `IDLE`, `BUSY`, or `OFFLINE` |
| `concurrency` | `INTEGER` | Maximum number of parallel jobs this worker can run |
| `current_job_count` | `INTEGER` | Currently assigned jobs — must never exceed `concurrency` |
| `last_heartbeat_at` | `TIMESTAMPTZ` | Timestamp of last heartbeat received |
| `registered_at` | `TIMESTAMPTZ` | When the worker first registered |
| `updated_at` | `TIMESTAMPTZ` | Last status change |

**Indexes:**
```sql
CREATE INDEX idx_workers_project_id ON workers(project_id);
CREATE INDEX idx_workers_status ON workers(project_id, status);
CREATE INDEX idx_workers_heartbeat ON workers(last_heartbeat_at) WHERE status != 'OFFLINE';
```

**Status rules:**
- `IDLE` — worker is registered and has available slots
- `BUSY` — worker has jobs running but still has free slots  
- `OFFLINE` — worker has not sent a heartbeat within the configured timeout window

---

### `worker_capabilities`

Maps workers to the job types they can handle. One row per worker-capability pair.

```sql
CREATE TABLE worker_capabilities (
    id          TEXT        PRIMARY KEY,
    worker_id   TEXT        NOT NULL REFERENCES workers(id) ON DELETE CASCADE,
    project_id  TEXT        NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    capability  TEXT        NOT NULL,              -- e.g. "send_email", "generate_pdf"
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),

    CONSTRAINT uq_worker_capability UNIQUE (worker_id, capability)
);
```

| Column | Type | Description |
|---|---|---|
| `id` | `TEXT` | Unique record identifier |
| `worker_id` | `TEXT` | Worker this capability belongs to (FK → workers) |
| `project_id` | `TEXT` | Denormalized for query performance (FK → projects) |
| `capability` | `TEXT` | Job type string this worker handles |
| `created_at` | `TIMESTAMPTZ` | When the capability was registered |

**Indexes:**
```sql
CREATE INDEX idx_worker_capabilities_worker_id ON worker_capabilities(worker_id);
CREATE INDEX idx_worker_capabilities_project_capability ON worker_capabilities(project_id, capability);
```

**Note:** `project_id` is denormalized here to allow efficient queries like "give me all active workers for capability X in project Y" without a join through `workers`.

---

### `jobs`

The core table. Stores every job with its full state.

```sql
CREATE TABLE jobs (
    id              TEXT        PRIMARY KEY,        -- e.g. job_abc123
    project_id      TEXT        NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    workflow_id     TEXT        REFERENCES workflows(id) ON DELETE SET NULL,
    job_type        TEXT        NOT NULL,           -- e.g. "send_email"
    payload         JSONB       NOT NULL DEFAULT '{}',
    status          TEXT        NOT NULL DEFAULT 'QUEUED',
    priority        INTEGER     NOT NULL DEFAULT 0, -- higher = more important
    max_retries     INTEGER     NOT NULL DEFAULT 3,
    attempt_count   INTEGER     NOT NULL DEFAULT 0,
    timeout_seconds INTEGER,                        -- NULL = no timeout
    backoff_type    TEXT        NOT NULL DEFAULT 'exponential', -- fixed | exponential
    backoff_seconds INTEGER     NOT NULL DEFAULT 5,
    scheduled_at    TIMESTAMPTZ,                    -- NULL = run immediately
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    result          JSONB,                          -- set on COMPLETED
    error           TEXT,                           -- set on FAILED
    worker_id       TEXT        REFERENCES workers(id) ON DELETE SET NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

| Column | Type | Description |
|---|---|---|
| `id` | `TEXT` | Unique job identifier |
| `project_id` | `TEXT` | Owning project |
| `workflow_id` | `TEXT` | Optional — the DAG this job belongs to |
| `job_type` | `TEXT` | The capability required to execute this job |
| `payload` | `JSONB` | Input data passed to the worker function |
| `status` | `TEXT` | Current state — see state machine in `03-ARCHITECTURE.md` |
| `priority` | `INTEGER` | Higher value = higher scheduling priority |
| `max_retries` | `INTEGER` | Maximum retry attempts before marking FAILED |
| `attempt_count` | `INTEGER` | How many attempts have been made |
| `timeout_seconds` | `INTEGER` | Max execution time per attempt; NULL = unlimited |
| `backoff_type` | `TEXT` | `fixed` or `exponential` |
| `backoff_seconds` | `INTEGER` | Base backoff duration in seconds |
| `scheduled_at` | `TIMESTAMPTZ` | Do not schedule before this time; NULL = immediate |
| `started_at` | `TIMESTAMPTZ` | When the most recent attempt started |
| `completed_at` | `TIMESTAMPTZ` | When the job reached a terminal state |
| `result` | `JSONB` | Output payload from successful execution |
| `error` | `TEXT` | Error message from final failed attempt |
| `worker_id` | `TEXT` | Worker currently executing the job (nullable) |
| `created_at` | `TIMESTAMPTZ` | When the job was enqueued |
| `updated_at` | `TIMESTAMPTZ` | Last state change |

**Indexes:**
```sql
-- Scheduler: find READY jobs by project and priority
CREATE INDEX idx_jobs_scheduler ON jobs(project_id, status, priority DESC, created_at ASC)
    WHERE status = 'READY';

-- DAG engine: find WAITING jobs by project
CREATE INDEX idx_jobs_waiting ON jobs(project_id, status)
    WHERE status = 'WAITING_FOR_DEPENDENCY';

-- Heartbeat recovery: find RUNNING jobs assigned to a specific worker
CREATE INDEX idx_jobs_worker ON jobs(worker_id, status)
    WHERE status = 'RUNNING';

-- Workflow queries
CREATE INDEX idx_jobs_workflow ON jobs(workflow_id) WHERE workflow_id IS NOT NULL;

-- General project queries
CREATE INDEX idx_jobs_project_id ON jobs(project_id, created_at DESC);
```

---

### `job_attempts`

Records every individual execution attempt for a job. Used for retry tracking, duplicate result detection, and audit history.

```sql
CREATE TABLE job_attempts (
    id           TEXT        PRIMARY KEY,           -- e.g. attempt_abc123
    job_id       TEXT        NOT NULL REFERENCES jobs(id) ON DELETE CASCADE,
    project_id   TEXT        NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    worker_id    TEXT        REFERENCES workers(id) ON DELETE SET NULL,
    attempt_num  INTEGER     NOT NULL,              -- 1-indexed attempt number
    status       TEXT        NOT NULL DEFAULT 'RUNNING', -- RUNNING | SUCCESS | FAILED | TIMEOUT
    started_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at TIMESTAMPTZ,
    result       JSONB,
    error        TEXT,

    CONSTRAINT uq_job_attempt_num UNIQUE (job_id, attempt_num)
);
```

| Column | Type | Description |
|---|---|---|
| `id` | `TEXT` | Unique attempt identifier |
| `job_id` | `TEXT` | The job this attempt belongs to |
| `project_id` | `TEXT` | Denormalized for query performance |
| `worker_id` | `TEXT` | Worker that executed this attempt |
| `attempt_num` | `INTEGER` | Attempt number (1 = first try, 2 = first retry, etc.) |
| `status` | `TEXT` | `RUNNING`, `SUCCESS`, `FAILED`, `TIMEOUT` |
| `started_at` | `TIMESTAMPTZ` | When this attempt began |
| `completed_at` | `TIMESTAMPTZ` | When this attempt ended |
| `result` | `JSONB` | Result from this specific attempt |
| `error` | `TEXT` | Error from this specific attempt |

**Indexes:**
```sql
CREATE INDEX idx_job_attempts_job_id ON job_attempts(job_id, attempt_num DESC);
CREATE INDEX idx_job_attempts_worker ON job_attempts(worker_id, status) WHERE status = 'RUNNING';
```

**Usage:**
- When a worker submits a result, the system checks if `attempt_num` matches the current expected attempt on the job. If not, the result is a late/duplicate submission and is ignored.

---

### `job_dependencies`

Stores dependency edges between jobs. One row per upstream → downstream relationship.

```sql
CREATE TABLE job_dependencies (
    id              TEXT        PRIMARY KEY,
    project_id      TEXT        NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    upstream_job_id TEXT        NOT NULL REFERENCES jobs(id) ON DELETE CASCADE,
    downstream_job_id TEXT      NOT NULL REFERENCES jobs(id) ON DELETE CASCADE,
    is_satisfied    BOOLEAN     NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    satisfied_at    TIMESTAMPTZ,

    CONSTRAINT uq_dependency_edge UNIQUE (upstream_job_id, downstream_job_id)
);
```

| Column | Type | Description |
|---|---|---|
| `id` | `TEXT` | Unique edge identifier |
| `project_id` | `TEXT` | Owning project |
| `upstream_job_id` | `TEXT` | Job that must complete first |
| `downstream_job_id` | `TEXT` | Job that is waiting |
| `is_satisfied` | `BOOLEAN` | True when the upstream job has completed |
| `created_at` | `TIMESTAMPTZ` | When the dependency was declared |
| `satisfied_at` | `TIMESTAMPTZ` | When the upstream job completed |

**Indexes:**
```sql
-- DAG engine: find all unsatisfied dependencies for a downstream job
CREATE INDEX idx_dependencies_downstream ON job_dependencies(downstream_job_id, is_satisfied);

-- DAG engine: when upstream completes, find all its downstream jobs
CREATE INDEX idx_dependencies_upstream ON job_dependencies(upstream_job_id) WHERE is_satisfied = false;
```

**Cycle detection:** Performed at dependency creation time by traversing the graph in the application layer — not enforced at the DB level.

---

### `workflows`

Stores DAG/workflow definitions and their overall completion state.

```sql
CREATE TABLE workflows (
    id           TEXT        PRIMARY KEY,           -- e.g. wf_abc123
    project_id   TEXT        NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    name         TEXT,
    status       TEXT        NOT NULL DEFAULT 'RUNNING', -- RUNNING | COMPLETED | FAILED | CANCELLED
    created_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at TIMESTAMPTZ
);
```

| Column | Type | Description |
|---|---|---|
| `id` | `TEXT` | Unique workflow identifier |
| `project_id` | `TEXT` | Owning project |
| `name` | `TEXT` | Optional human-readable name |
| `status` | `TEXT` | Overall workflow state |
| `created_at` | `TIMESTAMPTZ` | When the workflow was created |
| `updated_at` | `TIMESTAMPTZ` | Last status change |
| `completed_at` | `TIMESTAMPTZ` | When all jobs in the workflow finished |

**Indexes:**
```sql
CREATE INDEX idx_workflows_project_id ON workflows(project_id, status);
```

---

## ID Format Convention

All entity IDs use a prefixed, random string format:

| Entity | Prefix | Example |
|---|---|---|
| Project | `proj_` | `proj_k8x2mn4p` |
| API Key record | `key_` | `key_9z1qvrt5` |
| Worker | `worker_` | `worker_7j3nbf2a` |
| Job | `job_` | `job_4p8ycw6m` |
| Job Attempt | `attempt_` | `attempt_2x5hqe9r` |
| Dependency edge | `dep_` | `dep_1a7klp3n` |
| Workflow | `wf_` | `wf_6m2tcd8v` |

IDs are generated application-side (not via PostgreSQL sequences) using a cryptographically random suffix. This keeps IDs consistent across environments and avoids sequential enumeration.

---

## Enum Values Reference

### Job Status
```
CREATED | QUEUED | WAITING_FOR_DEPENDENCY | READY | RUNNING | COMPLETED | FAILED | RETRYING | CANCELLED
```

### Worker Status
```
IDLE | BUSY | OFFLINE
```

### Job Attempt Status
```
RUNNING | SUCCESS | FAILED | TIMEOUT
```

### Workflow Status
```
RUNNING | COMPLETED | FAILED | CANCELLED
```

### Backoff Type
```
fixed | exponential
```

---

## Key Query Patterns

### Authenticate an API key
```sql
SELECT project_id FROM api_keys
WHERE key_hash = $1
  AND is_active = true
  AND (expires_at IS NULL OR expires_at > now());
```

### Find READY jobs for scheduling (per project)
```sql
SELECT * FROM jobs
WHERE project_id = $1
  AND status = 'READY'
  AND (scheduled_at IS NULL OR scheduled_at <= now())
ORDER BY priority DESC, created_at ASC
LIMIT $2;
```

### Check if a downstream job's dependencies are all satisfied
```sql
SELECT COUNT(*) FROM job_dependencies
WHERE downstream_job_id = $1
  AND is_satisfied = false;
-- If 0, transition the job to READY
```

### Find jobs held by a dead worker (heartbeat recovery)
```sql
SELECT id FROM jobs
WHERE worker_id = $1
  AND status = 'RUNNING';
```

### Rebuild CRS from PostgreSQL on startup
```sql
SELECT w.id, w.concurrency, w.current_job_count, wc.capability
FROM workers w
JOIN worker_capabilities wc ON wc.worker_id = w.id
WHERE w.project_id = $1
  AND w.status != 'OFFLINE';
-- For each row: sched.BatchAdd(worker) into the correct CRS instance
```

---

## Data Integrity Rules

| Rule | Enforcement |
|---|---|
| A job can only belong to one project | `project_id` FK + all queries filter by it |
| A worker can only belong to one project | `project_id` FK on `workers` |
| A worker's `current_job_count` must never exceed `concurrency` | Application-layer check before dispatch |
| API key plaintext is never stored | Application layer hashes before insert |
| Dependency graphs must be acyclic | Application-layer cycle detection before insert |
| A job attempt number must be unique per job | `UNIQUE(job_id, attempt_num)` constraint |
| Dependency edges must be unique | `UNIQUE(upstream_job_id, downstream_job_id)` constraint |

---

## Related Documents

| Question | Document |
|---|---|
| How does the job state machine work? | `03-ARCHITECTURE.md` — Layer 2 |
| How is CRS rebuilt from this schema on restart? | `07-CRS-INTEGRATION.md` |
| How are dependencies resolved at runtime? | `06-JOB-SCHEDULING.md` |
| What reliability mechanisms use this schema? | `08-RELIABILITY.md` |
| How does project isolation apply to queries? | `11-PROJECT-ISOLATION.md` |
