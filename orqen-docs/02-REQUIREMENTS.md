# 02 — Requirements

> **Document purpose:** Defines all functional and non-functional requirements for Orqen V1.  
> **Read before:** Any implementation work.  
> **Reference when:** Making scope decisions, reviewing a feature, or checking if something is in V1.

---

## Requirement ID Convention

- `FR-XXX` — Functional Requirement
- `NFR-XXX` — Non-Functional Requirement
- `SEC-XXX` — Security Requirement

Status: `MUST` = required for V1 launch. `SHOULD` = strongly desired. `MAY` = optional enhancement.

---

## 1. Project & Control Layer

| ID | Status | Requirement |
|---|---|---|
| FR-001 | MUST | The system must support creating multiple independent projects inside one deployment |
| FR-002 | MUST | Each project must have a unique project ID |
| FR-003 | MUST | The system must generate one or more API keys per project on creation |
| FR-004 | MUST | API keys must be revocable — revoking a key must immediately invalidate it |
| FR-005 | MUST | API keys must be rotatable — rotating replaces the old key with a new one |
| FR-006 | MUST | The system must support deleting a project and all its associated data |
| FR-007 | MUST | Every inbound API request must be authenticated using an API key |
| FR-008 | MUST | The server must derive project context from the authenticated API key, never from a client-supplied project ID |
| FR-009 | MUST | Project A must not be able to access any resource belonging to Project B |

---

## 2. Job System

| ID | Status | Requirement |
|---|---|---|
| FR-010 | MUST | The system must accept job enqueue requests containing a job type and a JSON payload |
| FR-011 | MUST | Jobs must be stored durably in PostgreSQL immediately upon enqueue |
| FR-012 | MUST | Each job must have a unique job ID |
| FR-013 | MUST | Jobs must support a priority field that influences scheduling order |
| FR-014 | MUST | The system must track job status through a formal state machine |
| FR-015 | MUST | Valid job statuses are: `CREATED`, `QUEUED`, `WAITING_FOR_DEPENDENCY`, `READY`, `RUNNING`, `COMPLETED`, `FAILED`, `RETRYING`, `CANCELLED` |
| FR-016 | MUST | Jobs must support cancellation from `QUEUED` or `READY` state |
| FR-017 | MUST | The system must store job results on successful completion |
| FR-018 | MUST | The system must store error information on job failure |
| FR-019 | MUST | Jobs must support configurable `maxRetries`, `backoff`, and `timeout` fields |
| FR-020 | MUST | Each job must maintain an attempt log — a record of every individual execution attempt |
| FR-021 | SHOULD | Jobs must support an optional scheduled delay (run no earlier than a given time) |

---

## 3. Worker System

| ID | Status | Requirement |
|---|---|---|
| FR-022 | MUST | Workers must be able to register themselves with the Orqen server |
| FR-023 | MUST | Workers must declare their capabilities (job types they can handle) at registration |
| FR-024 | MUST | Workers must declare a concurrency limit at registration |
| FR-025 | MUST | Workers must send periodic heartbeats to signal liveness |
| FR-026 | MUST | The system must track each worker's current job count against its concurrency limit |
| FR-027 | MUST | The system must support graceful worker shutdown — in-flight jobs are not lost |
| FR-028 | MUST | Workers must be scoped to a project — a worker registered under Project A cannot receive Project B jobs |
| FR-029 | MUST | The system must support multiple workers per project with overlapping capabilities |
| FR-030 | SHOULD | Workers should be able to update their concurrency limit after registration |

---

## 4. Worker Communication Protocol

| ID | Status | Requirement |
|---|---|---|
| FR-031 | MUST | The worker communication protocol must be language-independent |
| FR-032 | MUST | Workers must receive jobs via HTTP long polling |
| FR-033 | MUST | The long-poll connection must remain open until a job is available or a timeout is reached |
| FR-034 | MUST | Workers must acknowledge job receipt before beginning execution |
| FR-035 | MUST | Workers must be able to submit a success result with a JSON payload |
| FR-036 | MUST | Workers must be able to submit a failure result with an error message |
| FR-037 | MUST | Workers must be able to reconnect after a network interruption without losing their registration |
| FR-038 | MUST | The protocol must support graceful shutdown signaling from worker to server |

