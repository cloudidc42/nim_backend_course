# Part 40: RPC Framework
## Steps 571-585: gRPC-style RPC in Nim

---

## 🎯 เป้าหมายของ Part นี้

- Custom RPC protocol over TCP
- Service definition with macros
- Request/response serialization (JSON)
- Middleware chain for RPC
- Client stub generation
- Streaming RPC calls

---

## Step 571: RPC Protocol Design

```nim
import asyncdispatch, asyncnet, json, strformat, tables, strutils, sequtils

# ============================
# RPC Protocol (JSON over TCP)
# ============================
# Frame format:
#   [4-byte length: uint32 big-endian][JSON body]
# JSON body:
#   Request:  {"id": "uuid", "method": "Service.Method", "params": {...}}
#   Response: {"id": "uuid", "result": {...}, "error": null}
#   Error:    {"id": "uuid", "result": null, "error": {"code": 404, "msg": "..."}}

type
  RpcRequest = object
    id: string
    meth: string    # "UserService.GetUser"
    params: JsonNode

  RpcResponse = object
    id: string
    result: JsonNode
    error: JsonNode

  RpcError = object of CatchableError
    code: int

proc encodeFrame(data: string): string =
  var frame = newString(4 + data.len)
  let length = uint32(data.len)
  frame[0] = char((length shr 24) and 0xFF)
  frame[1] = char((length shr 16) and 0xFF)
  frame[2] = char((length shr  8) and 0xFF)
  frame[3] = char(length and 0xFF)
  frame[4..^1] = data
  return frame

proc decodeFrameLength(header: string): int =
  if header.len < 4: return -1
  result = (int(header[0]) shl 24) or
           (int(header[1]) shl 16) or
           (int(header[2]) shl 8) or
           int(header[3])

proc requestToJson(req: RpcRequest): string =
  $(%*{"id": req.id, "method": req.meth, "params": req.params})

proc responseToJson(resp: RpcResponse): string =
  $(%*{"id": resp.id, "result": resp.result, "error": resp.error})

proc parseRequest(data: string): RpcRequest =
  let j = parseJson(data)
  RpcRequest(
    id: j["id"].getStr(),
    meth: j["method"].getStr(),
    params: j["params"]
  )

proc parseResponse(data: string): RpcResponse =
  let j = parseJson(data)
  RpcResponse(
    id: j["id"].getStr(),
    result: if j.hasKey("result"): j["result"] else: newJNull(),
    error: if j.hasKey("error"): j["error"] else: newJNull()
  )

# ============================
# Request ID generator
# ============================

var reqCounter = 0
proc nextReqId(): string =
  inc reqCounter
  fmt"req_{reqCounter}_{int(epochTime() * 1000) mod 100000}"

echo "RPC protocol types defined"
echo "Frame encoding: 4-byte length prefix + JSON body"
```

---

## Step 572: RPC Server

