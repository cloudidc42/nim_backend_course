# Part 20: Performance และ Optimization ด้วย Nim

## Steps 271-285

Nim มีความสามารถด้าน performance ที่ยอดเยี่ยม ในบทนี้เราจะเรียนรู้วิธีการ profiling, benchmarking, และ optimization เพื่อสร้าง high-performance API ที่รองรับ load สูง

---

## Step 271: Profiling Nim Code

```nim
# file: src/profiling_demo.nim
import times, strformat, sequtils, algorithm, random

# ===== ฟังก์ชันที่จะ profile =====

proc bubbleSort(arr: var seq[int]) =
  for i in 0..<arr.len:
    for j in 0..<arr.len - i - 1:
      if arr[j] > arr[j+1]:
        swap(arr[j], arr[j+1])

proc mergeSortHelper(arr: seq[int]): seq[int] =
  if arr.len <= 1:
    return arr
  
  let mid = arr.len div 2
  let left = mergeSortHelper(arr[0..<mid])
  let right = mergeSortHelper(arr[mid..^1])
  
  var result: seq[int] = @[]
  var i, j = 0
  
  while i < left.len and j < right.len:
    if left[i] <= right[j]:
      result.add(left[i])
      inc i
    else:
      result.add(right[j])
      inc j
  
  result.add(left[i..^1])
  result.add(right[j..^1])
  return result

# ===== Manual Timing =====

proc timeIt[T](name: string, runs: int, fn: proc(): T): T =
  let start = cpuTime()
  var result: T
  for i in 0..<runs:
    result = fn()
  let elapsed = cpuTime() - start
  let avgMs = (elapsed / runs.float) * 1000.0
  echo &"[TIMER] {name}: {elapsed * 1000:.2f}ms total, {avgMs:.4f}ms avg ({runs} runs)"
  return result

proc benchmarkSorts() =
  randomize()
  let size = 10_000
  let data = (0..<size).mapIt(rand(1_000_000))
  
  echo "=== Sort Algorithm Benchmarks ==="
  
  # Bubble sort (slow)
  var bubbleData = data
  timeIt("Bubble Sort", 1) do() -> void:
    bubbleSort(bubbleData)
  
  # Merge sort
  timeIt("Merge Sort", 10) do() -> seq[int]:
    mergeSortHelper(data)
  
  # Built-in sort (fastest)
  timeIt("std/algorithm sort", 100) do() -> seq[int]:
    var d = data
    d.sort()
    d

# ===== CPU Time vs Wall Time =====

proc demonstrateTiming() =
  echo "\n=== CPU Time vs Wall Time ==="
  
  let wallStart = epochTime()
  let cpuStart = cpuTime()
  
  # CPU-intensive work
  var sum = 0
  for i in 0..<10_000_000:
    sum += i
  
  let wallElapsed = epochTime() - wallStart
  let cpuElapsed = cpuTime() - cpuStart
  
  echo &"Result: {sum}"
  echo &"Wall time: {wallElapsed * 1000:.2f}ms"
  echo &"CPU time: {cpuElapsed * 1000:.2f}ms"

# ===== Memory Allocation Profiling =====

proc measureAllocations() =
  echo "\n=== Allocation Patterns ==="
  
  # Bad: many small allocations
  var badList: seq[string] = @[]
  let badStart = cpuTime()
  for i in 0..<10_000:
    badList.add(&"item_{i}")
  echo &"Many allocations: {(cpuTime() - badStart) * 1000:.2f}ms"
  
  # Better: pre-allocate
  var goodList = newSeqOfCap[string](10_000)
  let goodStart = cpuTime()
  for i in 0..<10_000:
    goodList.add(&"item_{i}")
  echo &"Pre-allocated: {(cpuTime() - goodStart) * 1000:.2f}ms"
  
  # Best: avoid string formatting in loop
  var intList = newSeqOfCap[int](10_000)
  let bestStart = cpuTime()
  for i in 0..<10_000:
    intList.add(i)
  echo &"Integer list: {(cpuTime() - bestStart) * 1000:.3f}ms"

benchmarkSorts()
demonstrateTiming()
measureAllocations()
```

---

## Step 272: std/monotimes สำหรับ High-Precision Timing

```nim
# file: src/monotime_bench.nim
import std/monotimes, std/times, strformat, sequtils

type
  BenchmarkResult = object
    name: string
    iterations: int
    totalNs: int64
    minNs: int64
    maxNs: int64
    avgNs: float
    medianNs: int64

proc benchmark(name: string, iterations: int, fn: proc()): BenchmarkResult =
  var times: seq[int64] = newSeqOfCap[int64](iterations)
  
  # Warmup runs
  for _ in 0..<min(10, iterations div 10):
    fn()
  
  # Actual benchmark
  for _ in 0..<iterations:
    let start = getMonoTime()
    fn()
    let elapsed = (getMonoTime() - start).inNanoseconds
    times.add(elapsed)
  
  times.sort()
  
  let total = times.foldl(a + b, 0i64)
  
  BenchmarkResult(
    name: name,
    iterations: iterations,
    totalNs: total,
    minNs: times[0],
    maxNs: times[^1],
    avgNs: total.float / iterations.float,
    medianNs: times[iterations div 2]
  )

proc printResult(r: BenchmarkResult) =
  echo &"Benchmark: {r.name}"
  echo &"  Iterations: {r.iterations}"
  echo &"  Min:    {r.minNs}ns  ({r.minNs.float/1000:.2f}µs)"
  echo &"  Max:    {r.maxNs}ns  ({r.maxNs.float/1000:.2f}µs)"
  echo &"  Avg:    {r.avgNs:.0f}ns  ({r.avgNs/1000:.2f}µs)"
  echo &"  Median: {r.medianNs}ns  ({r.medianNs.float/1000:.2f}µs)"
  echo &"  Throughput: {1_000_000_000.0/r.avgNs:.0f} ops/sec"
  echo ""

# ===== Benchmark ต่างๆ =====

proc benchmarkStringConcat() =
  # Bad: string concatenation in loop
  let r1 = benchmark("String concat (+=)", 1000) do():
    var s = ""
    for i in 0..<100:
      s = s & $i
  
  # Good: use join
  let r2 = benchmark("String join", 1000) do():
    var parts: seq[string] = @[]
    for i in 0..<100:
      parts.add($i)
    discard parts.join("")
  
  # Better: StringBuilder pattern
  let r3 = benchmark("String add (seq)", 1000) do():
    var parts = newSeqOfCap[string](100)
    for i in 0..<100:
      parts.add($i)
    discard parts.join("")
  
  printResult(r1)
  printResult(r2)
  printResult(r3)

proc benchmarkTableVsSeq() =
  import tables
  
  let data = (0..<1000).toSeq()
  var t = initTable[int, int]()
  for i in data: t[i] = i * i
  
  # Table lookup
  let r1 = benchmark("Table lookup", 10000) do():
    for i in 0..<100:
      discard t.getOrDefault(i, 0)
  
  # Sequential search
  let r2 = benchmark("Seq linear search", 1000) do():
    for i in 0..<100:
      for j in data:
        if j == i: break
  
  printResult(r1)
  printResult(r2)

proc benchmarkHashingAlgorithms() =
  import hashes
  
  let s = "Hello, World! This is a test string for benchmarking."
  
  let r1 = benchmark("Hash string", 100000) do():
    discard hash(s)
  
  let r2 = benchmark("Hash int", 100000) do():
    discard hash(42)
  
  printResult(r1)
  printResult(r2)

echo "=== Performance Benchmarks ==="
benchmarkStringConcat()
benchmarkTableVsSeq()
benchmarkHashingAlgorithms()
```

---

## Step 273: Memory Management และ GC Strategies

