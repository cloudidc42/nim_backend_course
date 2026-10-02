# Part 17: Redis ด้วย Nim

## Steps 226-240

Redis (Remote Dictionary Server) เป็น in-memory data structure store ที่ใช้เป็น database, cache, message broker และ queue ในบทนี้เราจะเรียนรู้การใช้ Redis กับ Nim ตั้งแต่พื้นฐานจนถึงระบบ session management และ caching ที่พร้อมใช้ใน production

---

## Step 226: Redis Installation และ Setup

ติดตั้ง Redis และ Nim library:

```bash
# Ubuntu/Debian
sudo apt-get install redis-server
sudo systemctl start redis

# macOS
brew install redis
brew services start redis

# ทดสอบ
redis-cli ping
# PONG
```

เพิ่ม redis library ใน nimble:

```nim
# myproject.nimble
requires "nim >= 2.0.0"
requires "redis >= 0.3.0"

# หรือใช้ nimble ติดตั้ง
# nimble install redis
```

---

## Step 227: Redis Connection พื้นฐาน

```nim
# file: src/redis_basic.nim
import redis, asyncdispatch, strformat, json

proc connectRedis(): Future[AsyncRedis] {.async.} =
  # เชื่อมต่อ Redis server
  let r = await openAsync("localhost", Port(6379))
  return r

proc basicOperations() {.async.} =
  let r = await connectRedis()
  defer: r.close()
  
  echo "=== String Operations ==="
  
  # SET
  let setResult = await r.setk("hello", "world")
  echo &"SET hello world: {setResult}"
  
  # GET
  let value = await r.get("hello")
  echo &"GET hello: {value}"  # world
  
  # SET with expiry (SETEX)
  await r.setex("temp_key", 60, "expires in 60 seconds")
  
  # TTL - ดูเวลาที่เหลือ
  let ttl = await r.ttl("temp_key")
  echo &"TTL temp_key: {ttl}s"
  
  # EXISTS
  let exists = await r.exists("hello")
  echo &"EXISTS hello: {exists}"  # 1 (true)
  
  # DEL
  let deleted = await r.del(@["hello", "temp_key"])
  echo &"DEL: {deleted} keys deleted"
  
  # INCR / DECR
  await r.setk("counter", "0")
  for i in 0..4:
    let newVal = await r.incr("counter")
    echo &"INCR counter: {newVal}"
  
  let decremented = await r.decr("counter")
  echo &"DECR counter: {decremented}"
  
  # INCRBY / DECRBY
  discard await r.incrby("counter", 10)
  let val = await r.get("counter")
  echo &"After INCRBY 10: {val}"
  
  # MSET / MGET - หลายค่าพร้อมกัน
  await r.mset(@[
    ("key1", "value1"),
    ("key2", "value2"),
    ("key3", "value3")
  ])
  
  let vals = await r.mget(@["key1", "key2", "key3", "nonexistent"])
  echo &"MGET: {vals}"
  
  # KEYS pattern
  let keys = await r.keys("key*")
  echo &"Keys matching 'key*': {keys}"

waitFor basicOperations()
```

---

## Step 228: String Operations ขั้นสูง

```nim
# file: src/redis_strings.nim
import redis, asyncdispatch, strformat, json, times

type
  RedisStringOps = ref object
    client: AsyncRedis

proc newRedisStringOps(host = "localhost", port = 6379): Future[RedisStringOps] {.async.} =
  let r = RedisStringOps()
  r.client = await openAsync(host, Port(port))
  return r

proc setWithOptions(r: RedisStringOps, key, value: string, 
                    ex = 0, px = 0, nx = false, xx = false) {.async.} =
  var args = @[key, value]
  
  if ex > 0:
    args.add("EX")
    args.add($ex)
  elif px > 0:
    args.add("PX")
    args.add($px)
  
  if nx:
    args.add("NX")  # Set only if NOT exists
  elif xx:
    args.add("XX")  # Set only if EXISTS
  
  discard await r.client.sendCommand("SET", args)

proc atomicCounter(r: RedisStringOps, key: string, 
                   amount = 1): Future[int64] {.async.} =
  if amount == 1:
    return await r.client.incr(key)
  else:
    return await r.client.incrby(key, amount)

proc getOrSet(r: RedisStringOps, key: string, 
              fetcher: proc(): Future[string] {.async.},
              ttl = 300): Future[string] {.async.} =
  # Cache-aside pattern
  let cached = await r.client.get(key)
  
  if cached.len > 0 and cached != "nil":
    return cached
  
  # Cache miss - fetch and store
  let value = await fetcher()
  await r.client.setex(key, ttl, value)
  return value

proc demo() {.async.} =
  let ops = await newRedisStringOps()
  
  # NX - Set only if not exists (good for distributed locks)
  await ops.setWithOptions("lock:resource1", "locked", ex = 30, nx = true)
  let lockVal = await ops.client.get("lock:resource1")
  echo &"Lock acquired: {lockVal}"
  
  # Try to acquire again - should fail (NX)
  await ops.setWithOptions("lock:resource1", "locked_again", nx = true)
  let lockVal2 = await ops.client.get("lock:resource1")
  echo &"Lock still: {lockVal2}"  # Still "locked"
  
  # Atomic counter
  for i in 1..5:
    let count = await ops.atomicCounter("page_views", 1)
    echo &"Page views: {count}"
  
  # Cache-aside
  let data = await ops.getOrSet(
    "user:1:profile",
    proc(): Future[string] {.async.} =
      echo "Fetching from database..."
      await sleepAsync(100)  # simulate DB query
      return """{"id": 1, "name": "Alice", "email": "alice@example.com"}""",
    ttl = 600
  )
  echo &"Profile: {data}"
  
  # Second call should hit cache
  let data2 = await ops.getOrSet(
    "user:1:profile",
    proc(): Future[string] {.async.} =
      echo "This shouldn't print (from cache)"
      return "{}",
    ttl = 600
  )
  echo &"From cache: {data2}"

waitFor demo()
```

---

## Step 229: Lists (Queue, Stack)

```nim
# file: src/redis_lists.nim
import redis, asyncdispatch, strformat, json

type
  TaskQueue = ref object
    client: AsyncRedis
    queueKey: string

proc newTaskQueue(queueKey: string): Future[TaskQueue] {.async.} =
  let q = TaskQueue(
    client: await openAsync("localhost", Port(6379)),
    queueKey: queueKey
  )
  return q

proc enqueue(q: TaskQueue, task: JsonNode) {.async.} =
  # RPUSH - เพิ่มท้าย list (FIFO queue)
  let taskStr = $task
  let length = await q.client.rpush(q.queueKey, @[taskStr])
  echo &"Enqueued task. Queue length: {length}"

proc dequeue(q: TaskQueue): Future[JsonNode] {.async.} =
  # LPOP - ดึงจากต้น list
  let taskStr = await q.client.lpop(q.queueKey)
  if taskStr.len == 0 or taskStr == "nil":
    return newJNull()
  return parseJson(taskStr)

proc peek(q: TaskQueue, count = 1): Future[seq[JsonNode]] {.async.} =
  # LRANGE - ดูโดยไม่ลบ
  let items = await q.client.lrange(q.queueKey, 0, count - 1)
  var result: seq[JsonNode] = @[]
  for item in items:
    if item.len > 0:
      result.add(parseJson(item))
  return result

proc queueLength(q: TaskQueue): Future[int] {.async.} =
  let len = await q.client.llen(q.queueKey)
  return int(len)

proc blockingDequeue(q: TaskQueue, timeout = 0): Future[JsonNode] {.async.} =
  # BLPOP - blocking pop (รอจนมีข้อมูล)
  let result = await q.client.blpop(@[q.queueKey], timeout)
  if result.len == 0:
    return newJNull()
  return parseJson(result[1])  # result[0] = key name, result[1] = value

# Stack operations (LIFO)
proc push(q: TaskQueue, item: JsonNode) {.async.} =
  # LPUSH - เพิ่มต้น list (stack push)
  discard await q.client.lpush(q.queueKey, @[$item])

proc pop(q: TaskQueue): Future[JsonNode] {.async.} =
  # LPOP - pop จากต้น (stack pop)
  let val = await q.client.lpop(q.queueKey)
  if val.len == 0 or val == "nil":
    return newJNull()
  return parseJson(val)

proc listDemo() {.async.} =
  let queue = await newTaskQueue("task:email")
  
  echo "=== Task Queue Demo ==="
  
  # Enqueue tasks
  for i in 1..5:
    await queue.enqueue(%*{
      "id": i,
      "type": "send_email",
      "to": &"user{i}@example.com",
      "subject": &"Task #{i}",
      "createdAt": $now()
    })
  
  echo &"Queue length: {await queue.queueLength()}"
  
  # Peek at first 3
  let peek = await queue.peek(3)
  echo "\nFirst 3 tasks:"
  for task in peek:
    echo &"  - {task[\"id\"].getInt()}: {task[\"to\"].getStr()}"
  
  # Process tasks
  echo "\nProcessing tasks:"
  while true:
    let task = await queue.dequeue()
    if task.kind == JNull:
      break
    echo &"  Processing: {task[\"id\"].getInt()} -> {task[\"to\"].getStr()}"
    await sleepAsync(100)  # simulate work
  
  echo &"\nQueue empty: {await queue.queueLength() == 0}"
  
  # Recent activity (capped list)
  echo "\n=== Recent Activity Log ==="
  let logKey = "activity:log"
  
  for i in 1..10:
    let entry = %*{"action": &"action_{i}", "timestamp": epochTime()}
    discard await queue.client.lpush(logKey, @[$entry])
    # Keep only last 5 entries
    await queue.client.ltrim(logKey, 0, 4)
  
  let recentLog = await queue.client.lrange(logKey, 0, -1)
  echo "Recent 5 activities:"
  for entry in recentLog:
    let data = parseJson(entry)
    echo &"  {data[\"action\"].getStr()}"

waitFor listDemo()
```

