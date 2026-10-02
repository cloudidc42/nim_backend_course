# Part 21: Jester Web Framework - Backend Intermediate
## Steps 286-300: สร้าง Web App ด้วย Jester

---

## 🎯 เป้าหมายของ Part นี้

- ใช้ Jester framework อย่างเต็มรูปแบบ
- Routing patterns
- Middleware chains
- Static files
- Template rendering
- Session management
- Complete web application

---

## Step 286: Jester Installation และ Setup

```bash
# nimble install jester
# nimble.lock:
requires "jester >= 0.5.0"
```

```nim
# hello_jester.nim - Minimal Jester app
import jester

router helloRouter:
  get "/":
    resp "Hello, World!"
  
  get "/greet/@name":
    resp "Hello, " & @"name" & "!"

let settings = newSettings(port = Port(8080))
var jester = initJester(helloRouter, settings = settings)
jester.serve()
```

---

## Step 287: Routes และ Parameters

```nim
import jester, std/json, std/strformat, std/strutils

router apiRouter:
  
  # Static route
  get "/":
    resp """{"message": "Welcome to the API"}"""
  
  # URL parameters
  get "/users/@id":
    let id = @"id"
    if id.isDigit():
      resp Http200, fmt"""{"id": {id}, "name": "User #{id}"}""", "application/json"
    else:
      resp Http400, """{"error": "Invalid ID"}""", "application/json"
  
  # Query parameters
  get "/search":
    let query = request.params.getOrDefault("q", "")
    let page  = request.params.getOrDefault("page", "1")
    let limit = request.params.getOrDefault("limit", "10")
    
    resp Http200, fmt"""{{
  "query": "{query}",
  "page": {page},
  "limit": {limit},
  "results": []
}}""", "application/json"
  
  # POST with body
  post "/users":
    let body = try: parseJson(request.body)
               except: 
                 resp Http400, """{"error": "Invalid JSON"}""", "application/json"
                 return
    
    let name  = body{"name"}.getStr("")
    let email = body{"email"}.getStr("")
    
    if name.len < 2:
      resp Http422, """{"error": "Name too short"}""", "application/json"
      return
    
    resp Http201, fmt"""{{
  "id": 1,
  "name": "{name}",
  "email": "{email}",
  "created": true
}}""", "application/json"
  
  # PUT update
  put "/users/@id":
    let id = @"id"
    let body = try: parseJson(request.body)
               except:
                 resp Http400, """{"error": "Invalid JSON"}""", "application/json"
                 return
    
    resp Http200, fmt"""{"id": {id}, "updated": true}""", "application/json"
  
  # DELETE
  delete "/users/@id":
    let id = @"id"
    resp Http200, fmt"""{"id": {id}, "deleted": true}""", "application/json"
  
  # Multiple HTTP methods
  options "/users":
    response.headers.add("Allow", "GET, POST, OPTIONS")
    resp Http200, ""
  
  # Catch-all / 404
  error Http404:
    resp Http404, """{"error": "Route not found"}""", "application/json"
  
  error Http500:
    resp Http500, """{"error": "Internal server error"}""", "application/json"

let settings = newSettings(
  port = Port(8080),
  appName = "My API"
)

var jester = initJester(apiRouter, settings = settings)
jester.serve()
```

---

## Step 288: Headers, Cookies, Sessions