```nim
# file: src/memory_management.nim
import strformat, times

# ===== GC Modes สำหรับ Backend =====
# Compile flags:
# --gc:refc   (default) - Reference counting + cycle collection
# --gc:arc    - Automatic Reference Counting (deterministic)
# --gc:orc    - Optimized ARC (handles cycles)
# --gc:none   - No GC (manual memory)
# --mm:arc    - New memory model (Nim 2.x)
# --mm:orc    - ORC (recommended for most apps)

# การเลือก:
# - Web API: --mm:orc (หรือ --gc:orc)
# - Real-time: --mm:arc --gc:arc
# - High throughput: --mm:arc -d:release --opt:speed

type
  # ใช้ ref object สำหรับ heap allocation
  HeapObject = ref object
    data: seq[int]
    name: string

  # ใช้ object (value type) สำหรับ stack allocation
  StackObject = object
    x, y: float
    active: bool

proc demonstrateMemoryPatterns() =
  echo "=== Memory Patterns ==="
  
  # Stack allocation (fast, no GC pressure)
  let start1 = cpuTime()
  for i in 0..<1_000_000:
    var s = StackObject(x: i.float, y: i.float * 2, active: true)
    discard s.x + s.y
  echo &"Stack alloc: {(cpuTime() - start1) * 1000:.2f}ms"
  
  # Heap allocation (with GC)
  let start2 = cpuTime()
  for i in 0..<100_000:
    var h = HeapObject(data: @[1, 2, 3], name: &"obj_{i}")
    discard h.data.len
  echo &"Heap alloc: {(cpuTime() - start2) * 1000:.2f}ms"
  
  # Object pool pattern
  let start3 = cpuTime()
  var pool: seq[StackObject] = newSeqOfCap[StackObject](1_000_000)
  for i in 0..<1_000_000:
    pool.add(StackObject(x: i.float, y: i.float * 2, active: true))
  echo &"Object pool: {(cpuTime() - start3) * 1000:.2f}ms"

# ===== Avoiding Unnecessary Allocations =====

proc badStringBuilding(n: int): string =
  var result = ""
  for i in 0..<n:
    result = result & $i & ","  # O(n²) allocations!
  return result

proc goodStringBuilding(n: int): string =
  var parts = newSeqOfCap[string](n)
  for i in 0..<n:
    parts.add($i)
  return parts.join(",")  # One allocation

proc bestStringBuilding(n: int): string =
  result = newStringOfCap(n * 3)  # Pre-size estimate
  for i in 0..<n:
    if i > 0: result.add(',')
    result.add($i)

proc compareStringBuilding() =
  echo "\n=== String Building ==="
  let n = 10_000
  
  let t1 = cpuTime()
  let s1 = badStringBuilding(n)
  echo &"Bad (concat): {(cpuTime()-t1)*1000:.2f}ms, len={s1.len}"
  
  let t2 = cpuTime()
  let s2 = goodStringBuilding(n)
  echo &"Good (join): {(cpuTime()-t2)*1000:.2f}ms, len={s2.len}"
  
  let t3 = cpuTime()
  let s3 = bestStringBuilding(n)
  echo &"Best (add): {(cpuTime()-t3)*1000:.2f}ms, len={s3.len}"

# ===== Nim 2.x Memory Management =====

# เปิดใช้ ARC/ORC:
# nim c --mm:arc -d:release src/server.nim

# ARC benefits:
# 1. Deterministic destruction (no GC pauses)
# 2. Better cache performance
# 3. Lower memory overhead

# ORC adds:
# 1. Cycle detection (safe with ref cycles)
# 2. Similar performance to ARC

demonstrateMemoryPatterns()
compareStringBuilding()
```

---

## Step 274: Compile-Time Optimizations

```nim
# file: src/compile_optimizations.nim

# ===== Compile flags ที่สำคัญ =====
# nim c -d:release              # Enable optimizations, disable assertions
# nim c -d:danger               # Like release but also disables bound checks
# nim c --opt:speed             # Optimize for speed
# nim c --opt:size              # Optimize for binary size
# nim c -d:lto                  # Link-time optimization
# nim c --passC:"-march=native" # CPU-specific optimizations
# nim c --gc:arc                # Use ARC GC
# nim c -d:useMalloc            # Use system malloc instead

# ===== SIMD และ Vectorization =====
# ใช้ {.align.} pragma สำหรับ SIMD-friendly data layout

type
  Vector4 {.align(16).} = object
    x, y, z, w: float32

  Matrix4x4 {.align(64).} = object
    data: array[16, float32]

# ===== Compile-Time Computation =====

# ใช้ const สำหรับ compile-time calculation
const
  MaxConnections = 1000
  BufferSize = 4096
  CacheLineSize = 64
  
  # คำนวณตอน compile time
  MaxBuffers = MaxConnections * 2
  TotalMemory = MaxBuffers * BufferSize

static:
  echo "MaxBuffers: ", MaxBuffers
  echo "TotalMemory: ", TotalMemory, " bytes"

# ===== Template สำหรับ Zero-Cost Abstractions =====

template measureTime*(name: string, body: untyped) =
  let _start = cpuTime()
  body
  let _elapsed = cpuTime() - _start
  echo name & ": " & $(_elapsed * 1000) & "ms"

template withBuffer*(size: int, name: untyped, body: untyped) =
  var name = newSeqOfCap[byte](size)
  body

# ===== Inline Functions =====

proc addFast(a, b: int): int {.inline.} =
  a + b

proc sumArrayFast(arr: openArray[int]): int64 {.inline.} =
  var sum: int64 = 0
  for x in arr:
    sum += x
  return sum

# ===== Pragma สำหรับ Performance =====

# {.noSideEffect.} - บอก compiler ว่าฟังก์ชันไม่มี side effects
proc pureCompute(x, y: int): int {.noSideEffect.} =
  x * x + y * y

# {.compiletime.} - รันตอน compile time
proc fib(n: int): int64 {.compiletime.} =
  if n <= 1: return n.int64
  return fib(n-1) + fib(n-2)

const fib20 = fib(20)  # คำนวณตอน compile time!

# ===== Unroll loops =====
proc sumUnrolled(arr: array[8, int]): int =
  # Manual loop unrolling
  arr[0] + arr[1] + arr[2] + arr[3] +
  arr[4] + arr[5] + arr[6] + arr[7]

# ===== Bit manipulation =====
proc isPowerOfTwo(n: int): bool {.inline.} =
  n > 0 and (n and (n - 1)) == 0

proc nextPowerOfTwo(n: int): int =
  var result = 1
  while result < n:
    result = result shl 1
  return result

proc popcount(n: uint64): int =
  # Count set bits
  var x = n
  var count = 0
  while x != 0:
    count += int(x and 1)
    x = x shr 1
  return count

proc demonstrateOptimizations() =
  import strformat, times
  
  echo &"fib(20) computed at compile time: {fib20}"
  echo &"isPowerOfTwo(64): {isPowerOfTwo(64)}"
  echo &"nextPowerOfTwo(100): {nextPowerOfTwo(100)}"
  echo &"popcount(255): {popcount(255)}"
  
  # Benchmark: compile time vs runtime
  measureTime("Sum 10M ints"):
    var sum: int64 = 0
    for i in 0..<10_000_000:
      sum += i
    echo &"  sum = {sum}"

demonstrateOptimizations()
```

---

## Step 275: Async Performance Tuning

