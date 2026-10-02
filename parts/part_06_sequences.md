# Part 06: Arrays, Sequences, Tuples & Sets
## Steps 56-70: โครงสร้างข้อมูลพื้นฐาน

---

## 🎯 เป้าหมายของ Part นี้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
- ใช้ Arrays ขนาดคงที่
- ใช้ Sequences (dynamic arrays) อย่างมีประสิทธิภาพ
- ใช้ Tuples สำหรับ grouping data
- ใช้ Sets และ HashSets
- ใช้ Tables (Hash Maps)
- เลือก data structure ที่เหมาะสม

---

## Step 56: Arrays - Fixed-Size Collections

```nim
# Array declaration
var arr: array[5, int] = [1, 2, 3, 4, 5]
var strs: array[3, string] = ["apple", "banana", "cherry"]

# Type inference
var nums = [10, 20, 30, 40, 50]  # array[0..4, int]

# Array ที่ zero-indexed
echo arr[0]   # 1
echo arr[4]   # 5
echo arr[^1]  # 5 (last element)
echo arr[^2]  # 4 (second to last)

# Array length
echo arr.len   # 5
echo low(arr)  # 0
echo high(arr) # 4

# Modify elements
arr[0] = 100
echo arr  # [100, 2, 3, 4, 5]

# Array slice
echo arr[1..3]  # [2, 3, 4]
echo arr[0..<3] # [1, 2, 3]

# 2D array
var matrix: array[3, array[3, int]] = [
  [1, 2, 3],
  [4, 5, 6],
  [7, 8, 9]
]

echo matrix[1][2]  # 6

# Iterate
for i in arr:
  write(stdout, $i & " ")
echo ""

for i, v in arr:
  echo fmt"  arr[{i}] = {v}"

# Array with custom index range
var weekdays: array[1..7, string]
weekdays[1] = "Monday"
weekdays[2] = "Tuesday"
weekdays[7] = "Sunday"

# Array functions
import std/algorithm

var sortable = [5, 2, 8, 1, 9, 3, 7, 4, 6]
sortable.sort()
echo sortable  # [1, 2, 3, 4, 5, 6, 7, 8, 9]

# Check membership
echo 5 in sortable  # true
echo 10 in sortable  # false
```

---

## Step 57: Sequences - Dynamic Arrays

```nim
import std/sequtils, std/algorithm

# Sequence creation
var empty: seq[int] = @[]
var nums = @[1, 2, 3, 4, 5]
var strs = @["apple", "banana", "cherry"]

# Sequence with type
var floats: seq[float] = newSeq[float](5)  # length 5, all 0.0

# newSeq with capacity hint
var big = newSeqOfCap[int](1000)  # pre-allocate

# Add elements
nums.add(6)
nums.add(7)
echo nums  # @[1, 2, 3, 4, 5, 6, 7]

# Insert
nums.insert(99, 2)  # insert 99 at index 2
echo nums  # @[1, 2, 99, 3, 4, 5, 6, 7]

# Delete
nums.delete(2)  # delete element at index 2
echo nums  # @[1, 2, 3, 4, 5, 6, 7]

# Pop (remove last)
let last = nums.pop()
echo last  # 7
echo nums  # @[1, 2, 3, 4, 5, 6]

# Remove specific element
nums.keepIf(proc(x: int): bool = x != 3)
echo nums  # @[1, 2, 4, 5, 6]

# Concatenate
var a = @[1, 2, 3]
var b = @[4, 5, 6]
var c = a & b
echo c  # @[1, 2, 3, 4, 5, 6]

# seq operations
echo nums.len        # length
echo nums[0]         # first
echo nums[^1]        # last
echo nums[1..3]      # slice

# map, filter, foldl
var doubled = nums.map(x => x * 2)
var evens   = nums.filter(x => x mod 2 == 0)
var sum     = nums.foldl(a + b, 0)

echo doubled  # @[2, 4, 8, 10, 12]
echo evens    # @[2, 4, 6]
echo sum      # 18

# sort
var unsorted = @[3, 1, 4, 1, 5, 9, 2, 6]
unsorted.sort()
echo unsorted  # @[1, 1, 2, 3, 4, 5, 6, 9]

# reverse
unsorted.reverse()
echo unsorted  # @[9, 6, 5, 4, 3, 2, 1, 1]

# find/contains
echo nums.find(4)     # 2 (index) or -1
echo nums.contains(5) # true

# any/all
echo nums.any(x => x > 5)   # true
echo nums.all(x => x > 0)   # true
```