---

## Step 230: Sets และ Sorted Sets

```nim
# file: src/redis_sets.nim
import redis, asyncdispatch, strformat, json, strutils

proc setsDemo() {.async.} =
  let r = await openAsync("localhost", Port(6379))
  defer: r.close()
  
  echo "=== Sets Demo ==="
  
  # SADD - เพิ่มสมาชิก
  let added = await r.sadd("tags:article:1", @["nim", "programming", "backend", "tutorial"])
  echo &"Added {added} tags"
  
  await r.sadd("tags:article:2", @["nim", "async", "performance", "tutorial"])
  await r.sadd("tags:article:3", @["python", "backend", "programming"])
  
  # SMEMBERS - ดูทุกสมาชิก
  let tags = await r.smembers("tags:article:1")
  echo &"Article 1 tags: {tags}"
  
  # SISMEMBER - ตรวจสอบการมีอยู่
  let hasTag = await r.sismember("tags:article:1", "nim")
  echo &"Has 'nim' tag: {hasTag == 1}"
  
  # SCARD - จำนวนสมาชิก
  let count = await r.scard("tags:article:1")
  echo &"Tag count: {count}"
  
  # SINTER - Intersection (tags ที่ทั้งสอง article มี)
  let common = await r.sinter(@["tags:article:1", "tags:article:2"])
  echo &"Common tags (1 & 2): {common}"
  
  # SUNION - Union (ทุก tags รวมกัน)
  let allTags = await r.sunion(@["tags:article:1", "tags:article:2", "tags:article:3"])
  echo &"All unique tags: {allTags}"
  
  # SDIFF - Difference (tags ที่อยู่ใน 1 แต่ไม่อยู่ใน 2)
  let diff = await r.sdiff(@["tags:article:1", "tags:article:2"])
  echo &"Tags in 1 but not 2: {diff}"
  
  # SREM - ลบสมาชิก
  let removed = await r.srem("tags:article:1", @["tutorial"])
  echo &"Removed: {removed}"

proc sortedSetsDemo() {.async.} =
  let r = await openAsync("localhost", Port(6379))
  defer: r.close()
  
  echo "\n=== Sorted Sets (Leaderboard) ==="
  
  # ZADD - เพิ่ม member พร้อม score
  let users = @[
    ("alice", 9500.0),
    ("bob", 8750.0),
    ("charlie", 9200.0),
    ("diana", 9800.0),
    ("eve", 7300.0),
    ("frank", 8100.0)
  ]
  
  for (username, score) in users:
    discard await r.zadd("leaderboard", @[(score, username)])
  
  # ZRANK - อันดับ (0-based, ascending)
  let rank = await r.zrank("leaderboard", "alice")
  echo &"Alice's rank (ascending): {rank}"
  
  # ZREVRANK - อันดับจากบน (descending)
  let revRank = await r.zrevrank("leaderboard", "diana")
  echo &"Diana's rank (1st = 0): {revRank}"
  
  # ZSCORE - ดู score
  let aliceScore = await r.zscore("leaderboard", "alice")
  echo &"Alice's score: {aliceScore}"
  
  # ZREVRANGE - top 3
  let top3 = await r.zrevrange("leaderboard", 0, 2, withScores = true)
  echo "\nTop 3 leaderboard:"
  var pos = 1
  var i = 0
  while i < top3.len:
    echo &"  #{pos}: {top3[i]} - {top3[i+1]}"
    inc pos
    i += 2
  
  # ZINCRBY - เพิ่ม score
  let newScore = await r.zincrby("leaderboard", 500, "alice")
  echo &"\nAlice's new score after +500: {newScore}"
  
  # ZRANGEBYSCORE - หา users ที่ score ระหว่าง range
  let highScorers = await r.zrangebyscore("leaderboard", "9000", "+inf")
  echo &"Users with score >= 9000: {highScorers}"
  
  # ZCOUNT - นับ members ใน score range
  let count = await r.zcount("leaderboard", "8000", "9000")
  echo &"Users with score 8000-9000: {count}"

proc rateLimitWithSortedSet() {.async.} =
  let r = await openAsync("localhost", Port(6379))
  defer: r.close()
  
  echo "\n=== Rate Limiting with Sorted Sets ==="
  
  let userId = "user123"
  let windowSeconds = 60  # 1 minute window
  let maxRequests = 10
  
  for i in 1..15:
    let now = epochTime()
    let windowStart = now - windowSeconds.float
    let key = &"ratelimit:{userId}"
    
    # ลบ entries เก่ากว่า window
    await r.zremrangebyscore(key, "-inf", $windowStart)
    
    # นับ requests ใน window
    let count = await r.zcard(key)
    
    if int(count) >= maxRequests:
      echo &"Request #{i}: RATE LIMITED (count: {count})"
      continue
    
    # เพิ่ม request ใหม่
    discard await r.zadd(key, @[(now, &"req_{i}")])
    await r.expire(key, windowSeconds)
    
    echo &"Request #{i}: OK (count: {count + 1})"
    await sleepAsync(50)

proc setsMain() {.async.} =
  await setsDemo()
  await sortedSetsDemo()
  await rateLimitWithSortedSet()

waitFor setsMain()
```

---

## Step 231: Hashes

