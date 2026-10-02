# Part 36: Real-Time Features
## Steps 511-525: Real-Time ระดับ Production

---

## 🎯 เป้าหมายของ Part นี้

- Server-Sent Events (SSE)
- Long polling
- WebSocket บน Jester
- Real-time notifications
- Live dashboard
- Presence system

---

## Step 511: Server-Sent Events (SSE)

```nim
import asyncdispatch, asynchttpserver, strformat, times, json, tables, sequtils

# SSE is simpler than WebSocket:
# - One-directional (server -> client)
# - Auto-reconnect built into browsers
# - Works over HTTP/1.1
# - No special library needed

type
  SseClient = object
    id: string
    channel: string
    connectedAt: float
    lastEventId: int

  SseBroadcaster = object
    clients: Table[string, AsyncHttpRequest]  # clientId -> request
    channels: Table[string, seq[string]]      # channel -> clientIds

var broadcaster = SseBroadcaster(
  clients: initTable[string, AsyncHttpRequest](),
  channels: initTable[string, seq[string]]()
)

proc sseHeaders(): HttpHeaders =
  newHttpHeaders([
    ("Content-Type", "text/event-stream"),
    ("Cache-Control", "no-cache"),
    ("Connection", "keep-alive"),
    ("Access-Control-Allow-Origin", "*"),
    ("X-Accel-Buffering", "no"),  # nginx: disable buffering
  ])

proc formatSseEvent(data: JsonNode, event: string = "", id: int = 0): string =
  var msg = ""
  if event.len > 0:
    msg &= "event: " & event & "\n"
  if id > 0:
    msg &= "id: " & $id & "\n"
  msg &= "data: " & $data & "\n\n"
  return msg

proc sendSseEvent(req: AsyncHttpRequest, data: JsonNode, event: string = "") {.async.} =
  let msg = formatSseEvent(data, event)
  try:
    await req.client.send(msg)
  except:
    discard  # client disconnected

# SSE endpoint handler
var eventCounter = 0
var sseClients: Table[string, AsyncHttpRequest] = initTable[string, AsyncHttpRequest]()

proc handleSse(req: Request) {.async.} =
  let path = req.url.path
  
  if path == "/events":
    # Keep-alive SSE connection
    await req.respond(Http200, "", sseHeaders())
    
    let clientId = fmt"client_{epochTime().int}"
    echo fmt"[SSE] Client connected: {clientId}"
    
    # Send initial connection event
    var welcome = formatSseEvent(%*{
      "type": "connected",
      "clientId": clientId,
      "timestamp": epochTime()
    }, "welcome")
    
    await req.client.send(welcome)
    
    # Keep sending heartbeats until client disconnects
    var counter = 0
    while true:
      try:
        await sleepAsync(5000)  # heartbeat every 5s
        inc counter
        let ping = formatSseEvent(%*{
          "type": "heartbeat",
          "counter": counter,
          "timestamp": epochTime()
        }, "heartbeat")
        await req.client.send(ping)
      except:
        echo fmt"[SSE] Client disconnected: {clientId}"
        break
    return
  
  await req.respond(Http404, "Not found")

# ============================
# Broadcast to all SSE clients
# ============================

proc broadcastSse(data: JsonNode, event: string = "") {.async.} =
  let msg = formatSseEvent(data, event)
  var toRemove: seq[string] = @[]
  
  for clientId, req in sseClients:
    try:
      await req.client.send(msg)
    except:
      toRemove.add(clientId)
  
  for clientId in toRemove:
    sseClients.del(clientId)
    echo fmt"[SSE] Removed disconnected client: {clientId}"

# Usage example - send notification when order placed
proc onOrderPlaced(orderId: int, userId: int) {.async.} =
  await broadcastSse(%*{
    "type": "order_placed",
    "orderId": orderId,
    "userId": userId,
    "timestamp": epochTime()
  }, "order")

echo "SSE setup complete"
echo "Connect with: curl -N http://localhost:8080/events"
```

---

## Step 512: Long Polling

