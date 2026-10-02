# Part 49: Service Mesh Patterns
## Steps 706-720: Service-to-Service Communication

---

## 🎯 เป้าหมายของ Part นี้

- Service registry & discovery
- Health check aggregation
- Retry with jitter
- Bulkhead pattern
- Timeout propagation
- Distributed tracing

---

## Step 706: Service Registry

```nim
import asyncdispatch, tables, strformat, times, sequtils, json, strutils, options, algorithm

# ============================
# Service registry
# ============================

type
  ServiceInstance = object
    id: string
    serviceName: string
    host: string
    port: int
    version: string
    tags: seq[string]
    metadata: Table[string, string]
    healthy: bool
    registeredAt: float
    lastHeartbeatAt: float
    weight: int

  ServiceRegistry = object
    instances: Table[string, ServiceInstance]    # instanceId -> instance
    services: Table[string, seq[string]]         # serviceName -> [instanceIds]
    heartbeatTimeout: float                       # seconds

var registry = ServiceRegistry(
  instances: initTable[string, ServiceInstance](),
  services: initTable[string, seq[string]](),
  heartbeatTimeout: 30.0
)

proc register(inst: ServiceInstance) =
  var i = inst
  i.registeredAt = epochTime()
  i.lastHeartbeatAt = epochTime()
  i.healthy = true
  
  registry.instances[i.id] = i
  
  if i.serviceName notin registry.services:
    registry.services[i.serviceName] = @[]
  
  if i.id notin registry.services[i.serviceName]:
    registry.services[i.serviceName].add(i.id)
  
  echo fmt"[Registry] Registered: {i.serviceName}/{i.id} at {i.host}:{i.port}"

proc deregister(instanceId: string) =
  if instanceId notin registry.instances:
    return
  
  let inst = registry.instances[instanceId]
  registry.instances.del(instanceId)
  
  if inst.serviceName in registry.services:
    registry.services[inst.serviceName] = registry.services[inst.serviceName]
      .filterIt(it != instanceId)
  
  echo fmt"[Registry] Deregistered: {instanceId}"

proc heartbeat(instanceId: string) =
  if instanceId in registry.instances:
    registry.instances[instanceId].lastHeartbeatAt = epochTime()
    registry.instances[instanceId].healthy = true

proc getInstances(serviceName: string, healthyOnly = true): seq[ServiceInstance] =
  if serviceName notin registry.services:
    return @[]
  
  let now = epochTime()
  var result: seq[ServiceInstance]
  
  for id in registry.services[serviceName]:
    if id notin registry.instances: continue
    let inst = registry.instances[id]
    
    if healthyOnly:
      let stale = now - inst.lastHeartbeatAt > registry.heartbeatTimeout
      if not inst.healthy or stale: continue
    
    result.add(inst)
  
  return result

proc expireStaleInstances() =
  let now = epochTime()
  var toRemove: seq[string]
  
  for id, inst in registry.instances:
    if now - inst.lastHeartbeatAt > registry.heartbeatTimeout * 2:
      toRemove.add(id)
  
  for id in toRemove:
    echo fmt"[Registry] Expired stale instance: {id}"
    deregister(id)

# ============================
# Service discovery client
# ============================

type
  DiscoveryClient = object
    localCache: Table[string, tuple[instances: seq[ServiceInstance], cachedAt: float]]
    cacheTtl: float

var discoveryClient = DiscoveryClient(
  localCache: initTable[string, tuple[instances: seq[ServiceInstance], cachedAt: float]](),
  cacheTtl: 5.0  # 5 second cache
)

proc resolve(serviceName: string): seq[ServiceInstance] =
  let now = epochTime()
  
  # Check cache
  if serviceName in discoveryClient.localCache:
    let cached = discoveryClient.localCache[serviceName]
    if now - cached.cachedAt < discoveryClient.cacheTtl:
      return cached.instances
  
  # Fetch from registry
  let instances = getInstances(serviceName)
  discoveryClient.localCache[serviceName] = (instances, now)
  
  return instances

proc resolveOne(serviceName: string): Option[ServiceInstance] =
  let instances = resolve(serviceName)
  if instances.len == 0:
    return none(ServiceInstance)
  
  # Simple round-robin
  let idx = int(epochTime() * 1000) mod instances.len
  return some(instances[idx])

# ============================
# Demo
# ============================

proc demo() =
  echo "=== Service Registry Demo ==="
  
  # Register instances
  for i in 0..<3:
    register(ServiceInstance(
      id: fmt"user-svc-{i}",
      serviceName: "user-service",
      host: fmt"10.0.0.{i + 10}",
      port: 8080,
      version: "v2.1.0",
      tags: @["production", "us-east"],
      weight: 10
    ))
  
  register(ServiceInstance(
    id: "product-svc-0",
    serviceName: "product-service",
    host: "10.0.0.20",
    port: 8081,
    version: "v1.5.0",
    tags: @["production"]
  ))
  
  echo fmt"\nRegistered services: {toSeq(registry.services.keys)}"
  
  let userInstances = getInstances("user-service")
  echo fmt"user-service instances: {userInstances.len}"
  for inst in userInstances:
    echo fmt"  - {inst.id}: {inst.host}:{inst.port} v{inst.version}"
  
  # Simulate heartbeat
  heartbeat("user-svc-0")
  heartbeat("user-svc-1")
  
  # Resolve
  let resolved = resolveOne("user-service")
  if resolved.isSome:
    let r = resolved.get()
    echo fmt"\nResolved user-service: {r.id} at {r.host}:{r.port}"
  
  # Deregister one
  deregister("user-svc-2")
  echo fmt"\nAfter deregister: {getInstances(\"user-service\").len} instances"

demo()
```

