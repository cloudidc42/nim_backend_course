# Part 16: WebSockets ด้วย Nim

## Steps 211-225

WebSockets เป็นโปรโตคอลที่ช่วยให้เกิดการสื่อสารแบบ two-way (bidirectional) แบบ real-time ระหว่าง client และ server โดยไม่ต้องส่ง HTTP request ซ้ำๆ ในบทนี้เราจะเรียนรู้การสร้าง WebSocket server ด้วย Nim ตั้งแต่พื้นฐานจนถึง live notification system

---

## Step 211: WebSocket คืออะไร และทำไมต้องใช้

WebSocket เป็นโปรโตคอลที่ทำงานบน TCP ซึ่งให้ full-duplex communication channel โดยมีข้อดีหลักๆ คือ:
- Low latency: ไม่ต้องสร้าง connection ใหม่ทุกครั้ง
- Real-time: ส่งข้อมูลได้ทันทีในทั้งสองทาง
- Efficient: overhead ต่ำกว่า HTTP polling

ก่อนอื่น ให้ติดตั้ง library ที่จำเป็น:

```nim
# nimble install ws
# หรือเพิ่มใน .nimble file:
# requires "ws >= 0.5.0"
```

โครงสร้างพื้นฐานของ WebSocket server:

```nim
# file: ws_basic.nim
import asyncdispatch, asynchttpserver, ws, json, strformat

proc handleWebSocket(req: Request) {.async.} =
  var socket = await newWebSocket(req)
  
  try:
    while socket.readyState == Open:
      let packet = await socket.receiveStrPacket()
      echo &"Received: {packet}"
      
      # Echo กลับไปให้ client
      await socket.send(packet)
  except WebSocketError as e:
    echo &"WebSocket error: {e.msg}"
  finally:
    socket.close()

proc main() {.async.} =
  var server = newAsyncHttpServer()
  
  proc cb(req: Request) {.async.} =
    if req.url.path == "/ws":
      await handleWebSocket(req)
    else:
      await req.respond(Http200, "WebSocket Server Running")
  
  echo "Server listening on port 8080..."
  await server.serve(Port(8080), cb)

waitFor main()
```

---

## Step 212: การติดตั้งและ Setup

ตั้งค่า project ให้พร้อมใช้งาน WebSocket:

```nim
# file: websocket_chat.nimble
# Package
version = "0.1.0"
author = "Your Name"
description = "Real-time WebSocket Chat Server"
license = "MIT"
srcDir = "src"
bin = @["server"]

# Dependencies
requires "nim >= 2.0.0"
requires "ws >= 0.5.0"
```

สร้างโครงสร้าง directory:

```bash
mkdir -p websocket_chat/src
cd websocket_chat
nimble init
```

---

## Step 213: WebSocket Handshake และ Connection

เข้าใจวิธีที่ WebSocket upgrade จาก HTTP:

```nim
# file: src/connection_demo.nim
import asyncdispatch, asynchttpserver, ws, json, strformat, times, tables

type
  ClientInfo = object
    id: string
    connectedAt: DateTime
    socket: WebSocket

var clients = initTable[string, ClientInfo]()
var clientCount = 0

proc generateClientId(): string =
  inc clientCount
  return &"client_{clientCount}_{epochTime().int}"

proc handleConnection(req: Request) {.async.} =
  # Upgrade HTTP connection เป็น WebSocket
  var ws = await newWebSocket(req)
  let clientId = generateClientId()
  
  clients[clientId] = ClientInfo(
    id: clientId,
    connectedAt: now(),
    socket: ws
  )
  
  echo &"[{now()}] New connection: {clientId}"
  echo &"Total clients: {clients.len}"
  
  # ส่ง welcome message
  let welcomeMsg = %*{
    "type": "welcome",
    "clientId": clientId,
    "message": "Connected to WebSocket server",
    "serverTime": $now()
  }
  await ws.send($welcomeMsg)
  
  try:
    while ws.readyState == Open:
      let data = await ws.receiveStrPacket()
      
      if data.len == 0:
        continue
      
      echo &"[{clientId}] Received: {data}"
      
      # Parse JSON message
      try:
        let msg = parseJson(data)
        let msgType = msg["type"].getStr()
        
        case msgType
        of "ping":
          let pong = %*{"type": "pong", "timestamp": epochTime()}
          await ws.send($pong)
        of "echo":
          let echo_msg = %*{
            "type": "echo",
            "original": msg["data"],
            "echoed_at": $now()
          }
          await ws.send($echo_msg)
        else:
          let unknown = %*{
            "type": "error",
            "message": &"Unknown message type: {msgType}"
          }
          await ws.send($unknown)
      except JsonParsingError:
        let errMsg = %*{
          "type": "error",
          "message": "Invalid JSON format"
        }
        await ws.send($errMsg)
  
  except WebSocketError as e:
    echo &"[{clientId}] WebSocket error: {e.msg}"
  finally:
    clients.del(clientId)
    ws.close()
    echo &"[{now()}] Disconnected: {clientId}"
    echo &"Total clients: {clients.len}"

proc main() {.async.} =
  var server = newAsyncHttpServer()
  
  proc cb(req: Request) {.async.} =
    if req.url.path == "/ws":
      await handleConnection(req)
    elif req.url.path == "/status":
      let status = %*{
        "status": "running",
        "connections": clients.len,
        "uptime": "ok"
      }
      await req.respond(Http200, $status, 
        newHttpHeaders([("Content-Type", "application/json")]))
    else:
      await req.respond(Http404, "Not Found")
  
  echo "WebSocket server starting on port 8080..."
  await server.serve(Port(8080), cb)

waitFor main()
```

---

## Step 214: Real-time Chat Server - ส่วนหลัก

สร้าง chat server ที่รองรับหลาย client พร้อมกัน:

