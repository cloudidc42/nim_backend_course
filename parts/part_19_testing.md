# Part 19: Testing ด้วย Nim

## Steps 256-270

การเขียน tests ที่ดีเป็นสิ่งจำเป็นสำหรับ production code ในบทนี้เราจะเรียนรู้ทุกด้านของการ testing ใน Nim ตั้งแต่ unit tests, integration tests, mocking จนถึง TDD และ test coverage

---

## Step 256: std/unittest พื้นฐาน

```nim
# file: tests/test_basic.nim
import std/unittest

# Basic test
suite "Basic Math Tests":
  test "addition":
    check 1 + 1 == 2
    check 10 + 20 == 30
  
  test "subtraction":
    check 10 - 3 == 7
  
  test "multiplication":
    check 4 * 5 == 20
  
  test "division":
    check 10 / 2 == 5.0
    check 10 div 2 == 5
  
  test "modulo":
    check 10 mod 3 == 1

# รัน: nim c -r tests/test_basic.nim
# หรือ: nimble test
```

---

## Step 257: การทดสอบ Strings และ Collections

```nim
# file: tests/test_strings.nim
import std/unittest, strutils, sequtils, strformat

suite "String Operations":
  test "uppercase conversion":
    check "hello".toUpperAscii() == "HELLO"
    check "world".toUpperAscii() == "WORLD"
  
  test "string contains":
    let text = "Hello World"
    check text.contains("World")
    check not text.contains("earth")
  
  test "split and join":
    let csv = "a,b,c,d"
    let parts = csv.split(",")
    check parts.len == 4
    check parts[0] == "a"
    check parts[^1] == "d"
    check parts.join("|") == "a|b|c|d"
  
  test "trim whitespace":
    check "  hello  ".strip() == "hello"
    check "\t\nhello\n\t".strip() == "hello"
  
  test "string formatting":
    let name = "Alice"
    let age = 28
    let result = &"Name: {name}, Age: {age}"
    check result == "Name: Alice, Age: 28"
  
  test "starts and ends with":
    let url = "https://example.com/api/users"
    check url.startsWith("https://")
    check url.endsWith("/users")

suite "Sequence Operations":
  test "map and filter":
    let nums = @[1, 2, 3, 4, 5, 6]
    let evens = nums.filter(proc(n: int): bool = n mod 2 == 0)
    let doubled = nums.map(proc(n: int): int = n * 2)
    
    check evens == @[2, 4, 6]
    check doubled == @[2, 4, 6, 8, 10, 12]
  
  test "find element":
    let fruits = @["apple", "banana", "cherry"]
    check "banana" in fruits
    check "grape" notin fruits
  
  test "foldl":
    let nums = @[1, 2, 3, 4, 5]
    let sum = nums.foldl(a + b, 0)
    check sum == 15
```

---

## Step 258: Test Suites พร้อม Setup/Teardown

```nim
# file: tests/test_with_setup.nim
import std/unittest, tables, strformat

type
  User = object
    id: int
    name: string
    email: string
    active: bool

  UserDatabase = ref object
    users: Table[int, User]
    nextId: int

proc newUserDatabase(): UserDatabase =
  UserDatabase(users: initTable[int, User](), nextId: 1)

proc addUser(db: UserDatabase, name, email: string): User =
  let user = User(
    id: db.nextId,
    name: name,
    email: email,
    active: true
  )
  db.users[db.nextId] = user
  inc db.nextId
  return user

proc getUser(db: UserDatabase, id: int): User =
  if id notin db.users:
    raise newException(KeyError, &"User {id} not found")
  return db.users[id]

proc updateUser(db: UserDatabase, id: int, name = "", email = "") =
  if id notin db.users:
    raise newException(KeyError, &"User {id} not found")
  if name.len > 0:
    db.users[id].name = name
  if email.len > 0:
    db.users[id].email = email

proc deleteUser(db: UserDatabase, id: int) =
  db.users.del(id)

proc countUsers(db: UserDatabase): int =
  db.users.len

# ===== Tests with Setup/Teardown =====

var db: UserDatabase

suite "User Database Tests":
  setup:
    # รันก่อนทุก test
    db = newUserDatabase()
    discard db.addUser("Alice", "alice@example.com")
    discard db.addUser("Bob", "bob@example.com")
  
  teardown:
    # รันหลังทุก test
    db = nil
  
  test "initial state has 2 users":
    check db.countUsers() == 2
  
  test "add new user":
    let charlie = db.addUser("Charlie", "charlie@example.com")
    check charlie.id == 3
    check charlie.name == "Charlie"
    check db.countUsers() == 3
  
  test "get existing user":
    let user = db.getUser(1)
    check user.name == "Alice"
    check user.email == "alice@example.com"
    check user.active == true
  
  test "get nonexistent user raises exception":
    expect KeyError:
      discard db.getUser(999)
  
  test "update user name":
    db.updateUser(1, name = "Alice Updated")
    let user = db.getUser(1)
    check user.name == "Alice Updated"
    check user.email == "alice@example.com"  # unchanged
  
  test "delete user":
    db.deleteUser(1)
    check db.countUsers() == 1
    expect KeyError:
      discard db.getUser(1)
  
  test "users are independent between tests":
    # ทุก test เริ่มต้นใหม่ด้วย setup
    check db.countUsers() == 2  # จาก setup เสมอ
```

---

## Step 259: Testing Errors และ Exceptions

