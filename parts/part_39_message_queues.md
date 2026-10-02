# Part 39: Message Queues
## Steps 556-570: Event-Driven Architecture

---

## 🎯 เป้าหมายของ Part นี้

- In-process message queue
- Dead letter queue (DLQ)
- Publisher/Subscriber
- Message routing (topic-based)
- Durable queues (disk-backed)
- NATS-style protocol over TCP

---

## Step 556: Core Message Queue

```nim
import asyncdispatch, queues, times, strformat, json, tables, sequtils, algorithm

# ============================
# Message types
# ============================

type
  MessagePriority = enum
    mpLow = 0, mpNormal = 1, mpHigh = 2, mpCritical = 3

  MessageStatus = enum
    msPending, msProcessing, msCompleted, msFailed, msDlq

  Message = object
    id: string
    topic: string
    payload: JsonNode
    priority: MessagePriority
    status: MessageStatus
    attempts: int
    maxAttempts: int
    createdAt: float
    scheduledAt: float   # for delayed messages
    expiresAt: float     # 0 = no expiry
    headers: Table[string, string]

  HandlerResult = enum
    hrAck, hrNack, hrRetry

  MessageHandler = proc(msg: Message): Future[HandlerResult] {.async.}

  Queue = object
    name: string
    messages: seq[Message]
    dlq: seq[Message]
    processed: int
    failed: int

var queues: Table[string, Queue] = initTable[string, Queue]()

proc msgId(): string =
  fmt"msg_{int(epochTime() * 1_000_000) mod 1_000_000_000}"

proc newMessage(topic: string, payload: JsonNode,
                priority = mpNormal, maxAttempts = 3,
                delayMs = 0, ttlMs = 0): Message =
  let now = epochTime()
  Message(
    id: msgId(),
    topic: topic,
    payload: payload,
    priority: priority,
    status: msPending,
    attempts: 0,
    maxAttempts: maxAttempts,
    createdAt: now,
    scheduledAt: now + float(delayMs) / 1000.0,
    expiresAt: if ttlMs > 0: now + float(ttlMs) / 1000.0 else: 0.0,
    headers: initTable[string, string]()
  )

proc ensureQueue(name: string) =
  if name notin queues:
    queues[name] = Queue(name: name, messages: @[], dlq: @[])

proc enqueue(queueName: string, msg: Message) =
  ensureQueue(queueName)
  queues[queueName].messages.add(msg)
  # Sort by priority descending, then by scheduledAt ascending
  queues[queueName].messages.sort(proc(a, b: Message): int =
    if a.priority != b.priority: cmp(b.priority, a.priority)
    else: cmp(a.scheduledAt, b.scheduledAt)
  )
  echo fmt"[Queue:{queueName}] Enqueued: {msg.id} (priority: {msg.priority})"

proc dequeue(queueName: string): Option[Message] =
  if queueName notin queues:
    return none(Message)
  
  var q = queues[queueName]
  let now = epochTime()
  
  for i, msg in q.messages:
    # Skip not-yet-scheduled messages
    if msg.scheduledAt > now:
      continue
    
    # Skip expired messages
    if msg.expiresAt > 0 and now > msg.expiresAt:
      q.messages.delete(i)
      queues[queueName] = q
      echo fmt"[Queue:{queueName}] Message expired: {msg.id}"
      return none(Message)
    
    var m = msg
    m.status = msProcessing
    m.attempts += 1
    q.messages.delete(i)
    queues[queueName] = q
    return some(m)
  
  return none(Message)

proc ackMessage(queueName: string, msg: Message) =
  queues[queueName].processed += 1
  echo fmt"[Queue:{queueName}] ACK: {msg.id} (attempts: {msg.attempts})"

proc nackMessage(queueName: string, var msg: Message) =
  msg.status = msFailed
  
  if msg.attempts >= msg.maxAttempts:
    # Move to DLQ
    msg.status = msDlq
    queues[queueName].dlq.add(msg)
    queues[queueName].failed += 1
    echo fmt"[Queue:{queueName}] DLQ: {msg.id} after {msg.attempts} attempts"
  else:
    # Re-queue with exponential backoff
    let backoff = float(2 ^ msg.attempts) * 1000.0  # ms
    msg.status = msPending
    msg.scheduledAt = epochTime() + backoff / 1000.0
    enqueue(queueName, msg)
    echo fmt"[Queue:{queueName}] NACK + retry in {backoff}ms: {msg.id}"

proc queueStats(queueName: string): JsonNode =
  if queueName notin queues:
    return %*{"error": "queue not found"}
  let q = queues[queueName]
  %*{
    "name": q.name,
    "pending": q.messages.len,
    "processed": q.processed,
    "failed": q.failed,
    "dlq": q.dlq.len
  }

# ============================
# Demo
# ============================

proc demo() {.async.} =
  echo "=== Message Queue Demo ==="
  
  ensureQueue("orders")
  ensureQueue("emails")
  
  # Enqueue messages with different priorities
  enqueue("orders", newMessage("order.created",
    %*{"orderId": 1, "total": 99.99}, priority = mpNormal))
  
  enqueue("orders", newMessage("order.refund",
    %*{"orderId": 2, "amount": 49.99}, priority = mpHigh))
  
  enqueue("orders", newMessage("order.reminder",
    %*{"orderId": 3}, priority = mpLow, delayMs = 5000))
  
  enqueue("emails", newMessage("email.welcome",
    %*{"to": "user@example.com", "name": "Alice"}))
  
  echo "\n--- Processing orders queue ---"
  
  for _ in 0..2:
    let msg = dequeue("orders")
    if msg.isSome:
      let m = msg.get()
      echo fmt"Processing: {m.topic} (priority: {m.priority})"
      
      # Simulate processing
      if m.topic == "order.refund":
        # Simulate failure
        nackMessage("orders", m)
      else:
        ackMessage("orders", m)
  
  echo "\n--- Queue stats ---"
  echo "orders: " & $queueStats("orders")
  echo "emails: " & $queueStats("emails")

waitFor demo()
```