```nim
# file: src/chat_server.nim
import asyncdispatch, asynchttpserver, ws, json, strformat
import times, tables, sets, strutils, random, os

type
  MessageType = enum
    mtJoin = "join"
    mtLeave = "leave"
    mtMessage = "message"
    mtSystem = "system"
    mtError = "error"
    mtUserList = "userList"

  User = ref object
    id: string
    username: string
    socket: WebSocket
    joinedAt: DateTime
    messageCount: int

  ChatMessage = object
    id: string
    msgType: MessageType
    fromUser: string
    content: string
    timestamp: DateTime

var
  users = initTable[string, User]()
  messageHistory = newSeq[ChatMessage]()
  messageCounter = 0

proc generateId(prefix: string): string =
  randomize()
  return &"{prefix}_{rand(999999):06d}"

proc createMessage(msgType: MessageType, fromUser, content: string): ChatMessage =
  inc messageCounter
  return ChatMessage(
    id: &"msg_{messageCounter}",
    msgType: msgType,
    fromUser: fromUser,
    content: content,
    timestamp: now()
  )

proc messageToJson(msg: ChatMessage): JsonNode =
  %*{
    "id": msg.id,
    "type": $msg.msgType,
    "from": msg.fromUser,
    "content": msg.content,
    "timestamp": msg.timestamp.format("yyyy-MM-dd HH:mm:ss")
  }

proc getUserList(): JsonNode =
  var userList = newJArray()
  for id, user in users:
    userList.add(%*{
      "id": user.id,
      "username": user.username,
      "joinedAt": user.joinedAt.format("HH:mm:ss")
    })
  return userList

proc broadcastToAll(message: JsonNode, excludeId = "") {.async.} =
  var toRemove: seq[string] = @[]
  
  for id, user in users:
    if id == excludeId:
      continue
    try:
      if user.socket.readyState == Open:
        await user.socket.send($message)
      else:
        toRemove.add(id)
    except WebSocketError:
      toRemove.add(id)
  
  for id in toRemove:
    users.del(id)

proc handleChatClient(req: Request) {.async.} =
  var socket = await newWebSocket(req)
  let userId = generateId("user")
  
  # รอรับ username จาก client
  var username = ""
  
  try:
    # รับ join message แรก
    let firstPacket = await socket.receiveStrPacket()
    let joinData = parseJson(firstPacket)
    
    if joinData["type"].getStr() != "join":
      await socket.send($(%*{"type": "error", "message": "First message must be join"}))
      socket.close()
      return
    
    username = joinData["username"].getStr().strip()
    
    if username.len == 0 or username.len > 20:
      await socket.send($(%*{"type": "error", "message": "Invalid username (1-20 chars)"}))
      socket.close()
      return
    
    # ตรวจสอบ username ซ้ำ
    for id, user in users:
      if user.username == username:
        await socket.send($(%*{"type": "error", "message": "Username already taken"}))
        socket.close()
        return
    
    # เพิ่ม user เข้า table
    users[userId] = User(
      id: userId,
      username: username,
      socket: socket,
      joinedAt: now(),
      messageCount: 0
    )
    
    echo &"[JOIN] {username} ({userId})"
    
    # ส่ง welcome พร้อม history
    let welcomeMsg = %*{
      "type": "welcome",
      "userId": userId,
      "username": username,
      "users": getUserList(),
      "recentMessages": block:
        var arr = newJArray()
        let start = max(0, messageHistory.len - 20)
        for i in start..<messageHistory.len:
          arr.add(messageToJson(messageHistory[i]))
        arr
    }
    await socket.send($welcomeMsg)
    
    # แจ้ง user อื่นว่ามีคนใหม่เข้ามา
    let joinMsg = createMessage(mtJoin, username, &"{username} has joined the chat")
    messageHistory.add(joinMsg)
    await broadcastToAll(messageToJson(joinMsg), excludeId = userId)
    
    # Main message loop
    while socket.readyState == Open:
      let packet = await socket.receiveStrPacket()
      
      if packet.len == 0:
        continue
      
      try:
        let data = parseJson(packet)
        let msgType = data["type"].getStr()
        
        case msgType
        of "message":
          let content = data["content"].getStr().strip()
          
          if content.len == 0:
            continue
          
          if content.len > 500:
            await socket.send($(%*{
              "type": "error",
              "message": "Message too long (max 500 chars)"
            }))
            continue
          
          inc users[userId].messageCount
          
          let chatMsg = createMessage(mtMessage, username, content)
          messageHistory.add(chatMsg)
          
          # Broadcast ไปทุก user รวมถึง sender
          let msgJson = messageToJson(chatMsg)
          await broadcastToAll(msgJson)
          
          echo &"[MSG] {username}: {content}"
        
        of "ping":
          await socket.send($(%*{
            "type": "pong",
            "timestamp": epochTime()
          }))
        
        of "getUserList":
          await socket.send($(%*{
            "type": "userList",
            "users": getUserList()
          }))
        
        else:
          await socket.send($(%*{
            "type": "error",
            "message": &"Unknown message type: {msgType}"
          }))
      
      except JsonParsingError:
        await socket.send($(%*{
          "type": "error",
          "message": "Invalid JSON"
        }))
  
  except WebSocketError as e:
    echo &"[ERROR] {username} ({userId}): {e.msg}"
  finally:
    if userId in users:
      users.del(userId)
    
    if username.len > 0:
      let leaveMsg = createMessage(mtLeave, username, &"{username} has left the chat")
      messageHistory.add(leaveMsg)
      asyncCheck broadcastToAll(messageToJson(leaveMsg))
      echo &"[LEAVE] {username} ({userId})"

proc main() {.async.} =
  var server = newAsyncHttpServer()
  
  proc cb(req: Request) {.async.} =
    case req.url.path
    of "/ws":
      await handleChatClient(req)
    of "/api/users":
      let resp = %*{
        "count": users.len,
        "users": getUserList()
      }
      await req.respond(Http200, $resp,
        newHttpHeaders([("Content-Type", "application/json")]))
    of "/api/messages":
      var msgs = newJArray()
      let start = max(0, messageHistory.len - 50)
      for i in start..<messageHistory.len:
        msgs.add(messageToJson(messageHistory[i]))
      await req.respond(Http200, $(%*{"messages": msgs}),
        newHttpHeaders([("Content-Type", "application/json")]))
    else:
      await req.respond(Http404, "Not Found")
  
  echo "Chat server running on port 8080"
  echo "WebSocket endpoint: ws://localhost:8080/ws"
  await server.serve(Port(8080), cb)

waitFor main()
```

---

## Step 215: Rooms และ Channel System

เพิ่มระบบ rooms เพื่อแยก conversation:

```nim
# file: src/room_system.nim
import asyncdispatch, asynchttpserver, ws, json, strformat
import times, tables, sets, strutils, sequtils

type
  RoomType = enum
    rtPublic = "public"
    rtPrivate = "private"

  Room = ref object
    id: string
    name: string
    roomType: RoomType
    members: HashSet[string]  # user IDs
    createdBy: string
    createdAt: DateTime
    description: string

  RoomUser = ref object
    id: string
    username: string
    socket: WebSocket
    rooms: HashSet[string]  # room IDs ที่ user อยู่

var
  rooms = initTable[string, Room]()
  roomUsers = initTable[string, RoomUser]()
  roomCounter = 0

proc createDefaultRooms() =
  rooms["general"] = Room(
    id: "general",
    name: "General",
    roomType: rtPublic,
    members: initHashSet[string](),
    createdBy: "system",
    createdAt: now(),
    description: "General discussion"
  )
  
  rooms["tech"] = Room(
    id: "tech",
    name: "Technology",
    roomType: rtPublic,
    members: initHashSet[string](),
    createdBy: "system",
    createdAt: now(),
    description: "Tech discussions"
  )
  
  rooms["random"] = Room(
    id: "random",
    name: "Random",
    roomType: rtPublic,
    members: initHashSet[string](),
    createdBy: "system",
    createdAt: now(),
    description: "Anything goes"
  )

proc getRoomList(publicOnly = true): JsonNode =
  var arr = newJArray()
  for id, room in rooms:
    if publicOnly and room.roomType != rtPublic:
      continue
    arr.add(%*{
      "id": room.id,
      "name": room.name,
      "type": $room.roomType,
      "memberCount": room.members.len,
      "description": room.description
    })
  return arr

proc broadcastToRoom(roomId: string, message: JsonNode, excludeUserId = "") {.async.} =
  if roomId notin rooms:
    return
  
  let room = rooms[roomId]
  var toRemove: seq[string] = @[]
  
  for userId in room.members:
    if userId == excludeUserId:
      continue
    if userId in roomUsers:
      let user = roomUsers[userId]
      try:
        if user.socket.readyState == Open:
          await user.socket.send($message)
        else:
          toRemove.add(userId)
      except WebSocketError:
        toRemove.add(userId)
  
  for userId in toRemove:
    room.members.excl(userId)

proc joinRoom(userId, roomId: string) {.async.} =
  if userId notin roomUsers or roomId notin rooms:
    return
  
  let user = roomUsers[userId]
  let room = rooms[roomId]
  
  room.members.incl(userId)
  user.rooms.incl(roomId)
  
  # แจ้งคนในห้อง
  let joinMsg = %*{
    "type": "roomEvent",
    "event": "joined",
    "roomId": roomId,
    "roomName": room.name,
    "username": user.username,
    "memberCount": room.members.len,
    "timestamp": $now()
  }
  await broadcastToRoom(roomId, joinMsg)
  
  # แจ้ง user เองว่าเข้าห้องสำเร็จ
  let ackMsg = %*{
    "type": "joinedRoom",
    "roomId": roomId,
    "roomName": room.name,
    "description": room.description,
    "memberCount": room.members.len
  }
  if user.socket.readyState == Open:
    await user.socket.send($ackMsg)

proc leaveRoom(userId, roomId: string) {.async.} =
  if userId notin roomUsers or roomId notin rooms:
    return
  
  let user = roomUsers[userId]
  let room = rooms[roomId]
  
  room.members.excl(userId)
  user.rooms.excl(roomId)
  
  let leaveMsg = %*{
    "type": "roomEvent",
    "event": "left",
    "roomId": roomId,
    "username": user.username,
    "memberCount": room.members.len,
    "timestamp": $now()
  }
  await broadcastToRoom(roomId, leaveMsg)

proc sendRoomMessage(userId, roomId, content: string) {.async.} =
  if userId notin roomUsers or roomId notin rooms:
    return
  
  let user = roomUsers[userId]
  
  if roomId notin user.rooms:
    let errMsg = %*{"type": "error", "message": "You are not in this room"}
    if user.socket.readyState == Open:
      await user.socket.send($errMsg)
    return
  
  let msg = %*{
    "type": "roomMessage",
    "roomId": roomId,
    "from": user.username,
    "content": content,
    "timestamp": $now()
  }
  
  await broadcastToRoom(roomId, msg)

proc handleRoomClient(req: Request) {.async.} =
  var socket = await newWebSocket(req)
  var userId = ""
  var username = ""
  
  try:
    # Initial handshake
    let firstPacket = await socket.receiveStrPacket()
    let joinData = parseJson(firstPacket)
    
    username = joinData["username"].getStr().strip()
    userId = &"u_{username}_{rand(9999):04d}"
    
    roomUsers[userId] = RoomUser(
      id: userId,
      username: username,
      socket: socket,
      rooms: initHashSet[string]()
    )
    
    # ส่ง welcome
    let welcome = %*{
      "type": "welcome",
      "userId": userId,
      "rooms": getRoomList()
    }
    await socket.send($welcome)
    
    # Auto-join general
    await joinRoom(userId, "general")
    
    # Main loop
    while socket.readyState == Open:
      let packet = await socket.receiveStrPacket()
      if packet.len == 0: continue
      
      let data = parseJson(packet)
      let msgType = data["type"].getStr()
      
      case msgType
      of "joinRoom":
        let roomId = data["roomId"].getStr()
        if roomId in rooms:
          await joinRoom(userId, roomId)
        else:
          await socket.send($(%*{"type": "error", "message": "Room not found"}))
      
      of "leaveRoom":
        let roomId = data["roomId"].getStr()
        await leaveRoom(userId, roomId)
      
      of "sendMessage":
        let roomId = data["roomId"].getStr()
        let content = data["content"].getStr()
        await sendRoomMessage(userId, roomId, content)
      
      of "createRoom":
        inc roomCounter
        let newRoomId = &"room_{roomCounter}"
        let newRoomName = data["name"].getStr()
        
        rooms[newRoomId] = Room(
          id: newRoomId,
          name: newRoomName,
          roomType: rtPublic,
          members: initHashSet[string](),
          createdBy: username,
          createdAt: now(),
          description: data.getOrDefault("description").getStr("No description")
        )
        
        await joinRoom(userId, newRoomId)
        echo &"[ROOM] Created: {newRoomName} by {username}"
      
      of "listRooms":
        await socket.send($(%*{
          "type": "roomList",
          "rooms": getRoomList()
        }))
      
      of "ping":
        await socket.send($(%*{"type": "pong"}))
  
  except WebSocketError as e:
    echo &"Error ({username}): {e.msg}"
  finally:
    if userId in roomUsers:
      let user = roomUsers[userId]
      for roomId in user.rooms:
        if roomId in rooms:
          rooms[roomId].members.excl(userId)
          asyncCheck broadcastToRoom(roomId, %*{
            "type": "roomEvent",
            "event": "left",
            "roomId": roomId,
            "username": username
          })
      roomUsers.del(userId)

proc main() {.async.} =
  createDefaultRooms()
  
  var server = newAsyncHttpServer()
  
  proc cb(req: Request) {.async.} =
    if req.url.path == "/ws":
      await handleRoomClient(req)
    else:
      await req.respond(Http404, "Not Found")
  
  echo "Room chat server on port 8080"
  await server.serve(Port(8080), cb)

waitFor main()
```

