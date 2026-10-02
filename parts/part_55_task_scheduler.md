# Part 55: Task Scheduler & Job Queue
## Steps 796-810: Cron Jobs and Background Workers

---

## 🎯 เป้าหมายของ Part นี้

- Cron expression parser & scheduler
- Priority job queue
- Worker pool
- Job retry & dead letter
- Job dependencies (DAG)
- Monitoring & history

---

## Step 796: Cron Parser & Scheduler

```nim
import asyncdispatch, tables, strformat, times, sequtils, json, strutils, algorithm, options, math

# ============================
# Cron expression parser
# ============================

type
  CronField = object
    values: seq[int]
    isWildcard: bool

  CronExpr = object
    minute: CronField    # 0-59
    hour: CronField      # 0-23
    dayOfMonth: CronField  # 1-31
    month: CronField     # 1-12
    dayOfWeek: CronField  # 0-6 (0=Sunday)

proc parseCronField(s: string, min_, max_: int): CronField =
  if s == "*":
    return CronField(values: @[], isWildcard: true)

  var values: seq[int]

  for part in s.split(","):
    if "/" in part:
      let sides = part.split("/")
      let step = parseInt(sides[1])
      let range_ = sides[0]
      let (rangeMin, rangeMax) = if range_ == "*":
        (min_, max_)
      elif "-" in range_:
        let r = range_.split("-")
        (parseInt(r[0]), parseInt(r[1]))
      else:
        (parseInt(range_), parseInt(range_))

      var v = rangeMin
      while v <= rangeMax:
        values.add(v)
        v += step

    elif "-" in part:
      let r = part.split("-")
      for v in parseInt(r[0])..parseInt(r[1]):
        values.add(v)

    else:
      values.add(parseInt(part))

  return CronField(values: values, isWildcard: false)

proc parseCron(expr: string): CronExpr =
  let parts = expr.split(" ")
  if parts.len != 5:
    raise newException(ValueError, fmt"Invalid cron: {expr}")

  return CronExpr(
    minute: parseCronField(parts[0], 0, 59),
    hour: parseCronField(parts[1], 0, 23),
    dayOfMonth: parseCronField(parts[2], 1, 31),
    month: parseCronField(parts[3], 1, 12),
    dayOfWeek: parseCronField(parts[4], 0, 6)
  )

proc matches(field: CronField, value: int): bool =
  if field.isWildcard: return true
  return value in field.values

proc cronMatches(expr: CronExpr, dt: DateTime): bool =
  matches(expr.minute, dt.minute) and
  matches(expr.hour, dt.hour) and
  matches(expr.dayOfMonth, dt.monthday) and
  matches(expr.month, int(dt.month)) and
  matches(expr.dayOfWeek, int(dt.weekday))

proc nextRun(expr: CronExpr, after: DateTime): DateTime =
  ## Find next matching time after given datetime
  var dt = after
  dt = dt + 1.minutes  # start from next minute
  dt = dt - initDuration(seconds = dt.second, nanoseconds = dt.nanosecond)

  for _ in 0..<525960:  # max 1 year of minutes
    if cronMatches(expr, dt):
      return dt
    dt = dt + 1.minutes

  raise newException(ValueError, "No next run found within 1 year")

# ============================
# Scheduled task types
# ============================

type
  TaskStatus = enum
    tsPending, tsRunning, tsCompleted, tsFailed, tsCancelled

  ScheduledTask = object
    id: string
    name: string
    cronExpr: string
    handler: string     # handler name
    params: JsonNode
    enabled: bool
    nextRunAt: float
    lastRunAt: float
    lastStatus: TaskStatus
    runCount: int
    failCount: int
    timeout: int        # seconds

  TaskRun = object
    taskId: string
    runId: string
    startedAt: float
    completedAt: float
    status: TaskStatus
    output: string
    error: string

  Scheduler = object
    tasks: Table[string, ScheduledTask]
    history: seq[TaskRun]
    running: Table[string, TaskRun]

var scheduler = Scheduler(
  tasks: initTable[string, ScheduledTask](),
  history: @[],
  running: initTable[string, TaskRun]()
)

var taskCounter = 0

proc scheduleTask(name, cronExpr, handler: string,
                  params: JsonNode = newJNull(),
                  timeout = 300): string =
  inc taskCounter
  let id = fmt"task_{taskCounter}"
  let cron = parseCron(cronExpr)
  let nextRun = nextRun(cron, now().utc())

  scheduler.tasks[id] = ScheduledTask(
    id: id,
    name: name,
    cronExpr: cronExpr,
    handler: handler,
    params: params,
    enabled: true,
    nextRunAt: nextRun.toTime().toUnixFloat(),
    lastStatus: tsPending,
    timeout: timeout
  )

  echo fmt"[Scheduler] Registered: {name} ({cronExpr}) next={nextRun}"
  return id

proc cancelTask(taskId: string) =
  if taskId in scheduler.tasks:
    scheduler.tasks[taskId].enabled = false
    echo fmt"[Scheduler] Cancelled: {taskId}"

proc getTasksDue(): seq[ScheduledTask] =
  let now_ = epochTime()
  var due: seq[ScheduledTask]
  for _, task in scheduler.tasks:
    if task.enabled and task.nextRunAt <= now_:
      due.add(task)
  return due

proc recordRun(taskId: string, status: TaskStatus, output, error: string) =
  if taskId notin scheduler.tasks: return
  let runId = fmt"{taskId}_run_{scheduler.history.len + 1}"

  let run = TaskRun(
    taskId: taskId,
    runId: runId,
    startedAt: epochTime() - 1.0,
    completedAt: epochTime(),
    status: status,
    output: output,
    error: error
  )

  scheduler.history.add(run)
  scheduler.running.del(taskId)

  var task = scheduler.tasks[taskId]
  task.lastRunAt = run.completedAt
  task.lastStatus = status
  inc task.runCount
  if status == tsFailed: inc task.failCount

  # Schedule next run
  let cron = parseCron(task.cronExpr)
  let nextRun = nextRun(cron, now().utc())
  task.nextRunAt = nextRun.toTime().toUnixFloat()
  scheduler.tasks[taskId] = task

  let icon = if status == tsCompleted: "✓" else: "✗"
  echo fmt"[Scheduler] {icon} Task {task.name}: {status}"

# ============================
# Worker pool
# ============================

type
  JobPriority = enum
    jpLow = 0, jpNormal = 5, jpHigh = 10, jpCritical = 20

  Job = object
    id: string
    name: string
    priority: JobPriority
    payload: JsonNode
    handler: string
    maxRetries: int
    retries: int
    scheduledAt: float
    startedAt: float
    completedAt: float
    status: TaskStatus
    lastError: string
    dependsOn: seq[string]   # job IDs

  WorkerPool = object
    jobs: Table[string, Job]
    queue: seq[string]         # job IDs sorted by priority
    workers: int
    activeWorkers: int
    completed: int
    failed: int

var pool = WorkerPool(
  jobs: initTable[string, Job](),
  queue: @[],
  workers: 4,
  activeWorkers: 0,
  completed: 0,
  failed: 0
)

var jobCounter = 0

proc enqueueJob(name, handler: string, payload: JsonNode,
                priority = jpNormal, maxRetries = 3,
                dependsOn: seq[string] = @[]): string =
  inc jobCounter
  let id = fmt"job_{jobCounter}"

  pool.jobs[id] = Job(
    id: id,
    name: name,
    priority: priority,
    payload: payload,
    handler: handler,
    maxRetries: maxRetries,
    retries: 0,
    scheduledAt: epochTime(),
    status: tsPending,
    dependsOn: dependsOn
  )

  pool.queue.add(id)
  # Sort by priority (higher = first)
  pool.queue.sort(proc(a, b: string): int =
    cmp(int(pool.jobs[b].priority), int(pool.jobs[a].priority)))

  echo fmt"[Pool] Enqueued: {name} (priority={priority})"
  return id

proc canRun(job: Job): bool =
  ## Check if all dependencies are completed
  for depId in job.dependsOn:
    if depId notin pool.jobs: continue
    if pool.jobs[depId].status != tsCompleted: return false
  return true

proc dequeueJob(): Option[Job] =
  for i, jobId in pool.queue:
    if jobId notin pool.jobs: continue
    let job = pool.jobs[jobId]
    if job.status == tsPending and canRun(job):
      pool.queue.delete(i)
      return some(job)
  return none(Job)

proc completeJob(jobId: string, output: string) =
  if jobId notin pool.jobs: return
  pool.jobs[jobId].status = tsCompleted
  pool.jobs[jobId].completedAt = epochTime()
  dec pool.activeWorkers
  inc pool.completed
  echo fmt"[Pool] Completed: {pool.jobs[jobId].name}"

proc failJob(jobId, error: string) =
  if jobId notin pool.jobs: return
  var job = pool.jobs[jobId]
  job.lastError = error
  inc job.retries

  if job.retries <= job.maxRetries:
    job.status = tsPending
    pool.queue.add(jobId)
    echo fmt"[Pool] Retry {job.retries}/{job.maxRetries}: {job.name}"
  else:
    job.status = tsFailed
    inc pool.failed
    echo fmt"[Pool] Failed (max retries): {job.name}"

  dec pool.activeWorkers
  pool.jobs[jobId] = job

proc processJobs(handlers: Table[string, proc(payload: JsonNode): string]): Future[void] {.async.} =
  while pool.queue.len > 0 or pool.activeWorkers > 0:
    while pool.activeWorkers < pool.workers and pool.queue.len > 0:
      let job = dequeueJob()
      if job.isNone: break

      let j = job.get()
      pool.jobs[j.id].status = tsRunning
      pool.jobs[j.id].startedAt = epochTime()
      inc pool.activeWorkers

      echo fmt"[Pool] Processing: {j.name}"

      await sleepAsync(10)  # simulate work

      if j.handler in handlers:
        try:
          let output = handlers[j.handler](j.payload)
          completeJob(j.id, output)
        except CatchableError as e:
          failJob(j.id, e.msg)
      else:
        completeJob(j.id, "")  # no-op handler

    await sleepAsync(50)

# ============================
# Demo
# ============================

proc demo() {.async.} =
  echo "=== Task Scheduler Demo ==="

  # Register cron tasks
  echo "\n--- Cron scheduler ---"
  let cleanupId = scheduleTask("cleanup_old_sessions",
    "0 2 * * *", "cleanup_sessions", timeout = 60)

  let reportId = scheduleTask("daily_report",
    "0 8 * * *", "send_report",
    params = %*{"recipient": "admin@example.com"})

  let metricsId = scheduleTask("collect_metrics",
    "*/5 * * * *", "collect_metrics")

  echo fmt"\nScheduled tasks: {scheduler.tasks.len}"

  # Simulate task runs
  recordRun(cleanupId, tsCompleted, "Cleaned 1234 sessions", "")
  recordRun(metricsId, tsCompleted, "Metrics collected", "")
  recordRun(reportId, tsFailed, "", "SMTP connection failed")

  for _, task in scheduler.tasks:
    echo fmt"  {task.name}: runs={task.runCount} failures={task.failCount}"

  # Job queue with dependencies
  echo "\n--- Job queue ---"
  let fetchId = enqueueJob("fetch_data", "http_fetch",
    %*{"url": "https://api.example.com/data"}, priority = jpHigh)

  let parseId = enqueueJob("parse_data", "parse_csv",
    %*{"format": "csv"}, priority = jpNormal, dependsOn = @[fetchId])

  let importId = enqueueJob("import_data", "db_import",
    %*{"table": "products"}, priority = jpNormal, dependsOn = @[parseId])

  discard enqueueJob("send_notification", "send_email",
    %*{"to": "admin@example.com", "subject": "Import complete"},
    priority = jpLow, dependsOn = @[importId])

  echo fmt"\nQueued jobs: {pool.queue.len}"

  # Process with handlers
  var handlers: Table[string, proc(payload: JsonNode): string]
  handlers["http_fetch"] = proc(p: JsonNode): string = "fetched 1000 rows"
  handlers["parse_csv"] = proc(p: JsonNode): string = "parsed CSV"
  handlers["db_import"] = proc(p: JsonNode): string = "imported to DB"
  handlers["send_email"] = proc(p: JsonNode): string = "email sent"

  await processJobs(handlers)

  echo fmt"\nJob pool stats:"
  echo fmt"  completed: {pool.completed}"
  echo fmt"  failed: {pool.failed}"

waitFor demo()
```

---

## 📝 สรุป Part 55

| Steps | หัวข้อ |
|-------|--------|
| 796 | Cron parser, scheduler, task history |
| 797-810 | Priority job queue, worker pool, dependency DAG, retry |

---

**← [Part 54: WebSocket](part_54_websocket.md) | [Part 56: Caching Strategies →](part_56_caching.md)**