```nim
import asyncdispatch, asynchttpserver, json, tables, times, strformat, sequtils

# Long polling: client waits up to N seconds for new data
# Server holds request until there's data or timeout

type
  PendingRequest = object
    req: Request
    userId: int
    since: float  # timestamp of last seen event
    deadline: float

  Notification = object
    id: int
    userId: int
    type_: string
    message: string
    data: JsonNode
    createdAt: float

var pendingRequests: seq[PendingRequest] = @[]
var notifications: seq[Notification] = @[]
var notifNextId = 1

proc getNotificationsSince(userId: int, since: float): seq[Notification] =
  notifications.filterIt(it.userId == userId and it.createdAt > since)

proc addNotification(userId: int, type_, message: string, data: JsonNode = nil) =
  let notif = Notification(
    id: notifNextId,
    userId: userId,
    type_: type_,
    message: message,
    data: if data.isNil: newJNull() else: data,
    createdAt: epochTime()
  )
  inc notifNextId
  notifications.add(notif)
  echo fmt"[Notif] Created for user {userId}: {type_}"
  
  # Wake up pending requests for this user
  for i, pending in pendingRequests:
    if pending.userId == userId:
      asyncCheck(proc() {.async.} =
        let notifs = getNotificationsSince(pending.userId, pending.since)
        if notifs.len > 0:
          let response = %*{
            "notifications": notifs.mapIt(%*{
              "id": it.id,
              "type": it.type_,
              "message": it.message,
              "createdAt": it.createdAt
            }),
            "count": notifs.len
          }
          await pending.req.respond(Http200, $response,
            newHttpHeaders([("Content-Type", "application/json")]))
      )()

proc handleLongPoll(req: Request) {.async.} =
  let path = req.url.path
  
  if req.reqMethod == HttpGet and path.startsWith("/poll/"):
    let userId = path[6..^1].parseInt()
    let since = req.url.query.split('=').getOrDefault(1, "0").parseFloat()
    
    # Check for immediate notifications
    let immediate = getNotificationsSince(userId, since)
    if immediate.len > 0:
      let response = %*{
        "notifications": immediate.mapIt(%*{
          "id": it.id, "type": it.type_, "message": it.message
        }),
        "count": immediate.len
      }
      await req.respond(Http200, $response,
        newHttpHeaders([("Content-Type", "application/json")]))
      return
    
    # No notifications yet - hold request for up to 30 seconds
    pendingRequests.add(PendingRequest(
      req: req,
      userId: userId,
      since: since,
      deadline: epochTime() + 30.0
    ))
    
    # Timeout after 30s
    await sleepAsync(30_000)
    
    # Return empty if still no notifications
    await req.respond(Http200, """{"notifications":[],"count":0}""",
      newHttpHeaders([("Content-Type", "application/json")]))
    return
  
  await req.respond(Http404, """{"error":"not found"}""",
    newHttpHeaders([("Content-Type", "application/json")]))

# Demo: add some notifications
proc demo() {.async.} =
  echo "=== Long Polling Demo ==="
  
  addNotification(1, "message", "You have a new message from Alice")
  addNotification(1, "order", "Order #1001 shipped", %*{"orderId": 1001})
  addNotification(2, "message", "Bob sent you a message")
  
  let user1Notifs = getNotificationsSince(1, 0)
  echo fmt"User 1 has {user1Notifs.len} notifications"
  for n in user1Notifs:
    echo fmt"  [{n.type_}] {n.message}"

waitFor demo()
```

---

## Step 513-525: Complete Real-Time Dashboard