```nim
# file: src/redis_hashes.nim
import redis, asyncdispatch, strformat, json, strutils, tables

type
  UserProfile = object
    id: string
    username: string
    email: string
    fullName: string
    age: int
    createdAt: string
    active: bool

proc userToHash(user: UserProfile): seq[(string, string)] =
  @[
    ("id", user.id),
    ("username", user.username),
    ("email", user.email),
    ("fullName", user.fullName),
    ("age", $user.age),
    ("createdAt", user.createdAt),
    ("active", $user.active)
  ]

proc hashToUser(data: Table[string, string]): UserProfile =
  UserProfile(
    id: data.getOrDefault("id", ""),
    username: data.getOrDefault("username", ""),
    email: data.getOrDefault("email", ""),
    fullName: data.getOrDefault("fullName", ""),
    age: parseInt(data.getOrDefault("age", "0")),
    createdAt: data.getOrDefault("createdAt", ""),
    active: data.getOrDefault("active", "false") == "true"
  )

type
  UserRepository = ref object
    redis: AsyncRedis

proc newUserRepository(): Future[UserRepository] {.async.} =
  UserRepository(redis: await openAsync("localhost", Port(6379)))

proc saveUser(repo: UserRepository, user: UserProfile) {.async.} =
  let key = &"user:{user.id}"
  let fields = userToHash(user)
  await repo.redis.hmset(key, fields)
  
  # เพิ่มใน users index
  discard await repo.redis.sadd("users:all", @[user.id])
  
  # Index by username
  await repo.redis.setk(&"username:{user.username}", user.id)
  
  echo &"Saved user: {user.username}"

proc getUser(repo: UserRepository, userId: string): Future[UserProfile] {.async.} =
  let key = &"user:{userId}"
  let data = await repo.redis.hgetall(key)
  
  if data.len == 0:
    raise newException(KeyError, &"User {userId} not found")
  
  var dataTable = initTable[string, string]()
  var i = 0
  while i < data.len - 1:
    dataTable[data[i]] = data[i + 1]
    i += 2
  
  return hashToUser(dataTable)

proc updateUser(repo: UserRepository, userId: string, 
                fields: seq[(string, string)]) {.async.} =
  let key = &"user:{userId}"
  await repo.redis.hmset(key, fields)

proc getUserField(repo: UserRepository, userId, field: string): Future[string] {.async.} =
  let key = &"user:{userId}"
  return await repo.redis.hget(key, field)

proc deleteUser(repo: UserRepository, userId: string) {.async.} =
  let user = await repo.getUser(userId)
  
  # ลบ username index
  await repo.redis.del(@[&"username:{user.username}"])
  
  # ลบออกจาก users index
  await repo.redis.srem("users:all", @[userId])
  
  # ลบ hash
  await repo.redis.del(@[&"user:{userId}"])
  
  echo &"Deleted user: {userId}"

proc getUserByUsername(repo: UserRepository, username: string): Future[UserProfile] {.async.} =
  let userId = await repo.redis.get(&"username:{username}")
  if userId.len == 0 or userId == "nil":
    raise newException(KeyError, &"Username {username} not found")
  return await repo.getUser(userId)

proc listAllUsers(repo: UserRepository): Future[seq[UserProfile]] {.async.} =
  let userIds = await repo.redis.smembers("users:all")
  var users: seq[UserProfile] = @[]
  
  for id in userIds:
    try:
      let user = await repo.getUser(id)
      users.add(user)
    except KeyError:
      discard
  
  return users

proc hashDemo() {.async.} =
  let repo = await newUserRepository()
  
  # สร้าง users
  let users = @[
    UserProfile(id: "1", username: "alice", email: "alice@example.com",
                fullName: "Alice Smith", age: 28, createdAt: "2024-01-01", active: true),
    UserProfile(id: "2", username: "bob", email: "bob@example.com",
                fullName: "Bob Jones", age: 35, createdAt: "2024-02-15", active: true),
    UserProfile(id: "3", username: "charlie", email: "charlie@example.com",
                fullName: "Charlie Brown", age: 22, createdAt: "2024-03-10", active: false)
  ]
  
  for user in users:
    await repo.saveUser(user)
  
  # ดึงโดย ID
  let alice = await repo.getUser("1")
  echo &"\nUser by ID: {alice.username} ({alice.email})"
  
  # ดึงโดย username
  let bob = await repo.getUserByUsername("bob")
  echo &"User by username: {bob.fullName}, age {bob.age}"
  
  # ดูแค่ field เดียว
  let email = await repo.getUserField("1", "email")
  echo &"Alice's email: {email}"
  
  # Update partial fields
  await repo.updateUser("1", @[
    ("age", "29"),
    ("email", "alice.updated@example.com")
  ])
  
  let updatedAlice = await repo.getUser("1")
  echo &"Updated Alice: age={updatedAlice.age}, email={updatedAlice.email}"
  
  # List all
  let allUsers = await repo.listAllUsers()
  echo &"\nAll users ({allUsers.len}):"
  for u in allUsers:
    echo &"  - {u.username} ({u.email})"

waitFor hashDemo()
```

---

## Step 232: Pub/Sub Pattern

```nim
# file: src/redis_pubsub.nim
import redis, asyncdispatch, strformat, json, os, asyncfutures

type
  MessageHandler = proc(channel, message: string) {.async.}

  PubSubClient = ref object
    publisher: AsyncRedis
    subscriber: AsyncRedis
    subscriptions: seq[string]
    handlers: seq[(string, MessageHandler)]

proc newPubSubClient(): Future[PubSubClient] {.async.} =
  PubSubClient(
    publisher: await openAsync("localhost", Port(6379)),
    subscriber: await openAsync("localhost", Port(6379)),
    subscriptions: @[],
    handlers: @[]
  )

proc publish(client: PubSubClient, channel, message: string): Future[int64] {.async.} =
  let count = await client.publisher.publish(channel, message)
  echo &"[PUBLISH] {channel}: {message} (received by {count} subscribers)"
  return count

proc subscribe(client: PubSubClient, channels: seq[string], 
               handler: MessageHandler) {.async.} =
  for ch in channels:
    client.subscriptions.add(ch)
    client.handlers.add((ch, handler))
  
  await client.subscriber.subscribe(channels)
  echo &"[SUBSCRIBE] Subscribed to: {channels}"

proc psubscribe(client: PubSubClient, pattern: string,
                handler: MessageHandler) {.async.} =
  client.handlers.add((pattern, handler))
  await client.subscriber.psubscribe(@[pattern])
  echo &"[PSUBSCRIBE] Pattern: {pattern}"

proc startListening(client: PubSubClient) {.async.} =
  echo "[LISTENER] Starting message loop..."
  
  while true:
    let (msgType, channel, message) = await client.subscriber.readMessage()
    
    case msgType
    of "subscribe":
      echo &"[SUBSCRIBED] {channel}"
    of "message":
      for (ch, handler) in client.handlers:
        if ch == channel:
          asyncCheck handler(channel, message)
    of "pmessage":
      # Pattern message: channel = pattern, message = "actual_channel|msg"
      for (pattern, handler) in client.handlers:
        asyncCheck handler(channel, message)
    of "unsubscribe":
      echo &"[UNSUBSCRIBED] {channel}"
    else:
      echo &"[UNKNOWN] {msgType}: {channel} = {message}"

proc pubSubDemo() {.async.} =
  let client = await newPubSubClient()
  
  # Define handlers
  proc notificationHandler(channel, message: string) {.async.} =
    let data = parseJson(message)
    echo &"[NOTIFICATION] Channel: {channel}"
    echo &"  Type: {data[\"type\"].getStr()}"
    echo &"  Message: {data[\"message\"].getStr()}"
  
  proc analyticsHandler(channel, message: string) {.async.} =
    echo &"[ANALYTICS] {channel}: {message}"
  
  # Subscribe
  await client.subscribe(
    @["notifications:user:1", "notifications:user:2"],
    notificationHandler
  )
  
  await client.subscribe(
    @["analytics:events"],
    analyticsHandler
  )
  
  # Pattern subscribe - รับทุก notifications
  await client.psubscribe("notifications:*", proc(ch, msg: string) {.async.} =
    echo &"[PATTERN MATCH] {ch}: {msg}"
  )
  
  # Start listening ใน background
  asyncCheck client.startListening()
  
  # Publish some messages
  await sleepAsync(100)
  
  await client.publish("notifications:user:1", $(%*{
    "type": "info",
    "message": "You have a new message",
    "from": "alice"
  }))
  
  await sleepAsync(50)
  
  await client.publish("notifications:user:2", $(%*{
    "type": "warning",
    "message": "Your password will expire soon"
  }))
  
  await sleepAsync(50)
  
  await client.publish("analytics:events", $(%*{
    "event": "page_view",
    "page": "/home",
    "userId": 123
  }))
  
  await sleepAsync(500)
  echo "Demo complete"

waitFor pubSubDemo()
```

---

## Step 233: Redis เป็น Session Store

