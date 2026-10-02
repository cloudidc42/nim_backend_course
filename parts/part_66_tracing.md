# Part 66: Distributed Tracing
## Steps 961-975: OpenTelemetry-style Tracing in Nim

---

## 🎯 เป้าหมายของ Part นี้

- Trace context propagation (W3C TraceContext)
- Span lifecycle (start, end, attributes, events)
- Parent-child span relationships
- Baggage propagation
- Trace sampling strategies
- Export to Jaeger/Zipkin-compatible format

---

## Step 961: Trace Context & Spans

```nim
import asyncdispatch, tables, strformat, times, sequtils, json, strutils, options, algorithm, hashes, math

# ============================
# Trace context (W3C TraceContext)
# ============================

type
  TraceId = distinct string   # 32 hex chars
  SpanId  = distinct string   # 16 hex chars
  TraceFlags = distinct byte

proc newTraceId(): TraceId =
  let t = int(epochTime() * 1e9)
  TraceId(fmt"{t:016x}{t xor 0xDEADBEEFCAFE:016x}")

proc newSpanId(): SpanId =
  let t = int(epochTime() * 1e9)
  SpanId(fmt"{t xor 0xA5A5A5A5A5A5A5A5:016x}")

proc `$`(t: TraceId): string = string(t)
proc `$`(s: SpanId): string = string(s)

# W3C traceparent header: version-traceId-parentId-flags
proc encodeTraceparent(traceId: TraceId, spanId: SpanId, sampled: bool): string =
  let flags = if sampled: "01" else: "00"
  fmt"00-{traceId}-{spanId}-{flags}"

proc parseTraceparent(header: string): Option[tuple[traceId: TraceId, spanId: SpanId, sampled: bool]] =
  let parts = header.split("-")
  if parts.len < 4: return none(tuple[traceId: TraceId, spanId: SpanId, sampled: bool])
  if parts[0] != "00": return none(tuple[traceId: TraceId, spanId: SpanId, sampled: bool])
  some((TraceId(parts[1]), SpanId(parts[2]), parts[3] == "01"))

# ============================
# Span
# ============================

type
  SpanKind = enum
    skInternal, skServer, skClient, skProducer, skConsumer

  SpanStatus = enum
    ssUnset, ssOk, ssError

  SpanEvent = object
    name: string
    timestamp: float
    attributes: Table[string, JsonNode]

  SpanLink = object
    traceId: TraceId
    spanId: SpanId
    attributes: Table[string, JsonNode]

  Span = ref object
    traceId: TraceId
    spanId: SpanId
    parentSpanId: Option[SpanId]
    name: string
    kind: SpanKind
    startTime: float
    endTime: float
    attributes: Table[string, JsonNode]
    events: seq[SpanEvent]
    links: seq[SpanLink]
    status: SpanStatus
    statusMessage: string
    sampled: bool
    children: seq[Span]

  Tracer = ref object
    serviceName: string
    spans: seq[Span]
    sampler: proc(traceId: TraceId): bool
    exporters: seq[proc(spans: seq[Span]) {.closure.}]

proc newTracer(serviceName: string): Tracer =
  Tracer(
    serviceName: serviceName,
    spans: @[],
    sampler: proc(t: TraceId): bool = true,  # sample everything
    exporters: @[]
  )

proc startSpan(tracer: Tracer, name: string,
               kind = skInternal,
               parent: Option[Span] = none(Span)): Span =
  let traceId = if parent.isSome: parent.get().traceId else: newTraceId()
  let span = Span(
    traceId: traceId,
    spanId: newSpanId(),
    parentSpanId: if parent.isSome: some(parent.get().spanId) else: none(SpanId),
    name: name,
    kind: kind,
    startTime: epochTime(),
    attributes: initTable[string, JsonNode](),
    events: @[],
    links: @[],
    status: ssUnset,
    sampled: tracer.sampler(traceId)
  )
  tracer.spans.add(span)
  return span

proc setAttribute(span: Span, key: string, value: JsonNode) =
  span.attributes[key] = value

proc addEvent(span: Span, name: string, attrs: Table[string, JsonNode] = initTable[string, JsonNode]()) =
  span.events.add(SpanEvent(
    name: name,
    timestamp: epochTime(),
    attributes: attrs
  ))

proc setStatus(span: Span, status: SpanStatus, message = "") =
  span.status = status
  span.statusMessage = message

proc endSpan(span: Span) =
  span.endTime = epochTime()

proc durationMs(span: Span): float =
  (span.endTime - span.startTime) * 1000.0

# ============================
# Context propagation
# ============================

proc injectHeaders(span: Span): Table[string, string] =
  result = initTable[string, string]()
  result["traceparent"] = encodeTraceparent(span.traceId, span.spanId, span.sampled)
  result["tracestate"] = ""

proc extractFromHeaders(tracer: Tracer, headers: Table[string, string],
                        spanName: string): Span =
  let traceparent = headers.getOrDefault("traceparent", "")
  if traceparent.len > 0:
    let parsed = parseTraceparent(traceparent)
    if parsed.isSome:
      let (traceId, parentSpanId, sampled) = parsed.get()
      let span = Span(
        traceId: traceId,
        spanId: newSpanId(),
        parentSpanId: some(parentSpanId),
        name: spanName,
        kind: skServer,
        startTime: epochTime(),
        attributes: initTable[string, JsonNode](),
        events: @[],
        links: @[],
        status: ssUnset,
        sampled: sampled
      )
      tracer.spans.add(span)
      return span

  return startSpan(tracer, spanName, skServer)

# ============================
# Sampling strategies
# ============================

proc alwaysSample(_: TraceId): bool = true
proc neverSample(_: TraceId): bool = false

proc ratioSampler(ratio: float): proc(t: TraceId): bool =
  proc sample(t: TraceId): bool =
    let h = hash(string(t)) mod 100
    return float(h) < ratio * 100.0
  return sample

proc parentBasedSampler(defaultSample: bool): proc(t: TraceId): bool =
  proc sample(_: TraceId): bool = defaultSample
  return sample

# ============================
# Jaeger-compatible JSON exporter
# ============================

proc spanToJaeger(span: Span, serviceName: string): JsonNode =
  var tags: seq[JsonNode] = @[]
  for k, v in span.attributes:
    tags.add(%*{"key": k, "type": "string", "value": v})

  var logs: seq[JsonNode] = @[]
  for ev in span.events:
    var fields: seq[JsonNode] = @[%*{"key": "event", "value": ev.name}]
    for k, v in ev.attributes:
      fields.add(%*{"key": k, "value": v})
    logs.add(%*{
      "timestamp": int(ev.timestamp * 1e6),  # microseconds
      "fields": fields
    })

  let refs = if span.parentSpanId.isSome: @[%*{
    "refType": "CHILD_OF",
    "traceID": $span.traceId,
    "spanID": $span.parentSpanId.get()
  }] else: @[]

  %*{
    "traceID": $span.traceId,
    "spanID": $span.spanId,
    "operationName": span.name,
    "references": refs,
    "startTime": int(span.startTime * 1e6),
    "duration": int(durationMs(span) * 1000),
    "tags": tags,
    "logs": logs,
    "processID": "p1",
    "warnings": nil
  }

proc exportToJaeger(spans: seq[Span], serviceName: string): JsonNode =
  var spanNodes: seq[JsonNode]
  for span in spans:
    spanNodes.add(spanToJaeger(span, serviceName))

  %*{
    "data": [{
      "traceID": if spans.len > 0: $spans[0].traceId else: "",
      "spans": spanNodes,
      "processes": {
        "p1": {
          "serviceName": serviceName,
          "tags": []
        }
      }
    }]
  }

# ============================
# Zipkin-compatible JSON exporter
# ============================

proc spanToZipkin(span: Span, serviceName: string): JsonNode =
  let kind = case span.kind
    of skServer: "SERVER"
    of skClient: "CLIENT"
    of skProducer: "PRODUCER"
    of skConsumer: "CONSUMER"
    else: "INTERNAL"

  var tags: Table[string, string]
  for k, v in span.attributes:
    tags[k] = v.getStr()
  if span.status == ssError:
    tags["error"] = span.statusMessage

  var node = %*{
    "traceId": $span.traceId,
    "id": $span.spanId,
    "name": span.name,
    "kind": kind,
    "timestamp": int(span.startTime * 1e6),
    "duration": int(durationMs(span) * 1000),
    "localEndpoint": {"serviceName": serviceName},
    "tags": %tags
  }

  if span.parentSpanId.isSome:
    node["parentId"] = %$span.parentSpanId.get()

  return node

# ============================
# Span tree printer
# ============================

proc printSpanTree(spans: seq[Span], indent = 0) =
  let prefix = "  ".repeat(indent)
  for span in spans:
    let status = case span.status
      of ssOk: "OK"
      of ssError: "ERROR"
      else: ""
    let statusStr = if status.len > 0: fmt" [{status}]" else: ""
    echo fmt"{prefix}├─ {span.name} ({durationMs(span):.2f}ms){statusStr}"
    for ev in span.events:
      echo fmt"{prefix}│  @ {ev.name}"

# ============================
# Demo: tracing a web request chain
# ============================

proc simulateDbQuery(tracer: Tracer, parent: Span): Future[void] {.async.} =
  let span = startSpan(tracer, "db.query", skClient, some(parent))
  span.setAttribute("db.system", %"postgresql")
  span.setAttribute("db.statement", %"SELECT * FROM users WHERE id = $1")
  span.setAttribute("db.table", %"users")
  await sleepAsync(15)
  span.addEvent("query_executed", {"rows_returned": %5}.toTable())
  span.setStatus(ssOk)
  endSpan(span)

proc simulateCacheCheck(tracer: Tracer, parent: Span): Future[bool] {.async.} =
  let span = startSpan(tracer, "cache.get", skClient, some(parent))
  span.setAttribute("cache.key", %"user:123")
  span.setAttribute("cache.backend", %"redis")
  await sleepAsync(2)
  let hit = true
  span.setAttribute("cache.hit", %hit)
  span.setStatus(ssOk)
  endSpan(span)
  return hit

proc simulateApiCall(tracer: Tracer, parent: Span): Future[void] {.async.} =
  let span = startSpan(tracer, "http.client.call", skClient, some(parent))
  span.setAttribute("http.method", %"POST")
  span.setAttribute("http.url", %"http://payment-service/charge")
  await sleepAsync(30)
  span.setAttribute("http.status_code", %200)
  span.setStatus(ssOk)
  endSpan(span)

proc demo() {.async.} =
  echo "=== Distributed Tracing Demo ==="

  let tracer = newTracer("api-gateway")

  # Simulate an HTTP request
  echo "\n--- Request trace ---"
  let rootSpan = startSpan(tracer, "GET /api/users/123", skServer)
  rootSpan.setAttribute("http.method", %"GET")
  rootSpan.setAttribute("http.route", %"/api/users/:id")
  rootSpan.setAttribute("http.flavor", %"1.1")
  rootSpan.setAttribute("net.peer.ip", %"192.168.1.1")

  rootSpan.addEvent("request_received")

  # Check cache
  let cacheHit = await simulateCacheCheck(tracer, rootSpan)
  if not cacheHit:
    await simulateDbQuery(tracer, rootSpan)

  await simulateApiCall(tracer, rootSpan)

  rootSpan.addEvent("response_sent", {"user_count": %1}.toTable())
  rootSpan.setAttribute("http.status_code", %200)
  rootSpan.setStatus(ssOk)
  endSpan(rootSpan)

  echo fmt"Total spans: {tracer.spans.len}"
  for span in tracer.spans:
    let parent = if span.parentSpanId.isSome: "child" else: "root"
    echo fmt"  [{parent}] {span.name}: {durationMs(span):.2f}ms status={span.status}"

  # Context propagation
  echo "\n--- Context propagation ---"
  let outHeaders = injectHeaders(rootSpan)
  echo fmt"traceparent: {outHeaders[\"traceparent\"]}"

  let parsed = parseTraceparent(outHeaders["traceparent"])
  if parsed.isSome:
    let (tid, sid, sampled) = parsed.get()
    echo fmt"  traceId: {tid}"
    echo fmt"  spanId: {sid}"
    echo fmt"  sampled: {sampled}"

  # Sampling
  echo "\n--- Sampling ---"
  let ratioSample = ratioSampler(0.5)
  var sampledCount = 0
  for i in 0..<100:
    let tid = newTraceId()
    if ratioSample(tid): inc sampledCount
  echo fmt"50% sampler: sampled {sampledCount}/100"

  # Export
  echo "\n--- Jaeger export (truncated) ---"
  let jaeger = exportToJaeger(tracer.spans, "api-gateway")
  let spans = jaeger["data"][0]["spans"]
  echo fmt"Exported {spans.len} spans to Jaeger format"
  if spans.len > 0:
    echo fmt"  First span: {spans[0][\"operationName\"].getStr()} {spans[0][\"duration\"].getInt()}μs"

waitFor demo()
```

---

## 📝 สรุป Part 66

| Steps | หัวข้อ |
|-------|--------|
| 961 | TraceId/SpanId, W3C traceparent, span lifecycle |
| 962-970 | Context propagation, parent-child spans, span events |
| 971-975 | Sampling strategies, Jaeger/Zipkin export |

---

**← [Part 65: gRPC](part_65_grpc.md) | [Part 67: Database Migrations →](part_67_migrations.md)**
