# Part 27: Background Jobs & Task Queues
## Steps 376-390: งาน Background ระดับ Production

---

## 🎯 เป้าหมายของ Part นี้

- In-process task queue
- Job scheduling (cron-like)
- Job retry with exponential backoff
- Priority queues
- Dead letter queue
- Worker pool pattern
- Async job processing

---

## Step 376: Simple Task Queue

```nim
import asyncdispatch, times, strformat, sequtils, options

type
  JobStatus = enum
    Pending, Running, Completed, Failed, Retrying

  JobPriority = enum
    Low = 0, Normal = 1, High = 2, Critical = 3

  Job = object
    id: string
    name: string
    payload: string
    status: JobStatus
    priority: JobPriority
    createdAt: float
    scheduledFor: float
    attempts: int
    maxAttempts: int
    lastError: string

  TaskHandler = proc(job: Job): Future[bool]

  TaskQueue = object
    pending: seq[Job]
    running: seq[Job]
    completed: seq[Job]
    failed: seq[Job]
    handlers: seq[(string, TaskHandler)]

proc newTaskQueue(): TaskQueue =
  TaskQueue(
    pending: @[],
    running: @[],
    completed: @[],
    failed: @[],
    handlers: @[]
  )

proc enqueue(queue: var TaskQueue, name, payload: string,
             priority: JobPriority = Normal,
             delay: float = 0.0,
             maxAttempts: int = 3): string =
  let jobId = fmt"job_{toUnix(getTime())}_{queue.pending.len}"
  let job = Job(
    id: jobId,
    name: name,
    payload: payload,
    status: Pending,
    priority: priority,
    createdAt: epochTime(),
    scheduledFor: epochTime() + delay,
    attempts: 0,
    maxAttempts: maxAttempts
  )
  queue.pending.add(job)
  
  # Sort by priority (highest first) then by scheduledFor
  queue.pending.sort(proc(a, b: Job): int =
    if a.priority.ord != b.priority.ord:
      return b.priority.ord - a.priority.ord  # higher priority first
    return cmp(a.scheduledFor, b.scheduledFor)  # earlier scheduled first
  )
  
  echo fmt"[Queue] Enqueued: {name} (id={jobId}, priority={priority})"
  return jobId

proc registerHandler(queue: var TaskQueue, jobName: string, handler: TaskHandler) =
  queue.handlers.add((jobName, handler))

proc findHandler(queue: TaskQueue, jobName: string): Option[TaskHandler] =
  for (name, handler) in queue.handlers:
    if name == jobName:
      return some(handler)
  return none(TaskHandler)

proc processNext(queue: var TaskQueue) {.async.} =
  let now = epochTime()
  
  # Find next job ready to run
  var nextIdx = -1
  for i, job in queue.pending:
    if job.scheduledFor <= now:
      nextIdx = i
      break
  
  if nextIdx < 0:
    return
  
  var job = queue.pending[nextIdx]
  queue.pending.delete(nextIdx)
  
  job.status = Running
  inc job.attempts
  queue.running.add(job)
  
  echo fmt"[Worker] Processing: {job.name} (attempt {job.attempts}/{job.maxAttempts})"
  
  let handlerOpt = queue.findHandler(job.name)
  
  var runningIdx = queue.running.len - 1
  
  if handlerOpt.isNone:
    job.status = Failed
    job.lastError = fmt"No handler for '{job.name}'"
    queue.running.delete(runningIdx)
    queue.failed.add(job)
    echo fmt"[Worker] Failed: no handler for {job.name}"
    return
  
  let handler = handlerOpt.get()
  
  try:
    let success = await handler(job)
    
    queue.running.delete(runningIdx)
    
    if success:
      job.status = Completed
      queue.completed.add(job)
      echo fmt"[Worker] Completed: {job.name}"
    else:
      if job.attempts < job.maxAttempts:
        # Retry with exponential backoff
        let delay = float(2 ^ job.attempts) * 1.0  # 2, 4, 8 seconds
        job.status = Retrying
        job.scheduledFor = epochTime() + delay
        queue.pending.add(job)
        echo fmt"[Worker] Retrying in {delay:.0f}s: {job.name}"
      else:
        job.status = Failed
        job.lastError = "Max attempts exceeded"
        queue.failed.add(job)
        echo fmt"[Worker] Failed (max attempts): {job.name}"
  
  except Exception as e:
    queue.running.delete(runningIdx)
    job.lastError = e.msg
    
    if job.attempts < job.maxAttempts:
      let delay = float(2 ^ job.attempts) * 1.0
      job.status = Retrying
      job.scheduledFor = epochTime() + delay
      queue.pending.add(job)
      echo fmt"[Worker] Error, retrying in {delay:.0f}s: {e.msg}"
    else:
      job.status = Failed
      queue.failed.add(job)
      echo fmt"[Worker] Fatal error: {e.msg}"

# Demo handlers
var queue = newTaskQueue()

queue.registerHandler("send_email", proc(job: Job): Future[bool] {.async.} =
  echo fmt"  Sending email with payload: {job.payload}"
  await sleepAsync(50)
  return true
)

queue.registerHandler("resize_image", proc(job: Job): Future[bool] {.async.} =
  echo fmt"  Resizing image: {job.payload}"
  # Simulate occasional failure
  if job.attempts < 2:
    return false  # Will retry
  return true
)

queue.registerHandler("generate_report", proc(job: Job): Future[bool] {.async.} =
  echo fmt"  Generating report: {job.payload}"
  await sleepAsync(100)
  return true
)

# Enqueue jobs
discard queue.enqueue("send_email", """{"to":"user@example.com","subject":"Welcome!"}""", High)
discard queue.enqueue("resize_image", """{"file":"photo.jpg","width":800}""", Normal)
discard queue.enqueue("generate_report", """{"type":"monthly","userId":42}""", Low)
discard queue.enqueue("send_email", """{"to":"admin@example.com","subject":"Alert!"}""", Critical)

proc runWorker() {.async.} =
  echo "\n=== Processing Queue ==="
  for i in 1..8:
    await queue.processNext()
    await sleepAsync(10)
  
  echo fmt"\nResults:"
  echo fmt"  Completed: {queue.completed.len}"
  echo fmt"  Failed: {queue.failed.len}"
  echo fmt"  Pending: {queue.pending.len}"

waitFor runWorker()
```