```nim
# file: src/async_performance.nim
import asyncdispatch, asynchttpserver, httpcore
import strformat, times, atomics, tables, json

# ===== Connection Pooling =====

type
  PooledConnection[T] = ref object
    conn: T
    inUse: bool
    lastUsed: float

  ConnectionPool[T] = ref object
    connections: seq[PooledConnection[T]]
    maxSize: int
    createFn: proc(): Future[T] {.async.}
    waiters: seq[Future[T]]

proc newConnectionPool[T](maxSize: int, 
                          createFn: proc(): Future[T] {.async.}): ConnectionPool[T] =
  ConnectionPool[T](
    connections: @[],
    maxSize: maxSize,
    createFn: createFn,
    waiters: @[]
  )

proc acquire[T](pool: ConnectionPool[T]): Future[T] {.async.} =
  # หา connection ที่ว่าง
  for conn in pool.connections:
    if not conn.inUse:
      conn.inUse = true
      conn.lastUsed = epochTime()
      return conn.conn
  
  # สร้าง connection ใหม่ถ้าไม่เกิน maxSize
  if pool.connections.len < pool.maxSize:
    let newConn = await pool.createFn()
    let pooled = PooledConnection[T](
      conn: newConn,
      inUse: true,
      lastUsed: epochTime()
    )
    pool.connections.add(pooled)
    return newConn
  
  # รอ connection ว่าง
  var fut = newFuture[T]("pool.acquire")
  pool.waiters.add(fut)
  return await fut

proc release[T](pool: ConnectionPool[T], conn: T) =
  for pc in pool.connections:
    if pc.conn == conn:
      pc.inUse = false
      pc.lastUsed = epochTime()
      
      # ให้ waiter รอคิว
      if pool.waiters.len > 0:
        let waiter = pool.waiters[0]
        pool.waiters.del(0)
        pc.inUse = true
        waiter.complete(conn)
      break

# ===== Request Batching =====

type
  BatchRequest = object
    key: string
    future: Future[string]

  BatchProcessor = ref object
    pending: seq[BatchRequest]
    processing: bool
    batchSize: int
    maxWaitMs: int

proc newBatchProcessor(batchSize = 50, maxWaitMs = 10): BatchProcessor =
  BatchProcessor(
    pending: @[],
    processing: false,
    batchSize: batchSize,
    maxWaitMs: maxWaitMs
  )

proc processBatch(bp: BatchProcessor) {.async.} =
  if bp.processing or bp.pending.len == 0:
    return
  
  bp.processing = true
  
  while bp.pending.len > 0:
    let batch = bp.pending[0..<min(bp.batchSize, bp.pending.len)]
    bp.pending = bp.pending[batch.len..^1]
    
    # Process batch (e.g., database query for all keys at once)
    let keys = batch.mapIt(it.key)
    
    # Simulate batch DB query
    await sleepAsync(5)
    
    for req in batch:
      req.future.complete(&"value_for_{req.key}")
  
  bp.processing = false

proc get(bp: BatchProcessor, key: string): Future[string] {.async.} =
  var fut = newFuture[string]("batch.get")
  bp.pending.add(BatchRequest(key: key, future: fut))
  
  # Start processing if not already running
  if not bp.processing:
    asyncCheck bp.processBatch()
  
  return await fut

# ===== HTTP Server Performance =====

type
  PerfMetrics = ref object
    requestCount: Atomic[int64]
    errorCount: Atomic[int64]
    totalLatencyNs: Atomic[int64]
    activeConnections: Atomic[int]

proc newPerfMetrics(): PerfMetrics =
  PerfMetrics()

proc recordRequest(m: PerfMetrics, latencyNs: int64, success: bool) =
  m.requestCount.atomicInc()
  if not success: m.errorCount.atomicInc()
  discard m.totalLatencyNs.atomicAddFetch(latencyNs)

proc getStats(m: PerfMetrics): JsonNode =
  let count = m.requestCount.load()
  let errors = m.errorCount.load()
  let totalLatency = m.totalLatencyNs.load()
  
  let avgLatencyMs = if count > 0: 
    totalLatency.float / count.float / 1_000_000.0 
  else: 0.0
  
  let errorRate = if count > 0:
    errors.float / count.float * 100
  else: 0.0
  
  %*{
    "totalRequests": count,
    "errors": errors,
    "errorRate": &"{errorRate:.2f}%",
    "avgLatencyMs": &"{avgLatencyMs:.2f}",
    "activeConnections": m.activeConnections.load()
  }

# ===== Optimized HTTP Server =====

proc buildOptimizedServer(port: int) {.async.} =
  let metrics = newPerfMetrics()
  var server = newAsyncHttpServer(maxBody = 1024 * 1024)  # 1MB max body
  
  # Response cache สำหรับ static responses
  var responseCache = initTable[string, string]()
  responseCache["/health"] = $(%*{"status": "ok"})
  
  proc handler(req: Request) {.async.} =
    let startTime = getMonoTime()
    metrics.activeConnections.atomicInc()
    
    var status = Http200
    var body = ""
    
    let headers = newHttpHeaders([
      ("Content-Type", "application/json"),
      ("Connection", "keep-alive"),
      ("X-Server", "nim-optimized")
    ])
    
    try:
      # Cache hit
      if req.url.path in responseCache:
        body = responseCache[req.url.path]
      else:
        case req.url.path
        of "/api/fast":
          # Fast path - no allocation
          body = """{"result":"fast"}"""
        of "/api/compute":
          var sum: int64 = 0
          for i in 0..<1000:
            sum += i
          body = &"""{{\"sum\":{sum}}}"""
        of "/metrics":
          body = $metrics.getStats()
        else:
          status = Http404
          body = """{"error":"not found"}"""
      
      await req.respond(status, body, headers)
    finally:
      let elapsed = (getMonoTime() - startTime).inNanoseconds
      metrics.recordRequest(elapsed, status == Http200)
      metrics.activeConnections.atomicDec()
  
  echo &"Optimized server on port {port}"
  await server.serve(Port(port), handler)

waitFor buildOptimizedServer(8080)
```

---

## Step 276: HTTP Benchmarking เตรียม wrk

```bash
#!/bin/bash
# file: scripts/benchmark.sh

# ติดตั้ง wrk
# apt-get install wrk
# หรือ brew install wrk

SERVER_URL="http://localhost:8080"
DURATION="30s"
CONNECTIONS=100
THREADS=4

echo "=== HTTP Benchmarking with wrk ==="
echo "Server: $SERVER_URL"
echo "Duration: $DURATION"
echo "Connections: $CONNECTIONS"
echo ""

# Basic benchmark
echo "--- Basic Benchmark ---"
wrk -t$THREADS -c$CONNECTIONS -d$DURATION "$SERVER_URL/health"

echo ""
echo "--- With custom script ---"
# wrk Lua script
cat > /tmp/post_bench.lua << 'EOF'
wrk.method = "POST"
wrk.body   = '{"username":"bench_user","password":"secret123"}'
wrk.headers["Content-Type"] = "application/json"

response = function(status, headers, body)
  if status ~= 200 then
    print("Error: " .. status)
  end
end
EOF

wrk -t$THREADS -c$CONNECTIONS -d$DURATION -s /tmp/post_bench.lua \
    "$SERVER_URL/api/login"

echo ""
echo "--- Latency Distribution ---"
wrk -t$THREADS -c$CONNECTIONS -d$DURATION \
    --latency "$SERVER_URL/api/data"
```

```nim
# file: src/benchmark_server.nim
# Server ที่ optimize สำหรับ benchmark

import asyncdispatch, asynchttpserver, httpcore, strformat

# Pre-compute responses
const
  healthResponse = """{"status":"ok","version":"1.0.0"}"""
  errorResponse = """{"error":"not found"}"""

proc handleBenchmark(req: Request) {.async.} =
  let headers = newHttpHeaders([
    ("Content-Type", "application/json"),
    ("Server", "nim/2.0")
  ])
  
  case req.url.path
  of "/health":
    await req.respond(Http200, healthResponse, headers)
  of "/api/echo":
    await req.respond(Http200, req.body, headers)
  of "/api/json":
    # Minimal JSON construction
    let response = &"""{{\"id\":{rand(1000)},\"data\":\"value\"}}"""
    await req.respond(Http200, response, headers)
  else:
    await req.respond(Http404, errorResponse, headers)

proc main() {.async.} =
  var server = newAsyncHttpServer()
  echo "Benchmark server on :8080"
  await server.serve(Port(8080), handleBenchmark)

waitFor main()
```

