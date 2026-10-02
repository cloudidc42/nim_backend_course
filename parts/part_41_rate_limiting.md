# Part 41: Advanced Rate Limiting
## Steps 586-600: Rate Limiting ระดับ Production

---

## 🎯 เป้าหมายของ Part นี้

- Token bucket algorithm
- Sliding window counter
- Leaky bucket
- Distributed rate limiting (Redis-based)
- Per-user / per-IP / per-endpoint limits
- Rate limit headers (X-RateLimit-*)

---

## Step 586: Token Bucket

```nim
import times, strformat, tables, math

# ============================
# Token Bucket Algorithm
# ============================
# - Burst allowed up to capacity
# - Tokens refill at constant rate
# - Best for: APIs that allow short bursts

type
  TokenBucket = object
    capacity: float     # max tokens
    tokens: float       # current tokens
    refillRate: float   # tokens per second
    lastRefill: float   # epoch timestamp

proc newTokenBucket(capacity: float, refillPerSecond: float): TokenBucket =
  TokenBucket(
    capacity: capacity,
    tokens: capacity,  # start full
    refillRate: refillPerSecond,
    lastRefill: epochTime()
  )

proc refill(b: var TokenBucket) =
  let now = epochTime()
  let elapsed = now - b.lastRefill
  let newTokens = elapsed * b.refillRate
  b.tokens = min(b.capacity, b.tokens + newTokens)
  b.lastRefill = now

proc tryConsume(b: var TokenBucket, tokens: float = 1.0): bool =
  b.refill()
  if b.tokens >= tokens:
    b.tokens -= tokens
    return true
  return false

proc waitTime(b: var TokenBucket, tokens: float = 1.0): float =
  ## Returns seconds to wait before `tokens` are available
  b.refill()
  if b.tokens >= tokens:
    return 0.0
  let needed = tokens - b.tokens
  return needed / b.refillRate

proc info(b: TokenBucket): string =
  fmt"TokenBucket(tokens: {b.tokens:.1f}/{b.capacity:.0f}, rate: {b.refillRate:.0f}/s)"

# ============================
# Sliding Window Counter
# ============================
# More accurate than fixed window, less memory than per-request log

type
  WindowSlot = object
    count: int
    timestamp: float

  SlidingWindowCounter = object
    windowSize: float   # seconds
    limit: int          # max requests per window
    slots: seq[WindowSlot]
    slotDuration: float # seconds per slot
    numSlots: int

proc newSlidingWindow(windowSecs: float, limit: int, numSlots: int = 10): SlidingWindowCounter =
  SlidingWindowCounter(
    windowSize: windowSecs,
    limit: limit,
    slots: newSeq[WindowSlot](numSlots),
    slotDuration: windowSecs / float(numSlots),
    numSlots: numSlots
  )

proc currentCount(sw: var SlidingWindowCounter): int =
  let now = epochTime()
  let cutoff = now - sw.windowSize
  var total = 0
  for slot in sw.slots:
    if slot.timestamp >= cutoff:
      total += slot.count
  return total

proc tryAllow(sw: var SlidingWindowCounter): bool =
  let now = epochTime()
  
  if sw.currentCount() >= sw.limit:
    return false
  
  # Find current slot
  let slotIdx = int(now / sw.slotDuration) mod sw.numSlots
  
  # Reset slot if it's from previous cycle
  if now - sw.slots[slotIdx].timestamp > sw.slotDuration * float(sw.numSlots):
    sw.slots[slotIdx] = WindowSlot(count: 0, timestamp: now)
  
  sw.slots[slotIdx].count += 1
  sw.slots[slotIdx].timestamp = now
  return true

proc remaining(sw: var SlidingWindowCounter): int =
  max(0, sw.limit - sw.currentCount())

# ============================
# Leaky Bucket
# ============================
# - Fixed output rate (queue drains at constant rate)
# - Best for: smoothing traffic spikes

type
  LeakyBucket = object
    capacity: int       # max queue size
    rate: float         # requests per second to process
    queue: int          # current queue depth
    lastLeak: float

proc newLeakyBucket(capacity: int, rate: float): LeakyBucket =
  LeakyBucket(capacity: capacity, rate: rate, queue: 0, lastLeak: epochTime())

proc leak(b: var LeakyBucket) =
  let now = epochTime()
  let elapsed = now - b.lastLeak
  let leaked = int(elapsed * b.rate)
  b.queue = max(0, b.queue - leaked)
  b.lastLeak = now

proc tryAdd(b: var LeakyBucket): bool =
  b.leak()
  if b.queue >= b.capacity:
    return false
  inc b.queue
  return true

# ============================
# Demo
# ============================

proc demo() =
  echo "=== Rate Limiting Algorithms Demo ==="
  
  echo "\n--- Token Bucket (10 capacity, 2 tokens/sec) ---"
  var tb = newTokenBucket(10, 2.0)
  echo tb.info()
  
  var allowed = 0
  for i in 0..<15:
    if tb.tryConsume():
      inc allowed
  
  echo fmt"Allowed: {allowed}/15 requests"
  echo fmt"Wait time for next: {tb.waitTime():.2f}s"
  
  echo "\n--- Sliding Window (10 req/second) ---"
  var sw = newSlidingWindow(1.0, 10, numSlots = 5)
  
  var swAllowed = 0
  for i in 0..<15:
    if sw.tryAllow():
      inc swAllowed
  
  echo fmt"Allowed: {swAllowed}/15 requests"
  echo fmt"Remaining: {sw.remaining()}"
  
  echo "\n--- Leaky Bucket (5 capacity, 3 req/sec) ---"
  var lb = newLeakyBucket(5, 3.0)
  
  var lbAllowed = 0
  for i in 0..<8:
    if lb.tryAdd():
      inc lbAllowed
  
  echo fmt"Allowed: {lbAllowed}/8 requests"
  echo fmt"Queue depth: {lb.queue}/{lb.capacity}"

demo()
```

