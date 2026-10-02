# Part 10: Modules & Package Management
## Steps 116-130: การจัดการ Modules และ Packages

---

## 🎯 เป้าหมายของ Part นี้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
- สร้างและใช้ Modules
- จัดการ namespaces
- ใช้ Nimble package manager
- สร้าง Libraries ของตัวเอง
- จัดการ Dependencies
- เขียน .nimble file

---

## Step 116: Modules พื้นฐาน

```nim
# ไฟล์: mymath.nim
# สร้าง module ของตัวเอง

proc add*(a, b: int): int = a + b        # public (*)
proc subtract*(a, b: int): int = a - b   # public
proc multiply*(a, b: int): int = a * b   # public

proc internalHelper(x: int): int = x * 2  # private (ไม่มี *)

# Constants
const PI* = 3.14159265358979

# Types
type
  Point2D* = object
    x*, y*: float   # fields ที่ public ต้องมี * ด้วย

proc newPoint*(x, y: float): Point2D =
  Point2D(x: x, y: y)

proc distance*(a, b: Point2D): float =
  let dx = a.x - b.x
  let dy = a.y - b.y
  sqrt(dx*dx + dy*dy)
```

```nim
# ไฟล์: main.nim
# ใช้ module

import mymath          # import ทั้งหมด
import std/math        # stdlib

echo add(3, 4)        # 7
echo PI               # 3.14159...

# import เฉพาะบางส่วน
from mymath import Point2D, newPoint, distance

let p1 = newPoint(0, 0)
let p2 = newPoint(3, 4)
echo distance(p1, p2)  # 5.0

# import พร้อม alias
import mymath as mm
echo mm.add(1, 2)   # 3

# qualified import (ต้องใช้ prefix เสมอ)
import mymath
echo mymath.add(5, 6)  # 11
```

---

## Step 117: Module Organization

```nim
# โครงสร้างโปรเจค:
# myapp/
# ├── myapp.nim        (main file)
# ├── myapp.nimble     (package file)
# ├── src/
# │   ├── config.nim
# │   ├── database.nim
# │   ├── models/
# │   │   ├── user.nim
# │   │   └── post.nim
# │   └── controllers/
# │       ├── user_controller.nim
# │       └── post_controller.nim
# └── tests/
#     ├── test_user.nim
#     └── test_post.nim

# src/models/user.nim
type
  UserId* = distinct int
  
  User* = object
    id*: UserId
    name*: string
    email*: string
    createdAt*: string

proc `$`*(id: UserId): string = "U" & $int(id)

proc newUser*(id: int, name, email: string): User* =
  User(
    id: UserId(id),
    name: name,
    email: email,
    createdAt: "2024-01-01"
  )

# src/models/post.nim
import user  # relative import

type
  PostId* = distinct int
  
  Post* = object
    id*: PostId
    title*: string
    content*: string
    authorId*: UserId
    published*: bool

# Re-export with include
# (ทำให้ import post ได้ User type ด้วย)
export user  # re-export user module

# src/controllers/user_controller.nim
import ../models/user

proc getUser*(id: int): User =
  # Simulate database query
  return newUser(id, "Alice", "alice@example.com")

proc listUsers*(): seq[User] =
  return @[
    newUser(1, "Alice", "alice@example.com"),
    newUser(2, "Bob", "bob@example.com"),
  ]
```

---

## Step 118: .nimble File

```nim
# myapp.nimble - Package descriptor file

# Package info
version = "1.0.0"
author = "Your Name"
description = "A sample Nim backend application"
license = "MIT"

# Nim version requirement
requires "nim >= 2.0.0"

# Dependencies
requires "jester >= 0.5.0"          # Web framework
requires "norm >= 2.8.0"            # ORM
requires "jsony >= 1.1.3"           # JSON library
requires "dotenv >= 2.0.0"          # .env file support

# Source directory
srcDir = "src"

# Binary to build
bin = @["myapp"]

# Build options
# --define:release for production

# Tasks (like npm scripts)
task test, "Run tests":
  exec "nim c -r tests/all_tests.nim"

task build, "Build for production":
  exec "nim c -d:release -o:bin/myapp src/myapp.nim"

task docker, "Build docker image":
  exec "docker build -t myapp ."

task migrate, "Run database migrations":
  exec "nim c -r src/migrations.nim"

task format, "Format code":
  exec "nimpretty --indent:2 src/**/*.nim"
```

---

## Step 119: Nimble Commands

