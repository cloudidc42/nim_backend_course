# Part 05: Procedures & Functions
## Steps 41-55: การเขียน Procedures ระดับมืออาชีพ

---

## 🎯 เป้าหมายของ Part นี้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
- เขียน procedures และ functions ทุกรูปแบบ
- ใช้ default parameters, named parameters
- เขียน overloaded procedures
- ใช้ closures และ first-class functions
- เข้าใจ method syntax
- สร้าง higher-order functions
- ใช้ variadic arguments

---

## Step 41: Procedures พื้นฐาน

```nim
# Procedure ที่ไม่มี return value
proc sayHello(name: string) =
  echo "สวัสดี, " & name & "!"

sayHello("Alice")  # สวัสดี, Alice!

# Procedure ที่มี return value
proc add(a, b: int): int =
  return a + b

echo add(5, 3)  # 8

# Implicit return (last expression)
proc multiply(a, b: int): int =
  a * b  # ไม่ต้องใช้ return

echo multiply(4, 5)  # 20

# Result variable
proc sumTo(n: int): int =
  result = 0  # result เป็น built-in สำหรับ return value
  for i in 1..n:
    result += i

echo sumTo(10)  # 55

# Multiple operations with result
proc stats(numbers: seq[float]): (float, float, float) =
  # Return (min, max, average)
  if numbers.len == 0:
    return (0.0, 0.0, 0.0)
  
  result[0] = numbers[0]  # min
  result[1] = numbers[0]  # max
  var sum = 0.0
  
  for n in numbers:
    if n < result[0]: result[0] = n
    if n > result[1]: result[1] = n
    sum += n
  
  result[2] = sum / float(numbers.len)

let (mn, mx, avg) = stats(@[3.0, 1.0, 4.0, 1.0, 5.0, 9.0, 2.0, 6.0])
echo fmt"Min: {mn}, Max: {mx}, Avg: {avg:.2f}"
```

---

## Step 42: Parameters ขั้นสูง

```nim
# Default parameters
proc greet(name: string, greeting: string = "สวัสดี"): string =
  return greeting & ", " & name & "!"

echo greet("Alice")                  # สวัสดี, Alice!
echo greet("Bob", "Hello")           # Hello, Bob!

# Named parameters
proc createUser(
  username: string,
  email: string,
  age: int = 0,
  isAdmin: bool = false,
  role: string = "member"
): string =
  var result = fmt"User: {username}, Email: {email}"
  if age > 0: result &= fmt", Age: {age}"
  if isAdmin: result &= " [ADMIN]"
  result &= fmt", Role: {role}"
  return result

# Named parameters สามารถใส่ได้ไม่เรียงลำดับ
echo createUser(
  username = "alice",
  email = "alice@example.com",
  isAdmin = true,
  age = 28
)

# Mutable parameters (var params)
proc increment(x: var int, by: int = 1) =
  x += by

var counter = 0
increment(counter)     # counter = 1
increment(counter, 5)  # counter = 6
echo counter  # 6

# Mutable reference (ref params)
type Node = ref object
  value: int
  next: Node

proc appendValue(head: var Node, value: int) =
  var current = head
  while current.next != nil:
    current = current.next
  current.next = Node(value: value)

# ส่ง procedure เป็น parameter
proc applyTwice(x: int, f: proc(n: int): int): int =
  return f(f(x))

echo applyTwice(3, proc(n: int): int = n * 2)  # 12 (3*2*2)
```

---

## Step 43: Overloading