---

## Step 707-720: Retry, Bulkhead, Tracing

```nim
import asyncdispatch, tables, strformat, times, math, json, sequtils, options

# ============================
# Retry with exponential backoff + jitter
# ============================

type
  RetryConfig = object
    maxAttempts: int
    baseDelayMs: int
    maxDelayMs: int
    jitterMs: int
    retryableErrors: seq[string]

  RetryResult[T] = object
    value: T
    attempts: int
    totalMs: float
    success: bool
    lastError: string

proc withRetry[T](config: RetryConfig, 
                   operation: proc(): Future[T] {.async.}): Future[RetryResult[T]] {.async.} =
  var result: RetryResult[T]
  let start = epochTime()
  
  for attempt in 1..config.maxAttempts:
    result.attempts = attempt
    
    try:
      result.value = await operation()
      result.success = true
      result.totalMs = (epochTime() - start) * 1000
      return result
    except CatchableError as e:
      result.lastError = e.msg
      echo fmt"[Retry] Attempt {attempt}/{config.maxAttempts} failed: {e.msg}"
      
      if attempt < config.maxAttempts:
        # Exponential backoff with jitter
        let baseDelay = min(config.baseDelayMs * (2 ^ (attempt - 1)), config.maxDelayMs)
        let jitter = rand(config.jitterMs)
        let delay = baseDelay + jitter
        echo fmt"[Retry] Waiting {delay}ms before retry..."
        await sleepAsync(delay)
  
  result.success = false
  result.totalMs = (epochTime() - start) * 1000
  return result

# ============================
# Bulkhead pattern
# ============================

type
  Bulkhead = object
    name: string
    maxConcurrent: int
    maxWaiting: int
    current: int
    waiting: int
    rejected: int
    completed: int

var bulkheads: Table[string, Bulkhead] = initTable[string, Bulkhead]()

proc newBulkhead(name: string, maxConcurrent, maxWaiting: int): string =
  bulkheads[name] = Bulkhead(
    name: name,
    maxConcurrent: maxConcurrent,
    maxWaiting: maxWaiting
  )
  return name

proc tryAcquire(name: string): bool =
  if name notin bulkheads: return true
  var b = bulkheads[name]
  
  if b.current >= b.maxConcurrent:
    if b.waiting >= b.maxWaiting:
      inc b.rejected
      bulkheads[name] = b
      echo fmt"[Bulkhead:{name}] Rejected (full: {b.current}/{b.maxConcurrent})"
      return false
    inc b.waiting
    bulkheads[name] = b
    return true  # will wait
  
  inc b.current
  bulkheads[name] = b
  return true

proc release(name: string) =
  if name notin bulkheads: return
  var b = bulkheads[name]
  if b.current > 0: dec b.current
  if b.waiting > 0: dec b.waiting
  inc b.completed
  bulkheads[name] = b

proc bulkheadStats(name: string): JsonNode =
  if name notin bulkheads: return newJNull()
  let b = bulkheads[name]
  %*{
    "current": b.current,
    "waiting": b.waiting,
    "rejected": b.rejected,
    "completed": b.completed,
    "utilization": float(b.current) / float(b.maxConcurrent) * 100
  }

# ============================
# Distributed tracing
# ============================

type
  SpanKind = enum
    skClient, skServer, skInternal, skProducer, skConsumer

  TraceSpan = object
    traceId: string
    spanId: string
    parentSpanId: string
    serviceName: string
    operationName: string
    kind: SpanKind
    startTime: float
    endTime: float
    status: string    # "ok" | "error"
    attributes: Table[string, string]
    events: seq[tuple[name: string, time: float, attrs: Table[string, string]]]
    links: seq[string]  # linked span IDs

  Tracer = object
    serviceName: string
    spans: seq[TraceSpan]

var tracers: Table[string, Tracer] = initTable[string, Tracer]()

proc newTracer(serviceName: string): Tracer =
  if serviceName in tracers:
    return tracers[serviceName]
  let t = Tracer(serviceName: serviceName, spans: @[])
  tracers[serviceName] = t
  return t

proc generateId(): string =
  fmt"{int(epochTime() * 1_000_000) mod 1_000_000_000_000:016x}"

proc startSpan(tracer: var Tracer, operation: string,
               parentId = "", kind = skInternal): TraceSpan =
  let traceId = if parentId.len > 0: parentId[0..15] else: generateId()
  
  TraceSpan(
    traceId: traceId,
    spanId: generateId(),
    parentSpanId: parentId,
    serviceName: tracer.serviceName,
    operationName: operation,
    kind: kind,
    startTime: epochTime(),
    status: "ok",
    attributes: initTable[string, string](),
    events: @[],
    links: @[]
  )

proc finishSpan(tracer: var Tracer, span: var TraceSpan,
                status = "ok", errorMsg = "") =
  span.endTime = epochTime()
  span.status = status
  if errorMsg.len > 0:
    span.attributes["error.message"] = errorMsg
  tracer.spans.add(span)
  
  let durationMs = int((span.endTime - span.startTime) * 1000)
  let statusIcon = if status == "ok": "✓" else: "✗"
  echo fmt"[Trace:{span.traceId[0..7]}] {statusIcon} {span.operationName} ({durationMs}ms)"

proc addSpanAttribute(span: var TraceSpan, key, value: string) =
  span.attributes[key] = value

proc addSpanEvent(span: var TraceSpan, name: string,
                  attrs: Table[string, string] = initTable[string, string]()) =
  span.events.add((name, epochTime(), attrs))

proc exportSpans(tracer: Tracer): JsonNode =
  var spans = newJArray()
  for s in tracer.spans:
    var attrs = newJObject()
    for k, v in s.attributes:
      attrs[k] = %v
    spans.add(%*{
      "traceId": s.traceId,
      "spanId": s.spanId,
      "parentSpanId": s.parentSpanId,
      "service": s.serviceName,
      "operation": s.operationName,
      "status": s.status,
      "durationMs": int((s.endTime - s.startTime) * 1000),
      "attributes": attrs
    })
  return spans

# ============================
# Demo
# ============================

proc demo() {.async.} =
  echo "=== Service Mesh Patterns Demo ==="
  
  # Retry demo
  echo "\n--- Retry with backoff ---"
  let retryConfig = RetryConfig(
    maxAttempts: 3,
    baseDelayMs: 50,
    maxDelayMs: 1000,
    jitterMs: 20,
    retryableErrors: @["connection_refused", "timeout"]
  )
  
  var callCount = 0
  let result = await withRetry(retryConfig, proc(): Future[string] {.async.} =
    inc callCount
    await sleepAsync(1)
    if callCount < 3:
      raise newException(IOError, "connection_refused")
    return "success"
  )
  
  echo fmt"Result: success={result.success} attempts={result.attempts}"
  
  # Bulkhead demo
  echo "\n--- Bulkhead ---"
  discard newBulkhead("db-pool", maxConcurrent: 3, maxWaiting: 2)
  
  for i in 0..<7:
    let ok = tryAcquire("db-pool")
    echo fmt"  Request {i+1}: {if ok: \"accepted\" else: \"rejected\"}"
  
  # Release some
  for _ in 0..<3: release("db-pool")
  echo "Bulkhead stats: " & $bulkheadStats("db-pool")
  
  # Tracing demo
  echo "\n--- Distributed Tracing ---"
  var tracer = newTracer("api-gateway")
  
  var rootSpan = tracer.startSpan("handle_request", kind = skServer)
  rootSpan.addSpanAttribute("http.method", "GET")
  rootSpan.addSpanAttribute("http.url", "/api/users/1")
  
  await sleepAsync(5)
  
  var dbSpan = tracer.startSpan("db.query", parentId = rootSpan.traceId, kind = skClient)
  dbSpan.addSpanAttribute("db.system", "postgresql")
  dbSpan.addSpanAttribute("db.statement", "SELECT * FROM users WHERE id = $1")
  
  await sleepAsync(10)
  tracer.finishSpan(dbSpan)
  
  rootSpan.addSpanEvent("user_found", {"user_id": "1"}.toTable())
  tracer.finishSpan(rootSpan)
  
  let exported = exportSpans(tracer)
  echo fmt"Exported {exported.len} spans"

waitFor demo()
```

---

## 📝 สรุป Part 49

| Steps | หัวข้อ |
|-------|--------|
| 706 | Service registry, health-aware discovery |
| 707-720 | Retry + jitter, bulkhead, distributed tracing |

---

**← [Part 48: API Gateway](part_48_api_gateway.md) | [Part 50: Data Pipeline →](part_50_data_pipeline.md)**
