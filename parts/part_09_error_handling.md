# Part 09: Error Handling - try/except/finally
## Steps 101-115: การจัดการ Errors อย่างมืออาชีพ

---

## 🎯 เป้าหมายของ Part นี้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
- ใช้ try/except/finally อย่างถูกต้อง
- สร้าง Custom Exceptions
- ใช้ Result types (Ok/Err pattern)
- จัดการ Panics
- เขียน Error handling สำหรับ backend APIs

---

## Step 101: พื้นฐาน try/except

```nim
import std/strutils

# Basic try/except
try:
  let num = parseInt("not a number")
  echo num
except ValueError as e:
  echo "Parse error: " & e.msg

# Multiple except clauses
proc parseAndDivide(a, b: string): int =
  try:
    let x = parseInt(a)
    let y = parseInt(b)
    
    if y == 0:
      raise newException(DivByZeroDefect, "Cannot divide by zero")
    
    return x div y
  
  except ValueError as e:
    echo "Invalid number: " & e.msg
    return 0
  
  except DivByZeroDefect as e:
    echo "Division error: " & e.msg
    return 0
  
  except CatchableError as e:
    echo "Unknown error: " & e.msg
    return -1

echo parseAndDivide("10", "2")    # 5
echo parseAndDivide("abc", "2")   # 0, with error msg
echo parseAndDivide("10", "0")    # 0, with error msg

# finally block
proc readFileSafely(path: string): string =
  var f: File
  try:
    f = open(path)
    return f.readAll()
  except IOError as e:
    echo "File error: " & e.msg
    return ""
  finally:
    if f != nil:
      close(f)  # always runs

# Catch all exceptions
try:
  raise newException(Exception, "test error")
except:
  let e = getCurrentException()
  echo "Caught: " & e.msg
  echo "Type: " & getCurrentExceptionMsg()
```

---

## Step 102: Exception Hierarchy

```nim
# Nim exception hierarchy:
# Exception (base)
# ├── Defect (programming errors, not catchable in safe mode)
# │   ├── AccessViolationDefect
# │   ├── DivByZeroDefect
# │   ├── IndexDefect
# │   ├── NilAccessDefect
# │   ├── StackOverflowDefect
# │   └── AssertionDefect (from assert)
# └── CatchableError (expected errors, should be caught)
#     ├── IOError
#     ├── OSError
#     ├── ValueError
#     ├── KeyError
#     ├── OverflowDefect
#     ├── RangeDefect
#     └── ... your custom errors

# Only CatchableError should be caught in normal code
try:
  let x = 5 div 0
except DivByZeroDefect as e:
  echo "Caught defect (unusual): " & e.msg

# Raising exceptions
proc validateAge(age: int) =
  if age < 0:
    raise newException(ValueError, "Age cannot be negative: " & $age)
  if age > 150:
    raise newException(ValueError, "Age too large: " & $age)

try:
  validateAge(-5)
except ValueError as e:
  echo "Validation failed: " & e.msg

# Re-raising exceptions
proc processData(data: string) =
  try:
    let num = parseInt(data)
    echo "Processing: " & $num
  except ValueError:
    echo "Caught and re-raising..."
    raise  # re-raise same exception
```

---

## Step 103: Custom Exceptions

```nim
import std/strformat

# Define custom exceptions
type
  AppError = object of CatchableError
    code: int

  DatabaseError = object of AppError
    query: string

  ValidationError = object of AppError
    field: string
    constraint: string

  AuthError = object of AppError
    userId: string

  NotFoundError = object of AppError
    resourceType: string
    resourceId: string

# Create exception constructors
proc newDatabaseError(msg, query: string, code: int = 500): ref DatabaseError =
  result = newException(DatabaseError, msg)
  result.code = code
  result.query = query

proc newValidationError(field, constraint, msg: string): ref ValidationError =
  result = newException(ValidationError, msg)
  result.code = 400
  result.field = field
  result.constraint = constraint

proc newNotFoundError(resourceType, resourceId: string): ref NotFoundError =
  result = newException(NotFoundError, fmt"{resourceType} not found: {resourceId}")
  result.code = 404
  result.resourceType = resourceType
  result.resourceId = resourceId

proc newAuthError(userId, msg: string): ref AuthError =
  result = newException(AuthError, msg)
  result.code = 401
  result.userId = userId

# Usage
proc findUser(id: int): string =
  if id <= 0:
    raise newValidationError("id", "must be positive", "Invalid user ID")
  
  if id > 1000:
    raise newNotFoundError("User", $id)
  
  return "User #" & $id

proc getProfile(userId: int): string =
  try:
    let user = findUser(userId)
    return "Profile: " & user
  
  except ValidationError as e:
    echo fmt"Validation error on field '{e.field}': {e.msg} (code: {e.code})"
    return ""
  
  except NotFoundError as e:
    echo fmt"Not found - {e.resourceType}:{e.resourceId} (code: {e.code})"
    return ""
  
  except AppError as e:
    echo fmt"App error (code: {e.code}): {e.msg}"
    return ""

echo getProfile(5)     # Profile: User #5
echo getProfile(-1)    # Validation error
echo getProfile(9999)  # Not found
```