```nim
import jester, std/json, std/times, std/strutils

# Secret for signing cookies
const SESSION_SECRET = "your-secret-key-min-32-chars-here"

router webRouter:
  
  # Read request headers
  get "/headers":
    let contentType = request.headers.getOrDefault("content-type", "not set")
    let userAgent   = request.headers.getOrDefault("user-agent", "unknown")
    let authHeader  = request.headers.getOrDefault("authorization", "")
    
    resp Http200, fmt"""{{
  "content-type": "{contentType}",
  "user-agent": "{userAgent}",
  "has-auth": {($authHeader.len > 0).toLowerAscii()}
}}""", "application/json"
  
  # Set response headers
  get "/custom-headers":
    response.headers.add("X-Custom-Header", "MyValue")
    response.headers.add("X-Request-Id", "req_12345")
    response.headers.add("Cache-Control", "no-cache, no-store, must-revalidate")
    response.headers.add("X-Rate-Limit", "100")
    response.headers.add("X-Rate-Remaining", "95")
    
    resp Http200, """{"message": "Check response headers"}""", "application/json"
  
  # Set cookie
  get "/set-cookie":
    setCookie("session_id", "abc123xyz", daysForward(7))
    setCookie("user_pref", "dark_mode", daysForward(365))
    resp Http200, """{"message": "Cookie set"}"""
  
  # Read cookie
  get "/read-cookie":
    let sessionId = request.cookies.getOrDefault("session_id", "")
    let userPref  = request.cookies.getOrDefault("user_pref", "light")
    
    resp Http200, fmt"""{{
  "session_id": "{sessionId}",
  "theme": "{userPref}"
}}""", "application/json"
  
  # Delete cookie
  get "/delete-cookie":
    setCookie("session_id", "", daysForward(-1))  # expire immediately
    resp Http200, """{"message": "Cookie deleted"}"""
```

---

## Step 289: Middleware ใน Jester

```nim
import jester, std/json, std/tables, std/times, std/strformat, std/strutils

# Rate limiter state
var requestCounts = initTable[string, (int, float)]()

proc getClientIp(request: Request): string =
  request.headers.getOrDefault("x-forwarded-for", 
    request.headers.getOrDefault("x-real-ip", "127.0.0.1"))
    .split(",")[0].strip()

# Authentication check
proc requireAuth(request: Request): bool =
  let auth = request.headers.getOrDefault("authorization", "")
  return auth.startsWith("Bearer ") and auth.len > 10

# Rate limiting check (100 req/min per IP)
proc checkRateLimit(ip: string): bool =
  let now = epochTime()
  
  if ip in requestCounts:
    let (count, windowStart) = requestCounts[ip]
    
    if now - windowStart < 60.0:
      if count >= 100:
        return false
      requestCounts[ip] = (count + 1, windowStart)
    else:
      requestCounts[ip] = (1, now)  # reset window
  else:
    requestCounts[ip] = (1, now)
  
  return true

# CORS headers
proc addCorsHeaders(response: var Response) =
  response.headers.add("Access-Control-Allow-Origin", "*")
  response.headers.add("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE, OPTIONS")
  response.headers.add("Access-Control-Allow-Headers", "Content-Type, Authorization")

router appRouter:
  
  # Middleware-style before filter
  before:
    let ip = getClientIp(request)
    
    # Add CORS
    addCorsHeaders(response)
    
    # Rate limiting
    if not checkRateLimit(ip):
      halt Http429, """{"error": "Rate limit exceeded. Try again in a minute."}"""
    
    # Log request
    echo fmt"[{now().format(\"HH:mm:ss\")}] {request.reqMethod} {request.pathInfo} from {ip}"
  
  # OPTIONS for CORS preflight
  options "/@path":
    resp Http200, ""
  
  # Public routes
  get "/api/v1/health":
    resp Http200, """{"status": "ok"}""", "application/json"
  
  # Protected routes
  get "/api/v1/me":
    if not requireAuth(request):
      halt Http401, """{"error": "Authentication required"}"""
    
    resp Http200, """{"id": 1, "name": "Alice", "role": "admin"}""", "application/json"
  
  # After filter
  after:
    response.headers.add("X-Response-Time", $epochTime())
    response.headers.add("X-Powered-By", "Nim/Jester")
```

---

## Step 290: Static Files และ Templates

