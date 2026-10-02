# Part 52: Event Streaming
## Steps 751-765: Kafka-Style Distributed Log

---

## 🎯 เป้าหมายของ Part นี้

- Append-only log (commit log)
- Topics, partitions, offsets
- Producer with partitioner
- Consumer groups with offset tracking
- Compaction & retention
- Stream processing (filter, map, join, window)

---

## Step 751: Commit Log Core

```nim
import asyncdispatch, tables, strformat, times, sequtils, json, strutils, options, algorithm, hashes, math

# ============================
# Core types
# ============================

type
  LogRecord = object
    offset: int64
    key: string
    value: string      # serialized payload
    headers: Table[string, string]
    timestamp: float
    partition: int

  Partition = object
    id: int
    topic: string
    records: seq[LogRecord]
    highWatermark: int64   # next offset to write
    retentionBytes: int64
    retentionMs: float

  Topic = object
    name: string
    partitions: seq[Partition]
    replicationFactor: int
    cleanupPolicy: string   # "delete" | "compact"

  Broker = object
    topics: Table[string, Topic]
    groupOffsets: Table[string, Table[string, int64]]  # group -> (topicPartition -> offset)

var broker = Broker(
  topics: initTable[string, Topic](),
  groupOffsets: initTable[string, Table[string, int64]]()
)

# ============================
# Topic management
# ============================

proc createTopic(name: string, numPartitions = 1,
                 replicationFactor = 1, cleanupPolicy = "delete") =
  var partitions: seq[Partition]
  for i in 0..<numPartitions:
    partitions.add(Partition(
      id: i,
      topic: name,
      records: @[],
      highWatermark: 0,
      retentionBytes: 1024 * 1024 * 1024,  # 1GB
      retentionMs: 7 * 24 * 3600 * 1000.0  # 7 days
    ))

  broker.topics[name] = Topic(
    name: name,
    partitions: partitions,
    replicationFactor: replicationFactor,
    cleanupPolicy: cleanupPolicy
  )
  echo fmt"[Broker] Created topic: {name} ({numPartitions} partitions)"

proc deleteTopic(name: string) =
  broker.topics.del(name)
  echo fmt"[Broker] Deleted topic: {name}"

# ============================
# Partitioner
# ============================

proc partitionByKey(key: string, numPartitions: int): int =
  if key.len == 0: return 0
  var h = 0u32
  for c in key:
    h = h * 31u32 + uint32(ord(c))
  return int(h mod uint32(numPartitions))

proc partitionRoundRobin(topic: string, numPartitions: int): int =
  ## Stateless round-robin using current time
  return int(epochTime() * 1000) mod numPartitions

# ============================
# Producer
# ============================

type
  ProducerConfig = object
    clientId: string
    compressionType: string   # "none" | "snappy" | "lz4"
    acks: int                 # 0, 1, -1 (all)
    batchSize: int
    lingerMs: int

  ProduceResult = object
    topic: string
    partition: int
    offset: int64
    timestamp: float
    success: bool
    error: string

proc produce(topic, key, value: string,
             partition = -1,
             headers: Table[string, string] = initTable[string, string]()): ProduceResult =
  if topic notin broker.topics:
    return ProduceResult(success: false, error: fmt"Unknown topic: {topic}")

  var t = broker.topics[topic]
  let numPartitions = t.partitions.len

  let targetPartition = if partition >= 0 and partition < numPartitions:
    partition
  elif key.len > 0:
    partitionByKey(key, numPartitions)
  else:
    partitionRoundRobin(topic, numPartitions)

  let offset = t.partitions[targetPartition].highWatermark
  let record = LogRecord(
    offset: offset,
    key: key,
    value: value,
    headers: headers,
    timestamp: epochTime(),
    partition: targetPartition
  )

  t.partitions[targetPartition].records.add(record)
  inc t.partitions[targetPartition].highWatermark
  broker.topics[topic] = t

  return ProduceResult(
    topic: topic,
    partition: targetPartition,
    offset: offset,
    timestamp: record.timestamp,
    success: true
  )

proc produceBatch(topic: string, records: seq[tuple[key, value: string]]): seq[ProduceResult] =
  var results: seq[ProduceResult]
  for (key, value) in records:
    results.add(produce(topic, key, value))
  return results

# ============================
# Consumer
# ============================

type
  ConsumerConfig = object
    groupId: string
    clientId: string
    autoOffsetReset: string   # "earliest" | "latest"
    maxPollRecords: int

  ConsumerAssignment = object
    topic: string
    partition: int

  Consumer = object
    config: ConsumerConfig
    assignments: seq[ConsumerAssignment]

proc subscribe(consumer: var Consumer, topics: seq[string]) =
  consumer.assignments = @[]
  for topic in topics:
    if topic in broker.topics:
      for p in broker.topics[topic].partitions:
        consumer.assignments.add(ConsumerAssignment(topic: topic, partition: p.id))
  echo fmt"[Consumer:{consumer.config.groupId}] Subscribed to: {topics}"

proc getOffset(groupId, topic: string, partition: int): int64 =
  let key = fmt"{topic}:{partition}"
  if groupId notin broker.groupOffsets: return -1
  return broker.groupOffsets[groupId].getOrDefault(key, -1)

proc commitOffset(groupId, topic: string, partition: int, offset: int64) =
  if groupId notin broker.groupOffsets:
    broker.groupOffsets[groupId] = initTable[string, int64]()
  let key = fmt"{topic}:{partition}"
  broker.groupOffsets[groupId][key] = offset

proc poll(consumer: Consumer, maxRecords = 100): seq[LogRecord] =
  var records: seq[LogRecord]
  let groupId = consumer.config.groupId

  for assignment in consumer.assignments:
    if assignment.topic notin broker.topics: continue
    let t = broker.topics[assignment.topic]
    if assignment.partition >= t.partitions.len: continue

    let partition = t.partitions[assignment.partition]
    var offset = getOffset(groupId, assignment.topic, assignment.partition)

    if offset < 0:
      offset = case consumer.config.autoOffsetReset
        of "earliest": 0
        of "latest": partition.highWatermark - 1
        else: 0

    let startIdx = max(0, int(offset))
    let endIdx = min(startIdx + maxRecords, partition.records.len)

    for i in startIdx..<endIdx:
      records.add(partition.records[i])

  return records

proc acknowledge(consumer: Consumer, records: seq[LogRecord]) =
  for r in records:
    let nextOffset = r.offset + 1
    commitOffset(consumer.config.groupId, r.topic, r.partition, nextOffset)

# ============================
# Log compaction
# ============================

proc compact(topicName: string) =
  ## Keep only the latest value for each key
  if topicName notin broker.topics: return
  var topic = broker.topics[topicName]

  for i, partition in topic.partitions:
    var latest: Table[string, LogRecord]
    for r in partition.records:
      if r.key.len > 0:
        latest[r.key] = r

    # Rebuild compacted log
    var compacted: seq[LogRecord]
    var newOffset: int64 = 0
    for _, r in latest:
      var rec = r
      rec.offset = newOffset
      compacted.add(rec)
      inc newOffset

    compacted.sort(proc(a, b: LogRecord): int = cmp(a.offset, b.offset))

    echo fmt"[Broker] Compacted {topicName}/{i}: {partition.records.len} -> {compacted.len} records"
    topic.partitions[i].records = compacted
    topic.partitions[i].highWatermark = newOffset

  broker.topics[topicName] = topic

# ============================
# Demo
# ============================

proc demo() =
  echo "=== Event Streaming Demo ==="

  # Create topics
  createTopic("orders", numPartitions = 3)
  createTopic("user-events", numPartitions = 2)
  createTopic("inventory-updates", numPartitions = 1, cleanupPolicy = "compact")

  # Produce messages
  echo "\n--- Producing messages ---"
  for i in 1..10:
    let key = fmt"user_{(i mod 3) + 1}"
    let value = $(%*{"orderId": i, "amount": float(i) * 9.99, "status": "created"})
    let result = produce("orders", key, value)
    if result.success:
      echo fmt"  Produced: offset={result.offset} partition={result.partition} key={key}"

  # Topic stats
  echo "\n--- Topic stats ---"
  for pName, topic in broker.topics:
    var totalRecords = 0
    for p in topic.partitions:
      totalRecords += p.records.len
    echo fmt"  {topic.name}: {totalRecords} records across {topic.partitions.len} partitions"

  # Consumer group 1
  echo "\n--- Consumer group: analytics ---"
  var consumer = Consumer(
    config: ConsumerConfig(
      groupId: "analytics",
      clientId: "analytics-1",
      autoOffsetReset: "earliest",
      maxPollRecords: 5
    )
  )
  consumer.subscribe(@["orders"])

  let batch = poll(consumer, maxRecords = 5)
  echo fmt"Polled {batch.len} records:"
  for r in batch:
    echo fmt"  [{r.partition}@{r.offset}] key={r.key}"

  acknowledge(consumer, batch)

  # Second poll should resume from committed offset
  let batch2 = poll(consumer, maxRecords = 5)
  echo fmt"\nSecond poll: {batch2.len} records"

  # Log compaction
  echo "\n--- Log compaction ---"
  for i in 1..5:
    let key = fmt"item_{i mod 3 + 1}"
    let value = $(%*{"itemId": i mod 3 + 1, "quantity": i * 10})
    discard produce("inventory-updates", key, value)

  echo fmt"Before compact: {broker.topics[\"inventory-updates\"].partitions[0].records.len} records"
  compact("inventory-updates")
  echo fmt"After compact: {broker.topics[\"inventory-updates\"].partitions[0].records.len} records"

demo()
```

