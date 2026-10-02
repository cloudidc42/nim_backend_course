# Part 28: Logging & Monitoring
## Steps 391-405: Logging ระดับ Production และ Monitoring

---

## 🎯 เป้าหมายของ Part นี้

- Structured logging (JSON logs)
- Log levels และ filtering
- Log rotation
- Metrics collection
- Health checks
- Performance tracing
- Alert system

---

## Step 391: Structured Logging

```nim
import times, strformat, json, os, strutils

type
  LogLevel = enum
    DEBUG = 0, INFO = 1, WARN = 2, ERROR = 3, FATAL = 4

  LogEntry = object
    timestamp: string
    level: string
    message: string
    service: string
    traceId: string
    fields: JsonNode

  Logger = object
    service: string
    minLevel: LogLevel
    outputs: seq[proc(entry: LogEntry)]

proc formatTimestamp(): string =
  let dt = now().utc()
  dt.format("yyyy-MM-dd'T'HH:mm:ss'Z'")

proc newLogger(service: string, minLevel: LogLevel = INFO): Logger =
  Logger(service: service, minLevel: minLevel, outputs: @[])

proc addOutput(logger: var Logger, output: proc(entry: LogEntry)) =
  logger.outputs.add(output)

proc log(logger: Logger, level: LogLevel, msg: string,
         traceId: string = "", fields: JsonNode = nil) =
  if level < logger.minLevel:
    return
  
  let entry = LogEntry(
    timestamp: formatTimestamp(),
    level: $level,
    message: msg,
    service: logger.service,
    traceId: traceId,
    fields: if fields.isNil: newJObject() else: fields
  )
  
  for output in logger.outputs:
    output(entry)

# Console output (human-readable)
proc consoleOutput(entry: LogEntry) =
  let color = case entry.level
    of "DEBUG": "\e[37m"   # gray
    of "INFO":  "\e[32m"   # green
    of "WARN":  "\e[33m"   # yellow
    of "ERROR": "\e[31m"   # red
    of "FATAL": "\e[35m"   # magenta
    else:       "\e[0m"
  
  var line = fmt"{color}[{entry.level[0..4]}]\e[0m {entry.timestamp} {entry.message}"
  
  if entry.traceId.len > 0:
    line &= fmt" trace={entry.traceId}"
  
  if entry.fields.len > 0:
    for key, val in entry.fields:
      line &= fmt" {key}={val}"
  
  echo line

# JSON output (for log aggregators like ELK, Datadog)
proc jsonOutput(entry: LogEntry) =
  let obj = %*{
    "@timestamp": entry.timestamp,
    "level": entry.level,
    "message": entry.message,
    "service": entry.service,
  }
  if entry.traceId.len > 0:
    obj["traceId"] = %entry.traceId
  if entry.fields.len > 0:
    for key, val in entry.fields:
      obj[key] = val
  echo $obj

# File output
proc fileOutput(filename: string): proc(entry: LogEntry) =
  return proc(entry: LogEntry) =
    let line = $(%*{
      "ts": entry.timestamp,
      "level": entry.level,
      "msg": entry.message,
      "svc": entry.service
    })
    var f = open(filename, fmAppend)
    f.writeLine(line)
    f.close()

# Convenience methods
template debug(logger: Logger, msg: string, fields: JsonNode = nil) =
  logger.log(DEBUG, msg, fields = fields)

template info(logger: Logger, msg: string, fields: JsonNode = nil) =
  logger.log(INFO, msg, fields = fields)

template warn(logger: Logger, msg: string, fields: JsonNode = nil) =
  logger.log(WARN, msg, fields = fields)

template error(logger: Logger, msg: string, fields: JsonNode = nil) =
  logger.log(ERROR, msg, fields = fields)

# Demo
var appLogger = newLogger("user-service", DEBUG)
appLogger.addOutput(consoleOutput)

echo "=== Structured Logging Demo ==="
appLogger.info("Server started", %*{"port": 8080})
appLogger.debug("Config loaded", %*{"env": "development"})
appLogger.warn("Slow query detected", %*{"duration_ms": 450, "query": "SELECT * FROM users"})
appLogger.error("DB connection failed", %*{"host": "localhost", "port": 5432, "error": "timeout"})

# With trace ID
appLogger.log(INFO, "HTTP request", "trace-abc123", %*{
  "method": "POST",
  "path": "/api/users",
  "ip": "192.168.1.1",
  "statusCode": 201,
  "duration_ms": 45
})
```

