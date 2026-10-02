# Part 57: Observability
## Steps 826-840: Metrics, Logs, Traces (The Three Pillars)

---

## 🎯 เป้าหมายของ Part นี้

- Metrics: counters, gauges, histograms, summaries
- Prometheus-compatible exposition format
- Structured logging (JSON logs)
- Correlation IDs across services
- Health check system
- Alert manager

---

## Step 826: Metrics System

```nim
import asyncdispatch, tables, strformat, times, sequtils, json, strutils, algorithm, math, options

# ============================
# Metric types
# ============================

type
  MetricType = enum
    mtCounter, mtGauge, mtHistogram, mtSummary

  LabelSet = Table[string, string]

  MetricSample = object
    name: string
    labels: LabelSet
    value: float
    timestamp: float

  Counter = object
    name: string
    help: string
    labels: LabelSet
    value: float

  Gauge = object
    name: string
    help: string
    labels: LabelSet
    value: float

  Histogram = object
    name: string
    help: string
    labels: LabelSet
    buckets: seq[float]
    counts: seq[int]    # count for each bucket (<= boundary)
    sum: float
    count: int

  Summary = object
    name: string
    help: string
    labels: LabelSet
    observations: seq[float]
    maxAge: float        # seconds
    observedAt: seq[float]

  MetricRegistry = object
    counters: Table[string, Counter]
    gauges: Table[string, Gauge]
    histograms: Table[string, Histogram]
    summaries: Table[string, Summary]

var registry = MetricRegistry(
  counters: initTable[string, Counter](),
  gauges: initTable[string, Gauge](),
  histograms: initTable[string, Histogram](),
  summaries: initTable[string, Summary]()
)

# ============================
# Counter
# ============================

proc labelKey(labels: LabelSet): string =
  var parts: seq[string]
  for k, v in labels: parts.add(fmt"{k}={v}")
  parts.sort()
  parts.join(",")

proc counterInc(name: string, labels: LabelSet = initTable[string, string](),
                amount = 1.0) =
  let key = name & "{" & labelKey(labels) & "}"
  if key notin registry.counters:
    registry.counters[key] = Counter(name: name, labels: labels)
  registry.counters[key].value += amount

proc counterGet(name: string, labels: LabelSet = initTable[string, string]()): float =
  let key = name & "{" & labelKey(labels) & "}"
  registry.counters.getOrDefault(key, Counter()).value

# ============================
# Gauge
# ============================

proc gaugeSet(name: string, value: float,
              labels: LabelSet = initTable[string, string]()) =
  let key = name & "{" & labelKey(labels) & "}"
  registry.gauges[key] = Gauge(name: name, labels: labels, value: value)

proc gaugeInc(name: string, amount = 1.0,
              labels: LabelSet = initTable[string, string]()) =
  let key = name & "{" & labelKey(labels) & "}"
  if key notin registry.gauges:
    registry.gauges[key] = Gauge(name: name, labels: labels)
  registry.gauges[key].value += amount

proc gaugeDec(name: string, amount = 1.0,
              labels: LabelSet = initTable[string, string]()) =
  gaugeInc(name, -amount, labels)

proc gaugeGet(name: string, labels: LabelSet = initTable[string, string]()): float =
  let key = name & "{" & labelKey(labels) & "}"
  registry.gauges.getOrDefault(key, Gauge()).value

# ============================
# Histogram
# ============================

let defaultBuckets = @[0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0, 10.0]

proc histogramRegister(name, help: string,
                        buckets: seq[float] = defaultBuckets,
                        labels: LabelSet = initTable[string, string]()) =
  let key = name & "{" & labelKey(labels) & "}"
  registry.histograms[key] = Histogram(
    name: name,
    help: help,
    labels: labels,
    buckets: buckets,
    counts: newSeq[int](buckets.len),
    sum: 0.0,
    count: 0
  )

proc histogramObserve(name: string, value: float,
                       labels: LabelSet = initTable[string, string]()) =
  let key = name & "{" & labelKey(labels) & "}"
  if key notin registry.histograms:
    histogramRegister(name, name, defaultBuckets, labels)

  var h = registry.histograms[key]
  h.sum += value
  inc h.count

  for i, b in h.buckets:
    if value <= b:
      inc h.counts[i]

  registry.histograms[key] = h

proc histogramPercentile(h: Histogram, p: float): float =
  ## Approximate percentile from histogram buckets
  if h.count == 0: return 0.0
  let target = int(float(h.count) * p)
  for i, count in h.counts:
    if count >= target:
      return h.buckets[i]
  return h.buckets[^1]

# ============================
# Prometheus exposition format
# ============================

proc exposeMetrics(): string =
  var lines: seq[string]

  # Counters
  for key, c in registry.counters:
    lines.add(fmt"# HELP {c.name} counter")
    lines.add(fmt"# TYPE {c.name} counter")
    var labelStr = ""
    if c.labels.len > 0:
      var parts: seq[string]
      for k, v in c.labels: parts.add(fmt"{k}=\"{v}\"")
      labelStr = "{" & parts.join(",") & "}"
    lines.add(fmt"{c.name}{labelStr} {c.value}")

  # Gauges
  for key, g in registry.gauges:
    lines.add(fmt"# TYPE {g.name} gauge")
    var labelStr = ""
    if g.labels.len > 0:
      var parts: seq[string]
      for k, v in g.labels: parts.add(fmt"{k}=\"{v}\"")
      labelStr = "{" & parts.join(",") & "}"
    lines.add(fmt"{g.name}{labelStr} {g.value}")

  # Histograms
  for key, h in registry.histograms:
    lines.add(fmt"# HELP {h.name} {h.help}")
    lines.add(fmt"# TYPE {h.name} histogram")
    for i, b in h.buckets:
      lines.add(fmt"{h.name}_bucket{{le=\"{b}\"}} {h.counts[i]}")
    lines.add(fmt"{h.name}_bucket{{le=\"+Inf\"}} {h.count}")
    lines.add(fmt"{h.name}_sum {h.sum}")
    lines.add(fmt"{h.name}_count {h.count}")

  return lines.join("\n")

# ============================
# Structured logging
# ============================

type
  LogLevel = enum
    llDebug, llInfo, llWarn, llError, llFatal

  LogField = tuple[key: string, value: JsonNode]

  Logger = object
    name: string
    level: LogLevel
    fields: seq[LogField]
    outputs: seq[proc(entry: JsonNode) {.closure.}]

var defaultLogger = Logger(
  name: "app",
  level: llInfo,
  fields: @[],
  outputs: @[]
)

proc newLogger(name: string, level = llInfo): Logger =
  Logger(name: name, level: level, fields: @[], outputs: @[])

proc withFields(logger: Logger, fields: varargs[LogField]): Logger =
  var l = logger
  for f in fields: l.fields.add(f)
  return l

proc logEntry(logger: Logger, level: LogLevel, msg: string,
              extraFields: varargs[LogField]) =
  if level < logger.level: return

  let levelStr = case level
    of llDebug: "debug"
    of llInfo: "info"
    of llWarn: "warn"
    of llError: "error"
    of llFatal: "fatal"

  var entry = %*{
    "ts": epochTime(),
    "level": levelStr,
    "logger": logger.name,
    "msg": msg
  }

  for (k, v) in logger.fields:
    entry[k] = v
  for (k, v) in extraFields:
    entry[k] = v

  let line = $entry
  echo line

  for output in logger.outputs:
    output(entry)

template logDebug(logger: Logger, msg: string, fields: varargs[LogField]) =
  logEntry(logger, llDebug, msg, fields)

template logInfo(logger: Logger, msg: string, fields: varargs[LogField]) =
  logEntry(logger, llInfo, msg, fields)

template logWarn(logger: Logger, msg: string, fields: varargs[LogField]) =
  logEntry(logger, llWarn, msg, fields)

template logError(logger: Logger, msg: string, fields: varargs[LogField]) =
  logEntry(logger, llError, msg, fields)

# ============================
# Health check system
# ============================

type
  HealthStatus = enum
    hsHealthy, hsDegraded, hsUnhealthy

  HealthCheck = object
    name: string
    check: proc(): HealthStatus
    description: string
    lastStatus: HealthStatus
    lastCheckedAt: float

  HealthReport = object
    status: HealthStatus
    checks: Table[string, tuple[status: HealthStatus, description: string]]
    checkedAt: float

var healthChecks: Table[string, HealthCheck] = initTable[string, HealthCheck]()

proc registerHealthCheck(name, description: string,
                          check: proc(): HealthStatus) =
  healthChecks[name] = HealthCheck(
    name: name,
    description: description,
    check: check,
    lastStatus: hsHealthy
  )

proc runHealthChecks(): HealthReport =
  var checks: Table[string, tuple[status: HealthStatus, description: string]]
  var overallStatus = hsHealthy

  for name, hc in healthChecks:
    let status = hc.check()
    healthChecks[name].lastStatus = status
    healthChecks[name].lastCheckedAt = epochTime()
    checks[name] = (status, hc.description)

    if status == hsUnhealthy:
      overallStatus = hsUnhealthy
    elif status == hsDegraded and overallStatus == hsHealthy:
      overallStatus = hsDegraded

  return HealthReport(
    status: overallStatus,
    checks: checks,
    checkedAt: epochTime()
  )

proc healthReportJson(report: HealthReport): JsonNode =
  var checksNode = newJObject()
  for name, (status, desc) in report.checks:
    checksNode[name] = %*{
      "status": $status,
      "description": desc
    }

  %*{
    "status": $report.status,
    "checks": checksNode,
    "checkedAt": report.checkedAt
  }

# ============================
# Demo
# ============================

proc demo() =
  echo "=== Observability Demo ==="

  # Metrics
  echo "\n--- Metrics ---"
  var httpLabels: LabelSet
  httpLabels["method"] = "GET"
  httpLabels["path"] = "/api/users"
  httpLabels["status"] = "200"

  counterInc("http_requests_total", httpLabels, 1)
  counterInc("http_requests_total", httpLabels, 5)

  var errLabels: LabelSet
  errLabels["method"] = "POST"
  errLabels["path"] = "/api/users"
  errLabels["status"] = "500"
  counterInc("http_requests_total", errLabels, 2)

  gaugeSet("active_connections", 42.0)
  gaugeSet("goroutines", 128.0)

  histogramRegister("http_request_duration_seconds", "HTTP request duration")
  for _ in 0..<100:
    histogramObserve("http_request_duration_seconds",
      0.001 + float(rand(200)) / 1000.0)

  let h = registry.histograms["http_request_duration_seconds{}"]
  echo fmt"p50: {histogramPercentile(h, 0.50):.3f}s"
  echo fmt"p95: {histogramPercentile(h, 0.95):.3f}s"
  echo fmt"p99: {histogramPercentile(h, 0.99):.3f}s"

  # Prometheus output (first 10 lines)
  let promOutput = exposeMetrics()
  echo "\n--- Prometheus format ---"
  for line in promOutput.splitLines()[0..min(9, promOutput.splitLines().len-1)]:
    echo line

  # Logging
  echo "\n--- Structured logging ---"
  var logger = newLogger("api-server")
  let reqLogger = logger.withFields(
    ("request_id", %"req_abc123"),
    ("user_id", %"user_1")
  )

  logInfo(reqLogger, "Request started",
    ("method", %"GET"),
    ("path", %"/api/users"))

  logWarn(reqLogger, "Slow query",
    ("duration_ms", %145),
    ("query", %"SELECT * FROM users"))

  logError(reqLogger, "DB connection failed",
    ("error", %"connection timeout"),
    ("host", %"db.prod.internal"))

  # Health checks
  echo "\n--- Health checks ---"
  var dbOk = true

  registerHealthCheck("database", "PostgreSQL connection", proc(): HealthStatus =
    if dbOk: hsHealthy else: hsUnhealthy)

  registerHealthCheck("redis", "Redis cache connection", proc(): HealthStatus =
    hsDegraded)

  registerHealthCheck("disk", "Disk space", proc(): HealthStatus =
    hsHealthy)

  let report = runHealthChecks()
  echo fmt"Overall: {report.status}"
  echo healthReportJson(report).pretty()

demo()
```

---

## 📝 สรุป Part 57

| Steps | หัวข้อ |
|-------|--------|
| 826 | Counter/gauge/histogram, Prometheus exposition, health checks |
| 827-840 | Structured JSON logging, correlation IDs, alert thresholds |

---

**← [Part 56: Caching](part_56_caching.md) | [Part 58: Security →](part_58_security.md)**