---

## Step 587: Rate Limiter with HTTP Headers

```nim
import asyncdispatch, asynchttpserver, strformat, tables, times, options, strutils

# ============================
# Rate limit policy per route
# ============================

type
  RateLimitPolicy = object
    requestsPerWindow: int
    windowSecs: float
    burst: int          # extra burst above limit
    keyType: string     # "ip" | "user" | "key" | "global"

  RateLimitEntry = object
    count: int
    windowStart: float
    burstUsed: int

  RateLimiter = object
    policies: Table[string, RateLimitPolicy]
    entries: Table[string, RateLimitEntry]

proc newRateLimiter(): RateLimiter =
  RateLimiter(
    policies: initTable[string, RateLimitPolicy](),
    entries: initTable[string, RateLimitEntry]()
  )

proc addPolicy(rl: var RateLimiter, route: string, policy: RateLimitPolicy) =
  rl.policies[route] = policy

proc matchPolicy(rl: RateLimiter, path: string): tuple[route: string, policy: RateLimitPolicy, found: bool] =
  # Exact match first
  if path in rl.policies:
    return (path, rl.policies[path], true)
  
  # Prefix match
  for route, policy in rl.policies:
    if path.startsWith(route):
      return (route, policy, true)
  
  # Default
  let defaultPolicy = RateLimitPolicy(
    requestsPerWindow: 1000, windowSecs: 60.0, burst: 50, keyType: "ip"
  )
  return ("*", defaultPolicy, true)

proc extractKey(req: Request, keyType: string): string =
  case keyType
  of "ip":
    return req.hostname
  of "user":
    # In real implementation: extract from JWT
    let auth = req.headers.getOrDefault("Authorization", "")
    if auth.startsWith("Bearer "):
      return "user:" & auth[7..min(20, auth.len-1)]
    return "anon:" & req.hostname
  of "key":
    return req.headers.getOrDefault("X-API-Key", req.hostname)
  of "global":
    return "global"
  else:
    return req.hostname

type
  RateLimitResult = object
    allowed: bool
    limit: int
    remaining: int
    resetAt: float
    retryAfter: float

proc check(rl: var RateLimiter, req: Request): RateLimitResult =
  let now = epochTime()
  let (_, policy, _) = rl.matchPolicy(req.url.path)
  let key = extractKey(req, policy.keyType)
  
  let entryKey = fmt"{req.url.path}:{key}"
  
  if entryKey notin rl.entries:
    rl.entries[entryKey] = RateLimitEntry(
      count: 0, windowStart: now, burstUsed: 0
    )
  
  var entry = rl.entries[entryKey]
  
  # Reset if window expired
  if now - entry.windowStart >= policy.windowSecs:
    entry = RateLimitEntry(count: 0, windowStart: now, burstUsed: 0)
  
  let effectiveLimit = policy.requestsPerWindow + policy.burst
  let resetAt = entry.windowStart + policy.windowSecs
  
  if entry.count >= effectiveLimit:
    rl.entries[entryKey] = entry
    return RateLimitResult(
      allowed: false,
      limit: policy.requestsPerWindow,
      remaining: 0,
      resetAt: resetAt,
      retryAfter: resetAt - now
    )
  
  inc entry.count
  rl.entries[entryKey] = entry
  
  return RateLimitResult(
    allowed: true,
    limit: policy.requestsPerWindow,
    remaining: max(0, effectiveLimit - entry.count),
    resetAt: resetAt,
    retryAfter: 0.0
  )

proc addRateLimitHeaders(headers: var HttpHeaders, result: RateLimitResult) =
  headers["X-RateLimit-Limit"] = $result.limit
  headers["X-RateLimit-Remaining"] = $result.remaining
  headers["X-RateLimit-Reset"] = $int(result.resetAt)
  if not result.allowed:
    headers["Retry-After"] = fmt"{result.retryAfter:.0f}"

# ============================
# Demo
# ============================

proc demo() =
  echo "=== Rate Limiter with HTTP Headers Demo ==="
  
  var rl = newRateLimiter()
  
  # Define policies
  rl.addPolicy("/api/auth/", RateLimitPolicy(
    requestsPerWindow: 5, windowSecs: 60.0, burst: 2, keyType: "ip"
  ))
  rl.addPolicy("/api/users", RateLimitPolicy(
    requestsPerWindow: 100, windowSecs: 60.0, burst: 10, keyType: "user"
  ))
  rl.addPolicy("/api/search", RateLimitPolicy(
    requestsPerWindow: 20, windowSecs: 60.0, burst: 5, keyType: "user"
  ))
  
  echo "\nTesting auth endpoint (limit: 5/min):"
  for i in 0..<8:
    # Create mock request data
    let hostname = "192.168.1.1"
    let path = "/api/auth/login"
    let entryKey = fmt"{path}:{hostname}"
    
    # Direct policy check simulation
    let now = epochTime()
    let policy = rl.policies["/api/auth/"]
    
    if entryKey notin rl.entries:
      rl.entries[entryKey] = RateLimitEntry(count: 0, windowStart: now, burstUsed: 0)
    
    var entry = rl.entries[entryKey]
    let effectiveLimit = policy.requestsPerWindow + policy.burst
    let allowed = entry.count < effectiveLimit
    
    if allowed: inc entry.count
    rl.entries[entryKey] = entry
    
    let icon = if allowed: "✓" else: "✗"
    echo fmt"  Request {i+1}: {icon} ({entry.count}/{effectiveLimit})"
    
    if not allowed:
      echo fmt"    → Retry-After: ~{policy.windowSecs:.0f}s"

demo()
```

