# Part 03: Control Flow - if/else, case/of, when
## Steps 21-30: การควบคุมการทำงานของโปรแกรม

---

## 🎯 เป้าหมายของ Part นี้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
- ใช้ `if/elif/else` ได้ทุกรูปแบบ
- ใช้ `case/of` สำหรับ pattern matching
- เข้าใจ `when` (compile-time conditionals)
- ใช้ conditional expressions
- ใช้ guard clauses และ early returns
- สร้าง complex business logic

---

## Step 21: if/elif/else พื้นฐาน

```nim
# if พื้นฐาน
var temperature = 35

if temperature > 30:
  echo "อากาศร้อนมาก"
elif temperature > 25:
  echo "อากาศอบอุ่น"
elif temperature > 15:
  echo "อากาศเย็นสบาย"
elif temperature > 5:
  echo "อากาศหนาว"
else:
  echo "อากาศหนาวมาก"

# if เป็น expression (ได้ค่ากลับ)
let weatherDesc = if temperature > 30: "ร้อน"
                  elif temperature > 20: "อบอุ่น"
                  else: "เย็น"
echo weatherDesc  # ร้อน

# Nested if
var age = 25
var hasLicense = true

if age >= 18:
  if hasLicense:
    echo "ขับรถได้"
  else:
    echo "อายุครบแต่ไม่มีใบขับขี่"
else:
  echo "อายุไม่ถึง 18 ปี"

# Single line if
if age > 0: echo "อายุถูกต้อง"

# Boolean checks ที่ดี
var name = ""
var items: seq[int] = @[]

if name.len == 0:    echo "ชื่อว่าง"
if name.isEmptyOrWhitespace(): echo "ชื่อว่างหรือมีแต่ space"
if items.len == 0:   echo "ไม่มีรายการ"
```

### Comparison Operators

```nim
var x = 10
var y = 20

# Comparisons
echo x == y    # false (equal)
echo x != y    # true  (not equal)
echo x < y     # true  (less than)
echo x > y     # false (greater than)
echo x <= y    # true  (less or equal)
echo x >= y    # false (greater or equal)

# Chained comparisons (Nim supports this!)
var z = 15
echo x < z and z < y    # Standard way: true
echo x < z < y          # Nim chained: true (syntactic sugar)

# String comparison
var str1 = "apple"
var str2 = "banana"
echo str1 < str2    # true (lexicographic)
echo str1 == "apple" # true

# Case-insensitive comparison
import std/strutils
echo str1.cmpIgnoreCase("APPLE") == 0  # true
```

---

## Step 22: case/of Statement

```nim
# case/of พื้นฐาน
var day = "Monday"

case day
of "Monday", "Tuesday", "Wednesday", "Thursday", "Friday":
  echo "วันทำงาน"
of "Saturday", "Sunday":
  echo "วันหยุด"
else:
  echo "ไม่รู้จักวันนี้"

# case กับ enum
type
  Season = enum
    Spring, Summer, Autumn, Winter

var currentSeason = Summer

case currentSeason
of Spring:
  echo "🌸 ฤดูใบไม้ผลิ"
of Summer:
  echo "☀️  ฤดูร้อน"
of Autumn:
  echo "🍂 ฤดูใบไม้ร่วง"
of Winter:
  echo "❄️  ฤดูหนาว"

# case กับ integer
var httpCode = 404

case httpCode
of 200:
  echo "OK"
of 201:
  echo "Created"
of 400:
  echo "Bad Request"
of 401:
  echo "Unauthorized"
of 403:
  echo "Forbidden"
of 404:
  echo "Not Found"
of 500..599:   # Range!
  echo "Server Error"
else:
  echo "Unknown Status: " & $httpCode

# case เป็น expression
let statusMessage = case httpCode
  of 200: "Success"
  of 201: "Resource created"
  of 400: "Invalid request"
  of 404: "Resource not found"
  else: "Unknown error"

echo statusMessage

# case กับ char
var grade = 'B'

case grade
of 'A':
  echo "ยอดเยี่ยม (80-100)"
of 'B':
  echo "ดี (70-79)"
of 'C':
  echo "พอใช้ (60-69)"
of 'D':
  echo "ผ่าน (50-59)"
of 'F':
  echo "สอบตก (< 50)"
else:
  echo "เกรดไม่ถูกต้อง"
```