---

## Step 392: Metrics Collection

```nim
import times, tables, strformat, math, sequtils

type
  MetricType = enum
    Counter, Gauge, Histogram, Summary

  Metric = object
    name: string
    mtype: MetricType
    help: string
    labels: seq[string]

  MetricValue = object
    value: float
    timestamp: float
    labels: Table[string, string]

  MetricsRegistry = object
    counters: Table[string, float]
    gauges: Table[string, float]
    histograms: Table[string, seq[float]]
    timestamps: Table[string, float]

var metrics = MetricsRegistry(
  counters: initTable[string, float](),
  gauges: initTable[string, float](),
  histograms: initTable[string, seq[float]]()
)

# Counter: monotonically increasing
proc incCounter(name: string, by: float = 1.0, labels: string = "") =
  let key = if labels.len > 0: name & "{" & labels & "}" else: name
  metrics.counters[key] = metrics.counters.getOrDefault(key, 0.0) + by

# Gauge: can go up or down
proc setGauge(name: string, value: float, labels: string = "") =
  let key = if labels.len > 0: name & "{" & labels & "}" else: name
  metrics.gauges[key] = value

proc incGauge(name: string, by: float = 1.0) =
  metrics.gauges[name] = metrics.gauges.getOrDefault(name, 0.0) + by

proc decGauge(name: string, by: float = 1.0) =
  metrics.gauges[name] = metrics.gauges.getOrDefault(name, 0.0) - by

# Histogram: track distribution
proc observeHistogram(name: string, value: float) =
  if name notin metrics.histograms:
    metrics.histograms[name] = @[]
  metrics.histograms[name].add(value)

proc percentile(values: seq[float], p: float): float =
  if values.len == 0: return 0.0
  let sorted = values.sorted()
  let idx = int(float(sorted.len - 1) * p / 100.0)
  return sorted[idx]

# Prometheus-style output
proc exportPrometheus(): string =
  var lines: seq[string] = @[]
  
  # Counters
  for name, value in metrics.counters:
    lines.add(fmt"# TYPE {name.split('{')[0]} counter")
    lines.add(fmt"{name} {value:.2f}")
  
  # Gauges
  for name, value in metrics.gauges:
    lines.add(fmt"# TYPE {name} gauge")
    lines.add(fmt"{name} {value:.2f}")
  
  # Histograms
  for name, values in metrics.histograms:
    if values.len == 0: continue
    lines.add(fmt"# TYPE {name} histogram")
    lines.add(fmt"{name}_count {values.len}")
    lines.add(fmt"{name}_sum {values.foldl(a + b, 0.0):.2f}")
    lines.add(fmt"{name}_p50 {values.percentile(50):.2f}")
    lines.add(fmt"{name}_p95 {values.percentile(95):.2f}")
    lines.add(fmt"{name}_p99 {values.percentile(99):.2f}")
  
  return lines.join("\n")

# Demo: Simulate API metrics
for i in 1..100:
  incCounter("http_requests_total", labels = fmt"method=\"GET\",path=\"/users\"")
  let duration = 10.0 + float(i mod 50)
  observeHistogram("http_request_duration_ms", duration)

for i in 1..20:
  incCounter("http_requests_total", labels = fmt"method=\"POST\",path=\"/users\"")
  observeHistogram("http_request_duration_ms", float(50 + i * 3))

# Simulated errors
incCounter("http_errors_total", 5.0, labels = "status=\"500\"")
incCounter("http_errors_total", 12.0, labels = "status=\"404\"")

setGauge("active_connections", 42.0)
setGauge("db_pool_size", 10.0)
setGauge("db_pool_active", 3.0)
setGauge("memory_used_bytes", 52_428_800.0)

echo "=== Prometheus Metrics ==="
echo exportPrometheus()
```

---

## Step 393: Request Tracing

