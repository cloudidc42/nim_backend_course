# Part 54: WebSocket Server
## Steps 781-795: Real-Time Bidirectional Communication

---

## 🎯 เป้าหมายของ Part นี้

- WebSocket handshake & framing
- Connection management
- Rooms / channels
- Broadcasting
- Heartbeat / ping-pong
- Message protocols (JSON-RPC style)

---

## Step 781: WebSocket Core

```nim
import asyncdispatch, asyncnet, tables, strformat, times, sequtils, json, strutils, hashes, base64

# ============================
# WebSocket frame types
# ============================

type
  WsOpcode = enum
    woContinuation = 0x0,
    woText = 0x1,
    woBinary = 0x2,
    woClose = 0x8,
    woPing = 0x9,
    woPong = 0xA

  WsFrame = object
    fin: bool
    opcode: WsOpcode
    masked: bool
    payload: string

  WsState = enum
    wsConnecting, wsOpen, wsClosing, wsClosed

  WsConnection = object
    id: string
    socket: AsyncSocket
    state: WsState
    userId: string
    rooms: seq[string]
    connectedAt: float
    lastPingAt: float
    lastPongAt: float
    metadata: Table[string, string]

  WsServer = object
    connections: Table[string, WsConnection]
    rooms: Table[string, seq[string]]   # room -> [connIds]
    counter: int
    pingIntervalMs: int
    pongTimeoutMs: int

var wsServer = WsServer(
  connections: initTable[string, WsConnection](),
  rooms: initTable[string, seq[string]](),
  counter: 0,
  pingIntervalMs: 30000,
  pongTimeoutMs: 5000
)

# ============================
# WebSocket handshake
# ============================

proc wsHandshakeKey(clientKey: string): string =
  ## Compute Sec-WebSocket-Accept header value
  let magic = "258EAFA5-E914-47DA-95CA-C5AB0DC85B11"
  let combined = clientKey & magic
  # In production, use SHA1 hash; here simplified
  encode(combined[0..min(combined.len-1, 20)])

proc parseHandshakeRequest(request: string): Table[string, string] =
  var headers: Table[string, string]
  for line in request.splitLines():
    let parts = line.split(": ", 1)
    if parts.len == 2:
      headers[parts[0].toLowerAscii()] = parts[1].strip()
  return headers

proc buildHandshakeResponse(wsKey: string): string =
  let acceptKey = wsHandshakeKey(wsKey)
  fmt"""HTTP/1.1 101 Switching Protocols\r
Upgrade: websocket\r
Connection: Upgrade\r
Sec-WebSocket-Accept: {acceptKey}\r
\r
"""

# ============================
# Frame encoding/decoding
# ============================

proc encodeFrame(opcode: WsOpcode, payload: string, fin = true): string =
  var frame = newString(0)

  # Byte 1: FIN + opcode
  let byte1 = (if fin: 0x80 else: 0x00) or int(opcode)
  frame.add(char(byte1))

  # Byte 2+: payload length
  let payloadLen = payload.len
  if payloadLen < 126:
    frame.add(char(payloadLen))
  elif payloadLen < 65536:
    frame.add(char(126))
    frame.add(char((payloadLen shr 8) and 0xFF))
    frame.add(char(payloadLen and 0xFF))
  else:
    frame.add(char(127))
    for shift in countdown(56, 0, 8):
      frame.add(char((payloadLen shr shift) and 0xFF))

  frame.add(payload)
  return frame

proc decodeFrameHeader(data: string): tuple[opcode: WsOpcode, payloadLen: int, masked: bool, headerLen: int] =
  if data.len < 2:
    return (woContinuation, 0, false, 0)

  let byte1 = ord(data[0])
  let byte2 = ord(data[1])
  let opcode = WsOpcode(byte1 and 0x0F)
  let masked = (byte2 and 0x80) != 0
  var payloadLen = byte2 and 0x7F
  var headerLen = 2

  if payloadLen == 126:
    if data.len < 4: return (woContinuation, 0, false, 0)
    payloadLen = (ord(data[2]) shl 8) or ord(data[3])
    headerLen = 4
  elif payloadLen == 127:
    if data.len < 10: return (woContinuation, 0, false, 0)
    payloadLen = 0
    for i in 2..9:
      payloadLen = (payloadLen shl 8) or ord(data[i])
    headerLen = 10

  if masked: headerLen += 4

  return (opcode, payloadLen, masked, headerLen)

proc unmaskPayload(data: string, maskStart, payloadStart, payloadLen: int): string =
  let mask = data[maskStart..<maskStart + 4]
  var result = newString(payloadLen)
  for i in 0..<payloadLen:
    result[i] = char(ord(data[payloadStart + i]) xor ord(mask[i mod 4]))
  return result

# ============================
# Connection management
# ============================

proc newConnId(): string =
  inc wsServer.counter
  fmt"ws_{wsServer.counter}_{int(epochTime() * 1000) mod 1_000_000}"

proc addConnection(socket: AsyncSocket, userId = ""): string =
  let id = newConnId()
  wsServer.connections[id] = WsConnection(
    id: id,
    socket: socket,
    state: wsOpen,
    userId: userId,
    rooms: @[],
    connectedAt: epochTime(),
    lastPingAt: epochTime(),
    lastPongAt: epochTime(),
    metadata: initTable[string, string]()
  )
  echo fmt"[WS] Connected: {id} (user={userId})"
  return id

proc removeConnection(connId: string) =
  if connId notin wsServer.connections: return
  let conn = wsServer.connections[connId]

  # Leave all rooms
  for room in conn.rooms:
    if room in wsServer.rooms:
      wsServer.rooms[room] = wsServer.rooms[room].filterIt(it != connId)

  wsServer.connections.del(connId)
  echo fmt"[WS] Disconnected: {connId}"

proc getConnection(connId: string): Option[WsConnection] =
  if connId in wsServer.connections:
    return some(wsServer.connections[connId])
  return none(WsConnection)

# ============================
# Room management
# ============================

proc joinRoom(connId, room: string) =
  if connId notin wsServer.connections: return
  if room notin wsServer.rooms:
    wsServer.rooms[room] = @[]

  if connId notin wsServer.rooms[room]:
    wsServer.rooms[room].add(connId)

  if room notin wsServer.connections[connId].rooms:
    wsServer.connections[connId].rooms.add(room)

  echo fmt"[WS] {connId} joined room: {room}"

proc leaveRoom(connId, room: string) =
  if room in wsServer.rooms:
    wsServer.rooms[room] = wsServer.rooms[room].filterIt(it != connId)
  if connId in wsServer.connections:
    wsServer.connections[connId].rooms = wsServer.connections[connId].rooms.filterIt(it != room)

proc getRoomMembers(room: string): seq[string] =
  wsServer.rooms.getOrDefault(room, @[])

# ============================
# Message sending
# ============================

proc sendText(connId, text: string): Future[void] {.async.} =
  if connId notin wsServer.connections: return
  let conn = wsServer.connections[connId]
  if conn.state != wsOpen: return

  let frame = encodeFrame(woText, text)
  try:
    await conn.socket.send(frame)
  except CatchableError as e:
    echo fmt"[WS] Send error for {connId}: {e.msg}"
    removeConnection(connId)

proc sendJson(connId: string, data: JsonNode): Future[void] {.async.} =
  await sendText(connId, $data)

proc broadcastToRoom(room, text: string, excludeConnId = ""): Future[void] {.async.} =
  let members = getRoomMembers(room)
  var futures: seq[Future[void]]
  for connId in members:
    if connId != excludeConnId:
      futures.add(sendText(connId, text))
  await all(futures)

proc broadcastToAll(text: string, excludeConnId = ""): Future[void] {.async.} =
  var futures: seq[Future[void]]
  for connId, _ in wsServer.connections:
    if connId != excludeConnId:
      futures.add(sendText(connId, text))
  await all(futures)

# ============================
# Message protocol (JSON-RPC style)
# ============================

type
  WsMessage = object
    id: string
    type_: string     # "request", "response", "event", "error"
    method_: string
    params: JsonNode
    result_: JsonNode
    error_: JsonNode

proc parseMessage(raw: string): Option[WsMessage] =
  try:
    let j = parseJson(raw)
    return some(WsMessage(
      id: j.getOrDefault("id", %"").getStr(),
      type_: j.getOrDefault("type", %"request").getStr(),
      method_: j.getOrDefault("method", %"").getStr(),
      params: j.getOrDefault("params", newJNull()),
      result_: j.getOrDefault("result", newJNull()),
      error_: j.getOrDefault("error", newJNull())
    ))
  except:
    return none(WsMessage)

proc buildResponse(id: string, result_: JsonNode): string =
  $(%*{"id": id, "type": "response", "result": result_})

proc buildEvent(event: string, data: JsonNode): string =
  $(%*{"type": "event", "event": event, "data": data})

proc buildError(id, code, message: string): string =
  $(%*{"id": id, "type": "error", "error": {"code": code, "message": message}})

# ============================
# Heartbeat
# ============================

proc sendPing(connId: string): Future[void] {.async.} =
  if connId notin wsServer.connections: return
  let frame = encodeFrame(woPing, "ping")
  try:
    await wsServer.connections[connId].socket.send(frame)
    wsServer.connections[connId].lastPingAt = epochTime()
  except:
    removeConnection(connId)

proc handlePong(connId: string) =
  if connId in wsServer.connections:
    wsServer.connections[connId].lastPongAt = epochTime()

proc checkDeadConnections() =
  let now = epochTime()
  var dead: seq[string]
  for connId, conn in wsServer.connections:
    let pingTimeout = now - conn.lastPingAt > float(wsServer.pingIntervalMs) / 1000 + 5.0
    if pingTimeout and conn.lastPongAt < conn.lastPingAt:
      dead.add(connId)
  for connId in dead:
    echo fmt"[WS] Dead connection: {connId}"
    removeConnection(connId)

# ============================
# Demo (simulated)
# ============================

proc demo() {.async.} =
  echo "=== WebSocket Server Demo ==="

  # Simulate connections using mock sockets
  let mockSocket1 = newAsyncSocket()
  let mockSocket2 = newAsyncSocket()
  let mockSocket3 = newAsyncSocket()

  let connId1 = addConnection(mockSocket1, userId = "user_1")
  let connId2 = addConnection(mockSocket2, userId = "user_2")
  let connId3 = addConnection(mockSocket3, userId = "user_3")

  echo fmt"\nActive connections: {wsServer.connections.len}"

  # Room management
  echo "\n--- Room management ---"
  joinRoom(connId1, "chat:general")
  joinRoom(connId2, "chat:general")
  joinRoom(connId3, "chat:general")
  joinRoom(connId1, "chat:private:1-2")
  joinRoom(connId2, "chat:private:1-2")

  echo fmt"general room: {getRoomMembers(\"chat:general\").len} members"
  echo fmt"private room: {getRoomMembers(\"chat:private:1-2\").len} members"

  # Message protocol
  echo "\n--- Message protocol ---"
  let chatMsg = buildEvent("chat.message", %*{
    "room": "chat:general",
    "from": "user_1",
    "text": "Hello everyone!"
  })
  echo fmt"Event JSON: {chatMsg}"

  let rpcReq = """{"id":"req_1","type":"request","method":"subscribe","params":{"room":"chat:general"}}"""
  let parsed = parseMessage(rpcReq)
  if parsed.isSome:
    let msg = parsed.get()
    echo fmt"Parsed: method={msg.method_} id={msg.id}"
    let resp = buildResponse(msg.id, %*{"subscribed": true, "room": "chat:general"})
    echo fmt"Response: {resp}"

  # Frame encoding
  echo "\n--- Frame encoding ---"
  let frame = encodeFrame(woText, "Hello WebSocket!")
  echo fmt"Frame bytes: {frame.len}"

  let (opcode, payloadLen, masked, headerLen) = decodeFrameHeader(frame)
  echo fmt"Decoded: opcode={opcode} payloadLen={payloadLen} headerLen={headerLen}"

  # Disconnect
  leaveRoom(connId2, "chat:general")
  echo fmt"\nAfter leave: general has {getRoomMembers(\"chat:general\").len} members"

  removeConnection(connId3)
  echo fmt"After disconnect: {wsServer.connections.len} connections"

waitFor demo()
```