---

## Step 277: Connection Pooling สำหรับ Database

```nim
# file: src/db_pool.nim
import asyncdispatch, db_postgres, strformat, times, tables

type
  DBConnection = ref object
    conn: DbConn
    id: int
    inUse: bool
    lastUsed: float
    queryCount: int

  DBPool = ref object
    connections: seq[DBConnection]
    maxSize: int
    url: string
    lock: bool
    waitQueue: seq[Future[DBConnection]]
    stats: PoolStats

  PoolStats = object
    totalQueries: int
    cacheHits: int
    totalWaitTime: float
    peakConnections: int

proc newDBPool(url: string, minSize = 5, maxSize = 20): Future[DBPool] {.async.} =
  let pool = DBPool(
    connections: @[],
    maxSize: maxSize,
    url: url,
    lock: false,
    waitQueue: @[]
  )
  
  # สร้าง minimum connections
  for i in 0..<minSize:
    let conn = open("", "", "", url)
    pool.connections.add(DBConnection(
      conn: conn,
      id: i,
      inUse: false,
      lastUsed: epochTime()
    ))
  
  return pool

proc acquire(pool: DBPool): Future[DBConnection] {.async.} =
  # หา available connection
  for dbconn in pool.connections:
    if not dbconn.inUse:
      dbconn.inUse = true
      dbconn.lastUsed = epochTime()
      pool.stats.peakConnections = max(
        pool.stats.peakConnections,
        pool.connections.countIt(it.inUse)
      )
      return dbconn
  
  # สร้าง connection ใหม่
  if pool.connections.len < pool.maxSize:
    let id = pool.connections.len
    let conn = open("", "", "", pool.url)
    let dbconn = DBConnection(
      conn: conn,
      id: id,
      inUse: true,
      lastUsed: epochTime()
    )
    pool.connections.add(dbconn)
    return dbconn
  
  # รอ connection ว่าง
  let waitStart = epochTime()
  var fut = newFuture[DBConnection]("pool.acquire")
  pool.waitQueue.add(fut)
  let result = await fut
  pool.stats.totalWaitTime += epochTime() - waitStart
  return result

proc release(pool: DBPool, dbconn: DBConnection) =
  dbconn.inUse = false
  dbconn.lastUsed = epochTime()
  
  # ให้ waiter ใน queue
  if pool.waitQueue.len > 0:
    let waiter = pool.waitQueue[0]
    pool.waitQueue.del(0)
    dbconn.inUse = true
    waiter.complete(dbconn)

template withConnection*(pool: DBPool, connName: untyped, body: untyped) =
  let connName = await pool.acquire()
  try:
    body
  finally:
    pool.release(connName)

proc query(pool: DBPool, sql: string, args: varargs[string, `$`]): Future[seq[Row]] {.async.} =
  withConnection(pool, dbconn):
    inc dbconn.queryCount
    inc pool.stats.totalQueries
    return dbconn.conn.getAllRows(SqlQuery(sql), args)

proc exec(pool: DBPool, sql: string, args: varargs[string, `$`]) {.async.} =
  withConnection(pool, dbconn):
    inc dbconn.queryCount
    inc pool.stats.totalQueries
    dbconn.conn.exec(SqlQuery(sql), args)

proc getPoolStats(pool: DBPool): JsonNode =
  let inUseCount = pool.connections.countIt(it.inUse)
  
  %*{
    "totalConnections": pool.connections.len,
    "inUse": inUseCount,
    "available": pool.connections.len - inUseCount,
    "waitQueueLength": pool.waitQueue.len,
    "totalQueries": pool.stats.totalQueries,
    "peakConnections": pool.stats.peakConnections,
    "avgWaitTimeMs": pool.stats.totalWaitTime * 1000 / max(1, pool.waitQueue.len).float
  }

proc cleanupIdleConnections(pool: DBPool, maxIdleSeconds = 300.0) {.async.} =
  ## ปิด connections ที่ idle นานเกินไป (แต่เก็บไว้ minimum)
  let minSize = 5
  let now = epochTime()
  
  var toRemove: seq[int] = @[]
  
  for i, conn in pool.connections:
    if not conn.inUse and 
       pool.connections.len - toRemove.len > minSize and
       now - conn.lastUsed > maxIdleSeconds:
      toRemove.add(i)
  
  # ลบ connections ที่ idle
  for idx in toRemove.reversed():
    pool.connections[idx].conn.close()
    pool.connections.del(idx)
  
  if toRemove.len > 0:
    echo &"[POOL] Closed {toRemove.len} idle connections"

proc startPoolMaintenance(pool: DBPool) {.async.} =
  while true:
    await sleepAsync(60_000)  # Every minute
    await pool.cleanupIdleConnections()

# ===== Usage Example =====

proc dbPoolDemo() {.async.} =
  let dbUrl = getEnv("DATABASE_URL", "postgres://user:pass@localhost/mydb")
  let pool = await newDBPool(dbUrl, minSize = 5, maxSize = 20)
  
  # Start maintenance
  asyncCheck startPoolMaintenance(pool)
  
  echo "Pool created, running queries..."
  
  # Concurrent queries
  var futs: seq[Future[void]] = @[]
  
  for i in 0..<50:
    let queryFut = proc() {.async.} =
      try:
        let rows = await pool.query("SELECT $1::int as num", $i)
        discard rows
      except Exception as e:
        echo &"Query error: {e.msg}"
    
    futs.add(queryFut())
  
  await all(futs)
  
  echo &"Stats: {pool.getPoolStats()}"
```

---

## Step 278: Caching Strategies