```nim
# file: src/redis_sessions.nim
import redis, asyncdispatch, strformat, json
import times, random, base64, sha256, strutils

type
  Session = object
    id: string
    userId: string
    username: string
    roles: seq[string]
    createdAt: DateTime
    lastAccessed: DateTime
    data: JsonNode

  SessionStore = ref object
    redis: AsyncRedis
    prefix: string
    ttl: int  # seconds

proc newSessionStore(prefix = "session", ttl = 3600): Future[SessionStore] {.async.} =
  SessionStore(
    redis: await openAsync("localhost", Port(6379)),
    prefix: prefix,
    ttl: ttl
  )

proc generateSessionId(): string =
  randomize()
  let random_bytes = $rand(int.high)
  let timestamp = $epochTime()
  let data = random_bytes & timestamp
  return base64.encode(data)[0..31]  # 32 char session ID

proc sessionKey(store: SessionStore, sessionId: string): string =
  &"{store.prefix}:{sessionId}"

proc createSession(store: SessionStore, userId, username: string, 
                   roles: seq[string] = @["user"],
                   extraData = newJObject()): Future[string] {.async.} =
  let sessionId = generateSessionId()
  let key = store.sessionKey(sessionId)
  
  let session = %*{
    "sessionId": sessionId,
    "userId": userId,
    "username": username,
    "roles": roles,
    "createdAt": $now(),
    "lastAccessed": $now(),
    "data": extraData
  }
  
  # เก็บ session เป็น hash
  await store.redis.hmset(key, @[
    ("sessionId", sessionId),
    ("userId", userId),
    ("username", username),
    ("roles", $(%roles)),
    ("createdAt", $now()),
    ("lastAccessed", $now()),
    ("data", $extraData)
  ])
  
  # Set TTL
  await store.redis.expire(key, store.ttl)
  
  # เพิ่มใน user's sessions index
  discard await store.redis.sadd(&"user_sessions:{userId}", @[sessionId])
  
  echo &"[SESSION] Created: {sessionId} for user {username}"
  return sessionId

proc getSession(store: SessionStore, sessionId: string): Future[JsonNode] {.async.} =
  let key = store.sessionKey(sessionId)
  let data = await store.redis.hgetall(key)
  
  if data.len == 0:
    return newJNull()
  
  var sessionData = newJObject()
  var i = 0
  while i < data.len - 1:
    sessionData[data[i]] = %data[i + 1]
    i += 2
  
  # Update last accessed
  await store.redis.hset(key, "lastAccessed", $now())
  
  # Refresh TTL
  await store.redis.expire(key, store.ttl)
  
  return sessionData

proc validateSession(store: SessionStore, sessionId: string): Future[bool] {.async.} =
  let key = store.sessionKey(sessionId)
  let ttl = await store.redis.ttl(key)
  return ttl > 0

proc deleteSession(store: SessionStore, sessionId: string) {.async.} =
  let key = store.sessionKey(sessionId)
  
  # หา userId ก่อน
  let userId = await store.redis.hget(key, "userId")
  
  # ลบ session
  await store.redis.del(@[key])
  
  # ลบออกจาก user's sessions
  if userId.len > 0:
    await store.redis.srem(&"user_sessions:{userId}", @[sessionId])
  
  echo &"[SESSION] Deleted: {sessionId}"

proc getUserSessions(store: SessionStore, userId: string): Future[seq[string]] {.async.} =
  let sessionIds = await store.redis.smembers(&"user_sessions:{userId}")
  
  # Filter out expired sessions
  var activeSessions: seq[string] = @[]
  for sid in sessionIds:
    if await store.validateSession(sid):
      activeSessions.add(sid)
    else:
      # Clean up expired session from index
      await store.redis.srem(&"user_sessions:{userId}", @[sid])
  
  return activeSessions

proc invalidateAllUserSessions(store: SessionStore, userId: string) {.async.} =
  let sessions = await store.getUserSessions(userId)
  for sessionId in sessions:
    await store.deleteSession(sessionId)
  echo &"[SESSION] Invalidated all sessions for user {userId}"

proc updateSessionData(store: SessionStore, sessionId: string, 
                       key: string, value: JsonNode) {.async.} =
  let sKey = store.sessionKey(sessionId)
  
  # Get existing data
  let dataStr = await store.redis.hget(sKey, "data")
  var data = if dataStr.len > 0 and dataStr != "nil": 
    parseJson(dataStr) 
  else: 
    newJObject()
  
  data[key] = value
  await store.redis.hset(sKey, "data", $data)

proc sessionDemo() {.async.} =
  let store = await newSessionStore(ttl = 300)
  
  # สร้าง sessions
  let sess1 = await store.createSession("1", "alice", @["user", "admin"])
  let sess2 = await store.createSession("1", "alice")  # Second session
  let sess3 = await store.createSession("2", "bob")
  
  # ดึง session
  let session = await store.getSession(sess1)
  if session.kind != JNull:
    echo &"\nSession data:"
    echo &"  User: {session[\"username\"].getStr()}"
    echo &"  Roles: {session[\"roles\"].getStr()}"
    echo &"  Created: {session[\"createdAt\"].getStr()}"
  
  # Validate
  let valid = await store.validateSession(sess1)
  echo &"\nSession valid: {valid}"
  
  # Get user's active sessions
  let aliceSessions = await store.getUserSessions("1")
  echo &"Alice's active sessions: {aliceSessions.len}"
  
  # Update session data
  await store.updateSessionData(sess1, "cart_items", %*["item1", "item2"])
  
  let updatedSession = await store.getSession(sess1)
  if updatedSession.kind != JNull:
    echo &"Cart: {updatedSession[\"data\"].getStr()}"
  
  # Delete single session (logout)
  await store.deleteSession(sess2)
  
  let aliceSessionsAfter = await store.getUserSessions("1")
  echo &"Alice's sessions after logout: {aliceSessionsAfter.len}"
  
  # Invalidate all (change password scenario)
  await store.invalidateAllUserSessions("1")
  
  let aliceSessionsFinal = await store.getUserSessions("1")
  echo &"Alice's sessions after password change: {aliceSessionsFinal.len}"

waitFor sessionDemo()
```

---

## Step 234: Redis เป็น Cache Layer

