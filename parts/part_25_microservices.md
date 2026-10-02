# Part 25: Microservices Architecture
## Steps 346-360: ออกแบบและพัฒนา Microservices ด้วย Nim

---

## 🎯 เป้าหมายของ Part นี้

- Microservices vs Monolith
- Service communication (HTTP, gRPC-style, Message Queue)
- API Gateway pattern
- Service discovery
- Circuit breaker pattern
- Health checks และ readiness probes
- Event-driven architecture

---

## Step 346: Microservices vs Monolith

```
Monolith:
┌─────────────────────────────┐
│         Monolith App        │
│  ┌──────┐ ┌───────┐ ┌────┐ │
│  │Users │ │Orders │ │Pay │ │
│  └──────┘ └───────┘ └────┘ │
└─────────────────────────────┘
         │
         ▼
     Single DB

Microservices:
┌──────────┐  ┌───────────┐  ┌──────────┐
│  Users   │  │  Orders   │  │ Payment  │
│ Service  │  │  Service  │  │ Service  │
└──────────┘  └───────────┘  └──────────┘
      │              │              │
   Users DB      Orders DB     Payment DB

ข้อดีของ Microservices:
- Deploy แต่ละ service แยกกัน
- Scale เฉพาะ service ที่ต้องการ
- Technology flexibility ต่อ service
- Fault isolation

ข้อเสีย:
- ซับซ้อนกว่า (distributed systems)
- Network latency
- Data consistency challenges
- More infrastructure needed
```

```nim
# ตัวอย่าง: User Service
# เป็น HTTP server ง่ายๆ ที่ handle user operations

import asyncdispatch, asynchttpserver, json, strutils, strformat, tables

type
  User = object
    id: int
    name: string
    email: string
    createdAt: string

var users: Table[int, User] = initTable[int, User]()
var nextId = 1

proc handleUsers(req: Request) {.async.} =
  let path = req.url.path
  let parts = path.strip(chars = {'/'}).split('/')
  
  if req.reqMethod == HttpGet and parts.len == 1 and parts[0] == "users":
    # GET /users
    var arr = newJArray()
    for _, user in users:
      arr.add(%*{"id": user.id, "name": user.name, "email": user.email})
    await req.respond(Http200, $arr, newHttpHeaders([("Content-Type", "application/json")]))
    return
  
  if req.reqMethod == HttpPost and parts.len == 1 and parts[0] == "users":
    # POST /users
    let body = parseJson(req.body)
    let user = User(
      id: nextId,
      name: body["name"].getStr(),
      email: body["email"].getStr(),
      createdAt: "2025-01-01"
    )
    users[nextId] = user
    inc nextId
    await req.respond(Http201, $(%*{"id": user.id, "name": user.name}),
      newHttpHeaders([("Content-Type", "application/json")]))
    return
  
  if req.reqMethod == HttpGet and parts.len == 2 and parts[0] == "users":
    # GET /users/:id
    let id = parseInt(parts[1])
    if id in users:
      let user = users[id]
      await req.respond(Http200,
        $(%*{"id": user.id, "name": user.name, "email": user.email}),
        newHttpHeaders([("Content-Type", "application/json")]))
    else:
      await req.respond(Http404, """{"error":"User not found"}""",
        newHttpHeaders([("Content-Type", "application/json")]))
    return
  
  # Health check
  if path == "/health":
    await req.respond(Http200, """{"status":"ok","service":"users"}""",
      newHttpHeaders([("Content-Type", "application/json")]))
    return
  
  await req.respond(Http404, """{"error":"Not found"}""",
    newHttpHeaders([("Content-Type", "application/json")]))

proc main() {.async.} =
  let server = newAsyncHttpServer()
  echo "User Service listening on port 8001"
  await server.serve(Port(8001), handleUsers)

waitFor main()
```

---

## Step 347: Service Communication (HTTP Client)