---

## Step 216: Direct Messages

เพิ่มฟีเจอร์ส่งข้อความส่วนตัว:

```nim
# file: src/direct_messages.nim
import asyncdispatch, asynchttpserver, ws, json, strformat
import times, tables, strutils

type
  DMMessage = object
    fromUser: string
    toUser: string
    content: string
    timestamp: DateTime
    read: bool

var
  connectedUsers = initTable[string, WebSocket]()  # username -> socket
  dmHistory = initTable[string, seq[DMMessage]]()  # "user1:user2" -> messages

proc getDMKey(user1, user2: string): string =
  # สร้าง key ที่ consistent ไม่ว่าจะสลับ user
  if user1 < user2:
    return &"{user1}:{user2}"
  else:
    return &"{user2}:{user1}"

proc storeDM(fromUser, toUser, content: string) =
  let key = getDMKey(fromUser, toUser)
  if key notin dmHistory:
    dmHistory[key] = @[]
  
  dmHistory[key].add(DMMessage(
    fromUser: fromUser,
    toUser: toUser,
    content: content,
    timestamp: now(),
    read: false
  ))
  
  # จำกัดไว้ที่ 100 messages ต่อ conversation
  if dmHistory[key].len > 100:
    dmHistory[key] = dmHistory[key][dmHistory[key].len - 100 .. ^1]

proc getDMHistory(user1, user2: string): JsonNode =
  let key = getDMKey(user1, user2)
  var arr = newJArray()
  
  if key in dmHistory:
    for msg in dmHistory[key]:
      arr.add(%*{
        "from": msg.fromUser,
        "to": msg.toUser,
        "content": msg.content,
        "timestamp": msg.timestamp.format("yyyy-MM-dd HH:mm:ss"),
        "read": msg.read
      })
  
  return arr

proc sendDirectMessage(fromUser, toUser, content: string) {.async.} =
  storeDM(fromUser, toUser, content)
  
  let dmMsg = %*{
    "type": "directMessage",
    "from": fromUser,
    "to": toUser,
    "content": content,
    "timestamp": $now()
  }
  
  # ส่งให้ recipient ถ้า online
  if toUser in connectedUsers:
    let recipientSocket = connectedUsers[toUser]
    if recipientSocket.readyState == Open:
      try:
        await recipientSocket.send($dmMsg)
      except WebSocketError:
        discard
  
  # ส่ง confirmation กลับให้ sender
  if fromUser in connectedUsers:
    let senderSocket = connectedUsers[fromUser]
    let confirmMsg = %*{
      "type": "dmSent",
      "to": toUser,
      "content": content,
      "delivered": toUser in connectedUsers,
      "timestamp": $now()
    }
    if senderSocket.readyState == Open:
      try:
        await senderSocket.send($confirmMsg)
      except WebSocketError:
        discard

proc handleDMClient(req: Request) {.async.} =
  var socket = await newWebSocket(req)
  var username = ""
  
  try:
    let firstPacket = await socket.receiveStrPacket()
    let loginData = parseJson(firstPacket)
    username = loginData["username"].getStr().strip()
    
    if username in connectedUsers:
      await socket.send($(%*{"type": "error", "message": "Username taken"}))
      socket.close()
      return
    
    connectedUsers[username] = socket
    
    # แจ้ง user ว่า login สำเร็จ
    let loginAck = %*{
      "type": "loggedIn",
      "username": username,
      "onlineUsers": block:
        var arr = newJArray()
        for u, _ in connectedUsers:
          if u != username:
            arr.add(%u)
        arr
    }
    await socket.send($loginAck)
    
    # แจ้ง user อื่นว่า user นี้ online
    let onlineMsg = %*{
      "type": "userOnline",
      "username": username
    }
    for u, s in connectedUsers:
      if u != username and s.readyState == Open:
        try:
          await s.send($onlineMsg)
        except WebSocketError:
          discard
    
    while socket.readyState == Open:
      let packet = await socket.receiveStrPacket()
      if packet.len == 0: continue
      
      let data = parseJson(packet)
      let msgType = data["type"].getStr()
      
      case msgType
      of "sendDM":
        let toUser = data["to"].getStr()
        let content = data["content"].getStr().strip()
        
        if toUser == username:
          await socket.send($(%*{"type": "error", "message": "Cannot DM yourself"}))
          continue
        
        if content.len == 0 or content.len > 1000:
          await socket.send($(%*{"type": "error", "message": "Invalid message length"}))
          continue
        
        await sendDirectMessage(username, toUser, content)
      
      of "getDMHistory":
        let withUser = data["with"].getStr()
        let history = getDMHistory(username, withUser)
        await socket.send($(%*{
          "type": "dmHistory",
          "with": withUser,
          "messages": history
        }))
      
      of "getOnlineUsers":
        var onlineArr = newJArray()
        for u, s in connectedUsers:
          if s.readyState == Open:
            onlineArr.add(%u)
        await socket.send($(%*{"type": "onlineUsers", "users": onlineArr}))
      
      of "ping":
        await socket.send($(%*{"type": "pong", "timestamp": epochTime()}))
  
  except WebSocketError as e:
    echo &"DM Error ({username}): {e.msg}"
  finally:
    if username.len > 0:
      connectedUsers.del(username)
      
      # แจ้ง user อื่นว่า user นี้ offline
      let offlineMsg = %*{
        "type": "userOffline",
        "username": username
      }
      for u, s in connectedUsers:
        if s.readyState == Open:
          try:
            asyncCheck s.send($offlineMsg)
          except: discard

proc main() {.async.} =
  var server = newAsyncHttpServer()
  
  proc cb(req: Request) {.async.} =
    if req.url.path == "/ws":
      await handleDMClient(req)
    else:
      await req.respond(Http404, "Not Found")
  
  echo "DM server running on port 8080"
  await server.serve(Port(8080), cb)

waitFor main()
```

