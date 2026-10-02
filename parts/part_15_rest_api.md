# Part 15: Building REST APIs
## Steps 191-210: สร้าง Production REST API

---

## 🎯 เป้าหมายของ Part นี้

- ออกแบบ RESTful API
- JWT Authentication
- Request validation
- Pagination & filtering
- Rate limiting
- API documentation

---

## Step 191: REST API Design Principles

```
RESTful URL Design:

GET    /api/v1/users          → List users
POST   /api/v1/users          → Create user
GET    /api/v1/users/:id      → Get user
PUT    /api/v1/users/:id      → Replace user
PATCH  /api/v1/users/:id      → Update user fields
DELETE /api/v1/users/:id      → Delete user

GET    /api/v1/users/:id/posts     → User's posts
POST   /api/v1/users/:id/posts     → Create post for user

# Filtering, sorting, pagination
GET /api/v1/products?category=electronics&sort=price&order=asc&page=1&limit=20

# Version in URL (recommended for major changes)
/api/v1/...
/api/v2/...

# HTTP Status Codes
200 OK           - Success
201 Created      - Resource created (POST)
204 No Content   - Success but no body (DELETE)
400 Bad Request  - Invalid input
401 Unauthorized - Not authenticated
403 Forbidden    - Authenticated but not authorized
404 Not Found    - Resource doesn't exist
409 Conflict     - Resource already exists
422 Unprocessable - Validation error
429 Too Many Requests - Rate limited
500 Internal Server Error

# Response format
{
  "success": true,
  "data": {...} or [...],
  "meta": {
    "total": 100,
    "page": 1,
    "limit": 20
  }
}

# Error format
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Validation failed",
    "details": [
      {"field": "email", "message": "Invalid email format"}
    ]
  }
}
```

---

## Step 192: JWT Authentication

```nim
import std/base64, std/json, std/hmac, std/strutils, std/times, std/strformat

# Simple JWT implementation (use proper library in production)
type
  JwtHeader = object
    alg: string
    typ: string

  JwtPayload = object
    sub: string      # subject (user ID)
    iss: string      # issuer
    exp: int64       # expiration timestamp
    iat: int64       # issued at timestamp
    role: string
    jti: string      # JWT ID (for revocation)

proc base64UrlEncode(data: string): string =
  result = encode(data)
    .replace("=", "")
    .replace("+", "-")
    .replace("/", "_")

proc base64UrlDecode(data: string): string =
  var padded = data
    .replace("-", "+")
    .replace("_", "/")
  
  while padded.len mod 4 != 0:
    padded.add('=')
  
  return decode(padded)

proc createJwt(payload: JwtPayload, secret: string): string =
  let header = JwtHeader(alg: "HS256", typ: "JWT")
  
  let headerJson = %*{"alg": header.alg, "typ": header.typ}
  let payloadJson = %*{
    "sub": payload.sub,
    "iss": payload.iss,
    "exp": payload.exp,
    "iat": payload.iat,
    "role": payload.role,
    "jti": payload.jti
  }
  
  let headerEncoded  = base64UrlEncode($headerJson)
  let payloadEncoded = base64UrlEncode($payloadJson)
  let signingInput   = headerEncoded & "." & payloadEncoded
  
  let signature = base64UrlEncode(hmac_sha256(secret, signingInput))
  
  return signingInput & "." & signature

proc verifyJwt(token, secret: string): Option[JsonNode] =
  let parts = token.split(".")
  if parts.len != 3:
    return none(JsonNode)
  
  let signingInput = parts[0] & "." & parts[1]
  let expectedSig  = base64UrlEncode(hmac_sha256(secret, signingInput))
  
  if parts[2] != expectedSig:
    return none(JsonNode)  # invalid signature
  
  let payloadJson = parseJson(base64UrlDecode(parts[1]))
  
  # Check expiration
  let exp = payloadJson["exp"].getInt()
  if exp < int(epochTime()):
    return none(JsonNode)  # expired
  
  return some(payloadJson)

# Token management
type
  TokenPair = object
    accessToken: string
    refreshToken: string
    expiresIn: int

proc generateTokens(userId: string, role: string, secret: string): TokenPair =
  let now = int64(epochTime())
  let jti = base64UrlEncode($now & userId)
  
  let accessPayload = JwtPayload(
    sub: userId,
    iss: "myapp",
    exp: now + 3600,  # 1 hour
    iat: now,
    role: role,
    jti: jti
  )
  
  let refreshPayload = JwtPayload(
    sub: userId,
    iss: "myapp",
    exp: now + 86400 * 30,  # 30 days
    iat: now,
    role: role,
    jti: jti & "_refresh"
  )
  
  TokenPair(
    accessToken: createJwt(accessPayload, secret),
    refreshToken: createJwt(refreshPayload, secret & "_refresh"),
    expiresIn: 3600
  )

# Test
let secret = "my-super-secret-key-at-least-32-chars"
let tokens = generateTokens("user123", "admin", secret)

echo "Access token: " & tokens.accessToken[0..50] & "..."
echo "Refresh token: " & tokens.refreshToken[0..50] & "..."

let payload = verifyJwt(tokens.accessToken, secret)
if payload.isSome:
  echo fmt"Valid token for user: {payload.get()[\"sub\"].getStr()}"
  echo fmt"Role: {payload.get()[\"role\"].getStr()}"
```

