# 11 — Project Isolation

> **Document purpose:** Defines the complete multi-project isolation model — what the rules are, where they are enforced, and a checklist for verifying isolation in any new feature.  
> **Read before:** Implementing any endpoint, query, middleware, or feature that touches project-scoped resources.  
> **Reference when:** Adding a new resource type, reviewing a cross-project security concern, or auditing isolation correctness.

---

## The Isolation Guarantee

> **Project A must never be able to read, write, or affect any resource belonging to Project B.**

This guarantee applies to:
- Jobs
- Workers
- Workflows
- Dependencies
- API Keys
- Dashboard views
- Scheduler / CRS pools
- Any future resource type

The isolation model in V1 is **logical isolation via `project_id` scoping**. All resources share the same PostgreSQL database and the same Orqen server process. Isolation is enforced at the application layer — not through separate databases, schemas, or network boundaries.

---

## The Two Isolation Rules

Every isolation decision in Orqen follows from two rules:

### Rule 1 — Project context is derived from the API key, never from the client

```
✅ CORRECT:
  Client sends: Authorization: Bearer orqen_live_xxx
  Server resolves: project_id = proj_abc123  (from API key lookup)
  Server uses: proj_abc123 for all downstream operations

❌ WRONG:
  Client sends: { "project_id": "proj_abc123" }
  Server uses: proj_abc123 directly  ← NEVER DO THIS
```

A client supplying a `project_id` in a request body or query param must never be trusted for authorization. The project context must always come from the authenticated API key.

### Rule 2 — Every database query on a project-scoped resource must include a `project_id` filter

```sql
-- ✅ CORRECT
SELECT * FROM jobs WHERE id = $1 AND project_id = $2;

-- ❌ WRONG — no project_id filter
SELECT * FROM jobs WHERE id = $1;
```

If a query omits the `project_id` filter, it is possible for Project A to read or mutate Project B's data by guessing or supplying a valid resource ID. All IDs are random — but defense must not rely on ID unguessability.

---

## Enforcement Points

Isolation is enforced at **every layer** independently. A failure at one layer must not expose data — defense in depth.

### Layer 1 — Authentication Middleware

Every inbound API request passes through auth middleware before reaching any handler.

```
Inbound request
      │
      ▼
Extract API key from Authorization header
      │
      ▼
Key missing or malformed?
  → 401 Unauthorized (stop)
      │
      ▼
Lookup: SELECT project_id FROM api_keys
        WHERE key_hash = hash(key)
          AND is_active = true
          AND (expires_at IS NULL OR expires_at > now())
      │
      ▼
No match?
  → 401 Unauthorized (stop)
      │
      ▼
Inject project_id into request context
      │
      ▼
Handler executes with project_id from context
```

**Rule:** Handlers must read `project_id` from the request context injected by middleware. They must never accept a `project_id` from the request body or query params for authorization purposes.

---

### Layer 2 — Database Query Layer

Every query that accesses a project-scoped resource must include `WHERE project_id = $project_id` as a mandatory filter.

**Project-scoped tables:**

| Table | project_id column | Notes |
|---|---|---|
| `workers` | `project_id` | Required on all queries |
| `worker_capabilities` | `project_id` | Denormalized for join-free filtering |
| `jobs` | `project_id` | Required on all queries |
| `job_attempts` | `project_id` | Denormalized for query performance |
| `job_dependencies` | `project_id` | Required on all queries |
| `workflows` | `project_id` | Required on all queries |
| `api_keys` | `project_id` | Required on all queries |

**Pattern every query must follow:**

```sql
-- Get a single resource — always include project_id
SELECT * FROM jobs
WHERE id = $1 AND project_id = $2;

-- List resources — always include project_id
SELECT * FROM jobs
WHERE project_id = $1
  AND status = $2
ORDER BY created_at DESC;

-- Update — always include project_id
UPDATE jobs
SET status = $1, updated_at = now()
WHERE id = $2 AND project_id = $3;

-- Delete — always include project_id
DELETE FROM job_dependencies
WHERE upstream_job_id = $1 AND project_id = $2;
```

**Cross-table queries:** When joining tables, the `project_id` filter must be applied on the root table. Denormalized `project_id` columns on child tables allow direct filtering without full joins.

---

### Layer 3 — Worker Assignment

Workers are project-scoped. A worker registered under Project A must only receive jobs from Project A.

**Enforcement:**
- On `POST /worker/register`: the worker is created with `project_id` derived from the API key — never from the request body
- On `GET /worker/poll`: the server looks up the polling worker by `(worker_id, project_id)` — if the `worker_id` exists but belongs to a different project, return `404 Not Found` (same as not found — do not reveal existence)
- On job dispatch: the scheduler only pulls `READY` jobs for the authenticated project, and only passes them to workers registered under that same project

```sql
-- Correct worker poll lookup
SELECT * FROM workers
WHERE id = $worker_id AND project_id = $project_id;
```

---

### Layer 4 — CRS Scheduler Pools

CRS instances are keyed by `(project_id, capability)`. There is no shared state between projects in the CRS registry.