```nim
# file: tests/test_errors.nim
import std/unittest, strformat

type
  ValidationError = object of ValueError
  NotFoundError = object of KeyError
  AuthError = object of AccessViolationDefect

proc validateAge(age: int): int =
  if age < 0:
    raise newException(ValidationError, "Age cannot be negative")
  if age > 150:
    raise newException(ValidationError, "Age seems unrealistic")
  return age

proc divide(a, b: float): float =
  if b == 0.0:
    raise newException(DivByZeroDefect, "Cannot divide by zero")
  return a / b

proc findUser(id: int): string =
  if id == 1: return "Alice"
  if id == 2: return "Bob"
  raise newException(NotFoundError, &"User {id} not found")

suite "Error Handling Tests":
  test "valid age passes":
    check validateAge(25) == 25
    check validateAge(0) == 0
    check validateAge(100) == 100
  
  test "negative age raises ValidationError":
    expect ValidationError:
      discard validateAge(-1)
  
  test "too large age raises ValidationError":
    expect ValidationError:
      discard validateAge(200)
  
  test "divide by zero raises exception":
    expect DivByZeroDefect:
      discard divide(10.0, 0.0)
  
  test "normal division works":
    check divide(10.0, 2.0) == 5.0
    check abs(divide(1.0, 3.0) - 0.333) < 0.001
  
  test "find existing user":
    check findUser(1) == "Alice"
    check findUser(2) == "Bob"
  
  test "find missing user raises NotFoundError":
    expect NotFoundError:
      discard findUser(999)
  
  test "exception message content":
    try:
      discard validateAge(-5)
      fail()
    except ValidationError as e:
      check e.msg.contains("negative")
  
  test "multiple exception types":
    # ทดสอบว่า catch ได้ exception ที่ถูกต้อง
    var caughtValidation = false
    var caughtOther = false
    
    try:
      discard validateAge(-1)
    except ValidationError:
      caughtValidation = true
    except Exception:
      caughtOther = true
    
    check caughtValidation == true
    check caughtOther == false
```

---

## Step 260: Testing Async Code

```nim
# file: tests/test_async.nim
import std/unittest, asyncdispatch, asyncfutures, strformat

proc fetchData(id: int): Future[string] {.async.} =
  await sleepAsync(10)  # Simulate async work
  if id <= 0:
    raise newException(ValueError, "ID must be positive")
  return &"Data for id={id}"

proc fetchMultiple(ids: seq[int]): Future[seq[string]] {.async.} =
  var futures: seq[Future[string]] = @[]
  for id in ids:
    futures.add(fetchData(id))
  return await all(futures)

proc withTimeout(timeoutMs: int): Future[string] {.async.} =
  await sleepAsync(timeoutMs)
  return "completed"

suite "Async Tests":
  test "basic async":
    proc doTest() {.async.} =
      let result = await fetchData(1)
      check result == "Data for id=1"
    
    waitFor doTest()
  
  test "async with error":
    proc doTest() {.async.} =
      var caught = false
      try:
        discard await fetchData(-1)
      except ValueError:
        caught = true
      check caught == true
    
    waitFor doTest()
  
  test "parallel async":
    proc doTest() {.async.} =
      let results = await fetchMultiple(@[1, 2, 3])
      check results.len == 3
      check results[0] == "Data for id=1"
      check results[2] == "Data for id=3"
    
    waitFor doTest()
  
  test "async timing":
    proc doTest() {.async.} =
      let start = epochTime()
      # รัน 3 tasks parallel
      let futures = @[
        fetchData(1),
        fetchData(2),
        fetchData(3)
      ]
      discard await all(futures)
      let elapsed = epochTime() - start
      
      # Parallel: ~10ms, Sequential: ~30ms
      check elapsed < 0.1  # ไม่ควรนานกว่า 100ms
    
    waitFor doTest()
```

---

## Step 261: Mocking และ Stubs

```nim
# file: tests/test_mocking.nim
import std/unittest, tables, asyncdispatch

# ===== Interfaces ที่จะ mock =====

type
  EmailService = ref object of RootObj

method sendEmail*(svc: EmailService, to, subject, body: string): Future[bool] {.base, async.} =
  discard

type
  SMSService = ref object of RootObj

method sendSMS*(svc: SMSService, phone, message: string): Future[bool] {.base, async.} =
  discard

# ===== Mock implementations =====

type
  MockEmailService = ref object of EmailService
    sentEmails: seq[tuple[to, subject, body: string]]
    shouldFail: bool

method sendEmail*(svc: MockEmailService, to, subject, body: string): Future[bool] {.async.} =
  if svc.shouldFail:
    return false
  svc.sentEmails.add((to, subject, body))
  return true

type
  MockSMSService = ref object of SMSService
    sentMessages: seq[tuple[phone, message: string]]
    callCount: int

method sendSMS*(svc: MockSMSService, phone, message: string): Future[bool] {.async.} =
  inc svc.callCount
  svc.sentMessages.add((phone, message))
  return true

# ===== System under test =====

type
  NotificationSystem = ref object
    emailSvc: EmailService
    smsSvc: SMSService

proc newNotificationSystem(email: EmailService, sms: SMSService): NotificationSystem =
  NotificationSystem(emailSvc: email, smsSvc: sms)

proc notifyUser(ns: NotificationSystem, email, phone, message: string): Future[bool] {.async.} =
  let emailSent = await ns.emailSvc.sendEmail(email, "Notification", message)
  let smsSent = await ns.smsSvc.sendSMS(phone, message)
  return emailSent and smsSent

proc sendWelcome(ns: NotificationSystem, email: string): Future[bool] {.async.} =
  return await ns.emailSvc.sendEmail(
    email,
    "Welcome to our service",
    "Thank you for joining!"
  )

# ===== Tests using mocks =====

suite "Notification System Tests":
  var mockEmail: MockEmailService
  var mockSMS: MockSMSService
  var notifSystem: NotificationSystem
  
  setup:
    mockEmail = MockEmailService(sentEmails: @[], shouldFail: false)
    mockSMS = MockSMSService(sentMessages: @[], callCount: 0)
    notifSystem = newNotificationSystem(mockEmail, mockSMS)
  
  test "sends both email and SMS":
    proc doTest() {.async.} =
      let result = await notifSystem.notifyUser(
        "user@example.com",
        "+1234567890",
        "Your order is ready"
      )
      
      check result == true
      check mockEmail.sentEmails.len == 1
      check mockEmail.sentEmails[0].to == "user@example.com"
      check mockSMS.sentMessages.len == 1
      check mockSMS.callCount == 1
    
    waitFor doTest()
  
  test "returns false when email fails":
    proc doTest() {.async.} =
      mockEmail.shouldFail = true
      
      let result = await notifSystem.notifyUser(
        "user@example.com",
        "+1234567890",
        "Test message"
      )
      
      check result == false
      check mockEmail.sentEmails.len == 0  # nothing was "sent"
    
    waitFor doTest()
  
  test "welcome email content":
    proc doTest() {.async.} =
      discard await notifSystem.sendWelcome("new@example.com")
      
      check mockEmail.sentEmails.len == 1
      let sent = mockEmail.sentEmails[0]
      check sent.to == "new@example.com"
      check sent.subject == "Welcome to our service"
      check sent.body.contains("Thank you")
    
    waitFor doTest()
  
  test "multiple notifications":
    proc doTest() {.async.} =
      for i in 1..5:
        discard await notifSystem.notifyUser(
          &"user{i}@example.com",
          &"+{i}234567890",
          &"Message {i}"
        )
      
      check mockEmail.sentEmails.len == 5
      check mockSMS.callCount == 5
    
    waitFor doTest()
```