```nim
import jester, std/os, std/strutils, std/tables

# Serve static files
settings:
  port = Port(8080)
  staticDir = "public"  # serve files from ./public directory

router staticRouter:
  
  # Manual static file serving
  get "/static/@filename":
    let filename = @"filename"
    let filepath = "public" / filename
    
    if not fileExists(filepath):
      resp Http404, "File not found"
      return
    
    let ext = splitFile(filename).ext.toLower()
    let contentType = case ext
      of ".html": "text/html"
      of ".css":  "text/css"
      of ".js":   "application/javascript"
      of ".json": "application/json"
      of ".png":  "image/png"
      of ".jpg":  "image/jpeg"
      of ".svg":  "image/svg+xml"
      else: "application/octet-stream"
    
    sendFile(filepath)
  
  # Simple HTML template rendering
  get "/page/@name":
    let pageName = @"name"
    let templates = {
      "index": """
        <!DOCTYPE html>
        <html>
        <head><title>Home</title></head>
        <body>
          <h1>Welcome to Nim Web App!</h1>
          <p>This page is served by Jester + Nim</p>
        </body>
        </html>
      """,
      "about": """
        <!DOCTYPE html>
        <html>
        <head><title>About</title></head>
        <body>
          <h1>About Us</h1>
          <p>Built with ❤️ in Nim</p>
        </body>
        </html>
      """
    }.toTable()
    
    if pageName in templates:
      resp Http200, templates[pageName], "text/html"
    else:
      resp Http404, "<h1>Page not found</h1>", "text/html"
```

---

## Step 291-300: Complete Jester Application - Blog API