---

## Step 782-795: Chat App & Presence

```nim
import asyncdispatch, tables, strformat, times, sequtils, json, strutils, algorithm, options

# ============================
# Presence system
# ============================

type
  PresenceStatus = enum
    psOnline, psAway, psBusy, psOffline

  UserPresence = object
    userId: string
    status: PresenceStatus
    statusMessage: string
    lastSeenAt: float
    connIds: seq[string]   # user can have multiple connections

  PresenceStore = object
    presences: Table[string, UserPresence]
    watchers: Table[string, seq[string]]   # userId -> [watcherUserIds]

var presence = PresenceStore(
  presences: initTable[string, UserPresence](),
  watchers: initTable[string, seq[string]]()
)

proc setOnline(userId, connId: string) =
  if userId notin presence.presences:
    presence.presences[userId] = UserPresence(userId: userId)

  var p = presence.presences[userId]
  p.status = psOnline
  p.lastSeenAt = epochTime()
  if connId notin p.connIds: p.connIds.add(connId)
  presence.presences[userId] = p
  echo fmt"[Presence] {userId} is online"

proc setOffline(userId, connId: string) =
  if userId notin presence.presences: return
  var p = presence.presences[userId]
  p.connIds = p.connIds.filterIt(it != connId)
  if p.connIds.len == 0:
    p.status = psOffline
    p.lastSeenAt = epochTime()
  presence.presences[userId] = p
  echo fmt"[Presence] {userId} is {p.status}"

proc setStatus(userId: string, status: PresenceStatus, message = "") =
  if userId notin presence.presences: return
  presence.presences[userId].status = status
  presence.presences[userId].statusMessage = message

proc isOnline(userId: string): bool =
  userId in presence.presences and
  presence.presences[userId].status != psOffline

proc getOnlineUsers(): seq[string] =
  var result: seq[string]
  for userId, p in presence.presences:
    if p.status != psOffline: result.add(userId)
  return result

# ============================
# Chat message store
# ============================

type
  ChatMessage = object
    id: string
    roomId: string
    fromUserId: string
    content: string
    contentType: string  # "text" | "image" | "file"
    timestamp: float
    editedAt: float
    deleted: bool
    reactions: Table[string, seq[string]]  # emoji -> [userId]
    replyTo: string  # message id

  ChatRoom = object
    id: string
    name: string
    type_: string   # "channel" | "dm" | "group"
    members: seq[string]
    createdAt: float
    lastMessageAt: float

var chatRooms: Table[string, ChatRoom] = initTable[string, ChatRoom]()
var chatMessages: Table[string, seq[ChatMessage]] = initTable[string, seq[ChatMessage]]()
var msgCounter = 0

proc createRoom(name, type_: string, members: seq[string]): string =
  let id = fmt"room_{chatRooms.len + 1}"
  chatRooms[id] = ChatRoom(
    id: id, name: name, type_: type_,
    members: members, createdAt: epochTime()
  )
  chatMessages[id] = @[]
  echo fmt"[Chat] Room created: {name} ({type_})"
  return id

proc sendMessage(roomId, fromUserId, content: string,
                 contentType = "text", replyTo = ""): ChatMessage =
  if roomId notin chatRooms:
    raise newException(ValueError, fmt"Room not found: {roomId}")

  inc msgCounter
  let msg = ChatMessage(
    id: fmt"msg_{msgCounter}",
    roomId: roomId,
    fromUserId: fromUserId,
    content: content,
    contentType: contentType,
    timestamp: epochTime(),
    deleted: false,
    reactions: initTable[string, seq[string]](),
    replyTo: replyTo
  )

  chatMessages[roomId].add(msg)
  chatRooms[roomId].lastMessageAt = msg.timestamp

  echo fmt"[Chat] [{roomId}] {fromUserId}: {content[0..min(content.len-1, 30)]}"
  return msg

proc editMessage(roomId, msgId, newContent: string) =
  if roomId notin chatMessages: return
  for i, msg in chatMessages[roomId]:
    if msg.id == msgId:
      chatMessages[roomId][i].content = newContent
      chatMessages[roomId][i].editedAt = epochTime()
      break

proc deleteMessage(roomId, msgId: string) =
  if roomId notin chatMessages: return
  for i, msg in chatMessages[roomId]:
    if msg.id == msgId:
      chatMessages[roomId][i].deleted = true
      break

proc addReaction(roomId, msgId, emoji, userId: string) =
  if roomId notin chatMessages: return
  for i, msg in chatMessages[roomId]:
    if msg.id == msgId:
      if emoji notin chatMessages[roomId][i].reactions:
        chatMessages[roomId][i].reactions[emoji] = @[]
      if userId notin chatMessages[roomId][i].reactions[emoji]:
        chatMessages[roomId][i].reactions[emoji].add(userId)
      break

proc getMessages(roomId: string, limit = 50, before = ""): seq[ChatMessage] =
  if roomId notin chatMessages: return @[]
  var msgs = chatMessages[roomId].filterIt(not it.deleted)

  if before.len > 0:
    var idx = msgs.len
    for i, m in msgs:
      if m.id == before: idx = i; break
    msgs = msgs[0..<min(idx, msgs.len)]

  let startIdx = max(0, msgs.len - limit)
  return msgs[startIdx..^1]

# ============================
# Typing indicators
# ============================

var typingUsers: Table[string, Table[string, float]] = initTable[string, Table[string, float]]()

proc startTyping(roomId, userId: string) =
  if roomId notin typingUsers:
    typingUsers[roomId] = initTable[string, float]()
  typingUsers[roomId][userId] = epochTime()

proc stopTyping(roomId, userId: string) =
  if roomId in typingUsers:
    typingUsers[roomId].del(userId)

proc getTypingUsers(roomId: string): seq[string] =
  if roomId notin typingUsers: return @[]
  let now = epochTime()
  var result: seq[string]
  for userId, ts in typingUsers[roomId]:
    if now - ts < 3.0:  # 3 second timeout
      result.add(userId)
  return result

# ============================
# Demo
# ============================

proc demo() =
  echo "=== Chat App Demo ==="

  # Set up presence
  setOnline("alice", "conn_1")
  setOnline("bob", "conn_2")
  setOnline("charlie", "conn_3")

  echo fmt"\nOnline users: {getOnlineUsers()}"

  # Create rooms
  let generalRoom = createRoom("general", "channel", @["alice", "bob", "charlie"])
  let dmRoom = createRoom("alice-bob-dm", "dm", @["alice", "bob"])

  echo "\n--- Messages ---"
  let m1 = sendMessage(generalRoom, "alice", "Hello everyone!")
  let m2 = sendMessage(generalRoom, "bob", "Hi Alice!")
  let m3 = sendMessage(generalRoom, "charlie", "Hey all, what's up?")
  let m4 = sendMessage(generalRoom, "alice", "Discussing Nim!", replyTo = m2.id)

  # Reactions
  addReaction(generalRoom, m1.id, "👍", "bob")
  addReaction(generalRoom, m1.id, "👍", "charlie")
  addReaction(generalRoom, m1.id, "❤️", "bob")

  # Get messages
  let msgs = getMessages(generalRoom)
  echo fmt"\nMessages in general ({msgs.len}):"
  for m in msgs:
    let replyStr = if m.replyTo.len > 0: fmt" (reply to {m.replyTo})" else: ""
    var reactions = ""
    for emoji, users in m.reactions:
      reactions &= fmt" {emoji}x{users.len}"
    echo fmt"  [{m.id}] {m.fromUserId}: {m.content}{replyStr}{reactions}"

  # Typing indicators
  echo "\n--- Typing indicators ---"
  startTyping(generalRoom, "alice")
  startTyping(generalRoom, "bob")
  echo fmt"Typing: {getTypingUsers(generalRoom)}"
  stopTyping(generalRoom, "alice")
  echo fmt"After alice stops: {getTypingUsers(generalRoom)}"

  # Presence change
  setStatus("bob", psBusy, "In a meeting")
  setOffline("charlie", "conn_3")
  echo fmt"\nOnline after changes: {getOnlineUsers()}"

demo()
```

---

## 📝 สรุป Part 54

| Steps | หัวข้อ |
|-------|--------|
| 781 | WebSocket framing, rooms, broadcasting, ping-pong |
| 782-795 | Chat app: presence, messages, reactions, typing indicators |

---

**← [Part 53: GraphQL](part_53_graphql.md) | [Part 55: Task Scheduler →](part_55_task_scheduler.md)**