---

## Step 23: case ขั้นสูง - Pattern Matching

```nim
import std/strutils

# case กับ string patterns
proc classifyInput(input: string): string =
  let trimmed = input.strip().toLower()
  case trimmed
  of "yes", "y", "true", "1":
    return "Positive"
  of "no", "n", "false", "0":
    return "Negative"
  else:
    return "Unknown"

echo classifyInput("YES")    # Positive
echo classifyInput("n")      # Negative
echo classifyInput("maybe")  # Unknown

# Nested case
type
  UserRole = enum
    Guest, Member, Admin

  Action = enum
    Read, Write, Delete

proc canPerform(role: UserRole, action: Action): bool =
  case role
  of Guest:
    case action
    of Read: return true
    of Write, Delete: return false
  of Member:
    case action
    of Read, Write: return true
    of Delete: return false
  of Admin:
    return true  # Admin สามารถทำได้ทุกอย่าง

# Test
echo canPerform(Guest, Read)     # true
echo canPerform(Guest, Write)    # false
echo canPerform(Member, Delete)  # false
echo canPerform(Admin, Delete)   # true

# case กับ tuples (Nim 2.0+)
proc classify(x, y: int): string =
  case (x > 0, y > 0)
  of (true, true):   return "Quadrant I (++)"
  of (false, true):  return "Quadrant II (-+)"
  of (false, false): return "Quadrant III (--)"
  of (true, false):  return "Quadrant IV (+-)"

echo classify(1, 1)    # Quadrant I (++)
echo classify(-1, 1)   # Quadrant II (-+)
echo classify(-1, -1)  # Quadrant III (--)
echo classify(1, -1)   # Quadrant IV (+-)
```

---

## Step 24: when Statement (Compile-time Conditionals)

`when` ทำงานเหมือน `if` แต่ evaluate ตอน compile time

```nim
# when สำหรับ OS detection
when defined(windows):
  echo "Running on Windows"
elif defined(macosx):
  echo "Running on macOS"
elif defined(linux):
  echo "Running on Linux"
else:
  echo "Unknown OS"

# when สำหรับ CPU architecture
when defined(amd64):
  echo "64-bit x86"
elif defined(i386):
  echo "32-bit x86"
elif defined(arm64):
  echo "ARM 64-bit"

# when สำหรับ Nim version
when NimMajor >= 2:
  echo "Nim 2.x or later"
else:
  echo "Nim 1.x"

# when สำหรับ custom defines
# nim c -d:myDebug myapp.nim
when defined(myDebug):
  echo "Debug mode is ON"
  proc debugLog(msg: string) = echo "[DEBUG] " & msg
else:
  proc debugLog(msg: string) = discard  # no-op in production

# when สำหรับ conditional compilation
when sizeof(int) == 8:
  echo "64-bit platform"
  type PlatformInt = int64
else:
  echo "32-bit platform"
  type PlatformInt = int32

# when กับ system detection
import std/os

when defined(posix):
  import std/posix
  echo "POSIX system"
elif defined(windows):
  import std/winlean
  echo "Windows system"
```

---

## Step 25: Conditional Expressions

