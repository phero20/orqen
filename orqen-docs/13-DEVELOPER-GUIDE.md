# 13 — Developer Guide

> **Document purpose:** Everything needed to run, build, and contribute to Orqen locally — prerequisites, environment setup, project structure, and development workflow.  
> **Read before:** Starting any local development or onboarding to the codebase.  
> **Reference when:** Setting up a new machine, debugging a local environment issue, or adding a new service/component.

---

## Prerequisites

| Tool | Version | Purpose |
|---|---|---|
| Go | 1.22+ | Backend server |
| PostgreSQL | 15+ | Primary database |
| `migrate` CLI | Latest | Database migrations (`golang-migrate`) |
| Node.js | 18+ | Dashboard frontend (if applicable) |
| Git | Any | Version control |

### Install Go
```
https://go.dev/dl/
```
Verify: `go version` should print `go1.22.x` or higher.

### Install PostgreSQL
Install locally or run via Docker:
```powershell
docker run --name orqen-postgres `
  -e POSTGRES_USER=orqen `
  -e POSTGRES_PASSWORD=orqen `
  -e POSTGRES_DB=orqen `
  -p 5432:5432 `
  -d postgres:15
```

### Install golang-migrate
```powershell
go install -tags 'postgres' github.com/golang-migrate/migrate/v4/cmd/migrate@latest
```

---

## Repository Structure

```
orqen/
│
├── cmd/
│   └── server/
│       └── main.go              ← Server entry point
│
├── internal/
│   ├── api/                     ← HTTP handlers and routing
│   │   ├── middleware/          ← Auth middleware, request context
│   │   ├── handlers/            ← Route handlers grouped by domain
│   │   └── router.go            ← Route registration
│   │
│   ├── project/                 ← Project & API key management
│   ├── job/                     ← Job system (enqueue, state machine)
│   ├── worker/                  ← Worker registration, heartbeat, liveness
│   ├── protocol/                ← Long-poll, job delivery, result handling
│   ├── scheduler/               ← Scheduler + CRS registry
│   ├── dag/                     ← Dependency engine, DAG validation
│   ├── reliability/             ← Heartbeat monitor, timeout monitor, recovery
│   └── dashboard/               ← Dashboard API (if separate from main API)
│
├── db/
│   ├── migrations/              ← SQL migration files (up + down)
│   └── queries/                 ← Raw SQL queries (if using sqlc)
│
├── pkg/
│   └── id/                      ← ID generation (prefixed random IDs)
│
├── orqen-docs/                  ← All specification documents
│
├── .env.example                 ← Environment variable template
├── go.mod
├── go.sum
└── Makefile                     ← Common development commands
```

---

## Environment Variables

Copy `.env.example` to `.env` and fill in values:

```bash
cp .env.example .env
```

### Required Variables

| Variable | Example | Description |
|---|---|---|
| `DATABASE_URL` | `postgres://orqen:orqen@localhost:5432/orqen?sslmode=disable` | PostgreSQL connection string |
| `ORQEN_ADMIN_TOKEN` | `admin_secret_token` | Master token for admin-level API endpoints |
| `ORQEN_PORT` | `4000` | HTTP server port |

### Optional Variables (with defaults)

| Variable | Default | Description |
|---|---|---|
| `ORQEN_HEARTBEAT_INTERVAL_SECONDS` | `15` | Expected worker heartbeat frequency |
| `ORQEN_HEARTBEAT_TIMEOUT_SECONDS` | `45` | Worker dead timeout |
| `ORQEN_MONITOR_INTERVAL_SECONDS` | `15` | Liveness + timeout monitor frequency |
| `ORQEN_DEFAULT_MAX_RETRIES` | `3` | Default job max retries |
| `ORQEN_DEFAULT_BACKOFF_TYPE` | `exponential` | Default backoff type |
| `ORQEN_DEFAULT_BACKOFF_SECONDS` | `5` | Default base backoff in seconds |
| `ORQEN_LOG_LEVEL` | `info` | Log level: `debug`, `info`, `warn`, `error` |
| `ORQEN_ENV` | `development` | Environment: `development`, `production` |
| `ORQEN_CRS_HEAP_COUNT` | `8` | CRS shard count per scheduler instance |

---

## Database Setup

### Run Migrations

```powershell
# Apply all pending migrations
migrate -path db/migrations -database $env:DATABASE_URL up

# Roll back one migration
migrate -path db/migrations -database $env:DATABASE_URL down 1

# Check current migration version
migrate -path db/migrations -database $env:DATABASE_URL version
```

### Migration File Naming Convention

```
db/migrations/
├── 000001_create_projects.up.sql
├── 000001_create_projects.down.sql
├── 000002_create_api_keys.up.sql
├── 000002_create_api_keys.down.sql
├── 000003_create_workers.up.sql
...
```

Each migration has an `up` and `down` file. Migrations are applied in numeric order.

### Reset Database (development only)

```powershell
migrate -path db/migrations -database $env:DATABASE_URL down -all
migrate -path db/migrations -database $env:DATABASE_URL up
```

---

## Running the Server

### Development

```powershell
# Run directly
go run ./cmd/server

# Or use the Makefile
make dev
```

The server starts on `http://localhost:4000` by default.