```nim
import asyncdispatch, asynchttpserver, httpclient, json, strformat, options

# Service URLs (in production, use service discovery)
const
  USER_SERVICE = "http://localhost:8001"
  INVENTORY_SERVICE = "http://localhost:8002"
  NOTIFICATION_SERVICE = "http://localhost:8003"

type
  ServiceClient = object
    baseUrl: string
    timeout: int  # milliseconds
    retries: int

proc newServiceClient(baseUrl: string, timeout: int = 5000, retries: int = 3): ServiceClient =
  ServiceClient(baseUrl: baseUrl, timeout: timeout, retries: retries)

proc get(client: ServiceClient, path: string): Future[JsonNode] {.async.} =
  let httpClient = newAsyncHttpClient()
  httpClient.headers = newHttpHeaders([
    ("Accept", "application/json"),
    ("X-Service-Name", "order-service")
  ])
  
  var lastError: ref Exception
  
  for attempt in 1..client.retries:
    try:
      let response = await httpClient.get(client.baseUrl & path)
      let body = await response.body
      return parseJson(body)
    except Exception as e:
      lastError = e
      if attempt < client.retries:
        await sleepAsync(100 * attempt)  # exponential backoff
  
  raise lastError

proc post(client: ServiceClient, path: string, body: JsonNode): Future[JsonNode] {.async.} =
  let httpClient = newAsyncHttpClient()
  httpClient.headers = newHttpHeaders([
    ("Content-Type", "application/json"),
    ("Accept", "application/json")
  ])
  
  let response = await httpClient.post(client.baseUrl & path, $body)
  let respBody = await response.body
  return parseJson(respBody)

# Order Service using other services
let userClient = newServiceClient(USER_SERVICE)
let inventoryClient = newServiceClient(INVENTORY_SERVICE)

proc createOrder(userId: int, productId: int, qty: int): Future[JsonNode] {.async.} =
  # 1. Validate user exists
  try:
    let user = await userClient.get(fmt"/users/{userId}")
    echo fmt"User found: {user[\"name\"].getStr()}"
  except:
    return %*{"error": "User not found"}
  
  # 2. Check inventory
  try:
    let inventory = await inventoryClient.get(fmt"/products/{productId}/stock")
    let available = inventory["quantity"].getInt()
    
    if available < qty:
      return %*{"error": "Insufficient stock", "available": available}
  except:
    return %*{"error": "Inventory service unavailable"}
  
  # 3. Create order (in real app, use DB)
  return %*{
    "orderId": 12345,
    "userId": userId,
    "productId": productId,
    "quantity": qty,
    "status": "pending"
  }

# Usage
# let order = waitFor createOrder(1, 101, 2)
# echo $order
```

---

## Step 348: Circuit Breaker Pattern

```nim
import times, strformat

type
  CircuitState = enum
    Closed    # Normal operation
    Open      # Failing, reject requests
    HalfOpen  # Testing if service recovered

  CircuitBreaker = object
    name: string
    state: CircuitState
    failureCount: int
    failureThreshold: int
    successCount: int
    successThreshold: int
    timeout: float    # seconds before trying again
    lastFailure: float
    totalRequests: int
    totalFailures: int

proc newCircuitBreaker(name: string, failureThreshold: int = 5,
    timeout: float = 30.0, successThreshold: int = 2): CircuitBreaker =
  CircuitBreaker(
    name: name,
    state: Closed,
    failureThreshold: failureThreshold,
    timeout: timeout,
    successThreshold: successThreshold
  )

proc canExecute(cb: var CircuitBreaker): bool =
  case cb.state
  of Closed:
    return true
  of Open:
    # Check if timeout has passed
    if epochTime() - cb.lastFailure > cb.timeout:
      echo fmt"[{cb.name}] Circuit half-opening..."
      cb.state = HalfOpen
      cb.successCount = 0
      return true
    return false
  of HalfOpen:
    return true

proc recordSuccess(cb: var CircuitBreaker) =
  inc cb.totalRequests
  case cb.state
  of HalfOpen:
    inc cb.successCount
    if cb.successCount >= cb.successThreshold:
      echo fmt"[{cb.name}] Circuit closed (recovered)"
      cb.state = Closed
      cb.failureCount = 0
  of Closed:
    cb.failureCount = 0
  else: discard

proc recordFailure(cb: var CircuitBreaker) =
  inc cb.totalRequests
  inc cb.totalFailures
  inc cb.failureCount
  cb.lastFailure = epochTime()
  
  case cb.state
  of Closed:
    if cb.failureCount >= cb.failureThreshold:
      echo fmt"[{cb.name}] Circuit opened! ({cb.failureCount} failures)"
      cb.state = Open
  of HalfOpen:
    echo fmt"[{cb.name}] Circuit re-opened (still failing)"
    cb.state = Open
  else: discard

template withCircuitBreaker(cb: var CircuitBreaker, body: untyped, fallback: untyped) =
  if cb.canExecute():
    try:
      body
      cb.recordSuccess()
    except:
      cb.recordFailure()
      fallback
  else:
    echo fmt"[{cb.name}] CIRCUIT OPEN - rejecting request"
    fallback

proc callPaymentService(amount: float): string =
  # Simulate failures
  if amount > 1000:
    raise newException(IOError, "Payment service timeout")
  return fmt"Payment of {amount} processed"

# Demo
var paymentCB = newCircuitBreaker("payment-service", failureThreshold = 3, timeout = 5.0)

for i in 1..10:
  let amount = if i <= 6: 2000.0 else: 100.0  # First 6 will fail
  
  withCircuitBreaker(paymentCB):
    let result = callPaymentService(amount)
    echo fmt"  Success: {result}"
  do:
    echo fmt"  Fallback: Using cached payment result"

echo fmt"\nCircuit state: {paymentCB.state}"
echo fmt"Total: {paymentCB.totalRequests} requests, {paymentCB.totalFailures} failures"
```

