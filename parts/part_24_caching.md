# Part 24: Caching Strategies
## Steps 331-345: Cache ระดับ Production

---

## 🎯 เป้าหมายของ Part นี้

- In-memory caching
- Redis caching
- Cache-aside pattern
- Write-through, write-back
- Cache invalidation strategies
- Distributed caching

---

## Step 331: In-Memory Cache

```nim
import std/tables, std/times, std/options, std/hashes, std/sequtils

type
  CachePolicy = enum
    LRU     # Least Recently Used
    LFU     # Least Frequently Used
    FIFO    # First In First Out
    TTL     # Time To Live only

  CacheStats = object
    hits: int
    misses: int
    evictions: int
    size: int

  CacheItem[V] = object
    value: V
    expiresAt: float
    accessCount: int
    lastAccessed: float
    insertedAt: float

  MemCache[K, V] = object
    items: Table[K, CacheItem[V]]
    maxSize: int
    defaultTtl: float
    policy: CachePolicy
    stats: CacheStats

proc newMemCache[K, V](
  maxSize: int = 1000,
  ttl: float = 300.0,
  policy: CachePolicy = LRU
): MemCache[K, V] =
  MemCache[K, V](
    items: initTable[K, CacheItem[V]](),
    maxSize: maxSize,
    defaultTtl: ttl,
    policy: policy,
    stats: CacheStats()
  )

proc isExpired[V](item: CacheItem[V]): bool =
  item.expiresAt > 0 and epochTime() > item.expiresAt

proc evict[K, V](cache: var MemCache[K, V]) =
  # Remove expired items first
  var expired: seq[K] = @[]
  for k, item in cache.items:
    if item.isExpired():
      expired.add(k)
  for k in expired:
    cache.items.del(k)
    inc cache.stats.evictions
  
  if cache.items.len < cache.maxSize:
    return
  
  # Apply eviction policy
  case cache.policy
  of LRU:
    var oldest = high(float)
    var oldestKey: K
    var found = false
    for k, item in cache.items:
      if item.lastAccessed < oldest:
        oldest = item.lastAccessed
        oldestKey = k
        found = true
    if found:
      cache.items.del(oldestKey)
      inc cache.stats.evictions
  
  of LFU:
    var minCount = high(int)
    var minKey: K
    var found = false
    for k, item in cache.items:
      if item.accessCount < minCount:
        minCount = item.accessCount
        minKey = k
        found = true
    if found:
      cache.items.del(minKey)
      inc cache.stats.evictions
  
  of FIFO:
    var oldest = high(float)
    var oldestKey: K
    var found = false
    for k, item in cache.items:
      if item.insertedAt < oldest:
        oldest = item.insertedAt
        oldestKey = k
        found = true
    if found:
      cache.items.del(oldestKey)
      inc cache.stats.evictions
  
  else: discard

proc set[K, V](cache: var MemCache[K, V], key: K, value: V, ttl: float = -1.0) =
  cache.evict()
  
  let actualTtl = if ttl < 0: cache.defaultTtl else: ttl
  let now = epochTime()
  
  cache.items[key] = CacheItem[V](
    value: value,
    expiresAt: if actualTtl > 0: now + actualTtl else: 0.0,
    accessCount: 0,
    lastAccessed: now,
    insertedAt: now
  )
  
  cache.stats.size = cache.items.len

proc get[K, V](cache: var MemCache[K, V], key: K): Option[V] =
  if key notin cache.items:
    inc cache.stats.misses
    return none(V)
  
  var item = cache.items[key]
  
  if item.isExpired():
    cache.items.del(key)
    dec cache.stats.size
    inc cache.stats.misses
    return none(V)
  
  # Update access metadata
  cache.items[key].accessCount += 1
  cache.items[key].lastAccessed = epochTime()
  inc cache.stats.hits
  
  return some(item.value)

proc del[K, V](cache: var MemCache[K, V], key: K) =
  if key in cache.items:
    cache.items.del(key)
    dec cache.stats.size

proc clear[K, V](cache: var MemCache[K, V]) =
  cache.items.clear()
  cache.stats.size = 0

proc hitRate[K, V](cache: MemCache[K, V]): float =
  let total = cache.stats.hits + cache.stats.misses
  if total == 0: return 0.0
  float(cache.stats.hits) / float(total)

# Test
var cache = newMemCache[string, string](maxSize = 5, ttl = 10.0, policy = LRU)

cache.set("key1", "value1")
cache.set("key2", "value2")
cache.set("key3", "value3")
cache.set("key4", "value4")
cache.set("key5", "value5")

echo cache.get("key1")  # some("value1")
echo cache.get("key6")  # none

# Adding 6th item should evict LRU (key2 was never accessed)
cache.set("key6", "value6")

echo fmt"Hit rate: {cache.hitRate():.1%}"
echo fmt"Cache size: {cache.stats.size}"
echo fmt"Evictions: {cache.stats.evictions}"
```