---

## Step 217: Heartbeat และ Ping-Pong

จัดการ connection ที่ inactive ด้วย heartbeat mechanism:

```nim
# file: src/heartbeat.nim
import asyncdispatch, asynchttpserver, ws, json, strformat
import times, tables, asyncfutures

type
  ClientState = enum
    csActive = "active"
    csWaiting = "waiting"     # รอ pong
    csDisconnected = "disconnected"

  HeartbeatClient = ref object
    id: string
    socket: WebSocket
    state: ClientState
    lastPing: DateTime
    lastPong: DateTime
    pingCount: int
    missedPings: int

const
  HEARTBEAT_INTERVAL_MS = 30_000  # 30 วินาที
  PING_TIMEOUT_MS = 10_000        # 10 วินาที
  MAX_MISSED_PINGS = 3            # disconnect หลัง 3 missed pings

var heartbeatClients = initTable[string, HeartbeatClient]()

proc sendPing(client: HeartbeatClient) {.async.} =
  if client.socket.readyState != Open:
    return
  
  client.state = csWaiting
  client.lastPing = now()
  inc client.pingCount
  
  let pingMsg = %*{
    "type": "ping",
    "pingId": client.pingCount,
    "timestamp": epochTime()
  }
  
  try:
    await client.socket.send($pingMsg)
    echo &"[PING -> {client.id}] #{client.pingCount}"
  except WebSocketError:
    client.state = csDisconnected

proc checkHeartbeats() {.async.} =
  while true:
    await sleepAsync(HEARTBEAT_INTERVAL_MS)
    
    var toDisconnect: seq[string] = @[]
    
    for id, client in heartbeatClients:
      if client.state == csDisconnected:
        toDisconnect.add(id)
        continue
      
      if client.state == csWaiting:
        # ยังรอ pong อยู่
        inc client.missedPings
        echo &"[MISSED PING] {id}: {client.missedPings}/{MAX_MISSED_PINGS}"
        
        if client.missedPings >= MAX_MISSED_PINGS:
          echo &"[TIMEOUT] Disconnecting {id}"
          client.socket.close()
          toDisconnect.add(id)
          continue
      else:
        client.missedPings = 0
      
      # ส่ง ping ใหม่
      asyncCheck sendPing(client)
    
    for id in toDisconnect:
      heartbeatClients.del(id)
    
    echo &"[HEARTBEAT] Active clients: {heartbeatClients.len}"

proc handleHeartbeatClient(req: Request) {.async.} =
  var socket = await newWebSocket(req)
  let clientId = &"client_{epochTime().int}_{rand(9999)}"
  
  let client = HeartbeatClient(
    id: clientId,
    socket: socket,
    state: csActive,
    lastPing: now(),
    lastPong: now(),
    pingCount: 0,
    missedPings: 0
  )
  
  heartbeatClients[clientId] = client
  
  # ส่ง initial ping
  asyncCheck sendPing(client)
  
  try:
    while socket.readyState == Open:
      let packet = await socket.receiveStrPacket()
      if packet.len == 0: continue
      
      let data = parseJson(packet)
      let msgType = data["type"].getStr()
      
      case msgType
      of "pong":
        let pingId = data["pingId"].getInt()
        echo &"[PONG <- {clientId}] #{pingId}"
        
        client.state = csActive
        client.lastPong = now()
        client.missedPings = 0
        
        # คำนวณ latency
        let latency = epochTime() - data["originalTimestamp"].getFloat()
        let ackMsg = %*{
          "type": "heartbeatAck",
          "pingId": pingId,
          "latencyMs": int(latency * 1000)
        }
        await socket.send($ackMsg)
      
      of "message":
        echo &"[MSG] {clientId}: {data[\"content\"].getStr()}"
        # Echo back
        await socket.send($(%*{
          "type": "message",
          "content": data["content"].getStr(),
          "echo": true
        }))
      
      else:
        discard
  
  except WebSocketError as e:
    echo &"[DISCONNECT] {clientId}: {e.msg}"
  finally:
    heartbeatClients.del(clientId)
    echo &"[REMOVED] {clientId}, remaining: {heartbeatClients.len}"

proc main() {.async.} =
  # Start heartbeat checker ใน background
  asyncCheck checkHeartbeats()
  
  var server = newAsyncHttpServer()
  
  proc cb(req: Request) {.async.} =
    if req.url.path == "/ws":
      await handleHeartbeatClient(req)
    elif req.url.path == "/api/clients":
      var arr = newJArray()
      for id, client in heartbeatClients:
        arr.add(%*{
          "id": id,
          "state": $client.state,
          "pingCount": client.pingCount,
          "missedPings": client.missedPings
        })
      await req.respond(Http200, $(%*{"clients": arr}),
        newHttpHeaders([("Content-Type", "application/json")]))
    else:
      await req.respond(Http404, "Not Found")
  
  echo "Heartbeat server on port 8080"
  await server.serve(Port(8080), cb)

waitFor main()
```

---

## Step 218: WebSocket Client ใน Nim

สร้าง WebSocket client เพื่อ test server:

