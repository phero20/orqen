# 12 — Dashboard

> **Document purpose:** Defines the web dashboard — page structure, what each page shows, what actions are available, data requirements, and the DAG visualization.  
> **Read before:** Implementing any dashboard frontend or dashboard API endpoints.  
> **Reference when:** Adding a new dashboard page, changing what data a page shows, or reviewing dashboard behavior.

---

## Overview

The Orqen dashboard is a web UI that provides real-time observability and management for all projects, jobs, workers, and workflows running inside a deployment.

It is the primary interface for:
- Creating and managing projects
- Monitoring job processing in real time
- Viewing worker health and capacity
- Inspecting workflow DAG execution
- Managing API keys

The dashboard is bundled as part of the Orqen server — no separate deployment needed.

---

## Authentication Model

The dashboard operates at two levels:

| Level | Auth | Access |
|---|---|---|
| **Admin level** | Deployment master token | Create/delete projects, view all projects |
| **Project level** | Project API key | View and manage one specific project |

When a user selects a project from the project list, the dashboard uses that project's API key for all subsequent requests. Data shown is scoped entirely to that project.

---

## Navigation Structure

```
Orqen Dashboard
│
├── /projects                    ← Project list (admin level)
│
└── /projects/:project_id        ← Project home
    │
    ├── /overview                ← Summary metrics
    ├── /jobs                    ← Job list and detail
    ├── /workers                 ← Worker list and detail
    ├── /workflows               ← Workflow list and DAG view
    └── /keys                    ← API key management
```

---

## Page Specifications

---

### Page: Project List (`/projects`)

**Purpose:** Entry point. Shows all projects in the deployment.

**Data displayed:**

| Column | Source | Description |
|---|---|---|
| Project Name | `projects.name` | Human-readable name |
| Project ID | `projects.id` | Prefixed ID |
| Created | `projects.created_at` | Creation timestamp |
| Active Workers | Computed | Count of non-OFFLINE workers |
| Jobs (last 24h) | Computed | Count of jobs created in last 24 hours |

