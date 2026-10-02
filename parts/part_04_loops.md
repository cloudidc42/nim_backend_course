# Part 04: Loops - for, while, break, continue, iterators
## Steps 31-40: การวนซ้ำและ Iterators

---

## 🎯 เป้าหมายของ Part นี้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
- ใช้ `for` loop ทุกรูปแบบ
- ใช้ `while` loop อย่างมีประสิทธิภาพ
- ใช้ `break` และ `continue`
- เข้าใจ iterators ของ Nim
- สร้าง custom iterators
- ใช้ functional-style iterations

---

## Step 31: for Loop พื้นฐาน

```nim
# for loop กับ range
for i in 1..10:
  write(stdout, $i & " ")
echo ""  # 1 2 3 4 5 6 7 8 9 10

# range ด้วย ..< (exclusive end)
for i in 0..<5:
  write(stdout, $i & " ")
echo ""  # 0 1 2 3 4

# Countdown
for i in countdown(10, 1):
  write(stdout, $i & " ")
echo ""  # 10 9 8 7 6 5 4 3 2 1

# Step
for i in countup(0, 20, 5):
  write(stdout, $i & " ")
echo ""  # 0 5 10 15 20

# Iterate over sequence
var fruits = @["apple", "banana", "cherry", "date"]

for fruit in fruits:
  echo "🍎 " & fruit

# With index
for i, fruit in fruits:
  echo fmt"{i+1}. {fruit}"

# Iterate over string
for c in "Hello":
  write(stdout, c)
  write(stdout, "-")
echo ""  # H-e-l-l-o-

# Unicode-safe iteration
import std/unicode
for rune in "สวัสดี".runes:
  write(stdout, $rune & "|")
echo ""
```

---

## Step 32: for Loop ขั้นสูง

```nim
import std/tables, std/sets

# Iterate over Table
var scores = {"Alice": 95, "Bob": 87, "Charlie": 72}.toTable()

for name, score in scores:
  echo fmt"{name}: {score}"

# Iterate over Set
var uniqueNums = {1, 2, 3, 4, 5}.toHashSet()

for n in uniqueNums:
  write(stdout, $n & " ")
echo ""

# Iterate over characters with index
var word = "Nim"
for i, c in word:
  echo fmt"  [{i}] = '{c}' (ASCII: {ord(c)})"

# Parallel iteration (zip)
import std/sequtils

var names = @["Alice", "Bob", "Charlie"]
var ages  = @[25, 30, 28]

for (name, age) in zip(names, ages):
  echo fmt"{name} is {age} years old"

# Enumerate
for i, name in names:
  echo fmt"{i}: {name}"

# Nested loops
echo "\nMultiplication Table:"
for i in 1..5:
  for j in 1..5:
    write(stdout, fmt"{i*j:4}")
  echo ""

# for with multiple variables
var coords = [(1, 2), (3, 4), (5, 6)]
for (x, y) in coords:
  echo fmt"({x}, {y})"
```

---

## Step 33: while Loop