---

## Step 349: API Gateway Pattern

```nim
import asyncdispatch, asynchttpserver, httpclient, json, tables, strutils, strformat, options

type
  ServiceConfig = object
    name: string
    url: string
    pathPrefix: string
    requiresAuth: bool
    rateLimit: int

  GatewayState = object
    services: seq[ServiceConfig]
    requestCounts: Table[string, int]

var gateway = GatewayState(
  services: @[
    ServiceConfig(name: "users",    url: "http://localhost:8001", pathPrefix: "/api/users",    requiresAuth: false, rateLimit: 100),
    ServiceConfig(name: "orders",   url: "http://localhost:8002", pathPrefix: "/api/orders",   requiresAuth: true,  rateLimit: 50),
    ServiceConfig(name: "products", url: "http://localhost:8003", pathPrefix: "/api/products", requiresAuth: false, rateLimit: 200),
  ],
  requestCounts: initTable[string, int]()
)

proc findService(path: string): Option[ServiceConfig] =
  for svc in gateway.services:
    if path.startsWith(svc.pathPrefix):
      return some(svc)
  return none(ServiceConfig)

proc checkRateLimit(clientIp: string, svc: ServiceConfig): bool =
  let key = clientIp & ":" & svc.name
  let count = gateway.requestCounts.getOrDefault(key, 0)
  if count >= svc.rateLimit:
    return false
  gateway.requestCounts[key] = count + 1
  return true

proc forwardRequest(req: Request, svc: ServiceConfig): Future[(HttpCode, string)] {.async.} =
  let httpClient = newAsyncHttpClient()
  
  # Forward headers
  var headers = newHttpHeaders([
    ("X-Forwarded-For", "127.0.0.1"),
    ("X-Gateway", "nim-gateway"),
  ])
  
  # Add auth header if original had it
  if req.headers.hasKey("Authorization"):
    headers["Authorization"] = req.headers["Authorization"]
  
  let targetUrl = svc.url & req.url.path & 
    (if req.url.query.len > 0: "?" & req.url.query else: "")
  
  try:
    let response = case req.reqMethod
      of HttpGet:    await httpClient.get(targetUrl)
      of HttpPost:   await httpClient.post(targetUrl, req.body)
      of HttpPut:    await httpClient.put(targetUrl, req.body)
      of HttpDelete: await httpClient.delete(targetUrl)
      else:          await httpClient.get(targetUrl)
    
    let body = await response.body
    return (response.code, body)
  except Exception as e:
    return (Http502, $(%*{"error": "Service unavailable", "message": e.msg}))

proc handleGateway(req: Request) {.async.} =
  let path = req.url.path
  
  # Health check
  if path == "/health":
    await req.respond(Http200, """{"status":"ok","service":"api-gateway"}""",
      newHttpHeaders([("Content-Type", "application/json")]))
    return
  
  # Find matching service
  let svcOpt = findService(path)
  if svcOpt.isNone:
    await req.respond(Http404, """{"error":"Route not found"}""",
      newHttpHeaders([("Content-Type", "application/json")]))
    return
  
  let svc = svcOpt.get()
  
  # Auth check
  if svc.requiresAuth and not req.headers.hasKey("Authorization"):
    await req.respond(Http401, """{"error":"Authentication required"}""",
      newHttpHeaders([("Content-Type", "application/json")]))
    return
  
  # Rate limit check
  let clientIp = "127.0.0.1"  # In real app, get from headers
  if not checkRateLimit(clientIp, svc):
    await req.respond(Http429, $(%*{"error": "Rate limit exceeded", "limit": svc.rateLimit}),
      newHttpHeaders([("Content-Type", "application/json")]))
    return
  
  # Forward request
  echo fmt"[Gateway] Forwarding {req.reqMethod} {path} -> {svc.name}"
  let (code, body) = await forwardRequest(req, svc)
  
  await req.respond(code, body, newHttpHeaders([
    ("Content-Type", "application/json"),
    ("X-Served-By", svc.name)
  ]))

proc runGateway() {.async.} =
  let server = newAsyncHttpServer()
  echo "API Gateway listening on port 8000"
  echo "Routes:"
  for svc in gateway.services:
    echo fmt"  {svc.pathPrefix} -> {svc.name} ({svc.url})"
  await server.serve(Port(8000), handleGateway)

# waitFor runGateway()
echo "API Gateway configured"
```