```bash
# Package management
nimble init          # สร้าง .nimble file ใหม่
nimble install       # ติดตั้ง dependencies ทั้งหมด
nimble install jester  # ติดตั้ง package เฉพาะ
nimble uninstall jester

# Version management
nimble install jester@0.5.0    # specific version
nimble install jester@>0.5.0   # version range
nimble install jester@latest   # latest version

# Build
nimble build         # build project
nimble build -d:release  # build with release optimization

# Run
nimble run           # run main binary
nimble run myapp     # run specific binary

# Tasks
nimble test          # run tests
nimble migrate       # run custom task

# Search and info
nimble search jester         # search packages
nimble show jester           # show package info
nimble list --installed      # list installed packages

# Publish
nimble publish       # publish to package registry
```

---

## Step 120: Standard Library Overview

```nim
# สำคัญมากสำหรับ backend

# String operations
import std/strutils    # split, join, replace, toLower, etc.
import std/strformat   # fmt"Hello {name}"
import std/strscans    # scanf-like scanning

# Collections
import std/sequtils    # map, filter, foldl, etc.
import std/tables      # Table, OrderedTable
import std/sets        # HashSet, OrderedSet
import std/deques      # Deque
import std/heapqueue   # HeapQueue

# Data formats
import std/json        # parse, generate JSON
import std/xmltree     # XML
import std/parsecfg    # .ini config files
import std/csv         # CSV

# File system
import std/os          # files, dirs, env vars
import std/paths       # path operations
import std/streams     # file streams
import std/io          # low-level I/O

# Networking
import std/net         # TCP/UDP sockets
import std/asyncnet    # async networking
import std/httpclient  # HTTP client
import std/asynchttpclient  # async HTTP client
import std/uri         # URL parsing

# Concurrency
import std/asyncdispatch  # async/await
import std/asyncio        # async file I/O
import std/threadpool     # thread pools
import std/locks          # mutexes, condition vars

# Time
import std/times       # DateTime, now(), etc.
import std/monotimes   # monotonic clock for benchmarking

# Math
import std/math        # sqrt, pow, floor, etc.
import std/random      # random number generation
import std/complex     # complex numbers
import std/stats       # statistics

# Algorithms
import std/algorithm   # sort, reverse, binary search
import std/hashes      # hash functions

# Testing
import std/unittest    # unit testing framework

# System
import std/osproc      # run external processes
import std/cpuinfo     # CPU information

# Other
import std/options     # Option[T] type
import std/sugar       # => arrow syntax, dup, etc.
import std/with        # with macro
import std/enumerate   # enumerate iterator
import std/logging     # logging
```

---

## Step 121-130: Full Project Structure

```nim
# src/app.nim - Application entry point

import std/asyncdispatch, std/os, std/strformat
import ./config
import ./server
import ./database

proc main() {.async.} =
  echo "╔══════════════════════════════════════╗"
  echo "║     Nim Backend Application v1.0     ║"
  echo "╚══════════════════════════════════════╝"
  
  # Load configuration
  let cfg = loadConfig()
  echo fmt"  Port: {cfg.serverPort}"
  echo fmt"  Debug: {cfg.debug}"
  echo fmt"  Database: {cfg.dbName}@{cfg.dbHost}"
  
  # Initialize database
  echo "\n  Connecting to database..."
  await initDatabase(cfg)
  echo "  ✅ Database connected"
  
  # Start server
  echo fmt"\n  Starting server on {cfg.serverHost}:{cfg.serverPort}..."
  await startServer(cfg)

when isMainModule:
  waitFor main()
```

```nim
# src/config.nim

import std/os, std/strutils

type
  Config* = object
    serverHost*: string
    serverPort*: int
    debug*: bool
    dbHost*: string
    dbPort*: int
    dbName*: string
    dbUser*: string
    dbPassword*: string
    jwtSecret*: string
    logLevel*: string

proc loadConfig*(): Config =
  Config(
    serverHost:  getEnv("HOST", "0.0.0.0"),
    serverPort:  parseInt(getEnv("PORT", "8080")),
    debug:       parseBool(getEnv("DEBUG", "false")),
    dbHost:      getEnv("DB_HOST", "localhost"),
    dbPort:      parseInt(getEnv("DB_PORT", "5432")),
    dbName:      getEnv("DB_NAME", "myapp"),
    dbUser:      getEnv("DB_USER", "postgres"),
    dbPassword:  getEnv("DB_PASSWORD", ""),
    jwtSecret:   getEnv("JWT_SECRET", "change-me-in-production"),
    logLevel:    getEnv("LOG_LEVEL", "info")
  )
```