---

## Step 262: Test Fixtures และ Data Builders

```nim
# file: tests/fixtures.nim
import times, random

type
  UserFixture* = object
    id*: int
    username*: string
    email*: string
    password*: string
    role*: string
    active*: bool
    createdAt*: DateTime

  ProductFixture* = object
    id*: int
    name*: string
    price*: float
    stock*: int
    category*: string

var fixtureCounter = 0

proc nextId*(): int =
  inc fixtureCounter
  return fixtureCounter

proc buildUser*(
  id = -1,
  username = "",
  email = "",
  password = "password123",
  role = "user",
  active = true
): UserFixture =
  let actualId = if id == -1: nextId() else: id
  let actualUsername = if username.len == 0: &"user_{actualId}" else: username
  let actualEmail = if email.len == 0: &"{actualUsername}@example.com" else: email
  
  UserFixture(
    id: actualId,
    username: actualUsername,
    email: actualEmail,
    password: password,
    role: role,
    active: active,
    createdAt: now()
  )

proc buildProduct*(
  id = -1,
  name = "",
  price = 0.0,
  stock = 100,
  category = "general"
): ProductFixture =
  let actualId = if id == -1: nextId() else: id
  let actualName = if name.len == 0: &"Product {actualId}" else: name
  let actualPrice = if price == 0.0: rand(9999).float / 100.0 else: price
  
  ProductFixture(
    id: actualId,
    name: actualName,
    price: actualPrice,
    stock: stock,
    category: category
  )

# ===== Usage in tests =====

# file: tests/test_fixtures.nim
import std/unittest
import fixtures

suite "Fixture Tests":
  test "build default user":
    let user = buildUser()
    check user.username.startsWith("user_")
    check user.email.contains("@example.com")
    check user.role == "user"
    check user.active == true
  
  test "build user with custom values":
    let admin = buildUser(
      username = "admin",
      email = "admin@company.com",
      role = "admin"
    )
    check admin.username == "admin"
    check admin.role == "admin"
  
  test "build multiple users with unique IDs":
    let user1 = buildUser()
    let user2 = buildUser()
    let user3 = buildUser()
    
    check user1.id != user2.id
    check user2.id != user3.id
  
  test "build inactive user":
    let inactive = buildUser(active = false)
    check inactive.active == false
```

---

## Step 263: Integration Tests

```nim
# file: tests/test_integration.nim
import std/unittest, asyncdispatch, asynchttpserver, httpclient, json
import strformat, os, times

# ====== Test Server ======

proc startTestServer(port: int): Future[AsyncHttpServer] {.async.} =
  var server = newAsyncHttpServer()
  
  proc handler(req: Request) {.async.} =
    case req.url.path
    of "/api/ping":
      await req.respond(Http200, $(%*{"pong": true, "time": $now()}),
        newHttpHeaders([("Content-Type", "application/json")]))
    
    of "/api/echo":
      await req.respond(Http200, req.body,
        newHttpHeaders([("Content-Type", "application/json")]))
    
    of "/api/users":
      if req.reqMethod == HttpGet:
        await req.respond(Http200, $(%*{
          "users": [
            {"id": 1, "name": "Alice"},
            {"id": 2, "name": "Bob"}
          ]
        }), newHttpHeaders([("Content-Type", "application/json")]))
      
      elif req.reqMethod == HttpPost:
        let body = parseJson(req.body)
        await req.respond(Http201, $(%*{
          "id": 3,
          "name": body["name"].getStr(),
          "created": true
        }), newHttpHeaders([("Content-Type", "application/json")]))
      
      else:
        await req.respond(Http405, "Method Not Allowed")
    
    of "/api/users/1":
      await req.respond(Http200, $(%*{
        "id": 1,
        "name": "Alice",
        "email": "alice@example.com"
      }), newHttpHeaders([("Content-Type", "application/json")]))
    
    else:
      await req.respond(Http404, $(%*{"error": "Not found"}))
  
  asyncCheck server.serve(Port(port), handler)
  await sleepAsync(100)  # Give server time to start
  return server

# ====== Integration Test Helpers ======

type
  TestClient = ref object
    client: AsyncHttpClient
    baseUrl: string

proc newTestClient(port: int): TestClient =
  TestClient(
    client: newAsyncHttpClient(),
    baseUrl: &"http://localhost:{port}"
  )

proc get(tc: TestClient, path: string): Future[Response] {.async.} =
  return await tc.client.get(tc.baseUrl & path)

proc post(tc: TestClient, path: string, body: JsonNode): Future[Response] {.async.} =
  tc.client.headers = newHttpHeaders([("Content-Type", "application/json")])
  return await tc.client.post(tc.baseUrl & path, $body)

proc getJson(tc: TestClient, path: string): Future[JsonNode] {.async.} =
  let resp = await tc.get(path)
  return parseJson(await resp.body)

proc postJson(tc: TestClient, path: string, body: JsonNode): Future[(int, JsonNode)] {.async.} =
  let resp = await tc.post(path, body)
  let data = parseJson(await resp.body)
  return (resp.status.int, data)

# ====== Tests ======

const TEST_PORT = 18080

suite "HTTP Integration Tests":
  var server: AsyncHttpServer
  var client: TestClient
  
  setup:
    proc doSetup() {.async.} =
      server = await startTestServer(TEST_PORT)
      client = newTestClient(TEST_PORT)
    waitFor doSetup()
  
  teardown:
    server.close()
    await sleepAsync(50)
  
  test "GET /api/ping returns pong":
    proc doTest() {.async.} =
      let data = await client.getJson("/api/ping")
      check data["pong"].getBool() == true
    waitFor doTest()
  
  test "GET /api/users returns list":
    proc doTest() {.async.} =
      let data = await client.getJson("/api/users")
      check data["users"].len == 2
      check data["users"][0]["name"].getStr() == "Alice"
    waitFor doTest()
  
  test "GET /api/users/1 returns user":
    proc doTest() {.async.} =
      let data = await client.getJson("/api/users/1")
      check data["id"].getInt() == 1
      check data["name"].getStr() == "Alice"
    waitFor doTest()
  
  test "POST /api/users creates user":
    proc doTest() {.async.} =
      let (status, data) = await client.postJson(
        "/api/users",
        %*{"name": "Charlie", "email": "charlie@example.com"}
      )
      check status == 201
      check data["id"].getInt() == 3
      check data["name"].getStr() == "Charlie"
      check data["created"].getBool() == true
    waitFor doTest()
  
  test "GET nonexistent returns 404":
    proc doTest() {.async.} =
      let resp = await client.get("/api/nonexistent")
      check resp.status == "404 Not Found"
    waitFor doTest()
  
  test "POST /api/echo echoes body":
    proc doTest() {.async.} =
      let payload = %*{"key": "value", "number": 42}
      let (_, data) = await client.postJson("/api/echo", payload)
      check data["key"].getStr() == "value"
      check data["number"].getInt() == 42
    waitFor doTest()
```

