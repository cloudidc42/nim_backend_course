# Part 02: Variables, Data Types & Constants
## Steps 11-20: ระบบชนิดข้อมูลใน Nim

---

## 🎯 เป้าหมายของ Part นี้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
- ประกาศตัวแปรด้วย `var`, `let`, `const`
- ใช้ชนิดข้อมูลพื้นฐานทั้งหมด
- เข้าใจ Type Inference ของ Nim
- แปลงชนิดข้อมูล (Type Conversion)
- ใช้ Type Aliases และ Distinct Types

---

## Step 11: การประกาศตัวแปร - var, let, const

Nim มี 3 วิธีในการประกาศค่า:

```nim
# var - ตัวแปรที่เปลี่ยนค่าได้ (mutable)
var name: string = "Alice"
var age: int = 25
var score: float = 98.5
var isActive: bool = true

# let - ค่าที่เปลี่ยนแปลงไม่ได้หลังจากกำหนด (immutable, runtime)
let pi: float = 3.14159265358979
let greeting: string = "สวัสดี"
let maxUsers: int = 1000

# const - ค่าคงที่ที่รู้ค่าตอน Compile Time
const MAX_CONNECTIONS: int = 100
const APP_NAME: string = "MyNimApp"
const VERSION: string = "1.0.0"
const PI: float = 3.14159265358979
```

### ความแตกต่างระหว่าง var, let, const

```nim
# var - เปลี่ยนค่าได้
var counter: int = 0
counter = 1          # OK
counter += 1         # OK

# let - เปลี่ยนค่าไม่ได้
let maxScore: int = 100
# maxScore = 200     # Error! cannot assign to let variable

# const - เปลี่ยนค่าไม่ได้, ค่าต้องรู้ตอน compile
const defaultPort = 8080
# const someValue = readLine(stdin)  # Error! ไม่รู้ค่าตอน compile

# เปรียบเทียบ let vs const
let runtimeValue = someFunction()    # OK - ค่าจากฟังก์ชัน
# const runtimeValue = someFunction() # Error - const ต้องรู้ตอน compile
```

### Type Inference

Nim สามารถ infer type ได้อัตโนมัติ:

```nim
var x = 42          # int (inferred)
var y = 3.14        # float (inferred)
var z = "hello"     # string (inferred)
var flag = true     # bool (inferred)

let message = "Nim is awesome"  # string (inferred)
const LIMIT = 1000              # int (inferred)

# ตัวอย่างที่ซับซ้อนขึ้น
var numbers = @[1, 2, 3, 4, 5]  # seq[int] (inferred)
var user = (name: "Bob", age: 30)  # tuple (inferred)
```

---

## Step 12: Integer Types (ชนิดจำนวนเต็ม)

```nim
# Signed integers
var a: int8   = 127          # -128 to 127
var b: int16  = 32767        # -32768 to 32767
var c: int32  = 2147483647   # -2,147,483,648 to 2,147,483,647
var d: int64  = 9223372036854775807  # ใหญ่มาก
var e: int    = 100          # platform-dependent (32 or 64 bit)

# Unsigned integers (ไม่มีค่าลบ)
var ua: uint8  = 255         # 0 to 255
var ub: uint16 = 65535       # 0 to 65535
var uc: uint32 = 4294967295  # 0 to 4,294,967,295
var ud: uint64 = 18446744073709551615  # ใหญ่มาก
var ue: uint   = 100         # platform-dependent

# การดำเนินการกับ Integer
var x: int = 10
var y: int = 3

echo x + y    # 13 (บวก)
echo x - y    # 7  (ลบ)
echo x * y    # 30 (คูณ)
echo x div y  # 3  (หารเต็ม) - ใน Nim ใช้ div ไม่ใช่ /
echo x mod y  # 1  (เศษ)
echo x / y    # 3.333... (หารจริง - ได้ float)

# Bitwise operations
echo 0b1010 and 0b1100   # 8 (AND)
echo 0b1010 or  0b1100   # 14 (OR)
echo 0b1010 xor 0b1100   # 6 (XOR)
echo not 0b1010          # -11 (NOT)
echo 1 shl 3             # 8 (shift left)
echo 8 shr 2             # 2 (shift right)

# Integer literals
let decimal   = 1_000_000    # underscore สำหรับอ่านง่าย
let hexValue  = 0xFF         # hexadecimal
let octalVal  = 0o77         # octal
let binaryVal = 0b1010_1010  # binary
```