```nim
# file: src/ws_client.nim
import asyncdispatch, ws, json, strformat, strutils, asyncfutures
import os, terminal, times

type
  ClientStatus = enum
    csConnecting
    csConnected
    csDisconnected

proc printMessage(msgType, content: string) =
  let timestamp = now().format("HH:mm:ss")
  case msgType
  of "system":
    setForegroundColor(fgYellow)
    echo &"[{timestamp}] [SYSTEM] {content}"
  of "message":
    setForegroundColor(fgGreen)
    echo &"[{timestamp}] {content}"
  of "error":
    setForegroundColor(fgRed)
    echo &"[{timestamp}] [ERROR] {content}"
  of "ping", "pong":
    setForegroundColor(fgCyan)
    echo &"[{timestamp}] [{msgType.toUpperAscii()}]"
  else:
    resetAttributes()
    echo &"[{timestamp}] {content}"
  resetAttributes()

proc runChatClient(serverUrl, username: string) {.async.} =
  var ws: WebSocket
  var status = csConnecting
  
  printMessage("system", &"Connecting to {serverUrl}...")
  
  try:
    ws = await newWebSocket(serverUrl)
    status = csConnected
    printMessage("system", "Connected!")
    
    # ส่ง join message
    let joinMsg = %*{
      "type": "join",
      "username": username
    }
    await ws.send($joinMsg)
    
    # อ่านข้อความใน background
    proc receiveLoop() {.async.} =
      while ws.readyState == Open:
        try:
          let packet = await ws.receiveStrPacket()
          if packet.len == 0: continue
          
          let data = parseJson(packet)
          let msgType = data["type"].getStr()
          
          case msgType
          of "welcome":
            printMessage("system", &"Welcome! You are: {data[\"username\"].getStr()}")
            printMessage("system", &"Users online: {data[\"users\"].len}")
          
          of "message":
            let fromUser = data["from"].getStr()
            let content = data["content"].getStr()
            printMessage("message", &"{fromUser}: {content}")
          
          of "join":
            printMessage("system", &"{data[\"from\"].getStr()} joined")
          
          of "leave":
            printMessage("system", &"{data[\"from\"].getStr()} left")
          
          of "pong":
            printMessage("pong", "")
          
          of "error":
            printMessage("error", data["message"].getStr())
          
          else:
            echo &"[RAW] {packet}"
        
        except WebSocketError:
          break
      
      status = csDisconnected
    
    asyncCheck receiveLoop()
    
    # Input loop
    while status == csConnected and ws.readyState == Open:
      let line = readLine(stdin)
      
      if line.startsWith("/"):
        # Commands
        let parts = line.splitWhitespace()
        case parts[0]
        of "/quit", "/exit":
          break
        of "/ping":
          let pingMsg = %*{"type": "ping"}
          await ws.send($pingMsg)
        of "/users":
          let getUsersMsg = %*{"type": "getUserList"}
          await ws.send($getUsersMsg)
        of "/help":
          echo "Commands: /quit, /ping, /users, /help"
        else:
          printMessage("error", &"Unknown command: {parts[0]}")
      else:
        # ส่งข้อความทั่วไป
        if line.len > 0:
          let chatMsg = %*{
            "type": "message",
            "content": line
          }
          await ws.send($chatMsg)
  
  except WebSocketError as e:
    printMessage("error", &"Connection failed: {e.msg}")
  finally:
    if not ws.isNil:
      ws.close()
    printMessage("system", "Disconnected")

proc main() {.async.} =
  let args = commandLineParams()
  
  let serverUrl = if args.len > 0: args[0] else: "ws://localhost:8080/ws"
  let username = if args.len > 1: args[1] else: "TestUser"
  
  echo &"Nim WebSocket Client"
  echo &"Server: {serverUrl}"
  echo &"Username: {username}"
  echo "Type messages or /help for commands"
  echo "---"
  
  await runChatClient(serverUrl, username)

waitFor main()
```

---

## Step 219: Load Testing WebSocket

ทดสอบ performance ด้วย concurrent connections:

```nim
# file: src/ws_load_test.nim
import asyncdispatch, ws, json, strformat
import times, atomics, strutils, os

type
  TestMetrics = object
    connections: Atomic[int]
    messagesSent: Atomic[int]
    messagesReceived: Atomic[int]
    errors: Atomic[int]
    totalLatency: Atomic[int64]  # milliseconds

var metrics: TestMetrics

proc runTestClient(clientId: int, serverUrl: string, messageCount: int) {.async.} =
  var socket: WebSocket
  
  try:
    socket = await newWebSocket(serverUrl)
    metrics.connections.atomicInc()
    
    let username = &"TestUser{clientId}"
    let joinMsg = %*{"type": "join", "username": username}
    await socket.send($joinMsg)
    
    # อ่าน welcome
    discard await socket.receiveStrPacket()
    
    for i in 0..<messageCount:
      let startTime = epochTime()
      
      let msg = %*{
        "type": "message",
        "content": &"Test message {i} from {username}"
      }
      
      await socket.send($msg)
      metrics.messagesSent.atomicInc()
      
      # รอ receive
      let response = await socket.receiveStrPacket()
      if response.len > 0:
        metrics.messagesReceived.atomicInc()
        let latency = int64((epochTime() - startTime) * 1000)
        discard metrics.totalLatency.atomicAddFetch(latency)
      
      await sleepAsync(10)  # 10ms delay between messages
    
  except WebSocketError as e:
    metrics.errors.atomicInc()
  finally:
    if not socket.isNil:
      socket.close()
    metrics.connections.atomicDec()

proc runLoadTest(serverUrl: string, numClients, messagesPerClient: int) {.async.} =
  echo &"Starting load test: {numClients} clients, {messagesPerClient} messages each"
  
  let startTime = epochTime()
  var futures: seq[Future[void]] = @[]
  
  for i in 0..<numClients:
    futures.add(runTestClient(i, serverUrl, messagesPerClient))
    await sleepAsync(10)  # Stagger connections slightly
  
  await all(futures)
  
  let duration = epochTime() - startTime
  let totalMessages = numClients * messagesPerClient
  let received = metrics.messagesReceived.load()
  let errors = metrics.errors.load()
  let totalLatency = metrics.totalLatency.load()
  
  echo "\n=== Load Test Results ==="
  echo &"Duration: {duration:.2f}s"
  echo &"Total clients: {numClients}"
  echo &"Messages sent: {metrics.messagesSent.load()}"
  echo &"Messages received: {received}"
  echo &"Errors: {errors}"
  echo &"Throughput: {totalMessages.float / duration:.2f} msg/s"
  if received > 0:
    echo &"Avg latency: {totalLatency div received}ms"

proc main() {.async.} =
  let serverUrl = "ws://localhost:8080/ws"
  await runLoadTest(serverUrl, numClients = 10, messagesPerClient = 5)

waitFor main()
```

---

## Step 220: WebSocket Middleware และ Authentication

เพิ่ม authentication ให้กับ WebSocket:

```nim
# file: src/ws_auth.nim
import asyncdispatch, asynchttpserver, ws, json, strformat
import times, tables, strutils, base64, sha1

type
  AuthToken = object
    userId: string
    username: string
    role: string
    expiresAt: DateTime

var validTokens = initTable[string, AuthToken]()

proc generateToken(userId, username, role: string): string =
  # ใน production ควรใช้ JWT จริงๆ
  let payload = &"{userId}:{username}:{role}:{epochTime()}"
  return base64.encode(payload)

proc validateToken(token: string): (bool, AuthToken) =
  if token in validTokens:
    let auth = validTokens[token]
    if auth.expiresAt > now():
      return (true, auth)
  return (false, AuthToken())

proc createSession(userId, username: string, role = "user"): string =
  let token = generateToken(userId, username, role)
  validTokens[token] = AuthToken(
    userId: userId,
    username: username,
    role: role,
    expiresAt: now() + 1.hours
  )
  return token

proc handleAuthWebSocket(req: Request) {.async.} =
  # ตรวจสอบ token ใน query string หรือ header
  var token = ""
  
  # ดึง token จาก URL: ws://server/ws?token=xxx
  let queryStr = req.url.query
  if queryStr.len > 0:
    for param in queryStr.split("&"):
      let parts = param.split("=")
      if parts.len == 2 and parts[0] == "token":
        token = parts[1]
  
  # ตรวจสอบ token
  let (valid, authInfo) = validateToken(token)
  
  if not valid:
    # ส่ง 401 response ก่อน upgrade
    await req.respond(Http401, "Unauthorized: Invalid or expired token")
    return
  
  var socket = await newWebSocket(req)
  
  try:
    # ส่ง auth confirmation
    let authMsg = %*{
      "type": "authenticated",
      "userId": authInfo.userId,
      "username": authInfo.username,
      "role": authInfo.role
    }
    await socket.send($authMsg)
    
    while socket.readyState == Open:
      let packet = await socket.receiveStrPacket()
      if packet.len == 0: continue
      
      let data = parseJson(packet)
      let msgType = data["type"].getStr()
      
      # Role-based access control
      case msgType
      of "adminCommand":
        if authInfo.role != "admin":
          await socket.send($(%*{
            "type": "error",
            "message": "Admin access required"
          }))
          continue
        
        # Process admin command
        echo &"[ADMIN] {authInfo.username}: {data}"
        await socket.send($(%*{"type": "adminAck", "status": "processed"}))
      
      of "message":
        let response = %*{
          "type": "message",
          "from": authInfo.username,
          "content": data["content"].getStr(),
          "role": authInfo.role
        }
        await socket.send($response)
      
      else:
        discard
  
  except WebSocketError as e:
    echo &"Auth WS error: {e.msg}"
  finally:
    socket.close()

proc main() {.async.} =
  # สร้าง test tokens
  let adminToken = createSession("1", "admin", "admin")
  let userToken = createSession("2", "alice", "user")
  
  echo &"Admin token: {adminToken}"
  echo &"User token: {userToken}"
  
  var server = newAsyncHttpServer()
  
  proc cb(req: Request) {.async.} =
    if req.url.path == "/ws":
      await handleAuthWebSocket(req)
    elif req.url.path == "/api/login":
      if req.reqMethod == HttpPost:
        let body = parseJson(req.body)
        let username = body["username"].getStr()
        let password = body["password"].getStr()
        
        # ใน production ตรวจสอบกับ database
        if username == "admin" and password == "secret":
          let token = createSession("1", "admin", "admin")
          await req.respond(Http200,
            $(%*{"token": token, "role": "admin"}),
            newHttpHeaders([("Content-Type", "application/json")]))
        elif username == "alice" and password == "pass123":
          let token = createSession("2", "alice", "user")
          await req.respond(Http200,
            $(%*{"token": token, "role": "user"}),
            newHttpHeaders([("Content-Type", "application/json")]))
        else:
          await req.respond(Http401,
            $(%*{"error": "Invalid credentials"}),
            newHttpHeaders([("Content-Type", "application/json")]))
      else:
        await req.respond(Http405, "Method Not Allowed")
    else:
      await req.respond(Http404, "Not Found")
  
  echo "Auth WebSocket server on port 8080"
  await server.serve(Port(8080), cb)

waitFor main()
```