---

## Step 193: Request Validation Framework

```nim
import std/json, std/strutils, std/re, std/strformat, std/sequtils

type
  FieldType = enum
    ftString, ftInt, ftFloat, ftBool, ftArray, ftObject, ftEmail, ftUrl, ftUuid

  FieldRule = object
    fieldType: FieldType
    required: bool
    minLen: int
    maxLen: int
    minVal: float
    maxVal: float
    pattern: string
    allowedValues: seq[string]

  ValidationSchema = Table[string, FieldRule]

  FieldError2 = object
    field: string
    message: string
    value: string

  ValidationReport = object
    valid: bool
    errors: seq[FieldError2]

proc required(ft: FieldType): FieldRule =
  FieldRule(fieldType: ft, required: true, minLen: 0, maxLen: high(int))

proc optional(ft: FieldType): FieldRule =
  FieldRule(fieldType: ft, required: false, minLen: 0, maxLen: high(int))

proc withMinLen(rule: FieldRule, n: int): FieldRule =
  result = rule
  result.minLen = n

proc withMaxLen(rule: FieldRule, n: int): FieldRule =
  result = rule
  result.maxLen = n

proc withRange(rule: FieldRule, min, max: float): FieldRule =
  result = rule
  result.minVal = min
  result.maxVal = max

proc withPattern(rule: FieldRule, pattern: string): FieldRule =
  result = rule
  result.pattern = pattern

proc withAllowed(rule: FieldRule, values: seq[string]): FieldRule =
  result = rule
  result.allowedValues = values

proc validate(body: JsonNode, schema: ValidationSchema): ValidationReport =
  var report = ValidationReport(valid: true, errors: @[])
  
  for fieldName, rule in schema:
    let node = body{fieldName}
    
    # Check required
    if rule.required and (node.isNil or node.kind == JNull):
      report.errors.add(FieldError2(
        field: fieldName,
        message: fieldName & " is required",
        value: ""
      ))
      report.valid = false
      continue
    
    if node.isNil or node.kind == JNull:
      continue  # optional field, skip
    
    let value = $node
    
    # Type checking
    case rule.fieldType
    of ftString:
      if node.kind != JString:
        report.errors.add(FieldError2(field: fieldName, message: "Must be a string", value: value))
        report.valid = false
        continue
      
      let str = node.getStr()
      if str.len < rule.minLen:
        report.errors.add(FieldError2(field: fieldName,
          message: fmt"Must be at least {rule.minLen} characters", value: str))
        report.valid = false
      
      if str.len > rule.maxLen:
        report.errors.add(FieldError2(field: fieldName,
          message: fmt"Must be at most {rule.maxLen} characters", value: str[0..min(20, str.len-1)]))
        report.valid = false
      
      if rule.pattern.len > 0 and not str.match(re(rule.pattern)):
        report.errors.add(FieldError2(field: fieldName,
          message: "Invalid format", value: str))
        report.valid = false
    
    of ftEmail:
      if node.kind != JString:
        report.errors.add(FieldError2(field: fieldName, message: "Must be a string"))
        report.valid = false
        continue
      
      let email = node.getStr()
      if not email.match(re"^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$"):
        report.errors.add(FieldError2(field: fieldName,
          message: "Must be a valid email address", value: email))
        report.valid = false
    
    of ftInt:
      if node.kind != JInt:
        report.errors.add(FieldError2(field: fieldName, message: "Must be an integer"))
        report.valid = false
        continue
      
      let num = float(node.getInt())
      if rule.minVal != 0 and num < rule.minVal:
        report.errors.add(FieldError2(field: fieldName,
          message: fmt"Must be >= {rule.minVal:.0f}"))
        report.valid = false
      
      if rule.maxVal != 0 and num > rule.maxVal:
        report.errors.add(FieldError2(field: fieldName,
          message: fmt"Must be <= {rule.maxVal:.0f}"))
        report.valid = false
    
    else: discard
    
    # Allowed values check
    if rule.allowedValues.len > 0:
      let strVal = node.getStr()
      if strVal notin rule.allowedValues:
        report.errors.add(FieldError2(field: fieldName,
          message: fmt"Must be one of: {rule.allowedValues.join(\", \")}",
          value: strVal))
        report.valid = false
  
  return report

# Define schemas
let createUserSchema: ValidationSchema = {
  "name":     required(ftString).withMinLen(2).withMaxLen(100),
  "email":    required(ftEmail),
  "password": required(ftString).withMinLen(8).withMaxLen(100),
  "role":     optional(ftString).withAllowed(@["user", "admin", "moderator"]),
}.toTable()

let updateProductSchema: ValidationSchema = {
  "name":     optional(ftString).withMinLen(2).withMaxLen(255),
  "price":    optional(ftInt).withRange(0, 9999999),
  "stock":    optional(ftInt).withRange(0, 99999),
  "category": optional(ftString).withAllowed(@["Electronics", "Stationery", "Office"]),
}.toTable()

# Test validation
let testBody = parseJson("""
{
  "name": "A",
  "email": "not-an-email",
  "password": "short",
  "role": "superadmin"
}
""")

let report = validate(testBody, createUserSchema)
echo fmt"Valid: {report.valid}"
for err in report.errors:
  echo fmt"  ❌ {err.field}: {err.message}"
```

