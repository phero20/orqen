# 10 — SDK Specification

> **Document purpose:** Language-neutral specification for every Orqen SDK. Defines what every SDK must implement, the method signatures (language-agnostic), lifecycle behavior, error handling, and reconnection logic.  
> **Read before:** Implementing any language SDK.  
> **Reference when:** Adding a new SDK language, reviewing SDK behavior, or ensuring SDK consistency across languages.

---

## Design Principles

1. **Consistency across languages** — every SDK exposes the same conceptual interface regardless of language idioms
2. **Protocol compliance** — every SDK is an implementation of `05-WORKER-PROTOCOL.md` — never deviate from the protocol contract
3. **Opaque to business logic** — the SDK handles all transport, retry, reconnect, and heartbeat concerns; the customer writes only their function
4. **At-least-once delivery** — SDKs must communicate to customers that worker functions should be idempotent
5. **Stable worker identity** — a worker ID must persist across restarts so the server can correctly track reconnects

---

## Supported Languages (V1)

| Language | Package Name | Status |
|---|---|---|
| Node.js / TypeScript | `@orqen/sdk` | V1 |
| Python | `orqen` | V1 |
| Go | `github.com/orqen/sdk-go` | V1 |

Any language with an HTTP client can implement the protocol defined in `05-WORKER-PROTOCOL.md`. The spec below is language-neutral — adapt syntax to the target language.

---

## SDK Components

Every SDK must implement these components:

| Component | Responsibility |
|---|---|
| `OrqenClient` | Top-level client — initialized with API key, provides access to all SDK features |
| `Worker` | Worker process — handles registration, polling loop, heartbeat, result submission |
| `JobContext` | Passed to customer handler functions — contains job data and completion methods |
| `JobHandler` | Customer-provided function — receives a `JobContext`, returns result or throws error |

---

## 1. OrqenClient

The top-level entry point. Initialized once with project credentials.

### Constructor

```
OrqenClient(config: ClientConfig)
```

**ClientConfig fields:**

| Field | Type | Required | Default | Description |
|---|---|---|---|---|
| `apiKey` | `string` | YES | — | Project API key (`orqen_live_xxx`) |
| `baseUrl` | `string` | NO | `http://localhost:4000` | Orqen server URL |
| `timeout` | `duration` | NO | `30s` | Default HTTP request timeout |

**Example (pseudocode):**
```
client = new OrqenClient({
    apiKey: "orqen_live_xxx",
    baseUrl: "https://orqen.mycompany.com"
})
```

### Methods

#### `client.worker(config: WorkerConfig) → Worker`
Creates and returns a `Worker` instance. Does not start polling yet.

#### `client.enqueue(jobType: string, payload: object, options?: EnqueueOptions) → EnqueueResult`
Enqueues a job. Calls `POST /api/v1/jobs`.

**EnqueueOptions:**

| Field | Type | Required | Default | Description |
|---|---|---|---|---|
| `priority` | `integer` | NO | `0` | Job priority |
| `maxRetries` | `integer` | NO | `3` | Max retry attempts |
| `timeoutSeconds` | `integer` | NO | `null` | Per-attempt timeout |
| `backoffType` | `string` | NO | `exponential` | `fixed` or `exponential` |
| `backoffSeconds` | `integer` | NO | `5` | Base backoff seconds |
| `scheduledAt` | `datetime` | NO | `null` | Earliest execution time |
| `workflowId` | `string` | NO | `null` | Associate with a workflow |
| `dependsOn` | `string[]` | NO | `[]` | Upstream job IDs |

**EnqueueResult:**

| Field | Type | Description |
|---|---|---|
| `jobId` | `string` | The created job's ID |
| `status` | `string` | Initial status (`QUEUED` or `WAITING_FOR_DEPENDENCY`) |
| `createdAt` | `datetime` | When the job was created |

#### `client.getJob(jobId: string) → Job`
Fetches job state. Calls `GET /api/v1/jobs/:job_id`.

#### `client.cancelJob(jobId: string) → void`
Cancels a job. Calls `POST /api/v1/jobs/:job_id/cancel`.