---

## Step 104: Result Type Pattern

```nim
import std/options, std/strformat

# Result type for functional error handling
type
  ResultKind = enum
    Ok, Err

  Result[T, E] = object
    case kind: ResultKind
    of Ok: value: T
    of Err: error: E

proc ok[T, E](val: T): Result[T, E] =
  Result[T, E](kind: Ok, value: val)

proc err[T, E](e: E): Result[T, E] =
  Result[T, E](kind: Err, error: e)

proc isOk[T, E](r: Result[T, E]): bool = r.kind == Ok
proc isErr[T, E](r: Result[T, E]): bool = r.kind == Err

proc get[T, E](r: Result[T, E]): T =
  if r.kind != Ok:
    raise newException(ValueError, "Called get() on Err result")
  r.value

proc getErr[T, E](r: Result[T, E]): E =
  if r.kind != Err:
    raise newException(ValueError, "Called getErr() on Ok result")
  r.error

proc getOrDefault[T, E](r: Result[T, E], default: T): T =
  if r.isOk: r.value else: default

# Usage
type AppError2 = object
  code: int
  message: string

proc parseAge(s: string): Result[int, AppError2] =
  try:
    let age = parseInt(s)
    if age < 0 or age > 150:
      return err[int, AppError2](AppError2(code: 400, message: "Age out of range"))
    return ok[int, AppError2](age)
  except ValueError:
    return err[int, AppError2](AppError2(code: 400, message: "Invalid number: " & s))

# Chain results
proc createUser(name, ageStr: string): Result[string, AppError2] =
  let ageResult = parseAge(ageStr)
  if ageResult.isErr:
    return err[string, AppError2](ageResult.getErr())
  
  let age = ageResult.get()
  if name.len < 2:
    return err[string, AppError2](AppError2(code: 400, message: "Name too short"))
  
  return ok[string, AppError2](fmt"User({name}, {age})")

# Usage
let r1 = createUser("Alice", "25")
let r2 = createUser("Alice", "abc")
let r3 = createUser("A", "25")

echo r1.get()                    # User(Alice, 25)
echo r2.getErr().message         # Invalid number: abc
echo r3.getErr().message         # Name too short

# Using getOrDefault
echo createUser("Bob", "30").getOrDefault("failed")  # User(Bob, 30)
echo createUser("B", "30").getOrDefault("failed")    # failed
```

---

## Step 105: Error Handling สำหรับ HTTP APIs

```nim
import std/strformat, std/json, std/tables

type
  HttpStatusCode = 200..599

  HttpError = object of CatchableError
    statusCode: int
    errorCode: string
    details: JsonNode

  BadRequestError   = object of HttpError
  UnauthorizedError = object of HttpError
  ForbiddenError    = object of HttpError
  NotFoundError2    = object of HttpError
  ConflictError     = object of HttpError
  InternalServerError = object of HttpError

proc newHttpError[T: HttpError](
  msg, errorCode: string,
  statusCode: int,
  details: JsonNode = newJNull()
): ref T =
  result = newException(T, msg)
  result.statusCode = statusCode
  result.errorCode = errorCode
  result.details = details

proc badRequest(msg: string, field: string = ""): ref BadRequestError =
  var details = newJNull()
  if field.len > 0:
    details = %*{"field": field}
  result = newHttpError[BadRequestError](msg, "BAD_REQUEST", 400, details)

proc unauthorized(msg: string = "Authentication required"): ref UnauthorizedError =
  result = newHttpError[UnauthorizedError](msg, "UNAUTHORIZED", 401)

proc notFound(resource: string): ref NotFoundError2 =
  result = newHttpError[NotFoundError2](resource & " not found", "NOT_FOUND", 404)

proc internalError(msg: string = "Internal server error"): ref InternalServerError =
  result = newHttpError[InternalServerError](msg, "INTERNAL_ERROR", 500)

# Error response formatter
proc toJsonResponse(e: ref HttpError): JsonNode =
  result = %*{
    "success": false,
    "error": {
      "code": e.errorCode,
      "message": e.msg,
      "statusCode": e.statusCode,
    }
  }
  
  if e.details.kind != JNull:
    result["error"]["details"] = e.details

# Middleware-style error handler
type
  Handler = proc(): JsonNode {.raises: [Exception].}
  
  ApiResponse = object
    status: int
    body: JsonNode

proc handleRequest(handler: Handler): ApiResponse =
  try:
    let result = handler()
    return ApiResponse(status: 200, body: %*{"success": true, "data": result})
  
  except BadRequestError as e:
    return ApiResponse(status: 400, body: toJsonResponse(e))
  
  except UnauthorizedError as e:
    return ApiResponse(status: 401, body: toJsonResponse(e))
  
  except ForbiddenError as e:
    return ApiResponse(status: 403, body: toJsonResponse(e))
  
  except NotFoundError2 as e:
    return ApiResponse(status: 404, body: toJsonResponse(e))
  
  except ConflictError as e:
    return ApiResponse(status: 409, body: toJsonResponse(e))
  
  except InternalServerError as e:
    return ApiResponse(status: 500, body: toJsonResponse(e))
  
  except CatchableError as e:
    let wrapper = internalError(e.msg)
    return ApiResponse(status: 500, body: toJsonResponse(wrapper))

# Demo handlers
proc getUserHandler(): JsonNode =
  raise notFound("User")

proc createUserHandler(): JsonNode =
  raise badRequest("Email is required", "email")

let r1 = handleRequest(getUserHandler)
echo fmt"Status: {r1.status}"
echo r1.body.pretty()

let r2 = handleRequest(createUserHandler)
echo fmt"Status: {r2.status}"
echo r2.body.pretty()
```