---

## Step 350: Event-Driven Architecture

```nim
import asyncdispatch, json, tables, sequtils, strformat, times

# Simple in-process event bus (production would use RabbitMQ/Kafka/NATS)

type
  EventHandler = proc(event: JsonNode): Future[void]
  
  EventBus = object
    subscribers: Table[string, seq[EventHandler]]
    deadLetterQueue: seq[(string, JsonNode, string)]  # (eventType, payload, error)

var eventBus = EventBus(
  subscribers: initTable[string, seq[EventHandler]](),
  deadLetterQueue: @[]
)

proc subscribe(bus: var EventBus, eventType: string, handler: EventHandler) =
  if eventType notin bus.subscribers:
    bus.subscribers[eventType] = @[]
  bus.subscribers[eventType].add(handler)
  echo fmt"[EventBus] Subscribed to '{eventType}'"

proc publish(bus: var EventBus, eventType: string, payload: JsonNode) {.async.} =
  echo fmt"[EventBus] Publishing '{eventType}': {payload}"
  
  if eventType notin bus.subscribers:
    echo fmt"[EventBus] No subscribers for '{eventType}'"
    return
  
  for handler in bus.subscribers[eventType]:
    try:
      await handler(payload)
    except Exception as e:
      echo fmt"[EventBus] Handler failed for '{eventType}': {e.msg}"
      bus.deadLetterQueue.add((eventType, payload, e.msg))

# Event types
const
  USER_REGISTERED = "user.registered"
  ORDER_PLACED = "order.placed"
  ORDER_PAID = "order.paid"
  ORDER_SHIPPED = "order.shipped"
  PAYMENT_FAILED = "payment.failed"

# Handlers
proc sendWelcomeEmail(event: JsonNode): Future[void] {.async.} =
  let email = event["email"].getStr()
  let name = event["name"].getStr()
  echo fmt"  [Email Service] Sending welcome email to {name} <{email}>"

proc createUserProfile(event: JsonNode): Future[void] {.async.} =
  let userId = event["userId"].getInt()
  echo fmt"  [Profile Service] Creating profile for user {userId}"

proc reserveInventory(event: JsonNode): Future[void] {.async.} =
  let orderId = event["orderId"].getInt()
  let items = event["items"]
  echo fmt"  [Inventory Service] Reserving items for order {orderId}: {items}"

proc processPayment(event: JsonNode): Future[void] {.async.} =
  let orderId = event["orderId"].getInt()
  let amount = event["total"].getFloat()
  echo fmt"  [Payment Service] Processing ${amount:.2f} for order {orderId}"

proc sendOrderConfirmation(event: JsonNode): Future[void] {.async.} =
  let orderId = event["orderId"].getInt()
  let userEmail = event["userEmail"].getStr()
  echo fmt"  [Email Service] Sending order confirmation for #{orderId} to {userEmail}"

proc updateAnalytics(event: JsonNode): Future[void] {.async.} =
  echo fmt"  [Analytics Service] Recording event: {event[\"type\"].getStr()}"

# Wire up subscriptions
eventBus.subscribe(USER_REGISTERED, sendWelcomeEmail)
eventBus.subscribe(USER_REGISTERED, createUserProfile)
eventBus.subscribe(USER_REGISTERED, proc(e: JsonNode): Future[void] {.async.} =
  await updateAnalytics(%*{"type": USER_REGISTERED, "data": e})
)
eventBus.subscribe(ORDER_PLACED, reserveInventory)
eventBus.subscribe(ORDER_PLACED, processPayment)
eventBus.subscribe(ORDER_PAID, sendOrderConfirmation)

# Simulate events
proc runDemo() {.async.} =
  echo "\n=== Event-Driven Demo ==="
  
  echo "\n1. User Registration:"
  await eventBus.publish(USER_REGISTERED, %*{
    "userId": 42,
    "name": "สมชาย ใจดี",
    "email": "somchai@example.com"
  })
  
  echo "\n2. Order Placed:"
  await eventBus.publish(ORDER_PLACED, %*{
    "orderId": 1001,
    "userId": 42,
    "userEmail": "somchai@example.com",
    "items": [{"productId": 1, "qty": 2}, {"productId": 5, "qty": 1}],
    "total": 1299.00
  })
  
  echo "\n3. Payment Successful:"
  await eventBus.publish(ORDER_PAID, %*{
    "orderId": 1001,
    "userEmail": "somchai@example.com",
    "amount": 1299.00
  })
  
  echo fmt"\n[Dead Letter Queue: {eventBus.deadLetterQueue.len} items]"

waitFor runDemo()
```

