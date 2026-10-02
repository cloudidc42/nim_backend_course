# Part 13: Async/Await - การเขียนโปรแกรมแบบ Asynchronous
## Steps 161-175: Async Programming ใน Nim

---

## 🎯 เป้าหมายของ Part นี้

- เข้าใจ Event Loop
- ใช้ async/await syntax
- จัดการ concurrent requests
- Async HTTP client
- Async database operations

---

## Step 161: Async พื้นฐาน

```nim
import std/asyncdispatch, std/times, std/strformat

# Basic async procedure
proc fetchData(url: string): Future[string] {.async.} =
  echo fmt"Fetching {url}..."
  await sleepAsync(100)  # simulate network delay
  return "Data from " & url

proc main() {.async.} =
  # Sequential (slow)
  let t1 = now()
  let data1 = await fetchData("https://api1.example.com")
  let data2 = await fetchData("https://api2.example.com")
  echo fmt"Sequential: {(now()-t1).inMilliseconds}ms"
  
  # Concurrent (fast)
  let t2 = now()
  let (res1, res2) = await (
    fetchData("https://api1.example.com"),
    fetchData("https://api2.example.com")
  )
  echo fmt"Concurrent: {(now()-t2).inMilliseconds}ms"
  
  echo res1
  echo res2

waitFor main()

# async with Future
proc slowAdd(a, b: int): Future[int] {.async.} =
  await sleepAsync(10)
  return a + b

proc computeAll(): Future[int] {.async.} =
  # Run multiple computations concurrently
  var futures = newSeq[Future[int]]()
  
  for i in 1..5:
    futures.add(slowAdd(i, i * 10))
  
  var total = 0
  for f in futures:
    total += await f
  
  return total

echo waitFor computeAll()  # 165 (1+10 + 2+20 + ... + 5+50)
```

---

## Step 162: Async HTTP Client

```nim
import std/asynchttpclient, std/asyncdispatch, std/json, std/strformat

type
  ApiClient = object
    client: AsyncHttpClient
    baseUrl: string
    headers: seq[(string, string)]

proc newApiClient(baseUrl: string): ApiClient =
  var client = newAsyncHttpClient()
  client.headers = newHttpHeaders({
    "Content-Type": "application/json",
    "Accept": "application/json",
  })
  
  ApiClient(
    client: client,
    baseUrl: baseUrl,
    headers: @[]
  )

proc get(api: ApiClient, path: string): Future[JsonNode] {.async.} =
  let url = api.baseUrl & path
  let response = await api.client.get(url)
  let body = await response.body
  return parseJson(body)

proc post(api: ApiClient, path: string, data: JsonNode): Future[JsonNode] {.async.} =
  let url = api.baseUrl & path
  let response = await api.client.post(url, body = $data)
  let body = await response.body
  return parseJson(body)

# Example usage with real API
proc fetchGithubUser(username: string): Future[JsonNode] {.async.} =
  var client = newAsyncHttpClient()
  client.headers = newHttpHeaders({"User-Agent": "NimApp/1.0"})
  defer: client.close()
  
  let response = await client.get("https://api.github.com/users/" & username)
  let body = await response.body
  return parseJson(body)

proc main() {.async.} =
  # Fetch multiple users concurrently
  let usernames = @["nim-lang", "torvalds", "defunkt"]
  
  var futures: seq[Future[JsonNode]] = @[]
  for username in usernames:
    futures.add(fetchGithubUser(username))
  
  for i, f in futures:
    try:
      let user = await f
      echo fmt"{usernames[i]}: {user{\"name\"}.getStr(\"N/A\")} ({user{\"public_repos\"}.getInt()} repos)"
    except:
      echo fmt"{usernames[i]}: failed"

# waitFor main()
```

---

## Step 163: Async TCP Server

```nim
import std/asyncnet, std/asyncdispatch, std/strformat, std/strutils

type
  Connection = object
    socket: AsyncSocket
    id: int
    address: string

var connections: seq[Connection] = @[]
var nextId = 1

proc handleClient(socket: AsyncSocket, address: string) {.async.} =
  let conn = Connection(socket: socket, id: nextId, address: address)
  inc nextId
  connections.add(conn)
  
  echo fmt"Client #{conn.id} connected from {address}"
  
  try:
    # Send welcome message
    await socket.send("Welcome! You are client #" & $conn.id & "\n")
    
    while true:
      let line = await socket.recvLine()
      if line.len == 0:
        break
      
      echo fmt"Client #{conn.id}: {line}"
      
      # Echo back to client
      await socket.send("Echo: " & line & "\n")
      
      if line.toLower() == "quit":
        await socket.send("Goodbye!\n")
        break
  
  except:
    echo fmt"Client #{conn.id} disconnected: {getCurrentExceptionMsg()}"
  
  finally:
    socket.close()
    echo fmt"Client #{conn.id} disconnected"

proc startServer(port: int = 8080) {.async.} =
  var server = newAsyncSocket()
  server.setSockOpt(OptReuseAddr, true)
  server.bindAddr(Port(port))
  server.listen()
  
  echo fmt"TCP Server listening on port {port}"
  
  while true:
    let (client, address) = await server.acceptAddr()
    asyncCheck handleClient(client, address)

# waitFor startServer(8888)
```