```nim
# file: src/caching_strategies.nim
import times, tables, hashes, strformat

# ===== LRU Cache =====

type
  LRUNode[K, V] = ref object
    key: K
    value: V
    prev, next: LRUNode[K, V]

  LRUCache[K, V] = ref object
    capacity: int
    map: Table[K, LRUNode[K, V]]
    head, tail: LRUNode[K, V]  # Dummy head and tail
    hits: int
    misses: int

proc newLRUCache*[K, V](capacity: int): LRUCache[K, V] =
  let cache = LRUCache[K, V](
    capacity: capacity,
    map: initTable[K, LRUNode[K, V]]()
  )
  
  # Dummy nodes
  cache.head = LRUNode[K, V]()
  cache.tail = LRUNode[K, V]()
  cache.head.next = cache.tail
  cache.tail.prev = cache.head
  
  return cache

proc removeNode[K, V](cache: LRUCache[K, V], node: LRUNode[K, V]) =
  node.prev.next = node.next
  node.next.prev = node.prev

proc addToFront[K, V](cache: LRUCache[K, V], node: LRUNode[K, V]) =
  node.next = cache.head.next
  node.prev = cache.head
  cache.head.next.prev = node
  cache.head.next = node

proc get*[K, V](cache: LRUCache[K, V], key: K): (bool, V) =
  if key in cache.map:
    let node = cache.map[key]
    # Move to front (most recently used)
    cache.removeNode(node)
    cache.addToFront(node)
    inc cache.hits
    return (true, node.value)
  
  inc cache.misses
  var zero: V
  return (false, zero)

proc put*[K, V](cache: LRUCache[K, V], key: K, value: V) =
  if key in cache.map:
    let node = cache.map[key]
    node.value = value
    cache.removeNode(node)
    cache.addToFront(node)
    return
  
  let node = LRUNode[K, V](key: key, value: value)
  cache.map[key] = node
  cache.addToFront(node)
  
  # Evict if over capacity
  if cache.map.len > cache.capacity:
    let lru = cache.tail.prev
    cache.removeNode(lru)
    cache.map.del(lru.key)

proc hitRate*[K, V](cache: LRUCache[K, V]): float =
  let total = cache.hits + cache.misses
  if total == 0: return 0.0
  return cache.hits.float / total.float * 100.0

# ===== TTL Cache =====

type
  TTLEntry[V] = object
    value: V
    expiresAt: float

  TTLCache[K, V] = ref object
    data: Table[K, TTLEntry[V]]
    defaultTtl: float

proc newTTLCache*[K, V](defaultTtlSeconds: float = 300): TTLCache[K, V] =
  TTLCache[K, V](
    data: initTable[K, TTLEntry[V]](),
    defaultTtl: defaultTtlSeconds
  )

proc set*[K, V](cache: TTLCache[K, V], key: K, value: V, ttl = -1.0) =
  let actualTtl = if ttl > 0: ttl else: cache.defaultTtl
  cache.data[key] = TTLEntry[V](
    value: value,
    expiresAt: epochTime() + actualTtl
  )

proc get*[K, V](cache: TTLCache[K, V], key: K): (bool, V) =
  if key in cache.data:
    let entry = cache.data[key]
    if epochTime() < entry.expiresAt:
      return (true, entry.value)
    cache.data.del(key)  # Expired
  
  var zero: V
  return (false, zero)

proc cleanup*[K, V](cache: TTLCache[K, V]) =
  let now = epochTime()
  var toDelete: seq[K] = @[]
  
  for key, entry in cache.data:
    if now >= entry.expiresAt:
      toDelete.add(key)
  
  for key in toDelete:
    cache.data.del(key)

# ===== Two-Level Cache =====

type
  TwoLevelCache[K, V] = ref object
    l1: LRUCache[K, V]  # Fast, small
    l2: TTLCache[K, V]  # Larger, with expiry

proc newTwoLevelCache*[K, V](l1Size: int, l2Ttl: float): TwoLevelCache[K, V] =
  TwoLevelCache[K, V](
    l1: newLRUCache[K, V](l1Size),
    l2: newTTLCache[K, V](l2Ttl)
  )

proc get*[K, V](cache: TwoLevelCache[K, V], key: K): (bool, V) =
  # Check L1 first
  let (found1, val1) = cache.l1.get(key)
  if found1: return (true, val1)
  
  # Check L2
  let (found2, val2) = cache.l2.get(key)
  if found2:
    # Promote to L1
    cache.l1.put(key, val2)
    return (true, val2)
  
  var zero: V
  return (false, zero)

proc put*[K, V](cache: TwoLevelCache[K, V], key: K, value: V) =
  cache.l1.put(key, value)
  cache.l2.set(key, value)

# ===== Demo =====

proc cachingDemo() =
  echo "=== LRU Cache Demo ==="
  
  var lru = newLRUCache[string, int](3)
  
  lru.put("a", 1)
  lru.put("b", 2)
  lru.put("c", 3)
  
  let (found, val) = lru.get("a")
  echo &"Get 'a': {found} = {val}"
  
  # Add 'd' - evicts 'b' (LRU)
  lru.put("d", 4)
  
  let (foundB, _) = lru.get("b")
  echo &"'b' still in cache: {foundB}"  # false (evicted)
  
  let (foundD, valD) = lru.get("d")
  echo &"'d' in cache: {foundD} = {valD}"  # true
  
  echo &"Hit rate: {lru.hitRate():.1f}%"
  
  echo "\n=== TTL Cache Demo ==="
  
  var ttl = newTTLCache[string, string](defaultTtlSeconds = 1.0)
  
  ttl.set("key1", "value1")
  ttl.set("key2", "value2", ttl = 0.1)  # Short TTL
  
  let (f1, v1) = ttl.get("key1")
  echo &"Immediate get: {f1} = {v1}"
  
  # Wait for short TTL to expire
  let start = epochTime()
  while epochTime() - start < 0.2: discard
  
  let (f2, _) = ttl.get("key2")
  echo &"After 200ms: key2 expired = {not f2}"
  
  let (f3, v3) = ttl.get("key1")
  echo &"key1 still valid: {f3} = {v3}"

cachingDemo()
```

---

## Step 279: Lock-Free Data Structures

```nim
# file: src/lockfree.nim
import atomics, strformat, times

# ===== Lock-free Counter =====

type
  AtomicCounter = ref object
    value: Atomic[int64]

proc newAtomicCounter(): AtomicCounter =
  AtomicCounter()

proc increment(c: AtomicCounter, amount = 1i64): int64 =
  return c.value.atomicAddFetch(amount, moRelaxed)

proc decrement(c: AtomicCounter, amount = 1i64): int64 =
  return c.value.atomicSubFetch(amount, moRelaxed)

proc load(c: AtomicCounter): int64 =
  return c.value.load(moRelaxed)

proc reset(c: AtomicCounter) =
  c.value.store(0, moRelaxed)

# ===== Metrics Collector =====

type
  Metrics = ref object
    requestTotal: Atomic[int64]
    requestErrors: Atomic[int64]
    requestLatencyNs: Atomic[int64]
    activeRequests: Atomic[int]
    byPath: array[16, Atomic[int64]]  # Fixed-size path counters

proc newMetrics(): Metrics =
  Metrics()

proc recordRequest(m: Metrics, latencyNs: int64, isError: bool, pathHash: int) =
  m.requestTotal.atomicInc()
  if isError:
    m.requestErrors.atomicInc()
  discard m.requestLatencyNs.atomicAddFetch(latencyNs)
  m.activeRequests.atomicInc()
  
  # Path-specific counter
  let idx = pathHash mod 16
  m.byPath[idx].atomicInc()

proc finishRequest(m: Metrics) =
  m.activeRequests.atomicDec()

proc snapshot(m: Metrics): JsonNode =
  let total = m.requestTotal.load()
  let errors = m.requestErrors.load()
  let latency = m.requestLatencyNs.load()
  
  %*{
    "total": total,
    "errors": errors,
    "active": m.activeRequests.load(),
    "avgLatencyNs": if total > 0: latency div total else: 0,
    "errorRate": if total > 0: errors.float / total.float else: 0.0
  }

# ===== Stack-based object pool =====

type
  ObjectPool[T] = ref object
    objects: seq[T]
    available: Atomic[int]
    capacity: int

proc newObjectPool[T](capacity: int, factory: proc(): T): ObjectPool[T] =
  var pool = ObjectPool[T](
    objects: newSeq[T](capacity),
    capacity: capacity
  )
  
  for i in 0..<capacity:
    pool.objects[i] = factory()
  
  pool.available.store(capacity)
  return pool

proc acquire[T](pool: ObjectPool[T]): ptr T =
  let idx = pool.available.atomicDecFetch() - 1
  if idx < 0:
    pool.available.atomicInc()
    return nil
  return addr pool.objects[idx]

proc release[T](pool: ObjectPool[T], obj: ptr T) =
  let idx = pool.available.atomicIncFetch() - 1
  if idx < pool.capacity:
    pool.objects[idx] = obj[]

proc lockFreeDemo() =
  echo "=== Lock-free Counter ==="
  
  let counter = newAtomicCounter()
  
  # Simulate concurrent increments
  var futs: seq[Future[void]] = @[]
  for i in 0..<1000:
    let val = counter.increment()
    if i mod 100 == 0:
      echo &"  Count at step {i}: {val}"
  
  echo &"Final count: {counter.load()}"
  
  echo "\n=== Metrics Demo ==="
  
  let metrics = newMetrics()
  
  for i in 0..<10000:
    let start = getMonoTime().ticks
    let isError = i mod 100 == 0  # 1% error rate
    let latency = getMonoTime().ticks - start
    metrics.recordRequest(latency, isError, i mod 16)
    metrics.finishRequest()
  
  echo &"Metrics: {metrics.snapshot()}"

lockFreeDemo()
```

