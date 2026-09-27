# 09 — API Reference

> **Document purpose:** Complete HTTP API reference — every endpoint, auth, request/response shapes, status codes, and error formats.  
> **Read before:** Implementing any API handler, writing any SDK method that calls the server, or building the dashboard backend.  
> **Reference when:** Looking up an endpoint contract, adding a new route, or debugging an API response.

---

## Conventions

### Base URL
```
https://<orqen-host>/api/v1
```

### Authentication
All endpoints require an API key in the `Authorization` header:
```
Authorization: Bearer orqen_live_xxxxxxxxxxxxxxxxxx
```
The server resolves the project from the key. A missing or invalid key returns `401`.

**Exception:** The Projects endpoints (`/projects`) use a master admin token (deployment-level auth), not a project API key. This is for dashboard and admin use. Defined separately in each section.

### Content-Type
All request bodies are `application/json`. All responses are `application/json`.

### Timestamps
All timestamps are ISO 8601 UTC strings: `"2026-09-27T10:00:00Z"`

### ID Format
All IDs are prefixed strings: `proj_`, `job_`, `worker_`, `wf_`, `key_`, `attempt_`

---

## Error Response Format

All errors follow this structure:

```json
{
    "error": "ERROR_CODE",
    "message": "Human-readable description"
}
```

### Global Error Codes

| Code | HTTP Status | Meaning |
|---|---|---|
| `UNAUTHORIZED` | `401` | Missing or invalid API key |
| `FORBIDDEN` | `403` | Valid key but insufficient permissions |
| `NOT_FOUND` | `404` | Resource does not exist in this project |
| `INVALID_REQUEST` | `400` | Malformed body or missing required fields |
| `CONFLICT` | `409` | State conflict (see per-endpoint codes) |
| `INTERNAL_ERROR` | `500` | Unexpected server error |

---

## 1. Projects

> These endpoints use **admin-level auth** (deployment master token), not a project API key. They are used by the dashboard and CLI tooling.

### 1.1 List Projects

```
GET /api/v1/projects
```

**Response — `200 OK`**
```json
{
    "projects": [
        {
            "id": "proj_abc123",
            "name": "My E-Commerce App",
            "created_at": "2026-09-27T10:00:00Z"
        }
    ]
}
```

---

### 1.2 Create Project

```
POST /api/v1/projects
```

