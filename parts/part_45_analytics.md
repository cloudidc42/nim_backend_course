# Part 45: Analytics System
## Steps 646-660: Analytics ระดับ Production

---

## 🎯 เป้าหมายของ Part นี้

- Event tracking
- Aggregation pipeline
- Time-series data
- Funnel analysis
- Cohort analysis
- Real-time dashboards

---

## Step 646: Event Tracking

```nim
import times, tables, strformat, json, sequtils, strutils, algorithm, math

# ============================
# Event types
# ============================

type
  EventProperties = Table[string, JsonNode]

  AnalyticsEvent = object
    id: string
    eventName: string      # "page_view", "button_click", "purchase"
    userId: string
    sessionId: string
    deviceId: string
    properties: EventProperties
    timestamp: float
    ipAddress: string
    userAgent: string

  EventStore = object
    events: seq[AnalyticsEvent]
    counter: int

var store = EventStore(events: @[], counter: 0)

proc eventId(): string =
  inc store.counter
  fmt"evt_{store.counter}_{int(epochTime() * 1000) mod 1_000_000}"

proc track(eventName, userId: string,
           properties: EventProperties = initTable[string, JsonNode](),
           sessionId = "", deviceId = ""): AnalyticsEvent =
  let evt = AnalyticsEvent(
    id: eventId(),
    eventName: eventName,
    userId: userId,
    sessionId: if sessionId.len > 0: sessionId else: fmt"sess_{int(epochTime()) mod 100000}",
    deviceId: deviceId,
    properties: properties,
    timestamp: epochTime()
  )
  store.events.add(evt)
  return evt

proc trackPageView(userId, page, referrer: string) =
  discard track("page_view", userId, {
    "page": %page,
    "referrer": %referrer
  }.toTable())

proc trackPurchase(userId: string, amount: float, productId: int, currency = "USD") =
  discard track("purchase", userId, {
    "amount": %amount,
    "product_id": %productId,
    "currency": %currency
  }.toTable())

proc trackButtonClick(userId, buttonId, page: string) =
  discard track("button_click", userId, {
    "button_id": %buttonId,
    "page": %page
  }.toTable())

# ============================
# Basic aggregations
# ============================

proc countEvents(eventName: string, since: float = 0.0): int =
  store.events.countIt(it.eventName == eventName and it.timestamp >= since)

proc uniqueUsers(eventName: string, since: float = 0.0): int =
  var users: seq[string]
  for evt in store.events:
    if evt.eventName == eventName and evt.timestamp >= since:
      if evt.userId notin users:
        users.add(evt.userId)
  return users.len

proc eventsByHour(eventName: string, hours = 24): seq[tuple[hour: int, count: int]] =
  var hourCounts: Table[int, int]
  let since = epochTime() - float(hours * 3600)
  
  for evt in store.events:
    if evt.eventName == eventName and evt.timestamp >= since:
      let h = int(evt.timestamp / 3600) mod 24
      hourCounts[h] = hourCounts.getOrDefault(h, 0) + 1
  
  var result: seq[tuple[hour: int, count: int]]
  for h in 0..<hours:
    result.add((h, hourCounts.getOrDefault(h, 0)))
  return result

# ============================
# Demo
# ============================

proc demo() =
  echo "=== Event Tracking Demo ==="
  
  # Simulate user activity
  for i in 0..<5:
    let userId = fmt"user_{i mod 3}"
    trackPageView(userId, "/home", "google.com")
    trackButtonClick(userId, "signup_cta", "/home")
  
  trackPageView("user_0", "/pricing", "/home")
  trackPurchase("user_0", 99.99, 101)
  trackPurchase("user_1", 49.99, 102)
  
  echo fmt"\nTotal events: {store.events.len}"
  echo fmt"page_view count: {countEvents(\"page_view\")}"
  echo fmt"purchase count: {countEvents(\"purchase\")}"
  echo fmt"unique users (page_view): {uniqueUsers(\"page_view\")}"
  
  echo "\nRecent events:"
  for evt in store.events[^5..^1]:
    echo fmt"  [{evt.eventName}] user={evt.userId}"

demo()
```

---

## Step 647-660: Funnel & Cohort Analysis