---

## Step 280: Efficient JSON Processing

```nim
# file: src/json_performance.nim
import std/json, strformat, times, strutils

# ===== เปรียบเทียบ JSON parsing strategies =====

proc generateTestData(count: int): string =
  var arr = newJArray()
  for i in 0..<count:
    arr.add(%*{
      "id": i,
      "name": &"User {i}",
      "email": &"user{i}@example.com",
      "score": rand(1000),
      "active": i mod 3 != 0
    })
  return $arr

type
  User = object
    id: int
    name: string
    email: string
    score: int
    active: bool

proc parseWithStdlib(jsonStr: string): seq[User] =
  var users: seq[User] = @[]
  let data = parseJson(jsonStr)
  
  for item in data:
    users.add(User(
      id: item["id"].getInt(),
      name: item["name"].getStr(),
      email: item["email"].getStr(),
      score: item["score"].getInt(),
      active: item["active"].getBool()
    ))
  
  return users

# Fast JSON value extraction (avoid re-parsing)
proc extractField(json: JsonNode, field: string, default = ""): string =
  if json.hasKey(field):
    case json[field].kind
    of JString: return json[field].str
    of JInt: return $json[field].num
    of JBool: return $json[field].bval
    else: return default
  return default

# JSON serialization optimization
proc userToJson(u: User): string =
  # Manual serialization (faster than %* for simple types)
  &"""{{\"id\":{u.id},\"name\":\"{u.name}\",\"email\":\"{u.email}\",\"score\":{u.score},\"active\":{u.active}}}"""

proc jsonPerfDemo() =
  let count = 10_000
  let testData = generateTestData(count)
  echo &"Test data size: {testData.len} bytes"
  
  # Parse benchmark
  let parseStart = cpuTime()
  let users = parseWithStdlib(testData)
  echo &"Parse {count} users: {(cpuTime()-parseStart)*1000:.2f}ms"
  
  # Serialize benchmark
  let serStart = cpuTime()
  var parts = newSeqOfCap[string](users.len)
  for u in users:
    parts.add(userToJson(u))
  let serialized = "[" & parts.join(",") & "]"
  echo &"Serialize {count} users: {(cpuTime()-serStart)*1000:.2f}ms"
  echo &"Serialized size: {serialized.len} bytes"

jsonPerfDemo()
```

---

## Step 281: High-Performance Request Router

```nim
# file: src/fast_router.nim
import tables, strutils, sequtils, httpcore, asyncdispatch, asynchttpserver

type
  RouteHandler = proc(req: Request, params: Table[string, string]): Future[void] {.async.}

  RouteNode = ref object
    children: Table[string, RouteNode]
    paramChild: RouteNode  # For :param segments
    paramName: string
    handlers: Table[string, RouteHandler]  # method -> handler

  Router = ref object
    root: RouteNode
    notFound: RouteHandler
    methodNotAllowed: RouteHandler

proc newRouteNode(): RouteNode =
  RouteNode(
    children: initTable[string, RouteNode](),
    handlers: initTable[string, RouteHandler]()
  )

proc newRouter(): Router =
  Router(
    root: newRouteNode(),
    notFound: proc(req: Request, params: Table[string, string]) {.async.} =
      await req.respond(Http404, """{"error":"Not Found"}""",
        newHttpHeaders([("Content-Type", "application/json")])),
    methodNotAllowed: proc(req: Request, params: Table[string, string]) {.async.} =
      await req.respond(Http405, """{"error":"Method Not Allowed"}""",
        newHttpHeaders([("Content-Type", "application/json")]))
  )

proc addRoute(router: Router, `method`, path: string, handler: RouteHandler) =
  let parts = path.split("/").filter(proc(s: string): bool = s.len > 0)
  var node = router.root
  
  for part in parts:
    if part.startsWith(":"):
      # Parameter segment
      if node.paramChild.isNil:
        node.paramChild = newRouteNode()
        node.paramChild.paramName = part[1..^1]
      node = node.paramChild
    else:
      if part notin node.children:
        node.children[part] = newRouteNode()
      node = node.children[part]
  
  node.handlers[`method`.toUpperAscii()] = handler

proc matchRoute(router: Router, `method`, path: string): (RouteHandler, Table[string, string]) =
  let parts = path.split("/").filter(proc(s: string): bool = s.len > 0)
  var node = router.root
  var params = initTable[string, string]()
  
  for part in parts:
    if part in node.children:
      node = node.children[part]
    elif not node.paramChild.isNil:
      params[node.paramChild.paramName] = part
      node = node.paramChild
    else:
      return (router.notFound, params)
  
  let methodUpper = `method`.toUpperAscii()
  
  if methodUpper in node.handlers:
    return (node.handlers[methodUpper], params)
  elif node.handlers.len > 0:
    return (router.methodNotAllowed, params)
  else:
    return (router.notFound, params)

proc handle*(router: Router, req: Request) {.async.} =
  let (handler, params) = router.matchRoute($req.reqMethod, req.url.path)
  await handler(req, params)

# ===== Usage Example =====

proc buildApiRouter(): Router =
  let router = newRouter()
  
  router.addRoute("GET", "/health", proc(req: Request, params: Table[string, string]) {.async.} =
    await req.respond(Http200, """{"status":"ok"}""",
      newHttpHeaders([("Content-Type", "application/json")])))
  
  router.addRoute("GET", "/api/users", proc(req: Request, params: Table[string, string]) {.async.} =
    await req.respond(Http200, """{"users":[]}""",
      newHttpHeaders([("Content-Type", "application/json")])))
  
  router.addRoute("GET", "/api/users/:id", proc(req: Request, params: Table[string, string]) {.async.} =
    let userId = params.getOrDefault("id", "0")
    await req.respond(Http200, &"""{{\"id\":{userId}}}""",
      newHttpHeaders([("Content-Type", "application/json")])))
  
  router.addRoute("POST", "/api/users", proc(req: Request, params: Table[string, string]) {.async.} =
    await req.respond(Http201, """{"created":true}""",
      newHttpHeaders([("Content-Type", "application/json")])))
  
  return router

proc routerDemo() {.async.} =
  let router = buildApiRouter()
  var server = newAsyncHttpServer()
  
  proc handler(req: Request) {.async.} =
    await router.handle(req)
  
  echo "Fast router server on port 8080"
  await server.serve(Port(8080), handler)

waitFor routerDemo()
```

---

## Step 282: Zero-Copy Response Streaming

