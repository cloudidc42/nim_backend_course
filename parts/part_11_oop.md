# Part 11: Object-Oriented Programming ใน Nim
## Steps 131-145: OOP สไตล์ Nim

---

## 🎯 เป้าหมายของ Part นี้

- ใช้ Objects และ Inheritance
- Method dispatch
- Interfaces ด้วย concepts
- Mixins และ composition
- Design patterns

---

## Step 131: Objects พื้นฐาน

```nim
import std/strformat

type
  Animal = object of RootObj
    name: string
    age: int

  Dog = object of Animal
    breed: string

  Cat = object of Animal
    isIndoor: bool

# Methods (procedures ที่ accept object)
proc speak(a: Animal): string =
  fmt"{a.name} makes a sound"

proc speak(d: Dog): string =
  fmt"{d.name} barks: Woof!"

proc speak(c: Cat): string =
  fmt"{c.name} meows: Meow~"

proc describe(a: Animal): string =
  fmt"Name: {a.name}, Age: {a.age}"

# Dynamic dispatch ต้องใช้ ref
type
  AnimalRef = ref object of RootObj
    name: string
    age: int

  DogRef = ref object of AnimalRef
    breed: string

method makeSound(a: AnimalRef): string {.base.} =
  "..."

method makeSound(d: DogRef): string =
  "Woof! (breed: " & d.breed & ")"

# Usage
var dog = DogRef(name: "Rex", age: 3, breed: "Labrador")
var animal: AnimalRef = dog  # upcasting

echo animal.makeSound()  # Woof! (dynamic dispatch works)
echo dog.makeSound()     # Woof!
```

---

## Step 132: Inheritance และ Polymorphism

```nim
import std/strformat, std/sequtils

type
  Shape = ref object of RootObj
    color: string
    filled: bool

  Circle = ref object of Shape
    radius: float

  Rectangle = ref object of Shape
    width, height: float

  Triangle = ref object of Shape
    base, height: float

# Constructor procs
proc newCircle(radius: float, color: string = "black", filled: bool = false): Circle =
  Circle(radius: radius, color: color, filled: filled)

proc newRect(w, h: float, color: string = "black", filled: bool = false): Rectangle =
  Rectangle(width: w, height: h, color: color, filled: filled)

# Abstract methods
method area(s: Shape): float {.base.} =
  raise newException(CatchableError, "Not implemented")

method perimeter(s: Shape): float {.base.} =
  raise newException(CatchableError, "Not implemented")

method draw(s: Shape): string {.base.} =
  fmt"Drawing shape ({s.color})"

# Concrete implementations
method area(c: Circle): float =
  3.14159 * c.radius * c.radius

method perimeter(c: Circle): float =
  2 * 3.14159 * c.radius

method draw(c: Circle): string =
  fmt"Circle(r={c.radius:.1f}, color={c.color}, filled={c.filled})"

method area(r: Rectangle): float =
  r.width * r.height

method perimeter(r: Rectangle): float =
  2 * (r.width + r.height)

method draw(r: Rectangle): string =
  fmt"Rect({r.width:.1f}x{r.height:.1f}, color={r.color})"

method area(t: Triangle): float =
  0.5 * t.base * t.height

# Polymorphism in action
var shapes: seq[Shape] = @[
  newCircle(5.0, "red", true),
  newRect(4.0, 6.0, "blue"),
  newCircle(3.0, "green"),
  newRect(10.0, 2.0, "black", true),
]

echo "=== Shape Report ==="
var totalArea = 0.0

for shape in shapes:
  echo fmt"  {shape.draw()}"
  echo fmt"  Area: {shape.area():.2f}"
  totalArea += shape.area()

echo fmt"\nTotal area: {totalArea:.2f}"
echo fmt"Largest: {shapes.maxByIt(it.area()).draw()}"
```

---

## Step 133: Composition Over Inheritance