### Build Binary

```powershell
go build -o orqen ./cmd/server

# Run the binary
./orqen
```

### With Makefile

```powershell
make build    # compile binary
make dev      # run with hot reload (if air is installed)
make test     # run all tests
make migrate  # run pending migrations
make lint     # run linter
```

---

## Running Tests

```powershell
# Run all tests
go test ./...

# Run with race detector (always use this before committing)
go test -race ./...

# Run a specific package
go test ./internal/scheduler/...

# Run with verbose output
go test -v ./...

# Run with coverage
go test -coverprofile=coverage.out ./...
go tool cover -html=coverage.out
```

### Test Database

Tests that require a database use a separate test database. Set:

```
TEST_DATABASE_URL=postgres://orqen:orqen@localhost:5432/orqen_test?sslmode=disable
```

The test suite creates and tears down its own schema. Do not run tests against the development database.

---

## Running a Local Worker (for testing)

You can run a local test worker using the Go SDK to verify job dispatch end-to-end:

```powershell
# From the sdk-go directory or a test script
go run ./examples/worker
```

Or write a minimal worker manually using the protocol from `05-WORKER-PROTOCOL.md`:

```
POST http://localhost:4000/api/v1/worker/register
Authorization: Bearer <your-api-key>
{
  "worker_id": "worker_local_test",
  "capabilities": ["test_job"],
  "concurrency": 1
}
```

Then poll:
```
GET http://localhost:4000/api/v1/worker/poll?worker_id=worker_local_test&timeout=10
Authorization: Bearer <your-api-key>
```

---

## Development Workflow

### Adding a New Endpoint

1. Add the route in `internal/api/router.go`
2. Create the handler in `internal/api/handlers/<domain>.go`
3. Add the DB query in the appropriate package under `internal/<domain>/`
4. Write tests for the handler and the DB query
5. Update `09-API-REFERENCE.md` if the endpoint is new or changed

### Adding a New Database Table

1. Create migration files:
   ```
   db/migrations/000NNN_create_<table>.up.sql
   db/migrations/000NNN_create_<table>.down.sql
   ```
2. Add the table definition following the schema conventions in `04-DATA-MODEL.md`
3. Add `project_id` column with FK if the table is project-scoped
4. Add required indexes
5. Run migrations locally: `make migrate`
6. Update `04-DATA-MODEL.md`
7. Add the isolation checklist entry per `11-PROJECT-ISOLATION.md`

### Adding a New Job Status

1. Update the state machine diagram in `03-ARCHITECTURE.md` and `06-JOB-SCHEDULING.md`
2. Update the status enum in `04-DATA-MODEL.md`
3. Add the new transition logic in `internal/job/`
4. Update `09-API-REFERENCE.md` if the status is exposed in API responses
5. Update `12-DASHBOARD.md` if the dashboard needs to display or filter by it

### Modifying CRS Integration

1. Read `07-CRS-INTEGRATION.md` fully before touching anything in `internal/scheduler/`
2. Do not change the one-instance-per-`(project_id, capability)` design without a team decision
3. Update `07-CRS-INTEGRATION.md` if behavior changes

---

## Code Style

- Follow standard Go conventions (`gofmt`, `golint`)
- All exported functions and types must have doc comments
- No global mutable state outside of the CRS registry and DB connection pool
- Errors are wrapped with context: `fmt.Errorf("scheduler: acquire worker: %w", err)`
- No `panic()` in production paths — return errors
- All handlers must read `project_id` from request context, never from request body (see `11-PROJECT-ISOLATION.md`)
- SQL queries always include `project_id` filter on project-scoped tables

---

## Linting

```powershell
# Install golangci-lint
go install github.com/golangci/golangci-lint/cmd/golangci-lint@latest

# Run linter
golangci-lint run ./...
```

---

## Health Check

Verify the server is running:

```powershell
curl http://localhost:4000/health
```

Expected response:
```json
{
    "status": "ok",
    "version": "1.0.0",
    "timestamp": "2026-09-27T10:00:00Z"
}
```

---

## Common Issues

### `migrate: no migration files found`
Ensure you are running the migrate command from the repo root, not a subdirectory.

### `pq: password authentication failed`
Check `DATABASE_URL` in your `.env`. Ensure the PostgreSQL user and password match your local setup.

### `bind: address already in use`
Port 4000 is taken. Either stop the conflicting process or change `ORQEN_PORT`.

### CRS `ErrNoResourceAvailable` in tests
This means no workers are registered in the CRS instance being tested. Ensure your test setup calls `sched.Add()` or `sched.BatchAdd()` before testing dispatch.

### Jobs stuck in `RUNNING` after test
The heartbeat monitor and timeout monitor are background goroutines. In tests, either mock time or explicitly call the recovery function rather than waiting for the monitor interval.

---

## Related Documents

| Question | Document |
|---|---|
| What is the full system architecture? | `03-ARCHITECTURE.md` |
| What does the database schema look like? | `04-DATA-MODEL.md` |
| What are the isolation rules to follow? | `11-PROJECT-ISOLATION.md` |
| How does CRS integration work? | `07-CRS-INTEGRATION.md` |
| What are all the API endpoints? | `09-API-REFERENCE.md` |