```nim
# file: src/streaming_response.nim
import asyncdispatch, asynchttpserver, asyncio, json, strformat, times

type
  StreamChunk = object
    data: string
    final: bool

proc streamSSE(req: Request, dataProducer: iterator(): string {.closure.}) {.async.} =
  ## Server-Sent Events (SSE) สำหรับ streaming
  let headers = newHttpHeaders([
    ("Content-Type", "text/event-stream"),
    ("Cache-Control", "no-cache"),
    ("Connection", "keep-alive"),
    ("X-Accel-Buffering", "no")
  ])
  
  # SSE requires we start response then keep sending
  await req.respond(Http200, "", headers)

proc generateLargeDataset(count: int): iterator(): string {.closure.} =
  return iterator(): string =
    for i in 0..<count:
      let item = %*{
        "id": i,
        "name": &"Item {i}",
        "value": rand(10000),
        "timestamp": epochTime()
      }
      
      # Yield JSON lines
      yield $item & "\n"
      
      if i mod 100 == 0:
        # Flush every 100 items
        yield ""

proc handleStreamingRequest(req: Request) {.async.} =
  let headers = newHttpHeaders([
    ("Content-Type", "application/x-ndjson"),  # Newline-delimited JSON
    ("Transfer-Encoding", "chunked")
  ])
  
  # ใน production ใช้ chunked transfer encoding จริงๆ
  var buffer = newStringOfCap(64 * 1024)  # 64KB buffer
  let producer = generateLargeDataset(1000)
  
  for chunk in producer():
    buffer.add(chunk)
    if buffer.len >= 8192:  # Flush every 8KB
      # ส่ง buffered data
      buffer.setLen(0)
  
  # Send remaining
  await req.respond(Http200, buffer, headers)

proc streamingDemo() {.async.} =
  var server = newAsyncHttpServer()
  
  proc handler(req: Request) {.async.} =
    if req.url.path == "/stream":
      await handleStreamingRequest(req)
    else:
      await req.respond(Http404, "Not found")
  
  echo "Streaming server on port 8080"
  await server.serve(Port(8080), handler)

waitFor streamingDemo()
```

---

## Step 283: Real-World: Optimized High-Performance API

```nim
# file: src/optimized_api.nim
# Production-ready high-performance API

import asyncdispatch, asynchttpserver, httpcore
import json, strformat, times, tables, atomics
import strutils, sequtils, algorithm, hashes

# ===== Compile with: =====
# nim c -d:release --opt:speed --gc:arc -d:ssl \
#        --passC:"-march=native" -d:lto \
#        -o:bin/api src/optimized_api.nim

type
  # Pre-computed responses
  StaticResponse = object
    body: string
    statusCode: HttpCode
    contentType: string

  RequestStats = ref object
    total: Atomic[int64]
    p50LatencyNs: Atomic[int64]
    p99LatencyNs: Atomic[int64]
    errors: Atomic[int64]

  Server = ref object
    stats: RequestStats
    cache: LRUCache[string, string]
    startTime: float

# ===== LRU Cache (inline for perf) =====

type
  LRUNode = ref object
    key: string
    value: string
    prev, next: LRUNode

  LRUCache = ref object
    capacity: int
    lookup: Table[string, LRUNode]
    head, tail: LRUNode

proc newLRUCache(cap: int): LRUCache =
  let cache = LRUCache(capacity: cap, lookup: initTable[string, LRUNode]())
  cache.head = LRUNode()
  cache.tail = LRUNode()
  cache.head.next = cache.tail
  cache.tail.prev = cache.head
  cache

proc moveToFront(cache: LRUCache, n: LRUNode) {.inline.} =
  n.prev.next = n.next
  n.next.prev = n.prev
  n.next = cache.head.next
  n.prev = cache.head
  cache.head.next.prev = n
  cache.head.next = n

proc get(cache: LRUCache, key: string): string =
  if key in cache.lookup:
    let n = cache.lookup[key]
    cache.moveToFront(n)
    return n.value
  return ""

proc put(cache: LRUCache, key, value: string) =
  if key in cache.lookup:
    let n = cache.lookup[key]
    n.value = value
    cache.moveToFront(n)
    return
  
  let n = LRUNode(key: key, value: value)
  cache.lookup[key] = n
  cache.moveToFront(n)
  
  if cache.lookup.len > cache.capacity:
    let lru = cache.tail.prev
    lru.prev.next = cache.tail
    cache.tail.prev = lru.prev
    cache.lookup.del(lru.key)

# ===== Response builder =====

var responseHeaders = newHttpHeaders([
  ("Content-Type", "application/json"),
  ("Connection", "keep-alive"),
  ("X-Content-Type-Options", "nosniff")
])

proc jsonResponse(req: Request, code: HttpCode, body: string) {.async, inline.} =
  await req.respond(code, body, responseHeaders)

# ===== Route handlers (optimized) =====

proc handleHealth(req: Request) {.async.} =
  await jsonResponse(req, Http200, """{"status":"ok","healthy":true}""")

proc handleMetrics(req: Request, server: Server) {.async.} =
  let total = server.stats.total.load()
  let errors = server.stats.errors.load()
  let uptime = epochTime() - server.startTime
  
  let response = &"""{{
    "requests":{{
      "total":{total},
      "errors":{errors},
      "successRate":{if total>0: (total-errors).float/total.float else: 1.0:.4f}
    }},
    "uptime":{uptime:.1f},
    "cacheSize":{server.cache.lookup.len}
  }}"""
  
  await jsonResponse(req, Http200, response)

proc handleUsers(req: Request, server: Server) {.async.} =
  # Check cache first
  let cacheKey = "users:list"
  let cached = server.cache.get(cacheKey)
  
  if cached.len > 0:
    responseHeaders["X-Cache"] = "HIT"
    await jsonResponse(req, Http200, cached)
    responseHeaders.del("X-Cache")
    return
  
  # Build response
  var parts = newSeqOfCap[string](100)
  for i in 1..100:
    parts.add(&"""{{\"id\":{i},\"name\":\"User {i}\"}}""")
  
  let response = "[" & parts.join(",") & "]"
  
  # Cache for 30 seconds
  server.cache.put(cacheKey, response)
  
  responseHeaders["X-Cache"] = "MISS"
  await jsonResponse(req, Http200, response)
  responseHeaders.del("X-Cache")

proc handleUserById(req: Request, userId: string) {.async.} =
  let id = try: parseInt(userId) except: 0
  
  if id <= 0:
    await jsonResponse(req, Http400, """{"error":"Invalid user ID"}""")
    return
  
  let response = &"""{{\"id\":{id},\"name\":\"User {id}\",\"active\":true}}"""
  await jsonResponse(req, Http200, response)

# ===== Main request dispatcher =====

proc dispatch(req: Request, server: Server) {.async.} =
  let startNs = getMonoTime().ticks
  server.stats.total.atomicInc()
  
  var success = true
  
  try:
    # Fast path matching
    let path = req.url.path
    
    if path.len == 7 and path == "/health":
      await handleHealth(req)
    
    elif path.len == 8 and path == "/metrics":
      await handleMetrics(req, server)
    
    elif path.len == 11 and path == "/api/users":
      await handleUsers(req, server)
    
    elif path.startsWith("/api/users/"):
      await handleUserById(req, path[11..^1])
    
    else:
      success = false
      await jsonResponse(req, Http404, """{"error":"Not found"}""")
  
  except Exception as e:
    success = false
    server.stats.errors.atomicInc()
    await jsonResponse(req, Http500, """{"error":"Internal Server Error"}""")
  
  if not success:
    server.stats.errors.atomicInc()
  
  let elapsed = getMonoTime().ticks - startNs

proc main() {.async.} =
  let server = Server(
    stats: RequestStats(),
    cache: newLRUCache(1000),
    startTime: epochTime()
  )
  
  var httpServer = newAsyncHttpServer()
  
  proc handler(req: Request) {.async.} =
    await dispatch(req, server)
  
  let port = parseInt(getEnv("PORT", "8080"))
  echo &"High-performance API server on port {port}"
  echo "Build: nim c -d:release --opt:speed --gc:arc"
  
  await httpServer.serve(Port(port), handler)

waitFor main()
```

---

## Step 284: wrk Benchmark Script