---

## Step 164: Async with Jester (Web Framework)

```nim
# backend_server.nim - Full async web server with Jester

import jester, std/asyncdispatch, std/json, std/strformat,
       std/tables, std/options, std/times

# ==============================
# Data Models
# ==============================

type
  TodoId = distinct int
  
  Todo = object
    id: TodoId
    title: string
    completed: bool
    createdAt: string

  TodoStore = ref object
    todos: Table[int, Todo]
    nextId: int

var store = TodoStore(todos: initTable[int, Todo](), nextId: 1)

proc `$`(id: TodoId): string = $int(id)

proc toJson(t: Todo): JsonNode =
  %*{
    "id": int(t.id),
    "title": t.title,
    "completed": t.completed,
    "createdAt": t.createdAt
  }

# ==============================
# Route Handlers
# ==============================

router app:
  
  # List all todos
  get "/todos":
    var todos = newJArray()
    for todo in store.todos.values:
      todos.add(todo.toJson())
    
    resp Http200, $(%*{"data": todos, "count": store.todos.len}),
         "application/json"
  
  # Get todo by ID
  get "/todos/@id":
    let id = try: parseInt(@"id") except: -1
    
    if id < 0 or id notin store.todos:
      resp Http404, $(%*{"error": "Todo not found"}), "application/json"
      return
    
    resp Http200, $store.todos[id].toJson(), "application/json"
  
  # Create todo
  post "/todos":
    let body = try: parseJson(request.body) except:
      resp Http400, $(%*{"error": "Invalid JSON"}), "application/json"
      return
    
    let title = body{"title"}.getStr("")
    if title.len == 0:
      resp Http400, $(%*{"error": "Title is required"}), "application/json"
      return
    
    let todo = Todo(
      id: TodoId(store.nextId),
      title: title,
      completed: false,
      createdAt: now().format("yyyy-MM-dd HH:mm:ss")
    )
    
    store.todos[store.nextId] = todo
    inc store.nextId
    
    resp Http201, $todo.toJson(), "application/json"
  
  # Update todo
  put "/todos/@id":
    let id = try: parseInt(@"id") except: -1
    
    if id < 0 or id notin store.todos:
      resp Http404, $(%*{"error": "Todo not found"}), "application/json"
      return
    
    let body = try: parseJson(request.body) except:
      resp Http400, $(%*{"error": "Invalid JSON"}), "application/json"
      return
    
    var todo = store.todos[id]
    if body{"title"}.kind != JNull:
      todo.title = body["title"].getStr()
    if body{"completed"}.kind != JNull:
      todo.completed = body["completed"].getBool()
    
    store.todos[id] = todo
    resp Http200, $todo.toJson(), "application/json"
  
  # Delete todo
  delete "/todos/@id":
    let id = try: parseInt(@"id") except: -1
    
    if id < 0 or id notin store.todos:
      resp Http404, $(%*{"error": "Todo not found"}), "application/json"
      return
    
    store.todos.del(id)
    resp Http200, $(%*{"message": "Todo deleted"}), "application/json"

# Run server
# port 8080
# runForever()
```

---

## Step 165-175: Full Async API Server