```nim
# if-expression
var x = 10
var result = if x > 5: "big" else: "small"
echo result  # big

# ternary-like (Nim ไม่มี ? :)
# แต่ใช้ if-expression แทน
var a = 5
var b = 10
var max = if a > b: a else: b
echo max  # 10

# Nested if-expressions
var score = 85
var grade = if score >= 90: 'A'
            elif score >= 80: 'B'
            elif score >= 70: 'C'
            elif score >= 60: 'D'
            else: 'F'
echo grade  # B

# case expression
type
  Status = enum
    Active, Inactive, Banned

var userStatus = Active

let statusIcon = case userStatus
  of Active: "✅"
  of Inactive: "⚪"
  of Banned: "🚫"

echo statusIcon  # ✅

# Complex expression
proc classify(n: int): string =
  if n < 0: "negative"
  elif n == 0: "zero"
  elif n < 10: "small"
  elif n < 100: "medium"
  else: "large"

echo classify(-5)   # negative
echo classify(0)    # zero
echo classify(7)    # small
echo classify(50)   # medium
echo classify(500)  # large
```

---

## Step 26: Guard Clauses และ Early Returns

```nim
import std/options, std/strutils

# ปัญหา: Deeply nested code (Pyramid of Doom)
proc processUserBad(name: string, age: int, email: string): string =
  if name.len > 0:
    if age >= 18:
      if email.contains('@'):
        return "Valid user: " & name
      else:
        return "Invalid email"
    else:
      return "Too young"
  else:
    return "Name required"

# ดีกว่า: Guard Clauses (Fail Fast)
proc processUser(name: string, age: int, email: string): string =
  # Guard clauses - ตรวจสอบ errors ก่อน
  if name.len == 0:
    return "Name required"
  
  if age < 18:
    return "Too young"
  
  if not email.contains('@'):
    return "Invalid email"
  
  # Happy path - ถึงตรงนี้แปลว่าทุกอย่างถูกต้อง
  return "Valid user: " & name

echo processUser("", 25, "test@test.com")     # Name required
echo processUser("Alice", 16, "test@test.com") # Too young
echo processUser("Alice", 25, "notanemail")    # Invalid email
echo processUser("Alice", 25, "alice@test.com") # Valid user: Alice

# Guard with Option type
proc findUser(id: int): Option[string] =
  if id <= 0:
    return none(string)
  
  # Simulated database lookup
  if id == 1:
    return some("Alice")
  elif id == 2:
    return some("Bob")
  
  return none(string)

proc getUserName(id: int): string =
  let user = findUser(id)
  
  if user.isNone():
    return "User not found"
  
  return user.get()

echo getUserName(1)   # Alice
echo getUserName(99)  # User not found
```

---

## Step 27: Complex Conditions

```nim
import std/times, std/options

# Complex boolean conditions
proc isValidPassword(password: string): bool =
  let minLen = 8
  var hasUpper = false
  var hasLower = false
  var hasDigit = false
  var hasSpecial = false
  
  if password.len < minLen:
    return false
  
  for c in password:
    if c.isUpper(): hasUpper = true
    elif c.isLower(): hasLower = true
    elif c.isDigit(): hasDigit = true
    elif c in "!@#$%^&*()_+-=[]{}|": hasSpecial = true
  
  return hasUpper and hasLower and hasDigit and hasSpecial

# Test passwords
let passwords = [
  "short",
  "alllowercase1!",
  "ALLUPPERCASE1!",
  "NoSpecialChars1",
  "Valid@Pass123"
]

for pwd in passwords:
  let valid = isValidPassword(pwd)
  echo fmt"{pwd:20} : {if valid: \"✅ Valid\" else: \"❌ Invalid\"}"

# Business logic with multiple conditions
type
  OrderStatus = enum
    Pending, Processing, Shipped, Delivered, Cancelled, Refunded

  Order = object
    id: int
    status: OrderStatus
    totalAmount: float
    isPaid: bool
    createdAt: DateTime
    estimatedDelivery: Option[DateTime]

proc canCancel(order: Order): bool =
  # ยกเลิกได้ถ้า: status อยู่ใน Pending หรือ Processing
  # และ ยังไม่จ่ายเงิน หรือ จ่ายแล้วแต่สินค้ายังไม่ส่ง
  if order.status == Cancelled:
    return false  # ยกเลิกแล้ว ยกเลิกอีกไม่ได้
  
  if order.status == Delivered:
    return false  # ส่งแล้ว ยกเลิกไม่ได้
  
  if order.status == Refunded:
    return false  # คืนเงินแล้ว
  
  # Pending หรือ Processing ยกเลิกได้
  if order.status in {Pending, Processing}:
    return true
  
  # Shipped แต่ยังไม่ถึงมือ - ยกเลิกได้แต่มีเงื่อนไข
  if order.status == Shipped:
    return false  # ส่งแล้ว ต้องรอรับแล้วค่อย Return
  
  return false

proc canRefund(order: Order): bool =
  # Refund ได้ถ้า: จ่ายแล้ว และ ส่งแล้วหรือรับแล้ว
  if not order.isPaid:
    return false
  
  if order.status == Refunded:
    return false  # refund แล้ว
  
  # ต้องส่งหรือรับแล้วถึงจะ refund ได้
  return order.status in {Shipped, Delivered}
```