```nim
import std/strformat

# Function overloading ตาม type
proc describe(x: int): string =
  fmt"Integer: {x}"

proc describe(x: float): string =
  fmt"Float: {x:.2f}"

proc describe(x: string): string =
  fmt"String: '{x}'"

proc describe(x: bool): string =
  fmt"Boolean: {x}"

echo describe(42)        # Integer: 42
echo describe(3.14)      # Float: 3.14
echo describe("hello")   # String: 'hello'
echo describe(true)      # Boolean: true

# Overloading ตามจำนวน parameters
proc connect(host: string): string =
  fmt"Connecting to {host}:80"

proc connect(host: string, port: int): string =
  fmt"Connecting to {host}:{port}"

proc connect(host: string, port: int, timeout: float): string =
  fmt"Connecting to {host}:{port} (timeout: {timeout}s)"

echo connect("localhost")
echo connect("localhost", 8080)
echo connect("localhost", 8080, 30.0)

# Operator overloading
type
  Vector2D = object
    x, y: float

proc `+`(a, b: Vector2D): Vector2D =
  Vector2D(x: a.x + b.x, y: a.y + b.y)

proc `-`(a, b: Vector2D): Vector2D =
  Vector2D(x: a.x - b.x, y: a.y - b.y)

proc `*`(v: Vector2D, scalar: float): Vector2D =
  Vector2D(x: v.x * scalar, y: v.y * scalar)

proc `$`(v: Vector2D): string =
  fmt"({v.x:.2f}, {v.y:.2f})"

proc `==`(a, b: Vector2D): bool =
  a.x == b.x and a.y == b.y

var v1 = Vector2D(x: 1.0, y: 2.0)
var v2 = Vector2D(x: 3.0, y: 4.0)

echo v1 + v2        # (4.00, 6.00)
echo v2 - v1        # (2.00, 2.00)
echo v1 * 2.5       # (2.50, 5.00)
echo v1 == v2       # false
```

---

## Step 44: Closures

```nim
# Closure พื้นฐาน
proc makeCounter(start: int = 0): proc(): int =
  var count = start
  return proc(): int =
    inc count
    return count

var counter1 = makeCounter()
var counter2 = makeCounter(10)

echo counter1()  # 1
echo counter1()  # 2
echo counter1()  # 3
echo counter2()  # 11
echo counter2()  # 12

# Closure กับ parameters
proc makeMultiplier(factor: int): proc(x: int): int =
  return proc(x: int): int = x * factor

var double = makeMultiplier(2)
var triple = makeMultiplier(3)

echo double(5)   # 10
echo triple(5)   # 15

# Closure ใน practical use - Rate Limiter
proc makeRateLimiter(maxRequests: int, perSeconds: float): proc(): bool =
  import std/times
  var requests: seq[float] = @[]
  let window = perSeconds
  
  return proc(): bool =
    let now = epochTime()
    # Remove old requests
    requests = requests.filterIt(now - it < window)
    
    if requests.len < maxRequests:
      requests.add(now)
      return true
    
    return false

var limiter = makeRateLimiter(5, 1.0)  # 5 requests per second

for i in 1..8:
  let allowed = limiter()
  echo fmt"Request {i}: {if allowed: \"✅ Allowed\" else: \"❌ Blocked\"}"
```

---

## Step 45: Higher-Order Functions

```nim
import std/sequtils, std/sugar

# Higher-order functions
proc compose[T, U, V](f: proc(x: U): V, g: proc(x: T): U): proc(x: T): V =
  return proc(x: T): V = f(g(x))

proc double(x: int): int = x * 2
proc addOne(x: int): int = x + 1
proc square(x: int): int = x * x

var doubleThenAddOne = compose(addOne, double)
var squareThenDouble = compose(double, square)

echo doubleThenAddOne(5)   # 11 (5*2+1)
echo squareThenDouble(3)   # 18 (3^2*2)

# Pipeline operator (|>)
template `|>`[T, U](x: T, f: proc(a: T): U): U = f(x)

echo 5 |> double |> addOne |> square   # ((5*2)+1)^2 = 121

# Partial application
proc partial[T, U, V](f: proc(a: T, b: U): V, a: T): proc(b: U): V =
  return proc(b: U): V = f(a, b)

proc power(base, exp: int): int =
  var result = 1
  for i in 1..exp:
    result *= base
  return result

var square2 = partial(power, 2)  # 2^x
var cube = partial(power, 3)     # 3^x... wait this is wrong

# Better partial application
proc makeAdder(n: int): proc(x: int): int =
  return proc(x: int): int = x + n

var add5 = makeAdder(5)
var add10 = makeAdder(10)

echo @[1, 2, 3].map(add5)   # @[6, 7, 8]
echo @[1, 2, 3].map(add10)  # @[11, 12, 13]

# Memoization
proc memoize[T, U](f: proc(x: T): U): proc(x: T): U =
  var cache = initTable[T, U]()
  return proc(x: T): U =
    if x notin cache:
      cache[x] = f(x)
    return cache[x]

var fib: proc(n: int): int
fib = memoize(proc(n: int): int =
  if n <= 1: return n
  return fib(n-1) + fib(n-2)
)

echo fib(40)  # 102334155 (computed fast with memoization)
```