---

## Step 106-115: Full Error Handling System

```nim
# error_system.nim
# Production-ready error handling for Nim backend

import std/strformat, std/json, std/tables, std/times,
       std/strutils, std/os

# ==============================
# Error Types
# ==============================

type
  ErrorSeverity = enum
    Low, Medium, High, Critical

  ErrorCategory = enum
    Validation, Authorization, NotFound, Conflict,
    Database, External, Internal, RateLimit

  StructuredError = object
    id: string
    timestamp: DateTime
    category: ErrorCategory
    severity: ErrorSeverity
    code: string
    message: string
    details: JsonNode
    stackTrace: string
    context: Table[string, string]

  ErrorHandler = object
    errors: seq[StructuredError]
    onError: proc(e: StructuredError)

# ==============================
# Error constructors
# ==============================

var errorHandler = ErrorHandler(
  errors: @[],
  onError: nil
)

proc generateErrorId(): string =
  result = "ERR_" & now().format("yyyyMMddHHmmss") & "_" &
           (rand(9999).intToStr().align(4, '0'))

proc createError(
  category: ErrorCategory,
  code, message: string,
  severity: ErrorSeverity = Medium,
  details: JsonNode = newJNull(),
  context: Table[string, string] = initTable[string, string]()
): StructuredError =
  StructuredError(
    id: generateErrorId(),
    timestamp: now(),
    category: category,
    severity: severity,
    code: code,
    message: message,
    details: details,
    stackTrace: "",
    context: context
  )

proc toJson(e: StructuredError): JsonNode =
  result = %*{
    "id": e.id,
    "timestamp": e.timestamp.format("yyyy-MM-dd'T'HH:mm:ss"),
    "category": $e.category,
    "severity": $e.severity,
    "code": e.code,
    "message": e.message,
  }
  
  if e.details.kind != JNull:
    result["details"] = e.details
  
  if e.context.len > 0:
    var ctx = newJObject()
    for k, v in e.context:
      ctx[k] = %v
    result["context"] = ctx

# ==============================
# Validation Framework
# ==============================

type
  FieldError = object
    field: string
    message: string
    rule: string

  ValidationResult2 = object
    isValid: bool
    errors: seq[FieldError]

proc newValidationResult(): ValidationResult2 =
  ValidationResult2(isValid: true, errors: @[])

proc addError(vr: var ValidationResult2, field, msg, rule: string) =
  vr.errors.add(FieldError(field: field, message: msg, rule: rule))
  vr.isValid = false

proc check(vr: var ValidationResult2, field, value: string, rule: proc(v: string): bool, msg: string) =
  if not rule(value):
    vr.addError(field, msg, "custom")

# Validators
proc required(field, value: string, vr: var ValidationResult2) =
  if value.strip().len == 0:
    vr.addError(field, field & " is required", "required")

proc minLength(field, value: string, min: int, vr: var ValidationResult2) =
  if value.len < min:
    vr.addError(field, fmt"{field} must be at least {min} characters", "minLength")

proc maxLength(field, value: string, max: int, vr: var ValidationResult2) =
  if value.len > max:
    vr.addError(field, fmt"{field} must be at most {max} characters", "maxLength")

proc isEmail(field, value: string, vr: var ValidationResult2) =
  import std/re
  if not value.match(re"^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$"):
    vr.addError(field, fmt"{field} must be a valid email", "email")

proc isPositive(field: string, value: int, vr: var ValidationResult2) =
  if value <= 0:
    vr.addError(field, fmt"{field} must be positive", "positive")

# ==============================
# Repository pattern with errors
# ==============================

type
  User2 = object
    id: int
    name: string
    email: string
    passwordHash: string
    createdAt: DateTime

  UserRepository = object
    users: Table[int, User2]
    nextId: int

proc newUserRepository(): UserRepository =
  UserRepository(users: initTable[int, User2](), nextId: 1)

proc findById(repo: UserRepository, id: int): Option[User2] =
  if id in repo.users:
    return some(repo.users[id])
  return none(User2)

proc findByEmail(repo: UserRepository, email: string): Option[User2] =
  for user in repo.users.values:
    if user.email.toLower() == email.toLower():
      return some(user)
  return none(User2)

type
  CreateUserInput = object
    name: string
    email: string
    password: string

proc validateCreateUserInput(input: CreateUserInput): ValidationResult2 =
  var vr = newValidationResult()
  
  required("name", input.name, vr)
  minLength("name", input.name, 2, vr)
  maxLength("name", input.name, 100, vr)
  
  required("email", input.email, vr)
  isEmail("email", input.email, vr)
  
  required("password", input.password, vr)
  minLength("password", input.password, 8, vr)
  
  return vr

proc createUser2(repo: var UserRepository, input: CreateUserInput): User2 =
  # Validate
  let validation = validateCreateUserInput(input)
  if not validation.isValid:
    let errDetails = %* {"errors": validation.errors.mapIt(%* {"field": it.field, "message": it.message})}
    let err = createError(Validation, "VALIDATION_ERROR", "Input validation failed",
                          details = errDetails)
    raise newException(CatchableError, err.toJson().pretty())
  
  # Check duplicate email
  if repo.findByEmail(input.email).isSome:
    let err = createError(Conflict, "EMAIL_EXISTS",
                          fmt"Email already registered: {input.email}")
    raise newException(CatchableError, err.toJson().pretty())
  
  # Create user
  let id = repo.nextId
  inc repo.nextId
  
  let user = User2(
    id: id,
    name: input.name,
    email: input.email,
    passwordHash: "hash_" & input.password,  # simplified
    createdAt: now()
  )
  
  repo.users[id] = user
  return user

# ==============================
# Demo
# ==============================

proc main() =
  var repo = newUserRepository()
  
  echo "=== Error Handling System Demo ==="
  
  # Test 1: Valid user creation
  echo "\n1. Creating valid user..."
  try:
    let user = repo.createUser2(CreateUserInput(
      name: "Alice Smith",
      email: "alice@example.com",
      password: "SecurePass123"
    ))
    echo fmt"   ✅ Created: {user.name} (ID: {user.id})"
  except CatchableError as e:
    echo "   ❌ Error: " & e.msg
  
  # Test 2: Validation error
  echo "\n2. Testing validation error..."
  try:
    let user = repo.createUser2(CreateUserInput(
      name: "A",           # too short
      email: "not-an-email", # invalid
      password: "short"    # too short
    ))
    echo "   ✅ Created: " & user.name
  except CatchableError as e:
    echo "   ❌ Validation Error (expected)"
    # Parse and display nicely
    try:
      let errJson = parseJson(e.msg)
      echo "   Category: " & errJson["category"].getStr()
      echo "   Code: " & errJson["code"].getStr()
      let errors = errJson["details"]["errors"]
      for err in errors:
        echo fmt"   - {err[\"field\"].getStr()}: {err[\"message\"].getStr()}"
    except:
      echo "   " & e.msg
  
  # Test 3: Duplicate email
  echo "\n3. Testing duplicate email..."
  try:
    let user = repo.createUser2(CreateUserInput(
      name: "Alice Clone",
      email: "alice@example.com",  # duplicate!
      password: "AnotherPass123"
    ))
    echo "   ✅ Created: " & user.name
  except CatchableError as e:
    echo "   ❌ Conflict Error (expected)"
    try:
      let errJson = parseJson(e.msg)
      echo "   Code: " & errJson["code"].getStr()
      echo "   Message: " & errJson["message"].getStr()
    except:
      echo "   " & e.msg

main()
```

---

## 📝 สรุป Part 09

| Steps | หัวข้อ |
|-------|--------|
| 101 | try/except/finally basics |
| 102 | Exception hierarchy |
| 103 | Custom exceptions |
| 104 | Result type pattern |
| 105 | HTTP API error handling |
| 106-115 | Full Error Handling System |

---

**← [Part 08: File I/O](part_08_file_io.md) | [Part 10: Modules →](part_10_modules.md)**