#### `client.createWorkflow(name?: string) → Workflow`
Creates a workflow. Calls `POST /api/v1/workflows`.

#### `client.getWorkflow(workflowId: string) → Workflow`
Fetches workflow and all its jobs. Calls `GET /api/v1/workflows/:workflow_id`.

---

## 2. Worker

The worker component manages the worker lifecycle — registration, heartbeat, polling loop, and result submission.

### Constructor (via `client.worker()`)

**WorkerConfig fields:**

| Field | Type | Required | Default | Description |
|---|---|---|---|---|
| `workerId` | `string` | NO | auto-generated | Stable worker ID — persisted locally and reused on restart |
| `concurrency` | `integer` | NO | `1` | Max parallel jobs |
| `pollTimeoutSeconds` | `integer` | NO | `30` | Long-poll timeout per request |
| `heartbeatIntervalSeconds` | `integer` | NO | `15` | How often to send heartbeats |

### Worker ID Persistence

The `workerId` must be stable across process restarts. If not provided:
1. Check for a persisted worker ID in a local file (e.g. `.orqen-worker-id`) or environment variable `ORQEN_WORKER_ID`
2. If not found: generate a new ID (`worker_` + random suffix)
3. Persist it to the local file for future restarts

This ensures the server correctly handles reconnects (reuses the existing registration) vs new registrations.

### Methods

#### `worker.register(jobType: string, handler: JobHandler) → void`
Registers a capability and its handler function. Can be called multiple times for different job types.

```
worker.register("send_email", async (ctx) => {
    await sendEmail(ctx.payload.to, ctx.payload.template)
    ctx.complete({ sent: true })
})

worker.register("generate_pdf", async (ctx) => {
    const url = await renderPdf(ctx.payload.documentId)
    ctx.complete({ pdfUrl: url })
})
```

**Rules:**
- `jobType` must be a non-empty string
- `handler` must be a function that accepts a `JobContext`
- The handler must call either `ctx.complete(result)` or `ctx.fail(error)` — not both, and exactly once
- Registering the same `jobType` twice replaces the previous handler

#### `worker.start() → void`
Starts the worker:
1. Calls `POST /worker/register` with all registered capabilities and concurrency
2. Starts the background heartbeat loop
3. Starts the polling loop for each concurrency slot

This call is non-blocking. The polling runs in background threads/goroutines/async loops depending on language.

#### `worker.stop() → void`
Gracefully shuts down the worker:
1. Calls `POST /worker/shutdown`
2. Stops accepting new jobs
3. Waits for all in-flight handlers to complete
4. Stops the heartbeat loop
5. Returns (or resolves, in async languages) when fully stopped

---

## 3. JobContext

Passed to the customer's handler function. Contains all job data and the methods to complete or fail the job.

### Fields (read-only)

| Field | Type | Description |
|---|---|---|
| `jobId` | `string` | The job's ID |
| `attemptId` | `string` | The current attempt's ID |
| `attemptNum` | `integer` | Which attempt number this is (1-indexed) |
| `jobType` | `string` | The job type string |
| `payload` | `object` | The job's input payload |
| `priority` | `integer` | The job's priority |
| `timeoutSeconds` | `integer \| null` | The configured timeout for this attempt |
| `createdAt` | `datetime` | When the job was originally enqueued |

### Methods

#### `ctx.complete(result?: object) → void`
Marks the job as successfully completed. Calls `POST /worker/jobs/:id/result`.
- `result` is optional — may be `null` or an empty object
- Must be called exactly once per handler invocation
- Calling after `ctx.fail()` is an error

#### `ctx.fail(error: string | Error) → void`
Marks the job as failed. Calls `POST /worker/jobs/:id/failure`.
- `error` is a string message or an Error object
- Must be called exactly once per handler invocation
- Calling after `ctx.complete()` is an error

---

## 4. Polling Loop

The SDK must implement the polling loop correctly. This is the most critical part of the SDK.

### Loop Behavior

