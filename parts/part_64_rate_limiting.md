# Part 64: Rate Limiting & Throttling
## Steps 931-945: Protecting APIs from Abuse

---

## 🎯 เป้าหมายของ Part นี้

- Token bucket algorithm
- Sliding window counter
- Fixed window counter
- Leaky bucket
- Distributed rate limiting
- Per-user, per-IP, per-endpoint limits
- Rate limit headers (X-RateLimit-*)

---

## Step 931: Rate Limit Algorithms

```nim
import tables, strformat, times, sequtils, json, strutils, options, algorithm, math

# ============================
# Token bucket
# ============================

type
  TokenBucket = object
    capacity: float       # max tokens
    tokens: float         # current tokens
    refillRate: float     # tokens per second
    lastRefill: float     # epoch time

proc newTokenBucket(capacity, refillRate: float): TokenBucket =
  TokenBucket(
    capacity: capacity,
    tokens: capacity,
    refillRate: refillRate,
    lastRefill: epochTime()
  )

proc refill(tb: var TokenBucket) =
  let now = epochTime()
  let elapsed = now - tb.lastRefill
  let newTokens = elapsed * tb.refillRate
  tb.tokens = min(tb.capacity, tb.tokens + newTokens)
  tb.lastRefill = now

proc tryConsume(tb: var TokenBucket, tokens = 1.0): bool =
  refill(tb)
  if tb.tokens >= tokens:
    tb.tokens -= tokens
    return true
  return false

proc waitTime(tb: var TokenBucket, tokens = 1.0): float =
  refill(tb)
  if tb.tokens >= tokens: return 0.0
  let needed = tokens - tb.tokens
  return needed / tb.refillRate

# ============================
# Leaky bucket
# ============================

type
  LeakyBucket = object
    capacity: int
    queue: seq[string]    # requests waiting
    leakRate: float       # requests per second
    lastLeak: float

proc newLeakyBucket(capacity: int, leakRate: float): LeakyBucket =
  LeakyBucket(
    capacity: capacity,
    queue: @[],
    leakRate: leakRate,
    lastLeak: epochTime()
  )

proc leak(lb: var LeakyBucket) =
  let now = epochTime()
  let elapsed = now - lb.lastLeak
  let toProcess = int(elapsed * lb.leakRate)
  if toProcess > 0:
    let removeCount = min(toProcess, lb.queue.len)
    lb.queue = lb.queue[removeCount..^1]
    lb.lastLeak = now

proc enqueue(lb: var LeakyBucket, requestId: string): bool =
  leak(lb)
  if lb.queue.len >= lb.capacity:
    return false  # drop
  lb.queue.add(requestId)
  return true

proc leakQueueLen(lb: var LeakyBucket): int =
  leak(lb)
  lb.queue.len

# ============================
# Fixed window counter
# ============================

type
  FixedWindowLimit = object
    windowSize: float   # seconds
    limit: int
    windows: Table[string, tuple[count: int, windowStart: float]]

proc newFixedWindowLimit(windowSize: float, limit: int): FixedWindowLimit =
  FixedWindowLimit(
    windowSize: windowSize,
    limit: limit,
    windows: initTable[string, tuple[count: int, windowStart: float]]()
  )

proc allowFixed(fwl: var FixedWindowLimit, key: string): tuple[allowed: bool, remaining: int, reset: float] =
  let now = epochTime()

  if key notin fwl.windows:
    fwl.windows[key] = (0, now)

  var (count, windowStart) = fwl.windows[key]

  # New window?
  if now - windowStart >= fwl.windowSize:
    count = 0
    windowStart = now

  if count < fwl.limit:
    inc count
    fwl.windows[key] = (count, windowStart)
    return (true, fwl.limit - count, windowStart + fwl.windowSize)
  else:
    return (false, 0, windowStart + fwl.windowSize)

# ============================
# Sliding window log
# ============================

type
  SlidingWindowLog = object
    windowSize: float
    limit: int
    requests: Table[string, seq[float]]  # key -> timestamps

proc newSlidingWindowLog(windowSize: float, limit: int): SlidingWindowLog =
  SlidingWindowLog(
    windowSize: windowSize,
    limit: limit,
    requests: initTable[string, seq[float]]()
  )

proc allowSliding(swl: var SlidingWindowLog, key: string): tuple[allowed: bool, remaining: int] =
  let now = epochTime()
  let cutoff = now - swl.windowSize

  if key notin swl.requests:
    swl.requests[key] = @[]

  # Remove old entries
  swl.requests[key] = swl.requests[key].filterIt(it > cutoff)

  let count = swl.requests[key].len
  if count < swl.limit:
    swl.requests[key].add(now)
    return (true, swl.limit - count - 1)
  return (false, 0)

proc slidingWindowCount(swl: var SlidingWindowLog, key: string): int =
  let now = epochTime()
  let cutoff = now - swl.windowSize
  if key notin swl.requests: return 0
  swl.requests[key].filterIt(it > cutoff).len

# ============================
# Multi-tier rate limiter
# ============================

type
  RateLimitTier = object
    name: string
    window: float
    limit: int

  MultiTierLimiter = object
    tiers: seq[RateLimitTier]
    logs: Table[string, SlidingWindowLog]

proc newMultiTierLimiter(tiers: seq[RateLimitTier]): MultiTierLimiter =
  var limiter = MultiTierLimiter(
    tiers: tiers,
    logs: initTable[string, SlidingWindowLog]()
  )
  for tier in tiers:
    limiter.logs[tier.name] = newSlidingWindowLog(tier.window, tier.limit)
  return limiter

proc checkAllTiers(ml: var MultiTierLimiter, key: string): tuple[allowed: bool, tier: string] =
  for tier in ml.tiers:
    let (allowed, _) = allowSliding(ml.logs[tier.name], key)
    if not allowed:
      return (false, tier.name)
  return (true, "")

# ============================
# Rate limit headers
# ============================

type
  RateLimitInfo = object
    limit: int
    remaining: int
    reset: float
    retryAfter: float

proc rateLimitHeaders(info: RateLimitInfo): Table[string, string] =
  result = initTable[string, string]()
  result["X-RateLimit-Limit"] = $info.limit
  result["X-RateLimit-Remaining"] = $info.remaining
  result["X-RateLimit-Reset"] = $int(info.reset)
  if info.retryAfter > 0:
    result["Retry-After"] = $int(ceil(info.retryAfter))

# ============================
# Per-user/IP limiter
# ============================

type
  LimitConfig = object
    ipLimit: int          # per minute
    userLimit: int        # per minute
    endpointLimit: int    # per minute per user per endpoint

  ApiRateLimiter = object
    config: LimitConfig
    ipLimiter: SlidingWindowLog
    userLimiter: SlidingWindowLog
    endpointLimiter: SlidingWindowLog

proc newApiRateLimiter(cfg: LimitConfig): ApiRateLimiter =
  ApiRateLimiter(
    config: cfg,
    ipLimiter: newSlidingWindowLog(60.0, cfg.ipLimit),
    userLimiter: newSlidingWindowLog(60.0, cfg.userLimit),
    endpointLimiter: newSlidingWindowLog(60.0, cfg.endpointLimit)
  )

proc checkRequest(limiter: var ApiRateLimiter,
                  ip, userId, endpoint: string
                 ): tuple[allowed: bool, reason: string, headers: Table[string, string]] =
  # Check IP
  let (ipOk, ipRemaining) = allowSliding(limiter.ipLimiter, ip)
  if not ipOk:
    return (false, "IP rate limit exceeded",
      rateLimitHeaders(RateLimitInfo(
        limit: limiter.config.ipLimit,
        remaining: 0,
        reset: epochTime() + 60,
        retryAfter: 60
      )))

  # Check user
  if userId.len > 0:
    let (userOk, userRemaining) = allowSliding(limiter.userLimiter, userId)
    if not userOk:
      return (false, "User rate limit exceeded",
        rateLimitHeaders(RateLimitInfo(
          limit: limiter.config.userLimit,
          remaining: 0,
          reset: epochTime() + 60,
          retryAfter: 60
        )))

    # Check endpoint
    let endpointKey = fmt"{userId}:{endpoint}"
    let (epOk, epRemaining) = allowSliding(limiter.endpointLimiter, endpointKey)
    if not epOk:
      return (false, "Endpoint rate limit exceeded",
        rateLimitHeaders(RateLimitInfo(
          limit: limiter.config.endpointLimit,
          remaining: 0,
          reset: epochTime() + 60,
          retryAfter: 60
        )))

    return (true, "", rateLimitHeaders(RateLimitInfo(
      limit: limiter.config.userLimit,
      remaining: userRemaining,
      reset: epochTime() + 60
    )))

  return (true, "", rateLimitHeaders(RateLimitInfo(
    limit: limiter.config.ipLimit,
    remaining: ipRemaining,
    reset: epochTime() + 60
  )))

# ============================
# Demo
# ============================

proc demo() =
  echo "=== Rate Limiting Demo ==="

  # Token bucket
  echo "\n--- Token bucket (10 tokens, 2/s refill) ---"
  var tb = newTokenBucket(10.0, 2.0)
  for i in 1..12:
    let ok = tryConsume(tb, 1.0)
    echo fmt"  Request {i}: {'allowed' if ok else 'denied'} (tokens: {tb.tokens:.1f})"

  # Fixed window
  echo "\n--- Fixed window (5 req/10s) ---"
  var fwl = newFixedWindowLimit(10.0, 5)
  for i in 1..7:
    let (ok, remaining, reset) = allowFixed(fwl, "user_1")
    echo fmt"  Request {i}: {'allowed' if ok else 'denied'} (remaining: {remaining})"

  # Sliding window
  echo "\n--- Sliding window log (5 req/10s) ---"
  var swl = newSlidingWindowLog(10.0, 5)
  for i in 1..7:
    let (ok, remaining) = allowSliding(swl, "user_2")
    echo fmt"  Request {i}: {'allowed' if ok else 'denied'} (remaining: {remaining})"

  # Leaky bucket
  echo "\n--- Leaky bucket (capacity=3, rate=1/s) ---"
  var lb = newLeakyBucket(3, 1.0)
  for i in 1..5:
    let ok = enqueue(lb, fmt"req_{i}")
    echo fmt"  Request {i}: {'queued' if ok else 'dropped'} (queue: {lb.queue.len})"

  # Multi-tier
  echo "\n--- Multi-tier limits ---"
  let tiers = @[
    RateLimitTier(name: "per_second", window: 1.0, limit: 3),
    RateLimitTier(name: "per_minute", window: 60.0, limit: 50),
  ]
  var ml = newMultiTierLimiter(tiers)
  for i in 1..5:
    let (ok, tier) = checkAllTiers(ml, "alice")
    let reason = if tier.len > 0: fmt" (blocked by: {tier})" else: ""
    echo fmt"  Request {i}: {'allowed' if ok else 'denied'}{reason}"

  # API limiter with headers
  echo "\n--- API rate limiter ---"
  let cfg = LimitConfig(ipLimit: 10, userLimit: 5, endpointLimit: 3)
  var apiLimiter = newApiRateLimiter(cfg)

  for i in 1..5:
    let (ok, reason, headers) = checkRequest(
      apiLimiter, "192.168.1.1", "alice", "/api/data")
    echo fmt"  Request {i}: {'allowed' if ok else 'denied'}"
    if headers.len > 0:
      echo fmt"    X-RateLimit-Remaining: {headers.getOrDefault(\"X-RateLimit-Remaining\", \"?\")}"
    if not ok:
      echo fmt"    Reason: {reason}"

demo()
```

---

## 📝 สรุป Part 64

| Steps | หัวข้อ |
|-------|--------|
| 931 | Token bucket, leaky bucket |
| 932-940 | Fixed window, sliding window log |
| 941-945 | Multi-tier limits, API rate limiter, rate limit headers |

---

**← [Part 63: CQRS](part_63_cqrs.md) | [Part 65: HTTP/2 & gRPC →](part_65_grpc.md)**
