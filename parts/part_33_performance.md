# Part 33: Performance Optimization
## Steps 466-480: เพิ่มประสิทธิภาพระดับ World-Class

---

## 🎯 เป้าหมายของ Part นี้

- Memory management (GC, ARC, ORC)
- Object pooling
- Zero-copy strings
- SIMD-friendly data structures
- Compile-time optimization
- Benchmark methodology
- Profiling techniques

---

## Step 466: Memory Management in Nim

```nim
# ============================
# GC Strategies
# ============================

# Default: Nim uses garbage collection
# --gc:refc   Reference counting (default in old Nim)
# --gc:arc    Automatic Reference Counting (deterministic)
# --gc:orc    ARC with cycle detection
# --gc:none   No GC (manual memory)
# --gc:boehm  Boehm GC

# For backend services: --gc:orc or --gc:arc is recommended
# - Deterministic memory deallocation
# - Lower latency spikes
# - Works well with async

# Compile with: nim c --gc:orc -d:release myapp.nim

# ============================
# Stack vs Heap allocation
# ============================

type
  # Stack allocated (fast, automatic cleanup)
  Point = object  # NOT ref
    x, y: float

  # Heap allocated (needs GC or manual free)
  BigBuffer = ref object
    data: array[65536, byte]

proc stackDemo() =
  var p = Point(x: 1.0, y: 2.0)  # on stack
  echo p.x  # No allocation, no GC pressure

proc heapDemo() =
  var buf = BigBuffer()  # on heap
  buf.data[0] = 0xFF
  # GC/ARC frees when buf goes out of scope

# ============================
# Avoid allocations in hot paths
# ============================

import strformat, times

# BAD: allocates a new string on each call
proc badFormat(x, y: int): string =
  fmt"({x}, {y})"  # heap allocation

# GOOD: write to buffer
proc goodFormat(x, y: int, buf: var string) =
  buf.setLen(0)
  buf.add('(')
  buf.addInt(x)
  buf.add(", ")
  buf.addInt(y)
  buf.add(')')

proc benchmark() =
  const N = 1_000_000
  
  var start = cpuTime()
  var s = ""
  for i in 0..<N:
    s = badFormat(i, i * 2)
  echo fmt"Bad format: {(cpuTime() - start) * 1000:.1f}ms"
  
  start = cpuTime()
  var buf = newStringOfCap(32)
  for i in 0..<N:
    goodFormat(i, i * 2, buf)
  echo fmt"Good format: {(cpuTime() - start) * 1000:.1f}ms"

benchmark()
```

---

## Step 467: Object Pooling

```nim
import times, strformat, sequtils, deques

# ============================
# Object Pool for reuse
# ============================

type
  Connection = object
    id: int
    inUse: bool
    createdAt: float
    lastUsed: float
    queryCount: int

  ConnectionPool = object
    all: seq[Connection]
    available: Deque[int]  # indices into all
    maxSize: int
    minSize: int
    borrowed: int

proc newConnectionPool(minSize, maxSize: int): ConnectionPool =
  var pool = ConnectionPool(
    all: @[],
    available: initDeque[int](),
    maxSize: maxSize,
    minSize: minSize
  )
  
  # Pre-allocate minimum connections
  for i in 0..<minSize:
    let conn = Connection(
      id: i,
      inUse: false,
      createdAt: epochTime()
    )
    pool.all.add(conn)
    pool.available.addLast(i)
  
  return pool

proc borrow(pool: var ConnectionPool): int =
  if pool.available.len > 0:
    let idx = pool.available.popFirst()
    pool.all[idx].inUse = true
    pool.all[idx].lastUsed = epochTime()
    inc pool.borrowed
    return idx
  
  if pool.all.len < pool.maxSize:
    let idx = pool.all.len
    pool.all.add(Connection(
      id: idx,
      inUse: true,
      createdAt: epochTime(),
      lastUsed: epochTime()
    ))
    inc pool.borrowed
    return idx
  
  raise newException(IOError, "Connection pool exhausted")

proc returnConn(pool: var ConnectionPool, idx: int) =
  pool.all[idx].inUse = false
  inc pool.all[idx].queryCount
  pool.available.addLast(idx)
  dec pool.borrowed

template withConnection(pool: var ConnectionPool, connVar, body: untyped) =
  let connVar = pool.borrow()
  try:
    body
  finally:
    pool.returnConn(connVar)

proc stats(pool: ConnectionPool) =
  echo fmt"Pool: total={pool.all.len}, available={pool.available.len}, borrowed={pool.borrowed}"

# Demo
var pool = newConnectionPool(minSize = 2, maxSize = 10)
pool.stats()

for i in 1..5:
  withConnection(pool, conn):
    echo fmt"Using connection {conn}"
    # Simulate work
    discard

pool.stats()

# ============================
# String interning
# ============================

type
  StringPool = object
    strings: seq[string]
    index: Table[string, int]

var strPool = StringPool(
  strings: @[],
  index: initTable[string, int]()
)

proc intern(pool: var StringPool, s: string): int =
  if s in pool.index:
    return pool.index[s]
  
  let id = pool.strings.len
  pool.strings.add(s)
  pool.index[s] = id
  return id

proc lookup(pool: StringPool, id: int): string =
  pool.strings[id]

# When many objects share the same string values, interning saves memory
let id1 = strPool.intern("application/json")
let id2 = strPool.intern("application/json")  # returns same id
let id3 = strPool.intern("text/html")

echo fmt"Same string: {id1 == id2}"  # true
echo fmt"Different: {id1 == id3}"    # false
echo fmt"Pool size: {strPool.strings.len}"  # 2
```