```nim
# blog_jester.nim - Complete Blog Application

import jester, std/asyncdispatch, std/json, std/strformat,
       std/tables, std/options, std/times, std/strutils, std/sequtils,
       db_connector/db_sqlite

# ==============================
# Database
# ==============================

let db = open("blog.db", "", "", "")

proc setupDb() =
  db.exec(sql"""
    CREATE TABLE IF NOT EXISTS articles (
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      title TEXT NOT NULL,
      slug TEXT UNIQUE NOT NULL,
      body TEXT NOT NULL,
      author TEXT NOT NULL,
      status TEXT DEFAULT 'draft',
      views INTEGER DEFAULT 0,
      created_at TEXT DEFAULT (datetime('now')),
      updated_at TEXT DEFAULT (datetime('now'))
    )
  """)
  
  # Seed data
  let count = db.getValue(sql"SELECT COUNT(*) FROM articles")
  if count == "0":
    let seedData = @[
      ("Getting Started with Nim", "getting-started-with-nim",
       "Nim is a great language for backend development.", "Alice"),
      ("Async Programming in Nim", "async-programming-in-nim",
       "Learn how to write async code with Nim's asyncdispatch.", "Bob"),
      ("Database Access in Nim", "database-access-in-nim",
       "Connect to SQLite and PostgreSQL from Nim.", "Alice"),
    ]
    
    for (title, slug, body, author) in seedData:
      db.exec(sql"""
        INSERT INTO articles (title, slug, body, author, status)
        VALUES (?, ?, ?, ?, 'published')
      """, title, slug, body, author)

setupDb()

# ==============================
# Helpers
# ==============================

proc slugify2(s: string): string =
  s.toLower().replace(" ", "-").replace("'", "").replace("\"", "")

proc jsonOk(data: JsonNode): string =
  $ %*{"success": true, "data": data}

proc jsonErr(msg: string): string =
  $ %*{"success": false, "error": msg}

# ==============================
# Routes
# ==============================

router blogRouter:
  
  before:
    response.headers.add("Content-Type", "application/json")
    response.headers.add("Access-Control-Allow-Origin", "*")
  
  # GET /articles - list with filtering
  get "/api/articles":
    let page   = try: parseInt(request.params.getOrDefault("page", "1")) except: 1
    let limit  = min(try: parseInt(request.params.getOrDefault("limit", "10")) except: 10, 50)
    let status = request.params.getOrDefault("status", "published")
    let author = request.params.getOrDefault("author", "")
    let offset = (page - 1) * limit
    
    var conditions = @["status = ?"]
    var params = @[status]
    
    if author.len > 0:
      conditions.add("author = ?")
      params.add(author)
    
    let whereStr = conditions.join(" AND ")
    
    let total = parseInt(db.getValue(
      SqlQuery("SELECT COUNT(*) FROM articles WHERE " & whereStr), params
    ))
    
    let rows = db.getAllRows(SqlQuery("""
      SELECT id, title, slug, author, status, views, created_at
      FROM articles WHERE """ & whereStr & """
      ORDER BY created_at DESC LIMIT ? OFFSET ?
    """), params & @[$limit, $offset])
    
    var articles = newJArray()
    for row in rows:
      articles.add(%*{
        "id": parseInt(row[0]),
        "title": row[1],
        "slug": row[2],
        "author": row[3],
        "status": row[4],
        "views": parseInt(row[5]),
        "createdAt": row[6]
      })
    
    resp Http200, $ %*{
      "success": true,
      "data": articles,
      "meta": {
        "total": total,
        "page": page,
        "limit": limit,
        "totalPages": (total + limit - 1) div limit
      }
    }
  
  # GET /articles/:slug
  get "/api/articles/@slug":
    let slug = @"slug"
    
    let rows = db.getAllRows(sql"""
      SELECT id, title, slug, body, author, status, views, created_at, updated_at
      FROM articles WHERE slug = ?
    """, slug)
    
    if rows.len == 0:
      resp Http404, jsonErr("Article not found")
      return
    
    let row = rows[0]
    
    # Increment view count
    db.exec(sql"UPDATE articles SET views = views + 1 WHERE slug = ?", slug)
    
    resp Http200, jsonOk(%*{
      "id": parseInt(row[0]),
      "title": row[1],
      "slug": row[2],
      "body": row[3],
      "author": row[4],
      "status": row[5],
      "views": parseInt(row[6]) + 1,
      "createdAt": row[7],
      "updatedAt": row[8]
    })
  
  # POST /articles
  post "/api/articles":
    let auth = request.headers.getOrDefault("authorization", "")
    if auth.len == 0:
      resp Http401, jsonErr("Authentication required")
      return
    
    let body = try: parseJson(request.body)
               except:
                 resp Http400, jsonErr("Invalid JSON")
                 return
    
    let title  = body{"title"}.getStr("")
    let text   = body{"body"}.getStr("")
    let status = body{"status"}.getStr("draft")
    
    if title.len < 5:
      resp Http422, jsonErr("Title must be at least 5 characters")
      return
    if text.len < 10:
      resp Http422, jsonErr("Body must be at least 10 characters")
      return
    if status notin ["draft", "published"]:
      resp Http422, jsonErr("Status must be 'draft' or 'published'")
      return
    
    let slug = slugify2(title)
    let author = "current_user"  # from JWT in production
    
    let id = db.insertID(sql"""
      INSERT INTO articles (title, slug, body, author, status)
      VALUES (?, ?, ?, ?, ?)
    """, title, slug, text, author, status)
    
    resp Http201, jsonOk(%*{
      "id": id, "slug": slug, "status": status,
      "message": "Article created"
    })
  
  # PATCH /articles/:slug
  patch "/api/articles/@slug":
    let auth = request.headers.getOrDefault("authorization", "")
    if auth.len == 0:
      resp Http401, jsonErr("Authentication required")
      return
    
    let slug = @"slug"
    let rows = db.getAllRows(sql"SELECT id FROM articles WHERE slug = ?", slug)
    if rows.len == 0:
      resp Http404, jsonErr("Article not found")
      return
    
    let id = rows[0][0]
    let body = try: parseJson(request.body)
               except:
                 resp Http400, jsonErr("Invalid JSON")
                 return
    
    var updates: seq[string] = @[]
    var params: seq[string] = @[]
    
    if body{"title"}.kind == JString:
      updates.add("title = ?")
      params.add(body["title"].getStr())
    
    if body{"body"}.kind == JString:
      updates.add("body = ?")
      params.add(body["body"].getStr())
    
    if body{"status"}.kind == JString:
      let newStatus = body["status"].getStr()
      if newStatus in ["draft", "published"]:
        updates.add("status = ?")
        params.add(newStatus)
    
    if updates.len == 0:
      resp Http400, jsonErr("No fields to update")
      return
    
    updates.add("updated_at = datetime('now')")
    params.add(id)
    
    db.exec(SqlQuery("UPDATE articles SET " & updates.join(", ") & " WHERE id = ?"), params)
    
    resp Http200, jsonOk(%*{"slug": slug, "message": "Article updated"})
  
  # DELETE /articles/:slug
  delete "/api/articles/@slug":
    let auth = request.headers.getOrDefault("authorization", "")
    if auth.len == 0:
      resp Http401, jsonErr("Authentication required")
      return
    
    let slug = @"slug"
    let rows = db.getAllRows(sql"SELECT id FROM articles WHERE slug = ?", slug)
    if rows.len == 0:
      resp Http404, jsonErr("Article not found")
      return
    
    db.exec(sql"DELETE FROM articles WHERE slug = ?", slug)
    resp Http200, jsonOk(%*{"message": "Article deleted"})
  
  # Stats
  get "/api/stats":
    let totalArticles   = db.getValue(sql"SELECT COUNT(*) FROM articles")
    let publishedCount  = db.getValue(sql"SELECT COUNT(*) FROM articles WHERE status = 'published'")
    let totalViews      = db.getValue(sql"SELECT SUM(views) FROM articles")
    
    let topRows = db.getAllRows(sql"""
      SELECT title, slug, views FROM articles 
      ORDER BY views DESC LIMIT 5
    """)
    
    var topArticles = newJArray()
    for row in topRows:
      topArticles.add(%*{"title": row[0], "slug": row[1], "views": parseInt(row[2])})
    
    resp Http200, jsonOk(%*{
      "total": parseInt(totalArticles),
      "published": parseInt(publishedCount),
      "totalViews": parseInt(if totalViews.len > 0: totalViews else: "0"),
      "topArticles": topArticles
    })
  
  # Health
  get "/health":
    resp Http200, $ %*{"status": "ok", "timestamp": $now()}
  
  error Http404:
    resp Http404, jsonErr("Route not found")
  
  error Http500:
    resp Http500, jsonErr("Internal server error")

# ==============================
# Start Server
# ==============================

let settings = newSettings(
  port = Port(8080),
  appName = "Blog API",
  bindAddr = "0.0.0.0"
)

echo """
╔══════════════════════════════════════╗
║          Blog API (Jester)           ║
╠══════════════════════════════════════╣
║  GET    /api/articles                ║
║  GET    /api/articles/:slug          ║
║  POST   /api/articles (auth)         ║
║  PATCH  /api/articles/:slug (auth)   ║
║  DELETE /api/articles/:slug (auth)   ║
║  GET    /api/stats                   ║
║  GET    /health                      ║
╚══════════════════════════════════════╝
"""

var jester = initJester(blogRouter, settings = settings)
jester.serve()
```

---

## 📝 สรุป Part 21

| Steps | หัวข้อ |
|-------|--------|
| 286 | Jester setup, hello world |
| 287 | Routes, URL params, query params |
| 288 | Headers, cookies |
| 289 | Middleware (rate limiting, CORS, auth) |
| 290 | Static files, templates |
| 291-300 | Complete Blog API with Jester |

---

**← [Part 20: Performance](part_20_performance.md) | [Part 22: Authentication →](part_22_authentication.md)**