---

## Step 264: HTTP API Testing Helpers

```nim
# file: tests/api_test_helper.nim
import asyncdispatch, asynchttpserver, httpclient
import json, strformat, tables, strutils, httpcore

type
  TestResponse* = object
    status*: int
    body*: string
    headers*: HttpHeaders
    json*: JsonNode

  ApiClient* = ref object
    client: AsyncHttpClient
    baseUrl: string
    defaultHeaders: HttpHeaders

proc newApiClient*(port: int): ApiClient =
  let headers = newHttpHeaders()
  ApiClient(
    client: newAsyncHttpClient(headers = headers),
    baseUrl: &"http://localhost:{port}",
    defaultHeaders: headers
  )

proc setHeader*(client: ApiClient, key, value: string) =
  client.defaultHeaders[key] = value

proc setBearerToken*(client: ApiClient, token: string) =
  client.setHeader("Authorization", &"Bearer {token}")

proc makeRequest(client: ApiClient, `method`: string, path: string, 
                 body = "", extraHeaders: seq[(string, string)] = @[]): Future[TestResponse] {.async.} =
  var headers = newHttpHeaders()
  
  # Copy default headers
  for key, val in client.defaultHeaders:
    headers[key] = val
  
  # Add extra headers
  for (key, val) in extraHeaders:
    headers[key] = val
  
  if body.len > 0:
    headers["Content-Type"] = "application/json"
  
  client.client.headers = headers
  
  let url = client.baseUrl & path
  
  let resp = case `method`
    of "GET": await client.client.get(url)
    of "POST": await client.client.post(url, body)
    of "PUT": await client.client.put(url, body)
    of "DELETE": await client.client.delete(url)
    else: await client.client.get(url)
  
  let respBody = await resp.body
  
  var jsonBody: JsonNode
  try:
    jsonBody = parseJson(respBody)
  except JsonParsingError:
    jsonBody = newJNull()
  
  TestResponse(
    status: resp.status.split(" ")[0].parseInt(),
    body: respBody,
    headers: resp.headers,
    json: jsonBody
  )

proc get*(client: ApiClient, path: string): Future[TestResponse] {.async.} =
  return await client.makeRequest("GET", path)

proc post*(client: ApiClient, path: string, body: JsonNode): Future[TestResponse] {.async.} =
  return await client.makeRequest("POST", path, $body)

proc put*(client: ApiClient, path: string, body: JsonNode): Future[TestResponse] {.async.} =
  return await client.makeRequest("PUT", path, $body)

proc delete*(client: ApiClient, path: string): Future[TestResponse] {.async.} =
  return await client.makeRequest("DELETE", path)

# ===== Assertion Helpers =====

proc assertStatus*(resp: TestResponse, expected: int) =
  if resp.status != expected:
    let msg = &"Expected status {expected}, got {resp.status}. Body: {resp.body}"
    raise newException(AssertionDefect, msg)

proc assertJsonField*(resp: TestResponse, field: string, expected: JsonNode) =
  if resp.json.kind == JNull:
    raise newException(AssertionDefect, "Response is not JSON")
  
  if not resp.json.hasKey(field):
    raise newException(AssertionDefect, &"Field '{field}' not found in response")
  
  if resp.json[field] != expected:
    raise newException(AssertionDefect, 
      &"Field '{field}': expected {expected}, got {resp.json[field]}")

proc assertHasField*(resp: TestResponse, field: string) =
  if resp.json.kind == JNull or not resp.json.hasKey(field):
    raise newException(AssertionDefect, &"Expected field '{field}' in response")

# ===== Usage =====

# file: tests/test_api_advanced.nim
import std/unittest, asyncdispatch
import api_test_helper

suite "Advanced API Tests":
  var client: ApiClient
  
  setup:
    client = newApiClient(8080)
  
  test "authenticated request":
    proc doTest() {.async.} =
      client.setBearerToken("valid_token_123")
      let resp = await client.get("/api/me")
      resp.assertStatus(200)
      resp.assertHasField("userId")
    waitFor doTest()
  
  test "create and retrieve resource":
    proc doTest() {.async.} =
      # Create
      let createResp = await client.post("/api/posts", %*{
        "title": "Test Post",
        "content": "Hello World"
      })
      createResp.assertStatus(201)
      createResp.assertHasField("id")
      
      let postId = createResp.json["id"].getInt()
      
      # Retrieve
      let getResp = await client.get(&"/api/posts/{postId}")
      getResp.assertStatus(200)
      getResp.assertJsonField("title", %"Test Post")
    waitFor doTest()
```