---

## Step 377: Job Scheduling (Cron-like)

```nim
import asyncdispatch, times, strformat, sequtils, strutils

type
  CronExpression = object
    minutes: seq[int]   # empty = every
    hours: seq[int]
    days: seq[int]
    months: seq[int]
    weekdays: seq[int]

  ScheduledJob = object
    id: string
    name: string
    cron: string
    lastRun: float
    nextRun: float
    handler: proc(): Future[void]
    enabled: bool

proc parseCronField(field: string, min, max: int): seq[int] =
  if field == "*":
    return @[]  # every value
  
  var values: seq[int] = @[]
  
  for part in field.split(','):
    if '/' in part:
      let slashParts = part.split('/')
      let start = if slashParts[0] == "*": min else: parseInt(slashParts[0])
      let step = parseInt(slashParts[1])
      var v = start
      while v <= max:
        values.add(v)
        v += step
    elif '-' in part:
      let rangeParts = part.split('-')
      for v in parseInt(rangeParts[0])..parseInt(rangeParts[1]):
        values.add(v)
    else:
      values.add(parseInt(part))
  
  return values

proc parseCron(expr: string): CronExpression =
  let fields = expr.strip().split(' ')
  if fields.len != 5:
    raise newException(ValueError, fmt"Invalid cron: {expr}")
  
  CronExpression(
    minutes:  parseCronField(fields[0], 0, 59),
    hours:    parseCronField(fields[1], 0, 23),
    days:     parseCronField(fields[2], 1, 31),
    months:   parseCronField(fields[3], 1, 12),
    weekdays: parseCronField(fields[4], 0, 6)
  )

proc matches(cron: CronExpression, dt: DateTime): bool =
  if cron.minutes.len > 0 and dt.minute notin cron.minutes:
    return false
  if cron.hours.len > 0 and dt.hour notin cron.hours:
    return false
  if cron.days.len > 0 and dt.monthday notin cron.days:
    return false
  if cron.months.len > 0 and dt.month.ord notin cron.months:
    return false
  if cron.weekdays.len > 0 and dt.weekday.ord notin cron.weekdays:
    return false
  return true

# Scheduler
type
  Scheduler = object
    jobs: seq[ScheduledJob]

var scheduler = Scheduler(jobs: @[])

proc addJob(s: var Scheduler, id, name, cronExpr: string, handler: proc(): Future[void]) =
  let job = ScheduledJob(
    id: id,
    name: name,
    cron: cronExpr,
    lastRun: 0.0,
    nextRun: 0.0,
    handler: handler,
    enabled: true
  )
  s.jobs.add(job)
  echo fmt"[Scheduler] Added: {name} ({cronExpr})"

proc tick(s: var Scheduler) {.async.} =
  let now = now()
  
  for job in s.jobs.mitems:
    if not job.enabled:
      continue
    
    let cronParsed = parseCron(job.cron)
    if cronParsed.matches(now):
      let nowTs = epochTime()
      # Avoid double-running in same minute
      if nowTs - job.lastRun < 60.0:
        continue
      
      echo fmt"[Scheduler] Running: {job.name}"
      job.lastRun = nowTs
      await job.handler()

# Example jobs
scheduler.addJob("cleanup", "Cleanup temp files", "0 3 * * *",  # Daily at 3am
  proc() {.async.} =
    echo "  Running temp file cleanup..."
)

scheduler.addJob("report", "Daily report", "0 8 * * 1-5",  # Weekdays at 8am
  proc() {.async.} =
    echo "  Generating daily report..."
)

scheduler.addJob("health", "Health check ping", "*/5 * * * *",  # Every 5 minutes
  proc() {.async.} =
    echo "  Health check ping..."
)

echo "Scheduler configured with", scheduler.jobs.len, "jobs"
for job in scheduler.jobs:
  echo fmt"  - {job.name}: {job.cron}"
```