---

## Step 221: Binary Data ผ่าน WebSocket

ส่งข้อมูล binary เช่นรูปภาพหรือไฟล์:

```nim
# file: src/ws_binary.nim
import asyncdispatch, asynchttpserver, ws, json, strformat
import times, base64, os, strutils

proc handleBinaryClient(req: Request) {.async.} =
  var socket = await newWebSocket(req)
  
  try:
    await socket.send($(%*{
      "type": "ready",
      "message": "Binary transfer ready",
      "maxSize": 10 * 1024 * 1024  # 10MB max
    }))
    
    while socket.readyState == Open:
      # รับข้อมูล - อาจเป็น text หรือ binary
      let (opcode, data) = await socket.receivePacket()
      
      case opcode
      of Opcode.Text:
        let jsonData = parseJson(data)
        let msgType = jsonData["type"].getStr()
        
        case msgType
        of "uploadStart":
          let filename = jsonData["filename"].getStr()
          let fileSize = jsonData["size"].getInt()
          echo &"File upload starting: {filename} ({fileSize} bytes)"
          await socket.send($(%*{
            "type": "uploadReady",
            "filename": filename
          }))
        
        of "uploadComplete":
          let filename = jsonData["filename"].getStr()
          echo &"File upload complete: {filename}"
          await socket.send($(%*{
            "type": "uploadSuccess",
            "filename": filename
          }))
        
        else:
          echo &"Text message: {data}"
      
      of Opcode.Binary:
        echo &"Received binary data: {data.len} bytes"
        # Process binary data
        # สำหรับ demo เราแค่ echo กลับ
        await socket.sendBinary(data)
      
      of Opcode.Ping:
        await socket.send(data, Opcode.Pong)
      
      of Opcode.Close:
        break
      
      else:
        discard
  
  except WebSocketError as e:
    echo &"Binary WS error: {e.msg}"
  finally:
    socket.close()

proc main() {.async.} =
  var server = newAsyncHttpServer()
  
  proc cb(req: Request) {.async.} =
    if req.url.path == "/ws":
      await handleBinaryClient(req)
    else:
      await req.respond(Http404, "Not Found")
  
  echo "Binary WebSocket server on port 8080"
  await server.serve(Port(8080), cb)

waitFor main()
```

---

## Step 222: Real-World: Live Notification System

สร้าง notification system ที่ใช้ใน production จริง:

```nim
# file: src/notification_system.nim
import asyncdispatch, asynchttpserver, ws, json, strformat
import times, tables, sets, strutils, sequtils, algorithm

type
  NotificationType = enum
    ntInfo = "info"
    ntWarning = "warning"
    ntError = "error"
    ntSuccess = "success"
    ntAlert = "alert"

  Priority = enum
    pLow = 1
    pNormal = 2
    pHigh = 3
    pCritical = 4

  Notification = object
    id: string
    notifType: NotificationType
    priority: Priority
    title: string
    message: string
    userId: string       # "" = broadcast to all
    channel: string      # specific channel/topic
    createdAt: DateTime
    expiresAt: DateTime
    read: bool
    metadata: JsonNode

  Subscriber = ref object
    userId: string
    socket: WebSocket
    channels: HashSet[string]
    preferences: NotifPreferences

  NotifPreferences = object
    minPriority: Priority
    enabledTypes: set[NotificationType]
    quietHoursStart: int  # hour 0-23
    quietHoursEnd: int

var
  subscribers = initTable[string, Subscriber]()
  notifications = initTable[string, Notification]()
  userNotifications = initTable[string, seq[string]]()  # userId -> notif IDs
  notifCounter = 0

proc createNotif(
  notifType: NotificationType,
  priority: Priority,
  title, message: string,
  userId = "",
  channel = "general",
  ttlSeconds = 3600,
  metadata = newJObject()
): Notification =
  inc notifCounter
  let id = &"notif_{notifCounter}_{epochTime().int}"
  return Notification(
    id: id,
    notifType: notifType,
    priority: priority,
    title: title,
    message: message,
    userId: userId,
    channel: channel,
    createdAt: now(),
    expiresAt: now() + initTimeInterval(seconds = ttlSeconds),
    read: false,
    metadata: metadata
  )

proc notifToJson(n: Notification): JsonNode =
  %*{
    "id": n.id,
    "type": $n.notifType,
    "priority": ord(n.priority),
    "title": n.title,
    "message": n.message,
    "channel": n.channel,
    "createdAt": n.createdAt.format("yyyy-MM-dd HH:mm:ss"),
    "expiresAt": n.expiresAt.format("yyyy-MM-dd HH:mm:ss"),
    "read": n.read,
    "metadata": n.metadata
  }

proc shouldReceive(sub: Subscriber, notif: Notification): bool =
  # ตรวจสอบ priority
  if ord(notif.priority) < ord(sub.preferences.minPriority):
    return false
  
  # ตรวจสอบ type
  if notif.notifType notin sub.preferences.enabledTypes:
    return false
  
  # ตรวจสอบ quiet hours
  let currentHour = now().hour
  if sub.preferences.quietHoursStart < sub.preferences.quietHoursEnd:
    if currentHour >= sub.preferences.quietHoursStart and 
       currentHour < sub.preferences.quietHoursEnd:
      if ord(notif.priority) < ord(pCritical):
        return false
  
  # ตรวจสอบ channel subscription
  if notif.channel != "system" and notif.channel notin sub.channels:
    return false
  
  return true

proc deliverNotification(notif: Notification) {.async.} =
  notifications[notif.id] = notif
  
  for userId, sub in subscribers:
    # ตรวจสอบว่าเป็น notification สำหรับ user นี้ หรือ broadcast
    if notif.userId.len > 0 and notif.userId != userId:
      continue
    
    if not shouldReceive(sub, notif):
      continue
    
    # เพิ่มใน user's notification list
    if userId notin userNotifications:
      userNotifications[userId] = @[]
    userNotifications[userId].add(notif.id)
    
    # ส่งถ้า socket ยังเปิดอยู่
    if sub.socket.readyState == Open:
      try:
        await sub.socket.send($(%*{
          "type": "notification",
          "notification": notifToJson(notif)
        }))
      except WebSocketError:
        discard

proc handleNotifClient(req: Request) {.async.} =
  var socket = await newWebSocket(req)
  var userId = ""
  
  try:
    let firstPacket = await socket.receiveStrPacket()
    let loginData = parseJson(firstPacket)
    userId = loginData["userId"].getStr()
    
    let sub = Subscriber(
      userId: userId,
      socket: socket,
      channels: initHashSet[string](),
      preferences: NotifPreferences(
        minPriority: pNormal,
        enabledTypes: {ntInfo, ntWarning, ntError, ntSuccess, ntAlert},
        quietHoursStart: 0,
        quietHoursEnd: 0
      )
    )
    
    # Default channels
    sub.channels.incl("general")
    sub.channels.incl("system")
    
    subscribers[userId] = sub
    
    # ส่ง unread notifications
    var unread = newJArray()
    if userId in userNotifications:
      for nId in userNotifications[userId]:
        if nId in notifications and not notifications[nId].read:
          if notifications[nId].expiresAt > now():
            unread.add(notifToJson(notifications[nId]))
    
    await socket.send($(%*{
      "type": "connected",
      "userId": userId,
      "unreadCount": unread.len,
      "notifications": unread
    }))
    
    while socket.readyState == Open:
      let packet = await socket.receiveStrPacket()
      if packet.len == 0: continue
      
      let data = parseJson(packet)
      let msgType = data["type"].getStr()
      
      case msgType
      of "subscribe":
        let channel = data["channel"].getStr()
        subscribers[userId].channels.incl(channel)
        await socket.send($(%*{
          "type": "subscribed",
          "channel": channel
        }))
      
      of "unsubscribe":
        let channel = data["channel"].getStr()
        if channel != "system":  # ไม่ให้ unsubscribe จาก system
          subscribers[userId].channels.excl(channel)
        await socket.send($(%*{
          "type": "unsubscribed",
          "channel": channel
        }))
      
      of "markRead":
        let notifId = data["notifId"].getStr()
        if notifId in notifications:
          notifications[notifId].read = true
          await socket.send($(%*{
            "type": "marked",
            "notifId": notifId
          }))
      
      of "markAllRead":
        if userId in userNotifications:
          for nId in userNotifications[userId]:
            if nId in notifications:
              notifications[nId].read = true
        await socket.send($(%*{"type": "allMarked"}))
      
      of "updatePreferences":
        let prefs = data["preferences"]
        let sub = subscribers[userId]
        
        if prefs.hasKey("minPriority"):
          sub.preferences.minPriority = Priority(prefs["minPriority"].getInt())
        if prefs.hasKey("quietHoursStart"):
          sub.preferences.quietHoursStart = prefs["quietHoursStart"].getInt()
        if prefs.hasKey("quietHoursEnd"):
          sub.preferences.quietHoursEnd = prefs["quietHoursEnd"].getInt()
        
        await socket.send($(%*{"type": "preferencesUpdated"}))
      
      of "ping":
        await socket.send($(%*{"type": "pong", "timestamp": epochTime()}))
  
  except WebSocketError as e:
    echo &"Notif error ({userId}): {e.msg}"
  finally:
    if userId.len > 0:
      subscribers.del(userId)

proc main() {.async.} =
  var server = newAsyncHttpServer()
  
  proc cb(req: Request) {.async.} =
    case req.url.path
    of "/ws":
      await handleNotifClient(req)
    
    of "/api/notify":
      if req.reqMethod == HttpPost:
        let body = parseJson(req.body)
        
        let notifTypeStr = body["type"].getStr("info")
        let notifType = case notifTypeStr
          of "warning": ntWarning
          of "error": ntError
          of "success": ntSuccess
          of "alert": ntAlert
          else: ntInfo
        
        let priorityInt = body.getOrDefault("priority").getInt(2)
        let priority = case priorityInt
          of 1: pLow
          of 3: pHigh
          of 4: pCritical
          else: pNormal
        
        let notif = createNotif(
          notifType = notifType,
          priority = priority,
          title = body["title"].getStr(),
          message = body["message"].getStr(),
          userId = body.getOrDefault("userId").getStr(""),
          channel = body.getOrDefault("channel").getStr("general"),
          ttlSeconds = body.getOrDefault("ttl").getInt(3600),
          metadata = body.getOrDefault("metadata")
        )
        
        asyncCheck deliverNotification(notif)
        
        await req.respond(Http202,
          $(%*{"id": notif.id, "status": "queued"}),
          newHttpHeaders([("Content-Type", "application/json")]))
      else:
        await req.respond(Http405, "Method Not Allowed")
    
    of "/api/stats":
      let activeConnections = subscribers.len
      let totalNotifs = notifications.len
      await req.respond(Http200,
        $(%*{
          "activeConnections": activeConnections,
          "totalNotifications": totalNotifs,
          "channels": ["general", "system"]
        }),
        newHttpHeaders([("Content-Type", "application/json")]))
    
    else:
      await req.respond(Http404, "Not Found")
  
  # ส่ง test notification ทุก 30 วินาที
  proc sendPeriodic() {.async.} =
    var count = 0
    while true:
      await sleepAsync(30_000)
      inc count
      let notif = createNotif(
        notifType = ntInfo,
        priority = pNormal,
        title = "System Status",
        message = &"System running normally. Check #{count}",
        channel = "system"
      )
      asyncCheck deliverNotification(notif)
  
  asyncCheck sendPeriodic()
  
  echo "Notification server on port 8080"
  echo "WebSocket: ws://localhost:8080/ws"
  echo "API: POST /api/notify"
  await server.serve(Port(8080), cb)

waitFor main()
```

---

## Step 223: Frontend HTML Client สำหรับ Chat

สร้าง HTML client เพื่อทดสอบ:

```nim
# file: src/serve_frontend.nim
# server ที่ serve ทั้ง WebSocket และ HTML frontend

import asyncdispatch, asynchttpserver, ws, json, strformat

const htmlClient = """
<!DOCTYPE html>
<html>
<head>
  <title>Nim Chat</title>
  <style>
    body { font-family: Arial; max-width: 800px; margin: 50px auto; }
    #messages { height: 400px; overflow-y: scroll; border: 1px solid #ccc; padding: 10px; }
    .message { margin: 5px 0; }
    .system { color: gray; font-style: italic; }
    .error { color: red; }
    input, button { margin: 5px; padding: 5px; }
    #chat-input { width: 70%; }
  </style>
</head>
<body>
  <h1>Nim WebSocket Chat</h1>
  
  <div id="login">
    <input id="username" placeholder="Username" />
    <button onclick="connect()">Connect</button>
  </div>
  
  <div id="chat" style="display:none">
    <div id="messages"></div>
    <div>
      <input id="chat-input" placeholder="Type a message..." 
             onkeypress="if(event.key==='Enter') sendMessage()" />
      <button onclick="sendMessage()">Send</button>
      <button onclick="disconnect()">Disconnect</button>
    </div>
    <div id="users">Users: <span id="user-count">0</span></div>
  </div>
  
  <script>
    let ws = null;
    
    function addMessage(text, type = 'message') {
      const div = document.getElementById('messages');
      const p = document.createElement('p');
      p.className = 'message ' + type;
      p.textContent = new Date().toLocaleTimeString() + ' ' + text;
      div.appendChild(p);
      div.scrollTop = div.scrollHeight;
    }
    
    function connect() {
      const username = document.getElementById('username').value.trim();
      if (!username) return alert('Enter username');
      
      ws = new WebSocket('ws://localhost:8080/ws');
      
      ws.onopen = () => {
        ws.send(JSON.stringify({type: 'join', username: username}));
        document.getElementById('login').style.display = 'none';
        document.getElementById('chat').style.display = 'block';
        addMessage('Connected!', 'system');
      };
      
      ws.onmessage = (event) => {
        const data = JSON.parse(event.data);
        switch(data.type) {
          case 'welcome':
            addMessage('Welcome ' + data.username, 'system');
            document.getElementById('user-count').textContent = data.users.length;
            break;
          case 'message':
            addMessage(data.from + ': ' + data.content);
            break;
          case 'join':
            addMessage(data.from + ' joined', 'system');
            break;
          case 'leave':
            addMessage(data.from + ' left', 'system');
            break;
          case 'error':
            addMessage('Error: ' + data.message, 'error');
            break;
        }
      };
      
      ws.onclose = () => addMessage('Disconnected', 'system');
      ws.onerror = () => addMessage('Connection error', 'error');
    }
    
    function sendMessage() {
      const input = document.getElementById('chat-input');
      const text = input.value.trim();
      if (!text || !ws) return;
      ws.send(JSON.stringify({type: 'message', content: text}));
      input.value = '';
    }
    
    function disconnect() {
      if (ws) ws.close();
    }
  </script>
</body>
</html>
"""

proc main() {.async.} =
  var server = newAsyncHttpServer()
  
  proc cb(req: Request) {.async.} =
    if req.url.path == "/":
      await req.respond(Http200, htmlClient,
        newHttpHeaders([("Content-Type", "text/html")]))
    elif req.url.path == "/ws":
      # จัดการ WebSocket connection (simplified)
      var socket = await newWebSocket(req)
      try:
        while socket.readyState == Open:
          let packet = await socket.receiveStrPacket()
          if packet.len > 0:
            await socket.send(packet)  # echo
      except: discard
      finally: socket.close()
    else:
      await req.respond(Http404, "Not Found")
  
  echo "Server at http://localhost:8080"
  await server.serve(Port(8080), cb)

waitFor main()
```