---

## Step 28: Error Handling with Conditionals

```nim
import std/strutils, std/options

# Result type pattern (ไม่ใช้ exception)
type
  ResultKind = enum
    Ok, Err

  ParseResult[T] = object
    case kind: ResultKind
    of Ok:
      value: T
    of Err:
      error: string

proc ok[T](value: T): ParseResult[T] =
  ParseResult[T](kind: Ok, value: value)

proc err[T](msg: string): ParseResult[T] =
  ParseResult[T](kind: Err, error: msg)

# Parse age safely
proc parseAge(s: string): ParseResult[int] =
  let trimmed = s.strip()
  
  if trimmed.len == 0:
    return err[int]("Age cannot be empty")
  
  var age: int
  try:
    age = parseInt(trimmed)
  except ValueError:
    return err[int]("'" & trimmed & "' is not a valid number")
  
  if age < 0:
    return err[int]("Age cannot be negative")
  
  if age > 150:
    return err[int]("Age seems unrealistic")
  
  return ok(age)

# ใช้งาน
let inputs = ["25", "abc", "-5", "200", "", "30"]

for input in inputs:
  let result = parseAge(input)
  case result.kind
  of Ok:
    echo fmt"✅ Age: {result.value}"
  of Err:
    echo fmt"❌ Error: {result.error}"
```

---

## Step 29: Short-Circuit Evaluation

```nim
import std/strutils

# Short-circuit ใน and
proc validateName(name: string): bool =
  echo fmt"  Checking name: '{name}'"
  return name.len >= 2 and name.len <= 50

proc validateEmail(email: string): bool =
  echo fmt"  Checking email: '{email}'"
  return email.contains('@') and email.contains('.')

proc validateAge(age: int): bool =
  echo fmt"  Checking age: {age}"
  return age >= 18 and age <= 100

proc validateUser(name: string, email: string, age: int): bool =
  echo "Validating user..."
  
  # Short-circuit: ถ้า name fail, email และ age จะไม่ถูก validate
  return validateName(name) and 
         validateEmail(email) and 
         validateAge(age)

echo "Test 1: Invalid name"
echo validateUser("A", "test@test.com", 25)
# จะแสดง "Checking name" แต่ไม่แสดง "Checking email" และ "Checking age"

echo "\nTest 2: All valid"
echo validateUser("Alice", "alice@test.com", 25)
# จะ validate ทั้งหมด

# Practical use: nil/none checks with short-circuit
var data: seq[int] = @[]

# ไม่ crash แม้ data ว่าง
if data.len > 0 and data[0] > 0:
  echo "First element is positive"
else:
  echo "No data or first element is not positive"

# Lazy evaluation with procedures
proc isExpensive(): bool =
  # Imagine this is slow
  echo "Running expensive check..."
  return true

proc isCheap(): bool =
  echo "Running cheap check..."
  return false

# isCheap ถูกเรียกก่อน, ถ้า false -> isExpensive จะไม่ถูกเรียก
if isCheap() and isExpensive():
  echo "both true"
else:
  echo "at least one false"
```