```nim
import std/strformat, std/tables

# Component/Mixin approach
type
  Serializable = concept s
    s.toJson() is string

  Validatable = concept v
    v.validate() is seq[string]

  Timestamped = object
    createdAt: string
    updatedAt: string

proc touch(t: var Timestamped) =
  t.updatedAt = "2024-01-01"  # simplified

# Entity with composition
type
  EntityBase = object
    id: int
    timestamps: Timestamped

  User3 = object
    entity: EntityBase
    name: string
    email: string
    role: string

proc newUser3(id: int, name, email: string): User3 =
  User3(
    entity: EntityBase(
      id: id,
      timestamps: Timestamped(createdAt: "2024-01-01", updatedAt: "2024-01-01")
    ),
    name: name,
    email: email,
    role: "user"
  )

proc getId(u: User3): int = u.entity.id
proc getCreatedAt(u: User3): string = u.entity.timestamps.createdAt

proc toJson(u: User3): string =
  fmt"""{"{"}"id": {u.getId()}, "name": "{u.name}", "email": "{u.email}"{"}"}"""

proc validate(u: User3): seq[string] =
  result = @[]
  if u.name.len < 2:
    result.add("Name too short")
  if '@' notin u.email:
    result.add("Invalid email")

let user = newUser3(1, "Alice", "alice@example.com")
echo user.toJson()
echo user.validate()  # @[] empty = valid

# Builder Pattern
type
  QueryBuilder2 = object
    table: string
    conditions: seq[string]
    orderBy: string
    limitVal: int
    offsetVal: int
    columns: seq[string]

proc from2(table: string): QueryBuilder2 =
  QueryBuilder2(table: table, columns: @["*"], limitVal: -1)

proc select2(qb: QueryBuilder2, cols: varargs[string]): QueryBuilder2 =
  result = qb
  result.columns = @cols

proc where2(qb: QueryBuilder2, cond: string): QueryBuilder2 =
  result = qb
  result.conditions.add(cond)

proc order2(qb: QueryBuilder2, col: string): QueryBuilder2 =
  result = qb
  result.orderBy = col

proc limit2(qb: QueryBuilder2, n: int): QueryBuilder2 =
  result = qb
  result.limitVal = n

proc offset2(qb: QueryBuilder2, n: int): QueryBuilder2 =
  result = qb
  result.offsetVal = n

proc build2(qb: QueryBuilder2): string =
  result = "SELECT " & qb.columns.join(", ")
  result &= " FROM " & qb.table
  
  if qb.conditions.len > 0:
    result &= " WHERE " & qb.conditions.join(" AND ")
  
  if qb.orderBy.len > 0:
    result &= " ORDER BY " & qb.orderBy
  
  if qb.limitVal > 0:
    result &= " LIMIT " & $qb.limitVal
  
  if qb.offsetVal > 0:
    result &= " OFFSET " & $qb.offsetVal

let query = from2("users")
  .select2("id", "name", "email")
  .where2("role = 'admin'")
  .where2("is_active = true")
  .order2("name ASC")
  .limit2(10)
  .offset2(20)
  .build2()

echo query
# SELECT id, name, email FROM users WHERE role = 'admin' AND is_active = true ORDER BY name ASC LIMIT 10 OFFSET 20
```

---

## Step 134: Design Patterns