---

## Step 58: Sequence Algorithms

```nim
import std/sequtils, std/algorithm, std/sugar

var data = @[5, 2, 8, 1, 9, 3, 7, 4, 6, 0]

# Sort with custom comparator
var sortedAsc = data.sorted()                     # ascending
var sortedDesc = data.sorted(cmp = (a, b) => b - a)  # descending

# Sort objects by field
type
  Student = object
    name: string
    gpa: float
    age: int

var students = @[
  Student(name: "Alice", gpa: 3.8, age: 22),
  Student(name: "Bob", gpa: 3.5, age: 20),
  Student(name: "Charlie", gpa: 3.9, age: 21),
  Student(name: "Diana", gpa: 3.5, age: 19),
]

# Sort by GPA (descending)
students.sort(proc(a, b: Student): int =
  cmp(b.gpa, a.gpa)
)

for s in students:
  echo fmt"  {s.name}: GPA {s.gpa}"

# Sort by GPA, then age
students.sort(proc(a, b: Student): int =
  let gpaComp = cmp(b.gpa, a.gpa)
  if gpaComp != 0: return gpaComp
  return cmp(a.age, b.age)
)

# Unique elements
var withDups = @[1, 2, 2, 3, 3, 3, 4, 4, 4, 4]
echo withDups.deduplicate()  # @[1, 2, 3, 4]

# Partition
let (evens2, odds) = data.partition(x => x mod 2 == 0)
echo "Evens: " & $evens2
echo "Odds:  " & $odds

# Group consecutive
proc groupConsecutive[T](s: seq[T]): seq[seq[T]] =
  if s.len == 0: return @[]
  
  result = @[]
  var group = @[s[0]]
  
  for i in 1..<s.len:
    if s[i] == s[i-1] + 1:  # consecutive
      group.add(s[i])
    else:
      result.add(group)
      group = @[s[i]]
  
  result.add(group)

echo groupConsecutive(@[1, 2, 3, 5, 6, 8, 9, 10])
# @[@[1, 2, 3], @[5, 6], @[8, 9, 10]]

# Flatten nested sequences
var nested = @[@[1, 2], @[3, 4], @[5, 6]]
echo nested.concat()  # @[1, 2, 3, 4, 5, 6]

# Zip multiple seqs
var names = @["Alice", "Bob", "Charlie"]
var scores = @[95, 87, 92]
var ages   = @[22, 20, 21]

for (n, s, a) in zip(zip(names, scores), ages):
  let (name, score) = n
  echo fmt"{name}: score={score}, age={a}"
```

---

## Step 59: Tuples

```nim
import std/strformat

# Tuple declaration
var point: (int, int) = (3, 4)
var person = ("Alice", 28, true)  # inferred: (string, int, bool)

# Named tuple
type
  Point = (x: float, y: float)
  Color = (r: int, g: int, b: int)
  RGB = tuple[r, g, b: uint8]

var p: Point = (x: 1.5, y: 2.5)
var red: Color = (r: 255, g: 0, b: 0)
var white: RGB = (r: 255, g: 255, b: 255)

# Access by position
echo point[0]   # 3
echo point[1]   # 4

# Access by name
echo p.x  # 1.5
echo p.y  # 2.5

# Destructure
let (x, y) = point
echo fmt"x={x}, y={y}"

let (name, age, active) = person
echo fmt"{name}, {age}, {active}"

# Tuple in functions
proc minMax(nums: seq[int]): (int, int) =
  var mn = nums[0]
  var mx = nums[0]
  for n in nums:
    if n < mn: mn = n
    if n > mx: mx = n
  return (mn, mx)

let (minimum, maximum) = minMax(@[3, 1, 4, 1, 5, 9, 2, 6])
echo fmt"Min: {minimum}, Max: {maximum}"

# Named tuple as object-lite
type
  UserInfo = tuple
    id: int
    name: string
    email: string
    isActive: bool

var user: UserInfo = (
  id: 1,
  name: "Alice",
  email: "alice@example.com",
  isActive: true
)

echo fmt"User: {user.name} ({user.email})"

# Tuple comparison
echo (1, 2) < (1, 3)    # true
echo (1, 2) == (1, 2)   # true
echo (2, 1) > (1, 9)    # true (compares first element first)

# Tuple as hash table key
import std/tables

var lookup = initTable[(string, int), string]()
lookup[("Alice", 2024)] = "Year Award"
lookup[("Bob", 2023)] = "Innovation Award"

echo lookup[("Alice", 2024)]  # Year Award

# Swap using tuple
var a = 10
var b = 20
(a, b) = (b, a)
echo fmt"a={a}, b={b}"  # a=20, b=10
```