---

## Step 265: Testing Database Operations

```nim
# file: tests/test_database.nim
import std/unittest, asyncdispatch, db_postgres
import strformat, os

# Test database URL (อ่านจาก env)
let testDbUrl = getEnv("TEST_DATABASE_URL", 
  "postgres://test_user:test_pass@localhost:5432/test_db")

type
  UserRepo = ref object
    db: DbConn

proc newUserRepo(db: DbConn): UserRepo =
  UserRepo(db: db)

proc createTable(repo: UserRepo) =
  repo.db.exec(sql"""
    CREATE TABLE IF NOT EXISTS test_users (
      id SERIAL PRIMARY KEY,
      username VARCHAR(50) UNIQUE NOT NULL,
      email VARCHAR(100) UNIQUE NOT NULL,
      active BOOLEAN DEFAULT true,
      created_at TIMESTAMPTZ DEFAULT NOW()
    )
  """)

proc dropTable(repo: UserRepo) =
  repo.db.exec(sql"DROP TABLE IF EXISTS test_users")

proc insertUser(repo: UserRepo, username, email: string): int64 =
  return repo.db.insertID(sql"""
    INSERT INTO test_users (username, email) VALUES (?, ?)
  """, username, email)

proc getUser(repo: UserRepo, id: int): Row =
  return repo.db.getRow(sql"""
    SELECT id, username, email, active FROM test_users WHERE id = ?
  """, id)

proc countUsers(repo: UserRepo): int =
  let row = repo.db.getRow(sql"SELECT COUNT(*) FROM test_users")
  return parseInt(row[0])

proc activateUser(repo: UserRepo, id: int) =
  repo.db.exec(sql"UPDATE test_users SET active = true WHERE id = ?", id)

proc deactivateUser(repo: UserRepo, id: int) =
  repo.db.exec(sql"UPDATE test_users SET active = false WHERE id = ?", id)

proc deleteUser(repo: UserRepo, id: int) =
  repo.db.exec(sql"DELETE FROM test_users WHERE id = ?", id)

suite "Database Tests":
  var db: DbConn
  var repo: UserRepo
  
  setup:
    try:
      db = open("", "", "", testDbUrl)
      repo = newUserRepo(db)
      repo.createTable()
      
      # Clean test data
      db.exec(sql"DELETE FROM test_users")
    except DbError as e:
      echo &"DB setup failed: {e.msg}"
      echo "Skipping DB tests (no database available)"
  
  teardown:
    if not db.isNil:
      repo.dropTable()
      db.close()
  
  test "insert and retrieve user":
    let id = repo.insertUser("alice", "alice@test.com")
    check id > 0
    
    let user = repo.getUser(int(id))
    check user[1] == "alice"
    check user[2] == "alice@test.com"
  
  test "count users":
    check repo.countUsers() == 0
    
    discard repo.insertUser("user1", "user1@test.com")
    discard repo.insertUser("user2", "user2@test.com")
    
    check repo.countUsers() == 2
  
  test "activate and deactivate":
    let id = int(repo.insertUser("testuser", "test@test.com"))
    
    repo.deactivateUser(id)
    let inactive = repo.getUser(id)
    check inactive[3] == "f"  # false
    
    repo.activateUser(id)
    let active = repo.getUser(id)
    check active[3] == "t"  # true
  
  test "delete user":
    let id = int(repo.insertUser("todelete", "delete@test.com"))
    check repo.countUsers() == 1
    
    repo.deleteUser(id)
    check repo.countUsers() == 0
    
    let deleted = repo.getUser(id)
    check deleted[0] == ""  # empty row
```

---

## Step 266: Test Coverage

```nim
# file: src/calculator.nim
# Module ที่จะเขียน tests สำหรับ coverage

type
  CalculatorError = object of ValueError

proc add*(a, b: int): int = a + b
proc subtract*(a, b: int): int = a - b
proc multiply*(a, b: int): int = a * b

proc divide*(a, b: int): int =
  if b == 0:
    raise newException(CalculatorError, "Division by zero")
  return a div b

proc power*(base: int, exp: int): int =
  if exp < 0:
    raise newException(CalculatorError, "Negative exponent not supported")
  if exp == 0: return 1
  var result = 1
  for i in 0..<exp:
    result *= base
  return result

proc factorial*(n: int): int64 =
  if n < 0:
    raise newException(CalculatorError, "Factorial of negative number")
  if n == 0 or n == 1: return 1
  var result: int64 = 1
  for i in 2..n:
    result *= i.int64
  return result

proc isPrime*(n: int): bool =
  if n < 2: return false
  if n == 2: return true
  if n mod 2 == 0: return false
  var i = 3
  while i * i <= n:
    if n mod i == 0: return false
    i += 2
  return true
```