```nim
import std/tables, std/sequtils

# ==============================
# Singleton Pattern
# ==============================

type
  AppState = ref object
    config: Table[string, string]
    running: bool

var appStateInstance: AppState = nil

proc getAppState(): AppState =
  if appStateInstance.isNil:
    appStateInstance = AppState(
      config: initTable[string, string](),
      running: false
    )
  return appStateInstance

# Usage
let state1 = getAppState()
let state2 = getAppState()
state1.running = true
echo state2.running  # true (same instance)

# ==============================
# Observer Pattern
# ==============================

type
  EventName = string
  EventData = Table[string, string]
  EventCallback = proc(data: EventData)
  
  EventBus = object
    listeners: Table[EventName, seq[EventCallback]]

proc newEventBus(): EventBus =
  EventBus(listeners: initTable[EventName, seq[EventCallback]]())

proc on(bus: var EventBus, event: EventName, cb: EventCallback) =
  if event notin bus.listeners:
    bus.listeners[event] = @[]
  bus.listeners[event].add(cb)

proc emit(bus: EventBus, event: EventName, data: EventData = initTable[string, string]()) =
  if event in bus.listeners:
    for cb in bus.listeners[event]:
      cb(data)

var bus = newEventBus()

bus.on("user.created", proc(data: EventData) =
  echo "Email: Sending welcome email to " & data.getOrDefault("email", "")
)

bus.on("user.created", proc(data: EventData) =
  echo "Analytics: New user registered - " & data.getOrDefault("name", "")
)

bus.emit("user.created", {"name": "Alice", "email": "alice@example.com"}.toTable())

# ==============================
# Strategy Pattern
# ==============================

type
  SortStrategy[T] = proc(items: seq[T]): seq[T]

proc bubbleSort[T](items: seq[T]): seq[T] =
  result = items
  for i in 0..<result.len:
    for j in 0..<result.len - i - 1:
      if result[j] > result[j+1]:
        swap(result[j], result[j+1])

proc quickSortHelper[T](items: var seq[T], lo, hi: int) =
  if lo < hi:
    let pivot = items[hi]
    var i = lo - 1
    for j in lo..<hi:
      if items[j] <= pivot:
        inc i
        swap(items[i], items[j])
    swap(items[i+1], items[hi])
    let pi = i + 1
    quickSortHelper(items, lo, pi - 1)
    quickSortHelper(items, pi + 1, hi)

proc myQuickSort[T](items: seq[T]): seq[T] =
  result = items
  if result.len > 1:
    quickSortHelper(result, 0, result.len - 1)

type
  Sorter[T] = object
    strategy: SortStrategy[T]

proc sort[T](sorter: Sorter[T], items: seq[T]): seq[T] =
  sorter.strategy(items)

var sorter = Sorter[int](strategy: myQuickSort[int])
echo sorter.sort(@[5, 2, 8, 1, 9, 3])  # @[1, 2, 3, 5, 8, 9]
```

---

## Step 135-145: Real-World OOP - HTTP Middleware System