---

## 5. Scheduler & CRS

| ID | Status | Requirement |
|---|---|---|
| FR-039 | MUST | Only jobs with status `READY` must enter the scheduler |
| FR-040 | MUST | The scheduler must match jobs to workers based on job type and worker capability |
| FR-041 | MUST | The scheduler must not assign a job to a worker that has reached its concurrency limit |
| FR-042 | MUST | Worker selection must be powered by the CRS library (`github.com/phero20/concurrent-resource-scheduler`) |
| FR-043 | MUST | One CRS scheduler instance must be maintained per `(project_id, capability)` pair |
| FR-044 | MUST | CRS instances must be lazily initialized when a new capability is first registered |
| FR-045 | MUST | When a worker reaches its concurrency limit, it must be excluded from all its CRS capability pools |
| FR-046 | MUST | When a worker frees a slot, it must be re-included in all its CRS capability pools |
| FR-047 | MUST | CRS state must be fully reconstructable from PostgreSQL on server restart via `BatchAdd` |
| FR-048 | SHOULD | The scheduler should support priority-based worker selection within a capability pool |
| FR-049 | SHOULD | The scheduler should support affinity-based routing for session-scoped jobs |

---

## 6. Dependency & DAG Engine

| ID | Status | Requirement |
|---|---|---|
| FR-050 | MUST | Jobs must be able to declare one or more upstream job dependencies |
| FR-051 | MUST | A job with unresolved dependencies must have status `WAITING_FOR_DEPENDENCY` and must not be scheduled |
| FR-052 | MUST | When all upstream dependencies complete, the downstream job must transition to `READY` automatically |
| FR-053 | MUST | The system must validate dependency graphs at creation time and reject cyclic graphs |
| FR-054 | MUST | The system must support multi-level dependency chains (A → B → C) |
| FR-055 | MUST | The system must support fan-out and fan-in DAG patterns (A → B, A → C, B+C → D) |
| FR-056 | MUST | The system must track the completion state of every dependency edge |
| FR-057 | MUST | The system must define and enforce a policy for what happens when an upstream job fails (block, propagate failure, or continue) |
| FR-058 | MUST | The system must track overall workflow completion state |

---

## 7. Reliability

| ID | Status | Requirement |
|---|---|---|
| FR-059 | MUST | Failed jobs must be automatically retried up to `maxRetries` times |
| FR-060 | MUST | Retry attempts must respect the configured `backoff` strategy |
| FR-061 | MUST | Each retry must be recorded as a new attempt in the attempt log |
| FR-062 | MUST | The system must detect worker failure via missed heartbeat |
| FR-063 | MUST | Jobs held by a failed worker must be recovered and requeued |
| FR-064 | MUST | The system must handle late results — a result arriving for an attempt that has already been superseded must be ignored |
| FR-065 | MUST | The system must handle duplicate results — the same attempt submitting a result more than once must be idempotent |
| FR-066 | MUST | After server restart, all unfinished jobs must be recovered from PostgreSQL |
| FR-067 | MUST | Jobs must never depend solely on in-memory state for durability |

---

## 8. Dashboard

| ID | Status | Requirement |
|---|---|---|
| FR-068 | MUST | The dashboard must be a web UI accessible via browser |
| FR-069 | MUST | The dashboard must show a list of all projects |
| FR-070 | MUST | Per project, the dashboard must show: Jobs, Workers, Workflows, API Keys pages |
| FR-071 | MUST | The Jobs page must show: job ID, type, status, priority, attempts, created at, updated at |
| FR-072 | MUST | The Workers page must show: worker ID, status, capabilities, concurrency, current job count, last heartbeat |
| FR-073 | MUST | The Workflows page must show a visual DAG representation with per-node status |
| FR-074 | MUST | The API Keys page must allow creating, viewing, and revoking keys |
| FR-075 | SHOULD | The dashboard should support real-time updates without full page refresh |
| FR-076 | SHOULD | The Jobs page should support filtering by status and job type |