```nim
# async_api.nim - Complete async REST API

import std/asyncdispatch, std/asyncnet, std/json, std/strformat,
       std/strutils, std/tables, std/times, std/options, std/sequtils

# ==============================
# Mini HTTP Server (from scratch)
# ==============================

type
  HttpRequest = object
    httpMethod: string
    path: string
    version: string
    headers: Table[string, string]
    body: string
    queryParams: Table[string, string]

  HttpResponse = object
    statusCode: int
    headers: Table[string, string]
    body: string

  RouteHandler2 = proc(req: HttpRequest): Future[HttpResponse] {.async.}

  Router2 = object
    routes: seq[(string, string, RouteHandler2)]

proc parseRequest(data: string): HttpRequest =
  result.headers = initTable[string, string]()
  result.queryParams = initTable[string, string]()
  
  let lines = data.split("\r\n")
  if lines.len == 0: return
  
  let parts = lines[0].split(" ")
  if parts.len >= 3:
    result.httpMethod = parts[0]
    let pathWithQuery = parts[1]
    result.version = parts[2]
    
    let qIdx = pathWithQuery.find('?')
    if qIdx >= 0:
      result.path = pathWithQuery[0..<qIdx]
      let query = pathWithQuery[qIdx+1..^1]
      for param in query.split('&'):
        let kv = param.split('=', 1)
        if kv.len == 2:
          result.queryParams[kv[0]] = kv[1]
    else:
      result.path = pathWithQuery
  
  var bodyStart = -1
  for i in 1..<lines.len:
    if lines[i].len == 0:
      bodyStart = i + 1
      break
    let kv = lines[i].split(": ", 1)
    if kv.len == 2:
      result.headers[kv[0].toLower()] = kv[1]
  
  if bodyStart > 0 and bodyStart < lines.len:
    result.body = lines[bodyStart..^1].join("\r\n")

proc buildResponse(resp: HttpResponse): string =
  let statusText = case resp.statusCode
    of 200: "OK"
    of 201: "Created"
    of 400: "Bad Request"
    of 401: "Unauthorized"
    of 403: "Forbidden"
    of 404: "Not Found"
    of 409: "Conflict"
    of 500: "Internal Server Error"
    else: "Unknown"
  
  result = fmt"HTTP/1.1 {resp.statusCode} {statusText}\r\n"
  result &= "Server: NimServer/1.0\r\n"
  result &= fmt"Date: {now().format(\"ddd, dd MMM yyyy HH:mm:ss\")} GMT\r\n"
  result &= fmt"Content-Length: {resp.body.len}\r\n"
  
  for key, val in resp.headers:
    result &= fmt"{key}: {val}\r\n"
  
  result &= "\r\n"
  result &= resp.body

proc jsonResponse(data: JsonNode, status: int = 200): HttpResponse =
  HttpResponse(
    statusCode: status,
    headers: {"Content-Type": "application/json"}.toTable(),
    body: data.pretty()
  )

proc errorResponse(msg: string, status: int = 400): HttpResponse =
  jsonResponse(%*{"error": msg}, status)

# ==============================
# Application Data
# ==============================

type
  Product = object
    id: int
    name: string
    price: float
    stock: int
    category: string
    createdAt: string

var products: Table[int, Product] = initTable[int, Product]()
var nextProductId = 1

# Seed data
proc seedProducts() =
  let seed = @[
    ("Laptop Pro 15\"", 45000.0, 10, "Electronics"),
    ("Wireless Mouse", 1200.0, 50, "Electronics"),
    ("Mechanical Keyboard", 3500.0, 25, "Electronics"),
    ("USB-C Hub 7-Port", 1800.0, 30, "Electronics"),
    ("Notebook A5 Pack", 150.0, 100, "Stationery"),
    ("Premium Pen Set", 250.0, 75, "Stationery"),
    ("Standing Desk Mat", 2200.0, 15, "Office"),
    ("Monitor Stand", 1500.0, 20, "Office"),
  ]
  
  for (name, price, stock, cat) in seed:
    let id = nextProductId
    inc nextProductId
    products[id] = Product(
      id: id, name: name, price: price,
      stock: stock, category: cat,
      createdAt: now().format("yyyy-MM-dd")
    )

# ==============================
# API Handlers
# ==============================

proc handleListProducts(req: HttpRequest): Future[HttpResponse] {.async.} =
  var filtered = products.values.toSeq()
  
  # Filter by category
  let category = req.queryParams.getOrDefault("category", "")
  if category.len > 0:
    filtered = filtered.filterIt(it.category == category)
  
  # Filter by search
  let search = req.queryParams.getOrDefault("q", "").toLower()
  if search.len > 0:
    filtered = filtered.filterIt(search in it.name.toLower())
  
  # Sort
  let sortBy = req.queryParams.getOrDefault("sort", "id")
  case sortBy
  of "price": filtered.sort(proc(a, b: Product): int =
    cmp(a.price, b.price))
  of "name": filtered.sort(proc(a, b: Product): int =
    cmp(a.name, b.name))
  else: filtered.sort(proc(a, b: Product): int = a.id - b.id)
  
  # Pagination
  let page = try: parseInt(req.queryParams.getOrDefault("page", "1")) except: 1
  let limit = try: parseInt(req.queryParams.getOrDefault("limit", "10")) except: 10
  let total = filtered.len
  let start = (page - 1) * limit
  let paged = if start < filtered.len:
    filtered[start..<min(start + limit, filtered.len)]
  else:
    @[]
  
  var items = newJArray()
  for p in paged:
    items.add(%*{
      "id": p.id, "name": p.name, "price": p.price,
      "stock": p.stock, "category": p.category
    })
  
  return jsonResponse(%*{
    "data": items,
    "pagination": {
      "total": total, "page": page, "limit": limit,
      "totalPages": (total + limit - 1) div limit
    }
  })

proc handleGetProduct(req: HttpRequest, id: int): Future[HttpResponse] {.async.} =
  if id notin products:
    return errorResponse("Product not found", 404)
  
  let p = products[id]
  return jsonResponse(%*{
    "id": p.id, "name": p.name, "price": p.price,
    "stock": p.stock, "category": p.category, "createdAt": p.createdAt
  })

proc handleCreateProduct(req: HttpRequest): Future[HttpResponse] {.async.} =
  let body = try: parseJson(req.body)
  except: return errorResponse("Invalid JSON body")
  
  let name = body{"name"}.getStr("")
  let price = body{"price"}.getFloat(-1)
  let stock = body{"stock"}.getInt(-1)
  let category = body{"category"}.getStr("")
  
  if name.len == 0: return errorResponse("name is required")
  if price < 0: return errorResponse("price must be non-negative")
  if stock < 0: return errorResponse("stock must be non-negative")
  if category.len == 0: return errorResponse("category is required")
  
  let id = nextProductId
  inc nextProductId
  
  products[id] = Product(
    id: id, name: name, price: price, stock: stock,
    category: category, createdAt: now().format("yyyy-MM-dd")
  )
  
  return jsonResponse(%*{"id": id, "message": "Product created"}, 201)

proc handleUpdateStock(req: HttpRequest, id: int): Future[HttpResponse] {.async.} =
  if id notin products:
    return errorResponse("Product not found", 404)
  
  let body = try: parseJson(req.body)
  except: return errorResponse("Invalid JSON body")
  
  let newStock = body{"stock"}.getInt(-1)
  if newStock < 0:
    return errorResponse("stock must be non-negative")
  
  products[id].stock = newStock
  
  return jsonResponse(%*{
    "id": id,
    "stock": newStock,
    "message": "Stock updated"
  })

# ==============================
# Request Router
# ==============================

proc dispatch(req: HttpRequest): Future[HttpResponse] {.async.} =
  let path = req.path
  let parts = path.strip(chars = {'/'}).split('/')
  
  # GET /products
  if req.httpMethod == "GET" and path == "/products":
    return await handleListProducts(req)
  
  # GET /products/:id
  if req.httpMethod == "GET" and parts.len == 2 and parts[0] == "products":
    let id = try: parseInt(parts[1]) except: -1
    if id > 0:
      return await handleGetProduct(req, id)
  
  # POST /products
  if req.httpMethod == "POST" and path == "/products":
    return await handleCreateProduct(req)
  
  # PATCH /products/:id/stock
  if req.httpMethod == "PATCH" and parts.len == 3 and
     parts[0] == "products" and parts[2] == "stock":
    let id = try: parseInt(parts[1]) except: -1
    if id > 0:
      return await handleUpdateStock(req, id)
  
  # GET /health
  if req.httpMethod == "GET" and path == "/health":
    return jsonResponse(%*{
      "status": "healthy",
      "timestamp": now().format("yyyy-MM-dd HH:mm:ss"),
      "products": products.len
    })
  
  return errorResponse("Route not found", 404)

# ==============================
# Server
# ==============================

proc handleConnection(client: AsyncSocket) {.async.} =
  try:
    var buf = ""
    
    # Read request (simplified - production would handle chunks)
    while true:
      let chunk = await client.recv(4096)
      if chunk.len == 0: break
      buf &= chunk
      if "\r\n\r\n" in buf: break
    
    if buf.len > 0:
      let req = parseRequest(buf)
      let resp = await dispatch(req)
      await client.send(buildResponse(resp))
  
  except:
    echo "Connection error: " & getCurrentExceptionMsg()
  
  finally:
    client.close()

proc runServer(port: int = 8080) {.async.} =
  seedProducts()
  
  var server = newAsyncSocket()
  server.setSockOpt(OptReuseAddr, true)
  server.bindAddr(Port(port))
  server.listen()
  
  echo fmt"🚀 Product API running on http://localhost:{port}"
  echo "  GET    /health"
  echo "  GET    /products?category=Electronics&sort=price&page=1&limit=5"
  echo "  GET    /products/:id"
  echo "  POST   /products"
  echo "  PATCH  /products/:id/stock"
  
  while true:
    let client = await server.accept()
    asyncCheck handleConnection(client)

# Uncomment to run:
# waitFor runServer(8080)
```

---

## 📝 สรุป Part 13

| Steps | หัวข้อ |
|-------|--------|
| 161 | Async basics, Future, waitFor |
| 162 | Async HTTP client |
| 163 | Async TCP server |
| 164 | Jester framework |
| 165-175 | Full async Product API |

---

**← [Part 12: Generics](part_12_generics.md) | [Part 14: Database →](part_14_database.md)**