```nim
# middleware_system.nim

import std/tables, std/strformat, std/times, std/strutils, std/sequtils

type
  # Request/Response types
  HttpMethod = enum
    GET, POST, PUT, DELETE, PATCH

  Headers = Table[string, string]

  Request = object
    httpMethod: HttpMethod
    path: string
    headers: Headers
    body: string
    params: Table[string, string]
    query: Table[string, string]

  Response = object
    statusCode: int
    headers: Headers
    body: string

  Context = ref object
    request: Request
    response: Response
    data: Table[string, string]  # shared data between middlewares
    stopped: bool

  MiddlewareFunc = proc(ctx: Context, next: proc())

# ==============================
# Context constructors
# ==============================

proc newContext(req: Request): Context =
  Context(
    request: req,
    response: Response(
      statusCode: 200,
      headers: {"Content-Type": "application/json"}.toTable()
    ),
    data: initTable[string, string](),
    stopped: false
  )

proc json(ctx: Context, data: string, status: int = 200) =
  ctx.response.statusCode = status
  ctx.response.body = data
  ctx.stopped = true

proc text(ctx: Context, data: string, status: int = 200) =
  ctx.response.headers["Content-Type"] = "text/plain"
  ctx.response.statusCode = status
  ctx.response.body = data
  ctx.stopped = true

# ==============================
# Built-in Middlewares
# ==============================

# Logging middleware
proc loggerMiddleware(): MiddlewareFunc =
  return proc(ctx: Context, next: proc()) =
    let start = now()
    next()
    let elapsed = (now() - start).inMilliseconds
    echo fmt"[{now().format(\"HH:mm:ss\")}] {ctx.request.httpMethod} {ctx.request.path} → {ctx.response.statusCode} ({elapsed}ms)"

# CORS middleware
proc corsMiddleware(origins: seq[string] = @["*"]): MiddlewareFunc =
  return proc(ctx: Context, next: proc()) =
    ctx.response.headers["Access-Control-Allow-Origin"] = origins.join(",")
    ctx.response.headers["Access-Control-Allow-Methods"] = "GET, POST, PUT, DELETE"
    ctx.response.headers["Access-Control-Allow-Headers"] = "Content-Type, Authorization"
    next()

# Auth middleware
proc authMiddleware(secretKey: string): MiddlewareFunc =
  return proc(ctx: Context, next: proc()) =
    let authHeader = ctx.request.headers.getOrDefault("Authorization", "")
    
    if authHeader.len == 0:
      ctx.json("""{"error": "Authentication required"}""", 401)
      return
    
    if not authHeader.startsWith("Bearer "):
      ctx.json("""{"error": "Invalid token format"}""", 401)
      return
    
    let token = authHeader[7..^1]
    # Simplified token validation
    if token == "valid-token-123":
      ctx.data["userId"] = "42"
      ctx.data["role"] = "admin"
      next()
    else:
      ctx.json("""{"error": "Invalid token"}""", 401)

# Rate limiter
type
  RateLimiter = object
    requests: Table[string, seq[float]]
    maxRequests: int
    windowSeconds: int

var rateLimiter = RateLimiter(
  requests: initTable[string, seq[float]](),
  maxRequests: 100,
  windowSeconds: 60
)

proc rateLimitMiddleware(): MiddlewareFunc =
  return proc(ctx: Context, next: proc()) =
    let clientId = ctx.request.headers.getOrDefault("X-Client-ID", "anonymous")
    let now2 = epochTime()
    
    if clientId notin rateLimiter.requests:
      rateLimiter.requests[clientId] = @[]
    
    # Clean old requests
    rateLimiter.requests[clientId] = rateLimiter.requests[clientId]
      .filterIt(now2 - it < float(rateLimiter.windowSeconds))
    
    if rateLimiter.requests[clientId].len >= rateLimiter.maxRequests:
      ctx.response.headers["Retry-After"] = $rateLimiter.windowSeconds
      ctx.json("""{"error": "Rate limit exceeded"}""", 429)
      return
    
    rateLimiter.requests[clientId].add(now2)
    next()

# Validation middleware factory
proc validateBody(required: seq[string]): MiddlewareFunc =
  return proc(ctx: Context, next: proc()) =
    import std/json
    
    if ctx.request.body.len == 0:
      ctx.json("""{"error": "Request body required"}""", 400)
      return
    
    try:
      let body = parseJson(ctx.request.body)
      var missing: seq[string] = @[]
      
      for field in required:
        if body{field}.isNil or body[field].kind == JNull:
          missing.add(field)
      
      if missing.len > 0:
        ctx.json(fmt"""{"{"}"error": "Missing required fields", "fields": {missing}{"}"}""", 400)
        return
      
      next()
    except JsonParsingError:
      ctx.json("""{"error": "Invalid JSON body"}""", 400)

# ==============================
# Router
# ==============================

type
  RouteHandler = proc(ctx: Context)
  
  Route = object
    httpMethod: HttpMethod
    pattern: string
    handler: RouteHandler
    middlewares: seq[MiddlewareFunc]

  Router = object
    routes: seq[Route]
    globalMiddlewares: seq[MiddlewareFunc]

proc newRouter(): Router =
  Router(routes: @[], globalMiddlewares: @[])

proc use(router: var Router, mw: MiddlewareFunc) =
  router.globalMiddlewares.add(mw)

proc addRoute(router: var Router, m: HttpMethod, pattern: string,
              handler: RouteHandler, middlewares: seq[MiddlewareFunc] = @[]) =
  router.routes.add(Route(
    httpMethod: m, pattern: pattern, handler: handler, middlewares: middlewares
  ))

proc get(router: var Router, pattern: string, handler: RouteHandler,
         middlewares: seq[MiddlewareFunc] = @[]) =
  router.addRoute(GET, pattern, handler, middlewares)

proc post(router: var Router, pattern: string, handler: RouteHandler,
          middlewares: seq[MiddlewareFunc] = @[]) =
  router.addRoute(POST, pattern, handler, middlewares)

proc matchRoute(router: Router, m: HttpMethod, path: string): Option[(Route, Table[string, string])] =
  for route in router.routes:
    if route.httpMethod != m: continue
    
    let patternParts = route.pattern.strip(chars = {'/'}).split('/')
    let pathParts = path.strip(chars = {'/'}).split('/')
    
    if patternParts.len != pathParts.len: continue
    
    var params = initTable[string, string]()
    var matched = true
    
    for i in 0..<patternParts.len:
      if patternParts[i].startsWith(':'):
        params[patternParts[i][1..^1]] = pathParts[i]
      elif patternParts[i] != pathParts[i]:
        matched = false
        break
    
    if matched:
      return some((route, params))
  
  return none((Route, Table[string, string]))

proc handle(router: Router, ctx: Context) =
  let matchResult = router.matchRoute(ctx.request.httpMethod, ctx.request.path)
  
  if matchResult.isNone:
    ctx.json("""{"error": "Route not found"}""", 404)
    return
  
  let (route, params) = matchResult.get()
  ctx.request.params = params
  
  # Build middleware chain
  var allMiddlewares = router.globalMiddlewares & route.middlewares
  var handler = route.handler
  
  proc runChain(idx: int) =
    if ctx.stopped: return
    if idx >= allMiddlewares.len:
      handler(ctx)
    else:
      allMiddlewares[idx](ctx, proc() = runChain(idx + 1))
  
  runChain(0)

# ==============================
# Demo Application
# ==============================

proc main() =
  var router = newRouter()
  
  # Global middlewares
  router.use(loggerMiddleware())
  router.use(corsMiddleware(@["https://example.com"]))
  router.use(rateLimitMiddleware())
  
  # Routes
  router.get("/health", proc(ctx: Context) =
    ctx.json("""{"status": "ok"}""")
  )
  
  router.get("/users", proc(ctx: Context) =
    let userId = ctx.data.getOrDefault("userId", "unknown")
    ctx.json(fmt"""[{{"id": 1, "name": "Alice"}}, {{"id": 2, "name": "Bob"}}]""")
  , @[authMiddleware("my-secret")])
  
  router.post("/users", proc(ctx: Context) =
    ctx.json("""{"id": 3, "name": "Charlie", "created": true}""", 201)
  , @[authMiddleware("my-secret"), validateBody(@["name", "email"])])
  
  router.get("/users/:id", proc(ctx: Context) =
    let id = ctx.request.params.getOrDefault("id", "")
    ctx.json(fmt"""{{"id": {id}, "name": "User #{id}"}}""")
  , @[authMiddleware("my-secret")])
  
  # Test requests
  echo "=== Middleware System Demo ==="
  
  # Test 1: Health check (no auth needed)
  var req1 = Request(httpMethod: GET, path: "/health",
                     headers: initTable[string, string](), body: "")
  var ctx1 = newContext(req1)
  router.handle(ctx1)
  echo "Body: " & ctx1.response.body
  
  # Test 2: Protected route without auth
  var req2 = Request(httpMethod: GET, path: "/users",
                     headers: initTable[string, string](), body: "")
  var ctx2 = newContext(req2)
  router.handle(ctx2)
  echo "Status: " & $ctx2.response.statusCode  # 401
  
  # Test 3: Protected route with valid token
  var req3 = Request(
    httpMethod: GET, path: "/users",
    headers: {"Authorization": "Bearer valid-token-123"}.toTable(),
    body: ""
  )
  var ctx3 = newContext(req3)
  router.handle(ctx3)
  echo "Status: " & $ctx3.response.statusCode  # 200
  echo "Body: " & ctx3.response.body

main()
```

---

## 📝 สรุป Part 11

| Steps | หัวข้อ |
|-------|--------|
| 131 | Objects, inheritance basics |
| 132 | Method dispatch, polymorphism |
| 133 | Composition over inheritance |
| 134 | Design patterns (Singleton, Observer, Strategy) |
| 135-145 | Real-world: HTTP Middleware System |

---

**← [Part 10: Modules](part_10_modules.md) | [Part 12: Generics →](part_12_generics.md)**
