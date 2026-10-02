# Part 68: Background Job Processing
## Steps 991-1000: Job Queues & Workers in Nim

---

## 🎯 เป้าหมายของ Part นี้

- Job queue with priorities
- Worker pool with concurrency control
- Retry with exponential backoff
- Dead letter queue (DLQ)
- Job lifecycle events
- Cron-based recurring jobs
- Job result storage & callbacks

---

## Steps 991-1000: Complete Job Processing System

```nim
import asyncdispatch, tables, strformat, times, sequtils, json, strutils, options, algorithm, math, hashes

# ============================
# Job types & lifecycle
# ============================

type
  JobStatus = enum
    jsPending, jsRunning, jsCompleted, jsFailed, jsRetrying, jsDead, jsCancelled

  JobPriority = enum
    jpLow = 0, jpNormal = 1, jpHigh = 2, jpCritical = 3

  JobError = object
    message: string
    stack: string
    occurredAt: float

  JobResult = object
    output: JsonNode
    metrics: Table[string, float]

  Job = ref object
    id: string
    queue: string
    type_: string
    payload: JsonNode
    priority: JobPriority
    status: JobStatus
    attempts: int
    maxAttempts: int
    errors: seq[JobError]
    result: Option[JobResult]
    scheduledAt: float
    startedAt: float
    completedAt: float
    createdAt: float
    timeout: float         # seconds
    tags: seq[string]
    callbacks: Table[string, string]  # event -> handler_id

  JobHandler = proc(job: Job): Future[JobResult] {.async.}

  WorkerConfig = object
    name: string
    queues: seq[string]
    concurrency: int
    pollInterval: float   # seconds

  Worker = ref object
    config: WorkerConfig
    running: bool
    activeJobs: seq[string]
    processedCount: int
    failedCount: int
    handlers: Table[string, JobHandler]

  JobQueue = ref object
    pending: seq[Job]
    running: Table[string, Job]
    completed: seq[Job]
    failed: seq[Job]
    dead: seq[Job]
    counter: int

# ============================
# Job queue implementation
# ============================

var jobQueue = JobQueue(
  pending: @[],
  running: initTable[string, Job](),
  completed: @[],
  failed: @[],
  dead: @[]
)

proc enqueue(jq: var JobQueue, type_, queue: string,
             payload: JsonNode,
             priority = jpNormal,
             maxAttempts = 3,
             delay = 0.0,
             timeout = 30.0,
             tags: seq[string] = @[]): Job =
  inc jq.counter
  let job = Job(
    id: fmt"job_{jq.counter}",
    queue: queue,
    type_: type_,
    payload: payload,
    priority: priority,
    status: jsPending,
    attempts: 0,
    maxAttempts: maxAttempts,
    errors: @[],
    result: none(JobResult),
    scheduledAt: epochTime() + delay,
    createdAt: epochTime(),
    timeout: timeout,
    tags: tags,
    callbacks: initTable[string, string]()
  )
  jq.pending.add(job)
  echo fmt"[Queue] Enqueued {job.id} ({type_}) priority={priority} queue={queue}"
  return job

proc dequeue(jq: var JobQueue, queues: seq[string]): Option[Job] =
  let now = epochTime()
  # Sort by priority (high first), then by scheduled time
  var available = jq.pending.filterIt(
    it.queue in queues and
    it.scheduledAt <= now and
    it.status == jsPending
  )
  available.sort(proc(a, b: Job): int =
    if ord(b.priority) != ord(a.priority):
      return ord(b.priority) - ord(a.priority)
    return cmp(a.scheduledAt, b.scheduledAt)
  )
  if available.len == 0: return none(Job)

  let job = available[0]
  jq.pending = jq.pending.filterIt(it.id != job.id)
  job.status = jsRunning
  job.startedAt = epochTime()
  jq.running[job.id] = job
  return some(job)

proc complete(jq: var JobQueue, jobId: string, result: JobResult) =
  if jobId notin jq.running: return
  let job = jq.running[jobId]
  job.status = jsCompleted
  job.completedAt = epochTime()
  job.result = some(result)
  jq.running.del(jobId)
  jq.completed.add(job)
  echo fmt"[Queue] Completed {jobId} in {(job.completedAt - job.startedAt) * 1000:.1f}ms"

proc fail(jq: var JobQueue, jobId: string, error: string) =
  if jobId notin jq.running: return
  let job = jq.running[jobId]
  job.errors.add(JobError(message: error, occurredAt: epochTime()))
  jq.running.del(jobId)

  if job.attempts < job.maxAttempts:
    # Retry with exponential backoff + jitter
    let base = 2.0 ^ float(job.attempts)
    let jitter = float(hash(job.id) mod 1000) / 1000.0
    let delay = base + jitter
    job.status = jsRetrying
    job.scheduledAt = epochTime() + delay
    jq.pending.add(job)
    echo fmt"[Queue] Retrying {jobId} in {delay:.1f}s (attempt {job.attempts}/{job.maxAttempts})"
  else:
    job.status = jsDead
    jq.dead.add(job)
    echo fmt"[Queue] Dead-lettered {jobId} after {job.attempts} attempts"

proc cancel(jq: var JobQueue, jobId: string): bool =
  for i, job in jq.pending:
    if job.id == jobId:
      jq.pending[i].status = jsCancelled
      jq.failed.add(jq.pending[i])
      jq.pending.del(i)
      echo fmt"[Queue] Cancelled {jobId}"
      return true
  return false

proc queueStats(jq: JobQueue): JsonNode =
  %*{
    "pending": jq.pending.len,
    "running": jq.running.len,
    "completed": jq.completed.len,
    "failed": jq.failed.len,
    "dead": jq.dead.len,
    "total": jq.counter
  }

# ============================
# Dead letter queue requeue
# ============================

proc requeueFromDLQ(jq: var JobQueue, jobId = "") =
  var toRequeue: seq[Job]
  if jobId.len > 0:
    toRequeue = jq.dead.filterIt(it.id == jobId)
    jq.dead = jq.dead.filterIt(it.id != jobId)
  else:
    toRequeue = jq.dead
    jq.dead = @[]

  for job in toRequeue:
    job.status = jsPending
    job.attempts = 0
    job.errors = @[]
    job.scheduledAt = epochTime()
    jq.pending.add(job)
    echo fmt"[DLQ] Requeued {job.id}"

# ============================
# Job handlers
# ============================

proc emailHandler(job: Job): Future[JobResult] {.async.} =
  let to = job.payload["to"].getStr()
  let subject = job.payload["subject"].getStr()
  await sleepAsync(50)  # simulate SMTP delay
  echo fmt"  [EmailJob] Sent to {to}: {subject}"
  return JobResult(
    output: %*{"sent": true, "messageId": fmt"msg_{job.id}"},
    metrics: {"send_ms": 50.0}.toTable()
  )

proc imageResizeHandler(job: Job): Future[JobResult] {.async.} =
  let url = job.payload["imageUrl"].getStr()
  let sizes = job.payload["sizes"].getElems()
  await sleepAsync(200)  # simulate processing
  echo fmt"  [ImageJob] Resized {url} to {sizes.len} sizes"
  return JobResult(
    output: %*{"resized": sizes.len, "originalUrl": url},
    metrics: {"resize_ms": 200.0, "sizes_count": float(sizes.len)}.toTable()
  )

proc notificationHandler(job: Job): Future[JobResult] {.async.} =
  let userId = job.payload["userId"].getStr()
  let message = job.payload["message"].getStr()
  await sleepAsync(20)
  echo fmt"  [NotifyJob] Notified user {userId}: {message}"
  return JobResult(
    output: %*{"delivered": true},
    metrics: {"deliver_ms": 20.0}.toTable()
  )

proc dataExportHandler(job: Job): Future[JobResult] {.async.} =
  let format_ = job.payload["format"].getStr()
  let rows = job.payload["rowCount"].getInt()
  await sleepAsync(rows div 10)  # simulate proportional work
  echo fmt"  [ExportJob] Exported {rows} rows as {format_}"
  return JobResult(
    output: %*{"exported": rows, "format": format_, "fileUrl": fmt"s3://exports/{job.id}.{format_}"},
    metrics: {"export_ms": float(rows div 10)}.toTable()
  )

var failNextWebhook = true
proc webhookHandler(job: Job): Future[JobResult] {.async.} =
  let url = job.payload["url"].getStr()
  await sleepAsync(30)
  if failNextWebhook and job.attempts < 2:
    failNextWebhook = false
    raise newException(IOError, "Connection refused")
  echo fmt"  [WebhookJob] Posted to {url}"
  return JobResult(
    output: %*{"statusCode": 200},
    metrics: {"http_ms": 30.0}.toTable()
  )

# ============================
# Worker pool
# ============================

proc newWorker(config: WorkerConfig): Worker =
  var w = Worker(
    config: config,
    running: false,
    activeJobs: @[],
    processedCount: 0,
    failedCount: 0,
    handlers: initTable[string, JobHandler]()
  )
  return w

proc registerHandler(worker: Worker, type_: string, handler: JobHandler) =
  worker.handlers[type_] = handler

proc processJob(worker: Worker, job: Job): Future[void] {.async.} =
  if job.type_ notin worker.handlers:
    fail(jobQueue, job.id, fmt"No handler for job type: {job.type_}")
    return

  job.attempts += 1
  echo fmt"[Worker:{worker.config.name}] Processing {job.id} ({job.type_}) attempt={job.attempts}"

  try:
    let result = await worker.handlers[job.type_](job)
    complete(jobQueue, job.id, result)
    inc worker.processedCount
  except CatchableError as e:
    fail(jobQueue, job.id, e.msg)
    inc worker.failedCount

proc runWorkerCycle(worker: Worker): Future[int] {.async.} =
  var processed = 0
  while worker.activeJobs.len < worker.config.concurrency:
    let maybeJob = dequeue(jobQueue, worker.config.queues)
    if maybeJob.isNone: break
    let job = maybeJob.get()
    worker.activeJobs.add(job.id)
    asyncCheck processJob(worker, job)
    worker.activeJobs = worker.activeJobs.filterIt(
      it in jobQueue.running or
      jobQueue.completed.anyIt(it.id == it) or
      jobQueue.dead.anyIt(it.id == it)
    )
    inc processed
  return processed

# ============================
# Recurring jobs (cron-like)
# ============================

type
  RecurringJob = object
    name: string
    type_: string
    queue: string
    payload: JsonNode
    intervalSeconds: float
    lastRunAt: float
    enabled: bool

var recurringJobs: seq[RecurringJob]

proc scheduleRecurring(name, type_, queue: string,
                       payload: JsonNode, intervalSeconds: float) =
  recurringJobs.add(RecurringJob(
    name: name,
    type_: type_,
    queue: queue,
    payload: payload,
    intervalSeconds: intervalSeconds,
    lastRunAt: 0.0,
    enabled: true
  ))
  echo fmt"[Recurring] Scheduled {name} every {intervalSeconds}s"

proc tickRecurring() =
  let now = epochTime()
  for i, rj in recurringJobs:
    if not rj.enabled: continue
    if now - rj.lastRunAt >= rj.intervalSeconds:
      discard enqueue(jobQueue, rj.type_, rj.queue, rj.payload, jpNormal)
      recurringJobs[i].lastRunAt = now

# ============================
# Job lifecycle hooks
# ============================

type
  JobEvent = enum
    jeEnqueued, jeStarted, jeCompleted, jeFailed, jeDead, jeCancelled

  LifecycleHook = proc(event: JobEvent, job: Job) {.closure.}

var lifecycleHooks: seq[tuple[event: JobEvent, hook: LifecycleHook]]

proc onJobEvent(event: JobEvent, hook: LifecycleHook) =
  lifecycleHooks.add((event, hook))

proc fireEvent(event: JobEvent, job: Job) =
  for (ev, hook) in lifecycleHooks:
    if ev == event: hook(event, job)

# ============================
# Demo
# ============================

proc demo() {.async.} =
  echo "=== Background Job Processing Demo ==="

  # Register lifecycle hooks
  onJobEvent(jeCompleted, proc(event: JobEvent, job: Job) {.closure.} =
    echo fmt"  [Lifecycle] Job {job.id} completed: {job.result.get().output}")

  onJobEvent(jeDead, proc(event: JobEvent, job: Job) {.closure.} =
    echo fmt"  [Lifecycle] Job {job.id} dead after {job.attempts} attempts")

  # Create worker
  let workerCfg = WorkerConfig(
    name: "main-worker",
    queues: @["default", "email", "images"],
    concurrency: 3
  )
  var worker = newWorker(workerCfg)
  registerHandler(worker, "send_email", emailHandler)
  registerHandler(worker, "resize_image", imageResizeHandler)
  registerHandler(worker, "notify_user", notificationHandler)
  registerHandler(worker, "export_data", dataExportHandler)
  registerHandler(worker, "send_webhook", webhookHandler)

  # Enqueue various jobs
  echo "\n--- Enqueueing jobs ---"
  let j1 = enqueue(jobQueue, "send_email", "email",
    %*{"to": "alice@example.com", "subject": "Welcome!"}, jpHigh)

  let j2 = enqueue(jobQueue, "resize_image", "images",
    %*{"imageUrl": "https://cdn.example.com/photo.jpg",
       "sizes": [%*{"w": 800}, %*{"w": 400}, %*{"w": 200}]})

  let j3 = enqueue(jobQueue, "notify_user", "default",
    %*{"userId": "user_1", "message": "Your order shipped!"}, jpCritical)

  let j4 = enqueue(jobQueue, "export_data", "default",
    %*{"format": "csv", "rowCount": 5000}, jpLow)

  let j5 = enqueue(jobQueue, "send_webhook", "default",
    %*{"url": "https://hooks.example.com/webhook", "event": "order.created"},
    maxAttempts = 3)

  echo fmt"\nQueue stats: {queueStats(jobQueue)}"

  # Process jobs
  echo "\n--- Processing jobs ---"
  for cycle in 0..<5:
    let count = await runWorkerCycle(worker)
    await sleepAsync(50)
    echo fmt"Cycle {cycle+1}: processed {count} jobs"

  # Check DLQ after retries
  await sleepAsync(200)
  for cycle in 0..<6:
    discard await runWorkerCycle(worker)
    await sleepAsync(50)

  echo fmt"\nFinal stats: {queueStats(jobQueue)}"
  echo fmt"Worker: processed={worker.processedCount} failed={worker.failedCount}"

  # DLQ handling
  if jobQueue.dead.len > 0:
    echo fmt"\n--- Dead Letter Queue ({jobQueue.dead.len} jobs) ---"
    for job in jobQueue.dead:
      echo fmt"  {job.id} ({job.type_}) errors={job.errors.len}"
      for err in job.errors:
        echo fmt"    {err.message}"

    echo "\nRequeuing from DLQ..."
    requeueFromDLQ(jobQueue)
    discard await runWorkerCycle(worker)

  # Recurring jobs
  echo "\n--- Recurring jobs ---"
  scheduleRecurring("cleanup_sessions", "export_data", "default",
    %*{"format": "noop", "rowCount": 10}, 0.05)  # every 50ms for demo

  for _ in 0..<3:
    tickRecurring()
    discard await runWorkerCycle(worker)
    await sleepAsync(60)

  # Cancel a job
  echo "\n--- Cancellation ---"
  let future = enqueue(jobQueue, "send_email", "email",
    %*{"to": "spam@example.com", "subject": "Cancel me"})
  let cancelled = cancel(jobQueue, future.id)
  echo fmt"Cancelled {future.id}: {cancelled}"

  # Final stats
  echo fmt"\n=== Final Queue Stats ==="
  let stats = queueStats(jobQueue)
  for k, v in stats.fields:
    echo fmt"  {k}: {v.getInt()}"

waitFor demo()
```

---

## 🏆 หลักสูตรครบ 1000 Steps!

ยินดีด้วย! คุณได้เรียนรู้ Nim Backend Development ครบทุก Step แล้ว

### สรุปหลักสูตรทั้งหมด

| Parts | หัวข้อ | Steps |
|-------|--------|-------|
| 01-10 | Nim Basics, Types, Control Flow | 1-150 |
| 11-20 | Functions, OOP, Modules, Error Handling | 151-300 |
| 21-30 | Async/Await, HTTP Server, JSON, DB | 301-450 |
| 31-40 | REST API, Auth, Middleware, Validation | 451-600 |
| 41-50 | Advanced Async, Templates, Testing | 601-750 |
| 51-60 | Config, Streaming, GraphQL, WebSocket | 751-885 |
| 61-68 | Microservices, CQRS, gRPC, Jobs | 886-1000 |

---

## 📝 สรุป Part 68

| Steps | หัวข้อ |
|-------|--------|
| 991 | Job types, status lifecycle, priority queue |
| 992-996 | Worker pool, retry with backoff, DLQ |
| 997-1000 | Recurring jobs, lifecycle hooks, cancellation |

---

**← [Part 67: Migrations](part_67_migrations.md) | หลักสูตรจบแล้ว! 🎉**