```nim
# src/models/user.nim

import std/times, std/options

type
  UserId* = distinct int
  Email* = distinct string
  
  UserRole* = enum
    urUser = "user"
    urAdmin = "admin"
    urModerator = "moderator"
  
  User* = object
    id*: UserId
    name*: string
    email*: Email
    role*: UserRole
    isActive*: bool
    createdAt*: DateTime
    updatedAt*: DateTime

proc `$`*(id: UserId): string = $int(id)
proc `$`*(e: Email): string = string(e)
proc `==`*(a, b: UserId): bool = int(a) == int(b)
proc `==`*(a, b: Email): bool = string(a).toLower() == string(b).toLower()

proc newUser*(
  id: int, name: string, email: string,
  role: UserRole = urUser
): User* =
  let now2 = now()
  User(
    id: UserId(id),
    name: name,
    email: Email(email),
    role: role,
    isActive: true,
    createdAt: now2,
    updatedAt: now2
  )

proc toJson*(user: User): string =
  fmt"""{{
  "id": {$user.id},
  "name": "{user.name}",
  "email": "{$user.email}",
  "role": "{$user.role}",
  "isActive": {$user.isActive}
}}"""
```

```nim
# src/repositories/user_repository.nim

import std/tables, std/options, std/sequtils
import ../models/user

type
  UserFilter* = object
    role*: Option[UserRole]
    isActive*: Option[bool]
    search*: Option[string]

  UserRepository* = ref object
    users: Table[int, User]
    nextId: int

proc newUserRepository*(): UserRepository =
  result = UserRepository(
    users: initTable[int, User](),
    nextId: 1
  )
  
  # Seed data
  let adminUser = newUser(1, "Admin", "admin@example.com", urAdmin)
  result.users[1] = adminUser
  result.nextId = 2

proc findById*(repo: UserRepository, id: int): Option[User] =
  if id in repo.users:
    return some(repo.users[id])
  return none(User)

proc findByEmail*(repo: UserRepository, email: string): Option[User] =
  for user in repo.users.values:
    if $user.email == email.toLower():
      return some(user)
  return none(User)

proc findAll*(repo: UserRepository, filter: UserFilter = UserFilter()): seq[User] =
  result = @[]
  
  for user in repo.users.values:
    # Apply filters
    if filter.role.isSome and user.role != filter.role.get():
      continue
    if filter.isActive.isSome and user.isActive != filter.isActive.get():
      continue
    if filter.search.isSome:
      let search = filter.search.get().toLower()
      if search notin user.name.toLower() and search notin $user.email:
        continue
    
    result.add(user)
  
  # Sort by ID
  result.sort(proc(a, b: User): int = int(a.id) - int(b.id))

proc save*(repo: var UserRepository, user: var User): User =
  if int(user.id) == 0:
    user.id = UserId(repo.nextId)
    inc repo.nextId
  
  user.updatedAt = now()
  repo.users[int(user.id)] = user
  return user

proc delete*(repo: var UserRepository, id: int): bool =
  if id in repo.users:
    repo.users.del(id)
    return true
  return false
```

```nim
# tests/test_user.nim

import std/unittest
import ../src/models/user
import ../src/repositories/user_repository

suite "User Model":
  test "create user with default role":
    let user = newUser(1, "Alice", "alice@example.com")
    check user.name == "Alice"
    check user.role == urUser
    check user.isActive == true
  
  test "email comparison is case-insensitive":
    let email1 = Email("Alice@Example.COM")
    let email2 = Email("alice@example.com")
    check email1 == email2

suite "User Repository":
  setup:
    var repo = newUserRepository()
  
  test "find by id returns user":
    let user = repo.findById(1)
    check user.isSome
    check user.get().name == "Admin"
  
  test "find by id returns none for missing":
    let user = repo.findById(999)
    check user.isNone
  
  test "save creates new user":
    var newUser2 = newUser(0, "Bob", "bob@example.com")
    let saved = repo.save(newUser2)
    check int(saved.id) > 0
    check saved.name == "Bob"
  
  test "findAll with filter by role":
    var adminUser = newUser(0, "Admin2", "admin2@example.com", urAdmin)
    discard repo.save(adminUser)
    
    let admins = repo.findAll(UserFilter(role: some(urAdmin)))
    check admins.len >= 1
    check admins.allIt(it.role == urAdmin)
  
  test "delete removes user":
    let initialCount = repo.findAll().len
    check repo.delete(1) == true
    check repo.findAll().len == initialCount - 1

when isMainModule:
  discard
```

---

## 📝 สรุป Part 10

| Steps | หัวข้อ |
|-------|--------|
| 116 | Module basics, public/private |
| 117 | Module organization |
| 118 | .nimble file |
| 119 | Nimble commands |
| 120 | Standard library overview |
| 121-130 | Full project structure |

---

**← [Part 09: Error Handling](part_09_error_handling.md) | [Part 11: OOP in Nim →](part_11_oop.md)**
