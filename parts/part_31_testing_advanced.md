# Part 31: Advanced Testing
## Steps 436-450: Testing ระดับ Production

---

## 🎯 เป้าหมายของ Part นี้

- Unit tests ด้วย `unittest`
- Test fixtures และ setup/teardown
- Mocking และ stub
- Integration tests
- Property-based testing
- Test coverage
- BDD-style tests

---

## Step 436: unittest Basics

```nim
import unittest, strformat

# ==============================
# Unit under test
# ==============================

func add(a, b: int): int = a + b
func multiply(a, b: int): int = a * b
func divide(a, b: float): float =
  if b == 0.0: raise newException(DivByZeroDefect, "Division by zero")
  a / b

func isPrime(n: int): bool =
  if n < 2: return false
  if n == 2: return true
  if n mod 2 == 0: return false
  var i = 3
  while i * i <= n:
    if n mod i == 0: return false
    i += 2
  return true

func reverseString(s: string): string =
  var r = s
  var i = 0
  var j = r.len - 1
  while i < j:
    swap(r[i], r[j])
    inc i
    dec j
  return r

# ==============================
# Test Suites
# ==============================

suite "Arithmetic":
  test "add positive numbers":
    check add(2, 3) == 5
    check add(0, 0) == 0
    check add(-1, 1) == 0
  
  test "multiply":
    check multiply(3, 4) == 12
    check multiply(0, 100) == 0
    check multiply(-2, -3) == 6
  
  test "divide":
    check divide(10.0, 2.0) == 5.0
    check divide(1.0, 3.0) - 0.3333 < 0.001
  
  test "divide by zero raises exception":
    expect(DivByZeroDefect):
      discard divide(1.0, 0.0)

suite "Prime Numbers":
  test "small primes":
    check isPrime(2) == true
    check isPrime(3) == true
    check isPrime(5) == true
    check isPrime(7) == true
    check isPrime(11) == true
    check isPrime(13) == true
  
  test "non-primes":
    check isPrime(0) == false
    check isPrime(1) == false
    check isPrime(4) == false
    check isPrime(9) == false
    check isPrime(100) == false
  
  test "larger primes":
    check isPrime(97) == true
    check isPrime(101) == true
    check isPrime(103) == true
  
  test "count primes below 20":
    let primes = (2..19).toSeq().filterIt(isPrime(it))
    check primes.len == 8  # 2,3,5,7,11,13,17,19

suite "String Operations":
  test "reverse empty string":
    check reverseString("") == ""
  
  test "reverse single char":
    check reverseString("a") == "a"
  
  test "reverse word":
    check reverseString("hello") == "olleh"
    check reverseString("nim") == "min"
  
  test "reverse palindrome":
    check reverseString("racecar") == "racecar"
    check reverseString("madam") == "madam"
```

---

## Step 437: Test Fixtures and Helpers