---

## Step 588-600: Distributed Rate Limiting

```nim
# distributed_rate_limiter.nim - Redis-backed rate limiting

import strformat, times, json, tables, hashes, math

# ============================
# Simulated Redis operations
# (In production: use actual Redis client)
# ============================

type
  MockRedis = object
    data: Table[string, tuple[value: string, expiry: float]]

var redis = MockRedis(data: initTable[string, tuple[value: string, expiry: float]]())

proc redisGet(key: string): string =
  if key notin redis.data: return ""
  let entry = redis.data[key]
  if entry.expiry > 0 and epochTime() > entry.expiry:
    redis.data.del(key)
    return ""
  return entry.value

proc redisSet(key, value: string, expiryMs: int = 0) =
  let expiry = if expiryMs > 0: epochTime() + float(expiryMs) / 1000.0 else: 0.0
  redis.data[key] = (value, expiry)

proc redisIncr(key: string): int =
  let current = redisGet(key)
  let newVal = if current.len == 0: 1 else: parseInt(current) + 1
  let expiry = if key in redis.data: redis.data[key].expiry else: 0.0
  redis.data[key] = ($newVal, expiry)
  return newVal

proc redisExpire(key: string, ms: int) =
  if key in redis.data:
    redis.data[key] = (redis.data[key].value, epochTime() + float(ms) / 1000.0)

proc redisTtl(key: string): float =
  if key notin redis.data: return -1.0
  let expiry = redis.data[key].expiry
  if expiry <= 0: return -1.0
  let ttl = expiry - epochTime()
  return max(0.0, ttl)

# ============================
# Distributed sliding window (GCRA)
# Generic Cell Rate Algorithm
# ============================

type
  GcraState = object
    tat: float   # theoretical arrival time

  GcraResult = object
    allowed: bool
    remaining: int
    resetIn: float   # seconds

proc gcraCheck(key: string, rate: float, burst: int): GcraResult =
  ## Generic Cell Rate Algorithm (leaky bucket as a meter)
  ## rate: requests per second
  ## burst: max burst size
  
  let now = epochTime()
  let period = 1.0 / rate
  let burstPeriod = period * float(burst)
  
  let tatStr = redisGet(key)
  let tat = if tatStr.len > 0: parseFloat(tatStr) else: now
  
  let newTat = max(tat, now) + period
  let allowed = newTat - now <= burstPeriod + period
  
  if allowed:
    redisSet(key, fmt"{newTat:.6f}", int(burstPeriod * 1000) + 1000)
    let remaining = int((burstPeriod + period - (newTat - now)) / period)
    return GcraResult(allowed: true, remaining: max(0, remaining), resetIn: 0.0)
  else:
    let resetIn = newTat - now - burstPeriod
    return GcraResult(allowed: false, remaining: 0, resetIn: resetIn)

# ============================
# Multi-tier rate limits
# ============================

type
  TierConfig = object
    name: string
    requestsPerSecond: float
    burstSize: int
    windowMinutes: int
    dailyLimit: int

let tiers = {
  "free": TierConfig(name: "free", requestsPerSecond: 1.0, burstSize: 5,
                     windowMinutes: 1, dailyLimit: 1000),
  "basic": TierConfig(name: "basic", requestsPerSecond: 10.0, burstSize: 20,
                      windowMinutes: 1, dailyLimit: 10000),
  "pro": TierConfig(name: "pro", requestsPerSecond: 100.0, burstSize: 200,
                    windowMinutes: 1, dailyLimit: 100000),
  "enterprise": TierConfig(name: "enterprise", requestsPerSecond: 1000.0, burstSize: 2000,
                           windowMinutes: 1, dailyLimit: -1),  # unlimited
}.toTable()

type
  MultiTierResult = object
    allowed: bool
    tier: string
    secondsRemaining: int
    minuteRemaining: int
    dailyRemaining: int
    resetIn: float

proc checkMultiTier(userId, tier: string): MultiTierResult =
  let config = tiers.getOrDefault(tier, tiers["free"])
  
  # Per-second rate check (GCRA)
  let secKey = fmt"rl:s:{userId}"
  let secResult = gcraCheck(secKey, config.requestsPerSecond, config.burstSize)
  
  if not secResult.allowed:
    return MultiTierResult(allowed: false, tier: tier,
                           secondsRemaining: 0, minuteRemaining: -1,
                           dailyRemaining: -1, resetIn: secResult.resetIn)
  
  # Per-minute counter
  let minute = int(epochTime() / 60)
  let minKey = fmt"rl:m:{userId}:{minute}"
  let minCount = redisIncr(minKey)
  if minCount == 1:
    redisExpire(minKey, 65_000)  # 65 seconds
  
  let perMinuteLimit = int(config.requestsPerSecond * 60)
  if minCount > perMinuteLimit:
    return MultiTierResult(allowed: false, tier: tier,
                           secondsRemaining: secResult.remaining,
                           minuteRemaining: 0, dailyRemaining: -1,
                           resetIn: 60.0 - float(int(epochTime()) mod 60))
  
  # Daily counter
  var dailyRemaining = -1
  if config.dailyLimit > 0:
    let day = int(epochTime() / 86400)
    let dayKey = fmt"rl:d:{userId}:{day}"
    let dayCount = redisIncr(dayKey)
    if dayCount == 1:
      redisExpire(dayKey, 86_400_000)  # 24 hours
    
    if dayCount > config.dailyLimit:
      return MultiTierResult(allowed: false, tier: tier,
                             secondsRemaining: secResult.remaining,
                             minuteRemaining: perMinuteLimit - minCount,
                             dailyRemaining: 0,
                             resetIn: float(86400 - int(epochTime()) mod 86400))
    
    dailyRemaining = config.dailyLimit - dayCount
  
  return MultiTierResult(
    allowed: true, tier: tier,
    secondsRemaining: secResult.remaining,
    minuteRemaining: perMinuteLimit - minCount,
    dailyRemaining: dailyRemaining,
    resetIn: 0.0
  )

# ============================
# IP-based blocking
# ============================

type
  IpBlockList = object
    blocked: Table[string, tuple[reason: string, until: float]]
    violations: Table[string, int]

var blockList = IpBlockList(
  blocked: initTable[string, tuple[reason: string, until: float]](),
  violations: initTable[string, int]()
)

proc isBlocked(ip: string): bool =
  if ip notin blockList.blocked: return false
  let entry = blockList.blocked[ip]
  if epochTime() > entry.until:
    blockList.blocked.del(ip)
    return false
  return true

proc recordViolation(ip: string) =
  blockList.violations[ip] = blockList.violations.getOrDefault(ip, 0) + 1
  let violations = blockList.violations[ip]
  
  let blockDuration = case violations
    of 1..3: 60.0      # 1 minute
    of 4..9: 3600.0    # 1 hour
    else: 86400.0      # 24 hours
  
  blockList.blocked[ip] = ("Too many violations", epochTime() + blockDuration)
  echo fmt"[BlockList] Blocked {ip} for {blockDuration:.0f}s (violations: {violations})"

# ============================
# Demo
# ============================

proc demo() =
  echo "=== Distributed Rate Limiting Demo ==="
  
  echo "\n--- GCRA single user ---"
  for i in 0..<8:
    let result = gcraCheck("user:123", 3.0, 5)  # 3 req/sec, burst 5
    let icon = if result.allowed: "✓" else: "✗"
    echo fmt"  Request {i+1}: {icon} remaining={result.remaining}"
  
  echo "\n--- Multi-tier rate limiting ---"
  let tiers_to_test = ["free", "basic", "pro"]
  for tierName in tiers_to_test:
    let result = checkMultiTier(fmt"user_{tierName}", tierName)
    echo fmt"  [{tierName:^10}] allowed={result.allowed} " &
         fmt"sec_remaining={result.secondsRemaining} " &
         fmt"daily={result.dailyRemaining}"
  
  echo "\n--- IP violation tracking ---"
  let testIp = "10.0.0.1"
  for i in 0..<5:
    recordViolation(testIp)
  echo fmt"  IP {testIp} blocked: {isBlocked(testIp)}"

demo()
```

---

## 📝 สรุป Part 41

| Steps | หัวข้อ |
|-------|--------|
| 586 | Token bucket, sliding window, leaky bucket |
| 587 | Rate limiter with X-RateLimit-* headers |
| 588-600 | GCRA, multi-tier limits, IP blocking |

---

**← [Part 40: RPC](part_40_rpc.md) | [Part 42: Search Engine →](part_42_search.md)**