---

## Step 224: Error Handling และ Reconnection

จัดการ errors และ reconnection อย่างถูกต้อง:

```nim
# file: src/ws_resilient.nim
import asyncdispatch, ws, json, strformat
import times, asyncfutures, os

type
  ReconnectStrategy = enum
    rsFixed = "fixed"
    rsExponential = "exponential"
    rsLinear = "linear"

  ClientConfig = object
    serverUrl: string
    reconnectStrategy: ReconnectStrategy
    initialDelay: int      # milliseconds
    maxDelay: int
    maxRetries: int
    onMessage: proc(data: JsonNode) {.async.}

proc calculateDelay(strategy: ReconnectStrategy, attempt: int, 
                    initialDelay, maxDelay: int): int =
  case strategy
  of rsFixed:
    return min(initialDelay, maxDelay)
  of rsExponential:
    return min(initialDelay * (2 ^ attempt), maxDelay)
  of rsLinear:
    return min(initialDelay + (attempt * 1000), maxDelay)

proc connectWithRetry(config: ClientConfig) {.async.} =
  var attempt = 0
  
  while attempt <= config.maxRetries or config.maxRetries < 0:
    echo &"[ATTEMPT {attempt + 1}] Connecting to {config.serverUrl}..."
    
    var socket: WebSocket
    var connected = false
    
    try:
      socket = await newWebSocket(config.serverUrl)
      connected = true
      attempt = 0  # reset attempt counter on success
      
      echo "[CONNECTED] Successfully connected"
      
      # ส่ง reconnect notification ถ้าไม่ใช่ครั้งแรก
      if attempt > 0:
        await socket.send($(%*{
          "type": "reconnected",
          "attempt": attempt
        }))
      
      while socket.readyState == Open:
        try:
          let packet = await socket.receiveStrPacket()
          if packet.len > 0:
            let data = parseJson(packet)
            asyncCheck config.onMessage(data)
        except WebSocketError as e:
          echo &"[RECV ERROR] {e.msg}"
          break
    
    except WebSocketError as e:
      echo &"[CONNECT FAILED] {e.msg}"
    finally:
      if not socket.isNil:
        socket.close()
    
    if config.maxRetries >= 0 and attempt >= config.maxRetries:
      echo "[MAX RETRIES] Giving up"
      break
    
    inc attempt
    let delay = calculateDelay(
      config.reconnectStrategy, attempt,
      config.initialDelay, config.maxDelay
    )
    
    echo &"[RETRY] Waiting {delay}ms before retry #{attempt + 1}..."
    await sleepAsync(delay)
  
  echo "[DISCONNECTED] Connection ended"

proc main() {.async.} =
  let config = ClientConfig(
    serverUrl: "ws://localhost:8080/ws",
    reconnectStrategy: rsExponential,
    initialDelay: 1000,
    maxDelay: 30_000,
    maxRetries: 5,
    onMessage: proc(data: JsonNode) {.async.} =
      echo &"[MSG] {data}"
  )
  
  await connectWithRetry(config)

waitFor main()
```

---

## Step 225: Testing WebSocket Endpoints

เขียน tests สำหรับ WebSocket server:

```nim
# file: tests/test_websocket.nim
import asyncdispatch, ws, json, strformat, unittest
import asyncfutures, times, os

proc connectAndSend(url: string, messages: seq[JsonNode]): Future[seq[string]] {.async.} =
  var responses: seq[string] = @[]
  var socket: WebSocket
  
  try:
    socket = await newWebSocket(url)
    
    for msg in messages:
      await socket.send($msg)
      let response = await socket.receiveStrPacket()
      responses.add(response)
    
    socket.close()
  except WebSocketError as e:
    echo &"Test error: {e.msg}"
  
  return responses

suite "WebSocket Server Tests":
  test "Connection and echo":
    proc testEcho() {.async.} =
      let responses = await connectAndSend(
        "ws://localhost:8080/ws",
        @[%*{"type": "ping"}]
      )
      check responses.len == 1
      let data = parseJson(responses[0])
      check data["type"].getStr() == "pong"
    
    waitFor testEcho()
  
  test "Join and receive welcome":
    proc testJoin() {.async.} =
      var socket = await newWebSocket("ws://localhost:8080/ws")
      
      await socket.send($(%*{"type": "join", "username": "TestUser"}))
      let response = await socket.receiveStrPacket()
      let data = parseJson(response)
      
      check data["type"].getStr() == "welcome"
      check data.hasKey("userId")
      
      socket.close()
    
    waitFor testJoin()
  
  test "Message broadcast":
    proc testBroadcast() {.async.} =
      # เชื่อมต่อสอง clients
      var sock1 = await newWebSocket("ws://localhost:8080/ws")
      var sock2 = await newWebSocket("ws://localhost:8080/ws")
      
      await sock1.send($(%*{"type": "join", "username": "User1"}))
      discard await sock1.receiveStrPacket()
      
      await sock2.send($(%*{"type": "join", "username": "User2"}))
      discard await sock2.receiveStrPacket()
      
      # User1 ส่งข้อความ
      await sock1.send($(%*{
        "type": "message",
        "content": "Hello from User1"
      }))
      
      # ทั้งสอง client ควรได้รับ
      let recv1 = await sock1.receiveStrPacket()
      let recv2 = await sock2.receiveStrPacket()
      
      let data1 = parseJson(recv1)
      let data2 = parseJson(recv2)
      
      check data1["type"].getStr() == "message"
      check data2["type"].getStr() == "message"
      check data2["content"].getStr() == "Hello from User1"
      
      sock1.close()
      sock2.close()
    
    waitFor testBroadcast()

# Run tests
when isMainModule:
  echo "Make sure server is running on port 8080 before running tests"
```

---

## 📝 สรุป Part 16

| Step | หัวข้อ | สิ่งที่เรียนรู้ |
|------|--------|----------------|
| 211 | WebSocket Basics | โปรโตคอล, การ upgrade, พื้นฐาน |
| 212 | Setup | Project structure, nimble dependencies |
| 213 | Connection | Handshake, client tracking |
| 214 | Chat Server | Multi-client, broadcast, message types |
| 215 | Room System | Channels, join/leave, room management |
| 216 | Direct Messages | Private messaging, online status |
| 217 | Heartbeat | Ping-pong, timeout detection |
| 218 | WS Client | Nim client, reconnection |
| 219 | Load Testing | Concurrent connections, metrics |
| 220 | Authentication | Token validation, RBAC |
| 221 | Binary Data | File transfer, binary frames |
| 222 | Notification System | Live notifications, preferences |
| 223 | Frontend Client | HTML/JS client integration |
| 224 | Error Handling | Resilient connections, retry logic |
| 225 | Testing | WS endpoint tests |

---

## Navigation

- [← Part 15: REST API](part_15_rest_api.md)
- [→ Part 17: Redis](part_17_redis.md)
- [กลับ README](../README.md)