```nim
import unittest, tables, json, times, options

# ==============================
# Domain types
# ==============================

type
  User = object
    id: int
    name: string
    email: string
    role: string
    createdAt: float

  UserRepo = object
    users: Table[int, User]
    nextId: int

# ==============================
# Fixtures
# ==============================

proc newTestRepo(): UserRepo =
  var repo = UserRepo(users: initTable[int, User](), nextId: 1)
  repo.users[1] = User(id: 1, name: "Alice", email: "alice@test.com",
                       role: "admin", createdAt: 1000.0)
  repo.users[2] = User(id: 2, name: "Bob", email: "bob@test.com",
                       role: "user", createdAt: 2000.0)
  repo.users[3] = User(id: 3, name: "Charlie", email: "charlie@test.com",
                       role: "user", createdAt: 3000.0)
  repo.nextId = 4
  return repo

proc makeUser(name: string, email: string = "", role: string = "user"): User =
  let actualEmail = if email.len > 0: email else: name.toLowerAscii() & "@test.com"
  User(id: 0, name: name, email: actualEmail, role: role, createdAt: epochTime())

# ==============================
# Repository functions to test
# ==============================

proc findById(repo: UserRepo, id: int): Option[User] =
  if id in repo.users:
    return some(repo.users[id])
  return none(User)

proc findByEmail(repo: UserRepo, email: string): Option[User] =
  for _, user in repo.users:
    if user.email == email:
      return some(user)
  return none(User)

proc create(repo: var UserRepo, user: User): User =
  var newUser = user
  newUser.id = repo.nextId
  inc repo.nextId
  repo.users[newUser.id] = newUser
  return newUser

proc findAll(repo: UserRepo): seq[User] =
  for _, user in repo.users:
    result.add(user)

proc deleteById(repo: var UserRepo, id: int): bool =
  if id in repo.users:
    repo.users.del(id)
    return true
  return false

proc countByRole(repo: UserRepo, role: string): int =
  for _, user in repo.users:
    if user.role == role:
      inc result

# ==============================
# Tests with fixtures
# ==============================

suite "UserRepo":
  var repo: UserRepo
  
  setup:
    repo = newTestRepo()
  
  test "findById - existing user":
    let user = repo.findById(1)
    check user.isSome
    check user.get().name == "Alice"
    check user.get().email == "alice@test.com"
  
  test "findById - non-existent":
    let user = repo.findById(999)
    check user.isNone
  
  test "findByEmail":
    let user = repo.findByEmail("bob@test.com")
    check user.isSome
    check user.get().name == "Bob"
  
  test "create user":
    let newUser = makeUser("David")
    let created = repo.create(newUser)
    check created.id == 4
    check created.name == "David"
    
    let found = repo.findById(4)
    check found.isSome
    check found.get().name == "David"
  
  test "findAll returns all users":
    let users = repo.findAll()
    check users.len == 3
  
  test "deleteById - success":
    let deleted = repo.deleteById(2)
    check deleted == true
    check repo.findById(2).isNone
    check repo.findAll().len == 2
  
  test "deleteById - non-existent":
    let deleted = repo.deleteById(999)
    check deleted == false
    check repo.findAll().len == 3
  
  test "countByRole":
    check repo.countByRole("admin") == 1
    check repo.countByRole("user") == 2
    check repo.countByRole("superadmin") == 0
  
  test "isolation - changes in one test don't affect others":
    discard repo.create(makeUser("Extra"))
    check repo.findAll().len == 4
    # After this test, setup runs again, so next test gets fresh repo
```

---

## Step 438: Mocking