```nim
import asyncdispatch, asyncnet, json, tables, strformat, strutils

# ============================
# Handler types
# ============================

type
  RpcContext = object
    clientAddr: string
    headers: Table[string, string]
    metadata: Table[string, string]

  RpcHandler = proc(ctx: RpcContext, params: JsonNode): Future[JsonNode] {.async.}

  RpcServer = object
    host: string
    port: int
    handlers: Table[string, RpcHandler]
    middlewares: seq[proc(ctx: var RpcContext, meth: string)]

var server = RpcServer(
  host: "0.0.0.0",
  port: 9090,
  handlers: initTable[string, RpcHandler](),
  middlewares: @[]
)

proc register(s: var RpcServer, service, meth: string, handler: RpcHandler) =
  let key = fmt"{service}.{meth}"
  s.handlers[key] = handler
  echo fmt"[RPC] Registered: {key}"

proc handleRequest(s: RpcServer, ctx: RpcContext, req: JsonNode): Future[JsonNode] {.async.} =
  let meth = req["method"].getStr()
  let reqId = req["id"].getStr()
  
  if meth notin s.handlers:
    return %*{
      "id": reqId,
      "result": newJNull(),
      "error": %*{"code": 404, "message": fmt"Method not found: {meth}"}
    }
  
  try:
    let params = if req.hasKey("params"): req["params"] else: newJObject()
    let handler = s.handlers[meth]
    let result = await handler(ctx, params)
    return %*{"id": reqId, "result": result, "error": newJNull()}
  except:
    let errMsg = getCurrentExceptionMsg()
    echo fmt"[RPC] Handler error for {meth}: {errMsg}"
    return %*{
      "id": reqId,
      "result": newJNull(),
      "error": %*{"code": 500, "message": errMsg}
    }

proc handleClient(s: RpcServer, client: AsyncSocket, addr: string) {.async.} =
  echo fmt"[RPC] Client connected: {addr}"
  let ctx = RpcContext(clientAddr: addr,
                       headers: initTable[string, string](),
                       metadata: initTable[string, string]())
  
  try:
    while true:
      # Read 4-byte length header
      let header = await client.recv(4)
      if header.len < 4: break
      
      let length = (int(header[0]) shl 24) or (int(header[1]) shl 16) or
                   (int(header[2]) shl 8) or int(header[3])
      
      if length <= 0 or length > 10_000_000:  # max 10MB
        echo fmt"[RPC] Invalid frame length: {length}"
        break
      
      # Read body
      let body = await client.recv(length)
      if body.len < length: break
      
      let req = parseJson(body)
      echo fmt"[RPC] Received: {req[\"method\"].getStr()}"
      
      let resp = await s.handleRequest(ctx, req)
      
      # Send response
      let respStr = $resp
      var frame = newString(4 + respStr.len)
      let ln = uint32(respStr.len)
      frame[0] = char((ln shr 24) and 0xFF)
      frame[1] = char((ln shr 16) and 0xFF)
      frame[2] = char((ln shr  8) and 0xFF)
      frame[3] = char(ln and 0xFF)
      frame[4..^1] = respStr
      
      await client.send(frame)
  except:
    echo fmt"[RPC] Client error: {getCurrentExceptionMsg()}"
  finally:
    client.close()
    echo fmt"[RPC] Client disconnected: {addr}"

# ============================
# Register handlers
# ============================

type
  UserServiceImpl = object
    users: Table[int, JsonNode]

var userService = UserServiceImpl(users: initTable[int, JsonNode]())

# Populate mock data
userService.users[1] = %*{"id": 1, "name": "Alice", "email": "alice@example.com"}
userService.users[2] = %*{"id": 2, "name": "Bob", "email": "bob@example.com"}

proc handleGetUser(ctx: RpcContext, params: JsonNode): Future[JsonNode] {.async.} =
  let userId = params["userId"].getInt()
  if userId in userService.users:
    return userService.users[userId]
  raise newException(ValueError, fmt"User not found: {userId}")

proc handleListUsers(ctx: RpcContext, params: JsonNode): Future[JsonNode] {.async.} =
  var users = newJArray()
  for _, u in userService.users:
    users.add(u)
  return %*{"users": users, "total": users.len}

proc handleCreateUser(ctx: RpcContext, params: JsonNode): Future[JsonNode] {.async.} =
  let newId = userService.users.len + 1
  let user = %*{
    "id": newId,
    "name": params["name"].getStr(),
    "email": params["email"].getStr()
  }
  userService.users[newId] = user
  return user

server.register("UserService", "GetUser", handleGetUser)
server.register("UserService", "ListUsers", handleListUsers)
server.register("UserService", "CreateUser", handleCreateUser)

# ============================
# Demo (without running server)
# ============================

proc demo() {.async.} =
  echo "=== RPC Server Demo ==="
  echo "Registered methods:"
  for meth in server.handlers.keys:
    echo fmt"  - {meth}"
  
  let ctx = RpcContext(clientAddr: "127.0.0.1:12345",
                       headers: initTable[string, string](),
                       metadata: initTable[string, string]())
  
  # Test GetUser
  echo "\n--- GetUser(1) ---"
  var req1 = %*{"id": "req_1", "method": "UserService.GetUser", "params": {"userId": 1}}
  let resp1 = await server.handleRequest(ctx, req1)
  echo $resp1
  
  # Test ListUsers
  echo "\n--- ListUsers ---"
  var req2 = %*{"id": "req_2", "method": "UserService.ListUsers", "params": {}}
  let resp2 = await server.handleRequest(ctx, req2)
  echo $resp2
  
  # Test invalid method
  echo "\n--- Unknown method ---"
  var req3 = %*{"id": "req_3", "method": "FakeService.DoThing", "params": {}}
  let resp3 = await server.handleRequest(ctx, req3)
  echo $resp3

waitFor demo()
```