```nim
# file: tests/test_calculator.nim
# เขียน tests เพื่อให้ coverage ครบถ้วน
import std/unittest
import ../src/calculator

suite "Calculator - Addition":
  test "positive numbers": check add(2, 3) == 5
  test "negative numbers": check add(-2, -3) == -5
  test "zero": check add(0, 5) == 5
  test "large numbers": check add(1_000_000, 2_000_000) == 3_000_000

suite "Calculator - Subtraction":
  test "basic": check subtract(10, 3) == 7
  test "result negative": check subtract(3, 10) == -7
  test "same numbers": check subtract(5, 5) == 0

suite "Calculator - Multiply":
  test "basic": check multiply(4, 5) == 20
  test "by zero": check multiply(10, 0) == 0
  test "negative": check multiply(-3, 4) == -12

suite "Calculator - Divide":
  test "even division": check divide(10, 2) == 5
  test "integer division": check divide(10, 3) == 3
  test "negative": check divide(-10, 2) == -5
  
  test "divide by zero":
    expect CalculatorError:
      discard divide(5, 0)

suite "Calculator - Power":
  test "basic": check power(2, 10) == 1024
  test "power of 0": check power(5, 0) == 1
  test "power of 1": check power(7, 1) == 7
  test "zero base": check power(0, 5) == 0
  
  test "negative exponent":
    expect CalculatorError:
      discard power(2, -1)

suite "Calculator - Factorial":
  test "zero": check factorial(0) == 1
  test "one": check factorial(1) == 1
  test "five": check factorial(5) == 120
  test "ten": check factorial(10) == 3_628_800
  
  test "negative":
    expect CalculatorError:
      discard factorial(-1)

suite "Calculator - IsPrime":
  test "not prime - 1": check isPrime(1) == false
  test "prime - 2": check isPrime(2) == true
  test "prime - 3": check isPrime(3) == true
  test "not prime - 4": check isPrime(4) == false
  test "prime - 17": check isPrime(17) == true
  test "not prime - 100": check isPrime(100) == false
  test "prime - 97": check isPrime(97) == true
  test "negative": check isPrime(-5) == false

# รันด้วย: nim c -r tests/test_calculator.nim
# ดู coverage: nim c --coverage tests/test_calculator.nim
```

---

## Step 267: TDD (Test-Driven Development)

```nim
# ===== TDD Demo: สร้าง Validator =====

# Step 1: เขียน tests ก่อน (RED)

# file: tests/test_validator.nim
import std/unittest

# Import module ที่ยังไม่มี
import ../src/validator

suite "Email Validator":
  test "valid emails pass":
    check isValidEmail("user@example.com") == true
    check isValidEmail("test.user@domain.co.uk") == true
    check isValidEmail("user+tag@example.com") == true
  
  test "invalid emails fail":
    check isValidEmail("") == false
    check isValidEmail("notanemail") == false
    check isValidEmail("@nodomain.com") == false
    check isValidEmail("no@domain") == false
    check isValidEmail("spaces in@email.com") == false

suite "Password Validator":
  test "valid passwords pass":
    check isValidPassword("MyPass123!") == true
    check isValidPassword("Str0ng#Pass") == true
  
  test "too short fails":
    check isValidPassword("Ab1!") == false
  
  test "no uppercase fails":
    check isValidPassword("mypass123!") == false
  
  test "no number fails":
    check isValidPassword("MyPassWord!") == false
  
  test "no special char fails":
    check isValidPassword("MyPassword123") == false

suite "Username Validator":
  test "valid usernames pass":
    check isValidUsername("alice") == true
    check isValidUsername("user123") == true
    check isValidUsername("my_user") == true
  
  test "too short fails":
    check isValidUsername("ab") == false
  
  test "too long fails":
    check isValidUsername("a".repeat(21)) == false
  
  test "special chars fail":
    check isValidUsername("user@name") == false
    check isValidUsername("user name") == false
```

```nim
# Step 2: สร้าง implementation (GREEN)

# file: src/validator.nim
import re, strutils

proc isValidEmail*(email: string): bool =
  if email.len == 0:
    return false
  
  # Basic email regex
  let emailPattern = re"^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$"
  
  if email.contains(" "):
    return false
  
  return email.match(emailPattern)

proc isValidPassword*(password: string): bool =
  if password.len < 8:
    return false
  
  var hasUpper = false
  var hasLower = false
  var hasDigit = false
  var hasSpecial = false
  
  const specialChars = "!@#$%^&*()_+-=[]{}|;':\",./<>?"
  
  for ch in password:
    if ch.isUpperAscii(): hasUpper = true
    elif ch.isLowerAscii(): hasLower = true
    elif ch.isDigit(): hasDigit = true
    elif ch in specialChars: hasSpecial = true
  
  return hasUpper and hasLower and hasDigit and hasSpecial

proc isValidUsername*(username: string): bool =
  if username.len < 3 or username.len > 20:
    return false
  
  for ch in username:
    if not (ch.isAlphaAscii() or ch.isDigit() or ch == '_'):
      return false
  
  return true

# Step 3: Refactor ถ้าจำเป็น
```

---

## Step 268: Property-Based Testing

```nim
# file: tests/test_property.nim
import std/unittest, random, strutils, sequtils

# Property-based testing: ทดสอบ properties ที่ควรเป็นจริงเสมอ

proc reverseString(s: string): string =
  var result = newString(s.len)
  for i, ch in s:
    result[s.len - 1 - i] = ch
  return result

proc sortSeq(s: seq[int]): seq[int] =
  result = s
  result.sort()

proc myAbs(n: int): int =
  if n < 0: -n else: n

# Property: reverse(reverse(s)) == s
proc propReverseInvolution(s: string): bool =
  reverseString(reverseString(s)) == s

# Property: reverse of empty = empty
proc propReverseEmpty(): bool =
  reverseString("") == ""

# Property: sorted sequence is non-decreasing
proc propSortedNonDecreasing(s: seq[int]): bool =
  let sorted = sortSeq(s)
  for i in 0..<sorted.len - 1:
    if sorted[i] > sorted[i+1]:
      return false
  return true

# Property: sorted preserves length
proc propSortPreservesLength(s: seq[int]): bool =
  sortSeq(s).len == s.len

# Property: abs(n) >= 0 always
proc propAbsNonNegative(n: int): bool =
  myAbs(n) >= 0

# Property: abs(-n) == abs(n)
proc propAbsSymmetric(n: int): bool =
  myAbs(n) == myAbs(-n)

suite "Property-Based Tests":
  const NUM_TESTS = 100
  
  test "reverse is involution":
    randomize()
    for _ in 0..<NUM_TESTS:
      let s = (0..rand(20)).mapIt(char(rand(25) + ord('a'))).join()
      check propReverseInvolution(s)
  
  test "reverse empty string":
    check propReverseEmpty()
  
  test "sort produces non-decreasing sequence":
    randomize()
    for _ in 0..<NUM_TESTS:
      let s = (0..rand(20)).mapIt(rand(1000) - 500)
      check propSortedNonDecreasing(s)
  
  test "sort preserves length":
    randomize()
    for _ in 0..<NUM_TESTS:
      let s = (0..rand(20)).mapIt(rand(1000))
      check propSortPreservesLength(s)
  
  test "abs is always non-negative":
    randomize()
    for _ in 0..<NUM_TESTS:
      let n = rand(2_000_000) - 1_000_000
      check propAbsNonNegative(n)
  
  test "abs is symmetric":
    randomize()
    for _ in 0..<NUM_TESTS:
      let n = rand(1_000_000) - 500_000
      check propAbsSymmetric(n)
```