```nim
import std/strutils

# while พื้นฐาน
var n = 0
while n < 5:
  echo n
  n += 1

# while กับ complex condition
var temperature = 100.0
var timeElapsed = 0

while temperature > 0.0 and timeElapsed < 1000:
  temperature -= 0.5
  timeElapsed += 1

echo fmt"Temperature: {temperature:.1f} after {timeElapsed} seconds"

# Infinite loop กับ break
var count = 0
while true:
  count += 1
  if count >= 5:
    break

echo "Count: " & $count  # 5

# Read until sentinel
echo "ป้อนตัวเลข (0 เพื่อหยุด):"
var sum = 0
while true:
  let input = readLine(stdin)
  let num = parseInt(input.strip())
  if num == 0:
    break
  sum += num
  echo fmt"  ผลรวมขณะนี้: {sum}"

echo fmt"ผลรวมทั้งหมด: {sum}"

# while loop กับ state machine
type
  ParserState = enum
    Start, InWord, InNumber, InSpace, End

proc tokenize(text: string): seq[string] =
  result = @[]
  var state = Start
  var current = ""
  var i = 0
  
  while i < text.len:
    let c = text[i]
    
    case state
    of Start, InSpace:
      if c.isAlpha():
        state = InWord
        current = $c
      elif c.isDigit():
        state = InNumber
        current = $c
      # else: skip whitespace
    
    of InWord:
      if c.isAlpha() or c == '_':
        current &= c
      else:
        result.add(current)
        current = ""
        state = InSpace
        dec i  # re-process this char
    
    of InNumber:
      if c.isDigit() or c == '.':
        current &= c
      else:
        result.add(current)
        current = ""
        state = InSpace
        dec i
    
    of End:
      break
    
    inc i
  
  if current.len > 0:
    result.add(current)

let tokens = tokenize("hello 123 world 456.7")
echo tokens  # @["hello", "123", "world", "456.7"]
```

---

## Step 34: break และ continue

```nim
# break - หยุด loop
echo "Break example:"
for i in 1..10:
  if i == 5:
    break
  write(stdout, $i & " ")
echo ""  # 1 2 3 4

# continue - ข้ามรอบนั้น
echo "\nContinue example (skip evens):"
for i in 1..10:
  if i mod 2 == 0:
    continue
  write(stdout, $i & " ")
echo ""  # 1 3 5 7 9

# break กับ label (nested loops)
echo "\nNested loop with label:"
block outer:
  for i in 1..3:
    for j in 1..3:
      if i == 2 and j == 2:
        break outer  # break ออกจาก outer loop
      echo fmt"  ({i},{j})"

# Named blocks
block findFirst:
  var numbers = @[5, 3, 8, 1, 9, 2, 7, 4, 6]
  for i, n in numbers:
    if n > 7:
      echo fmt"First number > 7: {n} at index {i}"
      break findFirst

# continue กับ label
echo "\nContinue with label:"
block outer:
  for i in 1..3:
    for j in 1..3:
      if j == 2:
        continue outer  # ข้ามไป outer loop ถัดไป
      echo fmt"  ({i},{j})"

# Practical: Find prime numbers
proc isPrime(n: int): bool =
  if n < 2: return false
  if n == 2: return true
  if n mod 2 == 0: return false
  
  var i = 3
  while i * i <= n:
    if n mod i == 0:
      return false
    i += 2
  
  return true

echo "\nPrime numbers up to 50:"
for n in 2..50:
  if isPrime(n):
    write(stdout, $n & " ")
echo ""
```

---

## Step 35: Iterators

Iterator คือ Generator ที่ yield ค่าทีละค่า

```nim
# Iterator พื้นฐาน
iterator countTo(n: int): int =
  var i = 1
  while i <= n:
    yield i
    i += 1

for x in countTo(5):
  write(stdout, $x & " ")
echo ""  # 1 2 3 4 5

# Iterator กับ seq
iterator items[T](s: seq[T]): T =
  for i in 0..<s.len:
    yield s[i]

# Iterator ที่ yield pairs
iterator pairs[T](s: seq[T]): (int, T) =
  for i in 0..<s.len:
    yield (i, s[i])

var fruits = @["apple", "banana", "cherry"]
for i, fruit in pairs(fruits):
  echo fmt"{i}: {fruit}"

# Fibonacci iterator
iterator fibonacci(limit: int): int =
  var a = 0
  var b = 1
  while a <= limit:
    yield a
    let temp = a + b
    a = b
    b = temp

echo "\nFibonacci numbers up to 100:"
for fib in fibonacci(100):
  write(stdout, $fib & " ")
echo ""

# Iterator กับ files (concept)
iterator lines(filename: string): string =
  let f = open(filename)
  defer: close(f)
  var line = ""
  while readLine(f, line):
    yield line

# Iterator สำหรับ permutations
iterator permutations[T](s: seq[T]): seq[T] =
  var arr = s
  let n = arr.len
  var c = newSeq[int](n)
  
  yield arr
  
  var i = 0
  while i < n:
    if c[i] < i:
      if i mod 2 == 0:
        swap(arr[0], arr[i])
      else:
        swap(arr[c[i]], arr[i])
      
      yield arr
      
      c[i] += 1
      i = 0
    else:
      c[i] = 0
      i += 1

echo "\nPermutations of [1,2,3]:"
var count = 0
for perm in permutations(@[1, 2, 3]):
  echo perm
  count += 1
echo fmt"Total: {count} permutations"
```