---

## Step 557: Pub/Sub System

```nim
import asyncdispatch, tables, strformat, json, sequtils, times

# ============================
# Topic-based pub/sub
# ============================

type
  SubscriberFn = proc(topic: string, payload: JsonNode) {.async.}

  Subscription = object
    id: string
    pattern: string   # "orders.*", "user.created", etc.
    handler: SubscriberFn
    receivedCount: int

  EventBus = object
    subscriptions: Table[string, seq[Subscription]]
    publishedCount: int
    subCounter: int

var bus = EventBus(
  subscriptions: initTable[string, seq[Subscription]](),
  publishedCount: 0,
  subCounter: 0
)

proc topicMatches(pattern, topic: string): bool =
  if pattern == topic:
    return true
  if pattern.endsWith(".*"):
    let prefix = pattern[0..^3]
    return topic.startsWith(prefix & ".")
  if pattern == "*":
    return true
  return false

proc subscribe(pattern: string, handler: SubscriberFn): string =
  inc bus.subCounter
  let subId = fmt"sub_{bus.subCounter}"
  
  if pattern notin bus.subscriptions:
    bus.subscriptions[pattern] = @[]
  
  bus.subscriptions[pattern].add(Subscription(
    id: subId,
    pattern: pattern,
    handler: handler,
    receivedCount: 0
  ))
  
  echo fmt"[EventBus] Subscribed: {subId} to '{pattern}'"
  return subId

proc unsubscribe(subId: string) =
  for pattern, subs in bus.subscriptions.mpairs:
    bus.subscriptions[pattern] = subs.filterIt(it.id != subId)
  echo fmt"[EventBus] Unsubscribed: {subId}"

proc publish(topic: string, payload: JsonNode) {.async.} =
  inc bus.publishedCount
  echo fmt"[EventBus] Publishing: {topic}"
  
  var handled = 0
  
  for pattern, subs in bus.subscriptions.mpairs:
    if topicMatches(pattern, topic):
      for i, sub in subs:
        try:
          await sub.handler(topic, payload)
          inc bus.subscriptions[pattern][i].receivedCount
          inc handled
        except:
          echo fmt"[EventBus] Handler error for {sub.id}: {getCurrentExceptionMsg()}"
  
  if handled == 0:
    echo fmt"[EventBus] No subscribers for: {topic}"

# ============================
# Event-driven saga
# ============================

type
  SagaState = enum
    ssRunning, ssCompleted, ssCompensating, ssFailed

  SagaStep = object
    name: string
    completed: bool
    compensated: bool
    result: JsonNode

  Saga = object
    id: string
    state: SagaState
    steps: seq[SagaStep]
    data: JsonNode

var sagas: Table[string, Saga] = initTable[string, Saga]()

proc startSaga(sagaId, firstEvent: string, data: JsonNode) {.async.} =
  sagas[sagaId] = Saga(
    id: sagaId,
    state: ssRunning,
    steps: @[],
    data: data
  )
  await publish(firstEvent, data %* {"sagaId": sagaId})

proc completeSagaStep(sagaId, stepName: string, result: JsonNode) =
  if sagaId notin sagas: return
  var saga = sagas[sagaId]
  saga.steps.add(SagaStep(name: stepName, completed: true, result: result))
  sagas[sagaId] = saga
  echo fmt"[Saga:{sagaId}] Step completed: {stepName}"

proc failSaga(sagaId: string) {.async.} =
  if sagaId notin sagas: return
  var saga = sagas[sagaId]
  saga.state = ssCompensating
  sagas[sagaId] = saga
  
  echo fmt"[Saga:{sagaId}] Compensating {saga.steps.len} completed steps..."
  
  # Run compensations in reverse
  for i in countdown(saga.steps.len - 1, 0):
    let step = saga.steps[i]
    await publish(fmt"saga.compensate.{step.name}", %*{"sagaId": sagaId})
  
  saga.state = ssFailed
  sagas[sagaId] = saga
  echo fmt"[Saga:{sagaId}] Failed and compensated"

# ============================
# Demo
# ============================

proc demo() {.async.} =
  echo "=== Pub/Sub + Saga Demo ==="
  
  # Set up subscribers
  discard subscribe("order.*", proc(topic: string, payload: JsonNode) {.async.} =
    echo fmt"  [OrderSvc] Received: {topic}"
  )
  
  discard subscribe("user.created", proc(topic: string, payload: JsonNode) {.async.} =
    echo fmt"  [UserSvc] User created: {payload[\"email\"].getStr()}"
  )
  
  discard subscribe("*", proc(topic: string, payload: JsonNode) {.async.} =
    echo fmt"  [Audit] All events: {topic}"
  )
  
  # Publish events
  await publish("order.created", %*{"orderId": 1, "total": 99.99})
  await publish("order.shipped", %*{"orderId": 1, "trackingNo": "TRACK123"})
  await publish("user.created", %*{"email": "alice@example.com"})
  await publish("payment.received", %*{"amount": 99.99})
  
  echo fmt"\nTotal published: {bus.publishedCount}"
  
  # Saga demo
  echo "\n--- Order Saga ---"
  let orderId = "saga_001"
  await startSaga(orderId, "payment.process", %*{"orderId": 1, "amount": 99.99})
  completeSagaStep(orderId, "payment", %*{"transactionId": "tx123"})
  completeSagaStep(orderId, "inventory", %*{"reserved": true})
  
  # Simulate failure
  await failSaga(orderId)

waitFor demo()
```