```nim
import unittest, asyncdispatch, options, json

# ==============================
# Interfaces (using object variants for mock)
# ==============================

type
  EmailSender = ref object of RootObj

method send(sender: EmailSender, to, subject, body: string): bool {.base.} =
  return false

# Real implementation
type
  SmtpSender = ref object of EmailSender
    host: string
    port: int

method send(sender: SmtpSender, to, subject, body: string): bool =
  echo fmt"[SMTP] Sending to {to}: {subject}"
  return true  # In real app, connect to SMTP

# Mock implementation
type
  MockEmailSender = ref object of EmailSender
    sentEmails: seq[tuple[to, subject, body: string]]
    shouldFail: bool

method send(sender: MockEmailSender, to, subject, body: string): bool =
  if sender.shouldFail:
    return false
  sender.sentEmails.add((to, subject, body))
  return true

proc newMockEmailSender(shouldFail: bool = false): MockEmailSender =
  MockEmailSender(sentEmails: @[], shouldFail: shouldFail)

# Service using the email sender
type
  UserService = object
    emailSender: EmailSender

proc notifyRegistration(svc: UserService, userEmail, userName: string): bool =
  return svc.emailSender.send(
    userEmail,
    "Welcome to our platform!",
    fmt"Hello {userName}, thank you for registering."
  )

proc notifyPasswordReset(svc: UserService, userEmail, resetToken: string): bool =
  return svc.emailSender.send(
    userEmail,
    "Password Reset Request",
    fmt"Your reset token is: {resetToken}"
  )

# ==============================
# Tests with mocks
# ==============================

suite "UserService with Mock Email":
  test "registration sends welcome email":
    let mockEmail = newMockEmailSender()
    let svc = UserService(emailSender: mockEmail)
    
    let result = svc.notifyRegistration("user@example.com", "Alice")
    
    check result == true
    check mockEmail.sentEmails.len == 1
    check mockEmail.sentEmails[0].to == "user@example.com"
    check "Welcome" in mockEmail.sentEmails[0].subject
    check "Alice" in mockEmail.sentEmails[0].body
  
  test "password reset sends email with token":
    let mockEmail = newMockEmailSender()
    let svc = UserService(emailSender: mockEmail)
    
    let result = svc.notifyPasswordReset("user@example.com", "tok-abc123")
    
    check result == true
    check mockEmail.sentEmails.len == 1
    check "tok-abc123" in mockEmail.sentEmails[0].body
  
  test "handles email send failure":
    let mockEmail = newMockEmailSender(shouldFail = true)
    let svc = UserService(emailSender: mockEmail)
    
    let result = svc.notifyRegistration("user@example.com", "Bob")
    
    check result == false
    check mockEmail.sentEmails.len == 0
  
  test "multiple notifications":
    let mockEmail = newMockEmailSender()
    let svc = UserService(emailSender: mockEmail)
    
    discard svc.notifyRegistration("alice@ex.com", "Alice")
    discard svc.notifyRegistration("bob@ex.com", "Bob")
    discard svc.notifyPasswordReset("alice@ex.com", "reset-token")
    
    check mockEmail.sentEmails.len == 3
    check mockEmail.sentEmails[0].to == "alice@ex.com"
    check mockEmail.sentEmails[1].to == "bob@ex.com"
    check mockEmail.sentEmails[2].to == "alice@ex.com"
```

---

## Step 439: Integration Tests (HTTP)

```nim
import asyncdispatch, asynchttpserver, httpclient, json, unittest, strutils

# ==============================
# App under test
# ==============================

var testItems: seq[string] = @["item1", "item2"]

proc testAppHandler(req: Request) {.async.} =
  case req.url.path
  of "/items":
    if req.reqMethod == HttpGet:
      await req.respond(Http200, $(%testItems),
        newHttpHeaders([("Content-Type", "application/json")]))
    elif req.reqMethod == HttpPost:
      let item = parseJson(req.body)["name"].getStr()
      testItems.add(item)
      await req.respond(Http201, $(%*{"name": item}),
        newHttpHeaders([("Content-Type", "application/json")]))
  of "/items/clear":
    testItems = @[]
    await req.respond(Http200, """{"cleared":true}""",
      newHttpHeaders([("Content-Type", "application/json")]))
  else:
    await req.respond(Http404, """{"error":"not found"}""",
      newHttpHeaders([("Content-Type", "application/json")]))

# ==============================
# Integration test helpers
# ==============================

type
  TestClient = object
    baseUrl: string
    client: AsyncHttpClient

proc newTestClient(baseUrl: string): TestClient =
  TestClient(baseUrl: baseUrl, client: newAsyncHttpClient())

proc get(tc: TestClient, path: string): Future[JsonNode] {.async.} =
  let resp = await tc.client.get(tc.baseUrl & path)
  let body = await resp.body
  return parseJson(body)

proc post(tc: TestClient, path: string, data: JsonNode): Future[JsonNode] {.async.} =
  tc.client.headers = newHttpHeaders([("Content-Type", "application/json")])
  let resp = await tc.client.post(tc.baseUrl & path, $data)
  let body = await resp.body
  return parseJson(body)

# ==============================
# Integration tests (simulated)
# ==============================

proc runIntegrationTests() {.async.} =
  echo "\n=== Integration Tests ==="
  
  # Reset state
  testItems = @["item1", "item2"]
  
  # Test 1: GET /items
  var server = newAsyncHttpServer()
  let port = Port(18181)
  
  asyncCheck server.serve(port, testAppHandler)
  await sleepAsync(50)
  
  let client = newTestClient("http://127.0.0.1:18181")
  
  let items = await client.get("/items")
  assert items.kind == JArray, "Should return array"
  assert items.len == 2, fmt"Should have 2 items, got {items.len}"
  echo "  ✓ GET /items returns existing items"
  
  # Test 2: POST /items
  let created = await client.post("/items", %*{"name": "item3"})
  assert "name" in created, "Should return created item"
  assert created["name"].getStr() == "item3"
  echo "  ✓ POST /items creates new item"
  
  # Test 3: Verify item was added
  let itemsAfter = await client.get("/items")
  assert itemsAfter.len == 3, "Should have 3 items"
  echo "  ✓ Added item persisted"
  
  echo "\n  All integration tests passed!"
  server.close()

waitFor runIntegrationTests()
```

