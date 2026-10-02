# Part 62: Performance Profiling & Optimization
## Steps 901-915: Making Nim Backend Code Fast

---

## 🎯 เป้าหมายของ Part นี้

- CPU profiling (sampling profiler)
- Memory profiling & allocation tracking
- Benchmarking framework
- Lock-free data structures
- Memory pool allocator
- SIMD-like vectorized operations (pure Nim)
- Async I/O optimization

---

## Step 901: Benchmarking Framework

```nim
import times, math, strformat, sequtils, algorithm, tables, strutils

# ============================
# Benchmark types
# ============================

type
  BenchResult = object
    name: string
    iterations: int
    totalNs: float
    minNs: float
    maxNs: float
    meanNs: float
    stddevNs: float
    p50Ns: float
    p95Ns: float
    p99Ns: float

  BenchSuite = object
    name: string
    results: seq[BenchResult]

proc measure(fn: proc()): float =
  let t = cpuTime()
  fn()
  return (cpuTime() - t) * 1e9  # nanoseconds

proc bench(name: string, iterations: int, fn: proc()): BenchResult =
  # Warmup
  for _ in 0..<min(10, iterations div 10):
    fn()

  # Collect samples
  var samples = newSeq[float](iterations)
  for i in 0..<iterations:
    samples[i] = measure(fn)

  samples.sort()

  let total = samples.foldl(a + b, 0.0)
  let mean = total / float(iterations)

  var variance = 0.0
  for s in samples:
    variance += (s - mean) * (s - mean)
  variance /= float(iterations)

  let p50idx = int(float(iterations) * 0.50)
  let p95idx = int(float(iterations) * 0.95)
  let p99idx = int(float(iterations) * 0.99)

  BenchResult(
    name: name,
    iterations: iterations,
    totalNs: total,
    minNs: samples[0],
    maxNs: samples[^1],
    meanNs: mean,
    stddevNs: sqrt(variance),
    p50Ns: samples[min(p50idx, iterations-1)],
    p95Ns: samples[min(p95idx, iterations-1)],
    p99Ns: samples[min(p99idx, iterations-1)]
  )

proc printBench(r: BenchResult) =
  echo fmt"{r.name}:"
  echo fmt"  iterations: {r.iterations}"
  echo fmt"  mean:  {r.meanNs:>8.1f} ns"
  echo fmt"  min:   {r.minNs:>8.1f} ns"
  echo fmt"  max:   {r.maxNs:>8.1f} ns"
  echo fmt"  p50:   {r.p50Ns:>8.1f} ns"
  echo fmt"  p95:   {r.p95Ns:>8.1f} ns"
  echo fmt"  p99:   {r.p99Ns:>8.1f} ns"
  echo fmt"  ops/s: {1e9 / r.meanNs:>8.0f}"

proc compareBench(a, b: BenchResult) =
  let ratio = a.meanNs / b.meanNs
  let faster = if ratio > 1.0: b.name else: a.name
  let slower = if ratio > 1.0: a.name else: b.name
  let speedup = if ratio > 1.0: ratio else: 1.0 / ratio
  echo fmt"  {faster} is {speedup:.2f}x faster than {slower}"

# ============================
# Memory pool allocator
# ============================

type
  PoolBlock[T] = object
    data: T
    next: int   # index of next free, -1 if end

  MemPool[T] = object
    blocks: seq[PoolBlock[T]]
    freeHead: int
    capacity: int
    allocCount: int
    freeCount: int

proc newMemPool[T](capacity: int): MemPool[T] =
  var pool = MemPool[T](
    blocks: newSeq[PoolBlock[T]](capacity),
    freeHead: 0,
    capacity: capacity
  )
  # Build free list
  for i in 0..<capacity - 1:
    pool.blocks[i].next = i + 1
  pool.blocks[capacity - 1].next = -1
  return pool

proc poolAlloc[T](pool: var MemPool[T]): int =
  if pool.freeHead == -1:
    raise newException(OutOfMemDefect, "Pool exhausted")
  let idx = pool.freeHead
  pool.freeHead = pool.blocks[idx].next
  inc pool.allocCount
  return idx

proc poolFree[T](pool: var MemPool[T], idx: int) =
  pool.blocks[idx].next = pool.freeHead
  pool.freeHead = idx
  inc pool.freeCount

proc poolGet[T](pool: var MemPool[T], idx: int): ptr T =
  addr pool.blocks[idx].data

proc poolStats[T](pool: MemPool[T]) =
  let inUse = pool.allocCount - pool.freeCount
  echo fmt"Pool stats: capacity={pool.capacity} alloc={pool.allocCount} free={pool.freeCount} in_use={inUse}"

# ============================
# Lock-free ring buffer (single producer, single consumer)
# ============================

type
  RingBuffer[T] = object
    data: seq[T]
    capacity: int
    readIdx: int
    writeIdx: int

proc newRingBuffer[T](capacity: int): RingBuffer[T] =
  RingBuffer[T](data: newSeq[T](capacity), capacity: capacity)

proc rbPush[T](rb: var RingBuffer[T], item: T): bool =
  let nextWrite = (rb.writeIdx + 1) mod rb.capacity
  if nextWrite == rb.readIdx: return false  # full
  rb.data[rb.writeIdx] = item
  rb.writeIdx = nextWrite
  return true

proc rbPop[T](rb: var RingBuffer[T]): tuple[ok: bool, item: T] =
  if rb.readIdx == rb.writeIdx: return (false, default(T))  # empty
  let item = rb.data[rb.readIdx]
  rb.readIdx = (rb.readIdx + 1) mod rb.capacity
  return (true, item)

proc rbLen[T](rb: RingBuffer[T]): int =
  if rb.writeIdx >= rb.readIdx:
    rb.writeIdx - rb.readIdx
  else:
    rb.capacity - rb.readIdx + rb.writeIdx

# ============================
# String interning (reduce allocations)
# ============================

type
  StringInterner = object
    table: Table[string, int]
    strings: seq[string]

proc newStringInterner(): StringInterner =
  StringInterner(table: initTable[string, int](), strings: @[])

proc intern(si: var StringInterner, s: string): int =
  if s in si.table:
    return si.table[s]
  let id = si.strings.len
  si.strings.add(s)
  si.table[s] = id
  return id

proc resolve(si: StringInterner, id: int): string =
  si.strings[id]

proc internerStats(si: StringInterner) =
  var totalBytes = 0
  for s in si.strings: totalBytes += s.len
  echo fmt"Interner: {si.strings.len} unique strings, {totalBytes} bytes"

# ============================
# Vectorized sum (loop unrolling)
# ============================

proc sumNaive(data: seq[float]): float =
  for x in data: result += x

proc sumUnrolled(data: seq[float]): float =
  let n = data.len
  let main = n - (n mod 4)
  var s0, s1, s2, s3 = 0.0
  var i = 0
  while i < main:
    s0 += data[i]
    s1 += data[i+1]
    s2 += data[i+2]
    s3 += data[i+3]
    i += 4
  result = s0 + s1 + s2 + s3
  for j in main..<n:
    result += data[j]

proc dotProductUnrolled(a, b: seq[float]): float =
  assert a.len == b.len
  let n = a.len
  let main = n - (n mod 4)
  var s0, s1, s2, s3 = 0.0
  var i = 0
  while i < main:
    s0 += a[i] * b[i]
    s1 += a[i+1] * b[i+1]
    s2 += a[i+2] * b[i+2]
    s3 += a[i+3] * b[i+3]
    i += 4
  result = s0 + s1 + s2 + s3
  for j in main..<n:
    result += a[j] * b[j]

# ============================
# Simple allocation tracker
# ============================

type
  AllocEvent = object
    size: int
    label: string
    timestamp: float
    freed: bool

  AllocTracker = object
    events: seq[AllocEvent]
    totalAllocs: int
    totalFrees: int
    totalBytes: int
    peakBytes: int
    currentBytes: int

var tracker = AllocTracker()

proc trackAlloc(label: string, size: int) =
  inc tracker.totalAllocs
  tracker.totalBytes += size
  tracker.currentBytes += size
  if tracker.currentBytes > tracker.peakBytes:
    tracker.peakBytes = tracker.currentBytes
  tracker.events.add(AllocEvent(
    size: size,
    label: label,
    timestamp: cpuTime(),
    freed: false
  ))

proc trackFree(label: string, size: int) =
  inc tracker.totalFrees
  tracker.currentBytes -= size
  for i, e in tracker.events:
    if e.label == label and not e.freed:
      tracker.events[i].freed = true
      break

proc allocReport() =
  echo fmt"Alloc report:"
  echo fmt"  total allocs: {tracker.totalAllocs}"
  echo fmt"  total bytes:  {tracker.totalBytes}"
  echo fmt"  peak bytes:   {tracker.peakBytes}"
  echo fmt"  current:      {tracker.currentBytes}"
  let leaked = tracker.events.filterIt(not it.freed)
  if leaked.len > 0:
    echo fmt"  LEAKS: {leaked.len}"
    for e in leaked:
      echo fmt"    {e.label}: {e.size} bytes"

# ============================
# Demo
# ============================

proc demo() =
  echo "=== Performance & Optimization Demo ==="

  # Benchmarks
  echo "\n--- Benchmarks ---"

  let data = newSeqWith(10000, float(rand(100)))

  let naiveBench = bench("sum_naive", 1000, proc() =
    discard sumNaive(data)
  )
  let unrolledBench = bench("sum_unrolled", 1000, proc() =
    discard sumUnrolled(data)
  )

  printBench(naiveBench)
  printBench(unrolledBench)
  compareBench(naiveBench, unrolledBench)

  # Memory pool
  echo "\n--- Memory pool ---"
  type Node = object
    value: int
    score: float

  var pool = newMemPool[Node](100)
  let idx1 = poolAlloc(pool)
  let idx2 = poolAlloc(pool)
  poolGet(pool, idx1).value = 42
  poolGet(pool, idx2).value = 99
  poolFree(pool, idx1)
  let idx3 = poolAlloc(pool)  # should reuse idx1
  echo fmt"Reused index: {idx3 == idx1}"
  poolStats(pool)

  # Ring buffer
  echo "\n--- Ring buffer ---"
  var rb = newRingBuffer[int](8)
  for i in 1..5:
    discard rbPush(rb, i * 10)

  echo fmt"Buffer size: {rbLen(rb)}"
  while true:
    let (ok, val) = rbPop(rb)
    if not ok: break
    echo fmt"  popped: {val}"

  # String interning
  echo "\n--- String interning ---"
  var si = newStringInterner()
  let id1 = si.intern("application/json")
  let id2 = si.intern("text/html")
  let id3 = si.intern("application/json")  # duplicate
  echo fmt"Same id for duplicate: {id1 == id3}"
  echo fmt"Resolved: {si.resolve(id2)}"
  internerStats(si)

  # Allocation tracking
  echo "\n--- Allocation tracking ---"
  trackAlloc("user_cache", 4096)
  trackAlloc("session_pool", 1024)
  trackAlloc("request_buf", 512)
  trackFree("request_buf", 512)
  allocReport()

demo()
```

---

## 📝 สรุป Part 62

| Steps | หัวข้อ |
|-------|--------|
| 901 | Benchmarking framework with percentiles |
| 902-910 | Memory pool, ring buffer, string interning |
| 911-915 | Loop unrolling, allocation tracking |

---

**← [Part 61: Microservices](part_61_microservices.md) | [Part 63: CQRS/Event Sourcing →](part_63_cqrs.md)**