---

## Step 46: Method Syntax (UFCS)

Nim ใช้ Universal Function Call Syntax (UFCS) - ทุก proc สามารถเรียกแบบ method ได้

```nim
import std/strutils, std/sequtils

# Procedure ปกติ
proc double(n: int): int = n * 2
proc isPositive(n: int): bool = n > 0
proc formatNumber(n: int, prefix: string = ""): string =
  prefix & $n

# เรียกแบบ UFCS (method-like syntax)
echo double(5)          # ปกติ
echo 5.double()         # UFCS
echo 5.double           # UFCS (ไม่ต้องมี parentheses)

# Chain method calls
echo 5.double.double     # 20 (5 -> 10 -> 20)
echo (-5).isPositive     # false

# เหมาะสำหรับ builder pattern
type
  QueryBuilder = object
    table: string
    conditions: seq[string]
    columns: seq[string]
    limitNum: int
    offsetNum: int

proc from(table: string): QueryBuilder =
  QueryBuilder(table: table, conditions: @[], columns: @["*"], limitNum: -1)

proc select(q: QueryBuilder, cols: varargs[string]): QueryBuilder =
  result = q
  result.columns = @cols

proc where(q: QueryBuilder, condition: string): QueryBuilder =
  result = q
  result.conditions.add(condition)

proc limit(q: QueryBuilder, n: int): QueryBuilder =
  result = q
  result.limitNum = n

proc offset(q: QueryBuilder, n: int): QueryBuilder =
  result = q
  result.offsetNum = n

proc build(q: QueryBuilder): string =
  var sql = "SELECT " & q.columns.join(", ")
  sql &= " FROM " & q.table
  
  if q.conditions.len > 0:
    sql &= " WHERE " & q.conditions.join(" AND ")
  
  if q.limitNum > 0:
    sql &= " LIMIT " & $q.limitNum
  
  if q.offsetNum > 0:
    sql &= " OFFSET " & $q.offsetNum
  
  return sql

# Fluent interface / method chaining
let query = from("users")
  .select("id", "name", "email")
  .where("age > 18")
  .where("is_active = true")
  .limit(10)
  .offset(20)
  .build()

echo query
# SELECT id, name, email FROM users WHERE age > 18 AND is_active = true LIMIT 10 OFFSET 20
```

---

## Step 47: Variadic Arguments

```nim
import std/strutils, std/sequtils

# varargs พื้นฐาน
proc sum(numbers: varargs[int]): int =
  result = 0
  for n in numbers:
    result += n

echo sum(1, 2, 3, 4, 5)       # 15
echo sum(10, 20)               # 30
echo sum()                     # 0

# varargs กับ type
proc printAll(values: varargs[string, `$`]) =
  # `$` converts each arg to string automatically
  for v in values:
    write(stdout, v & " ")
  echo ""

printAll("hello", 42, 3.14, true)  # hello 42 3.14 true

# varargs ส่งต่อ
proc myPrintf(fmt: string, args: varargs[string, `$`]) =
  var result = fmt
  for arg in args:
    let pos = result.find("{}")
    if pos >= 0:
      result = result[0..<pos] & arg & result[pos+2..^1]
  echo result

myPrintf("Hello {} from {}!", "World", "Nim")  # Hello World from Nim!

# Collect varargs เป็น seq
proc joinWith(sep: string, items: varargs[string]): string =
  @items.join(sep)

echo joinWith(", ", "apple", "banana", "cherry")  # apple, banana, cherry

# Template กับ varargs
template log(level: string, msgs: varargs[string, `$`]) =
  let combined = @msgs.join(" ")
  echo fmt"[{level}] {combined}"

log("INFO", "User", "alice", "logged in at", 12, "noon")
log("ERROR", "Connection failed: ", 500)
```