```nim
# file: src/redis_cache.nim
import redis, asyncdispatch, strformat, json
import times, strutils, math

type
  CacheOptions = object
    ttl: int              # seconds
    keyPrefix: string
    compression: bool

  CacheStats = object
    hits: int
    misses: int
    sets: int
    deletes: int

  Cache = ref object
    redis: AsyncRedis
    options: CacheOptions
    stats: CacheStats

proc newCache(opts = CacheOptions(ttl: 300, keyPrefix: "cache")): Future[Cache] {.async.} =
  Cache(
    redis: await openAsync("localhost", Port(6379)),
    options: opts,
    stats: CacheStats()
  )

proc buildKey(cache: Cache, key: string): string =
  if cache.options.keyPrefix.len > 0:
    return &"{cache.options.keyPrefix}:{key}"
  return key

proc get(cache: Cache, key: string): Future[string] {.async.} =
  let value = await cache.redis.get(cache.buildKey(key))
  
  if value.len == 0 or value == "nil":
    inc cache.stats.misses
    return ""
  
  inc cache.stats.hits
  return value

proc set(cache: Cache, key, value: string, ttl = -1) {.async.} =
  let actualTtl = if ttl > 0: ttl else: cache.options.ttl
  await cache.redis.setex(cache.buildKey(key), actualTtl, value)
  inc cache.stats.sets

proc del(cache: Cache, key: string) {.async.} =
  await cache.redis.del(@[cache.buildKey(key)])
  inc cache.stats.deletes

proc getOrCompute(cache: Cache, key: string, 
                  compute: proc(): Future[string] {.async.},
                  ttl = -1): Future[string] {.async.} =
  # Cache-aside pattern
  let cached = await cache.get(key)
  if cached.len > 0:
    return cached
  
  let value = await compute()
  await cache.set(key, value, ttl)
  return value

proc invalidatePattern(cache: Cache, pattern: string) {.async.} =
  let fullPattern = &"{cache.options.keyPrefix}:{pattern}"
  let keys = await cache.redis.keys(fullPattern)
  
  if keys.len > 0:
    await cache.redis.del(keys)
    cache.stats.deletes += keys.len
    echo &"[CACHE] Invalidated {keys.len} keys matching '{pattern}'"

proc getStats(cache: Cache): JsonNode =
  let total = cache.stats.hits + cache.stats.misses
  let hitRate = if total > 0: (cache.stats.hits.float / total.float * 100) else: 0.0
  
  %*{
    "hits": cache.stats.hits,
    "misses": cache.stats.misses,
    "sets": cache.stats.sets,
    "deletes": cache.stats.deletes,
    "hitRate": &"{hitRate:.1f}%"
  }

# Fragment Caching สำหรับ expensive operations
proc cacheUserProfile(cache: Cache, userId: string,
                       fetchFromDb: proc(id: string): Future[JsonNode] {.async.}): Future[JsonNode] {.async.} =
  let key = &"user:{userId}:profile"
  
  let cached = await cache.get(key)
  if cached.len > 0:
    return parseJson(cached)
  
  let profile = await fetchFromDb(userId)
  await cache.set(key, $profile, ttl = 600)
  return profile

# Multi-level cache
type
  TwoLevelCache = ref object
    l1: Cache  # Short TTL, frequently accessed
    l2: Cache  # Longer TTL, less frequent

proc newTwoLevelCache(): Future[TwoLevelCache] {.async.} =
  TwoLevelCache(
    l1: await newCache(CacheOptions(ttl: 60, keyPrefix: "l1")),
    l2: await newCache(CacheOptions(ttl: 3600, keyPrefix: "l2"))
  )

proc get(cache: TwoLevelCache, key: string): Future[string] {.async.} =
  # Check L1 first
  let l1Val = await cache.l1.get(key)
  if l1Val.len > 0:
    return l1Val
  
  # Check L2
  let l2Val = await cache.l2.get(key)
  if l2Val.len > 0:
    # Populate L1
    await cache.l1.set(key, l2Val)
    return l2Val
  
  return ""

proc set(cache: TwoLevelCache, key, value: string) {.async.} =
  await cache.l1.set(key, value)
  await cache.l2.set(key, value)

proc cacheDemo() {.async.} =
  let cache = await newCache(CacheOptions(
    ttl: 60,
    keyPrefix: "app",
    compression: false
  ))
  
  echo "=== Cache Demo ==="
  
  # Simulate database queries with caching
  var dbQueryCount = 0
  
  proc fetchUserFromDb(userId: string): Future[string] {.async.} =
    inc dbQueryCount
    echo &"  [DB QUERY #{dbQueryCount}] Fetching user {userId}..."
    await sleepAsync(50)  # Simulate DB latency
    return $(%*{
      "id": userId,
      "name": &"User {userId}",
      "email": &"user{userId}@example.com",
      "createdAt": $now()
    })
  
  # First access - cache miss
  echo "\nFirst access (cache miss):"
  let user1 = await cache.getOrCompute("user:1", proc(): Future[string] {.async.} =
    return await fetchUserFromDb("1")
  )
  echo &"  Got: {parseJson(user1)[\"name\"].getStr()}"
  
  # Second access - cache hit
  echo "\nSecond access (cache hit):"
  let user1Cached = await cache.getOrCompute("user:1", proc(): Future[string] {.async.} =
    return await fetchUserFromDb("1")
  )
  echo &"  Got: {parseJson(user1Cached)[\"name\"].getStr()}"
  
  # Multiple users
  echo "\nCaching multiple users:"
  for i in 2..5:
    let user = await cache.getOrCompute(&"user:{i}", proc(): Future[string] {.async.} =
      return await fetchUserFromDb($i)
    )
    discard user
  
  # Access cached users
  for i in 2..5:
    let user = await cache.getOrCompute(&"user:{i}", proc(): Future[string] {.async.} =
      return await fetchUserFromDb($i)  # won't be called
    )
    discard user
  
  echo &"\nDB queries made: {dbQueryCount} (should be 5)"
  
  # Stats
  let stats = cache.getStats()
  echo &"\nCache Stats:"
  echo &"  Hits: {stats[\"hits\"].getInt()}"
  echo &"  Misses: {stats[\"misses\"].getInt()}"
  echo &"  Hit Rate: {stats[\"hitRate\"].getStr()}"
  
  # Invalidate specific user
  await cache.del("user:1")
  echo "\nAfter deleting user:1 from cache:"
  let user1Again = await cache.getOrCompute("user:1", proc(): Future[string] {.async.} =
    return await fetchUserFromDb("1")
  )
  echo &"  Fetched fresh: {parseJson(user1Again)[\"name\"].getStr()}"

waitFor cacheDemo()
```

---

## Step 235: Redis Rate Limiting

```nim
# file: src/redis_ratelimit.nim
import redis, asyncdispatch, strformat, json, times, math

type
  RateLimitStrategy = enum
    rlFixed = "fixed"        # Fixed window
    rlSliding = "sliding"    # Sliding window
    rlToken = "token_bucket" # Token bucket

  RateLimitResult = object
    allowed: bool
    remaining: int
    resetAt: int64          # Unix timestamp
    retryAfter: int         # seconds

  RateLimiter = ref object
    redis: AsyncRedis
    strategy: RateLimitStrategy
    maxRequests: int
    windowSeconds: int

proc newRateLimiter(strategy = rlSliding, maxRequests = 100, 
                    windowSeconds = 60): Future[RateLimiter] {.async.} =
  RateLimiter(
    redis: await openAsync("localhost", Port(6379)),
    strategy: strategy,
    maxRequests: maxRequests,
    windowSeconds: windowSeconds
  )

proc fixedWindowLimit(rl: RateLimiter, key: string): Future[RateLimitResult] {.async.} =
  let now = epochTime()
  let windowStart = int64(now / rl.windowSeconds.float) * rl.windowSeconds
  let windowKey = &"ratelimit:fixed:{key}:{windowStart}"
  
  let count = await rl.redis.incr(windowKey)
  
  if count == 1:
    await rl.redis.expire(windowKey, rl.windowSeconds)
  
  let resetAt = windowStart + rl.windowSeconds
  let remaining = max(0, rl.maxRequests - int(count))
  
  return RateLimitResult(
    allowed: int(count) <= rl.maxRequests,
    remaining: remaining,
    resetAt: resetAt,
    retryAfter: if int(count) > rl.maxRequests: int(resetAt - int64(now)) else: 0
  )

proc slidingWindowLimit(rl: RateLimiter, key: string): Future[RateLimitResult] {.async.} =
  let now = epochTime()
  let windowStart = now - rl.windowSeconds.float
  let windowKey = &"ratelimit:sliding:{key}"
  
  # ลบ entries เก่า
  await rl.redis.zremrangebyscore(windowKey, "-inf", $windowStart)
  
  # นับ requests ใน window
  let count = await rl.redis.zcard(windowKey)
  let allowed = int(count) < rl.maxRequests
  
  if allowed:
    # เพิ่ม request นี้
    let requestId = &"{now}:{rand(999999)}"
    discard await rl.redis.zadd(windowKey, @[(now, requestId)])
    await rl.redis.expire(windowKey, rl.windowSeconds)
  
  return RateLimitResult(
    allowed: allowed,
    remaining: max(0, rl.maxRequests - int(count) - (if allowed: 1 else: 0)),
    resetAt: int64(now + rl.windowSeconds.float),
    retryAfter: if not allowed: 1 else: 0
  )

proc tokenBucketLimit(rl: RateLimiter, key: string): Future[RateLimitResult] {.async.} =
  # Token bucket algorithm
  let tokensKey = &"ratelimit:tokens:{key}"
  let lastRefillKey = &"ratelimit:lastrefill:{key}"
  
  let now = epochTime()
  let refillRate = rl.maxRequests.float / rl.windowSeconds.float  # tokens per second
  
  # ดู tokens และ last refill time
  let tokensStr = await rl.redis.get(tokensKey)
  let lastRefillStr = await rl.redis.get(lastRefillKey)
  
  var tokens = if tokensStr.len > 0 and tokensStr != "nil": 
    parseFloat(tokensStr) 
  else: 
    rl.maxRequests.float
  
  let lastRefill = if lastRefillStr.len > 0 and lastRefillStr != "nil": 
    parseFloat(lastRefillStr) 
  else: 
    now
  
  # Refill tokens
  let elapsed = now - lastRefill
  tokens = min(rl.maxRequests.float, tokens + elapsed * refillRate)
  
  let allowed = tokens >= 1.0
  
  if allowed:
    tokens -= 1.0
  
  # บันทึก state
  await rl.redis.setex(tokensKey, rl.windowSeconds * 2, $tokens)
  await rl.redis.setex(lastRefillKey, rl.windowSeconds * 2, $now)
  
  return RateLimitResult(
    allowed: allowed,
    remaining: int(tokens),
    resetAt: int64(now + (1.0 / refillRate)),
    retryAfter: if not allowed: int(ceil(1.0 / refillRate)) else: 0
  )

proc checkLimit(rl: RateLimiter, identifier: string): Future[RateLimitResult] {.async.} =
  case rl.strategy
  of rlFixed:
    return await rl.fixedWindowLimit(identifier)
  of rlSliding:
    return await rl.slidingWindowLimit(identifier)
  of rlToken:
    return await rl.tokenBucketLimit(identifier)

proc rateLimitDemo() {.async.} =
  echo "=== Rate Limiting Demo ==="
  
  let limiter = await newRateLimiter(
    strategy = rlSliding,
    maxRequests = 5,
    windowSeconds = 10
  )
  
  let userId = "user:123"
  
  echo "\nSending 8 requests (limit is 5 per 10 seconds):"
  for i in 1..8:
    let result = await limiter.checkLimit(userId)
    
    if result.allowed:
      echo &"  Request #{i}: ALLOWED (remaining: {result.remaining})"
    else:
      echo &"  Request #{i}: DENIED (retry after {result.retryAfter}s)"
    
    await sleepAsync(200)
  
  echo "\nWaiting 11 seconds for window to reset..."
  await sleepAsync(2000)  # In demo, just wait 2 seconds
  
  let result = await limiter.checkLimit(userId)
  echo &"After wait: {'ALLOWED' if result.allowed else 'DENIED'} (remaining: {result.remaining})"
  
  # Token bucket demo
  echo "\n=== Token Bucket Demo ==="
  let tokenLimiter = await newRateLimiter(
    strategy = rlToken,
    maxRequests = 10,
    windowSeconds = 60
  )
  
  for i in 1..12:
    let r = await tokenLimiter.checkLimit("api:client1")
    echo &"  Request #{i}: {'OK' if r.allowed else 'RATE LIMITED'} (tokens: {r.remaining})"

waitFor rateLimitDemo()
```