```
registry[(proj_abc123, "send_email")]  → completely separate from
registry[(proj_xyz789, "send_email")]
```

A job from Project A calling `registry[(proj_abc123, "send_email")].Acquire()` can only ever receive a worker from Project A's pool. There is no mechanism by which a Project B worker could be selected.

---

### Layer 5 — API Handler Layer

Every handler must validate resource ownership before operating on it.

**Pattern:**

```
Handler: GET /api/v1/jobs/:job_id

1. Extract project_id from request context (set by auth middleware)
2. Query: SELECT * FROM jobs WHERE id = $job_id AND project_id = $project_id
3. Row not found (either doesn't exist OR belongs to another project)?
   → Return 404 Not Found
   → Do NOT return 403 Forbidden (that would confirm the resource exists)
4. Row found? Return job data
```

**Why 404 and not 403?**
Returning `403 Forbidden` would confirm to the caller that the resource exists but they don't have access. This leaks information. Always return `404 Not Found` when a resource doesn't exist in the authenticated project — whether it truly doesn't exist or belongs to another project.

---

### Layer 6 — Dashboard

The dashboard session must be scoped to a single project. A dashboard user viewing Project A must never see Project B data.

**Enforcement:**
- Dashboard API calls use the project's API key for authentication
- All data returned is filtered by the authenticated project
- The project selector in the UI changes which API key is active for requests
- No "view all projects" data endpoint exists at the project API level — only at the admin level

---

## Cross-Resource Validation

When a request involves resources from multiple entities (e.g. creating a dependency between two jobs), both resources must belong to the same project.

### Example: Creating a job dependency

```
POST /api/v1/jobs/:downstream_job_id/dependencies
Body: { "depends_on": ["job_upstream1"] }
Authenticated project: proj_abc123

Validation:
1. Verify downstream_job_id exists WHERE project_id = proj_abc123 → 404 if not
2. For each upstream job ID:
   Verify upstream_job_id exists WHERE project_id = proj_abc123 → 400 if not
3. Both exist in the same project → proceed
```

An upstream job from a different project must never be linkable as a dependency — this would leak cross-project information and create a cross-project execution coupling.

---

## What Isolation Does NOT Cover in V1

V1 isolation is logical, not physical. Be aware of these limitations:

| Limitation | Explanation |
|---|---|
| **Shared database** | All projects share one PostgreSQL instance. A DB-level breach exposes all projects. |
| **Shared server process** | All projects run in one Orqen process. A server bug could theoretically leak data between projects in memory. |
| **No encryption at rest** | Job payloads are stored in plaintext JSONB. Sensitive payloads are the customer's responsibility to encrypt before enqueueing. |
| **No per-project resource limits** | In V1, one project can consume all available workers and jobs — no quotas. |
| **Admin endpoints bypass isolation** | Admin-level endpoints (project CRUD) operate above project isolation. These must be separately protected at the deployment level. |

These are accepted V1 tradeoffs. Future versions may introduce schema-per-project, payload encryption, or resource quotas.

---

## Isolation Checklist for New Features

Before shipping any new feature that adds a resource or endpoint, verify:

```
[ ] New resource table has a project_id column with FK to projects
[ ] All SELECT queries on the new table include WHERE project_id = $project_id
[ ] All UPDATE queries on the new table include AND project_id = $project_id
[ ] All DELETE queries on the new table include AND project_id = $project_id
[ ] The API handler reads project_id from request context, not request body
[ ] Resource-not-found returns 404 regardless of whether it belongs to another project
[ ] If the feature links two resources, both are validated to belong to the same project
[ ] CRS instances (if applicable) are keyed with project_id
[ ] Dashboard views for this feature are filtered by project
[ ] No new cross-project data flow is introduced
```

---

## Security Properties Summary

| Property | Status in V1 |
|---|---|
| Project A cannot read Project B jobs | ✅ Enforced (project_id filter + auth) |
| Project A cannot submit results for Project B jobs | ✅ Enforced (worker auth + project_id filter) |
| Project A worker cannot receive Project B jobs | ✅ Enforced (project-scoped dispatch) |
| Project A cannot see Project B workers | ✅ Enforced (project_id filter) |
| Project A cannot see Project B API keys | ✅ Enforced (project_id filter) |
| Project A cannot create dependencies on Project B jobs | ✅ Enforced (cross-resource validation) |
| API key plaintext never stored | ✅ Enforced (hashed on write) |
| Project context always server-derived | ✅ Enforced (middleware injection) |
| Physical data separation between projects | ❌ Not in V1 (shared DB) |
| Payload encryption at rest | ❌ Not in V1 (customer responsibility) |
| Per-project resource quotas | ❌ Not in V1 |

---

## Related Documents

| Question | Document |
|---|---|
| What are the security requirements? | `02-REQUIREMENTS.md` — SEC-001 to SEC-009 |
| What does the database schema look like? | `04-DATA-MODEL.md` |
| How does API key auth work in detail? | `09-API-REFERENCE.md` — Auth section |
| How are CRS pools isolated per project? | `07-CRS-INTEGRATION.md` — CRS Registry |