### Integer Overflow

```nim
# Nim ตรวจ overflow ใน debug mode
var maxInt8: int8 = 127
# maxInt8 += 1  # Overflow! Error in debug, wraps in release

# ใช้ succ() และ pred()
echo succ(5)  # 6
echo pred(5)  # 4

# Safe operations
import std/math
echo clamp(200, 0, 100)  # 100 (ไม่ให้เกิน range)
```

---

## Step 13: Float Types (ชนิดทศนิยม)

```nim
# Float types
var f32: float32 = 3.14159     # Single precision
var f64: float64 = 3.14159265358979  # Double precision
var f:   float   = 3.14        # alias สำหรับ float64

# Scientific notation
var sci1 = 1.5e10   # 15,000,000,000.0
var sci2 = 2.5e-3   # 0.0025

# Float operations
var a = 10.0
var b = 3.0

echo a + b    # 13.0
echo a - b    # 7.0
echo a * b    # 30.0
echo a / b    # 3.3333333333333335
echo a mod b  # 1.0 (float mod)

# Math functions
import std/math

echo sqrt(16.0)     # 4.0
echo pow(2.0, 10.0) # 1024.0
echo log10(100.0)   # 2.0
echo ln(exp(1.0))   # 1.0
echo sin(PI / 2)    # 1.0
echo cos(0.0)       # 1.0
echo abs(-5.5)      # 5.5
echo floor(3.7)     # 3.0
echo ceil(3.2)      # 4.0
echo round(3.5)     # 4.0

# Special values
echo Inf            # Infinity
echo NaN            # Not a Number
echo -Inf           # Negative Infinity
echo classify(Inf)  # fcInf
echo classify(NaN)  # fcNaN

# Float comparison (ระวัง!)
var x = 0.1 + 0.2
echo x         # 0.30000000000000004 (floating point issue!)
echo x == 0.3  # false!

# วิธีที่ถูกต้องในการเปรียบเทียบ float
proc almostEqual(a, b: float, epsilon: float = 1e-9): bool =
  abs(a - b) < epsilon

echo almostEqual(0.1 + 0.2, 0.3)  # true
```

---

## Step 14: Boolean Type และ String Type

### Boolean

```nim
# Boolean
var isReady: bool = true
var hasError: bool = false

# Boolean operations
echo true and false   # false
echo true or false    # true
echo not true         # false
echo true xor true    # false (XOR)

# Comparison operators (ผลลัพธ์เป็น bool)
var x = 10
echo x > 5    # true
echo x < 5    # false
echo x >= 10  # true
echo x <= 10  # true
echo x == 10  # true
echo x != 5   # true

# Short-circuit evaluation
proc check1(): bool =
  echo "check1 called"
  return true

proc check2(): bool =
  echo "check2 called"
  return false

# and: ถ้า check1 เป็น false จะไม่เรียก check2
if check1() and check2():
  echo "both true"

# or: ถ้า check1 เป็น true จะไม่เรียก check2
if check1() or check2():
  echo "at least one true"
```

### String Type