---

## Step 48: Recursive Procedures

```nim
import std/math

# Factorial (recursive)
proc factorial(n: int): int =
  if n <= 1: return 1
  return n * factorial(n - 1)

echo factorial(10)  # 3628800

# Fibonacci (recursive - slow)
proc fib(n: int): int =
  if n <= 1: return n
  return fib(n-1) + fib(n-2)

# Fibonacci (iterative - fast)
proc fibFast(n: int): int =
  if n <= 1: return n
  var a = 0
  var b = 1
  for i in 2..n:
    let c = a + b
    a = b
    b = c
  return b

echo fibFast(50)  # 12586269025

# Binary search (recursive)
proc binarySearch(arr: seq[int], target, low, high: int): int =
  if low > high:
    return -1
  
  let mid = (low + high) div 2
  
  if arr[mid] == target:
    return mid
  elif arr[mid] < target:
    return binarySearch(arr, target, mid + 1, high)
  else:
    return binarySearch(arr, target, low, mid - 1)

var sorted = @[1, 3, 5, 7, 9, 11, 13, 15, 17, 19]
echo binarySearch(sorted, 7, 0, sorted.len - 1)   # 3 (index)
echo binarySearch(sorted, 10, 0, sorted.len - 1)  # -1 (not found)

# Tree traversal (recursive)
type
  TreeNode = ref object
    value: int
    left, right: TreeNode

proc newNode(v: int): TreeNode =
  TreeNode(value: v, left: nil, right: nil)

proc insert(node: TreeNode, value: int): TreeNode =
  if node == nil:
    return newNode(value)
  
  if value < node.value:
    result = node
    result.left = insert(node.left, value)
  elif value > node.value:
    result = node
    result.right = insert(node.right, value)
  else:
    return node  # duplicate

proc inorder(node: TreeNode): seq[int] =
  if node == nil: return @[]
  result = inorder(node.left)
  result.add(node.value)
  result.add(inorder(node.right))

# Build BST
var root: TreeNode = nil
for v in [5, 3, 7, 1, 4, 6, 8]:
  root = insert(root, v)

echo inorder(root)  # @[1, 3, 4, 5, 6, 7, 8]
```

---

## Step 49: Exception Handling ใน Procedures

```nim
import std/strutils, std/options

# Procedure ที่ raise exception
proc divideStrict(a, b: float): float =
  if b == 0.0:
    raise newException(DivByZeroDefect, "Cannot divide by zero")
  return a / b

# การจัดการ exception
try:
  echo divideStrict(10.0, 2.0)   # 5.0
  echo divideStrict(10.0, 0.0)   # raises!
except DivByZeroDefect as e:
  echo "Error: " & e.msg

# Procedure ที่ return Option แทน exception
proc divide(a, b: float): Option[float] =
  if b == 0.0:
    return none(float)
  return some(a / b)

let r1 = divide(10.0, 2.0)
let r2 = divide(10.0, 0.0)

echo r1.isSome()   # true
echo r1.get()      # 5.0
echo r2.isNone()   # true

# Custom exceptions
type
  AppError = object of CatchableError
  ValidationError = object of AppError
  DatabaseError = object of AppError
  AuthError = object of AppError

proc validateAge(age: int) =
  if age < 0:
    raise newException(ValidationError, "Age cannot be negative")
  if age > 150:
    raise newException(ValidationError, "Age seems unrealistic")

proc authenticate(token: string) =
  if token.len == 0:
    raise newException(AuthError, "Token is required")
  if token != "valid-token":
    raise newException(AuthError, "Invalid token")

# Handle multiple exception types
proc processRequest(token: string, age: int) =
  try:
    authenticate(token)
    validateAge(age)
    echo fmt"Request processed for age {age}"
  except AuthError as e:
    echo "Auth Error: " & e.msg
  except ValidationError as e:
    echo "Validation Error: " & e.msg
  except CatchableError as e:
    echo "General Error: " & e.msg

processRequest("", 25)          # Auth Error
processRequest("valid-token", -5)  # Validation Error
processRequest("valid-token", 25)  # Success
```

