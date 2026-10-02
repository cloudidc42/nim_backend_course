# Part 12: Generics & Templates
## Steps 146-160: การเขียน Generic Code

---

## 🎯 เป้าหมายของ Part นี้

- ใช้ Generic types
- สร้าง Generic procedures
- ใช้ Concepts (type constraints)
- ใช้ Templates และ Macros
- Generic data structures

---

## Step 146: Generic Types

```nim
import std/strformat

# Generic container
type
  Box[T] = object
    value: T
    label: string

proc newBox[T](value: T, label: string = ""): Box[T] =
  Box[T](value: value, label: label)

proc get[T](box: Box[T]): T = box.value
proc set[T](box: var Box[T], value: T) = box.value = value

# Usage with different types
var intBox = newBox(42, "answer")
var strBox = newBox("Hello", "greeting")
var floatBox = newBox(3.14)

echo intBox.get()    # 42
echo strBox.get()    # Hello
echo floatBox.get()  # 3.14

# Generic pair
type
  Pair[A, B] = object
    first: A
    second: B

proc newPair[A, B](a: A, b: B): Pair[A, B] =
  Pair[A, B](first: a, second: b)

proc swap[A, B](p: Pair[A, B]): Pair[B, A] =
  newPair(p.second, p.first)

var p = newPair("name", 42)
var swapped = p.swap()
echo swapped.first   # 42
echo swapped.second  # name

# Generic Stack
type
  Stack[T] = object
    data: seq[T]

proc push[T](s: var Stack[T], item: T) =
  s.data.add(item)

proc pop[T](s: var Stack[T]): T =
  if s.data.len == 0:
    raise newException(IndexDefect, "Stack is empty")
  result = s.data[^1]
  s.data.setLen(s.data.len - 1)

proc peek[T](s: Stack[T]): T =
  if s.data.len == 0:
    raise newException(IndexDefect, "Stack is empty")
  s.data[^1]

proc isEmpty[T](s: Stack[T]): bool = s.data.len == 0
proc size[T](s: Stack[T]): int = s.data.len

var stack = Stack[int]()
stack.push(1)
stack.push(2)
stack.push(3)
echo stack.peek()  # 3
echo stack.pop()   # 3
echo stack.size()  # 2
```

---

## Step 147: Concepts - Type Constraints

```nim
import std/strformat

# Concept definition
type
  Printable = concept x
    $x is string           # must have $ operator returning string

  Comparable[T] = concept x, y
    x < y is bool          # must support < operator
    x == y is bool         # must support == operator

  Container[T] = concept c
    c.len() is int         # must have len()
    c.add(T)               # must have add(T)

  Serializable = concept s
    s.toJson() is string   # must have toJson()

# Procedures using concepts
proc printAll[T: Printable](items: seq[T]) =
  for item in items:
    echo $item

proc findMin[T: Comparable[T]](items: seq[T]): T =
  result = items[0]
  for item in items[1..^1]:
    if item < result:
      result = item

# Usage
printAll(@[1, 2, 3])           # works (int has $)
printAll(@["a", "b", "c"])     # works (string has $)

echo findMin(@[5, 2, 8, 1, 9]) # 1
echo findMin(@["banana", "apple", "cherry"]) # apple

# Custom type with concept
type
  Temperature = object
    celsius: float

proc `$`(t: Temperature): string =
  fmt"{t.celsius:.1f}°C"

proc `<`(a, b: Temperature): bool = a.celsius < b.celsius
proc `==`(a, b: Temperature): bool = a.celsius == b.celsius

var temps = @[
  Temperature(celsius: 25.0),
  Temperature(celsius: 18.5),
  Temperature(celsius: 32.0),
  Temperature(celsius: 10.0),
]

printAll(temps)
echo "Min temp: " & $findMin(temps)  # 10.0°C
```

---

## Step 148: Generic Algorithms