```nim
import asyncdispatch, times, strformat, tables, json, options

# Distributed tracing concepts (simplified OpenTelemetry-style)

type
  SpanStatus = enum
    Unset, Ok, Error

  Span = object
    traceId: string
    spanId: string
    parentSpanId: string
    name: string
    startTime: float
    endTime: float
    status: SpanStatus
    tags: Table[string, string]
    events: seq[(float, string)]

  Tracer = object
    spans: seq[Span]
    service: string

var tracer = Tracer(service: "order-service")

proc generateId(prefix: string = ""): string =
  let ts = toUnix(getTime()).int
  let rnd = hash(epochTime())
  fmt"{prefix}{ts:x}{rnd and 0xFFFF:x}"

proc startSpan(tracer: var Tracer, name: string, parentSpanId: string = ""): Span =
  let traceId = if parentSpanId.len > 0:
    # Find parent's trace ID
    var tid = generateId("t")
    for s in tracer.spans:
      if s.spanId == parentSpanId:
        tid = s.traceId
        break
    tid
  else:
    generateId("t")
  
  Span(
    traceId: traceId,
    spanId: generateId("s"),
    parentSpanId: parentSpanId,
    name: name,
    startTime: epochTime(),
    status: Unset,
    tags: initTable[string, string](),
    events: @[]
  )

proc finish(span: var Span, status: SpanStatus = Ok) =
  span.endTime = epochTime()
  span.status = status

proc addTag(span: var Span, key, value: string) =
  span.tags[key] = value

proc addEvent(span: var Span, message: string) =
  span.events.add((epochTime(), message))

proc record(tracer: var Tracer, span: Span) =
  tracer.spans.add(span)

proc duration(span: Span): float =
  (span.endTime - span.startTime) * 1000.0  # milliseconds

proc printTrace(tracer: Tracer, traceId: string) =
  echo fmt"\nTrace: {traceId}"
  let traceSpans = tracer.spans.filterIt(it.traceId == traceId)
  
  for span in traceSpans:
    let indent = if span.parentSpanId.len > 0: "  └─ " else: ""
    echo fmt"{indent}[{span.name}] {span.duration:.1f}ms ({span.status})"
    for key, val in span.tags:
      echo fmt"     {key}: {val}"

# Demo: Trace an HTTP request
proc handleRequest(path: string) {.async.} =
  var rootSpan = tracer.startSpan(fmt"HTTP GET {path}")
  rootSpan.addTag("http.method", "GET")
  rootSpan.addTag("http.path", path)
  
  # Span: Auth check
  var authSpan = tracer.startSpan("auth.verify", rootSpan.spanId)
  await sleepAsync(5)
  authSpan.addTag("auth.userId", "42")
  authSpan.finish(Ok)
  tracer.record(authSpan)
  
  # Span: DB query
  var dbSpan = tracer.startSpan("db.query", rootSpan.spanId)
  dbSpan.addTag("db.type", "postgresql")
  dbSpan.addTag("db.statement", "SELECT * FROM orders WHERE user_id = $1")
  await sleepAsync(25)
  dbSpan.addTag("db.rows_returned", "15")
  dbSpan.finish(Ok)
  tracer.record(dbSpan)
  
  # Span: Cache set
  var cacheSpan = tracer.startSpan("cache.set", rootSpan.spanId)
  cacheSpan.addTag("cache.key", fmt"orders:user:42")
  await sleepAsync(2)
  cacheSpan.finish(Ok)
  tracer.record(cacheSpan)
  
  rootSpan.addTag("http.status", "200")
  rootSpan.finish(Ok)
  tracer.record(rootSpan)
  
  tracer.printTrace(rootSpan.traceId)

waitFor handleRequest("/api/users/42/orders")
```

---

## Step 394-405: Complete Monitoring System