---

## Step 60: Sets

```nim
import std/sets

# Regular set (for ordinal types, small range)
var colors: set[char] = {'r', 'g', 'b'}
var bits: set[int8] = {0i8, 1i8, 2i8, 7i8}

echo colors  # {'b', 'g', 'r'}

# Set operations
var setA: set[int] = {1, 2, 3, 4, 5}
var setB: set[int] = {3, 4, 5, 6, 7}

echo setA + setB    # Union: {1, 2, 3, 4, 5, 6, 7}
echo setA * setB    # Intersection: {3, 4, 5}
echo setA - setB    # Difference: {1, 2}

# Membership
echo 3 in setA      # true
echo 6 in setA      # false
echo 6 notin setA   # true

# Modify set
setA.incl(6)
setA.excl(1)
echo setA  # {2, 3, 4, 5, 6}

# Subset/superset
var small = {2, 3}
echo small <= setA  # subset: true
echo setA >= small  # superset: true

# HashSet (for any type)
import std/tables

var strSet = initHashSet[string]()
strSet.incl("apple")
strSet.incl("banana")
strSet.incl("apple")  # duplicate, ignored

echo strSet.len      # 2 (unique)
echo "apple" in strSet  # true

# Convert seq to set (deduplicate)
var nums = @[1, 2, 2, 3, 3, 3, 4]
var uniqueNums = nums.toHashSet()
echo uniqueNums  # {1, 2, 3, 4}

# OrderedSet (maintains insertion order)
import std/sets
var orderedSet = initOrderedSet[string]()
orderedSet.incl("Charlie")
orderedSet.incl("Alice")
orderedSet.incl("Bob")

for item in orderedSet:
  write(stdout, item & " ")
echo ""  # Charlie Alice Bob (insertion order)
```

---

## Step 61: Tables (Hash Maps)

```nim
import std/tables, std/strformat

# Create table
var scores = initTable[string, int]()
scores["Alice"] = 95
scores["Bob"] = 87
scores["Charlie"] = 92

# Or from pairs
var colors2 = {"red": "#FF0000", "green": "#00FF00", "blue": "#0000FF"}.toTable()

# Access
echo scores["Alice"]  # 95

# Safe access (default value)
echo scores.getOrDefault("Diana", 0)  # 0

# Check existence
echo "Alice" in scores   # true
echo "Diana" in scores   # false

# Iterate
for name, score in scores:
  echo fmt"  {name}: {score}"

# Keys and values
echo scores.keys.toSeq()    # names
echo scores.values.toSeq()  # scores

# Update
scores["Alice"] = 98  # update
scores.del("Bob")     # delete

# mgetOrPut (modify in place)
scores.mgetOrPut("Eve", 0) += 10

# Table of sequences
var groups = initTable[string, seq[string]]()
groups["fruits"] = @["apple", "banana"]
groups["fruits"].add("cherry")
groups["veggies"] = @["carrot", "broccoli"]

for category, items in groups:
  echo fmt"{category}: {items.join(\", \")}"

# Count occurrences
proc countItems[T](items: seq[T]): Table[T, int] =
  result = initTable[T, int]()
  for item in items:
    result[item] = result.getOrDefault(item, 0) + 1

let words = @["apple", "banana", "apple", "cherry", "banana", "apple"]
let wordCount = countItems(words)

for word, count in wordCount:
  echo fmt"  {word}: {count}"

# OrderedTable (maintains insertion order)
import std/tables
var orderedMap = initOrderedTable[string, int]()
orderedMap["z"] = 3
orderedMap["a"] = 1
orderedMap["m"] = 2

for k, v in orderedMap:
  echo fmt"{k}: {v}"  # z: 3, a: 1, m: 2 (insertion order)

# Sort by value
import std/algorithm
var sortedPairs = wordCount.pairs.toSeq()
sortedPairs.sort(proc(a, b: (string, int)): int = b[1] - a[1])

echo "\nWord frequency (sorted):"
for (word, count) in sortedPairs:
  echo fmt"  {word}: {count}"
```