```nim
import std/strutils, std/strformat

# String declaration
var greeting: string = "สวัสดี"
var name = "Nim"
let message = "Hello, World!"

# String concatenation
var fullMessage = greeting & ", " & name & "!"
echo fullMessage  # สวัสดี, Nim!

# String length
echo greeting.len   # 12 (bytes, not characters for Unicode!)
echo greeting.runeLen()  # 6 (actual Thai characters)

# String indexing (ระวัง: index เป็น byte, ไม่ใช่ character)
echo message[0]    # H
echo message[^1]   # ! (last character)
echo message[0..4] # Hello

# String methods
var s = "  Hello World  "
echo s.strip()              # "Hello World"
echo s.toLower()            # "  hello world  "
echo s.toUpper()            # "  HELLO WORLD  "
echo s.replace("Hello", "Hi")  # "  Hi World  "
echo s.contains("World")   # true
echo s.startsWith("  H")   # true
echo s.endsWith("  ")      # true
echo s.count("l")          # 3

# Split and Join
var csv = "apple,banana,orange"
var fruits = csv.split(",")
echo fruits        # @["apple", "banana", "orange"]
echo fruits.join(", ")  # apple, banana, orange

# String formatting
let price = 29.99
let qty = 3
echo fmt"ราคา: {price:.2f} บาท, จำนวน: {qty} ชิ้น"
echo fmt"รวม: {price * qty.float:.2f} บาท"

# Multiline strings
var multiline = """
  บรรทัดที่ 1
  บรรทัดที่ 2
  บรรทัดที่ 3
"""
echo multiline

# String interpolation
let user = "Alice"
let score = 95
echo &"ผู้ใช้ {user} ได้คะแนน {score}/100"
```

---

## Step 15: Char Type และ Special Types

### Char Type

```nim
# Char - ตัวอักษรเดี่ยว (1 byte, ASCII only)
var c: char = 'A'
var newline: char = '\n'
var tab: char = '\t'
var backslash: char = '\\'
var quote: char = '\''

# Char operations
echo c           # A
echo ord(c)      # 65 (ASCII code)
echo chr(65)     # A
echo c.isAlpha() # true
echo c.isDigit() # false
echo c.isLower() # false
echo c.isUpper() # true
echo c.toLower() # a
echo c.toUpper() # A

# Char comparison
echo 'A' < 'B'   # true
echo 'Z' > 'A'   # true
echo 'a' == 'a'  # true
```

### Nil Type

```nim
# nil - ค่าว่างสำหรับ reference types
var p: ref int = nil      # nil reference
var s: seq[int] = @[]     # empty sequence (ไม่ใช่ nil)

# เช็ค nil
if p == nil:
  echo "p is nil"
  
# ใน Nim ส่วนใหญ่ใช้ Option type แทน nil
import std/options

var maybeValue: Option[int] = some(42)
var noValue: Option[int] = none(int)

echo maybeValue.isSome()    # true
echo noValue.isNone()       # true
echo maybeValue.get()       # 42
echo maybeValue.unsafeGet() # 42 (ไม่มี check)
```

---

## Step 16: Type Conversion (การแปลงชนิดข้อมูล)

```nim
import std/strutils

# Numeric conversions
var i: int = 42
var f: float = 3.14
var i8: int8 = 100

# int -> float
var fromInt: float = float(i)      # 42.0
var fromInt2: float = i.toFloat()  # 42.0 (method style)

# float -> int (ตัดทศนิยม)
var fromFloat: int = int(f)        # 3
var fromFloat2: int = f.toInt()    # 3

# int8 -> int
var toInt: int = int(i8)           # 100

# int -> int8 (อาจ overflow!)
var toInt8: int8 = int8(200)       # overflow! ระวัง

# String conversions
var numStr = "42"
var floatStr = "3.14"
var boolStr = "true"

# string -> int
var parsedInt: int = parseInt(numStr)       # 42
var parsedInt2: int = numStr.parseInt()    # 42 (method style)

# string -> float
var parsedFloat = parseFloat(floatStr)     # 3.14

# string -> bool
var parsedBool = parseBool(boolStr)        # true

# int/float -> string
var intToStr: string = $i         # "42"
var floatToStr: string = $f       # "3.14"
var boolToStr: string = $true     # "true"

# Format string
var formatted = fmt"{f:.4f}"     # "3.1400"

# Safe parsing (ไม่ throw exception)
var (ok, value) = (true, 0)
try:
  value = parseInt("abc")
except ValueError:
  ok = false
  
if not ok:
  echo "การ parse ล้มเหลว"

# หรือใช้ tryParse
import std/strutils
var result: int
if result.parseInt("123"):
  echo "parsed: " & $result
```

---

## Step 17: Type Aliases และ Distinct Types