```nim
import std/sequtils, std/algorithm

# Generic binary search
proc binarySearch[T: Comparable[T]](
  sorted: seq[T], target: T
): int =
  var lo = 0
  var hi = sorted.len - 1
  
  while lo <= hi:
    let mid = (lo + hi) div 2
    if sorted[mid] == target:
      return mid
    elif sorted[mid] < target:
      lo = mid + 1
    else:
      hi = mid - 1
  
  return -1  # not found

let nums = @[1, 3, 5, 7, 9, 11, 13, 15]
echo binarySearch(nums, 7)   # 3 (index)
echo binarySearch(nums, 8)   # -1 (not found)

# Generic map with transformation
proc transform[T, U](items: seq[T], fn: proc(item: T): U): seq[U] =
  result = newSeq[U](items.len)
  for i, item in items:
    result[i] = fn(item)

echo transform(@[1, 2, 3, 4], proc(x: int): string = "N" & $x)
# @["N1", "N2", "N3", "N4"]

# Generic group by
proc groupBy[T, K](items: seq[T], keyFn: proc(item: T): K): Table[K, seq[T]] =
  result = initTable[K, seq[T]]()
  for item in items:
    let key = keyFn(item)
    if key notin result:
      result[key] = @[]
    result[key].add(item)

type Person = object
  name: string
  dept: string
  salary: int

let employees = @[
  Person(name: "Alice", dept: "Engineering", salary: 90000),
  Person(name: "Bob", dept: "Marketing", salary: 70000),
  Person(name: "Charlie", dept: "Engineering", salary: 85000),
  Person(name: "Diana", dept: "Marketing", salary: 75000),
  Person(name: "Eve", dept: "Engineering", salary: 95000),
]

let byDept = groupBy(employees, proc(p: Person): string = p.dept)

for dept, people in byDept:
  let names = people.mapIt(it.name)
  let avgSalary = people.foldl(a + b.salary, 0) div people.len
  echo fmt"{dept}: {names.join(\", \")} (avg salary: {avgSalary})"
```

---

## Step 149: Templates

```nim
import std/strformat, std/times

# Simple template (code substitution)
template debug(msg: string) =
  when defined(debug):
    echo "[DEBUG] " & msg

template benchmark(name: string, body: untyped): float =
  let start = cpuTime()
  body
  let elapsed = cpuTime() - start
  echo fmt"{name}: {elapsed:.4f}s"
  elapsed

# Usage
debug("This only prints in debug mode")

let time = benchmark("Fibonacci 35"):
  proc fib(n: int): int =
    if n <= 1: n
    else: fib(n-1) + fib(n-2)
  echo fib(35)

# Template for retry logic
template retry(attempts: int, body: untyped): bool =
  var success = false
  for i in 1..attempts:
    try:
      body
      success = true
      break
    except CatchableError as e:
      if i < attempts:
        echo fmt"Attempt {i} failed: {e.msg}, retrying..."
      else:
        echo fmt"All {attempts} attempts failed: {e.msg}"
  success

let ok = retry(3):
  # Simulate flaky operation
  import std/random
  if rand(2) != 0:
    raise newException(IOError, "Connection failed")
  echo "Connected!"

echo "Connected: " & $ok

# Template for timed cache
template cached(key: string, ttl: int, body: untyped): untyped =
  import std/tables, std/times
  
  type CacheEntry = object
    value: type(body)
    expires: float
  
  var cache {.global.}: Table[string, CacheEntry]
  
  let now2 = epochTime()
  if key in cache and cache[key].expires > now2:
    cache[key].value
  else:
    let result = body
    cache[key] = CacheEntry(value: result, expires: now2 + float(ttl))
    result
```

---

## Step 150-160: Generic Data Structures สำหรับ Backend