---

## Step 50: Procedures สำหรับ Backend

```nim
import std/times, std/strutils, std/options, std/tables, std/sequtils

# ==============================
# Utility Procedures
# ==============================

# Slug generator
proc toSlug(title: string): string =
  result = title.toLower()
  result = result.replace(" ", "-")
  var clean = newString(0)
  for c in result:
    if c.isAlpha() or c.isDigit() or c == '-':
      clean.add(c)
  # Remove multiple dashes
  while "--" in clean:
    clean = clean.replace("--", "-")
  # Remove leading/trailing dashes
  return clean.strip(chars = {'-'})

echo toSlug("Hello World! This is a Test")
# hello-world-this-is-a-test

# Truncate with ellipsis
proc truncate(s: string, maxLen: int, ellipsis: string = "..."): string =
  if s.len <= maxLen:
    return s
  return s[0..<maxLen - ellipsis.len] & ellipsis

echo truncate("Hello World", 8)        # Hello...
echo truncate("Hi", 10)               # Hi

# Mask sensitive data
proc maskEmail(email: string): string =
  let atPos = email.find('@')
  if atPos < 2: return "***"
  
  let username = email[0..<atPos]
  let domain = email[atPos..^1]
  let visible = username[0..1]
  let masked = "*".repeat(username.len - 2)
  
  return visible & masked & domain

echo maskEmail("alice@example.com")   # al***@example.com
echo maskEmail("bob@test.com")        # bo@test.com

# Format duration
proc formatDuration(seconds: float): string =
  let secs = int(seconds)
  if secs < 60:
    return fmt"{secs}s"
  elif secs < 3600:
    return fmt"{secs div 60}m {secs mod 60}s"
  elif secs < 86400:
    return fmt"{secs div 3600}h {(secs mod 3600) div 60}m"
  else:
    return fmt"{secs div 86400}d {(secs mod 86400) div 3600}h"

echo formatDuration(45.0)      # 45s
echo formatDuration(125.0)     # 2m 5s
echo formatDuration(3725.0)    # 1h 2m
echo formatDuration(90125.0)   # 1d 1h

# Pagination calculator
type
  PaginationInfo = object
    currentPage: int
    totalPages: int
    totalItems: int
    itemsPerPage: int
    startItem: int
    endItem: int
    hasPrev: bool
    hasNext: bool

proc calculatePagination(
  totalItems: int,
  currentPage: int,
  itemsPerPage: int = 20
): PaginationInfo =
  let totalPages = max(1, (totalItems + itemsPerPage - 1) div itemsPerPage)
  let safePage = clamp(currentPage, 1, totalPages)
  let startItem = (safePage - 1) * itemsPerPage + 1
  let endItem = min(safePage * itemsPerPage, totalItems)
  
  return PaginationInfo(
    currentPage: safePage,
    totalPages: totalPages,
    totalItems: totalItems,
    itemsPerPage: itemsPerPage,
    startItem: startItem,
    endItem: endItem,
    hasPrev: safePage > 1,
    hasNext: safePage < totalPages
  )

let pagination = calculatePagination(totalItems = 235, currentPage = 3)
echo fmt"Page {pagination.currentPage}/{pagination.totalPages}"
echo fmt"Showing items {pagination.startItem}-{pagination.endItem} of {pagination.totalItems}"
echo fmt"Prev: {pagination.hasPrev}, Next: {pagination.hasNext}"
```

---

## Step 51: Callback Patterns