**Request**
```json
{
    "name": "My E-Commerce App"
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `name` | `string` | YES | Human-readable project name |

**Response — `201 Created`**
```json
{
    "id": "proj_abc123",
    "name": "My E-Commerce App",
    "api_key": "orqen_live_xxxxxxxxxxxxxxxxxx",
    "created_at": "2026-09-27T10:00:00Z"
}
```

> The `api_key` is returned **once** at creation time. It is never returned again. Store it immediately.

---

### 1.3 Get Project

```
GET /api/v1/projects/:project_id
```

**Response — `200 OK`**
```json
{
    "id": "proj_abc123",
    "name": "My E-Commerce App",
    "created_at": "2026-09-27T10:00:00Z",
    "updated_at": "2026-09-27T10:00:00Z"
}
```

---

### 1.4 Delete Project

```
DELETE /api/v1/projects/:project_id
```

Permanently deletes the project and all its data (workers, jobs, workflows, API keys).

**Response — `200 OK`**
```json
{
    "deleted": true,
    "project_id": "proj_abc123"
}
```

---

## 2. API Keys

> These endpoints use **project API key auth** (`Authorization: Bearer orqen_live_xxx`).

### 2.1 List API Keys

```
GET /api/v1/keys
```

Returns all keys for the authenticated project (hashes are never returned — only prefix and metadata).

**Response — `200 OK`**
```json
{
    "keys": [
        {
            "id": "key_xyz789",
            "name": "Production key",
            "key_prefix": "orqen_li",
            "is_active": true,
            "expires_at": null,
            "last_used_at": "2026-09-27T09:55:00Z",
            "created_at": "2026-09-01T00:00:00Z"
        }
    ]
}
```

---

### 2.2 Create API Key

```
POST /api/v1/keys
```

**Request**
```json
{
    "name": "Staging key",
    "expires_at": "2027-01-01T00:00:00Z"
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `name` | `string` | NO | Optional label |
| `expires_at` | `string` | NO | ISO 8601 expiry timestamp; null = never |

**Response — `201 Created`**
```json
{
    "id": "key_abc456",
    "name": "Staging key",
    "api_key": "orqen_live_yyyyyyyyyyyyyyyyyy",
    "key_prefix": "orqen_li",
    "expires_at": "2027-01-01T00:00:00Z",
    "created_at": "2026-09-27T10:00:00Z"
}
```

> The `api_key` is returned **once** at creation. Store it immediately.

---

### 2.3 Revoke API Key

```
DELETE /api/v1/keys/:key_id
```

Sets `is_active = false`. The key is immediately invalid.

**Response — `200 OK`**
```json
{
    "revoked": true,
    "key_id": "key_abc456"
}
```

---

## 3. Jobs

> All job endpoints use **project API key auth**.

### 3.1 Enqueue Job

```
POST /api/v1/jobs
```

**Request**
```json
{
    "job_type": "send_email",
    "payload": {
        "to": "user@example.com",
        "template": "welcome"
    },
    "priority": 5,
    "max_retries": 3,
    "timeout_seconds": 60,
    "backoff_type": "exponential",
    "backoff_seconds": 5,
    "scheduled_at": null,
    "workflow_id": null
}
```

| Field | Type | Required | Default | Description |
|---|---|---|---|---|
| `job_type` | `string` | YES | — | Capability required to execute this job |
| `payload` | `object` | NO | `{}` | Arbitrary JSON input for the worker function |
| `priority` | `integer` | NO | `0` | Higher = scheduled first |
| `max_retries` | `integer` | NO | `3` | Max retry attempts on failure |
| `timeout_seconds` | `integer` | NO | `null` | Per-attempt execution timeout; null = unlimited |
| `backoff_type` | `string` | NO | `exponential` | `fixed` or `exponential` |
| `backoff_seconds` | `integer` | NO | `5` | Base backoff duration |
| `scheduled_at` | `string` | NO | `null` | Do not schedule before this time; null = immediate |
| `workflow_id` | `string` | NO | `null` | Associate with an existing workflow |

**Response — `201 Created`**
```json
{
    "job_id": "job_4p8ycw6m",
    "job_type": "send_email",
    "status": "QUEUED",
    "priority": 5,
    "created_at": "2026-09-27T10:01:00Z"
}
```

---

### 3.2 Get Job

```
GET /api/v1/jobs/:job_id
```

**Response — `200 OK`**
```json
{
    "job_id": "job_4p8ycw6m",
    "project_id": "proj_abc123",
    "job_type": "send_email",
    "payload": { "to": "user@example.com", "template": "welcome" },
    "status": "COMPLETED",
    "priority": 5,
    "max_retries": 3,
    "attempt_count": 1,
    "timeout_seconds": 60,
    "scheduled_at": null,
    "workflow_id": null,
    "worker_id": null,
    "result": { "message_id": "msg_abc" },
    "error": null,
    "created_at": "2026-09-27T10:01:00Z",
    "started_at": "2026-09-27T10:01:02Z",
    "completed_at": "2026-09-27T10:01:05Z",
    "updated_at": "2026-09-27T10:01:05Z"
}
```

---

### 3.3 List Jobs

```
GET /api/v1/jobs?status=RUNNING&job_type=send_email&limit=50&offset=0
```

| Query Param | Type | Required | Description |
|---|---|---|---|
| `status` | `string` | NO | Filter by job status |
| `job_type` | `string` | NO | Filter by job type |
| `workflow_id` | `string` | NO | Filter by workflow |
| `limit` | `integer` | NO | Default: 50, Max: 200 |
| `offset` | `integer` | NO | Default: 0 |

**Response — `200 OK`**
```json
{
    "jobs": [ { ...job object... } ],
    "total": 142,
    "limit": 50,
    "offset": 0
}
```

---

### 3.4 Cancel Job

```
POST /api/v1/jobs/:job_id/cancel
```

Cancels a job. Only valid when status is `QUEUED` or `READY`.

**Response — `200 OK`**
```json
{
    "job_id": "job_4p8ycw6m",
    "status": "CANCELLED"
}
```

**Response — `409 Conflict`** (job already running or terminal)
```json
{
    "error": "INVALID_STATE_TRANSITION",
    "message": "cannot cancel a job with status RUNNING"
}
```

---

### 3.5 Get Job Attempts

```
GET /api/v1/jobs/:job_id/attempts
```

**Response — `200 OK`**
```json
{
    "job_id": "job_4p8ycw6m",
    "attempts": [
        {
            "attempt_id": "attempt_001",
            "attempt_num": 1,
            "worker_id": "worker_7j3nbf2a",
            "status": "FAILED",
            "error": "connection refused",
            "started_at": "2026-09-27T10:01:02Z",
            "completed_at": "2026-09-27T10:01:10Z"
        },
        {
            "attempt_id": "attempt_002",
            "attempt_num": 2,
            "worker_id": "worker_7j3nbf2a",
            "status": "SUCCESS",
            "result": { "message_id": "msg_abc" },
            "started_at": "2026-09-27T10:01:15Z",
            "completed_at": "2026-09-27T10:01:18Z"
        }
    ]
}
```

---

### 3.6 Create Job Dependency

```
POST /api/v1/jobs/:job_id/dependencies
```

Declares that `:job_id` depends on one or more upstream jobs.

**Request**
```json
{
    "depends_on": ["job_upstream1", "job_upstream2"]
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `depends_on` | `string[]` | YES | List of upstream job IDs that must complete first |

**Response — `201 Created`**
```json
{
    "job_id": "job_4p8ycw6m",
    "dependencies_added": ["job_upstream1", "job_upstream2"],
    "status": "WAITING_FOR_DEPENDENCY"
}
```

**Response — `409 Conflict`** (cycle detected)
```json
{
    "error": "CYCLIC_DEPENDENCY",
    "message": "adding this dependency would create a cycle in the DAG"
}
```

**Response — `400 Bad Request`** (upstream job not in same project)
```json
{
    "error": "INVALID_REQUEST",
    "message": "job_upstream1 does not exist in this project"
}
```

---

## 4. Workers

> All worker endpoints use **project API key auth**.

### 4.1 List Workers

```
GET /api/v1/workers?status=IDLE
```

| Query Param | Type | Required | Description |
|---|---|---|---|
| `status` | `string` | NO | Filter by `IDLE`, `BUSY`, `OFFLINE` |

**Response — `200 OK`**
```json
{
    "workers": [
        {
            "worker_id": "worker_7j3nbf2a",
            "status": "BUSY",
            "capabilities": ["send_email", "generate_pdf"],
            "concurrency": 10,
            "current_job_count": 3,
            "last_heartbeat_at": "2026-09-27T10:01:50Z",
            "registered_at": "2026-09-27T09:00:00Z"
        }
    ]
}
```

---

### 4.2 Get Worker

```
GET /api/v1/workers/:worker_id
```

**Response — `200 OK`**
```json
{
    "worker_id": "worker_7j3nbf2a",
    "project_id": "proj_abc123",
    "status": "IDLE",
    "capabilities": ["send_email", "generate_pdf"],
    "concurrency": 10,
    "current_job_count": 0,
    "last_heartbeat_at": "2026-09-27T10:01:50Z",
    "registered_at": "2026-09-27T09:00:00Z",
    "updated_at": "2026-09-27T10:01:50Z"
}
```

---

## 5. Workflows

> All workflow endpoints use **project API key auth**.

### 5.1 Create Workflow

```
POST /api/v1/workflows
```

Creates a workflow container. Jobs are associated with it via the `workflow_id` field on enqueue, or by adding them after creation.

**Request**
```json
{
    "name": "Invoice Processing Pipeline"
}
```

**Response — `201 Created`**
```json
{
    "workflow_id": "wf_6m2tcd8v",
    "name": "Invoice Processing Pipeline",
    "status": "RUNNING",
    "created_at": "2026-09-27T10:00:00Z"
}
```

---

### 5.2 Get Workflow

```
GET /api/v1/workflows/:workflow_id
```

**Response — `200 OK`**
```json
{
    "workflow_id": "wf_6m2tcd8v",
    "name": "Invoice Processing Pipeline",
    "status": "RUNNING",
    "created_at": "2026-09-27T10:00:00Z",
    "updated_at": "2026-09-27T10:01:00Z",
    "completed_at": null,
    "jobs": [
        {
            "job_id": "job_aaa",
            "job_type": "validate_invoice",
            "status": "COMPLETED",
            "depends_on": []
        },
        {
            "job_id": "job_bbb",
            "job_type": "process_payment",
            "status": "RUNNING",
            "depends_on": ["job_aaa"]
        },
        {
            "job_id": "job_ccc",
            "job_type": "send_confirmation",
            "status": "WAITING_FOR_DEPENDENCY",
            "depends_on": ["job_bbb"]
        }
    ]
}
```

---

### 5.3 List Workflows

```
GET /api/v1/workflows?status=RUNNING&limit=50&offset=0
```

**Response — `200 OK`**
```json
{
    "workflows": [
        {
            "workflow_id": "wf_6m2tcd8v",
            "name": "Invoice Processing Pipeline",
            "status": "RUNNING",
            "created_at": "2026-09-27T10:00:00Z"
        }
    ],
    "total": 12,
    "limit": 50,
    "offset": 0
}
```

---

### 5.4 Cancel Workflow

```
POST /api/v1/workflows/:workflow_id/cancel
```

Cancels all `QUEUED` and `READY` jobs in the workflow. `RUNNING` jobs are allowed to complete.

**Response — `200 OK`**
```json
{
    "workflow_id": "wf_6m2tcd8v",
    "status": "CANCELLED",
    "jobs_cancelled": 3
}
```

---

## 6. Worker Protocol Endpoints

> These endpoints are called by worker SDKs. They use **project API key auth**.  
> Full protocol details are in `05-WORKER-PROTOCOL.md`.

### Quick Reference

| Operation | Method | Path |
|---|---|---|
| Register Worker | `POST` | `/api/v1/worker/register` |
| Heartbeat | `POST` | `/api/v1/worker/heartbeat` |
| Poll for Job | `GET` | `/api/v1/worker/poll` |
| Acknowledge Job | `POST` | `/api/v1/worker/jobs/:job_id/ack` |
| Submit Success | `POST` | `/api/v1/worker/jobs/:job_id/result` |
| Submit Failure | `POST` | `/api/v1/worker/jobs/:job_id/failure` |
| Graceful Shutdown | `POST` | `/api/v1/worker/shutdown` |

> Full request/response shapes for these endpoints are in `05-WORKER-PROTOCOL.md`.

---

## 7. System Endpoints

### 7.1 Health Check

```
GET /health
```

No authentication required.

**Response — `200 OK`**
```json
{
    "status": "ok",
    "version": "1.0.0",
    "timestamp": "2026-09-27T10:00:00Z"
}
```

**Response — `503 Service Unavailable`** (database unreachable)
```json
{
    "status": "degraded",
    "checks": {
        "database": "unreachable"
    }
}
```

---

### 7.2 Metrics

```
GET /metrics
```

Prometheus-compatible metrics endpoint. No project auth required (protected by deployment-level access control).

---

## 8. Pagination

List endpoints that return collections support cursor-free offset pagination:

```
GET /api/v1/jobs?limit=50&offset=100
```

Response always includes:
```json
{
    "total": 347,
    "limit": 50,
    "offset": 100
}
```

Default `limit` is `50`. Maximum `limit` is `200`.

---

## 9. Rate Limiting

In V1, rate limiting is not enforced at the API layer. It is left to the operator to configure at the infrastructure level (reverse proxy, load balancer). A `429 Too Many Requests` response may be returned by infrastructure in front of Orqen.

---

## Related Documents

| Question | Document |
|---|---|
| How does auth and project isolation work? | `11-PROJECT-ISOLATION.md` |
| What are the full worker protocol request/response shapes? | `05-WORKER-PROTOCOL.md` |
| What does the database schema behind these endpoints look like? | `04-DATA-MODEL.md` |
| How does the SDK call these endpoints? | `10-SDK-SPEC.md` |