---

## Step 269: Test Organization และ Best Practices

```nim
# file: tests/test_organization.nim
import std/unittest, strformat

# ===== Test helper macros =====

template assertApprox*(actual, expected, tolerance: float) =
  let diff = abs(actual - expected)
  if diff > tolerance:
    fail(&"Expected {expected} ± {tolerance}, got {actual} (diff: {diff})")

template assertContains*(haystack: string, needle: string) =
  if not haystack.contains(needle):
    fail(&"Expected '{haystack}' to contain '{needle}'")

template assertEmpty*(collection: untyped) =
  if collection.len != 0:
    fail(&"Expected empty collection, got {collection.len} items")

template assertNotEmpty*(collection: untyped) =
  if collection.len == 0:
    fail("Expected non-empty collection")

template assertBetween*(value, low, high: untyped) =
  if value < low or value > high:
    fail(&"Expected {value} to be between {low} and {high}")

# ===== Parameterized tests =====

proc testDivide(a, b, expected: int) =
  let result = a div b
  if result != expected:
    fail(&"{a} / {b} = {result}, expected {expected}")

suite "Parameterized Division Tests":
  test "multiple cases":
    let cases = @[
      (10, 2, 5),
      (9, 3, 3),
      (100, 4, 25),
      (7, 2, 3),  # integer division
      (-10, 2, -5)
    ]
    
    for (a, b, expected) in cases:
      testDivide(a, b, expected)

# ===== Custom assertions =====

suite "Custom Assertion Tests":
  test "approximate equality":
    assertApprox(3.14159, 3.14, 0.01)
    assertApprox(1.0/3.0, 0.333, 0.001)
  
  test "string contains":
    assertContains("Hello World", "World")
    assertContains("nim is great", "great")
  
  test "empty collections":
    let emptySeq: seq[int] = @[]
    assertEmpty(emptySeq)
  
  test "between range":
    assertBetween(5, 1, 10)
    assertBetween(50, 0, 100)

# ===== Test grouping =====

# Group related tests logically
suite "User Registration Validation":
  # Username tests
  suite "Username validation":
    test "min length 3": check "ab".len < 3
    test "max length 20": check "a".repeat(21).len > 20
  
  # Email tests
  suite "Email validation":
    test "must contain @": check "@" in "test@example.com"
    test "must have domain": check "." in "example.com"
```

---

## Step 270: Full Test Suite สำหรับ REST API