```nim
import times, tables, strformat, sequtils, algorithm, math, strutils, options

# ============================
# Funnel analysis
# ============================

type
  FunnelStep = object
    eventName: string
    conditions: Table[string, string]  # property filters

  FunnelResult = object
    steps: seq[tuple[name: string, count: int, dropoffRate: float]]
    overallConversion: float

proc analyzeFunnel(steps: seq[FunnelStep], events: seq[AnalyticsEvent],
                   windowSecs = 3600.0): FunnelResult =
  ## Count users that complete each step in order within the time window
  
  # Group events by user
  var userEvents: Table[string, seq[AnalyticsEvent]]
  for evt in events:
    if evt.userId notin userEvents:
      userEvents[evt.userId] = @[]
    userEvents[evt.userId].add(evt)
  
  # Sort each user's events by time
  for userId in userEvents.keys:
    userEvents[userId].sort(proc(a, b: AnalyticsEvent): int = cmp(a.timestamp, b.timestamp))
  
  var stepCounts = newSeq[int](steps.len)
  
  for userId, userEvts in userEvents:
    var stepIdx = 0
    var stepStartTime = 0.0
    
    for evt in userEvts:
      if stepIdx >= steps.len: break
      
      let step = steps[stepIdx]
      if evt.eventName != step.eventName: continue
      
      # Check time window
      if stepIdx > 0 and evt.timestamp - stepStartTime > windowSecs:
        stepIdx = 0
        stepStartTime = 0.0
        continue
      
      inc stepCounts[stepIdx]
      
      if stepIdx == 0:
        stepStartTime = evt.timestamp
      
      inc stepIdx
  
  var stepResults: seq[tuple[name: string, count: int, dropoffRate: float]]
  let topCount = if stepCounts[0] > 0: stepCounts[0] else: 1
  
  for i, step in steps:
    let count = stepCounts[i]
    let prevCount = if i > 0: stepCounts[i-1] else: count
    let dropoff = if prevCount > 0: 1.0 - float(count) / float(prevCount) else: 0.0
    stepResults.add((step.eventName, count, dropoff))
  
  let lastCount = stepCounts[^1]
  let overallConv = if topCount > 0: float(lastCount) / float(topCount) else: 0.0
  
  return FunnelResult(steps: stepResults, overallConversion: overallConv)

# ============================
# Cohort analysis
# ============================

type
  CohortData = object
    cohortPeriod: string    # "2024-01" (monthly cohort)
    users: seq[string]
    retention: seq[float]   # retention by period (0=100%, 1=period1, etc.)

proc buildCohorts(events: seq[AnalyticsEvent],
                  cohortEvent = "signup",
                  retentionEvent = "page_view",
                  periods = 6): seq[CohortData] =
  ## Build monthly cohort retention
  
  # Find signup dates per user
  var firstSeen: Table[string, float]
  for evt in events:
    if evt.eventName == cohortEvent:
      if evt.userId notin firstSeen:
        firstSeen[evt.userId] = evt.timestamp
  
  # Group users by cohort month
  var cohorts: Table[string, seq[string]]
  for userId, ts in firstSeen:
    let dt = fromUnix(int(ts))
    let cohortKey = fmt"{dt.year}-{int(dt.month):02d}"
    if cohortKey notin cohorts:
      cohorts[cohortKey] = @[]
    cohorts[cohortKey].add(userId)
  
  # Calculate retention for each cohort
  var result: seq[CohortData]
  
  for cohortKey, users in cohorts:
    var retention = newSeq[float](periods)
    retention[0] = 1.0  # 100% in period 0
    
    for period in 1..<periods:
      let periodStart = epochTime() - float((periods - period) * 30 * 86400)
      let periodEnd = periodStart + float(30 * 86400)
      
      var activeUsers = 0
      for userId in users:
        let hasActivity = events.anyIt(
          it.userId == userId and
          it.eventName == retentionEvent and
          it.timestamp >= periodStart and
          it.timestamp < periodEnd
        )
        if hasActivity: inc activeUsers
      
      retention[period] = if users.len > 0:
        float(activeUsers) / float(users.len)
      else: 0.0
    
    result.add(CohortData(
      cohortPeriod: cohortKey,
      users: users,
      retention: retention
    ))
  
  result.sort(proc(a, b: CohortData): int = cmp(a.cohortPeriod, b.cohortPeriod))
  return result

# ============================
# Time-series metrics
# ============================

type
  TimeSeriesMetric = object
    name: string
    interval: string    # "hourly" | "daily" | "weekly" | "monthly"
    points: seq[tuple[ts: float, value: float]]

proc dailyActiveUsers(events: seq[AnalyticsEvent], days = 30): TimeSeriesMetric =
  var dailyCounts: Table[string, seq[string]]
  
  for evt in events:
    let dt = fromUnix(int(evt.timestamp))
    let dayKey = fmt"{dt.year}-{int(dt.month):02d}-{dt.monthday:02d}"
    if dayKey notin dailyCounts:
      dailyCounts[dayKey] = @[]
    if evt.userId notin dailyCounts[dayKey]:
      dailyCounts[dayKey].add(evt.userId)
  
  var points: seq[tuple[ts: float, value: float]]
  var keys = toSeq(dailyCounts.keys)
  keys.sort()
  
  for k in keys:
    let ts = parseTime(k, "yyyy-MM-dd", utc()).toTime().toUnixFloat()
    points.add((ts, float(dailyCounts[k].len)))
  
  return TimeSeriesMetric(
    name: "dau",
    interval: "daily",
    points: points
  )

proc revenueByDay(events: seq[AnalyticsEvent], days = 30): TimeSeriesMetric =
  var dailyRevenue: Table[string, float]
  
  for evt in events:
    if evt.eventName != "purchase": continue
    let dt = fromUnix(int(evt.timestamp))
    let dayKey = fmt"{dt.year}-{int(dt.month):02d}-{dt.monthday:02d}"
    
    if "amount" in evt.properties:
      let amount = evt.properties["amount"].getFloat()
      dailyRevenue[dayKey] = dailyRevenue.getOrDefault(dayKey, 0.0) + amount
  
  var points: seq[tuple[ts: float, value: float]]
  var keys = toSeq(dailyRevenue.keys)
  keys.sort()
  
  for k in keys:
    let ts = parseTime(k, "yyyy-MM-dd", utc()).toTime().toUnixFloat()
    points.add((ts, dailyRevenue[k]))
  
  return TimeSeriesMetric(
    name: "daily_revenue",
    interval: "daily",
    points: points
  )

# ============================
# Demo
# ============================

proc demo() =
  echo "=== Analytics Demo ==="
  
  # Build event dataset
  var allEvents: seq[AnalyticsEvent]
  let baseTime = epochTime() - 7 * 86400.0
  
  # Simulate signups and activities
  for i in 0..<20:
    let userId = fmt"user_{i}"
    let signupTime = baseTime + float(i * 3600)
    
    allEvents.add(AnalyticsEvent(
      id: fmt"e{i}_signup",
      eventName: "signup",
      userId: userId,
      sessionId: fmt"sess_{i}",
      deviceId: "",
      properties: initTable[string, JsonNode](),
      timestamp: signupTime
    ))
    
    allEvents.add(AnalyticsEvent(
      id: fmt"e{i}_view",
      eventName: "page_view",
      userId: userId,
      sessionId: fmt"sess_{i}",
      deviceId: "",
      properties: {"page": %"/home"}.toTable(),
      timestamp: signupTime + 60
    ))
    
    if i mod 3 == 0:
      allEvents.add(AnalyticsEvent(
        id: fmt"e{i}_purchase",
        eventName: "purchase",
        userId: userId,
        sessionId: fmt"sess_{i}",
        deviceId: "",
        properties: {"amount": %(float(i) * 10 + 9.99), "product_id": %(i mod 5 + 1)}.toTable(),
        timestamp: signupTime + 300
      ))
  
  echo fmt"\nTotal events: {allEvents.len}"
  
  # Funnel analysis
  echo "\n--- Funnel: signup -> page_view -> purchase ---"
  let funnel = analyzeFunnel(@[
    FunnelStep(eventName: "signup"),
    FunnelStep(eventName: "page_view"),
    FunnelStep(eventName: "purchase"),
  ], allEvents, windowSecs: 3600.0)
  
  for step in funnel.steps:
    echo fmt"  {step.name}: {step.count} users (dropoff: {step.dropoffRate * 100:.1f}%)"
  echo fmt"  Overall conversion: {funnel.overallConversion * 100:.1f}%"
  
  # Revenue
  echo "\n--- Revenue Metrics ---"
  let revMetric = revenueByDay(allEvents)
  var totalRev = 0.0
  for pt in revMetric.points:
    totalRev += pt.value
  echo fmt"Total revenue ({revMetric.points.len} days): ${totalRev:.2f}"
  
  # DAU
  let dauMetric = dailyActiveUsers(allEvents)
  echo fmt"Days with activity: {dauMetric.points.len}"
  let avgDau = if dauMetric.points.len > 0:
    dauMetric.points.mapIt(it.value).foldl(a + b, 0.0) / float(dauMetric.points.len)
  else: 0.0
  echo fmt"Average DAU: {avgDau:.1f}"

demo()
```

---

## 📝 สรุป Part 45

| Steps | หัวข้อ |
|-------|--------|
| 646 | Event tracking, counting, aggregation |
| 647-660 | Funnel analysis, cohort analysis, time-series |

---

**← [Part 44: Payments](part_44_payments.md) | [Part 46: Notifications →](part_46_notifications.md)**