---

## Step 36: Functional-style Loops

```nim
import std/sequtils, std/algorithm, std/sugar

var numbers = @[1, 2, 3, 4, 5, 6, 7, 8, 9, 10]

# map - แปลงทุก element
var doubled = numbers.map(x => x * 2)
echo doubled  # @[2, 4, 6, 8, 10, 12, 14, 16, 18, 20]

# filter - กรอง elements
var evens = numbers.filter(x => x mod 2 == 0)
echo evens  # @[2, 4, 6, 8, 10]

# foldl (reduce left)
var sum = numbers.foldl(a + b)
echo sum  # 55

# foldl กับ starting value
var product = numbers.foldl(a * b, 1)
echo product  # 3628800

# filterIt, mapIt (คล้าย list comprehension)
var bigSquares = numbers.filterIt(it > 5).mapIt(it * it)
echo bigSquares  # @[36, 49, 64, 81, 100]

# any/all
echo numbers.any(x => x > 9)   # true
echo numbers.all(x => x > 0)   # true
echo numbers.all(x => x > 5)   # false

# find
echo numbers.find(8)   # 7 (index)
echo numbers.contains(5)  # true

# sort
var data = @[3, 1, 4, 1, 5, 9, 2, 6, 5, 3]
data.sort()
echo data  # sorted ascending

data.sort(Descending)
echo data  # sorted descending

# Custom sort
var words = @["banana", "apple", "cherry", "date"]
words.sort(proc(a, b: string): int = a.len - b.len)
echo words  # sorted by length

# zip (parallel iteration)
var a = @[1, 2, 3]
var b = @["x", "y", "z"]
for (num, letter) in zip(a, b):
  echo fmt"{num}: {letter}"

# unzip
var pairs2 = @[(1, "a"), (2, "b"), (3, "c")]
let (nums, letters) = pairs2.unzip()
echo nums     # @[1, 2, 3]
echo letters  # @["a", "b", "c"]

# deduplicate
var dups = @[1, 2, 2, 3, 3, 3, 4]
echo dups.deduplicate()  # @[1, 2, 3, 4]

# flatten
var nested = @[@[1, 2], @[3, 4], @[5, 6]]
echo nested.concat()  # @[1, 2, 3, 4, 5, 6]
```

---

## Step 37: Loop Patterns