```nim
# realtime_dashboard.nim - Live metrics dashboard

import asyncdispatch, asynchttpserver, json, tables, times, strformat,
       strutils, sequtils, math

# ============================
# Metrics collection
# ============================

type
  TimeSeriesPoint = object
    timestamp: float
    value: float

  MetricSeries = object
    name: string
    points: seq[TimeSeriesPoint]
    maxPoints: int

  Dashboard = object
    series: Table[string, MetricSeries]
    sseClients: Table[string, AsyncHttpRequest]

var dashboard = Dashboard(
  series: initTable[string, MetricSeries](),
  sseClients: initTable[string, AsyncHttpRequest]()
)

proc addSeries(dash: var Dashboard, name: string, maxPoints: int = 60) =
  dash.series[name] = MetricSeries(name: name, points: @[], maxPoints: maxPoints)

proc recordMetric(dash: var Dashboard, name: string, value: float) =
  if name notin dash.series:
    dash.addSeries(name)
  
  var series = dash.series[name]
  series.points.add(TimeSeriesPoint(timestamp: epochTime(), value: value))
  
  # Keep only recent points
  if series.points.len > series.maxPoints:
    series.points = series.points[series.points.len - series.maxPoints .. ^1]
  
  dash.series[name] = series

proc getMetricJson(dash: Dashboard, name: string): JsonNode =
  if name notin dash.series:
    return newJArray()
  
  let series = dash.series[name]
  var arr = newJArray()
  for pt in series.points:
    arr.add(%*{"ts": pt.timestamp, "v": pt.value})
  return arr

proc broadcastMetrics(dash: var Dashboard) {.async.} =
  var data = newJObject()
  for name, _ in dash.series:
    data[name] = dash.getMetricJson(name)
  
  let msg = "data: " & $(%*{"type": "metrics", "data": data}) & "\n\n"
  
  var toRemove: seq[string] = @[]
  for clientId, req in dash.sseClients:
    try:
      await req.client.send(msg)
    except:
      toRemove.add(clientId)
  
  for clientId in toRemove:
    dash.sseClients.del(clientId)

# ============================
# Dashboard HTML
# ============================

const dashboardHtml = """<!DOCTYPE html>
<html>
<head>
<meta charset="utf-8">
<title>Nim Backend Dashboard</title>
<script src="https://cdn.jsdelivr.net/npm/chart.js@4/dist/chart.umd.min.js"></script>
<style>
  body { font-family: system-ui; background: #0d1117; color: #c9d1d9; margin: 0; padding: 20px; }
  h1 { color: #58a6ff; }
  .grid { display: grid; grid-template-columns: repeat(2, 1fr); gap: 20px; }
  .card { background: #161b22; border: 1px solid #30363d; border-radius: 8px; padding: 16px; }
  .metric { font-size: 2em; font-weight: bold; color: #58a6ff; }
  .label { color: #8b949e; font-size: 12px; text-transform: uppercase; }
  canvas { max-height: 200px; }
  .status { color: #3fb950; }
</style>
</head>
<body>
<h1>Nim Backend Dashboard</h1>
<div id="status" class="status">Connecting...</div>

<div class="grid" id="metrics">
  <div class="card">
    <div class="label">Requests/sec</div>
    <div class="metric" id="rps">0</div>
    <canvas id="rpsChart"></canvas>
  </div>
  <div class="card">
    <div class="label">Response Time (ms)</div>
    <div class="metric" id="latency">0</div>
    <canvas id="latencyChart"></canvas>
  </div>
  <div class="card">
    <div class="label">Active Connections</div>
    <div class="metric" id="connections">0</div>
  </div>
  <div class="card">
    <div class="label">Error Rate</div>
    <div class="metric" id="errors">0%</div>
  </div>
</div>

<script>
const charts = {};

function initChart(id, label) {
  const ctx = document.getElementById(id);
  if (!ctx) return null;
  return new Chart(ctx, {
    type: 'line',
    data: {
      datasets: [{
        label,
        data: [],
        borderColor: '#58a6ff',
        backgroundColor: 'rgba(88,166,255,0.1)',
        tension: 0.4,
        fill: true
      }]
    },
    options: {
      animation: false,
      scales: {
        x: { display: false },
        y: { ticks: { color: '#8b949e' }, grid: { color: '#21262d' } }
      },
      plugins: { legend: { display: false } }
    }
  });
}

charts.rps = initChart('rpsChart', 'Req/sec');
charts.latency = initChart('latencyChart', 'ms');

const es = new EventSource('/dashboard/events');

es.onopen = () => {
  document.getElementById('status').textContent = '● Connected';
};

es.onmessage = (e) => {
  const msg = JSON.parse(e.data);
  if (msg.type !== 'metrics') return;
  
  const d = msg.data;
  
  if (d.rps && d.rps.length > 0) {
    const last = d.rps[d.rps.length - 1];
    document.getElementById('rps').textContent = last.v.toFixed(1);
    if (charts.rps) {
      charts.rps.data.datasets[0].data = d.rps.slice(-30).map(p => ({x: p.ts, y: p.v}));
      charts.rps.update('none');
    }
  }
  
  if (d.latency && d.latency.length > 0) {
    const last = d.latency[d.latency.length - 1];
    document.getElementById('latency').textContent = last.v.toFixed(1);
    if (charts.latency) {
      charts.latency.data.datasets[0].data = d.latency.slice(-30).map(p => ({x: p.ts, y: p.v}));
      charts.latency.update('none');
    }
  }
  
  if (d.connections && d.connections.length > 0)
    document.getElementById('connections').textContent = d.connections[d.connections.length-1].v.toFixed(0);
  
  if (d.error_rate && d.error_rate.length > 0)
    document.getElementById('errors').textContent = d.error_rate[d.error_rate.length-1].v.toFixed(1) + '%';
};

es.onerror = () => {
  document.getElementById('status').textContent = '○ Disconnected (reconnecting...)';
  document.getElementById('status').style.color = '#f85149';
};
</script>
</body>
</html>"""

# ============================
# HTTP Handler
# ============================

proc handleDashboard(req: Request) {.async.} =
  let path = req.url.path
  
  if path == "/" or path == "/dashboard":
    await req.respond(Http200, dashboardHtml,
      newHttpHeaders([("Content-Type", "text/html; charset=utf-8")]))
    return
  
  if path == "/dashboard/events":
    await req.respond(Http200, "", newHttpHeaders([
      ("Content-Type", "text/event-stream"),
      ("Cache-Control", "no-cache"),
      ("Connection", "keep-alive"),
    ]))
    
    let clientId = fmt"dash_{epochTime().int}"
    dashboard.sseClients[clientId] = req
    echo fmt"[Dashboard] Client connected: {clientId}"
    
    while clientId in dashboard.sseClients:
      await sleepAsync(100)
    
    echo fmt"[Dashboard] Client disconnected: {clientId}"
    return
  
  await req.respond(Http404, """{"error":"not found"}""",
    newHttpHeaders([("Content-Type", "application/json")]))

# ============================
# Metrics simulation
# ============================

proc simulateMetrics() {.async.} =
  # Add metrics series
  dashboard.addSeries("rps", 60)
  dashboard.addSeries("latency", 60)
  dashboard.addSeries("connections", 60)
  dashboard.addSeries("error_rate", 60)
  
  var tick = 0
  while true:
    inc tick
    
    # Simulate realistic metrics
    let rps = 100.0 + sin(float(tick) * 0.1) * 30.0 + float(rand(20))
    let latency = 25.0 + abs(sin(float(tick) * 0.05)) * 15.0 + float(rand(10))
    let connections = 50.0 + float(tick mod 20)
    let errorRate = if tick mod 50 == 0: 2.0 else: 0.1
    
    dashboard.recordMetric("rps", rps)
    dashboard.recordMetric("latency", latency)
    dashboard.recordMetric("connections", connections)
    dashboard.recordMetric("error_rate", errorRate)
    
    # Broadcast to SSE clients
    if dashboard.sseClients.len > 0:
      await dashboard.broadcastMetrics()
    
    await sleepAsync(1000)

# ============================
# Demo (without running server)
# ============================

proc demo() =
  echo "=== Real-Time Dashboard Demo ==="
  dashboard.addSeries("rps", 60)
  dashboard.addSeries("latency", 60)
  
  # Simulate some data points
  for i in 0..<10:
    dashboard.recordMetric("rps", 100.0 + float(i * 5))
    dashboard.recordMetric("latency", 20.0 + float(i))
  
  echo fmt"RPS series: {dashboard.series[\"rps\"].points.len} points"
  echo fmt"Latency series: {dashboard.series[\"latency\"].points.len} points"
  
  let rpsData = dashboard.getMetricJson("rps")
  echo fmt"Latest RPS: {rpsData[rpsData.len-1][\"v\"].getFloat():.1f}"
  
  echo "\nDashboard HTML size: " & $dashboardHtml.len & " bytes"
  echo "SSE endpoint: /dashboard/events"
  echo "Dashboard URL: /dashboard"

demo()
```

---

## 📝 สรุป Part 36

| Steps | หัวข้อ |
|-------|--------|
| 511 | Server-Sent Events (SSE) |
| 512 | Long polling |
| 513-525 | Real-time dashboard with SSE + Chart.js |

---

**← [Part 35: Advanced Patterns](part_35_advanced_patterns.md) | [Part 37: Database Advanced →](part_37_database_advanced.md)**