```nim
# Type Alias - เป็นชื่อย่อของ type
type
  Filename = string
  Username = string
  Port = int
  Seconds = float

var file: Filename = "data.txt"
var user: Username = "alice"
var port: Port = 8080
var timeout: Seconds = 30.0

# แต่ type aliases สามารถผสมกันได้ (ไม่มี type safety!)
var f: Filename = user  # OK! เพราะทั้งคู่เป็น string

# Distinct Type - Type ใหม่ที่แตกต่างกันจริงๆ
type
  Meter = distinct float
  Kilogram = distinct float
  Second = distinct float

var distance: Meter = Meter(10.5)
var weight: Kilogram = Kilogram(70.0)
var time: Second = Second(5.0)

# ไม่สามารถผสมกันได้!
# var wrong: Meter = weight  # Error! type mismatch

# ต้อง convert ก่อน
var distFloat: float = float(distance)

# Distinct Type ที่ใช้งานได้จริง
type
  UserId = distinct int
  ProductId = distinct int

proc getUserById(id: UserId): string =
  return "User " & $int(id)

proc getProductById(id: ProductId): string =
  return "Product " & $int(id)

var uid = UserId(1)
var pid = ProductId(1)

echo getUserById(uid)    # OK
echo getProductById(pid) # OK
# echo getUserById(pid)  # Error! type mismatch - 

# Borrow keyword สำหรับ distinct types
type
  NormalizedString = distinct string

proc `$`(s: NormalizedString): string {.borrow.}  # borrow $ from string
proc len(s: NormalizedString): int {.borrow.}       # borrow len from string

var ns = NormalizedString("hello")
echo ns.len    # 5
echo $ns       # "hello"
```

---

## Step 18: Enum Types (ชนิดข้อมูลแจงนับ)

```nim
# Enum พื้นฐาน
type
  Color = enum
    Red, Green, Blue

  Direction = enum
    North, South, East, West

  Day = enum
    Monday, Tuesday, Wednesday, Thursday, Friday, Saturday, Sunday

# ใช้งาน Enum
var myColor: Color = Red
var dir: Direction = North
var today: Day = Wednesday

# Enum มีค่าเป็น int โดยอัตโนมัติ
echo int(Red)    # 0
echo int(Green)  # 1
echo int(Blue)   # 2

# Enum with custom values
type
  HttpStatus = enum
    Ok = 200
    Created = 201
    BadRequest = 400
    Unauthorized = 401
    NotFound = 404
    InternalServerError = 500

echo int(Ok)          # 200
echo int(NotFound)    # 404

# Enum with string values
type
  Season = enum
    Spring = "spring"
    Summer = "summer"
    Autumn = "autumn"
    Winter = "winter"

echo $Spring  # spring
echo $Winter  # winter

# Enum operations
echo succ(Monday)   # Tuesday
echo pred(Friday)   # Thursday
echo ord(Wednesday) # 2

# Enum in case statement
let weather = Summer
case weather
of Spring, Autumn:
  echo "อากาศเย็นสบาย"
of Summer:
  echo "อากาศร้อน"
of Winter:
  echo "อากาศหนาว"

# Enum sets
type
  Permission = enum
    Read, Write, Execute

  PermissionSet = set[Permission]

var filePermissions: PermissionSet = {Read, Write}

echo filePermissions.contains(Read)     # true
echo filePermissions.contains(Execute)  # false

filePermissions.incl(Execute)
echo filePermissions  # {Read, Write, Execute}

filePermissions.excl(Write)
echo filePermissions  # {Read, Execute}
```

---

## Step 19: Ranges และ Ordinal Types

```nim
# Range types
type
  Percentage = range[0..100]
  SmallInt = range[-100..100]
  AsciiChar = range[char('A')..char('Z')]

var pct: Percentage = 75
var small: SmallInt = -50
var letter: AsciiChar = 'M'

# ตรวจสอบ range ใน debug mode
# var overPct: Percentage = 150  # Error! out of range

# Subrange ของ enum
type
  Weekday = range[Monday..Friday]  # เฉพาะวันทำงาน

var workday: Weekday = Wednesday
# var weekend: Weekday = Saturday  # Error!

# Ordinal types ทั้งหมด
# int, char, bool, enum, range ล้วนเป็น ordinal

# Low and High
echo low(int8)    # -128
echo high(int8)   # 127
echo low(bool)    # false
echo high(bool)   # true
echo low(char)    # '\x00'
echo high(char)   # '\xff'

# Iterate ordinal
for c in 'A'..'Z':
  write(stdout, c)
echo ""  # ABCDEFGHIJKLMNOPQRSTUVWXYZ

for i in 1..10:
  write(stdout, $i & " ")
echo ""  # 1 2 3 4 5 6 7 8 9 10
```