```nim
# generic_cache.nim - Production-ready generic cache

import std/tables, std/times, std/options, std/sequtils, std/algorithm

type
  CacheEntry[V] = object
    value: V
    expiresAt: float
    hitCount: int
    createdAt: float

  Cache[K, V] = object
    entries: Table[K, CacheEntry[V]]
    maxSize: int
    defaultTtl: int
    hits: int
    misses: int

proc newCache[K, V](maxSize: int = 1000, defaultTtl: int = 300): Cache[K, V] =
  Cache[K, V](
    entries: initTable[K, CacheEntry[V]](),
    maxSize: maxSize,
    defaultTtl: defaultTtl,
    hits: 0,
    misses: 0
  )

proc evictExpired[K, V](cache: var Cache[K, V]) =
  let now = epochTime()
  var toDelete: seq[K] = @[]
  
  for key, entry in cache.entries:
    if entry.expiresAt < now:
      toDelete.add(key)
  
  for key in toDelete:
    cache.entries.del(key)

proc set[K, V](cache: var Cache[K, V], key: K, value: V, ttl: int = -1) =
  cache.evictExpired()
  
  # Evict LFU (least frequently used) if at capacity
  if cache.entries.len >= cache.maxSize and key notin cache.entries:
    var minHits = high(int)
    var minKey: K
    var first = true
    
    for k, entry in cache.entries:
      if first or entry.hitCount < minHits:
        minHits = entry.hitCount
        minKey = k
        first = false
    
    if not first:
      cache.entries.del(minKey)
  
  let actualTtl = if ttl < 0: cache.defaultTtl else: ttl
  let now = epochTime()
  
  cache.entries[key] = CacheEntry[V](
    value: value,
    expiresAt: now + float(actualTtl),
    hitCount: 0,
    createdAt: now
  )

proc get[K, V](cache: var Cache[K, V], key: K): Option[V] =
  cache.evictExpired()
  
  if key in cache.entries:
    let now = epochTime()
    if cache.entries[key].expiresAt > now:
      cache.entries[key].hitCount += 1
      inc cache.hits
      return some(cache.entries[key].value)
    else:
      cache.entries.del(key)
  
  inc cache.misses
  return none(V)

proc getOrSet[K, V](cache: var Cache[K, V], key: K, factory: proc(): V, ttl: int = -1): V =
  let cached = cache.get(key)
  if cached.isSome:
    return cached.get()
  
  let value = factory()
  cache.set(key, value, ttl)
  return value

proc hitRate[K, V](cache: Cache[K, V]): float =
  let total = cache.hits + cache.misses
  if total == 0: return 0.0
  float(cache.hits) / float(total)

# Generic Repository
type
  Repository[T; ID] = object
    items: Table[ID, T]
    nextId: int

proc newRepository[T; ID](): Repository[T, ID] =
  Repository[T, ID](items: initTable[ID, T](), nextId: 1)

# Test
var userCache = newCache[string, string](maxSize: 100, defaultTtl: 60)

userCache.set("user:1", """{"id":1,"name":"Alice"}""")
userCache.set("user:2", """{"id":2,"name":"Bob"}""")

echo userCache.get("user:1")   # some(...)
echo userCache.get("user:99")  # none

echo fmt"Hit rate: {userCache.hitRate():.1%}"  # 50.0%

# Cached function call
var computeCache = newCache[string, int](maxSize: 1000, defaultTtl: 3600)

proc expensiveComputation(n: int): int =
  # Simulate slow computation
  var result = 0
  for i in 1..n:
    result += i
  result

proc cachedCompute(n: int): int =
  computeCache.getOrSet("sum:" & $n, proc(): int = expensiveComputation(n))

echo cachedCompute(100)   # 5050
echo cachedCompute(100)   # 5050 (from cache)
echo computeCache.hitRate()  # 50%
```

---

## 📝 สรุป Part 12

| Steps | หัวข้อ |
|-------|--------|
| 146 | Generic types (Box, Stack, Pair) |
| 147 | Concepts - type constraints |
| 148 | Generic algorithms |
| 149 | Templates |
| 150-160 | Generic data structures สำหรับ backend |

---

**← [Part 11: OOP](part_11_oop.md) | [Part 13: Async/Await →](part_13_async.md)**