---

## Step 194-210: Complete REST API

```nim
# rest_api.nim - Complete Blog API

import jester, std/asyncdispatch, std/json, std/strformat,
       std/tables, std/options, std/times, std/strutils, std/sequtils,
       db_connector/db_sqlite

# ==============================
# Database Setup
# ==============================

let db = open("blog.db", "", "", "")

db.exec(sql"""
  CREATE TABLE IF NOT EXISTS users (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT NOT NULL,
    email TEXT UNIQUE NOT NULL,
    password_hash TEXT NOT NULL,
    role TEXT DEFAULT 'user',
    created_at TEXT DEFAULT (datetime('now'))
  )
""")

db.exec(sql"""
  CREATE TABLE IF NOT EXISTS posts (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    title TEXT NOT NULL,
    slug TEXT UNIQUE NOT NULL,
    content TEXT,
    author_id INTEGER NOT NULL REFERENCES users(id),
    status TEXT DEFAULT 'draft',
    created_at TEXT DEFAULT (datetime('now')),
    updated_at TEXT DEFAULT (datetime('now'))
  )
""")

db.exec(sql"""
  CREATE TABLE IF NOT EXISTS tags (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT UNIQUE NOT NULL
  )
""")

db.exec(sql"""
  CREATE TABLE IF NOT EXISTS post_tags (
    post_id INTEGER REFERENCES posts(id) ON DELETE CASCADE,
    tag_id INTEGER REFERENCES tags(id) ON DELETE CASCADE,
    PRIMARY KEY (post_id, tag_id)
  )
""")

# ==============================
# Helpers
# ==============================

let JWT_SECRET = "change-this-in-production-minimum-32-chars"

proc jsonResp(data: JsonNode, status: HttpCode = Http200): (HttpCode, string, string) =
  (status, $data, "application/json")

proc errorResp(msg: string, status: HttpCode = Http400, code: string = "ERROR"): (HttpCode, string, string) =
  jsonResp(%*{"success": false, "error": {"code": code, "message": msg}}, status)

proc successResp(data: JsonNode, meta: JsonNode = newJNull()): (HttpCode, string, string) =
  var resp = %*{"success": true, "data": data}
  if meta.kind != JNull:
    resp["meta"] = meta
  jsonResp(resp)

proc getUserFromToken(token: string): Option[JsonNode] =
  # Simplified - real implementation would verify JWT
  if token.len > 0:
    return some(%*{"id": 1, "role": "admin"})
  return none(JsonNode)

proc getCurrentUser(request: Request): Option[JsonNode] =
  let auth = request.headers.getOrDefault("authorization", "")
  if auth.startsWith("Bearer "):
    return getUserFromToken(auth[7..^1])
  return none(JsonNode)

proc slugify(title: string): string =
  result = title.toLower()
    .replace(" ", "-")
    .replace("'", "")
    .replace("\"", "")
  # Remove non-alphanumeric except hyphen
  result = result.filterIt(it.isAlphaAscii() or it.isDigit() or it == '-').join("")

# ==============================
# Routes
# ==============================

router blogApi:
  
  # ── Auth ───────────────────
  
  post "/api/v1/auth/register":
    let body = try: parseJson(request.body)
               except: return errorResp("Invalid JSON", Http400)
    
    let name  = body{"name"}.getStr("")
    let email = body{"email"}.getStr("")
    let pass  = body{"password"}.getStr("")
    
    if name.len < 2:
      return errorResp("Name must be at least 2 characters", Http422)
    if '@' notin email:
      return errorResp("Invalid email address", Http422)
    if pass.len < 8:
      return errorResp("Password must be at least 8 characters", Http422)
    
    # Check duplicate
    let existing = db.getRow(sql"SELECT id FROM users WHERE email = ?", email)
    if existing[0].len > 0:
      return errorResp("Email already registered", Http409, "EMAIL_EXISTS")
    
    let userId = db.insertID(sql"""
      INSERT INTO users (name, email, password_hash) VALUES (?, ?, ?)
    """, name, email, "bcrypt_" & pass)  # Use real bcrypt in production
    
    let user = %*{
      "id": userId,
      "name": name,
      "email": email,
      "role": "user"
    }
    
    return successResp(user)
  
  post "/api/v1/auth/login":
    let body = try: parseJson(request.body)
               except: return errorResp("Invalid JSON", Http400)
    
    let email = body{"email"}.getStr("")
    let pass  = body{"password"}.getStr("")
    
    let row = db.getRow(sql"""
      SELECT id, name, email, role FROM users 
      WHERE email = ? AND password_hash = ?
    """, email, "bcrypt_" & pass)
    
    if row[0].len == 0:
      return errorResp("Invalid email or password", Http401, "INVALID_CREDENTIALS")
    
    # Generate token (simplified)
    let token = "token_" & row[0] & "_" & $int(epochTime())
    
    return successResp(%*{
      "token": token,
      "tokenType": "Bearer",
      "expiresIn": 3600,
      "user": {
        "id": parseInt(row[0]),
        "name": row[1],
        "email": row[2],
        "role": row[3]
      }
    })
  
  # ── Posts ───────────────────
  
  get "/api/v1/posts":
    let page   = try: parseInt(request.params.getOrDefault("page", "1")) except: 1
    let limit  = min(try: parseInt(request.params.getOrDefault("limit", "10")) except: 10, 100)
    let status = request.params.getOrDefault("status", "published")
    let search = request.params.getOrDefault("q", "")
    let offset = (page - 1) * limit
    
    var whereParts = @["p.status = ?"]
    var params = @[status]
    
    if search.len > 0:
      whereParts.add("(p.title LIKE ? OR p.content LIKE ?)")
      params.add("%" & search & "%")
      params.add("%" & search & "%")
    
    let whereClause = whereParts.join(" AND ")
    
    let countRow = db.getRow(SqlQuery("SELECT COUNT(*) FROM posts p WHERE " & whereClause), params)
    let total = parseInt(countRow[0])
    
    let rows = db.getAllRows(SqlQuery("""
      SELECT p.id, p.title, p.slug, p.status, p.created_at,
             u.id as author_id, u.name as author_name
      FROM posts p
      JOIN users u ON p.author_id = u.id
      WHERE """ & whereClause & """
      ORDER BY p.created_at DESC
      LIMIT ? OFFSET ?
    """), params & @[$limit, $offset])
    
    var posts = newJArray()
    for row in rows:
      posts.add(%*{
        "id": parseInt(row[0]),
        "title": row[1],
        "slug": row[2],
        "status": row[3],
        "createdAt": row[4],
        "author": {"id": parseInt(row[5]), "name": row[6]}
      })
    
    return successResp(posts, %*{
      "total": total,
      "page": page,
      "limit": limit,
      "totalPages": (total + limit - 1) div limit
    })
  
  get "/api/v1/posts/@slug":
    let slug = @"slug"
    
    let rows = db.getAllRows(sql"""
      SELECT p.id, p.title, p.slug, p.content, p.status, p.created_at, p.updated_at,
             u.id, u.name
      FROM posts p
      JOIN users u ON p.author_id = u.id
      WHERE p.slug = ?
    """, slug)
    
    if rows.len == 0:
      return errorResp("Post not found", Http404, "NOT_FOUND")
    
    let row = rows[0]
    
    # Get tags
    let tagRows = db.getAllRows(sql"""
      SELECT t.name FROM tags t
      JOIN post_tags pt ON pt.tag_id = t.id
      WHERE pt.post_id = ?
    """, row[0])
    
    var tags = newJArray()
    for tRow in tagRows:
      tags.add(%tRow[0])
    
    return successResp(%*{
      "id": parseInt(row[0]),
      "title": row[1],
      "slug": row[2],
      "content": row[3],
      "status": row[4],
      "createdAt": row[5],
      "updatedAt": row[6],
      "author": {"id": parseInt(row[7]), "name": row[8]},
      "tags": tags
    })
  
  post "/api/v1/posts":
    let currentUser = getCurrentUser(request)
    if currentUser.isNone:
      return errorResp("Authentication required", Http401, "UNAUTHORIZED")
    
    let body = try: parseJson(request.body)
               except: return errorResp("Invalid JSON")
    
    let title   = body{"title"}.getStr("")
    let content = body{"content"}.getStr("")
    let status  = body{"status"}.getStr("draft")
    
    if title.len < 5:
      return errorResp("Title must be at least 5 characters", Http422)
    if status notin ["draft", "published"]:
      return errorResp("Status must be 'draft' or 'published'", Http422)
    
    let slug = slugify(title)
    let authorId = currentUser.get()["id"].getInt()
    
    let postId = db.insertID(sql"""
      INSERT INTO posts (title, slug, content, author_id, status) 
      VALUES (?, ?, ?, ?, ?)
    """, title, slug, content, $authorId, status)
    
    # Handle tags
    let tagsNode = body{"tags"}
    if not tagsNode.isNil and tagsNode.kind == JArray:
      for tagNode in tagsNode:
        let tagName = tagNode.getStr()
        if tagName.len > 0:
          # Get or create tag
          var tagRow = db.getRow(sql"SELECT id FROM tags WHERE name = ?", tagName)
          var tagId: string
          if tagRow[0].len == 0:
            tagId = $db.insertID(sql"INSERT INTO tags (name) VALUES (?)", tagName)
          else:
            tagId = tagRow[0]
          
          db.exec(sql"INSERT OR IGNORE INTO post_tags (post_id, tag_id) VALUES (?, ?)",
                  $postId, tagId)
    
    return successResp(%*{"id": postId, "slug": slug, "message": "Post created"})
  
  # ── Health ───────────────────
  
  get "/api/v1/health":
    let userCount = db.getValue(sql"SELECT COUNT(*) FROM users")
    let postCount = db.getValue(sql"SELECT COUNT(*) FROM posts")
    
    return successResp(%*{
      "status": "healthy",
      "timestamp": now().format("yyyy-MM-dd'T'HH:mm:ss"),
      "stats": {
        "users": parseInt(userCount),
        "posts": parseInt(postCount)
      },
      "version": "1.0.0"
    })

# Run
let settings = newSettings(port = Port(8080), appName = "Blog API")
var jester = initJester(blogApi, settings = settings)
# jester.serve()
```

---

## 📝 สรุป Part 15

| Steps | หัวข้อ |
|-------|--------|
| 191 | REST API design principles |
| 192 | JWT authentication |
| 193 | Request validation framework |
| 194-210 | Complete Blog REST API |

---

**← [Part 14: Database](part_14_database.md) | [Part 16: WebSockets →](part_16_websockets.md)**