---

## Step 20: Type System Exercise - Build a Type-Safe API

มาสร้าง Type-Safe Data Model สำหรับระบบ User Management:

```nim
# user_management.nim
# Type-Safe User Management System

import std/times, std/strutils, std/strformat, std/options

# ==============================
# Type Definitions
# ==============================

type
  UserId = distinct int
  Email = distinct string
  HashedPassword = distinct string
  
  UserRole = enum
    Guest, Member, Moderator, Admin, SuperAdmin
  
  UserStatus = enum
    Active, Inactive, Banned, Pending
  
  AgeRange = range[0..150]
  
  UserProfile = object
    userId: UserId
    email: Email
    username: string
    role: UserRole
    status: UserStatus
    age: Option[AgeRange]
    createdAt: DateTime
    lastLogin: Option[DateTime]

# ==============================
# Constructor / Factory
# ==============================

proc newUserId(id: int): UserId =
  if id <= 0:
    raise newException(ValueError, "UserId ต้องเป็นบวก")
  return UserId(id)

proc newEmail(email: string): Email =
  let trimmed = email.strip().toLower()
  if not trimmed.contains('@'):
    raise newException(ValueError, "Email ไม่ถูกต้อง")
  return Email(trimmed)

proc newUserProfile(
  id: int,
  emailStr: string,
  username: string,
  role: UserRole = Member,
  age: int = -1
): UserProfile =
  result.userId = newUserId(id)
  result.email = newEmail(emailStr)
  result.username = username.strip()
  result.role = role
  result.status = Pending
  result.createdAt = now()
  result.lastLogin = none(DateTime)
  
  if age >= 0:
    result.age = some(AgeRange(age))
  else:
    result.age = none(AgeRange)

# ==============================
# Methods
# ==============================

proc `$`(uid: UserId): string = "User#" & $int(uid)
proc `$`(email: Email): string = string(email)

proc activate(user: var UserProfile) =
  user.status = Active
  echo fmt"✅ {user.username} ถูก activate แล้ว"

proc ban(user: var UserProfile, reason: string) =
  user.status = Banned
  echo fmt"🚫 {user.username} ถูก ban เนื่องจาก: {reason}"

proc promote(user: var UserProfile, newRole: UserRole) =
  let oldRole = user.role
  user.role = newRole
  echo fmt"⬆️  {user.username}: {oldRole} → {newRole}"

proc login(user: var UserProfile) =
  if user.status != Active:
    echo fmt"❌ {user.username} ไม่สามารถ login ได้ (status: {user.status})"
    return
  user.lastLogin = some(now())
  echo fmt"✅ {user.username} login สำเร็จ"

proc displayInfo(user: UserProfile) =
  echo "\n" & "━".repeat(40)
  echo fmt"👤 User Profile"
  echo "━".repeat(40)
  echo fmt"ID:       {user.userId}"
  echo fmt"Email:    {user.email}"
  echo fmt"Username: {user.username}"
  echo fmt"Role:     {user.role}"
  echo fmt"Status:   {user.status}"
  
  if user.age.isSome():
    echo fmt"Age:      {user.age.get()} ปี"
  else:
    echo "Age:      ไม่ระบุ"
    
  echo fmt"Created:  {user.createdAt.format("dd/MM/yyyy")}"
  
  if user.lastLogin.isSome():
    echo fmt"Last Login: {user.lastLogin.get().format("dd/MM/yyyy HH:mm")}"
  else:
    echo "Last Login: ยังไม่เคย login"

# ==============================
# Main Program
# ==============================

proc main() =
  echo "╔══════════════════════════════════════╗"
  echo "║    Type-Safe User Management Demo    ║"
  echo "╚══════════════════════════════════════╝"
  
  # สร้าง users
  var alice = newUserProfile(
    id = 1,
    emailStr = "alice@example.com",
    username = "Alice",
    role = Member,
    age = 28
  )
  
  var bob = newUserProfile(
    id = 2,
    emailStr = "bob@example.com",
    username = "Bob",
    role = Guest
  )
  
  var admin = newUserProfile(
    id = 100,
    emailStr = "admin@example.com",
    username = "Admin",
    role = Admin,
    age = 35
  )
  
  # แสดงข้อมูล
  alice.displayInfo()
  bob.displayInfo()
  
  echo "\n--- User Operations ---"
  
  # Activate users
  alice.activate()
  admin.activate()
  
  # Login
  alice.login()
  bob.login()    # จะ fail เพราะยัง pending
  
  # Activate bob แล้ว login ใหม่
  bob.activate()
  bob.login()
  
  # Promote alice
  alice.promote(Moderator)
  
  # Ban bob
  bob.ban("ละเมิดกฎของชุมชน")
  bob.login()  # จะ fail เพราะ banned
  
  # แสดงข้อมูลหลัง operations
  echo "\n--- Final Status ---"
  alice.displayInfo()
  bob.displayInfo()
  admin.displayInfo()
  
  # Type safety demo
  echo "\n--- Type Safety Demo ---"
  let uid1 = newUserId(1)
  let uid2 = newUserId(2)
  echo fmt"uid1: {uid1}"
  echo fmt"uid2: {uid2}"
  # echo uid1 + uid2  # Error! cannot add UserIds (by design!)
  
  # Email validation
  try:
    let badEmail = newEmail("notanemail")
    discard badEmail
  except ValueError as e:
    echo fmt"❌ Email error: {e.msg}"

main()
```