---

## Step 378-390: Complete Job Queue System

```nim
# job_queue.nim - Complete background job processing system

import asyncdispatch, asynchttpserver, json, times, strformat,
       tables, sequtils, options, strutils, hashes

# ============================
# Job Types
# ============================

type
  JobType = enum
    SendEmail, ResizeImage, GenerateReport, SendNotification, SyncData

  JobState = enum
    Queued, Running, Done, Error, Dead

  JobRecord = object
    id: string
    jobType: string
    payload: JsonNode
    state: JobState
    priority: int       # 0=low, 5=normal, 10=high
    attempt: int
    maxAttempts: int
    createdAt: float
    startedAt: float
    finishedAt: float
    error: string
    result: JsonNode

  WorkerPool = object
    jobs: seq[JobRecord]
    handlers: Table[string, proc(j: JsonNode): Future[JsonNode]]
    concurrency: int
    running: int
    processed: int
    errors: int

# ============================
# Worker Pool
# ============================

proc newWorkerPool(concurrency: int = 5): WorkerPool =
  WorkerPool(
    jobs: @[],
    handlers: initTable[string, proc(j: JsonNode): Future[JsonNode]](),
    concurrency: concurrency
  )

var pool = newWorkerPool(concurrency = 3)

proc register(pool: var WorkerPool, jobType: string,
              handler: proc(j: JsonNode): Future[JsonNode]) =
  pool.handlers[jobType] = handler

proc enqueue(pool: var WorkerPool, jobType: string, payload: JsonNode,
             priority: int = 5, maxAttempts: int = 3): string =
  let id = fmt"j{toUnix(getTime())}{hash(payload)}"
  pool.jobs.add(JobRecord(
    id: id,
    jobType: jobType,
    payload: payload,
    state: Queued,
    priority: priority,
    attempt: 0,
    maxAttempts: maxAttempts,
    createdAt: epochTime()
  ))
  pool.jobs.sort(proc(a, b: JobRecord): int = b.priority - a.priority)
  echo fmt"[Pool] Enqueued {jobType} (id={id[0..11]}...)"
  return id

proc getNextJob(pool: var WorkerPool): Option[int] =
  for i, job in pool.jobs:
    if job.state == Queued and
       (job.startedAt == 0 or epochTime() > job.startedAt):
      return some(i)
  return none(int)

proc runJob(pool: var WorkerPool, idx: int) {.async.} =
  var job = pool.jobs[idx]
  job.state = Running
  job.startedAt = epochTime()
  inc job.attempt
  pool.jobs[idx] = job
  inc pool.running
  
  echo fmt"[Worker] Running {job.jobType} attempt {job.attempt}"
  
  if job.jobType notin pool.handlers:
    pool.jobs[idx].state = Dead
    pool.jobs[idx].error = "No handler registered"
    dec pool.running
    inc pool.errors
    return
  
  let handler = pool.handlers[job.jobType]
  
  try:
    let result = await handler(job.payload)
    pool.jobs[idx].state = Done
    pool.jobs[idx].result = result
    pool.jobs[idx].finishedAt = epochTime()
    inc pool.processed
    echo fmt"[Worker] Done: {job.jobType}"
  
  except Exception as e:
    if job.attempt < job.maxAttempts:
      # Schedule retry with backoff
      let backoff = float(4 ^ job.attempt)
      pool.jobs[idx].state = Queued
      pool.jobs[idx].startedAt = epochTime() + backoff
      echo fmt"[Worker] Error, retry in {backoff:.0f}s: {e.msg}"
    else:
      pool.jobs[idx].state = Dead
      pool.jobs[idx].error = e.msg
      pool.jobs[idx].finishedAt = epochTime()
      inc pool.errors
      echo fmt"[Worker] Dead (max attempts): {e.msg}"
  
  dec pool.running

proc processBatch(pool: var WorkerPool) {.async.} =
  var futures: seq[Future[void]] = @[]
  
  while pool.running < pool.concurrency:
    let nextOpt = pool.getNextJob()
    if nextOpt.isNone:
      break
    
    let idx = nextOpt.get()
    pool.jobs[idx].state = Running  # Mark immediately to prevent double-pick
    futures.add(pool.runJob(idx))
  
  if futures.len > 0:
    await all(futures)

# ============================
# Register Handlers
# ============================

pool.register("send_email", proc(payload: JsonNode): Future[JsonNode] {.async.} =
  let to = payload["to"].getStr()
  let subject = payload["subject"].getStr()
  echo fmt"  📧 Email to {to}: {subject}"
  await sleepAsync(20)
  return %*{"sent": true, "to": to}
)

pool.register("resize_image", proc(payload: JsonNode): Future[JsonNode] {.async.} =
  let file = payload["file"].getStr()
  let w = payload["width"].getInt()
  let h = payload["height"].getInt()
  echo fmt"  🖼 Resize {file} to {w}x{h}"
  await sleepAsync(30)
  return %*{"resized": true, "url": fmt"https://cdn.example.com/{file}"}
)

pool.register("generate_report", proc(payload: JsonNode): Future[JsonNode] {.async.} =
  let reportType = payload["type"].getStr()
  echo fmt"  📊 Generate {reportType} report"
  await sleepAsync(50)
  return %*{"reportUrl": fmt"https://reports.example.com/{reportType}.pdf"}
)

pool.register("send_notification", proc(payload: JsonNode): Future[JsonNode] {.async.} =
  let userId = payload["userId"].getInt()
  let msg = payload["message"].getStr()
  echo fmt"  🔔 Notify user {userId}: {msg}"
  await sleepAsync(10)
  return %*{"delivered": true}
)

# ============================
# HTTP API for Job Queue
# ============================

proc handleJobApi(req: Request) {.async.} =
  let path = req.url.path
  
  if req.reqMethod == HttpPost and path == "/jobs":
    let body = parseJson(req.body)
    let jobType = body["type"].getStr()
    let payload = body["payload"]
    let priority = body.getOrDefault("priority").getInt(5)
    
    let jobId = pool.enqueue(jobType, payload, priority)
    
    await req.respond(Http201, $(%*{"jobId": jobId, "status": "queued"}),
      newHttpHeaders([("Content-Type", "application/json")]))
    return
  
  if req.reqMethod == HttpGet and path == "/jobs/stats":
    let stats = %*{
      "total": pool.jobs.len,
      "queued": pool.jobs.filterIt(it.state == Queued).len,
      "running": pool.running,
      "completed": pool.processed,
      "errors": pool.errors
    }
    await req.respond(Http200, $stats,
      newHttpHeaders([("Content-Type", "application/json")]))
    return
  
  if req.reqMethod == HttpGet and path.startsWith("/jobs/"):
    let jobId = path[6..^1]
    var found = false
    for job in pool.jobs:
      if job.id == jobId:
        found = true
        let resp = %*{
          "id": job.id,
          "type": job.jobType,
          "state": $job.state,
          "attempt": job.attempt,
          "createdAt": job.createdAt
        }
        if job.state == Done:
          resp["result"] = job.result
        elif job.state == Dead:
          resp["error"] = %job.error
        await req.respond(Http200, $resp,
          newHttpHeaders([("Content-Type", "application/json")]))
        break
    if not found:
      await req.respond(Http404, """{"error":"Job not found"}""",
        newHttpHeaders([("Content-Type", "application/json")]))
    return
  
  await req.respond(Http404, """{"error":"Not found"}""",
    newHttpHeaders([("Content-Type", "application/json")]))

# ============================
# Demo
# ============================

proc demoJobQueue() {.async.} =
  echo "=== Background Job Queue Demo ==="
  
  # Enqueue jobs
  discard pool.enqueue("send_email", %*{
    "to": "user@example.com",
    "subject": "Order confirmed #1001"
  }, priority = 10)
  
  discard pool.enqueue("send_notification", %*{
    "userId": 42,
    "message": "Your order is ready!"
  }, priority = 8)
  
  discard pool.enqueue("resize_image", %*{
    "file": "product_photo.jpg",
    "width": 800,
    "height": 600
  }, priority = 5)
  
  discard pool.enqueue("generate_report", %*{
    "type": "monthly",
    "userId": 1
  }, priority = 2)
  
  discard pool.enqueue("resize_image", %*{
    "file": "thumbnail.jpg",
    "width": 150,
    "height": 150
  }, priority = 5)
  
  echo fmt"\n[Stats] {pool.jobs.len} jobs queued"
  
  # Process multiple batches
  for batch in 1..3:
    echo fmt"\n--- Batch {batch} ---"
    await pool.processBatch()
    await sleepAsync(100)
  
  echo "\n=== Final Stats ==="
  echo fmt"Total jobs:  {pool.jobs.len}"
  echo fmt"Completed:   {pool.processed}"
  echo fmt"Errors:      {pool.errors}"
  echo fmt"Still queued: {pool.jobs.filterIt(it.state == Queued).len}"

waitFor demoJobQueue()
```

---

## 📝 สรุป Part 27

| Steps | หัวข้อ |
|-------|--------|
| 376 | Simple task queue with priority, retry |
| 377 | Cron scheduler with cron expression parsing |
| 378-390 | Complete worker pool + HTTP API for jobs |

---

**← [Part 26: File Uploads](part_26_file_uploads.md) | [Part 28: Logging & Monitoring →](part_28_logging_monitoring.md)**