```
for each concurrency slot (run in parallel):
    loop:
        try:
            response = GET /worker/poll?worker_id=X&timeout=30

            if response.status == 200:  // job received
                job = response.body
                POST /worker/jobs/:id/ack { worker_id, attempt_id }
                run handler(JobContext) in isolation
                // handler calls ctx.complete() or ctx.fail() internally

            elif response.status == 204:  // no job, timeout
                continue loop  // immediately re-poll

        catch NetworkError:
            wait backoff (exponential, starting 1s, max 30s)
            continue loop
```

### Concurrency Model

The SDK must run `concurrency` number of polling loops in parallel. For example, a worker with `concurrency: 5` runs 5 simultaneous polling connections to the server.

**Language-specific implementation:**
- **Node.js:** `Promise.all` with `concurrency` async loops
- **Python:** `asyncio` tasks or thread pool depending on sync/async handler
- **Go:** goroutines

### Handler Isolation

Each handler invocation must be isolated:
- A panic/exception in one handler must not crash other running handlers
- A panic/exception must result in `ctx.fail()` being called automatically
- The polling loop must continue after a handler error

### Handler Timeout Enforcement

If `job.timeoutSeconds` is set, the SDK should enforce a local timeout:
- If the handler does not call `ctx.complete()` or `ctx.fail()` within `timeoutSeconds`, the SDK calls `ctx.fail("handler timeout")` automatically
- This is a client-side safety net — the server also enforces timeouts independently

---

## 5. Heartbeat Loop

The SDK must send heartbeats in a background loop while the worker is running.

```
loop (every heartbeatIntervalSeconds, default 15):
    try:
        POST /worker/heartbeat { worker_id }
    catch NetworkError:
        log warning, continue loop
        // Do NOT stop the worker on heartbeat failure — the polling loop handles reconnect
```

The heartbeat loop must:
- Run independently of the polling loop
- Never block or delay the polling loop
- Not crash the worker on failure — just log and retry next interval

---

## 6. Reconnection Behavior

Network failures are expected. The SDK must handle them gracefully.

### On Poll Network Error
```
catch NetworkError on GET /worker/poll:
    attempt = 1
    loop:
        wait min(1s * 2^attempt, 30s)  // exponential backoff, max 30s
        try:
            POST /worker/register  // re-register in case server restarted
            break
        catch:
            attempt++
    continue polling loop
```

### On Result Submission Network Error
```
catch NetworkError on POST /result or /failure:
    attempt = 1
    loop:
        wait min(1s * 2^attempt, 10s)
        try:
            POST result/failure again (same attempt_id)
            break  // server is idempotent — safe to retry
        catch:
            attempt++
```

Result submission is **safe to retry** — the server handles duplicate submissions idempotently (same `attempt_id` → no-op if already processed).

### On Heartbeat Network Error
Log and continue. Do not attempt reconnect from the heartbeat loop — the polling loop handles reconnection.

---

## 7. Error Handling Requirements

| Situation | SDK Behavior |
|---|---|
| `401 UNAUTHORIZED` | Throw/raise a fatal error — stop the worker, do not retry |
| `404 WORKER_NOT_FOUND` on poll | Re-register and continue |
| `409 WORKER_AT_CAPACITY` on poll | Wait 1s and retry — this should not happen if SDK manages concurrency correctly |
| `409 ATTEMPT_MISMATCH` on result | Log warning, discard — the attempt was superseded |
| `500 INTERNAL_ERROR` | Treat as transient, retry with backoff |
| Handler throws uncaught exception | Catch, call `ctx.fail(error.message)`, continue polling loop |
| Handler calls neither complete nor fail | SDK must detect this (timeout or explicit check) and call `ctx.fail("handler did not complete")` |

---

## 8. Logging Requirements

SDKs must emit structured logs for these events (to stdout, at minimum):

| Event | Level | Message |
|---|---|---|
| Worker registered | `INFO` | `"worker registered: {worker_id}, capabilities: {list}"` |
| Job received | `INFO` | `"job received: {job_id}, type: {job_type}, attempt: {n}"` |
| Job completed | `INFO` | `"job completed: {job_id}"` |
| Job failed | `WARN` | `"job failed: {job_id}, error: {message}"` |
| Worker at capacity | `DEBUG` | `"worker at capacity, waiting"` |
| Reconnecting | `WARN` | `"connection lost, reconnecting..."` |
| Heartbeat failed | `WARN` | `"heartbeat failed, will retry"` |
| Worker shutdown | `INFO` | `"worker shutting down gracefully"` |