---

## Step 332: Cache-Aside Pattern

```nim
import std/tables, std/options, std/strformat, std/times

# Simulated database
type
  UserDb = Table[int, string]

var userDb: UserDb = {
  1: "Alice Smith",
  2: "Bob Jones",
  3: "Charlie Brown",
}.toTable()

var queryCount = 0

proc findUserInDb(id: int): Option[string] =
  inc queryCount
  echo fmt"  [DB Query #{queryCount}] SELECT name FROM users WHERE id = {id}"
  if id in userDb:
    return some(userDb[id])
  return none(string)

# Cache-aside implementation
var userCache = newMemCache[int, string](maxSize = 100, ttl = 60.0)

proc findUser(id: int): Option[string] =
  # 1. Check cache first
  let cached = userCache.get(id)
  if cached.isSome:
    echo fmt"  [CACHE HIT] user:{id}"
    return cached
  
  # 2. Cache miss: query database
  echo fmt"  [CACHE MISS] user:{id}"
  let dbResult = findUserInDb(id)
  
  # 3. Store in cache
  if dbResult.isSome:
    userCache.set(id, dbResult.get())
  
  return dbResult

proc updateUser(id: int, name: string) =
  # Update database
  userDb[id] = name
  echo fmt"  [DB UPDATE] users SET name = '{name}' WHERE id = {id}"
  
  # Invalidate cache (Cache-Aside requires manual invalidation)
  userCache.del(id)
  echo fmt"  [CACHE INVALIDATE] user:{id}"

# Demo
echo "=== Cache-Aside Pattern ==="
echo "\nFirst access (cache miss):"
echo $findUser(1)

echo "\nSecond access (cache hit):"
echo $findUser(1)
echo $findUser(1)

echo "\nUpdate user:"
updateUser(1, "Alice Cooper")

echo "\nAfter update (cache miss again):"
echo $findUser(1)

echo fmt"\nTotal DB queries: {queryCount}"
echo fmt"Cache hit rate: {userCache.hitRate():.1%}"
```

---

## Step 333: Multi-Level Cache