```bash
# Compile และ Run
nim c -r user_management.nim

# Expected Output:
# ╔══════════════════════════════════════╗
# ║    Type-Safe User Management Demo    ║
# ╚══════════════════════════════════════╝
# ...
```

---

## 📝 สรุป Part 02

| Step | หัวข้อ | สิ่งที่เรียนรู้ |
|------|--------|----------------|
| 11 | var, let, const | ประกาศตัวแปร 3 แบบ |
| 12 | Integer Types | int8 ถึง int64, operations |
| 13 | Float Types | float32, float64, math functions |
| 14 | Bool & String | boolean logic, string manipulation |
| 15 | Char & Nil | single characters, option types |
| 16 | Type Conversion | casting, parsing |
| 17 | Type Aliases & Distinct | custom types |
| 18 | Enum Types | enumeration, sets |
| 19 | Ranges & Ordinals | range types, ordinal operations |
| 20 | Type-Safe API | real-world application |

---

## 🏋️ แบบฝึกหัด Part 02

### Exercise 1: Data Validation System

```nim
# สร้างระบบ validation สำหรับ form data
type
  FormData = object
    name: string
    email: string
    age: int
    phone: string

proc validateForm(data: FormData): seq[string] =
  # TODO: Return list of validation errors
  # - name ต้องมีความยาว 2-50 ตัวอักษร
  # - email ต้อง valid
  # - age ต้องอยู่ระหว่าง 13-120
  # - phone ต้องมีแค่ตัวเลขและ - และ +
  discard
```

### Exercise 2: Unit Converter

```nim
# สร้าง type-safe unit converter
type
  Meters = distinct float
  Feet = distinct float
  Inches = distinct float
  Centimeters = distinct float

# Implement conversion procedures
proc toFeet(m: Meters): Feet = ?
proc toInches(m: Meters): Inches = ?
proc toCentimeters(m: Meters): Centimeters = ?
```

### Exercise 3: Status Machine

```nim
# สร้าง Order Status Machine
type
  OrderStatus = enum
    Draft, Pending, Confirmed, Shipped, Delivered, Cancelled

proc canTransition(from: OrderStatus, to: OrderStatus): bool =
  # TODO: Define valid transitions
  # Draft -> Pending -> Confirmed -> Shipped -> Delivered
  # Any -> Cancelled (except Delivered)
  discard
```

---

**← [Part 01: เริ่มต้น Nim](part_01_introduction.md) | [Part 03: Control Flow →](part_03_control_flow.md)**