---

## Step 30: Real-World Application - Request Validator

มาสร้าง HTTP Request Validator ที่ใช้งานได้จริง:

```nim
# request_validator.nim
# Validates HTTP API requests

import std/strutils, std/options, std/tables, std/sequtils

# ==============================
# Types
# ==============================

type
  ValidationType = enum
    Required, MinLength, MaxLength, IsEmail, IsNumeric,
    MinValue, MaxValue, IsIn, Pattern

  ValidationRule = object
    kind: ValidationType
    message: string
    case kind
    of MinLength, MaxLength:
      length: int
    of MinValue, MaxValue:
      numValue: float
    of IsIn:
      allowedValues: seq[string]
    of Pattern:
      pattern: string
    else:
      discard

  FieldValidation = object
    fieldName: string
    rules: seq[ValidationRule]

  ValidationError = object
    field: string
    message: string

  ValidationResult = object
    isValid: bool
    errors: seq[ValidationError]

# ==============================
# Rule Builders
# ==============================

proc required(msg = "This field is required"): ValidationRule =
  ValidationRule(kind: Required, message: msg)

proc minLength(n: int, msg = ""): ValidationRule =
  ValidationRule(
    kind: MinLength,
    length: n,
    message: if msg.len > 0: msg else: fmt"Minimum {n} characters required"
  )

proc maxLength(n: int, msg = ""): ValidationRule =
  ValidationRule(
    kind: MaxLength,
    length: n,
    message: if msg.len > 0: msg else: fmt"Maximum {n} characters allowed"
  )

proc isEmail(msg = "Invalid email format"): ValidationRule =
  ValidationRule(kind: IsEmail, message: msg)

proc isNumeric(msg = "Must be a number"): ValidationRule =
  ValidationRule(kind: IsNumeric, message: msg)

proc isIn(values: seq[string], msg = ""): ValidationRule =
  ValidationRule(
    kind: IsIn,
    allowedValues: values,
    message: if msg.len > 0: msg else: "Invalid value"
  )

# ==============================
# Validators
# ==============================

proc validateField(value: string, rule: ValidationRule): Option[string] =
  case rule.kind
  of Required:
    if value.strip().len == 0:
      return some(rule.message)
  
  of MinLength:
    if value.len < rule.length:
      return some(rule.message)
  
  of MaxLength:
    if value.len > rule.length:
      return some(rule.message)
  
  of IsEmail:
    let atPos = value.find('@')
    if atPos < 0:
      return some(rule.message)
    let domain = value[atPos+1..^1]
    if not domain.contains('.'):
      return some(rule.message)
  
  of IsNumeric:
    for c in value:
      if not c.isDigit() and c != '.' and c != '-':
        return some(rule.message)
  
  of IsIn:
    if value.toLower() notin rule.allowedValues.mapIt(it.toLower()):
      return some(rule.message)
  
  of MinValue, MaxValue, Pattern:
    discard  # simplified
  
  return none(string)

proc validate(
  data: Table[string, string],
  fields: seq[FieldValidation]
): ValidationResult =
  
  result.isValid = true
  result.errors = @[]
  
  for field in fields:
    let value = data.getOrDefault(field.fieldName, "")
    
    for rule in field.rules:
      let error = validateField(value, rule)
      
      if error.isSome():
        result.isValid = false
        result.errors.add(ValidationError(
          field: field.fieldName,
          message: error.get()
        ))
        break  # หยุดที่ error แรกของ field นี้

# ==============================
# Schema Builder
# ==============================

proc field(name: string, rules: varargs[ValidationRule]): FieldValidation =
  FieldValidation(
    fieldName: name,
    rules: @rules
  )

# ==============================
# Main Demo
# ==============================

proc main() =
  echo "╔══════════════════════════════════════╗"
  echo "║    HTTP Request Validator Demo        ║"
  echo "╚══════════════════════════════════════╝\n"
  
  # Define validation schema
  let registerSchema = @[
    field("username",
      required(),
      minLength(3),
      maxLength(20)
    ),
    field("email",
      required(),
      isEmail()
    ),
    field("password",
      required(),
      minLength(8)
    ),
    field("role",
      required(),
      isIn(@["user", "admin", "moderator"])
    )
  ]
  
  # Test case 1: Valid data
  echo "📋 Test 1: Valid registration data"
  let validData = {
    "username": "alice123",
    "email": "alice@example.com",
    "password": "SecurePass123!",
    "role": "user"
  }.toTable()
  
  let result1 = validate(validData, registerSchema)
  if result1.isValid:
    echo "  ✅ Validation passed!"
  else:
    for err in result1.errors:
      echo fmt"  ❌ {err.field}: {err.message}"
  
  # Test case 2: Invalid data
  echo "\n📋 Test 2: Invalid registration data"
  let invalidData = {
    "username": "a",              # too short
    "email": "not-an-email",      # invalid email
    "password": "123",            # too short
    "role": "superadmin"          # not in allowed values
  }.toTable()
  
  let result2 = validate(invalidData, registerSchema)
  if result2.isValid:
    echo "  ✅ Validation passed!"
  else:
    echo fmt"  Found {result2.errors.len} validation errors:"
    for err in result2.errors:
      echo fmt"  ❌ {err.field}: {err.message}"
  
  # Test case 3: Missing required fields
  echo "\n📋 Test 3: Missing required fields"
  let missingData = {
    "username": "bob"
    # email, password, role missing
  }.toTable()
  
  let result3 = validate(missingData, registerSchema)
  if result3.isValid:
    echo "  ✅ Validation passed!"
  else:
    echo fmt"  Found {result3.errors.len} validation errors:"
    for err in result3.errors:
      echo fmt"  ❌ {err.field}: {err.message}"

main()
```

