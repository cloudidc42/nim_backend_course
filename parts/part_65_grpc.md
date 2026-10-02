# Part 65: gRPC & Protocol Buffers
## Steps 946-960: High-Performance RPC in Nim

---

## 🎯 เป้าหมายของ Part นี้

- Protocol Buffer encoding/decoding (wire format)
- gRPC service definitions
- Unary RPC
- Server streaming RPC
- Client streaming RPC
- Bidirectional streaming RPC
- Deadline/timeout propagation
- Interceptors (middleware)

---

## Step 946: Protocol Buffer Wire Format

```nim
import asyncdispatch, tables, strformat, times, sequtils, json, strutils, options, algorithm, math

# ============================
# Protobuf wire types
# ============================

type
  WireType = enum
    wtVarint = 0
    wt64Bit = 1
    wtLengthDelimited = 2
    wtStartGroup = 3
    wtEndGroup = 4
    wt32Bit = 5

  FieldType = enum
    ftInt32, ftInt64, ftUint32, ftUint64,
    ftSint32, ftSint64,
    ftBool, ftFixed64, ftSfixed64, ftDouble,
    ftString, ftBytes, ftMessage,
    ftFixed32, ftSfixed32, ftFloat

  ProtoField = object
    number: int
    type_: FieldType
    name: string

  ProtoMessage = object
    name: string
    fields: seq[ProtoField]

  ProtoValue = object
    case kind: FieldType
    of ftInt32: int32Val: int32
    of ftInt64, ftSint64, ftSfixed64: int64Val: int64
    of ftUint32, ftFixed32: uint32Val: uint32
    of ftUint64, ftFixed64: uint64Val: uint64
    of ftBool: boolVal: bool
    of ftDouble, ftFloat: floatVal: float64
    of ftString, ftBytes: strVal: string
    of ftMessage: msgVal: JsonNode
    of ftSint32, ftSfixed32: sint32Val: int32

# ============================
# Simple protobuf encoder
# ============================

proc encodeVarint(value: uint64): seq[byte] =
  var v = value
  while v > 0x7F:
    result.add(byte((v and 0x7F) or 0x80))
    v = v shr 7
  result.add(byte(v and 0x7F))

proc decodeVarint(data: seq[byte], offset: int): tuple[value: uint64, nextOffset: int] =
  var value = 0u64
  var shift = 0
  var i = offset
  while i < data.len:
    let b = data[i]
    value = value or (uint64(b and 0x7F) shl shift)
    inc i
    if (b and 0x80) == 0: break
    shift += 7
  return (value, i)

proc zigzagEncode(n: int64): uint64 =
  uint64((n shl 1) xor (n shr 63))

proc zigzagDecode(n: uint64): int64 =
  int64((n shr 1) xor (-(int64(n and 1))))

proc encodeTag(fieldNumber: int, wireType: WireType): seq[byte] =
  let tag = uint64(fieldNumber shl 3) or uint64(wireType)
  return encodeVarint(tag)

proc encodeInt32Field(fieldNumber: int, value: int32): seq[byte] =
  result = encodeTag(fieldNumber, wtVarint)
  result.add(encodeVarint(uint64(value)))

proc encodeStringField(fieldNumber: int, value: string): seq[byte] =
  result = encodeTag(fieldNumber, wtLengthDelimited)
  result.add(encodeVarint(uint64(value.len)))
  for c in value: result.add(byte(ord(c)))

proc encodeFloat64Field(fieldNumber: int, value: float64): seq[byte] =
  result = encodeTag(fieldNumber, wt64Bit)
  let bits = cast[uint64](value)
  for i in 0..<8:
    result.add(byte((bits shr (i * 8)) and 0xFF))

proc encodeBoolField(fieldNumber: int, value: bool): seq[byte] =
  result = encodeTag(fieldNumber, wtVarint)
  result.add(encodeVarint(if value: 1 else: 0))

# ============================
# gRPC message types
# ============================

type
  UserRequest = object
    userId: string

  UserResponse = object
    userId: string
    name: string
    email: string
    age: int
    active: bool

  CreateUserRequest = object
    name: string
    email: string
    age: int

  CreateUserResponse = object
    userId: string
    name: string
    email: string

  ListUsersRequest = object
    pageSize: int
    pageToken: string
    filter: string

  ListUsersResponse = object
    users: seq[UserResponse]
    nextPageToken: string
    totalCount: int

# Encode/decode UserResponse
proc encodeUserResponse(user: UserResponse): seq[byte] =
  result.add(encodeStringField(1, user.userId))
  result.add(encodeStringField(2, user.name))
  result.add(encodeStringField(3, user.email))
  result.add(encodeInt32Field(4, int32(user.age)))
  result.add(encodeBoolField(5, user.active))

proc encodeUserResponseJson(user: UserResponse): string =
  $(%*{
    "userId": user.userId,
    "name": user.name,
    "email": user.email,
    "age": user.age,
    "active": user.active
  })

# ============================
# gRPC status codes
# ============================

type
  GrpcStatus = enum
    grpcOk = 0
    grpcCancelled = 1
    grpcUnknown = 2
    grpcInvalidArgument = 3
    grpcDeadlineExceeded = 4
    grpcNotFound = 5
    grpcAlreadyExists = 6
    grpcPermissionDenied = 7
    grpcUnauthenticated = 16
    grpcResourceExhausted = 8
    grpcFailedPrecondition = 9
    grpcInternal = 13
    grpcUnavailable = 14

  GrpcError = object of CatchableError
    code: GrpcStatus

  GrpcResponse[T] = object
    status: GrpcStatus
    message: string
    data: Option[T]
    trailers: Table[string, string]

proc grpcOkResponse[T](data: T): GrpcResponse[T] =
  GrpcResponse[T](
    status: grpcOk,
    data: some(data),
    trailers: initTable[string, string]()
  )

proc grpcErrorResponse[T](code: GrpcStatus, msg: string): GrpcResponse[T] =
  GrpcResponse[T](
    status: code,
    message: msg,
    data: none(T),
    trailers: initTable[string, string]()
  )

# ============================
# Interceptors (middleware)
# ============================

type
  GrpcContext = object
    method_: string
    deadline: float   # epoch time
    metadata: Table[string, string]
    peer: string

  UnaryInterceptor[Req, Resp] = proc(ctx: GrpcContext, req: Req,
    handler: proc(ctx: GrpcContext, req: Req): Future[GrpcResponse[Resp]] {.async.}
  ): Future[GrpcResponse[Resp]] {.async.}

proc loggingInterceptor[Req, Resp](
    ctx: GrpcContext, req: Req,
    handler: proc(ctx: GrpcContext, req: Req): Future[GrpcResponse[Resp]] {.async.}
): Future[GrpcResponse[Resp]] {.async.} =
  let start = cpuTime()
  echo fmt"[gRPC] --> {ctx.method_} from {ctx.peer}"
  let resp = await handler(ctx, req)
  let elapsed = (cpuTime() - start) * 1000
  echo fmt"[gRPC] <-- {ctx.method_} status={resp.status} ({elapsed:.2f}ms)"
  return resp

proc deadlineInterceptor[Req, Resp](
    ctx: GrpcContext, req: Req,
    handler: proc(ctx: GrpcContext, req: Req): Future[GrpcResponse[Resp]] {.async.}
): Future[GrpcResponse[Resp]] {.async.} =
  if ctx.deadline > 0 and epochTime() > ctx.deadline:
    return grpcErrorResponse[Resp](grpcDeadlineExceeded, "Deadline exceeded")
  return await handler(ctx, req)

proc authInterceptor[Req, Resp](
    ctx: GrpcContext, req: Req,
    handler: proc(ctx: GrpcContext, req: Req): Future[GrpcResponse[Resp]] {.async.}
): Future[GrpcResponse[Resp]] {.async.} =
  let token = ctx.metadata.getOrDefault("authorization", "")
  if token.len == 0 or not token.startsWith("Bearer "):
    return grpcErrorResponse[Resp](grpcUnauthenticated, "Missing authorization token")
  return await handler(ctx, req)

# ============================
# Mock UserService gRPC server
# ============================

var mockUserDB: Table[string, UserResponse] = {
  "user_1": UserResponse(userId: "user_1", name: "Alice", email: "alice@example.com", age: 30, active: true),
  "user_2": UserResponse(userId: "user_2", name: "Bob", email: "bob@example.com", age: 25, active: true),
  "user_3": UserResponse(userId: "user_3", name: "Charlie", email: "charlie@example.com", age: 35, active: false),
}.toTable()

var userCounter = 3

# Unary RPC
proc getUser(ctx: GrpcContext, req: UserRequest): Future[GrpcResponse[UserResponse]] {.async.} =
  if req.userId notin mockUserDB:
    return grpcErrorResponse[UserResponse](grpcNotFound, fmt"User {req.userId} not found")
  return grpcOkResponse(mockUserDB[req.userId])

# Unary RPC
proc createUser(ctx: GrpcContext, req: CreateUserRequest): Future[GrpcResponse[CreateUserResponse]] {.async.} =
  if req.name.len == 0:
    return grpcErrorResponse[CreateUserResponse](grpcInvalidArgument, "Name is required")
  if "@" notin req.email:
    return grpcErrorResponse[CreateUserResponse](grpcInvalidArgument, "Invalid email")

  inc userCounter
  let userId = fmt"user_{userCounter}"
  mockUserDB[userId] = UserResponse(
    userId: userId, name: req.name, email: req.email, age: req.age, active: true)

  return grpcOkResponse(CreateUserResponse(
    userId: userId, name: req.name, email: req.email))

# Server streaming RPC
proc listUsers(ctx: GrpcContext, req: ListUsersRequest,
               onNext: proc(user: UserResponse) {.async.}
              ): Future[GrpcResponse[ListUsersResponse]] {.async.} =
  var users: seq[UserResponse]
  for _, user in mockUserDB:
    if req.filter.len == 0 or req.filter in user.name.toLowerAscii():
      users.add(user)

  # Simulate streaming
  var count = 0
  for user in users:
    if req.pageSize > 0 and count >= req.pageSize: break
    await onNext(user)
    inc count
    await sleepAsync(5)  # simulate async work

  return grpcOkResponse(ListUsersResponse(
    users: users[0..<count],
    totalCount: users.len
  ))

# ============================
# gRPC service definition
# ============================

type
  RpcMethod = object
    name: string
    isServerStreaming: bool
    isClientStreaming: bool

  GrpcService = object
    name: string
    package: string
    methods: seq[RpcMethod]

proc defineService(name, package: string): GrpcService =
  GrpcService(name: name, package: package, methods: @[])

proc addMethod(svc: var GrpcService, name: string,
               serverStreaming = false, clientStreaming = false) =
  svc.methods.add(RpcMethod(
    name: name,
    isServerStreaming: serverStreaming,
    isClientStreaming: clientStreaming
  ))

proc printServiceDefinition(svc: GrpcService) =
  echo fmt"service {svc.name} {{"
  for m in svc.methods:
    let reqStream = if m.isClientStreaming: "stream " else: ""
    let respStream = if m.isServerStreaming: "stream " else: ""
    let reqType = m.name & "Request"
    let respType = m.name & "Response"
    echo fmt"  rpc {m.name} ({reqStream}{reqType}) returns ({respStream}{respType});"
  echo "}"

# ============================
# Demo
# ============================

proc demo() {.async.} =
  echo "=== gRPC & Protocol Buffers Demo ==="

  # Protobuf encoding
  echo "\n--- Protobuf encoding ---"
  let user = UserResponse(
    userId: "user_1", name: "Alice", email: "alice@example.com",
    age: 30, active: true)
  let encoded = encodeUserResponse(user)
  echo fmt"Encoded bytes: {encoded.len}"
  echo fmt"JSON equivalent: {encodeUserResponseJson(user)}"

  # Varint encoding
  echo "\n--- Varint encoding ---"
  for val in [0u64, 1, 127, 128, 300, 65536]:
    let enc = encodeVarint(val)
    let (dec, _) = decodeVarint(enc, 0)
    echo fmt"  {val} -> {enc.len} bytes -> decoded: {dec}"

  # Service definition
  echo "\n--- Service definition (proto-style) ---"
  var userSvc = defineService("UserService", "api.v1")
  userSvc.addMethod("GetUser")
  userSvc.addMethod("CreateUser")
  userSvc.addMethod("UpdateUser")
  userSvc.addMethod("DeleteUser")
  userSvc.addMethod("ListUsers", serverStreaming = true)
  userSvc.addMethod("WatchUsers", serverStreaming = true, clientStreaming = true)
  printServiceDefinition(userSvc)

  # Unary RPC calls
  echo "\n--- Unary RPC ---"
  let ctx = GrpcContext(
    method_: "/api.v1.UserService/GetUser",
    deadline: epochTime() + 5.0,
    peer: "127.0.0.1:50051",
    metadata: {"authorization": "Bearer test_token_123"}.toTable()
  )

  # Test getUser
  let resp1 = await getUser(ctx, UserRequest(userId: "user_1"))
  echo fmt"GetUser user_1: status={resp1.status}"
  if resp1.data.isSome:
    echo fmt"  name={resp1.data.get().name} email={resp1.data.get().email}"

  let resp2 = await getUser(ctx, UserRequest(userId: "user_999"))
  echo fmt"GetUser user_999: status={resp2.status} msg={resp2.message}"

  # Test createUser
  let resp3 = await createUser(ctx, CreateUserRequest(
    name: "Diana", email: "diana@example.com", age: 28))
  echo fmt"CreateUser: status={resp3.status}"
  if resp3.data.isSome:
    echo fmt"  created userId={resp3.data.get().userId}"

  # Server streaming RPC
  echo "\n--- Server streaming RPC ---"
  var streamedUsers: seq[string]
  let streamResp = await listUsers(ctx,
    ListUsersRequest(pageSize: 10, filter: ""),
    proc(user: UserResponse) {.async.} =
      streamedUsers.add(user.name)
      echo fmt"  streamed: {user.name}"
  )
  echo fmt"ListUsers: got {streamedUsers.len} users"

  # Deadline exceeded
  echo "\n--- Deadline exceeded ---"
  let expiredCtx = GrpcContext(
    method_: "/api.v1.UserService/GetUser",
    deadline: epochTime() - 1.0,  # already expired
    peer: "127.0.0.1:50051",
    metadata: {"authorization": "Bearer test_token_123"}.toTable()
  )
  let deadlineResp = await deadlineInterceptor[UserRequest, UserResponse](
    expiredCtx,
    UserRequest(userId: "user_1"),
    getUser
  )
  echo fmt"Deadline exceeded: status={deadlineResp.status} msg={deadlineResp.message}"

waitFor demo()
```

---

## 📝 สรุป Part 65

| Steps | หัวข้อ |
|-------|--------|
| 946 | Protobuf wire format encoding/decoding |
| 947-955 | gRPC service definitions, unary/streaming RPC |
| 956-960 | Interceptors, deadline propagation, status codes |

---

**← [Part 64: Rate Limiting](part_64_rate_limiting.md) | [Part 66: Distributed Tracing →](part_66_tracing.md)**