---

## Step 62: Advanced Collections

```nim
import std/deques, std/heapqueue

# Deque (double-ended queue)
var dq = initDeque[int]()
dq.addLast(1)
dq.addLast(2)
dq.addLast(3)
dq.addFirst(0)

echo dq             # Deque with [0, 1, 2, 3]
echo dq.popFirst()  # 0
echo dq.peekFirst() # 1
echo dq.popLast()   # 3

# HeapQueue (priority queue)
var pq = initHeapQueue[int]()
pq.push(5)
pq.push(1)
pq.push(3)
pq.push(2)
pq.push(4)

while pq.len > 0:
  write(stdout, $pq.pop() & " ")
echo ""  # 1 2 3 4 5 (min heap)

# Custom priority
type
  Task = object
    priority: int
    name: string

proc `<`(a, b: Task): bool = a.priority < b.priority

var taskQueue = initHeapQueue[Task]()
taskQueue.push(Task(priority: 3, name: "Low priority"))
taskQueue.push(Task(priority: 1, name: "High priority"))
taskQueue.push(Task(priority: 2, name: "Medium priority"))

while taskQueue.len > 0:
  let task = taskQueue.pop()
  echo fmt"Processing: {task.name} (priority: {task.priority})"

# Circular buffer
type
  CircularBuffer[T] = object
    data: seq[T]
    head, tail, size, capacity: int

proc newCircularBuffer[T](cap: int): CircularBuffer[T] =
  CircularBuffer[T](data: newSeq[T](cap), capacity: cap)

proc push[T](buf: var CircularBuffer[T], item: T) =
  if buf.size == buf.capacity:
    buf.head = (buf.head + 1) mod buf.capacity
  else:
    inc buf.size
  buf.data[buf.tail] = item
  buf.tail = (buf.tail + 1) mod buf.capacity

proc toSeq[T](buf: CircularBuffer[T]): seq[T] =
  result = newSeq[T](buf.size)
  for i in 0..<buf.size:
    result[i] = buf.data[(buf.head + i) mod buf.capacity]

var cbuf = newCircularBuffer[int](5)
for i in 1..8:
  cbuf.push(i)

echo cbuf.toSeq()  # @[4, 5, 6, 7, 8] (only last 5)
```

---

## Step 63: Data Structure Patterns

```nim
import std/tables, std/sets, std/sequtils

# LRU Cache implementation
type
  LRUCache[K, V] = object
    capacity: int
    cache: OrderedTable[K, V]

proc newLRUCache[K, V](capacity: int): LRUCache[K, V] =
  LRUCache[K, V](capacity: capacity, cache: initOrderedTable[K, V]())

proc get[K, V](lru: var LRUCache[K, V], key: K): Option[V] =
  if key notin lru.cache:
    return none(V)
  
  let value = lru.cache[key]
  lru.cache.del(key)
  lru.cache[key] = value  # Move to end (most recently used)
  return some(value)

proc put[K, V](lru: var LRUCache[K, V], key: K, value: V) =
  if key in lru.cache:
    lru.cache.del(key)
  elif lru.cache.len >= lru.capacity:
    # Remove least recently used (first item)
    for k in lru.cache.keys:
      lru.cache.del(k)
      break
  
  lru.cache[key] = value

var cache = newLRUCache[string, int](3)
cache.put("a", 1)
cache.put("b", 2)
cache.put("c", 3)

echo cache.get("a")  # some(1) - a moves to recent
cache.put("d", 4)    # removes b (least recently used)

echo cache.get("b")  # none - b was evicted
echo cache.get("c")  # some(3)
echo cache.get("d")  # some(4)

# Inverted index (for search)
type
  Document = object
    id: int
    title: string
    content: string

proc buildIndex(docs: seq[Document]): Table[string, seq[int]] =
  result = initTable[string, seq[int]]()
  
  for doc in docs:
    let words = doc.content.toLower().split()
    for word in words.toHashSet():  # unique words
      if word notin result:
        result[word] = @[]
      result[word].add(doc.id)

proc search(index: Table[string, seq[int]], query: string): seq[int] =
  let words = query.toLower().split()
  if words.len == 0: return @[]
  
  var results = initHashSet[int]()
  var first = true
  
  for word in words:
    if word in index:
      let docIds = index[word].toHashSet()
      if first:
        results = docIds
        first = false
      else:
        results = results * docIds  # intersection
  
  return results.toSeq()

let docs = @[
  Document(id: 1, title: "Nim Tutorial", content: "nim programming language tutorial"),
  Document(id: 2, title: "Web Development", content: "web development with nim jester"),
  Document(id: 3, title: "Database Guide", content: "nim database sqlite postgresql"),
]

let index = buildIndex(docs)
echo search(index, "nim")           # @[1, 2, 3]
echo search(index, "nim database")  # @[3] (intersection)
```