---

## Step 558-570: Durable Message Queue

```nim
# durable_queue.nim - Disk-backed message queue for persistence

import asyncdispatch, os, strformat, json, tables, times, sequtils, streams

# ============================
# WAL (Write-Ahead Log) based queue
# ============================

type
  WalEntry = object
    sequence: int
    op: string         # "enqueue" | "ack" | "nack" | "dlq"
    msgId: string
    topic: string
    payload: string    # JSON string
    timestamp: float

  DurableMessage = object
    id: string
    topic: string
    payload: JsonNode
    attempts: int
    maxAttempts: int
    createdAt: float
    state: string     # pending | processing | acked | failed

  DurableQueue = object
    name: string
    walPath: string
    dataDir: string
    messages: Table[string, DurableMessage]
    sequence: int

proc openQueue(name, dataDir: string): DurableQueue =
  let walPath = dataDir / fmt"{name}.wal"
  createDir(dataDir)
  
  var q = DurableQueue(
    name: name,
    walPath: walPath,
    dataDir: dataDir,
    messages: initTable[string, DurableMessage](),
    sequence: 0
  )
  
  # Replay WAL on startup
  if fileExists(walPath):
    let content = readFile(walPath)
    for line in content.splitLines():
      if line.len == 0: continue
      try:
        let entry = parseJson(line)
        let op = entry["op"].getStr()
        let msgId = entry["msgId"].getStr()
        
        case op
        of "enqueue":
          q.messages[msgId] = DurableMessage(
            id: msgId,
            topic: entry["topic"].getStr(),
            payload: parseJson(entry["payload"].getStr()),
            attempts: 0,
            maxAttempts: 3,
            createdAt: entry["ts"].getFloat(),
            state: "pending"
          )
          inc q.sequence
        of "ack":
          if msgId in q.messages:
            q.messages.del(msgId)
        of "nack":
          if msgId in q.messages:
            q.messages[msgId].state = "failed"
        else: discard
      except:
        echo fmt"[WAL] Skipping malformed entry: {getCurrentExceptionMsg()}"
    
    echo fmt"[DurableQueue:{name}] Replayed {q.messages.len} messages from WAL"
  
  return q

proc appendWal(q: var DurableQueue, entry: WalEntry) =
  let line = $(%*{
    "seq": entry.sequence,
    "op": entry.op,
    "msgId": entry.msgId,
    "topic": entry.topic,
    "payload": entry.payload,
    "ts": entry.timestamp
  }) & "\n"
  
  let f = open(q.walPath, fmAppend)
  defer: f.close()
  f.write(line)

proc enqueue(q: var DurableQueue, topic: string, payload: JsonNode,
             maxAttempts = 3): string =
  inc q.sequence
  let msgId = fmt"msg_{q.sequence}_{int(epochTime() * 1000) mod 10000}"
  
  let msg = DurableMessage(
    id: msgId, topic: topic, payload: payload,
    attempts: 0, maxAttempts: maxAttempts,
    createdAt: epochTime(), state: "pending"
  )
  q.messages[msgId] = msg
  
  q.appendWal(WalEntry(
    sequence: q.sequence,
    op: "enqueue",
    msgId: msgId,
    topic: topic,
    payload: $payload,
    timestamp: epochTime()
  ))
  
  echo fmt"[DurableQueue:{q.name}] Enqueued: {msgId}"
  return msgId

proc dequeue(q: var DurableQueue): Option[DurableMessage] =
  for msgId, msg in q.messages:
    if msg.state == "pending":
      q.messages[msgId].state = "processing"
      q.messages[msgId].attempts += 1
      return some(q.messages[msgId])
  return none(DurableMessage)

proc ack(q: var DurableQueue, msgId: string) =
  if msgId in q.messages:
    q.messages.del(msgId)
    q.appendWal(WalEntry(
      sequence: q.sequence,
      op: "ack",
      msgId: msgId,
      topic: "", payload: "",
      timestamp: epochTime()
    ))
    echo fmt"[DurableQueue:{q.name}] ACK: {msgId}"

proc nack(q: var DurableQueue, msgId: string) =
  if msgId notin q.messages: return
  
  var msg = q.messages[msgId]
  
  if msg.attempts >= msg.maxAttempts:
    q.messages[msgId].state = "failed"
    q.appendWal(WalEntry(
      sequence: q.sequence, op: "dlq", msgId: msgId,
      topic: msg.topic, payload: "", timestamp: epochTime()
    ))
    echo fmt"[DurableQueue:{q.name}] DLQ: {msgId}"
  else:
    q.messages[msgId].state = "pending"
    q.appendWal(WalEntry(
      sequence: q.sequence, op: "nack", msgId: msgId,
      topic: msg.topic, payload: "", timestamp: epochTime()
    ))
    echo fmt"[DurableQueue:{q.name}] NACK (retry {msg.attempts}/{msg.maxAttempts}): {msgId}"

proc compactWal(q: var DurableQueue) =
  # Rewrite WAL with only current state (remove acked messages)
  var newWal = ""
  for msgId, msg in q.messages:
    if msg.state in ["pending", "processing"]:
      inc q.sequence
      newWal &= $(%*{
        "seq": q.sequence,
        "op": "enqueue",
        "msgId": msg.id,
        "topic": msg.topic,
        "payload": $msg.payload,
        "ts": msg.createdAt
      }) & "\n"
  
  writeFile(q.walPath, newWal)
  echo fmt"[DurableQueue:{q.name}] WAL compacted: {q.messages.len} messages"

# ============================
# Demo
# ============================

proc demo() =
  echo "=== Durable Queue Demo ==="
  
  let dataDir = "/tmp/queues"
  var q = openQueue("tasks", dataDir)
  
  # Enqueue messages
  let id1 = q.enqueue("email.send", %*{"to": "alice@example.com", "subject": "Welcome"})
  let id2 = q.enqueue("sms.send", %*{"phone": "+1234567890", "text": "Hello"}, maxAttempts = 5)
  let id3 = q.enqueue("webhook.notify", %*{"url": "https://api.example.com/hook"})
  
  echo fmt"\nQueued {q.messages.len} messages"
  
  # Process messages
  echo "\n--- Processing ---"
  for _ in 0..2:
    let msg = q.dequeue()
    if msg.isSome:
      let m = msg.get()
      echo fmt"Processing: {m.topic} [{m.id}]"
      
      # Simulate: email succeeds, sms fails
      if m.topic == "sms.send":
        q.nack(m.id)  # fail and retry
      else:
        q.ack(m.id)   # success
  
  echo fmt"\nRemaining in queue: {q.messages.len}"
  
  # Compact WAL
  q.compactWal()
  
  # Cleanup
  removeDir(dataDir)

demo()
```

---

## 📝 สรุป Part 39

| Steps | หัวข้อ |
|-------|--------|
| 556 | In-memory queue, priority, DLQ, retry with backoff |
| 557 | Pub/Sub with wildcard topics, Event Saga |
| 558-570 | Durable WAL-backed queue with compaction |

---

**← [Part 38: Multi-Tenancy](part_38_multitenancy.md) | [Part 40: gRPC-style RPC →](part_40_rpc.md)**