```nim
import std/times, std/asyncdispatch

# Callback type definition
type
  OnSuccess = proc(data: string)
  OnError = proc(err: string)
  OnComplete = proc()

# Procedure ที่รับ callbacks
proc fetchData(
  url: string,
  onSuccess: OnSuccess,
  onError: OnError = nil,
  onComplete: OnComplete = nil
) =
  # Simulate async fetch
  if url.len == 0:
    if onError != nil:
      onError("URL cannot be empty")
  else:
    onSuccess("Data from " & url)
  
  if onComplete != nil:
    onComplete()

fetchData(
  "https://api.example.com/users",
  onSuccess = proc(data: string) = echo "Got: " & data,
  onError = proc(err: string) = echo "Error: " & err,
  onComplete = proc() = echo "Done!"
)

# Event system
type
  EventHandler = proc(event: string, data: string)
  EventEmitter = object
    handlers: Table[string, seq[EventHandler]]

proc newEventEmitter(): EventEmitter =
  EventEmitter(handlers: initTable[string, seq[EventHandler]]())

proc on(emitter: var EventEmitter, event: string, handler: EventHandler) =
  if event notin emitter.handlers:
    emitter.handlers[event] = @[]
  emitter.handlers[event].add(handler)

proc emit(emitter: EventEmitter, event: string, data: string = "") =
  if event in emitter.handlers:
    for handler in emitter.handlers[event]:
      handler(event, data)

var events = newEventEmitter()

events.on("user:login", proc(event, data: string) =
  echo fmt"[Log] {event}: {data}"
)
events.on("user:login", proc(event, data: string) =
  echo fmt"[Analytics] Track: {event}"
)
events.on("user:logout", proc(event, data: string) =
  echo fmt"[Session] Clear session for {data}"
)

events.emit("user:login", "alice")
events.emit("user:logout", "alice")
```

---

## Step 52: Functional Composition

```nim
import std/sequtils, std/strutils

# Function composition
proc compose2[A, B, C](f: proc(x: B): C, g: proc(x: A): B): proc(x: A): C =
  return proc(x: A): C = f(g(x))

# String pipeline
let normalize = compose2(
  proc(s: string): string = s.toLower(),
  proc(s: string): string = s.strip()
)

let process = compose2(
  proc(s: string): string = s.replace(" ", "_"),
  normalize
)

echo normalize("  Hello World  ")   # hello world
echo process("  Hello World  ")     # hello_world

# Data transformation pipeline
type
  RawUser = object
    name: string
    email: string
    age: string  # from JSON, might be string

  ProcessedUser = object
    name: string
    email: string
    age: int
    slug: string

proc parseRawUser(raw: RawUser): ProcessedUser =
  ProcessedUser(
    name: raw.name.strip().toLower().capitalizeAscii(),
    email: raw.email.strip().toLower(),
    age: try: parseInt(raw.age) except: 0,
    slug: raw.name.strip().toLower().replace(" ", "-")
  )

let rawUsers = @[
  RawUser(name: "  alice smith  ", email: "ALICE@EXAMPLE.COM", age: "28"),
  RawUser(name: "bob jones", email: "bob@test.com", age: "thirty"),
]

for raw in rawUsers:
  let processed = parseRawUser(raw)
  echo fmt"{processed.name} ({processed.email}) - age: {processed.age}"
```

---

## Step 53: Procedures ใน Real-World Context