```lua
-- file: benchmarks/api_test.lua
-- ใช้กับ: wrk -t4 -c100 -d30s -s benchmarks/api_test.lua http://localhost:8080

local counter = 0
local paths = {
  "/health",
  "/api/users",
  "/api/users/1",
  "/api/users/42",
  "/api/users/100",
  "/metrics"
}

function request()
  counter = counter + 1
  local path = paths[(counter % #paths) + 1]
  return wrk.format("GET", path)
end

local ok = 0
local errors = 0

function response(status, headers, body)
  if status >= 200 and status < 400 then
    ok = ok + 1
  else
    errors = errors + 1
  end
end

function done(summary, latency, requests)
  io.write("\n=== Custom Report ===\n")
  io.write(string.format("Successful requests: %d\n", ok))
  io.write(string.format("Error requests: %d\n", errors))
  io.write(string.format("Requests/sec: %.2f\n", requests.rate))
  io.write(string.format("Latency P50: %.2fms\n", latency:percentile(50) / 1000))
  io.write(string.format("Latency P99: %.2fms\n", latency:percentile(99) / 1000))
  io.write(string.format("Latency P999: %.2fms\n", latency:percentile(99.9) / 1000))
end
```

```bash
# file: benchmarks/run_benchmark.sh
#!/bin/bash

echo "Building server..."
nim c -d:release --opt:speed --gc:arc \
    -o:bin/api_server src/optimized_api.nim

echo "Starting server..."
./bin/api_server &
SERVER_PID=$!
sleep 2

echo "Running benchmarks..."

# Warm up
echo "--- Warmup ---"
wrk -t2 -c10 -d5s http://localhost:8080/health > /dev/null

# Actual benchmarks
echo ""
echo "=== Benchmark: /health ==="
wrk -t4 -c100 -d30s http://localhost:8080/health

echo ""
echo "=== Benchmark: /api/users (with cache) ==="
wrk -t4 -c100 -d30s http://localhost:8080/api/users

echo ""
echo "=== Benchmark: Mixed endpoints ==="
wrk -t4 -c100 -d30s -s benchmarks/api_test.lua http://localhost:8080

echo ""
echo "=== Benchmark: High concurrency ==="
wrk -t8 -c500 -d30s http://localhost:8080/health

# Cleanup
kill $SERVER_PID
echo ""
echo "Benchmark complete!"
```

---

## Step 285: Performance Checklist และ Summary

```nim
# file: src/performance_checklist.nim
# สรุป best practices สำหรับ Nim backend performance

# ===== 1. Compile Flags =====
# Production:
#   nim c -d:release -d:danger --opt:speed --gc:arc -d:ssl
#         --passC:"-march=native" -d:lto
# Development:
#   nim c --debuginfo --lineDir:on

# ===== 2. Memory Management =====
# - ใช้ --gc:arc หรือ --mm:orc (deterministic, no pauses)
# - Pre-allocate sequences: newSeqOfCap
# - ใช้ value types (object) แทน ref object เมื่อเป็นไปได้
# - Avoid unnecessary allocations ใน hot path

# ===== 3. String Performance =====
# - ใช้ newStringOfCap() เมื่อรู้ขนาดประมาณ
# - ใช้ add() แทน & สำหรับ incremental building
# - ใช้ join() แทน concatenation loop

# ===== 4. Table Performance =====
# - initTable[K, V]() สำหรับ hash table
# - getOrDefault() สำหรับ safe lookup
# - ตรวจสอบ key ด้วย in ก่อน access

# ===== 5. Async Performance =====
# - ใช้ asyncCheck สำหรับ fire-and-forget
# - ใช้ all() สำหรับ parallel operations
# - ใช้ waitFor เฉพาะใน main
# - ระวัง blocking calls ใน async context

# ===== 6. HTTP Server =====
# - Pre-compute static responses
# - ใช้ connection pooling สำหรับ DB
# - Cache frequently-accessed data (LRU)
# - ใช้ keep-alive connections

# ===== 7. Database =====
# - Connection pooling (min: 5, max: 20)
# - Prepare statements
# - Batch operations เมื่อเป็นไปได้
# - Index ที่ใช้บ่อย

# ===== 8. Profiling =====
# - ใช้ --profiler:on เพื่อ profiling
# - ใช้ std/monotimes สำหรับ accurate timing
# - Benchmark ด้วย wrk/hey/ab

# ===== 9. Caching =====
# - LRU cache สำหรับ frequently-accessed data
# - TTL cache สำหรับ external data
# - Redis สำหรับ shared cache
# - Response caching สำหรับ static content

# ===== 10. Monitoring =====
# - Track request latency (P50, P99, P999)
# - Monitor error rates
# - Track GC pauses (กับ gc:refc)
# - Resource utilization (CPU, Memory)

proc demonstratePerformanceTips() =
  import times, strformat
  
  echo "=== Performance Best Practices Demo ==="
  
  # Tip 1: Pre-allocated sequences
  let start1 = cpuTime()
  var slow: seq[int] = @[]
  for i in 0..<100_000:
    slow.add(i)
  echo &"Without pre-alloc: {(cpuTime()-start1)*1000:.2f}ms"
  
  let start2 = cpuTime()
  var fast = newSeqOfCap[int](100_000)
  for i in 0..<100_000:
    fast.add(i)
  echo &"With pre-alloc: {(cpuTime()-start2)*1000:.2f}ms"
  
  # Tip 2: String building
  let start3 = cpuTime()
  var s = newStringOfCap(100_000 * 5)
  for i in 0..<100_000:
    s.add($i)
    s.add(',')
  echo &"Efficient string build: {(cpuTime()-start3)*1000:.2f}ms, len={s.len}"
  
  # Tip 3: Table vs linear search
  import tables
  
  var lookup = initTable[int, string]()
  for i in 0..<10_000:
    lookup[i] = &"value_{i}"
  
  let start4 = cpuTime()
  for i in 0..<100_000:
    discard lookup.getOrDefault(i mod 10_000, "")
  echo &"Table lookup: {(cpuTime()-start4)*1000:.2f}ms"
  
  echo "\n=== Summary ==="
  echo "Key metrics for high-performance Nim backend:"
  echo "  - Target: 10,000-100,000+ req/sec"
  echo "  - P99 latency: < 10ms"
  echo "  - Memory: < 50MB steady state"
  echo "  - GC pauses: 0 (with arc/orc)"

demonstratePerformanceTips()
```

---

## 📝 สรุป Part 20

| Step | หัวข้อ | สิ่งที่เรียนรู้ |
|------|--------|----------------|
| 271 | Profiling | cpuTime, manual timing, allocation patterns |
| 272 | Monotimes | High-precision timing, BenchmarkResult |
| 273 | Memory Management | GC modes, ARC/ORC, allocation patterns |
| 274 | Compile Optimizations | -d:release, --opt:speed, pragmas |
| 275 | Async Performance | Connection pooling, request batching |
| 276 | wrk Benchmarking | HTTP load testing, Lua scripts |
| 277 | DB Connection Pool | Pool implementation, maintenance |
| 278 | Caching Strategies | LRU, TTL, Two-level cache |
| 279 | Lock-Free | Atomic counters, object pools |
| 280 | JSON Performance | Fast parsing, manual serialization |
| 281 | Fast Router | Trie-based routing, O(k) lookup |
| 282 | Streaming | SSE, chunked transfer, zero-copy |
| 283 | Optimized API | Production-ready high-performance server |
| 284 | Benchmark Scripts | wrk Lua scripts, CI benchmarks |
| 285 | Performance Checklist | All optimization strategies |

---

### Performance Targets สำหรับ Nim Backend

| Metric | Good | Excellent | World-class |
|--------|------|-----------|-------------|
| Req/sec | 10,000 | 50,000 | 100,000+ |
| P50 Latency | < 5ms | < 2ms | < 1ms |
| P99 Latency | < 20ms | < 10ms | < 5ms |
| Memory | < 100MB | < 50MB | < 20MB |
| GC Pauses | < 10ms | 0ms (ARC) | 0ms (ARC) |

---

## Navigation

- [← Part 19: Testing](part_19_testing.md)
- [กลับ README](../README.md)