---

## Step 351-360: Complete Microservices Demo

```nim
# microservices_demo.nim - สาธิต Microservices patterns ครบ

import asyncdispatch, asynchttpserver, json, tables, strutils,
       strformat, times, options, hashes, sequtils

# ==========================================
# Shared types and utilities
# ==========================================

type
  ServiceHealth = object
    service: string
    status: string
    uptime: float
    version: string
    dependencies: seq[tuple[name: string, status: string]]

  ServiceRegistry = object
    services: Table[string, string]  # name -> url

var registry = ServiceRegistry(services: initTable[string, string]())

proc register(reg: var ServiceRegistry, name, url: string) =
  reg.services[name] = url
  echo fmt"[Registry] Registered: {name} -> {url}"

proc discover(reg: ServiceRegistry, name: string): Option[string] =
  if name in reg.services:
    return some(reg.services[name])
  return none(string)

# ==========================================
# Health Check System
# ==========================================

proc healthCheck(service: string): ServiceHealth =
  ServiceHealth(
    service: service,
    status: "healthy",
    uptime: epochTime() - 1000.0,
    version: "1.0.0",
    dependencies: @[
      (name: "database", status: "connected"),
      (name: "cache", status: "connected"),
    ]
  )

# ==========================================
# Saga Pattern for Distributed Transactions
# ==========================================

type
  SagaStep = object
    name: string
    execute: proc(): bool
    compensate: proc()

  Saga = object
    name: string
    steps: seq[SagaStep]
    executedSteps: seq[int]

proc newSaga(name: string): Saga =
  Saga(name: name, steps: @[], executedSteps: @[])

proc addStep(saga: var Saga, step: SagaStep) =
  saga.steps.add(step)

proc run(saga: var Saga): bool =
  echo fmt"\n[Saga] Starting '{saga.name}'"
  
  for i, step in saga.steps:
    echo fmt"  [{i+1}/{saga.steps.len}] Executing: {step.name}"
    
    if step.execute():
      saga.executedSteps.add(i)
      echo fmt"  ✓ {step.name} succeeded"
    else:
      echo fmt"  ✗ {step.name} failed — compensating..."
      
      # Compensate in reverse order
      for j in countdown(saga.executedSteps.len - 1, 0):
        let idx = saga.executedSteps[j]
        echo fmt"  ↩ Compensating: {saga.steps[idx].name}"
        saga.steps[idx].compensate()
      
      echo fmt"[Saga] '{saga.name}' rolled back"
      return false
  
  echo fmt"[Saga] '{saga.name}' completed successfully"
  return true

# Demo: Order Processing Saga
var orderReserved = false
var paymentCharged = false
var inventoryReduced = false

var orderSaga = newSaga("CreateOrder")

orderSaga.addStep(SagaStep(
  name: "Reserve Order",
  execute: proc(): bool =
    orderReserved = true
    echo "    Order #1001 reserved"
    return true,
  compensate: proc() =
    orderReserved = false
    echo "    Order #1001 cancelled"
))

orderSaga.addStep(SagaStep(
  name: "Charge Payment",
  execute: proc(): bool =
    # Simulate success
    paymentCharged = true
    echo "    Payment of $99.99 charged"
    return true,
  compensate: proc() =
    paymentCharged = false
    echo "    Payment of $99.99 refunded"
))

orderSaga.addStep(SagaStep(
  name: "Reduce Inventory",
  execute: proc(): bool =
    # Simulate failure (out of stock)
    echo "    Checking stock... OUT OF STOCK!"
    return false,
  compensate: proc() = discard
))

let success = orderSaga.run()
echo fmt"\nSaga result: {if success: \"SUCCESS\" else: \"FAILED\"}"
echo fmt"Order reserved: {orderReserved}"
echo fmt"Payment charged: {paymentCharged}"

# ==========================================
# Service Configuration
# ==========================================

type
  ServiceConfig2 = object
    port: int
    logLevel: string
    dbUrl: string
    jwtSecret: string
    redisUrl: string
    serviceName: string
    environment: string

proc loadConfig(env: string): ServiceConfig2 =
  case env
  of "production":
    ServiceConfig2(
      port: 8080,
      logLevel: "warn",
      dbUrl: "postgres://prod-db:5432/myapp",
      jwtSecret: "prod-secret-from-vault",
      redisUrl: "redis://prod-redis:6379",
      serviceName: "user-service",
      environment: "production"
    )
  of "staging":
    ServiceConfig2(
      port: 8080,
      logLevel: "info",
      dbUrl: "postgres://staging-db:5432/myapp",
      jwtSecret: "staging-secret",
      redisUrl: "redis://staging-redis:6379",
      serviceName: "user-service",
      environment: "staging"
    )
  else:  # development
    ServiceConfig2(
      port: 8001,
      logLevel: "debug",
      dbUrl: "postgres://localhost:5432/myapp_dev",
      jwtSecret: "dev-secret-not-for-prod",
      redisUrl: "redis://localhost:6379",
      serviceName: "user-service",
      environment: "development"
    )

# ==========================================
# Main Demo
# ==========================================

proc main() =
  echo "=== Microservices Architecture Demo ==="
  
  # Register services
  registry.register("users", "http://localhost:8001")
  registry.register("orders", "http://localhost:8002")
  registry.register("payments", "http://localhost:8003")
  
  # Discover service
  let usersUrl = registry.discover("users")
  echo fmt"\nDiscovered users service: {usersUrl.get()}"
  
  # Health checks
  echo "\nHealth Checks:"
  for name in ["users", "orders", "payments"]:
    let health = healthCheck(name)
    echo fmt"  {health.service}: {health.status} (v{health.version})"
  
  # Load config
  let config = loadConfig("development")
  echo fmt"\nConfig loaded for: {config.environment}"
  echo fmt"  Port: {config.port}"
  echo fmt"  Service: {config.serviceName}"
  
  echo "\nMicroservices demo complete!"

main()
```

---

## 📝 สรุป Part 25

| Steps | หัวข้อ |
|-------|--------|
| 346 | Microservices vs Monolith concepts |
| 347 | Service-to-service HTTP communication |
| 348 | Circuit breaker pattern |
| 349 | API Gateway pattern |
| 350 | Event-driven architecture |
| 351-360 | Saga pattern, service registry, health checks |

---

**← [Part 24: Caching](part_24_caching.md) | [Part 26: File Uploads →](part_26_file_uploads.md)**