---

## Step 236: Transaction และ Pipeline

```nim
# file: src/redis_transactions.nim
import redis, asyncdispatch, strformat, json

proc transactionDemo() {.async.} =
  let r = await openAsync("localhost", Port(6379))
  defer: r.close()
  
  echo "=== Redis Transactions ==="
  
  # Setup initial values
  await r.setk("balance:alice", "1000")
  await r.setk("balance:bob", "500")
  
  # Transfer money atomically using MULTI/EXEC
  proc transfer(from_user, to_user: string, amount: int) {.async.} =
    # ตรวจสอบยอดก่อน
    let fromBalance = parseInt(await r.get(&"balance:{from_user}"))
    
    if fromBalance < amount:
      echo &"Transfer failed: insufficient balance ({fromBalance} < {amount})"
      return
    
    # MULTI - start transaction
    await r.multi()
    
    # Queue commands
    await r.decrby(&"balance:{from_user}", amount)
    await r.incrby(&"balance:{to_user}", amount)
    
    # EXEC - execute atomically
    let results = await r.exec()
    
    echo &"Transfer {amount} from {from_user} to {to_user}:"
    echo &"  {from_user} balance: {results[0]}"
    echo &"  {to_user} balance: {results[1]}"
  
  await transfer("alice", "bob", 200)
  
  let aliceBal = await r.get("balance:alice")
  let bobBal = await r.get("balance:bob")
  echo &"\nFinal balances: alice={aliceBal}, bob={bobBal}"
  
  # Optimistic locking with WATCH
  echo "\n=== Optimistic Locking with WATCH ==="
  
  proc safeUpdate(key, expectedValue, newValue: string): Future[bool] {.async.} =
    await r.watch(@[key])
    
    let current = await r.get(key)
    if current != expectedValue:
      await r.unwatch()
      return false
    
    await r.multi()
    await r.setk(key, newValue)
    
    let results = await r.exec()
    return results.len > 0
  
  await r.setk("version", "1")
  
  let success = await safeUpdate("version", "1", "2")
  echo &"Update 1->2: {success}"
  
  let success2 = await safeUpdate("version", "1", "3")  # Should fail
  echo &"Update 1->3 (optimistic fail): {success2}"

proc pipelineDemo() {.async.} =
  let r = await openAsync("localhost", Port(6379))
  defer: r.close()
  
  echo "\n=== Pipeline Demo ==="
  
  # Without pipeline - แต่ละ command รอ response
  let startTime = epochTime()
  for i in 0..99:
    await r.setk(&"item:{i}", &"value_{i}")
  let withoutPipeline = epochTime() - startTime
  echo &"Without pipeline: {withoutPipeline:.3f}s"
  
  # With pipeline - ส่ง commands batch
  let startTime2 = epochTime()
  await r.startPipelining()
  
  for i in 100..199:
    await r.setk(&"item:{i}", &"value_{i}")
  
  discard await r.setk("done", "true")
  let results = await r.endPipelining()
  
  let withPipeline = epochTime() - startTime2
  echo &"With pipeline: {withPipeline:.3f}s"
  echo &"Speedup: {withoutPipeline / withPipeline:.1f}x"

proc transMain() {.async.} =
  await transactionDemo()
  await pipelineDemo()

waitFor transMain()
```

---

## Step 237: Real-World Session + Caching System

สร้างระบบ complete ที่รวม sessions และ caching:

```nim
# file: src/complete_session_cache.nim
import redis, asyncdispatch, asynchttpserver, httpcore
import strformat, json, times, random, strutils, tables, base64

type
  AppContext = ref object
    redis: AsyncRedis
    sessionTtl: int
    cacheTtl: int

proc newAppContext(): Future[AppContext] {.async.} =
  AppContext(
    redis: await openAsync("localhost", Port(6379)),
    sessionTtl: 3600,   # 1 hour sessions
    cacheTtl: 300       # 5 minute cache
  )

# ====== Session Management ======

proc generateToken(): string =
  randomize()
  let data = &"{rand(int.high)}{epochTime()}"
  return base64.encode(data)[0..31]

proc createSession(ctx: AppContext, userId, username: string): Future[string] {.async.} =
  let token = generateToken()
  let key = &"sess:{token}"
  
  await ctx.redis.hmset(key, @[
    ("userId", userId),
    ("username", username),
    ("createdAt", $epochTime()),
    ("lastActive", $epochTime())
  ])
  await ctx.redis.expire(key, ctx.sessionTtl)
  
  # Track user's sessions
  discard await ctx.redis.sadd(&"user_sess:{userId}", @[token])
  await ctx.redis.expire(&"user_sess:{userId}", ctx.sessionTtl + 60)
  
  return token

proc getSession(ctx: AppContext, token: string): Future[JsonNode] {.async.} =
  let key = &"sess:{token}"
  let data = await ctx.redis.hgetall(key)
  
  if data.len == 0:
    return newJNull()
  
  # Update last active
  await ctx.redis.hset(key, "lastActive", $epochTime())
  await ctx.redis.expire(key, ctx.sessionTtl)
  
  var obj = newJObject()
  var i = 0
  while i < data.len - 1:
    obj[data[i]] = %data[i+1]
    i += 2
  return obj

proc deleteSession(ctx: AppContext, token: string) {.async.} =
  let key = &"sess:{token}"
  let userId = await ctx.redis.hget(key, "userId")
  await ctx.redis.del(@[key])
  if userId.len > 0:
    await ctx.redis.srem(&"user_sess:{userId}", @[token])

# ====== Cache Layer ======

proc cacheGet(ctx: AppContext, key: string): Future[string] {.async.} =
  let val = await ctx.redis.get(&"cache:{key}")
  if val == "nil" or val.len == 0:
    return ""
  return val

proc cacheSet(ctx: AppContext, key, value: string, ttl = -1) {.async.} =
  let actualTtl = if ttl > 0: ttl else: ctx.cacheTtl
  await ctx.redis.setex(&"cache:{key}", actualTtl, value)

proc cacheInvalidate(ctx: AppContext, pattern: string) {.async.} =
  let keys = await ctx.redis.keys(&"cache:{pattern}")
  if keys.len > 0:
    await ctx.redis.del(keys)

# ====== Rate Limiting ======

proc checkRateLimit(ctx: AppContext, identifier: string, 
                    limit = 60, window = 60): Future[(bool, int)] {.async.} =
  let key = &"rl:{identifier}"
  let now = epochTime()
  let windowStart = now - window.float
  
  await ctx.redis.zremrangebyscore(key, "-inf", $windowStart)
  let count = int(await ctx.redis.zcard(key))
  
  if count >= limit:
    return (false, 0)
  
  discard await ctx.redis.zadd(key, @[(now, &"{now}")])
  await ctx.redis.expire(key, window)
  
  return (true, limit - count - 1)

# ====== HTTP Server ======

proc getAuthUser(ctx: AppContext, req: Request): Future[JsonNode] {.async.} =
  let authHeader = req.headers.getOrDefault("Authorization")
  
  if not authHeader.startsWith("Bearer "):
    return newJNull()
  
  let token = authHeader[7..^1]
  return await ctx.getSession(token)

proc handleRequest(ctx: AppContext, req: Request) {.async.} =
  # Rate limiting by IP
  let clientIp = req.headers.getOrDefault("X-Real-IP", "unknown")
  let (allowed, remaining) = await ctx.checkRateLimit(clientIp)
  
  if not allowed:
    await req.respond(Http429, 
      $(%*{"error": "Too Many Requests"}),
      newHttpHeaders([
        ("Content-Type", "application/json"),
        ("Retry-After", "60")
      ]))
    return
  
  let headers = newHttpHeaders([
    ("Content-Type", "application/json"),
    ("X-RateLimit-Remaining", $remaining)
  ])
  
  case req.url.path
  of "/api/login":
    if req.reqMethod != HttpPost:
      await req.respond(Http405, "Method Not Allowed", headers)
      return
    
    let body = parseJson(req.body)
    let username = body["username"].getStr()
    let password = body["password"].getStr()
    
    # Simulate auth check
    if username == "admin" and password == "secret":
      let token = await ctx.createSession("1", username)
      await req.respond(Http200, 
        $(%*{"token": token, "expiresIn": ctx.sessionTtl}),
        headers)
    else:
      await req.respond(Http401,
        $(%*{"error": "Invalid credentials"}),
        headers)
  
  of "/api/logout":
    let user = await ctx.getAuthUser(req)
    if user.kind == JNull:
      await req.respond(Http401, $(%*{"error": "Not authenticated"}), headers)
      return
    
    let authHeader = req.headers.getOrDefault("Authorization")
    let token = authHeader[7..^1]
    await ctx.deleteSession(token)
    await req.respond(Http200, $(%*{"message": "Logged out"}), headers)
  
  of "/api/products":
    let cacheKey = "products:list"
    let cached = await ctx.cacheGet(cacheKey)
    
    if cached.len > 0:
      let resp = parseJson(cached)
      resp["cached"] = %true
      await req.respond(Http200, $resp, headers)
      return
    
    # Simulate DB query
    await sleepAsync(50)
    let products = %*{
      "products": [
        {"id": 1, "name": "Widget A", "price": 29.99},
        {"id": 2, "name": "Widget B", "price": 49.99},
        {"id": 3, "name": "Widget C", "price": 99.99}
      ],
      "total": 3,
      "cached": false
    }
    
    await ctx.cacheSet(cacheKey, $products, ttl = 60)
    await req.respond(Http200, $products, headers)
  
  of "/api/me":
    let user = await ctx.getAuthUser(req)
    if user.kind == JNull:
      await req.respond(Http401, $(%*{"error": "Not authenticated"}), headers)
      return
    
    await req.respond(Http200, $user, headers)
  
  else:
    await req.respond(Http404, $(%*{"error": "Not Found"}), headers)

proc main() {.async.} =
  let ctx = await newAppContext()
  var server = newAsyncHttpServer()
  
  proc cb(req: Request) {.async.} =
    await handleRequest(ctx, req)
  
  echo "Session + Cache server on port 8080"
  echo "Endpoints:"
  echo "  POST /api/login"
  echo "  POST /api/logout (requires Bearer token)"
  echo "  GET  /api/products (cached)"
  echo "  GET  /api/me (requires Bearer token)"
  await server.serve(Port(8080), cb)

waitFor main()
```

---

## Step 238: Redis Streams (Event Log)

```nim
# file: src/redis_streams.nim
import redis, asyncdispatch, strformat, json, times

proc streamsDemo() {.async.} =
  let r = await openAsync("localhost", Port(6379))
  defer: r.close()
  
  echo "=== Redis Streams ==="
  
  # XADD - เพิ่ม event ลงใน stream
  proc addEvent(eventType: string, data: seq[(string, string)]): Future[string] {.async.} =
    var fields = @[("type", eventType)] & data
    let id = await r.xadd("events", "*", fields)
    return id
  
  # เพิ่ม events
  let id1 = await addEvent("user.login", @[
    ("userId", "1"), ("username", "alice"), ("ip", "192.168.1.1")
  ])
  echo &"Event 1 ID: {id1}"
  
  let id2 = await addEvent("user.purchase", @[
    ("userId", "1"), ("productId", "42"), ("amount", "29.99")
  ])
  echo &"Event 2 ID: {id2}"
  
  let id3 = await addEvent("user.logout", @[
    ("userId", "1"), ("sessionDuration", "3600")
  ])
  echo &"Event 3 ID: {id3}"
  
  # XLEN - จำนวน entries
  let len = await r.xlen("events")
  echo &"\nTotal events: {len}"
  
  # XRANGE - อ่าน events
  let events = await r.xrange("events", "-", "+")
  echo "\nAll events:"
  for event in events:
    echo &"  {event[0]}: {event[1]}"
  
  # XREAD - อ่านใหม่ตาม ID
  let newEvents = await r.xread(@[("events", "0-0")], count = 10)
  echo &"\nRead {newEvents.len} events from beginning"
  
  # Consumer group
  await r.xgroupCreate("events", "processors", "0", mkstream = true)
  
  # XREADGROUP - อ่านสำหรับ consumer group
  let groupEvents = await r.xreadgroup("processors", "worker1", @[("events", ">")], count = 10)
  echo &"\nGroup read: {groupEvents.len} events"
  
  # XACK - acknowledge processed events
  for event in groupEvents:
    await r.xack("events", "processors", @[event[0]])
  echo "Acknowledged all events"

waitFor streamsDemo()
```

---

## Step 239: Redis Cluster-Ready Patterns

