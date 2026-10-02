# Part 56: Caching Strategies
## Steps 811-825: Multi-Layer Cache Architecture

---

## 🎯 เป้าหมายของ Part นี้

- LRU / LFU / ARC cache
- Write-through / write-back / write-around
- Cache stampede prevention (single-flight)
- TTL + sliding window expiry
- Distributed cache (mock Redis-style)
- Cache warming & invalidation patterns

---

## Step 811: LRU & LFU Cache

```nim
import tables, strformat, times, sequtils, json, strutils, options, algorithm, hashes, math

# ============================
# LRU Cache
# ============================

type
  LruNode[K, V] = ref object
    key: K
    value: V
    prev: LruNode[K, V]
    next: LruNode[K, V]
    expiresAt: float
    hitCount: int

  LruCache[K, V] = object
    capacity: int
    map: Table[K, LruNode[K, V]]
    head: LruNode[K, V]  # most recent
    tail: LruNode[K, V]  # least recent
    hits: int
    misses: int
    evictions: int

proc newLruCache[K, V](capacity: int): LruCache[K, V] =
  var c = LruCache[K, V](
    capacity: capacity,
    map: initTable[K, LruNode[K, V]]()
  )
  # sentinel nodes
  c.head = LruNode[K, V]()
  c.tail = LruNode[K, V]()
  c.head.next = c.tail
  c.tail.prev = c.head
  return c

proc moveToFront[K, V](cache: var LruCache[K, V], node: LruNode[K, V]) =
  # Remove from current position
  node.prev.next = node.next
  node.next.prev = node.prev
  # Insert after head
  node.next = cache.head.next
  node.prev = cache.head
  cache.head.next.prev = node
  cache.head.next = node

proc lruGet[K, V](cache: var LruCache[K, V], key: K): Option[V] =
  if key notin cache.map:
    inc cache.misses
    return none(V)

  let node = cache.map[key]

  # Check expiry
  if node.expiresAt > 0 and epochTime() > node.expiresAt:
    cache.map.del(key)
    inc cache.misses
    return none(V)

  inc node.hitCount
  inc cache.hits
  moveToFront(cache, node)
  return some(node.value)

proc lruSet[K, V](cache: var LruCache[K, V], key: K, value: V, ttlSeconds = 0.0) =
  let expiresAt = if ttlSeconds > 0: epochTime() + ttlSeconds else: 0.0

  if key in cache.map:
    let node = cache.map[key]
    node.value = value
    node.expiresAt = expiresAt
    moveToFront(cache, node)
    return

  let node = LruNode[K, V](
    key: key, value: value,
    expiresAt: expiresAt, hitCount: 0
  )
  cache.map[key] = node

  # Insert at front
  node.next = cache.head.next
  node.prev = cache.head
  cache.head.next.prev = node
  cache.head.next = node

  # Evict if over capacity
  if cache.map.len > cache.capacity:
    let lru = cache.tail.prev
    if lru != cache.head:
      lru.prev.next = cache.tail
      cache.tail.prev = lru.prev
      cache.map.del(lru.key)
      inc cache.evictions

proc lruDelete[K, V](cache: var LruCache[K, V], key: K) =
  if key notin cache.map: return
  let node = cache.map[key]
  node.prev.next = node.next
  node.next.prev = node.prev
  cache.map.del(key)

proc lruStats[K, V](cache: LruCache[K, V]): JsonNode =
  let total = cache.hits + cache.misses
  let hitRate = if total > 0: float(cache.hits) / float(total) * 100 else: 0.0
  %*{
    "size": cache.map.len,
    "capacity": cache.capacity,
    "hits": cache.hits,
    "misses": cache.misses,
    "evictions": cache.evictions,
    "hitRate": fmt"{hitRate:.1f}%"
  }

# ============================
# LFU Cache
# ============================

type
  LfuEntry[V] = object
    value: V
    freq: int
    expiresAt: float

  LfuCache[K, V] = object
    capacity: int
    entries: Table[K, LfuEntry[V]]
    freqMap: Table[int, seq[K]]   # freq -> [keys]
    minFreq: int
    hits: int
    misses: int

proc newLfuCache[K, V](capacity: int): LfuCache[K, V] =
  LfuCache[K, V](
    capacity: capacity,
    entries: initTable[K, LfuEntry[V]](),
    freqMap: initTable[int, seq[K]](),
    minFreq: 0
  )

proc lfuGet[K, V](cache: var LfuCache[K, V], key: K): Option[V] =
  if key notin cache.entries:
    inc cache.misses
    return none(V)

  var entry = cache.entries[key]
  if entry.expiresAt > 0 and epochTime() > entry.expiresAt:
    cache.entries.del(key)
    inc cache.misses
    return none(V)

  # Update frequency
  let oldFreq = entry.freq
  inc entry.freq
  cache.entries[key] = entry

  if oldFreq in cache.freqMap:
    cache.freqMap[oldFreq] = cache.freqMap[oldFreq].filterIt(it != key)
    if cache.freqMap[oldFreq].len == 0:
      cache.freqMap.del(oldFreq)
      if cache.minFreq == oldFreq: inc cache.minFreq

  if entry.freq notin cache.freqMap:
    cache.freqMap[entry.freq] = @[]
  cache.freqMap[entry.freq].add(key)

  inc cache.hits
  return some(entry.value)

proc lfuSet[K, V](cache: var LfuCache[K, V], key: K, value: V, ttlSeconds = 0.0) =
  if cache.capacity <= 0: return

  if key in cache.entries:
    cache.entries[key].value = value
    discard lfuGet(cache, key)
    return

  # Evict if full
  if cache.entries.len >= cache.capacity:
    if cache.minFreq in cache.freqMap and cache.freqMap[cache.minFreq].len > 0:
      let evictKey = cache.freqMap[cache.minFreq][0]
      cache.freqMap[cache.minFreq].delete(0)
      cache.entries.del(evictKey)

  let expiresAt = if ttlSeconds > 0: epochTime() + ttlSeconds else: 0.0
  cache.entries[key] = LfuEntry[V](value: value, freq: 1, expiresAt: expiresAt)
  cache.minFreq = 1
  if 1 notin cache.freqMap: cache.freqMap[1] = @[]
  cache.freqMap[1].add(key)

# ============================
# Single-flight (cache stampede prevention)
# ============================

type
  FlightKey = string
  InFlight = object
    futures: seq[Future[string]]
    result: string
    done: bool

var inFlightMap: Table[FlightKey, InFlight] = initTable[FlightKey, InFlight]()

proc singleFlight(key: FlightKey,
                  fetch: proc(): Future[string] {.async.}): Future[string] {.async.} =
  ## If multiple callers request the same key simultaneously,
  ## only one fetch is made and the result is shared
  if key in inFlightMap and inFlightMap[key].done:
    return inFlightMap[key].result

  if key notin inFlightMap:
    inFlightMap[key] = InFlight(futures: @[], done: false)
    echo fmt"[SingleFlight] Fetching: {key}"

    let value = await fetch()
    inFlightMap[key].result = value
    inFlightMap[key].done = true
    echo fmt"[SingleFlight] Done: {key}"
    return value
  else:
    # Duplicate request — wait for result
    let f = newFuture[string]("singleFlight.wait")
    inFlightMap[key].futures.add(f)
    await sleepAsync(5)  # poll
    return inFlightMap[key].result

# ============================
# Write-through cache pattern
# ============================

type
  CachePolicy = enum
    cpWriteThrough,   # write to cache AND store simultaneously
    cpWriteBack,      # write to cache, async flush to store
    cpWriteAround     # write to store only, cache on read

  CacheStore[K, V] = object
    cache: LruCache[K, V]
    policy: CachePolicy
    dirtyKeys: seq[K]
    flushCount: int

proc cacheRead[K, V](store: var CacheStore[K, V], key: K,
                      loader: proc(k: K): V): V =
  let cached = lruGet(store.cache, key)
  if cached.isSome: return cached.get()

  let value = loader(key)
  lruSet(store.cache, key, value)
  return value

proc cacheWrite[K, V](store: var CacheStore[K, V], key: K, value: V,
                       persist: proc(k: K, v: V)) =
  case store.policy
  of cpWriteThrough:
    lruSet(store.cache, key, value)
    persist(key, value)

  of cpWriteBack:
    lruSet(store.cache, key, value)
    if key notin store.dirtyKeys:
      store.dirtyKeys.add(key)

  of cpWriteAround:
    lruDelete(store.cache, key)  # invalidate
    persist(key, value)

proc flushDirty[K, V](store: var CacheStore[K, V],
                        persist: proc(k: K, v: V)) =
  for key in store.dirtyKeys:
    let v = lruGet(store.cache, key)
    if v.isSome:
      persist(key, v.get())
      inc store.flushCount
  store.dirtyKeys = @[]

# ============================
# Distributed cache (mock Redis-style)
# ============================

type
  CacheEntry = object
    value: string
    expiresAt: float
    type_: string   # "string" | "list" | "set" | "hash"

  DistributedCache = object
    shards: seq[Table[string, CacheEntry]]
    numShards: int

proc newDistributedCache(shards = 4): DistributedCache =
  var dc = DistributedCache(numShards: shards)
  for _ in 0..<shards:
    dc.shards.add(initTable[string, CacheEntry]())
  return dc

proc getShard(dc: DistributedCache, key: string): int =
  var h = 0u32
  for c in key: h = h * 31u32 + uint32(ord(c))
  return int(h mod uint32(dc.numShards))

proc dcSet(dc: var DistributedCache, key, value: string, ttlSeconds = 0) =
  let shard = getShard(dc, key)
  let expiresAt = if ttlSeconds > 0: epochTime() + float(ttlSeconds) else: 0.0
  dc.shards[shard][key] = CacheEntry(value: value, expiresAt: expiresAt, type_: "string")

proc dcGet(dc: var DistributedCache, key: string): Option[string] =
  let shard = getShard(dc, key)
  if key notin dc.shards[shard]: return none(string)
  let entry = dc.shards[shard][key]
  if entry.expiresAt > 0 and epochTime() > entry.expiresAt:
    dc.shards[shard].del(key)
    return none(string)
  return some(entry.value)

proc dcDel(dc: var DistributedCache, key: string) =
  let shard = getShard(dc, key)
  dc.shards[shard].del(key)

proc dcMGet(dc: var DistributedCache, keys: seq[string]): seq[Option[string]] =
  keys.mapIt(dcGet(dc, it))

proc dcStats(dc: DistributedCache): JsonNode =
  var total = 0
  var perShard = newJArray()
  for i, shard in dc.shards:
    total += shard.len
    perShard.add(%shard.len)
  %*{"total_keys": total, "shards": perShard}

# ============================
# Demo
# ============================

proc demo() {.async.} =
  echo "=== Caching Strategies Demo ==="

  # LRU Cache
  echo "\n--- LRU Cache ---"
  var lru = newLruCache[string, string](capacity = 3)

  lruSet(lru, "a", "apple", ttlSeconds = 60)
  lruSet(lru, "b", "banana")
  lruSet(lru, "c", "cherry")
  lruSet(lru, "d", "date")  # should evict "a" (LRU)

  echo fmt"get a: {lruGet(lru, \"a\")}"  # should be none (evicted)
  echo fmt"get b: {lruGet(lru, \"b\")}"
  echo fmt"get d: {lruGet(lru, \"d\")}"

  lruSet(lru, "e", "elderberry")  # should evict "c" (LRU)

  echo fmt"Stats: {lruStats(lru).pretty()}"

  # LFU Cache
  echo "\n--- LFU Cache ---"
  var lfu = newLfuCache[string, int](capacity = 3)
  lfuSet(lfu, "x", 1)
  lfuSet(lfu, "y", 2)
  lfuSet(lfu, "z", 3)

  # Access x and y multiple times
  for _ in 0..2: discard lfuGet(lfu, "x")
  for _ in 0..1: discard lfuGet(lfu, "y")

  lfuSet(lfu, "w", 4)  # should evict "z" (lowest freq)
  echo fmt"get z after eviction: {lfuGet(lfu, \"z\")}"
  echo fmt"get x: {lfuGet(lfu, \"x\")}"

  # Single-flight
  echo "\n--- Single-flight ---"
  var callCount = 0
  proc fetchUser(): Future[string] {.async.} =
    inc callCount
    await sleepAsync(20)
    return fmt"{{\"id\":\"1\",\"name\":\"Alice\"}} (fetch #{callCount})"

  # Simulate concurrent requests
  let f1 = singleFlight("user:1", fetchUser)
  let f2 = singleFlight("user:1", fetchUser)  # should not call again
  let r1 = await f1
  let r2 = await f2
  echo fmt"Result 1: {r1[0..30]}..."
  echo fmt"Result 2: {r2[0..30]}..."
  echo fmt"Total fetches: {callCount} (should be 1)"

  # Distributed cache
  echo "\n--- Distributed Cache ---"
  var dc = newDistributedCache(shards = 4)
  dcSet(dc, "user:1", """{"name":"Alice"}""", ttlSeconds = 300)
  dcSet(dc, "user:2", """{"name":"Bob"}""")
  dcSet(dc, "session:abc", "user_1", ttlSeconds = 3600)

  echo fmt"user:1 = {dcGet(dc, \"user:1\")}"
  echo fmt"user:2 = {dcGet(dc, \"user:2\")}"
  echo fmt"unknown = {dcGet(dc, \"unknown\")}"

  let multiResults = dcMGet(dc, @["user:1", "user:2", "user:3"])
  echo fmt"mget results: {multiResults.len} (missing={multiResults.countIt(it.isNone)})"

  echo fmt"Cache stats: {dcStats(dc)}"

waitFor demo()
```

---

## 📝 สรุป Part 56

| Steps | หัวข้อ |
|-------|--------|
| 811 | LRU cache (doubly linked list + hash), LFU cache, single-flight |
| 812-825 | Write-through/write-back patterns, distributed cache (sharded) |

---

**← [Part 55: Task Scheduler](part_55_task_scheduler.md) | [Part 57: Observability →](part_57_observability.md)**