```nim
import std/tables, std/strutils

# Accumulator pattern
proc sumOfSquares(numbers: seq[int]): int =
  result = 0
  for n in numbers:
    result += n * n

echo sumOfSquares(@[1, 2, 3, 4, 5])  # 55

# Running total
proc runningTotal(numbers: seq[int]): seq[int] =
  result = @[]
  var total = 0
  for n in numbers:
    total += n
    result.add(total)

echo runningTotal(@[1, 2, 3, 4, 5])  # @[1, 3, 6, 10, 15]

# Sliding window
proc slidingAverage(data: seq[float], windowSize: int): seq[float] =
  result = @[]
  if data.len < windowSize:
    return
  
  for i in 0..data.len - windowSize:
    var sum = 0.0
    for j in i..<i + windowSize:
      sum += data[j]
    result.add(sum / float(windowSize))

let temps = @[20.0, 22.0, 18.0, 25.0, 24.0, 21.0, 23.0]
echo slidingAverage(temps, 3)  # 3-day moving average

# Group by pattern
proc groupBy[T, K](items: seq[T], keyFn: proc(x: T): K): Table[K, seq[T]] =
  result = initTable[K, seq[T]]()
  for item in items:
    let key = keyFn(item)
    if key notin result:
      result[key] = @[]
    result[key].add(item)

type
  Person = object
    name: string
    department: string
    salary: float

var employees = @[
  Person(name: "Alice", department: "Engineering", salary: 80000),
  Person(name: "Bob", department: "Marketing", salary: 65000),
  Person(name: "Charlie", department: "Engineering", salary: 90000),
  Person(name: "Diana", department: "HR", salary: 60000),
  Person(name: "Eve", department: "Marketing", salary: 70000)
]

let byDept = groupBy(employees, proc(p: Person): string = p.department)

for dept, people in byDept:
  echo fmt"\n{dept}:"
  for p in people:
    echo fmt"  - {p.name} (${p.salary:.0f})"

# Chunking
proc chunked[T](s: seq[T], size: int): seq[seq[T]] =
  result = @[]
  var i = 0
  while i < s.len:
    result.add(s[i..<min(i + size, s.len)])
    i += size

let items = @[1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
echo items.chunked(3)  # @[@[1,2,3], @[4,5,6], @[7,8,9], @[10]]
```

---

## Step 38: Loop Optimization

```nim
import std/times, std/strutils

# Bad: Concatenating strings in loop (slow!)
proc buildStringBad(n: int): string =
  var s = ""
  for i in 1..n:
    s &= $i & ","  # Creates new string each time!
  return s

# Good: Use seq then join (fast!)
proc buildStringGood(n: int): string =
  var parts: seq[string] = @[]
  for i in 1..n:
    parts.add($i)
  return parts.join(",")

# Even better: Use a buffer
import std/strformat

proc buildStringBest(n: int): string =
  var buf = newStringOfCap(n * 4)  # Pre-allocate
  for i in 1..n:
    if i > 1: buf.add(',')
    buf.add($i)
  return buf

# Benchmark (approximate)
let n = 10000

let t1 = epochTime()
discard buildStringBad(n)
let t2 = epochTime()
discard buildStringGood(n)
let t3 = epochTime()
discard buildStringBest(n)
let t4 = epochTime()

echo fmt"Bad:   {(t2-t1)*1000:.2f}ms"
echo fmt"Good:  {(t3-t2)*1000:.2f}ms"
echo fmt"Best:  {(t4-t3)*1000:.2f}ms"

# Pre-allocate sequences
proc processDataBad(n: int): seq[int] =
  result = @[]
  for i in 1..n:
    result.add(i * 2)  # May reallocate multiple times

proc processDataGood(n: int): seq[int] =
  result = newSeqOfCap[int](n)  # Pre-allocate!
  for i in 1..n:
    result.add(i * 2)
```

---

## Step 39: Custom Iterators สำหรับ Backend

```nim
import std/options, std/tables

# Paginated iterator
type
  Page[T] = object
    items: seq[T]
    pageNum: int
    totalPages: int
    hasNext: bool
    hasPrev: bool

iterator paginate[T](items: seq[T], pageSize: int): Page[T] =
  let totalPages = (items.len + pageSize - 1) div pageSize
  
  for pageNum in 1..totalPages:
    let startIdx = (pageNum - 1) * pageSize
    let endIdx = min(startIdx + pageSize, items.len)
    
    yield Page[T](
      items: items[startIdx..<endIdx],
      pageNum: pageNum,
      totalPages: totalPages,
      hasNext: pageNum < totalPages,
      hasPrev: pageNum > 1
    )

# Usage
var users = @["Alice", "Bob", "Charlie", "Diana", "Eve", "Frank", "Grace"]

echo "Pages of 3:"
for page in paginate(users, 3):
  echo fmt"\nPage {page.pageNum}/{page.totalPages}:"
  for user in page.items:
    echo fmt"  - {user}"
  if page.hasNext: echo "  [Next page available]"
  if page.hasPrev: echo "  [Previous page available]"

# Batch processing iterator
iterator batches[T](items: seq[T], batchSize: int): seq[T] =
  var i = 0
  while i < items.len:
    yield items[i..<min(i + batchSize, items.len)]
    i += batchSize

# Process 1000 records in batches of 100
var records = newSeq[int](1000)
for i in 0..<1000: records[i] = i + 1

var processed = 0
for batch in batches(records, 100):
  processed += batch.len
  echo fmt"Processing batch of {batch.len} records... ({processed}/1000)"

# Retry iterator
iterator withRetry[T](operation: proc(): T, maxRetries: int): T =
  var attempts = 0
  var succeeded = false
  
  while attempts < maxRetries and not succeeded:
    try:
      let result = operation()
      succeeded = true
      yield result
    except:
      attempts += 1
      if attempts < maxRetries:
        echo fmt"Attempt {attempts} failed, retrying..."
      else:
        echo fmt"All {maxRetries} attempts failed"
```