```nim
# file: tests/test_rest_api_full.nim
import std/unittest, asyncdispatch, asynchttpserver, httpclient
import json, strformat, tables, strutils, times

# ===== Application Code (simplified) =====

type
  AppUser = object
    id: int
    username: string
    email: string
    createdAt: DateTime

  App = ref object
    users: Table[int, AppUser]
    nextId: int
    tokens: Table[string, int]  # token -> userId

proc newApp(): App =
  var app = App(
    users: initTable[int, AppUser](),
    nextId: 1,
    tokens: initTable[string, int]()
  )
  
  # Add test data
  app.users[1] = AppUser(id: 1, username: "alice", email: "alice@example.com", createdAt: now())
  app.users[2] = AppUser(id: 2, username: "bob", email: "bob@example.com", createdAt: now())
  app.nextId = 3
  app.tokens["valid_token"] = 1  # alice's token
  
  return app

proc handleApiRequest(app: App, req: Request) {.async.} =
  let headers = newHttpHeaders([("Content-Type", "application/json")])
  
  # Auth middleware
  proc getAuthUserId(): int =
    let auth = req.headers.getOrDefault("Authorization", "")
    if not auth.startsWith("Bearer "):
      return 0
    let token = auth[7..^1]
    return app.tokens.getOrDefault(token, 0)
  
  case req.url.path
  of "/api/health":
    await req.respond(Http200, $(%*{"status": "ok"}), headers)
  
  of "/api/users":
    if req.reqMethod == HttpGet:
      var userList = newJArray()
      for id, user in app.users:
        userList.add(%*{
          "id": user.id,
          "username": user.username,
          "email": user.email
        })
      await req.respond(Http200, $(%*{"users": userList, "total": app.users.len}), headers)
    
    elif req.reqMethod == HttpPost:
      let userId = getAuthUserId()
      if userId == 0:
        await req.respond(Http401, $(%*{"error": "Unauthorized"}), headers)
        return
      
      let body = parseJson(req.body)
      let username = body["username"].getStr()
      let email = body["email"].getStr()
      
      if username.len == 0 or email.len == 0:
        await req.respond(Http400, $(%*{"error": "username and email required"}), headers)
        return
      
      let id = app.nextId
      inc app.nextId
      app.users[id] = AppUser(id: id, username: username, email: email, createdAt: now())
      
      await req.respond(Http201, $(%*{
        "id": id,
        "username": username,
        "email": email
      }), headers)
  
  elif req.url.path.startsWith("/api/users/"):
    let idStr = req.url.path[11..^1]
    let id = try: parseInt(idStr) except: 0
    
    if id == 0 or id notin app.users:
      await req.respond(Http404, $(%*{"error": "User not found"}), headers)
      return
    
    let user = app.users[id]
    
    if req.reqMethod == HttpGet:
      await req.respond(Http200, $(%*{
        "id": user.id,
        "username": user.username,
        "email": user.email
      }), headers)
    
    elif req.reqMethod == HttpDelete:
      let userId = getAuthUserId()
      if userId == 0:
        await req.respond(Http401, $(%*{"error": "Unauthorized"}), headers)
        return
      
      app.users.del(id)
      await req.respond(Http204, "", headers)
  
  else:
    await req.respond(Http404, $(%*{"error": "Not found"}), headers)

# ===== Test Infrastructure =====

const TEST_PORT = 18081

proc startApp(): Future[App] {.async.} =
  let app = newApp()
  var server = newAsyncHttpServer()
  
  proc handler(req: Request) {.async.} =
    await handleApiRequest(app, req)
  
  asyncCheck server.serve(Port(TEST_PORT), handler)
  await sleepAsync(100)
  return app

type
  ApiTestClient = ref object
    http: AsyncHttpClient
    baseUrl: string

proc newApiTestClient(token = ""): ApiTestClient =
  var headers = newHttpHeaders([("Content-Type", "application/json")])
  if token.len > 0:
    headers["Authorization"] = &"Bearer {token}"
  ApiTestClient(
    http: newAsyncHttpClient(headers = headers),
    baseUrl: &"http://localhost:{TEST_PORT}"
  )

proc apiGet(c: ApiTestClient, path: string): Future[(int, JsonNode)] {.async.} =
  let resp = await c.http.get(c.baseUrl & path)
  let body = await resp.body
  let status = resp.status.split(" ")[0].parseInt()
  let json = try: parseJson(body) except: newJNull()
  return (status, json)

proc apiPost(c: ApiTestClient, path: string, data: JsonNode): Future[(int, JsonNode)] {.async.} =
  let resp = await c.http.post(c.baseUrl & path, $data)
  let body = await resp.body
  let status = resp.status.split(" ")[0].parseInt()
  let json = try: parseJson(body) except: newJNull()
  return (status, json)

proc apiDelete(c: ApiTestClient, path: string): Future[int] {.async.} =
  let resp = await c.http.delete(c.baseUrl & path)
  return resp.status.split(" ")[0].parseInt()

# ===== Full Test Suite =====

suite "REST API Full Test Suite":
  var app: App
  var anonClient, aliceClient: ApiTestClient
  
  setup:
    proc doSetup() {.async.} =
      app = await startApp()
      anonClient = newApiTestClient()
      aliceClient = newApiTestClient("valid_token")
    waitFor doSetup()
  
  test "health endpoint":
    proc doTest() {.async.} =
      let (status, body) = await anonClient.apiGet("/api/health")
      check status == 200
      check body["status"].getStr() == "ok"
    waitFor doTest()
  
  test "list users - no auth required":
    proc doTest() {.async.} =
      let (status, body) = await anonClient.apiGet("/api/users")
      check status == 200
      check body["users"].len == 2
      check body["total"].getInt() == 2
    waitFor doTest()
  
  test "get specific user":
    proc doTest() {.async.} =
      let (status, body) = await anonClient.apiGet("/api/users/1")
      check status == 200
      check body["id"].getInt() == 1
      check body["username"].getStr() == "alice"
    waitFor doTest()
  
  test "get nonexistent user returns 404":
    proc doTest() {.async.} =
      let (status, _) = await anonClient.apiGet("/api/users/999")
      check status == 404
    waitFor doTest()
  
  test "create user requires auth":
    proc doTest() {.async.} =
      let (status, body) = await anonClient.apiPost("/api/users", %*{
        "username": "charlie",
        "email": "charlie@example.com"
      })
      check status == 401
      check body["error"].getStr().contains("Unauthorized")
    waitFor doTest()
  
  test "create user with auth succeeds":
    proc doTest() {.async.} =
      let (status, body) = await aliceClient.apiPost("/api/users", %*{
        "username": "charlie",
        "email": "charlie@example.com"
      })
      check status == 201
      check body["username"].getStr() == "charlie"
      check body["id"].getInt() > 2
      
      # Verify in list
      let (listStatus, listBody) = await anonClient.apiGet("/api/users")
      check listStatus == 200
      check listBody["total"].getInt() == 3
    waitFor doTest()
  
  test "create user missing fields returns 400":
    proc doTest() {.async.} =
      let (status, _) = await aliceClient.apiPost("/api/users", %*{
        "username": "incomplete"
        # missing email
      })
      check status == 400
    waitFor doTest()
  
  test "delete user requires auth":
    proc doTest() {.async.} =
      let status = await anonClient.apiDelete("/api/users/2")
      check status == 401
    waitFor doTest()
  
  test "delete user with auth":
    proc doTest() {.async.} =
      let status = await aliceClient.apiDelete("/api/users/2")
      check status == 204
      
      # Verify deleted
      let (getStatus, _) = await anonClient.apiGet("/api/users/2")
      check getStatus == 404
    waitFor doTest()
  
  test "unknown endpoint returns 404":
    proc doTest() {.async.} =
      let (status, body) = await anonClient.apiGet("/api/unknown")
      check status == 404
      check body["error"].getStr() == "Not found"
    waitFor doTest()

# รัน: nimble test หรือ nim c -r tests/test_rest_api_full.nim
```

---

## 📝 สรุป Part 19

| Step | หัวข้อ | สิ่งที่เรียนรู้ |
|------|--------|----------------|
| 256 | std/unittest Basics | check, expect, test, suite |
| 257 | String & Collection Tests | map, filter, contains |
| 258 | Setup/Teardown | before/after each test |
| 259 | Exception Testing | expect, try/except patterns |
| 260 | Async Tests | waitFor, async test patterns |
| 261 | Mocking | Interface mocking, method dispatch |
| 262 | Test Fixtures | Builder pattern, data factories |
| 263 | Integration Tests | Real HTTP server testing |
| 264 | API Test Helpers | TestResponse, assertions |
| 265 | Database Tests | DB setup/teardown, SQL testing |
| 266 | Test Coverage | Complete path coverage |
| 267 | TDD | Red-Green-Refactor cycle |
| 268 | Property-Based | Invariants, random testing |
| 269 | Organization | Custom assertions, parameterized |
| 270 | Full Test Suite | End-to-end API testing |

---

## Navigation

- [← Part 18: Docker](part_18_docker.md)
- [→ Part 20: Performance](part_20_performance.md)
- [กลับ README](../README.md)