---

## 📝 สรุป Part 03

| Step | หัวข้อ | สิ่งที่เรียนรู้ |
|------|--------|----------------|
| 21 | if/elif/else | Conditional branching |
| 22 | case/of | Pattern matching |
| 23 | Advanced case | Nested case, tuples |
| 24 | when | Compile-time conditionals |
| 25 | Conditional Expressions | if/case as expressions |
| 26 | Guard Clauses | Early returns, fail-fast |
| 27 | Complex Conditions | Business logic |
| 28 | Error Handling | Result types |
| 29 | Short-circuit | Evaluation order |
| 30 | Real-world App | Request Validator |

---

## 🏋️ แบบฝึกหัด Part 03

### Exercise 1: Grade Calculator

```nim
proc calculateGrade(scores: seq[float]): char =
  # คำนวณคะแนนเฉลี่ยและคืน grade A-F
  # A: >= 80, B: >= 70, C: >= 60, D: >= 50, F: < 50
  discard

proc getGradeDescription(grade: char): string =
  # คืนคำอธิบาย grade
  discard
```

### Exercise 2: Permission System

```nim
type
  Resource = enum
    Users, Products, Orders, Reports
  
  Permission = enum
    Read, Write, Delete, Export

proc hasPermission(role: string, resource: Resource, action: Permission): bool =
  # Define permission matrix
  # Admin: ทุกอย่าง
  # Manager: Read+Write ทุก resource, Delete Orders เท่านั้น
  # Staff: Read+Write Products+Orders
  # Guest: Read Products เท่านั้น
  discard
```

---

**← [Part 02: Variables & Types](part_02_variables_types.md) | [Part 04: Loops →](part_04_loops.md)**