---

## Step 64: Sequences สำหรับ Backend

```nim
import std/strutils, std/sequtils, std/algorithm, std/tables

# Pipeline data processing
type
  DataRecord = object
    id: int
    category: string
    value: float
    tags: seq[string]

proc processData(records: seq[DataRecord]): Table[string, float] =
  # Group by category and sum values
  result = initTable[string, float]()
  
  for record in records:
    if record.category notin result:
      result[record.category] = 0.0
    result[record.category] += record.value

# Batch operations
proc processBatch[T, R](
  items: seq[T],
  batchSize: int,
  processor: proc(batch: seq[T]): seq[R]
): seq[R] =
  result = @[]
  var i = 0
  while i < items.len:
    let batch = items[i..<min(i + batchSize, items.len)]
    result.add(processor(batch))
    i += batchSize

let numbers = (1..100).toSeq()
let processed = processBatch(numbers, 10, proc(batch: seq[int]): seq[int] =
  batch.map(x => x * 2)
)
echo processed.len  # 100

# Efficient string building with seq
proc buildHtml(items: seq[string]): string =
  var parts: seq[string] = @[]
  parts.add("<ul>")
  for item in items:
    parts.add(fmt"  <li>{item}</li>")
  parts.add("</ul>")
  return parts.join("\n")

echo buildHtml(@["Item 1", "Item 2", "Item 3"])
```

---

## Step 65: Real-World Application - Shopping Cart