---

## 9. Non-Functional Requirements

### Performance

| ID | Status | Requirement |
|---|---|---|
| NFR-001 | MUST | Job enqueue must complete within 100ms under normal load |
| NFR-002 | MUST | Job dispatch to worker (from READY to worker receiving) must complete within 500ms under normal load |
| NFR-003 | MUST | The scheduler must not perform O(N) linear scans over all workers to find a match |
| NFR-004 | SHOULD | The system should support at least 1,000 concurrent workers per deployment in V1 |
| NFR-005 | SHOULD | CRS acquire operations must complete in sub-millisecond time under normal load |

### Reliability

| ID | Status | Requirement |
|---|---|---|
| NFR-006 | MUST | No job must be permanently lost due to a server restart |
| NFR-007 | MUST | No job must be permanently lost due to a worker crash |
| NFR-008 | MUST | Duplicate result submissions must not corrupt job state |
| NFR-009 | MUST | The system must recover in-flight jobs after server restart within 30 seconds |

### Scalability

| ID | Status | Requirement |
|---|---|---|
| NFR-010 | SHOULD | Adding more workers to a capability pool must not degrade scheduler performance |
| NFR-011 | SHOULD | Adding more projects must not degrade performance for existing projects |
| NFR-012 | MAY | The system may support horizontal scaling of the API server in a future version |

### Operability

| ID | Status | Requirement |
|---|---|---|
| NFR-013 | MUST | The system must expose structured logs |
| NFR-014 | MUST | The system must be configurable via environment variables |
| NFR-015 | SHOULD | The system should expose a health check endpoint |
| NFR-016 | SHOULD | The system should expose metrics compatible with Prometheus |

---

## 10. Security Requirements

| ID | Status | Requirement |
|---|---|---|
| SEC-001 | MUST | All API requests must be authenticated via an API key in the request header |
| SEC-002 | MUST | API keys must be stored hashed — plaintext keys must never be stored in the database |
| SEC-003 | MUST | The server must never trust a project ID supplied by the client — project context must always be derived from the authenticated API key |
| SEC-004 | MUST | Every database query that accesses project-scoped resources must include a `project_id` filter |
| SEC-005 | MUST | A worker authenticated under Project A must never receive jobs belonging to Project B |
| SEC-006 | MUST | The dashboard must enforce project-level access — a user viewing Project A must not see Project B data |
| SEC-007 | MUST | Job payloads must be treated as opaque data — Orqen must not inspect, log, or expose payload contents beyond what is needed for job management |
| SEC-008 | SHOULD | API keys should have configurable expiry |
| SEC-009 | SHOULD | All inter-service communication should use TLS in production deployments |

---

## 11. Constraints

| Constraint | Description |
|---|---|
| Language | Backend must be written in Go |
| Database | PostgreSQL is the only supported database in V1 |
| Worker protocol | Long polling over HTTP — no WebSocket or gRPC in V1 |
| Scheduling library | CRS (`github.com/phero20/concurrent-resource-scheduler`) — no other scheduler library |
| Self-hosted only | No managed cloud offering in V1 |
| No function execution | Orqen must never execute customer function code |

---

## Related Documents

| Question | Document |
|---|---|
| What is Orqen and why does it exist? | `01-PROJECT.md` |
| How is the system structured to meet these requirements? | `03-ARCHITECTURE.md` |
| What does the data model look like? | `04-DATA-MODEL.md` |
| How does CRS specifically satisfy FR-042 to FR-049? | `07-CRS-INTEGRATION.md` |
| How does reliability satisfy FR-059 to FR-067? | `08-RELIABILITY.md` |
| How does isolation satisfy SEC-001 to SEC-006? | `11-PROJECT-ISOLATION.md` |