```nim
import std/tables, std/options, std/times, std/strformat

# L1: In-process memory cache (fast, small)
# L2: Redis cache (shared between instances, medium)
# L3: Database (persistent, slow)

type
  CacheLevel = enum
    L1Memory, L2Redis, L3Database

  MultiLevelCache[K, V] = object
    l1: MemCache[K, V]
    l2Size: int  # simulated Redis
    l2Data: Table[K, (V, float)]  # (value, expires)

proc newMultiLevelCache[K, V](
  l1MaxSize: int = 100,
  l1Ttl: float = 60.0,
  l2MaxSize: int = 1000
): MultiLevelCache[K, V] =
  MultiLevelCache[K, V](
    l1: newMemCache[K, V](l1MaxSize, l1Ttl),
    l2Size: l2MaxSize,
    l2Data: initTable[K, (V, float)]()
  )

proc setL2[K, V](mlc: var MultiLevelCache[K, V], key: K, value: V, ttl: float) =
  mlc.l2Data[key] = (value, epochTime() + ttl)

proc getL2[K, V](mlc: var MultiLevelCache[K, V], key: K): Option[V] =
  if key in mlc.l2Data:
    let (value, expires) = mlc.l2Data[key]
    if epochTime() < expires:
      return some(value)
    else:
      mlc.l2Data.del(key)
  return none(V)

proc get[K, V](
  mlc: var MultiLevelCache[K, V],
  key: K,
  fetcher: proc(k: K): Option[V],
  l1Ttl: float = 60.0,
  l2Ttl: float = 300.0
): (V, CacheLevel) =
  # Try L1
  let l1Result = mlc.l1.get(key)
  if l1Result.isSome:
    return (l1Result.get(), L1Memory)
  
  # Try L2
  let l2Result = mlc.getL2(key)
  if l2Result.isSome:
    # Promote to L1
    mlc.l1.set(key, l2Result.get(), l1Ttl)
    return (l2Result.get(), L2Redis)
  
  # Fetch from source
  let fetched = fetcher(key)
  if fetched.isSome:
    let value = fetched.get()
    mlc.l1.set(key, value, l1Ttl)
    mlc.setL2(key, value, l2Ttl)
    return (value, L3Database)
  
  raise newException(KeyError, "Key not found")

# Demo
var mlc = newMultiLevelCache[string, string](l1MaxSize = 10, l1Ttl = 30.0)

proc fetchFromDb(key: string): Option[string] =
  echo fmt"  [DB FETCH] {key}"
  return some("value_" & key)

for i in 1..3:
  echo fmt"\nAccess {i} for key 'user:1':"
  let (value, level) = mlc.get("user:1", fetchFromDb)
  echo fmt"  Got '{value}' from {level}"
```

---

## Step 334: Cache Invalidation Patterns

```nim
import std/tables, std/sets, std/sequtils, std/strformat

# Tag-based cache invalidation
type
  TaggedCache[V] = object
    data: Table[string, V]
    tags: Table[string, HashSet[string]]  # tag -> keys
    keyTags: Table[string, seq[string]]   # key -> tags

proc newTaggedCache[V](): TaggedCache[V] =
  TaggedCache[V](
    data: initTable[string, V](),
    tags: initTable[string, HashSet[string]](),
    keyTags: initTable[string, seq[string]]()
  )

proc set[V](cache: var TaggedCache[V], key: string, value: V, tags: seq[string] = @[]) =
  cache.data[key] = value
  cache.keyTags[key] = tags
  
  for tag in tags:
    if tag notin cache.tags:
      cache.tags[tag] = initHashSet[string]()
    cache.tags[tag].incl(key)

proc get[V](cache: TaggedCache[V], key: string): Option[V] =
  if key in cache.data:
    return some(cache.data[key])
  return none(V)

proc invalidateByTag[V](cache: var TaggedCache[V], tag: string) =
  if tag notin cache.tags:
    return
  
  let keys = cache.tags[tag].toSeq()
  echo fmt"Invalidating {keys.len} items with tag '{tag}'"
  
  for key in keys:
    cache.data.del(key)
    cache.keyTags.del(key)
  
  cache.tags.del(tag)

proc invalidateByPrefix[V](cache: var TaggedCache[V], prefix: string) =
  let toDelete = cache.data.keys.toSeq().filterIt(it.startsWith(prefix))
  echo fmt"Invalidating {toDelete.len} items with prefix '{prefix}'"
  for key in toDelete:
    cache.data.del(key)
    cache.keyTags.del(key)

# Demo: E-commerce cache
var productCache = newTaggedCache[string]()

productCache.set("product:1", "Laptop", @["products", "electronics", "category:electronics"])
productCache.set("product:2", "Mouse", @["products", "electronics", "category:electronics"])
productCache.set("product:3", "Notebook", @["products", "stationery", "category:stationery"])
productCache.set("product_list:electronics", "[1,2]", @["product_lists", "electronics"])
productCache.set("product_list:all", "[1,2,3]", @["product_lists"])

echo "\nBefore invalidation:"
echo fmt"Cache size: {productCache.data.len}"

# Update a product -> invalidate its tags
echo "\nUpdating Laptop (product:1)..."
productCache.invalidateByTag("category:electronics")
# This removes: product:1, product:2, product_list:electronics

echo fmt"\nCache size after: {productCache.data.len}"
echo "product:1 exists: " & $(productCache.get("product:1").isSome)  # false
echo "product:3 exists: " & $(productCache.get("product:3").isSome)  # true
```