```nim
# file: src/redis_patterns.nim
import redis, asyncdispatch, strformat, json, times, strutils

# Distributed Lock ด้วย SET NX EX
type
  DistributedLock = ref object
    redis: AsyncRedis
    key: string
    token: string
    ttl: int

proc acquireLock(redis: AsyncRedis, resource: string, ttl = 30): Future[DistributedLock] {.async.} =
  let token = &"lock_{rand(int.high)}_{epochTime()}"
  let key = &"dlock:{resource}"
  
  # SET key token NX EX ttl - atomic operation
  let result = await redis.sendCommand("SET", @[key, token, "NX", "EX", $ttl])
  
  if result == "OK":
    return DistributedLock(redis: redis, key: key, token: token, ttl: ttl)
  
  return nil  # Lock not acquired

proc releaseLock(lock: DistributedLock) {.async.} =
  # ตรวจสอบว่า token ยังเป็นของเราอยู่ก่อนลบ (atomic)
  # ใช้ Lua script เพื่อ atomic check-and-delete
  let luaScript = """
    if redis.call("get", KEYS[1]) == ARGV[1] then
      return redis.call("del", KEYS[1])
    else
      return 0
    end
  """
  
  let result = await lock.redis.eval(luaScript, @[lock.key], @[lock.token])
  echo &"Lock released: {result == \"1\"}"

proc withLock(redis: AsyncRedis, resource: string, 
              work: proc() {.async.}, ttl = 30) {.async.} =
  var retries = 0
  var lock: DistributedLock
  
  while retries < 5:
    lock = await acquireLock(redis, resource, ttl)
    if not lock.isNil:
      break
    
    inc retries
    echo &"Lock busy, retry #{retries}..."
    await sleepAsync(100 * retries)
  
  if lock.isNil:
    raise newException(IOError, &"Could not acquire lock for {resource}")
  
  try:
    await work()
  finally:
    await lock.releaseLock()

# Circuit Breaker Pattern
type
  CircuitState = enum
    csClosed = "closed"       # Normal operation
    csOpen = "open"           # Failing, reject requests
    csHalfOpen = "half_open"  # Testing recovery

  CircuitBreaker = ref object
    redis: AsyncRedis
    name: string
    failureThreshold: int
    successThreshold: int
    timeout: int  # seconds before trying again

proc getState(cb: CircuitBreaker): Future[CircuitState] {.async.} =
  let stateStr = await cb.redis.get(&"cb:{cb.name}:state")
  case stateStr
  of "open": return csOpen
  of "half_open": return csHalfOpen
  else: return csClosed

proc recordSuccess(cb: CircuitBreaker) {.async.} =
  let state = await cb.getState()
  
  if state == csHalfOpen:
    let count = await cb.redis.incr(&"cb:{cb.name}:successes")
    if int(count) >= cb.successThreshold:
      await cb.redis.setk(&"cb:{cb.name}:state", "closed")
      await cb.redis.del(@[&"cb:{cb.name}:failures", &"cb:{cb.name}:successes"])
      echo &"[CIRCUIT] {cb.name}: CLOSED (recovered)"

proc recordFailure(cb: CircuitBreaker) {.async.} =
  let count = await cb.redis.incr(&"cb:{cb.name}:failures")
  await cb.redis.expire(&"cb:{cb.name}:failures", 60)
  
  if int(count) >= cb.failureThreshold:
    await cb.redis.setk(&"cb:{cb.name}:state", "open")
    await cb.redis.expire(&"cb:{cb.name}:state", cb.timeout)
    echo &"[CIRCUIT] {cb.name}: OPEN (too many failures)"

proc execute(cb: CircuitBreaker, 
             work: proc(): Future[string] {.async.}): Future[string] {.async.} =
  let state = await cb.getState()
  
  case state
  of csOpen:
    raise newException(IOError, &"Circuit breaker OPEN for {cb.name}")
  
  of csClosed, csHalfOpen:
    try:
      let result = await work()
      asyncCheck cb.recordSuccess()
      return result
    except:
      asyncCheck cb.recordFailure()
      raise

proc patternDemo() {.async.} =
  let r = await openAsync("localhost", Port(6379))
  defer: r.close()
  
  echo "=== Distributed Lock Demo ==="
  
  var counter = 0
  
  proc criticalSection() {.async.} =
    inc counter
    echo &"  In critical section, counter={counter}"
    await sleepAsync(100)
    echo &"  Leaving critical section, counter={counter}"
  
  # Concurrent lock attempts
  let tasks = @[
    withLock(r, "resource1", criticalSection),
    withLock(r, "resource1", criticalSection),
    withLock(r, "resource1", criticalSection)
  ]
  
  await all(tasks)
  echo &"Final counter (should be 3): {counter}"
  
  echo "\n=== Circuit Breaker Demo ==="
  
  let cb = CircuitBreaker(
    redis: r,
    name: "external_api",
    failureThreshold: 3,
    successThreshold: 2,
    timeout: 10
  )
  
  var failCount = 0
  
  proc callExternalApi(): Future[string] {.async.} =
    inc failCount
    if failCount <= 5:
      raise newException(IOError, "API Error")
    return "API Response"
  
  for i in 1..8:
    try:
      let result = await cb.execute(callExternalApi)
      echo &"  Request #{i}: {result}"
    except IOError as e:
      echo &"  Request #{i}: Failed - {e.msg}"
    
    await sleepAsync(50)

waitFor patternDemo()
```

---

## Step 240: Complete Redis-Powered Application

```nim
# file: src/redis_complete_app.nim
import redis, asyncdispatch, asynchttpserver, httpcore
import strformat, json, times, random, strutils, sequtils

type
  App = ref object
    redis: AsyncRedis

proc newApp(): Future[App] {.async.} =
  App(redis: await openAsync("localhost", Port(6379)))

proc getStats(app: App): Future[JsonNode] {.async.} =
  let cacheKey = "stats:global"
  let cached = await app.redis.get(cacheKey)
  
  if cached.len > 0 and cached != "nil":
    var result = parseJson(cached)
    result["fromCache"] = %true
    return result
  
  # Simulate computing stats
  await sleepAsync(100)
  
  let stats = %*{
    "totalUsers": rand(10000),
    "activeUsers": rand(1000),
    "totalPosts": rand(50000),
    "serverLoad": rand(100),
    "computedAt": $now(),
    "fromCache": false
  }
  
  await app.redis.setex(cacheKey, 30, $stats)
  return stats

proc handleStats(app: App, req: Request) {.async.} =
  let stats = await app.getStats()
  await req.respond(Http200, $stats,
    newHttpHeaders([("Content-Type", "application/json")]))

proc handleLeaderboard(app: App, req: Request) {.async.} =
  let top10 = await app.redis.zrevrange("scores", 0, 9, withScores = true)
  
  var board = newJArray()
  var rank = 1
  var i = 0
  while i < top10.len - 1:
    board.add(%*{
      "rank": rank,
      "user": top10[i],
      "score": parseInt(top10[i+1])
    })
    inc rank
    i += 2
  
  await req.respond(Http200,
    $(%*{"leaderboard": board}),
    newHttpHeaders([("Content-Type", "application/json")]))

proc handleAddScore(app: App, req: Request) {.async.} =
  if req.reqMethod != HttpPost:
    await req.respond(Http405, "Method Not Allowed")
    return
  
  let body = parseJson(req.body)
  let user = body["user"].getStr()
  let score = body["score"].getInt()
  
  discard await app.redis.zadd("scores", @[(score.float, user)])
  
  let rank = await app.redis.zrevrank("scores", user)
  await req.respond(Http200,
    $(%*{"user": user, "score": score, "rank": int(rank) + 1}),
    newHttpHeaders([("Content-Type", "application/json")]))

proc seedData(app: App) {.async.} =
  randomize()
  let users = @["alice", "bob", "charlie", "diana", "eve", "frank", "grace", "henry"]
  for user in users:
    discard await app.redis.zadd("scores", @[(rand(10000).float, user)])

proc main() {.async.} =
  let app = await newApp()
  await app.seedData()
  
  var server = newAsyncHttpServer()
  
  proc cb(req: Request) {.async.} =
    case req.url.path
    of "/api/stats":
      await handleStats(app, req)
    of "/api/leaderboard":
      await handleLeaderboard(app, req)
    of "/api/scores":
      await handleAddScore(app, req)
    else:
      await req.respond(Http404, $(%*{"error": "Not found"}),
        newHttpHeaders([("Content-Type", "application/json")]))
  
  echo "Redis-powered app on port 8080"
  await server.serve(Port(8080), cb)

waitFor main()
```

---

## 📝 สรุป Part 17

| Step | หัวข้อ | สิ่งที่เรียนรู้ |
|------|--------|----------------|
| 226 | Installation | Redis setup, Nim library |
| 227 | Connection Basics | SET/GET/EXPIRE/EXISTS/DEL |
| 228 | String Advanced | NX/XX options, atomic counters, cache-aside |
| 229 | Lists | Queue, Stack, BLPOP, activity log |
| 230 | Sets & Sorted Sets | Tags, leaderboard, rate limiting |
| 231 | Hashes | User objects, repositories, partial updates |
| 232 | Pub/Sub | Event streaming, pattern subscription |
| 233 | Session Store | JWT-less sessions, multi-device |
| 234 | Cache Layer | Cache-aside, TTL, invalidation |
| 235 | Rate Limiting | Fixed/Sliding window, Token bucket |
| 236 | Transactions | MULTI/EXEC, WATCH, Pipeline |
| 237 | Complete System | Sessions + Cache + Rate limiting |
| 238 | Streams | Event log, consumer groups |
| 239 | Patterns | Distributed lock, Circuit breaker |
| 240 | Full Application | Leaderboard, stats, caching |

---

## Navigation

- [← Part 16: WebSockets](part_16_websockets.md)
- [→ Part 18: Docker](part_18_docker.md)
- [กลับ README](../README.md)