---

## Step 573-585: Complete RPC Client + Service Definition

```nim
# rpc_framework.nim - Complete RPC framework with code generation patterns

import asyncdispatch, asyncnet, json, tables, strformat, strutils, options

# ============================
# Service definition DSL
# ============================
# In Nim we can use templates/macros to define services

template defineService(name: untyped, body: untyped) =
  type name = object
  body

template rpcMethod(svcName: string, methName: string, 
                   paramType, returnType: typedesc) =
  echo fmt"Defined: {svcName}.{methName}({$paramType}) -> {$returnType}"

# ============================
# Typed RPC client
# ============================

type
  RpcClientConfig = object
    host: string
    port: int
    timeout: int  # ms
    maxRetries: int

  RpcClient = object
    config: RpcClientConfig
    socket: AsyncSocket
    pending: Table[string, Future[JsonNode]]
    reqCounter: int
    connected: bool

proc newRpcClient(host: string, port: int): RpcClient =
  RpcClient(
    config: RpcClientConfig(host: host, port: port, timeout: 5000, maxRetries: 3),
    pending: initTable[string, Future[JsonNode]](),
    reqCounter: 0,
    connected: false
  )

proc callRaw(client: var RpcClient, meth: string, params: JsonNode): Future[JsonNode] {.async.} =
  inc client.reqCounter
  let reqId = fmt"req_{client.reqCounter}"
  
  let reqBody = $(%*{"id": reqId, "method": meth, "params": params})
  
  # Mock response for demo (without actual network)
  echo fmt"[RpcClient] Calling: {meth}({params})"
  
  # Simulate network round-trip
  await sleepAsync(1)
  
  # Mock responses
  case meth
  of "UserService.GetUser":
    let userId = params["userId"].getInt()
    if userId == 1:
      return %*{"id": 1, "name": "Alice", "email": "alice@example.com"}
    else:
      raise newException(ValueError, fmt"User {userId} not found")
  
  of "UserService.ListUsers":
    return %*{
      "users": [
        %*{"id": 1, "name": "Alice"},
        %*{"id": 2, "name": "Bob"}
      ],
      "total": 2
    }
  
  of "ProductService.GetProduct":
    return %*{"id": params["productId"].getInt(), "name": "Widget", "price": 9.99}
  
  else:
    raise newException(ValueError, fmt"Unknown method: {meth}")

# ============================
# Typed client stubs
# ============================

type
  GetUserParams = object
    userId: int

  CreateUserParams = object
    name: string
    email: string
    role: string

  User = object
    id: int
    name: string
    email: string

  ListUsersResult = object
    users: seq[User]
    total: int

proc toJson(p: GetUserParams): JsonNode =
  %*{"userId": p.userId}

proc toJson(p: CreateUserParams): JsonNode =
  %*{"name": p.name, "email": p.email, "role": p.role}

proc fromJson[T](j: JsonNode): T =
  when T is User:
    T(id: j["id"].getInt(), name: j["name"].getStr(),
      email: j.getOrDefault("email", %"").getStr())
  elif T is ListUsersResult:
    var users: seq[User]
    for u in j["users"]:
      users.add(User(id: u["id"].getInt(), name: u["name"].getStr()))
    T(users: users, total: j["total"].getInt())
  else:
    T()

# Type-safe client stub for UserService
type
  UserServiceClient = object
    client: RpcClient

proc newUserServiceClient(host: string, port: int): UserServiceClient =
  UserServiceClient(client: newRpcClient(host, port))

proc getUser(svc: var UserServiceClient, userId: int): Future[User] {.async.} =
  let params = GetUserParams(userId: userId)
  let result = await svc.client.callRaw("UserService.GetUser", params.toJson())
  return fromJson[User](result)

proc listUsers(svc: var UserServiceClient): Future[ListUsersResult] {.async.} =
  let result = await svc.client.callRaw("UserService.ListUsers", newJObject())
  return fromJson[ListUsersResult](result)

proc createUser(svc: var UserServiceClient, name, email, role: string): Future[User] {.async.} =
  let params = CreateUserParams(name: name, email: email, role: role)
  let result = await svc.client.callRaw("UserService.CreateUser", params.toJson())
  return fromJson[User](result)

# ============================
# Streaming RPC (channel-based)
# ============================

type
  StreamResult[T] = object
    value: T
    done: bool
    error: string

  RpcStream[T] = object
    items: seq[T]
    closed: bool
    pos: int

proc next[T](stream: var RpcStream[T]): Option[T] =
  if stream.pos >= stream.items.len:
    return none(T)
  let item = stream.items[stream.pos]
  inc stream.pos
  return some(item)

proc hasNext[T](stream: RpcStream[T]): bool =
  stream.pos < stream.items.len

# Server-side streaming: server sends multiple responses
proc streamUsers(client: var RpcClient): Future[RpcStream[User]] {.async.} =
  echo "[RpcClient] Starting user stream..."
  await sleepAsync(1)
  
  # In real implementation, this would read from a stream connection
  return RpcStream[User](
    items: @[
      User(id: 1, name: "Alice", email: "alice@example.com"),
      User(id: 2, name: "Bob", email: "bob@example.com"),
      User(id: 3, name: "Charlie", email: "charlie@example.com"),
    ],
    closed: false,
    pos: 0
  )

# ============================
# RPC Middleware
# ============================

type
  RpcMiddlewareFn = proc(meth: string, params: JsonNode,
                          next: proc(p: JsonNode): Future[JsonNode] {.async.}
                         ): Future[JsonNode] {.async.}

proc authMiddleware(meth: string, params: JsonNode,
                    next: proc(p: JsonNode): Future[JsonNode] {.async.}
                   ): Future[JsonNode] {.async.} =
  echo fmt"[Auth] Checking method: {meth}"
  # In real implementation: validate JWT from metadata
  return await next(params)

proc loggingMiddleware(meth: string, params: JsonNode,
                       next: proc(p: JsonNode): Future[JsonNode] {.async.}
                      ): Future[JsonNode] {.async.} =
  let start = epochTime()
  let result = await next(params)
  let elapsed = int((epochTime() - start) * 1000)
  echo fmt"[RPC] {meth} completed in {elapsed}ms"
  return result

proc rateLimitMiddleware(meth: string, params: JsonNode,
                         next: proc(p: JsonNode): Future[JsonNode] {.async.}
                        ): Future[JsonNode] {.async.} =
  # In real implementation: check rate limits
  return await next(params)

# ============================
# Demo
# ============================

proc demo() {.async.} =
  echo "=== RPC Framework Demo ==="
  
  var svc = newUserServiceClient("localhost", 9090)
  
  echo "\n1. GetUser(1)"
  let user = await svc.getUser(1)
  echo fmt"   User: {user.name} <{user.email}>"
  
  echo "\n2. ListUsers"
  let list = await svc.listUsers()
  echo fmt"   Total: {list.total}"
  for u in list.users:
    echo fmt"   - {u.name}"
  
  echo "\n3. Streaming users"
  var stream = await svc.client.streamUsers()
  echo "   Stream:"
  while stream.hasNext():
    let u = stream.next().get()
    echo fmt"   - {u.name} ({u.email})"
  
  echo "\n4. Error handling"
  try:
    let notFound = await svc.getUser(999)
    discard notFound
  except ValueError as e:
    echo fmt"   Error caught: {e.msg}"
  
  echo "\nRPC framework demo complete!"

waitFor demo()
```

---

## 📝 สรุป Part 40

| Steps | หัวข้อ |
|-------|--------|
| 571 | RPC protocol design (framing, JSON) |
| 572 | RPC server with handler registration |
| 573-585 | Typed client stubs, streaming RPC, middleware |

---

**← [Part 39: Message Queues](part_39_message_queues.md) | [Part 41: Rate Limiting →](part_41_rate_limiting.md)**