**Actions:**
- **Create Project** — opens a modal, accepts a project name, calls `POST /api/v1/projects`, displays the generated API key once (with copy button and warning that it won't be shown again)
- **Open Project** — navigates to `/projects/:project_id/overview`
- **Delete Project** — confirmation dialog required; calls `DELETE /api/v1/projects/:id`

---

### Page: Project Overview (`/projects/:id/overview`)

**Purpose:** High-level health snapshot for the selected project.

**Metrics cards:**

| Metric | Description |
|---|---|
| Active Workers | Count of IDLE + BUSY workers |
| Jobs Running | Count of RUNNING jobs |
| Jobs Queued | Count of QUEUED + READY jobs |
| Jobs Completed (24h) | COMPLETED in last 24 hours |
| Jobs Failed (24h) | FAILED in last 24 hours |
| Success Rate (24h) | COMPLETED / (COMPLETED + FAILED) × 100% |

**Recent Activity Feed:**
- Last 20 job status transitions (job ID, type, old status → new status, timestamp)
- Ordered by most recent first
- Refreshes automatically (polling or SSE)

**Worker Health Summary:**
- List of workers with status indicators (green = IDLE/BUSY, red = OFFLINE)
- Shows last heartbeat age for each worker

---

### Page: Jobs (`/projects/:id/jobs`)

**Purpose:** Browse, filter, and inspect all jobs in the project.

**Job List Table:**

| Column | Source | Sortable | Filterable |
|---|---|---|---|
| Job ID | `jobs.id` | No | No |
| Type | `jobs.job_type` | No | Yes |
| Status | `jobs.status` | No | Yes |
| Priority | `jobs.priority` | Yes | No |
| Attempts | `jobs.attempt_count` | No | No |
| Created | `jobs.created_at` | Yes | No |
| Updated | `jobs.updated_at` | Yes | No |

**Filters:**
- Status (multi-select): `QUEUED`, `READY`, `RUNNING`, `COMPLETED`, `FAILED`, `RETRYING`, `WAITING_FOR_DEPENDENCY`, `CANCELLED`
- Job Type (text input with autocomplete from known types)
- Date range (created_at)

**Pagination:** 50 per page, offset-based.

**Job Detail Panel (on row click):**
Opens a side panel or detail page showing:

```
Job ID:         job_4p8ycw6m
Type:           generate_pdf
Status:         COMPLETED
Priority:       5
Workflow:       wf_6m2tcd8v (if applicable, clickable link)

Payload:
  { "documentId": "doc_xyz", "format": "A4" }

Result:
  { "pdfUrl": "https://..." }

Timing:
  Created:    2026-09-27T10:01:00Z
  Started:    2026-09-27T10:01:02Z
  Completed:  2026-09-27T10:01:05Z
  Duration:   3s

Attempts:
  ┌─────────────┬─────────┬──────────┬────────────┬──────────┐
  │ Attempt     │ Worker  │ Status   │ Duration   │ Error    │
  ├─────────────┼─────────┼──────────┼────────────┼──────────┤
  │ #1          │ wkr_abc │ FAILED   │ 30s        │ timeout  │
  │ #2          │ wkr_abc │ SUCCESS  │ 3s         │ —        │
  └─────────────┴─────────┴──────────┴────────────┴──────────┘
```

**Actions on a job:**
- **Cancel** — available only when status is `QUEUED` or `READY`; calls `POST /jobs/:id/cancel`; requires confirmation

---

### Page: Workers (`/projects/:id/workers`)

**Purpose:** View all workers registered to the project and their real-time health.

**Worker List Table:**

| Column | Source | Description |
|---|---|---|
| Worker ID | `workers.id` | Prefixed ID |
| Status | `workers.status` | Color-coded: green (IDLE), yellow (BUSY), red (OFFLINE) |
| Capabilities | `worker_capabilities` | Comma-separated list of job types |
| Concurrency | `workers.concurrency` | Max parallel jobs |
| Active Jobs | `workers.current_job_count` | Current running jobs — shown as `3 / 10` |
| Last Heartbeat | `workers.last_heartbeat_at` | Shown as relative time (e.g. "12s ago") |
| Registered | `workers.registered_at` | Registration timestamp |

**Status color coding:**
- `IDLE` — green (registered, has free slots, no current jobs)
- `BUSY` — yellow (has jobs running, may still have free slots)
- `OFFLINE` — red (heartbeat timed out)

**Worker Detail Panel (on row click):**

```
Worker ID:      worker_7j3nbf2a
Status:         BUSY
Capabilities:   send_email, generate_pdf
Concurrency:    10
Active Jobs:    3 / 10
Last Heartbeat: 8 seconds ago
Registered:     2026-09-27T09:00:00Z

Current Jobs:
  job_aaa  —  send_email   —  RUNNING  —  started 45s ago
  job_bbb  —  generate_pdf —  RUNNING  —  started 12s ago
  job_ccc  —  send_email   —  RUNNING  —  started 3s ago
```

**Auto-refresh:** Worker list and heartbeat timestamps refresh every 10 seconds.

---

### Page: Workflows (`/projects/:id/workflows`)

**Purpose:** View all workflows and their DAG execution state.

**Workflow List Table:**

| Column | Source | Description |
|---|---|---|
| Workflow ID | `workflows.id` | Prefixed ID |
| Name | `workflows.name` | Optional human name |
| Status | `workflows.status` | `RUNNING`, `COMPLETED`, `FAILED`, `CANCELLED` |
| Jobs | Computed | Total job count in workflow |
| Progress | Computed | Completed / Total (e.g. `3 / 5`) |
| Created | `workflows.created_at` | Creation timestamp |
| Completed | `workflows.completed_at` | Null if still running |

**Workflow Detail — DAG Visualization (on row click):**

The DAG is rendered as a visual directed graph. Each node represents a job. Edges represent dependencies (upstream → downstream).

**Node appearance:**

| Status | Color | Icon |
|---|---|---|
| `WAITING_FOR_DEPENDENCY` | Gray | Clock |
| `READY` | Blue | Arrow |
| `RUNNING` | Yellow (pulsing) | Spinner |
| `COMPLETED` | Green | Checkmark |
| `FAILED` | Red | X |
| `CANCELLED` | Gray (strikethrough) | Dash |

**Node content:**
```
┌─────────────────────┐
│  generate_pdf       │  ← job_type
│  job_4p8ycw6m       │  ← job_id (truncated)
│  ● COMPLETED        │  ← status
│  3s                 │  ← duration (if completed)
└─────────────────────┘
```

**Example DAG rendering:**

```
        ┌───────────────────┐
        │  validate_invoice │
        │  COMPLETED  ✓     │
        └─────────┬─────────┘
                  │
       ┌──────────┴──────────┐
       │                     │
┌──────▼──────┐       ┌──────▼──────┐
│ check_fraud │       │ calc_tax    │
│ COMPLETED ✓ │       │ RUNNING  ⟳ │
└──────┬──────┘       └──────┬──────┘
       │                     │
       └──────────┬──────────┘
                  │
        ┌─────────▼─────────┐
        │  process_payment  │
        │  WAITING    ⏱     │
        └───────────────────┘
```

**Clicking a node** opens the Job Detail Panel for that job (same as from the Jobs page).

**DAG layout:** Top-to-bottom directed layout. Dependencies flow downward. The layout engine should handle both linear chains and complex fan-out/fan-in structures.

---

### Page: API Keys (`/projects/:id/keys`)

**Purpose:** Manage API keys for the project.

**Key List Table:**

| Column | Source | Description |
|---|---|---|
| Name | `api_keys.name` | Optional label |
| Prefix | `api_keys.key_prefix` | First characters of the key (e.g. `orqen_li...`) |
| Status | `api_keys.is_active` | Active / Revoked |
| Last Used | `api_keys.last_used_at` | Relative time |
| Expires | `api_keys.expires_at` | Date or "Never" |
| Created | `api_keys.created_at` | Creation date |

**Actions:**
- **Create Key** — opens a modal:
  - Optional name input
  - Optional expiry date picker
  - On confirm: calls `POST /api/v1/keys`
  - Displays the full API key **once** with a copy button and a prominent warning: "This key will not be shown again. Copy it now."
- **Revoke Key** — confirmation dialog; calls `DELETE /api/v1/keys/:key_id`; key is immediately marked inactive

---

## Real-Time Updates

The dashboard should reflect current state without requiring full page refreshes.

### V1 Approach — Polling

In V1, the dashboard uses client-side polling:
- Overview metrics: refresh every **10 seconds**
- Jobs list: refresh every **15 seconds** (or on user action)
- Workers list: refresh every **10 seconds**
- Active workflow DAG: refresh every **5 seconds** while any job is `RUNNING`
- Completed workflows: no auto-refresh needed

### Future — Server-Sent Events (SSE)

A future version may implement SSE for push-based updates. The dashboard UI should be designed to support swapping the polling mechanism for SSE without restructuring the components.

---

## Empty States

Every list view must handle the empty state gracefully:

| Page | Empty State Message |
|---|---|
| Project List | "No projects yet. Create your first project to get started." |
| Jobs | "No jobs found. Enqueue a job from your application using the SDK." |
| Workers | "No workers connected. Start a worker using the SDK to begin processing jobs." |
| Workflows | "No workflows found. Create a workflow with dependencies using the SDK." |
| API Keys | "No API keys. This project has no active keys — create one to authenticate requests." |

---

## Error States

| Scenario | Behavior |
|---|---|
| API request fails (5xx) | Show an inline error banner: "Failed to load data. Retrying..." |
| Auth fails (401) | Redirect to login / project selection |
| Resource not found (404) | Show "Not found" state in the panel |
| Network offline | Show a persistent "Connection lost" banner; stop auto-refresh |

---

## Data APIs the Dashboard Consumes

All dashboard data comes from the standard Orqen API defined in `09-API-REFERENCE.md`. The dashboard does not have any private API — it uses the same endpoints available to SDK users.

| Dashboard Section | API Calls |
|---|---|
| Project list | `GET /api/v1/projects` |
| Create project | `POST /api/v1/projects` |
| Overview metrics | `GET /api/v1/jobs` (with status filters + aggregation) |
| Jobs list | `GET /api/v1/jobs` |
| Job detail | `GET /api/v1/jobs/:id`, `GET /api/v1/jobs/:id/attempts` |
| Cancel job | `POST /api/v1/jobs/:id/cancel` |
| Workers list | `GET /api/v1/workers` |
| Worker detail | `GET /api/v1/workers/:id` |
| Workflows list | `GET /api/v1/workflows` |
| Workflow DAG | `GET /api/v1/workflows/:id` |
| API keys list | `GET /api/v1/keys` |
| Create key | `POST /api/v1/keys` |
| Revoke key | `DELETE /api/v1/keys/:id` |

---

## Related Documents

| Question | Document |
|---|---|
| What data does the dashboard read from the API? | `09-API-REFERENCE.md` |
| What does the job state machine look like? | `06-JOB-SCHEDULING.md` — Part 1 |
| What does the worker data model look like? | `04-DATA-MODEL.md` — workers table |
| How is dashboard access scoped per project? | `11-PROJECT-ISOLATION.md` — Layer 6 |