```nim
# shopping_cart.nim

import std/tables, std/sequtils, std/algorithm, std/strformat, std/options

type
  ProductId = distinct int
  
  Product = object
    id: ProductId
    name: string
    price: float
    category: string
    tags: seq[string]
  
  CartItem = object
    product: Product
    quantity: int
    note: string
  
  Cart = object
    items: seq[CartItem]
    coupon: Option[string]
    discountPct: float

# ==============================
# Product Operations
# ==============================

proc `$`(p: ProductId): string = "P" & $int(p)

var catalog: Table[ProductId, Product] = initTable[ProductId, Product]()

proc addProduct(
  id: int,
  name: string,
  price: float,
  category: string,
  tags: seq[string] = @[]
) =
  let pid = ProductId(id)
  catalog[pid] = Product(
    id: pid,
    name: name,
    price: price,
    category: category,
    tags: tags
  )

# Setup catalog
addProduct(1, "Laptop Pro", 45000.0, "Electronics", @["computer", "work"])
addProduct(2, "Wireless Mouse", 1200.0, "Electronics", @["peripheral", "office"])
addProduct(3, "Mechanical Keyboard", 3500.0, "Electronics", @["peripheral", "gaming"])
addProduct(4, "USB-C Hub", 1800.0, "Electronics", @["accessory"])
addProduct(5, "Notebook A5", 150.0, "Stationery", @["writing"])
addProduct(6, "Pen Set", 250.0, "Stationery", @["writing"])

# ==============================
# Cart Operations
# ==============================

proc newCart(): Cart =
  Cart(items: @[], coupon: none(string), discountPct: 0.0)

proc addToCart(cart: var Cart, productId: ProductId, qty: int = 1, note: string = ""): bool =
  if productId notin catalog:
    return false
  
  let product = catalog[productId]
  
  for i in 0..<cart.items.len:
    if int(cart.items[i].product.id) == int(productId):
      cart.items[i].quantity += qty
      return true
  
  cart.items.add(CartItem(product: product, quantity: qty, note: note))
  return true

proc removeFromCart(cart: var Cart, productId: ProductId): bool =
  for i in 0..<cart.items.len:
    if int(cart.items[i].product.id) == int(productId):
      cart.items.delete(i)
      return true
  return false

proc updateQuantity(cart: var Cart, productId: ProductId, qty: int): bool =
  if qty <= 0:
    return removeFromCart(cart, productId)
  
  for i in 0..<cart.items.len:
    if int(cart.items[i].product.id) == int(productId):
      cart.items[i].quantity = qty
      return true
  return false

proc applyCoupon(cart: var Cart, code: string): bool =
  let coupons = {"SAVE10": 10.0, "SAVE20": 20.0, "HALFOFF": 50.0}.toTable()
  
  if code in coupons:
    cart.coupon = some(code)
    cart.discountPct = coupons[code]
    return true
  return false

proc getSubtotal(cart: Cart): float =
  result = 0.0
  for item in cart.items:
    result += item.product.price * float(item.quantity)

proc getDiscount(cart: Cart): float =
  getSubtotal(cart) * cart.discountPct / 100.0

proc getTotal(cart: Cart): float =
  getSubtotal(cart) - getDiscount(cart)

proc getItemsByCategory(cart: Cart): Table[string, seq[CartItem]] =
  result = initTable[string, seq[CartItem]]()
  for item in cart.items:
    let cat = item.product.category
    if cat notin result:
      result[cat] = @[]
    result[cat].add(item)

proc printCartReceipt(cart: Cart) =
  echo "\n" & "═".repeat(50)
  echo "             Shopping Cart"
  echo "═".repeat(50)
  
  if cart.items.len == 0:
    echo "  Cart is empty"
    return
  
  let byCategory = getItemsByCategory(cart)
  
  for category in byCategory.keys.toSeq().sorted():
    echo fmt"\n  📦 {category}"
    echo "  " & "─".repeat(46)
    
    for item in byCategory[category]:
      let lineTotal = item.product.price * float(item.quantity)
      echo fmt"  {item.product.name:<28} {item.quantity:2}x ฿{item.product.price:8.2f}"
      echo fmt"  {'':28}    = ฿{lineTotal:8.2f}"
      if item.note.len > 0:
        echo fmt"  {'':28}    ({item.note})"
  
  echo "\n" & "─".repeat(50)
  echo fmt"  {'Subtotal':>40}: ฿{getSubtotal(cart):8.2f}"
  
  if cart.discountPct > 0:
    echo fmt"  {'Coupon: ' & cart.coupon.get():>40}: -฿{getDiscount(cart):7.2f}"
  
  echo "═".repeat(50)
  echo fmt"  {'TOTAL':>40}: ฿{getTotal(cart):8.2f}"
  echo "═".repeat(50)
  
  echo fmt"\n  Items: {cart.items.foldl(a + b.quantity, 0)}"
  if cart.coupon.isSome():
    echo fmt"  Coupon Applied: {cart.coupon.get()} ({cart.discountPct:.0f}% off)"

# ==============================
# Demo
# ==============================

proc main() =
  var cart = newCart()
  
  # Add items
  discard cart.addToCart(ProductId(1), 1, "For work")  # 1 Laptop
  discard cart.addToCart(ProductId(2), 2)               # 2 Mice
  discard cart.addToCart(ProductId(3), 1, "RGB variant")  # 1 Keyboard
  discard cart.addToCart(ProductId(5), 5)               # 5 Notebooks
  discard cart.addToCart(ProductId(6), 2)               # 2 Pen Sets
  
  cart.printCartReceipt()
  
  echo "\n--- Applying coupon SAVE20 ---"
  if cart.applyCoupon("SAVE20"):
    echo "✅ Coupon applied!"
  
  cart.printCartReceipt()
  
  echo "\n--- Removing laptop ---"
  discard cart.removeFromCart(ProductId(1))
  cart.printCartReceipt()

main()
```

---

## 📝 สรุป Part 06

| Step | หัวข้อ |
|------|--------|
| 56 | Arrays - fixed-size, indexing, slicing |
| 57 | Sequences - dynamic, add/remove/sort |
| 58 | Sequence algorithms |
| 59 | Tuples - named, destructuring |
| 60 | Sets - operations, HashSet |
| 61 | Tables (Hash Maps) |
| 62 | Advanced: Deque, HeapQueue, CircularBuffer |
| 63 | Data patterns: LRU cache, inverted index |
| 64 | Backend sequences patterns |
| 65 | Real-world: Shopping Cart |

---

**← [Part 05: Procedures](part_05_procedures.md) | [Part 07: Strings →](part_07_strings.md)**