---

## Step 40: Real-World Application - Log Processor

```nim
# log_processor.nim
# Processing log files with iterators

import std/strutils, std/times, std/tables, std/sequtils, std/algorithm

# ==============================
# Types
# ==============================

type
  LogLevel = enum
    Debug, Info, Warning, Error, Critical

  LogEntry = object
    timestamp: string
    level: LogLevel
    service: string
    message: string
    duration: Option[float]  # สำหรับ request logs

# ==============================
# Log Generator (simulated)
# ==============================

proc generateLogs(): seq[LogEntry] =
  # Simulate log entries
  let entries = [
    ("2024-01-15 10:00:01", "INFO",     "API",      "Server started", 0.0),
    ("2024-01-15 10:00:05", "INFO",     "AUTH",     "User login: alice", 45.0),
    ("2024-01-15 10:00:07", "DEBUG",    "DB",       "Query executed", 12.5),
    ("2024-01-15 10:00:10", "WARNING",  "API",      "Rate limit approaching", 0.0),
    ("2024-01-15 10:00:15", "ERROR",    "DB",       "Connection timeout", 5000.0),
    ("2024-01-15 10:00:20", "INFO",     "AUTH",     "User logout: alice", 23.0),
    ("2024-01-15 10:00:25", "CRITICAL", "API",      "Unhandled exception", 0.0),
    ("2024-01-15 10:00:30", "INFO",     "API",      "Health check OK", 5.0),
    ("2024-01-15 10:00:35", "ERROR",    "AUTH",     "Invalid token", 10.0),
    ("2024-01-15 10:00:40", "WARNING",  "DB",       "Slow query detected", 2500.0),
    ("2024-01-15 10:00:45", "INFO",     "API",      "Request processed", 150.0),
    ("2024-01-15 10:00:50", "ERROR",    "API",      "Service unavailable", 0.0),
  ]
  
  result = @[]
  for (ts, lvl, svc, msg, dur) in entries:
    let level = case lvl
      of "DEBUG": Debug
      of "INFO": Info
      of "WARNING": Warning
      of "ERROR": Error
      of "CRITICAL": Critical
      else: Info
    
    result.add(LogEntry(
      timestamp: ts,
      level: level,
      service: svc,
      message: msg,
      duration: if dur > 0: some(dur) else: none(float)
    ))

# ==============================
# Iterators
# ==============================

iterator filterByLevel(logs: seq[LogEntry], minLevel: LogLevel): LogEntry =
  for entry in logs:
    if entry.level >= minLevel:
      yield entry

iterator filterByService(logs: seq[LogEntry], service: string): LogEntry =
  for entry in logs:
    if entry.service == service:
      yield entry

iterator slowRequests(logs: seq[LogEntry], threshold: float): LogEntry =
  for entry in logs:
    if entry.duration.isSome() and entry.duration.get() > threshold:
      yield entry

# ==============================
# Analysis
# ==============================

proc analyzeErrors(logs: seq[LogEntry]) =
  echo "\n🔴 Errors and Critical Issues:"
  echo "━".repeat(50)
  
  var count = 0
  for entry in logs.filterByLevel(Error):
    inc count
    let levelIcon = if entry.level == Critical: "🔥" else: "❌"
    echo fmt"{levelIcon} [{entry.timestamp}] {entry.service}: {entry.message}"
  
  echo fmt"\nTotal errors: {count}"

proc analyzeByService(logs: seq[LogEntry]) =
  echo "\n📊 Logs by Service:"
  echo "━".repeat(50)
  
  var serviceCounts = initTable[string, int]()
  var serviceErrors = initTable[string, int]()
  
  for entry in logs:
    serviceCounts[entry.service] = serviceCounts.getOrDefault(entry.service, 0) + 1
    if entry.level >= Error:
      serviceErrors[entry.service] = serviceErrors.getOrDefault(entry.service, 0) + 1
  
  for service in serviceCounts.keys.toSeq().sorted():
    let total = serviceCounts[service]
    let errors = serviceErrors.getOrDefault(service, 0)
    let errPct = if total > 0: float(errors) / float(total) * 100 else: 0.0
    echo fmt"{service:12} : {total:3} logs, {errors:2} errors ({errPct:.1f}%)"

proc analyzeSlowRequests(logs: seq[LogEntry], threshold: float = 1000.0) =
  echo fmt"\n🐌 Slow Requests (> {threshold:.0f}ms):"
  echo "━".repeat(50)
  
  var slowOnes: seq[(float, string, string)] = @[]
  for entry in logs.slowRequests(threshold):
    slowOnes.add((entry.duration.get(), entry.service, entry.message))
  
  slowOnes.sort(proc(a, b: (float, string, string)): int =
    cmp(b[0], a[0])  # Sort by duration descending
  )
  
  for (dur, svc, msg) in slowOnes:
    echo fmt"  {dur:8.1f}ms | {svc:10} | {msg}"

proc generateSummary(logs: seq[LogEntry]) =
  echo "\n📈 Log Summary:"
  echo "━".repeat(50)
  
  var levelCounts: Table[LogLevel, int]
  for entry in logs:
    levelCounts[entry.level] = levelCounts.getOrDefault(entry.level, 0) + 1
  
  let levels = [Debug, Info, Warning, Error, Critical]
  let icons = ["🔵", "✅", "⚠️ ", "❌", "🔥"]
  
  for i, lvl in levels:
    let count = levelCounts.getOrDefault(lvl, 0)
    let bar = "█".repeat(count * 2)
    echo fmt"{icons[i]} {$lvl:10}: {count:3} {bar}"

# ==============================
# Main
# ==============================

proc main() =
  echo "╔══════════════════════════════════════╗"
  echo "║         Log Analyzer (Nim)            ║"
  echo "╚══════════════════════════════════════╝"
  
  let logs = generateLogs()
  echo fmt"\nLoaded {logs.len} log entries"
  
  generateSummary(logs)
  analyzeErrors(logs)
  analyzeByService(logs)
  analyzeSlowRequests(logs, 1000.0)
  
  echo "\n✅ Analysis complete!"

main()
```

---

## 📝 สรุป Part 04

| Step | หัวข้อ |
|------|--------|
| 31 | for loop พื้นฐาน - ranges, sequences |
| 32 | for ขั้นสูง - tables, sets, parallel |
| 33 | while loop - condition, state machine |
| 34 | break/continue - control flow |
| 35 | Iterators - yield, custom iterators |
| 36 | Functional-style - map/filter/fold |
| 37 | Loop patterns - accumulator, sliding window |
| 38 | Loop optimization - pre-allocation |
| 39 | Backend iterators - pagination, batching |
| 40 | Real-world - Log Processor |

---

**← [Part 03: Control Flow](part_03_control_flow.md) | [Part 05: Procedures & Functions →](part_05_procedures.md)**