```nim
import std/times, std/strutils, std/options, std/tables, std/sequtils, std/algorithm

# ==============================
# Order Processing System
# ==============================

type
  ProductId = distinct int
  OrderId = distinct int
  CustomerId = distinct int
  
  Product = object
    id: ProductId
    name: string
    price: float
    stock: int
  
  OrderItem = object
    product: Product
    quantity: int
  
  Order = object
    id: OrderId
    customerId: CustomerId
    items: seq[OrderItem]
    createdAt: DateTime
    status: string
    discount: float
  
  OrderSummary = object
    orderId: OrderId
    itemCount: int
    subtotal: float
    discountAmount: float
    total: float
    formattedTotal: string

# ==============================
# Business Logic Procedures
# ==============================

proc createOrder(customerId: CustomerId): Order =
  Order(
    id: OrderId(int(epochTime() * 1000)),
    customerId: customerId,
    items: @[],
    createdAt: now(),
    status: "pending",
    discount: 0.0
  )

proc addItem(order: var Order, product: Product, quantity: int): bool =
  if product.stock < quantity:
    return false
  
  # Check if product already in order
  for i in 0..<order.items.len:
    if int(order.items[i].product.id) == int(product.id):
      order.items[i].quantity += quantity
      return true
  
  order.items.add(OrderItem(product: product, quantity: quantity))
  return true

proc removeItem(order: var Order, productId: ProductId): bool =
  for i in 0..<order.items.len:
    if int(order.items[i].product.id) == int(productId):
      order.items.delete(i)
      return true
  return false

proc calculateSubtotal(order: Order): float =
  result = 0.0
  for item in order.items:
    result += item.product.price * float(item.quantity)

proc applyDiscount(order: var Order, discountPct: float) =
  order.discount = clamp(discountPct, 0.0, 100.0)

proc calculateTotal(order: Order): float =
  let subtotal = calculateSubtotal(order)
  let discountAmount = subtotal * order.discount / 100.0
  return subtotal - discountAmount

proc summarize(order: Order): OrderSummary =
  let subtotal = calculateSubtotal(order)
  let discountAmount = subtotal * order.discount / 100.0
  let total = subtotal - discountAmount
  
  OrderSummary(
    orderId: order.id,
    itemCount: order.items.foldl(a + b.quantity, 0),
    subtotal: subtotal,
    discountAmount: discountAmount,
    total: total,
    formattedTotal: fmt"฿{total:,.2f}"
  )

proc printReceipt(order: Order) =
  let summary = summarize(order)
  
  echo "\n" & "═".repeat(40)
  echo "         ใบเสร็จรับเงิน"
  echo "═".repeat(40)
  echo fmt"Order ID: {int(summary.orderId)}"
  echo fmt"Date:     {order.createdAt.format(\"dd/MM/yyyy HH:mm\")}"
  echo "─".repeat(40)
  
  for item in order.items:
    let lineTotal = item.product.price * float(item.quantity)
    echo fmt"  {item.product.name:<20} {item.quantity:2}x ฿{item.product.price:7.2f}"
    echo fmt"  {'':20}    ฿{lineTotal:7.2f}"
  
  echo "─".repeat(40)
  echo fmt"{'Subtotal':>30}: ฿{summary.subtotal:8.2f}"
  
  if summary.discountAmount > 0:
    echo fmt"{'Discount (' & $order.discount & '%)':>30}: -฿{summary.discountAmount:7.2f}"
  
  echo "═".repeat(40)
  echo fmt"{'TOTAL':>30}: ฿{summary.total:8.2f}"
  echo "═".repeat(40)

# ==============================
# Demo
# ==============================

proc main() =
  let products = @[
    Product(id: ProductId(1), name: "Laptop", price: 35000.0, stock: 10),
    Product(id: ProductId(2), name: "Mouse", price: 990.0, stock: 50),
    Product(id: ProductId(3), name: "Keyboard", price: 2500.0, stock: 30),
    Product(id: ProductId(4), name: "Monitor", price: 12000.0, stock: 5),
  ]
  
  var order = createOrder(CustomerId(1001))
  
  echo "Creating order..."
  
  discard addItem(order, products[0], 1)   # 1 Laptop
  discard addItem(order, products[1], 2)   # 2 Mice
  discard addItem(order, products[2], 1)   # 1 Keyboard
  
  # Try to add more than stock
  let added = addItem(order, products[3], 10)  # Only 5 in stock
  echo fmt"Add 10 monitors: {if added: \"Success\" else: \"Failed (insufficient stock)\"}"
  
  discard addItem(order, products[3], 3)  # 3 monitors - OK
  
  applyDiscount(order, 10.0)  # 10% discount
  
  printReceipt(order)

main()
```

---

## 📝 สรุป Part 05

| Step | หัวข้อ |
|------|--------|
| 41 | Procedures พื้นฐาน, result variable |
| 42 | Default params, named params, var params |
| 43 | Overloading, operator overloading |
| 44 | Closures, stateful functions |
| 45 | Higher-order functions, compose, memoize |
| 46 | UFCS, method chaining, builder pattern |
| 47 | Variadic arguments |
| 48 | Recursive procedures |
| 49 | Exception handling |
| 50 | Backend utility procedures |
| 51 | Callbacks, event systems |
| 52 | Functional composition |
| 53 | Real-world order processing |

---

**← [Part 04: Loops](part_04_loops.md) | [Part 06: Arrays & Sequences →](part_06_sequences.md)**