---

## Step 468: Compile-Time Optimization

```nim
import macros, strutils, times

# ============================
# Compile-time computation
# ============================

# Values computed at compile time
const
  MAX_CONNECTIONS = 100
  HASH_TABLE_SIZE = 1024  # power of 2 for fast modulo

# Static tables (computed at compile time)
const httpStatusMessages: array[100..599, string] = block:
  var msgs: array[100..599, string]
  msgs[200] = "OK"
  msgs[201] = "Created"
  msgs[204] = "No Content"
  msgs[301] = "Moved Permanently"
  msgs[302] = "Found"
  msgs[304] = "Not Modified"
  msgs[400] = "Bad Request"
  msgs[401] = "Unauthorized"
  msgs[403] = "Forbidden"
  msgs[404] = "Not Found"
  msgs[405] = "Method Not Allowed"
  msgs[409] = "Conflict"
  msgs[422] = "Unprocessable Entity"
  msgs[429] = "Too Many Requests"
  msgs[500] = "Internal Server Error"
  msgs[502] = "Bad Gateway"
  msgs[503] = "Service Unavailable"
  msgs

proc getStatusMessage(code: int): string =
  if code >= 100 and code <= 599 and httpStatusMessages[code].len > 0:
    return httpStatusMessages[code]
  return "Unknown"

# ============================
# Inline pragma for small functions
# ============================

func min2(a, b: int): int {.inline.} =
  if a < b: a else: b

func max2(a, b: int): int {.inline.} =
  if a > b: a else: b

func clamp2(x, lo, hi: int): int {.inline.} =
  min2(max2(x, lo), hi)

# ============================
# Template for zero-overhead abstraction
# ============================

template swap2[T](a, b: var T) {.dirty.} =
  let tmp = a
  a = b
  b = tmp

template doTimes(n: int, body: untyped) =
  var i {.inject.} = 0
  while i < n:
    body
    inc i

# ============================
# Benchmark helper
# ============================

template bench(name: string, iterations: int, body: untyped): untyped =
  let start = cpuTime()
  for _ in 1..iterations:
    body
  let elapsed = cpuTime() - start
  let perOp = elapsed / float(iterations) * 1_000_000.0  # microseconds
  echo fmt"{name}: {elapsed * 1000:.1f}ms total, {perOp:.3f}µs/op"

# Test
bench("getStatusMessage x1M", 1_000_000):
  discard getStatusMessage(404)

bench("string concat x100k", 100_000):
  let s = "hello" & " " & "world"
  discard s

bench("fmt x100k", 100_000):
  let x = 42
  let s = fmt"value={x}"
  discard s

echo "\nStatus: " & getStatusMessage(200)
echo "Status: " & getStatusMessage(404)
echo "Status: " & getStatusMessage(999)
```