```nim
# monitoring.nim - Complete logging, metrics, and health check system

import asyncdispatch, asynchttpserver, json, times, tables,
       strformat, strutils, os, sequtils, math

# ============================
# Structured Logger
# ============================

type
  Level = enum
    DEBUG = 0, INFO = 1, WARN = 2, ERROR = 3

  StructLog = object
    service: string
    level: Level
    writers: seq[proc(line: string)]

var appLog = StructLog(service: "app", level: INFO, writers: @[])

proc addWriter(log: var StructLog, w: proc(line: string)) =
  log.writers.add(w)

proc write(log: StructLog, lvl: Level, msg: string, extra: JsonNode = nil) =
  if lvl < log.level: return
  
  let entry = %*{
    "ts": now().format("yyyy-MM-dd HH:mm:ss"),
    "lvl": ($lvl).toLowerAscii(),
    "svc": log.service,
    "msg": msg
  }
  if not extra.isNil:
    for k, v in extra: entry[k] = v
  
  let line = $entry
  for w in log.writers:
    w(line)

proc info(log: StructLog, msg: string, extra: JsonNode = nil) = log.write(INFO, msg, extra)
proc warn(log: StructLog, msg: string, extra: JsonNode = nil) = log.write(WARN, msg, extra)
proc error(log: StructLog, msg: string, extra: JsonNode = nil) = log.write(ERROR, msg, extra)

# ============================
# Metrics (Prometheus-like)
# ============================

type
  SimpleMetrics = object
    counters: Table[string, float]
    gauges: Table[string, float]
    histBuckets: Table[string, seq[float]]

var sysMetrics = SimpleMetrics(
  counters: initTable[string, float](),
  gauges: initTable[string, float](),
  histBuckets: initTable[string, seq[float]]()
)

template counter(name: string, val: float = 1.0) =
  sysMetrics.counters[name] = sysMetrics.counters.getOrDefault(name, 0.0) + val

template gauge(name: string, val: float) =
  sysMetrics.gauges[name] = val

template histogram(name: string, val: float) =
  if name notin sysMetrics.histBuckets:
    sysMetrics.histBuckets[name] = @[]
  sysMetrics.histBuckets[name].add(val)

# ============================
# Health Check
# ============================

type
  CheckStatus = enum
    Healthy, Degraded, Unhealthy

  HealthCheck = object
    name: string
    status: CheckStatus
    message: string
    duration_ms: float
    checkedAt: string

  HealthReport = object
    status: CheckStatus
    version: string
    uptime: float
    checks: seq[HealthCheck]

let startTime = epochTime()

proc checkDatabase(): HealthCheck =
  let t0 = epochTime()
  # Simulate DB check
  let ok = true
  return HealthCheck(
    name: "database",
    status: if ok: Healthy else: Unhealthy,
    message: if ok: "Connected" else: "Connection failed",
    duration_ms: (epochTime() - t0) * 1000,
    checkedAt: $now()
  )

proc checkCache(): HealthCheck =
  let t0 = epochTime()
  return HealthCheck(
    name: "cache",
    status: Healthy,
    message: "Redis connected",
    duration_ms: (epochTime() - t0) * 1000,
    checkedAt: $now()
  )

proc checkDisk(): HealthCheck =
  let t0 = epochTime()
  # Check available disk space
  let available = 10 * 1024 * 1024 * 1024  # 10GB simulated
  let status = if available > 1024 * 1024 * 1024: Healthy else: Degraded
  return HealthCheck(
    name: "disk",
    status: status,
    message: fmt"Available: {available div (1024 * 1024 * 1024)}GB",
    duration_ms: (epochTime() - t0) * 1000,
    checkedAt: $now()
  )

proc getHealthReport(): HealthReport =
  let checks = @[checkDatabase(), checkCache(), checkDisk()]
  
  var overallStatus = Healthy
  for check in checks:
    if check.status == Unhealthy:
      overallStatus = Unhealthy
      break
    if check.status == Degraded and overallStatus != Unhealthy:
      overallStatus = Degraded
  
  return HealthReport(
    status: overallStatus,
    version: "1.0.0",
    uptime: epochTime() - startTime,
    checks: checks
  )

# ============================
# HTTP Handler
# ============================

proc handleMonitoring(req: Request) {.async.} =
  let path = req.url.path
  
  # GET /health - Full health check
  if path == "/health":
    let report = getHealthReport()
    
    let checksJson = newJArray()
    for check in report.checks:
      checksJson.add(%*{
        "name": check.name,
        "status": ($check.status).toLowerAscii(),
        "message": check.message,
        "duration_ms": check.duration_ms
      })
    
    let resp = %*{
      "status": ($report.status).toLowerAscii(),
      "version": report.version,
      "uptime_seconds": report.uptime,
      "checks": checksJson
    }
    
    let httpCode = if report.status == Unhealthy: Http503
                   elif report.status == Degraded: Http207
                   else: Http200
    
    await req.respond(httpCode, $resp,
      newHttpHeaders([("Content-Type", "application/json")]))
    return
  
  # GET /ready - Readiness probe (is app ready to serve traffic?)
  if path == "/ready":
    # Just check critical deps
    let dbCheck = checkDatabase()
    if dbCheck.status == Unhealthy:
      await req.respond(Http503, """{"ready":false,"reason":"database unavailable"}""",
        newHttpHeaders([("Content-Type", "application/json")]))
    else:
      await req.respond(Http200, """{"ready":true}""",
        newHttpHeaders([("Content-Type", "application/json")]))
    return
  
  # GET /live - Liveness probe (is app alive?)
  if path == "/live":
    await req.respond(Http200, """{"alive":true}""",
      newHttpHeaders([("Content-Type", "application/json")]))
    return
  
  # GET /metrics - Prometheus format
  if path == "/metrics":
    var lines: seq[string] = @[]
    
    for name, val in sysMetrics.counters:
      lines.add(fmt"# TYPE {name} counter")
      lines.add(fmt"{name} {val}")
    
    for name, val in sysMetrics.gauges:
      lines.add(fmt"# TYPE {name} gauge")
      lines.add(fmt"{name} {val}")
    
    for name, vals in sysMetrics.histBuckets:
      if vals.len == 0: continue
      let sorted = vals.sorted()
      lines.add(fmt"# TYPE {name} histogram")
      lines.add(fmt"{name}_count {vals.len}")
      lines.add(fmt"{name}_sum {vals.foldl(a + b, 0.0):.3f}")
      lines.add(fmt"{name}_p50 {sorted[sorted.len div 2]:.3f}")
      lines.add(fmt"{name}_p95 {sorted[int(float(sorted.len) * 0.95)]:.3f}")
    
    await req.respond(Http200, lines.join("\n"),
      newHttpHeaders([("Content-Type", "text/plain")]))
    return
  
  await req.respond(Http404, """{"error":"not found"}""",
    newHttpHeaders([("Content-Type", "application/json")]))

# ============================
# Demo
# ============================

proc main() {.async.} =
  # Setup logger
  appLog.addWriter(proc(line: string) = echo line)
  
  appLog.info("Service starting", %*{"version": "1.0.0"})
  
  # Simulate some metrics
  for i in 1..50:
    counter("http_requests_total")
    histogram("http_duration_ms", float(10 + i mod 30))
  
  counter("http_errors_total", 3.0)
  gauge("active_connections", 15.0)
  gauge("db_connections", 5.0)
  
  appLog.info("Metrics initialized", %*{"counters": sysMetrics.counters.len})
  
  # Health report
  let health = getHealthReport()
  appLog.info("Health check", %*{
    "status": ($health.status).toLowerAscii(),
    "uptime": health.uptime
  })
  
  echo "\n=== Health Report ==="
  echo fmt"Status: {health.status}"
  for check in health.checks:
    echo fmt"  [{check.name}] {check.status} - {check.message} ({check.duration_ms:.1f}ms)"
  
  echo "\n=== Sample Metrics ==="
  echo fmt"http_requests_total: {sysMetrics.counters.getOrDefault(\"http_requests_total\", 0)}"
  echo fmt"http_errors_total: {sysMetrics.counters.getOrDefault(\"http_errors_total\", 0)}"
  echo fmt"active_connections: {sysMetrics.gauges.getOrDefault(\"active_connections\", 0)}"

waitFor main()
```

---

## 📝 สรุป Part 28

| Steps | หัวข้อ |
|-------|--------|
| 391 | Structured logging (JSON, multi-output) |
| 392 | Prometheus-style metrics |
| 393 | Distributed tracing (spans) |
| 394-405 | Complete monitoring: health checks, readiness, liveness, metrics API |

---

**← [Part 27: Background Jobs](part_27_background_jobs.md) | [Part 29: API Documentation →](part_29_api_documentation.md)**