---

## Step 440-450: Complete Test Suite

```nim
# complete_test_suite.nim

import unittest, strutils, strformat, tables, options, math, sequtils, json

# ==============================
# Property-based testing helpers
# ==============================

proc generateStrings(count: int = 100): seq[string] =
  result = @[]
  for i in 0..<count:
    let len = i mod 20
    var s = ""
    for j in 0..<len:
      s.add(char(ord('a') + (i + j) mod 26))
    result.add(s)

proc generateInts(count: int = 100, min: int = -1000, max: int = 1000): seq[int] =
  result = @[]
  for i in 0..<count:
    result.add(min + (i * 37 + 13) mod (max - min + 1))

# ==============================
# Functions under test
# ==============================

func clamp(x, lo, hi: int): int =
  if x < lo: lo
  elif x > hi: hi
  else: x

func wordCount(text: string): Table[string, int] =
  result = initTable[string, int]()
  for word in text.toLowerAscii().split({' ', ',', '.', '!', '?', '\n', '\t'}):
    if word.len > 0:
      result[word] = result.getOrDefault(word, 0) + 1

func truncate(s: string, maxLen: int): string =
  if s.len <= maxLen: s
  else: s[0..<maxLen] & "..."

func parseIntSafe(s: string): Option[int] =
  try:
    some(parseInt(s))
  except ValueError:
    none(int)

func toBase64Len(inputLen: int): int =
  ((inputLen + 2) div 3) * 4

# ==============================
# Property-based tests
# ==============================

suite "Property-Based: clamp":
  test "result is always in [lo, hi]":
    for x in generateInts(100, -2000, 2000):
      let lo = -100
      let hi = 100
      let result = clamp(x, lo, hi)
      check lo <= result and result <= hi
  
  test "idempotent: clamping twice = clamping once":
    for x in generateInts(50):
      check clamp(clamp(x, -10, 10), -10, 10) == clamp(x, -10, 10)
  
  test "identity when already in range":
    for x in -100..100:
      check clamp(x, -100, 100) == x

suite "Property-Based: reverseString":
  test "reverse(reverse(s)) == s":
    for s in generateStrings(50):
      check reverseString(reverseString(s)) == s
  
  test "reverse preserves length":
    for s in generateStrings(50):
      check reverseString(s).len == s.len

suite "Word Count":
  test "empty string":
    let wc = wordCount("")
    check wc.len == 0
  
  test "single word":
    let wc = wordCount("hello")
    check wc.len == 1
    check wc["hello"] == 1
  
  test "word frequency":
    let wc = wordCount("the quick brown fox jumps over the lazy dog the")
    check wc["the"] == 3
    check wc["quick"] == 1
    check wc.len == 8  # unique words
  
  test "case insensitive":
    let wc = wordCount("Hello hello HELLO")
    check wc["hello"] == 3

suite "Truncate":
  test "short string unchanged":
    check truncate("hello", 10) == "hello"
    check truncate("hi", 2) == "hi"
  
  test "long string truncated with ...":
    let result = truncate("Hello, World!", 5)
    check result == "Hello..."
    check result.len == 8
  
  test "exact length unchanged":
    check truncate("hello", 5) == "hello"

suite "parseIntSafe":
  test "valid integers":
    check parseIntSafe("42").get() == 42
    check parseIntSafe("-100").get() == -100
    check parseIntSafe("0").get() == 0
  
  test "invalid strings":
    check parseIntSafe("abc").isNone
    check parseIntSafe("12.5").isNone
    check parseIntSafe("").isNone
    check parseIntSafe("  ").isNone
  
  test "boundary values":
    check parseIntSafe("2147483647").isSome
    check parseIntSafe("-2147483648").isSome

suite "Base64 Length":
  test "formula is correct":
    check toBase64Len(0) == 0
    check toBase64Len(1) == 4
    check toBase64Len(2) == 4
    check toBase64Len(3) == 4
    check toBase64Len(4) == 8
    check toBase64Len(6) == 8

# ==============================
# JSON Schema Validation Tests
# ==============================

type
  JsonSchema = object
    required: seq[string]
    types: Table[string, string]
    minLength: Table[string, int]
    maxLength: Table[string, int]

proc validateJson(data: JsonNode, schema: JsonSchema): seq[string] =
  var errors: seq[string] = @[]
  
  for field in schema.required:
    if field notin data or data[field].kind == JNull:
      errors.add(fmt"'{field}' is required")
      continue
    
    if field in schema.types:
      let expectedType = schema.types[field]
      let actualType = case data[field].kind
        of JString: "string"
        of JInt: "integer"
        of JFloat: "number"
        of JBool: "boolean"
        of JArray: "array"
        of JObject: "object"
        else: "null"
      
      if actualType != expectedType:
        errors.add(fmt"'{field}' must be {expectedType}, got {actualType}")
    
    if field in schema.minLength and data[field].kind == JString:
      if data[field].getStr().len < schema.minLength[field]:
        errors.add(fmt"'{field}' min length is {schema.minLength[field]}")
    
    if field in schema.maxLength and data[field].kind == JString:
      if data[field].getStr().len > schema.maxLength[field]:
        errors.add(fmt"'{field}' max length is {schema.maxLength[field]}")
  
  return errors

let userSchema = JsonSchema(
  required: @["name", "email", "password"],
  types: {"name": "string", "email": "string", "password": "string"}.toTable(),
  minLength: {"name": 2, "email": 5, "password": 8}.toTable(),
  maxLength: {"name": 100, "email": 255, "password": 100}.toTable()
)

suite "JSON Schema Validation":
  test "valid user data":
    let data = %*{"name": "Alice", "email": "alice@example.com", "password": "secret123"}
    let errors = validateJson(data, userSchema)
    check errors.len == 0
  
  test "missing required field":
    let data = %*{"name": "Alice", "email": "alice@example.com"}
    let errors = validateJson(data, userSchema)
    check errors.len == 1
    check "password" in errors[0]
  
  test "wrong type":
    let data = %*{"name": 123, "email": "alice@ex.com", "password": "pass1234"}
    let errors = validateJson(data, userSchema)
    check errors.len == 1
    check "must be string" in errors[0]
  
  test "too short password":
    let data = %*{"name": "Alice", "email": "alice@ex.com", "password": "short"}
    let errors = validateJson(data, userSchema)
    check errors.len == 1
    check "min length" in errors[0]
  
  test "multiple errors":
    let data = %*{"name": "A"}  # missing email, password, name too short
    let errors = validateJson(data, userSchema)
    check errors.len >= 3
```

---

## 📝 สรุป Part 31

| Steps | หัวข้อ |
|-------|--------|
| 436 | unittest basics - check, expect, suites |
| 437 | Test fixtures with setup/teardown |
| 438 | Mocking with method dispatch |
| 439 | Integration tests with HTTP server |
| 440-450 | Property-based testing, JSON validation tests |

---

**← [Part 30: GraphQL](part_30_graphql.md) | [Part 32: Security →](part_32_security.md)**