---

## Step 469-480: Complete High-Performance Backend

```nim
# highperf_backend.nim - Performance-optimized Nim backend patterns

import asyncdispatch, asynchttpserver, times, strutils, strformat,
       tables, deques, json, hashes, options, sequtils

# ============================
# Fast routing (trie-based concept)
# ============================

type
  RouteParamKind = enum
    Static, Dynamic

  RoutePart = object
    kind: RouteParamKind
    value: string

  Route = object
    parts: seq[RoutePart]
    httpMethod: string
    handler: proc(req: Request, params: Table[string, string]): Future[void]

  Router2 = object
    routes: seq[Route]

proc parseRoute(pattern: string): seq[RoutePart] =
  for part in pattern.strip(chars = {'/'}).split('/'):
    if part.startsWith(':'):
      result.add(RoutePart(kind: Dynamic, value: part[1..^1]))
    elif part.startsWith('{') and part.endsWith('}'):
      result.add(RoutePart(kind: Dynamic, value: part[1..^2]))
    else:
      result.add(RoutePart(kind: Static, value: part))

proc on(router: var Router2, httpMethod, pattern: string,
        handler: proc(req: Request, params: Table[string, string]): Future[void]) =
  router.routes.add(Route(
    parts: parseRoute(pattern),
    httpMethod: httpMethod.toUpperAscii(),
    handler: handler
  ))

proc matchRoute(router: Router2, reqMethod, path: string):
    Option[tuple[handler: proc(req: Request, params: Table[string, string]): Future[void], params: Table[string, string]]] =
  
  let pathParts = path.strip(chars = {'/'}).split('/')
  
  for route in router.routes:
    if route.httpMethod != reqMethod:
      continue
    if route.parts.len != pathParts.len:
      continue
    
    var params = initTable[string, string]()
    var matched = true
    
    for i, part in route.parts:
      case part.kind
      of Static:
        if part.value != pathParts[i]:
          matched = false
          break
      of Dynamic:
        params[part.value] = pathParts[i]
    
    if matched:
      return some((handler: route.handler, params: params))
  
  return none(tuple[handler: proc(req: Request, params: Table[string, string]): Future[void], params: Table[string, string]])

# ============================
# Response builder (minimize allocations)
# ============================

type
  ResponseWriter = object
    status: HttpCode
    headers: HttpHeaders
    body: string

proc newResponseWriter(status: HttpCode = Http200): ResponseWriter =
  ResponseWriter(
    status: status,
    headers: newHttpHeaders([("Content-Type", "application/json")]),
    body: ""
  )

proc json2(rw: var ResponseWriter, data: JsonNode) =
  rw.body = $data

proc send(rw: ResponseWriter, req: Request) {.async.} =
  await req.respond(rw.status, rw.body, rw.headers)

# ============================
# In-memory response cache
# ============================

type
  CachedResponse = object
    body: string
    headers: HttpHeaders
    status: HttpCode
    expiresAt: float

  ResponseCache = object
    items: Table[uint64, CachedResponse]
    maxItems: int

proc newResponseCache(maxItems: int = 1000): ResponseCache =
  ResponseCache(
    items: initTable[uint64, CachedResponse](),
    maxItems: maxItems
  )

proc cacheKey(httpMethod, path, query: string): uint64 =
  hash(httpMethod & path & query).uint64

proc get2(cache: ResponseCache, key: uint64): Option[CachedResponse] =
  if key in cache.items:
    let item = cache.items[key]
    if epochTime() < item.expiresAt:
      return some(item)
  return none(CachedResponse)

proc set2(cache: var ResponseCache, key: uint64, resp: CachedResponse) =
  if cache.items.len >= cache.maxItems:
    # Simple eviction: remove first item
    for k in cache.items.keys:
      cache.items.del(k)
      break
  cache.items[key] = resp

# ============================
# Performance metrics (lock-free counters)
# ============================

type
  AtomicCounter = object
    val: int

  PerfMetrics = object
    totalRequests: int
    totalErrors: int
    totalBytesOut: int
    p50: float
    p95: float
    p99: float
    durations: seq[float]

var perf = PerfMetrics(durations: @[])

proc recordRequest(durationMs: float, statusCode: int) =
  inc perf.totalRequests
  if statusCode >= 400:
    inc perf.totalErrors
  
  perf.durations.add(durationMs)
  
  # Compute percentiles every 1000 requests
  if perf.durations.len >= 1000:
    let sorted = perf.durations.sorted()
    perf.p50 = sorted[500]
    perf.p95 = sorted[950]
    perf.p99 = sorted[990]
    perf.durations.setLen(0)

# ============================
# High-performance HTTP handler
# ============================

var appRouter = Router2(routes: @[])
var responseCache = newResponseCache(1000)

appRouter.on("GET", "/api/ping", proc(req: Request, params: Table[string, string]) {.async.} =
  var rw = newResponseWriter()
  rw.json2(%*{"pong": true, "ts": epochTime()})
  await rw.send(req)
)

appRouter.on("GET", "/api/users/:id", proc(req: Request, params: Table[string, string]) {.async.} =
  let userId = params.getOrDefault("id", "0")
  var rw = newResponseWriter()
  rw.json2(%*{"id": parseInt(userId), "name": "User " & userId})
  await rw.send(req)
)

appRouter.on("GET", "/metrics", proc(req: Request, params: Table[string, string]) {.async.} =
  let resp = $(%*{
    "requests": perf.totalRequests,
    "errors": perf.totalErrors,
    "p50_ms": perf.p50,
    "p95_ms": perf.p95,
    "p99_ms": perf.p99
  })
  await req.respond(Http200, resp,
    newHttpHeaders([("Content-Type", "application/json")]))
)

proc highPerfHandler(req: Request) {.async.} =
  let t0 = epochTime()
  
  # Check response cache
  let cKey = cacheKey($req.reqMethod, req.url.path, req.url.query)
  let cached = responseCache.get2(cKey)
  
  if cached.isSome:
    let c = cached.get()
    await req.respond(c.status, c.body, c.headers)
    recordRequest((epochTime() - t0) * 1000, c.status.int)
    return
  
  # Route matching
  let match = appRouter.matchRoute($req.reqMethod, req.url.path)
  
  if match.isSome:
    let (handler, params) = match.get()
    await handler(req, params)
  else:
    await req.respond(Http404, """{"error":"not found"}""",
      newHttpHeaders([("Content-Type", "application/json")]))
  
  recordRequest((epochTime() - t0) * 1000, 200)

# ============================
# Demo
# ============================

proc demo() =
  echo "=== High-Performance Backend Demo ==="
  
  echo "\nRoutes registered:"
  for route in appRouter.routes:
    let pathStr = route.parts.mapIt(
      if it.kind == Static: it.value else: ":" & it.value
    ).join("/")
    echo fmt"  {route.httpMethod} /{pathStr}"
  
  # Simulate route matching
  echo "\nRoute matching:"
  let tests = [
    ("GET", "/api/ping"),
    ("GET", "/api/users/42"),
    ("POST", "/api/users/42"),
    ("GET", "/unknown"),
  ]
  
  for (meth, path) in tests:
    let m = appRouter.matchRoute(meth, path)
    if m.isSome:
      echo fmt"  {meth} {path} -> matched (params: {m.get().params})"
    else:
      echo fmt"  {meth} {path} -> 404"
  
  # Simulate metrics
  for i in 0..<100:
    recordRequest(float(5 + i mod 95), 200)
  for i in 0..<5:
    recordRequest(float(200 + i * 50), 500)
  
  echo fmt"\nMetrics after 105 requests:"
  echo fmt"  Total: {perf.totalRequests}"
  echo fmt"  Errors: {perf.totalErrors}"

demo()
```

---

## 📝 สรุป Part 33

| Steps | หัวข้อ |
|-------|--------|
| 466 | Memory management, GC:arc/orc |
| 467 | Object pooling, string interning |
| 468 | Compile-time optimization, benchmarking |
| 469-480 | High-perf router, response cache, metrics |

---

**← [Part 32: Security](part_32_security.md) | [Part 34: Deployment →](part_34_deployment.md)**