---

## Step 752-765: Stream Processing

```nim
import asyncdispatch, tables, strformat, times, sequtils, json, strutils, options, algorithm

# ============================
# Stream processor types
# ============================

type
  StreamRecord = object
    key: string
    value: JsonNode
    topic: string
    partition: int
    offset: int64
    timestamp: float

  ProcessorFn = proc(r: StreamRecord): Option[StreamRecord]
  SinkFn = proc(r: StreamRecord) {.closure.}

  StreamProcessor = object
    name: string
    inputTopics: seq[string]
    outputTopic: string
    processors: seq[ProcessorFn]
    sinks: seq[SinkFn]
    processed: int
    dropped: int

# ============================
# Filter processor
# ============================

proc filterProcessor(field, value: string): ProcessorFn =
  proc(r: StreamRecord): Option[StreamRecord] =
    if r.value.kind != JObject: return none(StreamRecord)
    if field notin r.value: return none(StreamRecord)
    if r.value[field].getStr() == value:
      return some(r)
    return none(StreamRecord)

# ============================
# Map processor
# ============================

proc mapProcessor(transform: proc(v: JsonNode): JsonNode): ProcessorFn =
  proc(r: StreamRecord): Option[StreamRecord] =
    var rec = r
    try:
      rec.value = transform(r.value)
      return some(rec)
    except:
      return none(StreamRecord)

# ============================
# Windowed aggregation
# ============================

type
  WindowType = enum
    wtTumbling, wtSliding, wtSession

  Window = object
    startTs: float
    endTs: float
    records: seq[StreamRecord]

  WindowedAggregator = object
    windowType: WindowType
    windowSizeMs: float
    slideMs: float       # for sliding windows
    windows: seq[Window]

proc addToWindow(agg: var WindowedAggregator, r: StreamRecord) =
  let ts = r.timestamp * 1000  # to ms

  case agg.windowType
  of wtTumbling:
    let windowStart = float(int(ts / agg.windowSizeMs)) * agg.windowSizeMs
    let windowEnd = windowStart + agg.windowSizeMs

    var found = false
    for i, w in agg.windows:
      if w.startTs == windowStart:
        agg.windows[i].records.add(r)
        found = true
        break

    if not found:
      agg.windows.add(Window(
        startTs: windowStart,
        endTs: windowEnd,
        records: @[r]
      ))

  of wtSliding:
    let windowStart = ts - agg.windowSizeMs
    var found = false
    for i, w in agg.windows:
      if abs(w.startTs - windowStart) < agg.slideMs:
        agg.windows[i].records.add(r)
        found = true
        break
    if not found:
      agg.windows.add(Window(
        startTs: windowStart,
        endTs: ts,
        records: @[r]
      ))

  of wtSession:
    # Session: gap-based grouping
    var latestWindow = -1
    var latestEnd = 0.0
    for i, w in agg.windows:
      if w.endTs > latestEnd:
        latestEnd = w.endTs
        latestWindow = i

    if latestWindow >= 0 and ts - latestEnd < agg.windowSizeMs:
      agg.windows[latestWindow].records.add(r)
      agg.windows[latestWindow].endTs = ts
    else:
      agg.windows.add(Window(startTs: ts, endTs: ts, records: @[r]))

proc expireWindows(agg: var WindowedAggregator) =
  let cutoff = epochTime() * 1000 - agg.windowSizeMs * 2
  agg.windows = agg.windows.filterIt(it.endTs > cutoff)

type
  WindowResult = object
    windowStart: float
    windowEnd: float
    count: int
    sum: float
    avg: float

proc aggregateWindows(agg: WindowedAggregator, valueField: string): seq[WindowResult] =
  var results: seq[WindowResult]
  for w in agg.windows:
    var sum = 0.0
    var count = 0
    for r in w.records:
      if r.value.kind == JObject and valueField in r.value:
        let v = r.value[valueField]
        let n = case v.kind
          of JFloat: v.getFloat()
          of JInt: float(v.getInt())
          of JString:
            try: parseFloat(v.getStr())
            except: 0.0
          else: 0.0
        sum += n
        inc count

    results.add(WindowResult(
      windowStart: w.startTs / 1000,
      windowEnd: w.endTs / 1000,
      count: count,
      sum: sum,
      avg: if count > 0: sum / float(count) else: 0.0
    ))

  results.sort(proc(a, b: WindowResult): int = cmp(a.windowStart, b.windowStart))
  return results

# ============================
# Stream join
# ============================

type
  JoinType = enum
    jtInner, jtLeft, jtRight

  StreamJoin = object
    joinType: JoinType
    windowMs: float
    leftBuffer: Table[string, seq[StreamRecord]]   # key -> records
    rightBuffer: Table[string, seq[StreamRecord]]

proc bufferRecord(join: var StreamJoin, side: string, r: StreamRecord) =
  if side == "left":
    if r.key notin join.leftBuffer: join.leftBuffer[r.key] = @[]
    join.leftBuffer[r.key].add(r)
  else:
    if r.key notin join.rightBuffer: join.rightBuffer[r.key] = @[]
    join.rightBuffer[r.key].add(r)

proc joinRecords(join: StreamJoin, key: string): seq[tuple[left, right: Option[StreamRecord]]] =
  let lefts = join.leftBuffer.getOrDefault(key, @[])
  let rights = join.rightBuffer.getOrDefault(key, @[])

  if lefts.len == 0 and rights.len == 0: return @[]

  var result: seq[tuple[left, right: Option[StreamRecord]]]

  case join.joinType
  of jtInner:
    for l in lefts:
      for r in rights:
        if abs(l.timestamp - r.timestamp) <= join.windowMs / 1000:
          result.add((some(l), some(r)))

  of jtLeft:
    for l in lefts:
      var matched = false
      for r in rights:
        if abs(l.timestamp - r.timestamp) <= join.windowMs / 1000:
          result.add((some(l), some(r)))
          matched = true
      if not matched:
        result.add((some(l), none(StreamRecord)))

  of jtRight:
    for r in rights:
      var matched = false
      for l in lefts:
        if abs(l.timestamp - r.timestamp) <= join.windowMs / 1000:
          result.add((some(l), some(r)))
          matched = true
      if not matched:
        result.add((none(StreamRecord), some(r)))

  return result

# ============================
# Demo
# ============================

proc demo() =
  echo "=== Stream Processing Demo ==="

  # Sample event stream
  let events: seq[StreamRecord] = @[
    StreamRecord(key: "user_1", topic: "orders",
      value: %*{"amount": 99.99, "status": "created"}, timestamp: epochTime()),
    StreamRecord(key: "user_2", topic: "orders",
      value: %*{"amount": 49.50, "status": "created"}, timestamp: epochTime()),
    StreamRecord(key: "user_1", topic: "orders",
      value: %*{"amount": 150.00, "status": "paid"}, timestamp: epochTime()),
    StreamRecord(key: "user_3", topic: "orders",
      value: %*{"amount": 25.00, "status": "created"}, timestamp: epochTime()),
    StreamRecord(key: "user_2", topic: "orders",
      value: %*{"amount": 75.00, "status": "paid"}, timestamp: epochTime()),
  ]

  # Filter: only paid orders
  echo "\n--- Filter: paid orders ---"
  let paidFilter = filterProcessor("status", "paid")
  var paidOrders: seq[StreamRecord]
  for evt in events:
    let result = paidFilter(evt)
    if result.isSome:
      paidOrders.add(result.get())
  echo fmt"Paid orders: {paidOrders.len}"
  for o in paidOrders:
    echo fmt"  key={o.key} amount={o.value[\"amount\"].getFloat():.2f}"

  # Map: add tax
  echo "\n--- Map: add tax ---"
  let taxMapper = mapProcessor(proc(v: JsonNode): JsonNode =
    var result = v
    if v.kind == JObject and "amount" in v:
      let amount = v["amount"].getFloat()
      result["tax"] = %(amount * 0.07)
      result["total"] = %(amount + amount * 0.07)
    return result
  )
  for o in paidOrders:
    let mapped = taxMapper(o)
    if mapped.isSome:
      echo fmt"  total={mapped.get().value[\"total\"].getFloat():.2f}"

  # Windowed aggregation
  echo "\n--- Tumbling window (5min) ---"
  var agg = WindowedAggregator(
    windowType: wtTumbling,
    windowSizeMs: 300000.0  # 5 min
  )

  for evt in events:
    addToWindow(agg, evt)

  let windowResults = aggregateWindows(agg, "amount")
  echo fmt"Windows: {windowResults.len}"
  for w in windowResults:
    echo fmt"  count={w.count} sum=${w.sum:.2f} avg=${w.avg:.2f}"

  # Stream join: orders + user profiles
  echo "\n--- Stream join ---"
  var join = StreamJoin(joinType: jtLeft, windowMs: 5000.0,
    leftBuffer: initTable[string, seq[StreamRecord]](),
    rightBuffer: initTable[string, seq[StreamRecord]]())

  for o in events:
    bufferRecord(join, "left", o)

  let userProfiles: seq[StreamRecord] = @[
    StreamRecord(key: "user_1", topic: "users",
      value: %*{"name": "Alice", "plan": "pro"}, timestamp: epochTime()),
    StreamRecord(key: "user_2", topic: "users",
      value: %*{"name": "Bob", "plan": "free"}, timestamp: epochTime()),
  ]

  for p in userProfiles:
    bufferRecord(join, "right", p)

  echo "Joined (orders LEFT JOIN users):"
  for key in ["user_1", "user_2", "user_3"]:
    let joined = joinRecords(join, key)
    for (left, right) in joined:
      let amount = if left.isSome: left.get().value["amount"].getFloat() else: 0.0
      let name = if right.isSome: right.get().value["name"].getStr() else: "unknown"
      echo fmt"  {key}: amount=${amount:.2f} user={name}"

demo()
```

---

## 📝 สรุป Part 52

| Steps | หัวข้อ |
|-------|--------|
| 751 | Commit log, topics, partitions, producer, consumer groups |
| 752-765 | Stream processing: filter/map, windowed aggregation, stream join |

---

**← [Part 51: Config Management](part_51_config_management.md) | [Part 53: GraphQL Server →](part_53_graphql.md)**
