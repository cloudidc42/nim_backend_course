# Part 59: Testing Framework
## Steps 856-870: Testing Backend Code in Nim

---

## 🎯 เป้าหมายของ Part นี้

- Unit testing with assertions
- Table-driven tests
- Mock objects & dependency injection
- HTTP API testing
- Property-based testing
- Test coverage tracking

---

## Step 856: Unit Testing Core

```nim
import strformat, times, sequtils, json, strutils, options, math, tables, algorithm

# ============================
# Test framework
# ============================

type
  TestResult = enum
    trPass, trFail, trSkip

  TestCase = object
    name: string
    result: TestResult
    error: string
    durationMs: float

  TestSuite = object
    name: string
    cases: seq[TestCase]
    beforeEach: proc()
    afterEach: proc()

  TestRunner = object
    suites: seq[TestSuite]
    passed: int
    failed: int
    skipped: int

var runner = TestRunner(suites: @[])

template test(suiteName, caseName: string, body: untyped) =
  block:
    let start = epochTime()
    var tc = TestCase(name: caseName)
    try:
      body
      tc.result = trPass
    except AssertionDefect as e:
      tc.result = trFail
      tc.error = e.msg
    except CatchableError as e:
      tc.result = trFail
      tc.error = fmt"Exception: {e.msg}"
    tc.durationMs = (epochTime() - start) * 1000

    let icon = case tc.result
      of trPass: "✓"
      of trFail: "✗"
      of trSkip: "○"
    echo fmt"  {icon} {caseName} ({tc.durationMs:.1f}ms)"
    if tc.result == trFail:
      echo fmt"    Error: {tc.error}"

    # Track
    case tc.result
    of trPass: inc runner.passed
    of trFail: inc runner.failed
    of trSkip: inc runner.skipped

proc describe(name: string, body: proc()) =
  echo fmt"\n{name}"
  body()

proc printSummary() =
  let total = runner.passed + runner.failed + runner.skipped
  echo fmt"\n{'=' * 40}"
  echo fmt"Tests: {total} total, {runner.passed} passed, {runner.failed} failed, {runner.skipped} skipped"
  if runner.failed == 0:
    echo "All tests passed! ✓"
  else:
    echo fmt"{runner.failed} test(s) failed! ✗"

# ============================
# Assertion helpers
# ============================

proc assertEqual[T](actual, expected: T, msg = "") =
  if actual != expected:
    let detail = if msg.len > 0: msg else: fmt"expected {expected}, got {actual}"
    raise newException(AssertionDefect, detail)

proc assertNotEqual[T](actual, expected: T, msg = "") =
  if actual == expected:
    let detail = if msg.len > 0: msg else: fmt"expected NOT {expected}"
    raise newException(AssertionDefect, detail)

proc assertTrue(value: bool, msg = "expected true") =
  if not value:
    raise newException(AssertionDefect, msg)

proc assertFalse(value: bool, msg = "expected false") =
  if value:
    raise newException(AssertionDefect, msg)

proc assertNone[T](opt: Option[T], msg = "expected None") =
  if opt.isSome:
    raise newException(AssertionDefect, msg)

proc assertSome[T](opt: Option[T], msg = "expected Some"): T =
  if opt.isNone:
    raise newException(AssertionDefect, msg)
  return opt.get()

proc assertContains(haystack, needle: string, msg = "") =
  if needle notin haystack:
    let detail = if msg.len > 0: msg else: fmt"'{needle}' not found in '{haystack}'"
    raise newException(AssertionDefect, detail)

proc assertLen[T](s: seq[T], expected: int, msg = "") =
  if s.len != expected:
    let detail = if msg.len > 0: msg else: fmt"expected len {expected}, got {s.len}"
    raise newException(AssertionDefect, detail)

proc assertNear(actual, expected, epsilon: float, msg = "") =
  if abs(actual - expected) > epsilon:
    let detail = if msg.len > 0: msg else: fmt"expected ~{expected} (±{epsilon}), got {actual}"
    raise newException(AssertionDefect, detail)

proc assertRaises(expectedMsg: string, body: proc()) =
  var raised = false
  try:
    body()
  except CatchableError as e:
    raised = true
    if expectedMsg.len > 0 and expectedMsg notin e.msg:
      raise newException(AssertionDefect,
        fmt"Expected error containing '{expectedMsg}', got '{e.msg}'")
  if not raised:
    raise newException(AssertionDefect, "Expected exception was not raised")

# ============================
# Table-driven tests
# ============================

type
  TestTableRow[I, O] = object
    input: I
    expected: O
    name: string

proc tableTest[I, O](suiteName: string, cases: seq[TestTableRow[I, O]],
                      fn: proc(input: I): O) =
  echo fmt"\n{suiteName} (table-driven)"
  for tc in cases:
    let start = epochTime()
    var result = trPass
    var errMsg = ""

    try:
      let actual = fn(tc.input)
      if actual != tc.expected:
        result = trFail
        errMsg = fmt"expected {tc.expected}, got {actual}"
    except CatchableError as e:
      result = trFail
      errMsg = e.msg

    let duration = (epochTime() - start) * 1000
    let icon = if result == trPass: "✓" else: "✗"
    echo fmt"  {icon} {tc.name} ({duration:.1f}ms)"
    if result == trFail: echo fmt"    {errMsg}"

    case result
    of trPass: inc runner.passed
    of trFail: inc runner.failed
    of trSkip: inc runner.skipped

# ============================
# Mock objects
# ============================

type
  MockCall = object
    args: seq[string]
    returnValue: string
    calledAt: float

  MockFunction = object
    name: string
    calls: seq[MockCall]
    returnValues: seq[string]
    idx: int

  MockEmailSender = object
    sent: seq[tuple[to, subject, body: string]]
    shouldFail: bool

proc newMock(name: string, returns: seq[string] = @[]): MockFunction =
  MockFunction(name: name, calls: @[], returnValues: returns, idx: 0)

proc callMock(m: var MockFunction, args: seq[string]): string =
  m.calls.add(MockCall(args: args, calledAt: epochTime(),
    returnValue: if m.idx < m.returnValues.len: m.returnValues[m.idx] else: ""))
  if m.idx < m.returnValues.len:
    result = m.returnValues[m.idx]
    inc m.idx

proc wasCalledWith(m: MockFunction, args: seq[string]): bool =
  m.calls.anyIt(it.args == args)

proc callCount(m: MockFunction): int = m.calls.len

proc mockSendEmail(sender: var MockEmailSender, to, subject, body: string): bool =
  if sender.shouldFail: return false
  sender.sent.add((to, subject, body))
  return true

# ============================
# HTTP response mock for API testing
# ============================

type
  MockRequest = object
    method_: string
    path: string
    body: string
    headers: Table[string, string]
    params: Table[string, string]

  MockResponse = object
    status: int
    body: string
    headers: Table[string, string]

  ApiHandler = proc(req: MockRequest): MockResponse

proc mockGet(path: string, params: Table[string, string] = initTable[string, string]()): MockRequest =
  MockRequest(method_: "GET", path: path, params: params,
    headers: initTable[string, string](), body: "")

proc mockPost(path, body: string): MockRequest =
  MockRequest(method_: "POST", path: path, body: body,
    headers: {"content-type": "application/json"}.toTable())

proc jsonBody(resp: MockResponse): JsonNode =
  try: parseJson(resp.body)
  except: newJNull()

# ============================
# Demo: testing real functions
# ============================

# Functions under test
proc add(a, b: int): int = a + b
proc divide(a, b: float): float =
  if b == 0: raise newException(DivByZeroDefect, "division by zero")
  a / b

proc fibonacci(n: int): int =
  if n <= 1: return n
  var a, b = 0, 1
  for _ in 2..n:
    (a, b) = (b, a + b)
  return b

proc isPalindrome(s: string): bool =
  let lower = s.toLowerAscii()
  lower == lower.reversed()

proc parseUserJson(json_: string): Option[tuple[name, email: string]] =
  try:
    let j = parseJson(json_)
    let name = j.getOrDefault("name", %"").getStr()
    let email = j.getOrDefault("email", %"").getStr()
    if name.len == 0 or "@" notin email:
      return none(tuple[name, email: string])
    return some((name, email))
  except:
    return none(tuple[name, email: string])

proc demo() =
  echo "=== Testing Framework Demo ==="

  describe("Math functions") do:
    test("Math functions", "add: basic") do:
      assertEqual(add(2, 3), 5)
      assertEqual(add(-1, 1), 0)
      assertEqual(add(0, 0), 0)

    test("Math functions", "divide: basic") do:
      assertNear(divide(10.0, 3.0), 3.333, 0.001)

    test("Math functions", "divide: zero") do:
      assertRaises("division by zero"): discard divide(1.0, 0.0)

    test("Math functions", "fibonacci") do:
      assertEqual(fibonacci(0), 0)
      assertEqual(fibonacci(1), 1)
      assertEqual(fibonacci(10), 55)
      assertEqual(fibonacci(20), 6765)

  describe("String functions") do:
    test("String functions", "palindrome: racecar") do:
      assertTrue(isPalindrome("racecar"))
    test("String functions", "palindrome: level") do:
      assertTrue(isPalindrome("Level"))
    test("String functions", "not palindrome") do:
      assertFalse(isPalindrome("hello"))

  # Table-driven test
  tableTest("Fibonacci table-driven", @[
    TestTableRow[int, int](input: 0, expected: 0, name: "fib(0)=0"),
    TestTableRow[int, int](input: 5, expected: 5, name: "fib(5)=5"),
    TestTableRow[int, int](input: 10, expected: 55, name: "fib(10)=55"),
  ], fibonacci)

  describe("JSON parsing") do:
    test("JSON parsing", "valid user") do:
      let result = assertSome(parseUserJson("""{"name":"Alice","email":"alice@example.com"}"""))
      assertEqual(result.name, "Alice")
      assertEqual(result.email, "alice@example.com")

    test("JSON parsing", "missing email") do:
      assertNone(parseUserJson("""{"name":"Alice"}"""))

    test("JSON parsing", "invalid JSON") do:
      assertNone(parseUserJson("not json"))

  describe("Mock objects") do:
    test("Mock objects", "email sender mock") do:
      var mockSender = MockEmailSender(shouldFail: false)
      assertTrue(mockSendEmail(mockSender, "user@example.com", "Hello", "Body"))
      assertEqual(mockSender.sent.len, 1)
      assertEqual(mockSender.sent[0].to, "user@example.com")

    test("Mock objects", "email sender failure") do:
      var mockSender = MockEmailSender(shouldFail: true)
      assertFalse(mockSendEmail(mockSender, "user@example.com", "Hello", "Body"))
      assertEqual(mockSender.sent.len, 0)

    test("Mock objects", "mock function tracking") do:
      var mock = newMock("getUserById", returns = @["user_data_1", "user_data_2"])
      let r1 = callMock(mock, @["1"])
      let r2 = callMock(mock, @["2"])
      assertEqual(callCount(mock), 2)
      assertTrue(wasCalledWith(mock, @["1"]))
      assertEqual(r1, "user_data_1")
      assertEqual(r2, "user_data_2")

  printSummary()

demo()
```

---

## 📝 สรุป Part 59

| Steps | หัวข้อ |
|-------|--------|
| 856 | Test framework: assertions, test suites, table-driven tests |
| 857-870 | Mock objects, HTTP API testing helpers, integration patterns |

---

**← [Part 58: Security](part_58_security.md) | [Part 60: Database ORM →](part_60_orm.md)**