Logging should be configurable (level and output). Sensitive data (payload contents, API keys) must never appear in logs.

---

## 9. Complete SDK Usage Example (Pseudocode)

```
// Initialize client
client = new OrqenClient({
    apiKey: "orqen_live_xxx",
    baseUrl: "https://orqen.mycompany.com"
})

// Create worker
worker = client.worker({
    concurrency: 10
})

// Register handlers
worker.register("send_email", async (ctx) => {
    try {
        result = await emailService.send(ctx.payload.to, ctx.payload.template)
        ctx.complete({ messageId: result.id })
    } catch (err) {
        ctx.fail(err.message)
    }
})

worker.register("generate_pdf", async (ctx) => {
    try {
        url = await pdfService.render(ctx.payload.documentId)
        ctx.complete({ pdfUrl: url })
    } catch (err) {
        ctx.fail(err.message)
    }
})

// Start worker (non-blocking)
worker.start()

// Enqueue a job from elsewhere in the application
result = await client.enqueue("send_email", {
    to: "user@example.com",
    template: "welcome"
}, {
    priority: 5,
    maxRetries: 3
})

print(result.jobId)  // job_4p8ycw6m

// Enqueue a workflow with dependencies
jobA = await client.enqueue("validate_invoice", { invoiceId: "inv_001" })
jobB = await client.enqueue("process_payment", { invoiceId: "inv_001" }, {
    dependsOn: [jobA.jobId]
})
jobC = await client.enqueue("send_confirmation", { invoiceId: "inv_001" }, {
    dependsOn: [jobB.jobId]
})

// Graceful shutdown on process exit
onProcessExit(() => {
    worker.stop()
})
```

---

## 10. SDK Compliance Checklist

Before releasing a new language SDK, verify every item:

```
[ ] OrqenClient initializes with apiKey and baseUrl
[ ] worker() method creates a Worker instance
[ ] Worker generates a stable worker_id and persists it across restarts
[ ] worker.register() accepts jobType and handler
[ ] worker.start() calls POST /worker/register
[ ] worker.start() starts background heartbeat loop (every 15s)
[ ] worker.start() starts N polling loops (N = concurrency)
[ ] Polling loop calls GET /worker/poll with worker_id and timeout
[ ] 200 response → extract job, call POST /ack, invoke handler
[ ] 204 response → immediately re-poll
[ ] Handler receives JobContext with all required fields
[ ] ctx.complete() calls POST /worker/jobs/:id/result
[ ] ctx.fail() calls POST /worker/jobs/:id/failure
[ ] Handler exceptions are caught — ctx.fail() is called automatically
[ ] Handler timeout is enforced if timeoutSeconds is set
[ ] Result submission is retried on network error (same attempt_id)
[ ] Poll network errors trigger exponential backoff + re-register
[ ] 401 response causes fatal error — worker stops
[ ] 409 ATTEMPT_MISMATCH is logged and discarded
[ ] worker.stop() calls POST /worker/shutdown
[ ] worker.stop() waits for in-flight handlers to complete
[ ] Heartbeat failure is logged but does not stop the worker
[ ] Payload contents never appear in logs
[ ] API key never appears in logs
[ ] enqueue() calls POST /api/v1/jobs
[ ] getJob() calls GET /api/v1/jobs/:id
[ ] cancelJob() calls POST /api/v1/jobs/:id/cancel
```

---

## Related Documents

| Question | Document |
|---|---|
| What HTTP endpoints does the SDK call? | `09-API-REFERENCE.md` |
| What is the full worker protocol? | `05-WORKER-PROTOCOL.md` |
| How does the server handle the jobs the SDK enqueues? | `06-JOB-SCHEDULING.md` |
| How does the server handle late/duplicate results from the SDK? | `08-RELIABILITY.md` |