---

## Step 335-345: Complete Caching System

```nim
# caching_system.nim - Production-ready caching layer

import std/tables, std/options, std/times, std/strformat,
       std/json, std/hashes, std/sequtils

# ==============================
# Generic TTL Cache with metrics
# ==============================

type
  CacheMetrics = object
    hits: int
    misses: int
    sets: int
    evictions: int
    expirations: int

  TtlCache[K, V] = object
    data: OrderedTable[K, (V, float)]  # (value, expires)
    maxSize: int
    metrics: CacheMetrics

proc newTtlCache[K, V](maxSize: int = 10000): TtlCache[K, V] =
  TtlCache[K, V](
    data: initOrderedTable[K, (V, float)](),
    maxSize: maxSize
  )

proc set[K, V](cache: var TtlCache[K, V], key: K, value: V, ttl: float = 300.0) =
  let expires = epochTime() + ttl
  
  if key notin cache.data and cache.data.len >= cache.maxSize:
    # Evict oldest
    for k in cache.data.keys:
      cache.data.del(k)
      inc cache.metrics.evictions
      break
  
  cache.data[key] = (value, expires)
  inc cache.metrics.sets

proc get[K, V](cache: var TtlCache[K, V], key: K): Option[V] =
  if key notin cache.data:
    inc cache.metrics.misses
    return none(V)
  
  let (value, expires) = cache.data[key]
  
  if epochTime() > expires:
    cache.data.del(key)
    inc cache.metrics.expirations
    inc cache.metrics.misses
    return none(V)
  
  inc cache.metrics.hits
  return some(value)

proc del[K, V](cache: var TtlCache[K, V], key: K) =
  cache.data.del(key)

proc cleanup[K, V](cache: var TtlCache[K, V]) =
  let now = epochTime()
  var expired: seq[K] = @[]
  for k, (_, expires) in cache.data:
    if now > expires:
      expired.add(k)
  for k in expired:
    cache.data.del(k)
    inc cache.metrics.expirations

proc printMetrics[K, V](cache: TtlCache[K, V], name: string) =
  let total = cache.metrics.hits + cache.metrics.misses
  let hitRate = if total > 0: float(cache.metrics.hits) / float(total) else: 0.0
  
  echo fmt"\n{name} Cache Metrics:"
  echo fmt"  Hits:       {cache.metrics.hits}"
  echo fmt"  Misses:     {cache.metrics.misses}"
  echo fmt"  Hit Rate:   {hitRate:.1%}"
  echo fmt"  Sets:       {cache.metrics.sets}"
  echo fmt"  Evictions:  {cache.metrics.evictions}"
  echo fmt"  Expirations:{cache.metrics.expirations}"
  echo fmt"  Current Size: {cache.data.len}"

# ==============================
# Application-level caches
# ==============================

# User cache (15 min TTL)
var userCache2 = newTtlCache[int, JsonNode](maxSize = 1000)

# Session cache (30 min TTL)  
var sessionCache = newTtlCache[string, JsonNode](maxSize = 10000)

# Product catalog cache (5 min TTL)
var productCache2 = newTtlCache[string, JsonNode](maxSize = 500)

# API response cache (1 min TTL)
var apiCache = newTtlCache[string, string](maxSize = 200)

# ==============================
# Cache key generators
# ==============================

proc userKey(id: int): int = id

proc sessionKey(token: string): string = "session:" & token

proc productListKey(category: string, page, limit: int): string =
  fmt"products:{category}:page{page}:limit{limit}"

proc apiResponseKey(httpMethod, path, queryString: string): string =
  fmt"api:{httpMethod}:{path}:{hash(queryString)}"

# ==============================
# Cached service functions
# ==============================

# Simulated DB
var usersDb: Table[int, JsonNode] = {
  1: %*{"id": 1, "name": "Alice", "email": "alice@example.com"},
  2: %*{"id": 2, "name": "Bob", "email": "bob@example.com"},
}.toTable()

proc getUserCached(id: int): Option[JsonNode] =
  let cached = userCache2.get(id)
  if cached.isSome:
    return cached
  
  # DB lookup
  if id in usersDb:
    let user = usersDb[id]
    userCache2.set(id, user, ttl = 900.0)  # 15 min
    return some(user)
  
  return none(JsonNode)

proc getProductListCached(category: string, page, limit: int): JsonNode =
  let key = productListKey(category, page, limit)
  
  let cached = productCache2.get(key)
  if cached.isSome:
    return cached.get()
  
  # Simulated DB query
  echo fmt"  [DB] Fetching products for category={category} page={page}"
  let result = %*{
    "items": [],
    "total": 0,
    "category": category,
    "page": page
  }
  
  productCache2.set(key, result, ttl = 300.0)  # 5 min
  return result

proc invalidateUserCache(id: int) =
  userCache2.del(id)
  echo fmt"Invalidated user cache for id={id}"

proc invalidateProductCache() =
  # Clear all product caches when products change
  # In production, use tag-based invalidation
  var toDelete: seq[string] = @[]
  for k in productCache2.data.keys:
    if k.startsWith("products:"):
      toDelete.add(k)
  for k in toDelete:
    productCache2.del(k)
  echo fmt"Invalidated {toDelete.len} product cache entries"

# ==============================
# Demo
# ==============================

proc main() =
  echo "=== Caching System Demo ==="
  
  echo "\n--- User Cache ---"
  for i in 1..4:
    let user = getUserCached(1)
    if user.isSome:
      echo fmt"  Access {i}: Got user '{user.get()[\"name\"].getStr()}'"
  
  userCache2.printMetrics("User")
  
  echo "\n--- Product Cache ---"
  for page in 1..3:
    let products = getProductListCached("electronics", page, 10)
    echo fmt"  Page {page}: category={products[\"category\"].getStr()}"
  
  # Access page 1 again (should be cached)
  discard getProductListCached("electronics", 1, 10)
  discard getProductListCached("electronics", 1, 10)
  
  productCache2.printMetrics("Product")
  
  echo "\n--- Cache Invalidation ---"
  invalidateUserCache(1)
  invalidateProductCache()
  
  echo "\n--- After Invalidation ---"
  let user = getUserCached(1)
  echo fmt"User 1 re-fetched: {user.isSome}"
  
  userCache2.printMetrics("User (after invalidation)")

main()
```

---

## 📝 สรุป Part 24

| Steps | หัวข้อ |
|-------|--------|
| 331 | In-memory cache (LRU, LFU, FIFO, TTL) |
| 332 | Cache-aside pattern |
| 333 | Multi-level cache (L1/L2/L3) |
| 334 | Tag-based cache invalidation |
| 335-345 | Complete caching system |

---

**← [Part 23: Advanced Database](part_23_advanced_database.md) | [Part 25: Microservices →](part_25_microservices.md)**
