# Part 48: API Gateway
## Steps 691-705: API Gateway ระดับ Production

---

## 🎯 เป้าหมายของ Part นี้

- Request routing
- Load balancing
- Circuit breaker integration
- Request/response transformation
- API versioning
- Canary deployments

---

## Step 691: Core API Gateway

```nim
import asyncdispatch, asynchttpserver, asyncnet, tables, strformat, times,
       strutils, sequtils, json, options, math

# ============================
# Gateway types
# ============================

type
  BackendServer = object
    id: string
    host: string
    port: int
    weight: int       # for weighted routing
    healthy: bool
    activeRequests: int
    totalRequests: int
    failedRequests: int
    avgResponseMs: float

  LoadBalanceStrategy = enum
    lbRoundRobin, lbLeastConn, lbWeighted, lbIpHash

  Route = object
    pattern: string       # "/api/users*", "/api/v2/**"
    backends: seq[BackendServer]
    strategy: LoadBalanceStrategy
    timeout: int          # ms
    retries: int
    stripPrefix: string   # remove prefix before forwarding
    addPrefix: string
    headers: Table[string, string]   # add to forwarded request
    rateLimit: int        # req/min per IP, 0 = unlimited
    auth: bool            # require JWT

  GatewayConfig = object
    routes: seq[Route]
    globalHeaders: Table[string, string]
    corsOrigins: seq[string]
    maxRequestSize: int

# ============================
# Load balancer
# ============================

var roundRobinCounters: Table[string, int] = initTable[string, int]()

proc selectBackend(route: Route): Option[BackendServer] =
  let healthy = route.backends.filterIt(it.healthy)
  
  if healthy.len == 0:
    return none(BackendServer)
  
  case route.strategy
  of lbRoundRobin:
    let key = route.pattern
    if key notin roundRobinCounters:
      roundRobinCounters[key] = 0
    let idx = roundRobinCounters[key] mod healthy.len
    roundRobinCounters[key] = (roundRobinCounters[key] + 1) mod healthy.len
    return some(healthy[idx])
  
  of lbLeastConn:
    var best = healthy[0]
    for b in healthy:
      if b.activeRequests < best.activeRequests:
        best = b
    return some(best)
  
  of lbWeighted:
    # Weighted random
    let totalWeight = healthy.foldl(a + b.weight, 0)
    if totalWeight == 0: return some(healthy[0])
    
    var r = rand(totalWeight - 1)
    for b in healthy:
      r -= b.weight
      if r < 0:
        return some(b)
    return some(healthy[^1])
  
  of lbIpHash:
    # Fall through to round-robin for demo
    return some(healthy[0])

# ============================
# Request proxy
# ============================

proc matchRoute(config: GatewayConfig, path: string): Option[Route] =
  for route in config.routes:
    let pattern = route.pattern
    if pattern.endsWith("**"):
      if path.startsWith(pattern[0..^3]):
        return some(route)
    elif pattern.endsWith("*"):
      if path.startsWith(pattern[0..^2]):
        return some(route)
    elif path == pattern:
      return some(route)
  return none(Route)

proc forwardRequest(backend: BackendServer, path, meth, body: string,
                    headers: Table[string, string]): Future[tuple[status: int, body: string, headers: Table[string, string]]] {.async.} =
  ## Mock HTTP forward - in production use asynchttpclient
  let url = fmt"http://{backend.host}:{backend.port}{path}"
  echo fmt"  [Gateway] Forwarding {meth} {url}"
  
  await sleepAsync(rand(10) + 5)  # simulate network latency
  
  # Mock response based on path
  var responseBody = ""
  var status = 200
  
  if path.contains("users"):
    responseBody = """{"users":[{"id":1,"name":"Alice"}],"total":1}"""
  elif path.contains("products"):
    responseBody = """{"products":[{"id":1,"name":"Widget","price":9.99}]}"""
  elif path.contains("health"):
    responseBody = """{"status":"ok","service":"backend"}"""
  else:
    responseBody = fmt"""{"message":"OK from {backend.id}"}"""
  
  var respHeaders: Table[string, string]
  respHeaders["X-Backend"] = backend.id
  respHeaders["Content-Type"] = "application/json"
  
  return (status, responseBody, respHeaders)

# ============================
# API versioning middleware
# ============================

type
  ApiVersion = object
    version: string         # "v1", "v2"
    deprecated: bool
    sunsetDate: string
    handler: string         # route prefix

proc detectVersion(path: string): tuple[version: string, strippedPath: string] =
  let parts = path.split('/')
  if parts.len >= 3 and parts[1] == "api":
    if parts[2].startsWith("v") and parts[2].len > 1:
      let ver = parts[2]
      let stripped = "/api/" & parts[3..^1].join("/")
      return (ver, stripped)
  return ("v1", path)  # default version

let knownVersions = {
  "v1": ApiVersion(version: "v1", deprecated: true, sunsetDate: "2025-01-01",
                   handler: "backend_v1"),
  "v2": ApiVersion(version: "v2", deprecated: false, sunsetDate: "",
                   handler: "backend_v2"),
  "v3": ApiVersion(version: "v3", deprecated: false, sunsetDate: "",
                   handler: "backend_v3"),
}.toTable()

proc versionHeaders(version: string): Table[string, string] =
  var h: Table[string, string]
  h["X-API-Version"] = version
  
  if version in knownVersions:
    let v = knownVersions[version]
    if v.deprecated:
      h["Deprecation"] = "true"
      h["Sunset"] = v.sunsetDate
      h["Link"] = fmt"""<https://api.example.com/v3>; rel="successor-version" """
  
  return h

# ============================
# Gateway handler
# ============================

let gatewayConfig = GatewayConfig(
  routes: @[
    Route(
      pattern: "/api/v*/users*",
      backends: @[
        BackendServer(id: "user-svc-1", host: "user-svc", port: 8081,
                      weight: 5, healthy: true),
        BackendServer(id: "user-svc-2", host: "user-svc-2", port: 8081,
                      weight: 5, healthy: true),
      ],
      strategy: lbRoundRobin,
      timeout: 5000,
      retries: 2,
      auth: true
    ),
    Route(
      pattern: "/api/v*/products*",
      backends: @[
        BackendServer(id: "product-svc", host: "product-svc", port: 8082,
                      weight: 10, healthy: true),
      ],
      strategy: lbLeastConn,
      timeout: 3000,
      retries: 1,
      auth: false
    ),
    Route(
      pattern: "/health",
      backends: @[
        BackendServer(id: "health-svc", host: "health-svc", port: 8099,
                      weight: 1, healthy: true),
      ],
      strategy: lbRoundRobin,
      timeout: 1000,
      retries: 0,
      auth: false
    ),
  ],
  globalHeaders: {"X-Powered-By": "NimGateway/1.0"}.toTable(),
  corsOrigins: @["https://app.example.com", "https://admin.example.com"],
  maxRequestSize: 10_000_000
)

proc handleGatewayRequest(req: Request) {.async.} =
  let path = req.url.path
  let startTime = epochTime()
  
  # Version detection
  let (version, normalizedPath) = detectVersion(path)
  
  # Add CORS headers
  var responseHeaders = newHttpHeaders()
  let origin = req.headers.getOrDefault("Origin", "")
  if origin in gatewayConfig.corsOrigins:
    responseHeaders["Access-Control-Allow-Origin"] = origin
    responseHeaders["Access-Control-Allow-Methods"] = "GET, POST, PUT, DELETE, OPTIONS"
  
  # Handle preflight
  if req.reqMethod == HttpOptions:
    await req.respond(Http204, "", responseHeaders)
    return
  
  # Add version headers
  let verHeaders = versionHeaders(version)
  for k, v in verHeaders:
    responseHeaders[k] = v
  
  # Add global headers
  for k, v in gatewayConfig.globalHeaders:
    responseHeaders[k] = v
  
  # Find matching route
  let routeOpt = matchRoute(gatewayConfig, path)
  if routeOpt.isNone:
    responseHeaders["Content-Type"] = "application/json"
    await req.respond(Http404,
      """{"error":"No route found","path":"""" & path & """"}""",
      responseHeaders)
    return
  
  let route = routeOpt.get()
  
  # Select backend
  let backendOpt = selectBackend(route)
  if backendOpt.isNone:
    responseHeaders["Content-Type"] = "application/json"
    await req.respond(Http503,
      """{"error":"No healthy backends","route":"""" & route.pattern & """"}""",
      responseHeaders)
    return
  
  let backend = backendOpt.get()
  
  # Forward request
  var fwdHeaders = initTable[string, string]()
  fwdHeaders["X-Real-IP"] = req.hostname
  fwdHeaders["X-Request-ID"] = fmt"req_{int(epochTime() * 1000) mod 1_000_000}"
  
  let (status, body, backendHeaders) = await forwardRequest(
    backend, normalizedPath, $req.reqMethod, req.body, fwdHeaders
  )
  
  # Copy backend headers
  for k, v in backendHeaders:
    responseHeaders[k] = v
  
  let elapsed = int((epochTime() - startTime) * 1000)
  responseHeaders["X-Response-Time"] = fmt"{elapsed}ms"
  
  await req.respond(HttpCode(status), body, responseHeaders)

# ============================
# Demo
# ============================

proc demo() =
  echo "=== API Gateway Demo ==="
  
  echo "\nGateway routes:"
  for route in gatewayConfig.routes:
    echo fmt"  {route.pattern} -> {route.backends.mapIt(it.id).join(\", \")} ({route.strategy})"
  
  echo "\nLoad balancer tests:"
  for route in gatewayConfig.routes[0..1]:
    for _ in 0..2:
      let backend = selectBackend(route)
      if backend.isSome:
        echo fmt"  {route.pattern} -> {backend.get().id}"
  
  echo "\nAPI version detection:"
  for path in ["/api/v1/users", "/api/v2/products/123", "/api/v3/orders", "/health"]:
    let (ver, stripped) = detectVersion(path)
    echo fmt"  {path} -> version={ver} stripped={stripped}"
  
  echo "\nVersion headers:"
  let h1 = versionHeaders("v1")
  let h2 = versionHeaders("v2")
  echo fmt"  v1: deprecated={h1.getOrDefault(\"Deprecation\", \"false\")}"
  echo fmt"  v2: deprecated={h2.getOrDefault(\"Deprecation\", \"false\")}"

demo()
```

---

## Step 692-705: Canary Deployments

```nim
import tables, strformat, times, sequtils, math, strutils

# ============================
# Canary deployment
# ============================

type
  DeploymentVersion = object
    id: string
    image: string
    weight: float       # 0.0 - 1.0
    minInstances: int
    maxInstances: int
    instances: int
    errors: int
    requests: int
    latencyP50: float
    latencyP99: float
    isCanary: bool
    deployedAt: float

  CanaryConfig = object
    stable: DeploymentVersion
    canary: Option[DeploymentVersion]
    errorThreshold: float     # percentage to trigger rollback
    latencyThreshold: float   # ms P99 to trigger rollback
    autoRollback: bool
    stepsPercent: seq[float]  # promotion steps: [1, 5, 20, 50, 100]
    currentStep: int

var canaryConfigs: Table[string, CanaryConfig] = initTable[string, CanaryConfig]()

proc startCanary(serviceId: string, newImage: string,
                 initialWeight = 0.01): CanaryConfig =
  let stable = DeploymentVersion(
    id: "stable-v1",
    image: "myapp:v1.2.3",
    weight: 1.0 - initialWeight,
    instances: 10,
    isCanary: false,
    deployedAt: epochTime() - 86400.0
  )
  
  let canary = DeploymentVersion(
    id: "canary-v2",
    image: newImage,
    weight: initialWeight,
    instances: 1,
    isCanary: true,
    deployedAt: epochTime()
  )
  
  let config = CanaryConfig(
    stable: stable,
    canary: some(canary),
    errorThreshold: 5.0,
    latencyThreshold: 500.0,
    autoRollback: true,
    stepsPercent: @[1.0, 5.0, 20.0, 50.0, 100.0],
    currentStep: 0
  )
  
  canaryConfigs[serviceId] = config
  echo fmt"[Canary] Started for {serviceId}: {newImage} at {initialWeight * 100:.1f}%"
  return config

proc routeRequest(serviceId, userId: string): string =
  ## Returns which version to route request to
  if serviceId notin canaryConfigs:
    return "stable"
  
  let config = canaryConfigs[serviceId]
  if config.canary.isNone:
    return "stable"
  
  let canary = config.canary.get()
  
  # Use consistent hashing: same user always gets same version
  var hash = 0u32
  for c in userId:
    hash = hash * 31 + uint32(ord(c))
  let bucket = float(hash mod 100)
  
  if bucket < canary.weight * 100.0:
    return "canary"
  return "stable"

proc promoteCanary(serviceId: string) =
  if serviceId notin canaryConfigs: return
  var config = canaryConfigs[serviceId]
  
  if config.canary.isNone: return
  if config.currentStep >= config.stepsPercent.len - 1: return
  
  inc config.currentStep
  let newWeight = config.stepsPercent[config.currentStep] / 100.0
  
  var canary = config.canary.get()
  canary.weight = newWeight
  config.canary = some(canary)
  config.stable.weight = 1.0 - newWeight
  
  canaryConfigs[serviceId] = config
  echo fmt"[Canary] Promoted {serviceId} to {newWeight * 100:.1f}%"

proc rollbackCanary(serviceId: string, reason: string) =
  if serviceId notin canaryConfigs: return
  var config = canaryConfigs[serviceId]
  
  config.stable.weight = 1.0
  config.canary = none(DeploymentVersion)
  config.currentStep = 0
  
  canaryConfigs[serviceId] = config
  echo fmt"[Canary] Rolled back {serviceId}: {reason}"

proc fullyPromote(serviceId: string) =
  if serviceId notin canaryConfigs: return
  var config = canaryConfigs[serviceId]
  
  if config.canary.isNone: return
  
  let canaryVersion = config.canary.get()
  config.stable = canaryVersion
  config.stable.weight = 1.0
  config.stable.isCanary = false
  config.canary = none(DeploymentVersion)
  config.currentStep = 0
  
  canaryConfigs[serviceId] = config
  echo fmt"[Canary] Fully promoted {serviceId}: {canaryVersion.image}"

proc checkCanaryHealth(serviceId: string): bool =
  if serviceId notin canaryConfigs: return true
  let config = canaryConfigs[serviceId]
  
  if config.canary.isNone: return true
  let canary = config.canary.get()
  
  if canary.requests == 0: return true
  
  let errorRate = float(canary.errors) / float(canary.requests) * 100.0
  
  if errorRate > config.errorThreshold:
    echo fmt"[Canary] High error rate: {errorRate:.1f}% > {config.errorThreshold:.1f}%"
    return false
  
  if canary.latencyP99 > config.latencyThreshold:
    echo fmt"[Canary] High latency: {canary.latencyP99:.0f}ms > {config.latencyThreshold:.0f}ms"
    return false
  
  return true

proc recordCanaryMetrics(serviceId: string, errors, requests: int, p99: float) =
  if serviceId notin canaryConfigs: return
  var config = canaryConfigs[serviceId]
  if config.canary.isNone: return
  
  var canary = config.canary.get()
  canary.errors += errors
  canary.requests += requests
  canary.latencyP99 = p99
  config.canary = some(canary)
  canaryConfigs[serviceId] = config

# ============================
# Demo
# ============================

proc demo() =
  echo "=== Canary Deployment Demo ==="
  
  let serviceId = "user-service"
  
  # Start canary
  discard startCanary(serviceId, "myapp:v2.0.0")
  
  # Show routing
  echo "\nRequest routing (before promotion):"
  var stableCount = 0
  var canaryCount = 0
  for i in 0..<100:
    let target = routeRequest(serviceId, fmt"user_{i}")
    if target == "canary": inc canaryCount
    else: inc stableCount
  
  echo fmt"  Stable: {stableCount}/100, Canary: {canaryCount}/100"
  
  # Record good metrics
  recordCanaryMetrics(serviceId, 0, 1000, 45.0)
  
  # Promote
  promoteCanary(serviceId)
  
  # Recheck routing
  stableCount = 0; canaryCount = 0
  for i in 0..<100:
    let target = routeRequest(serviceId, fmt"user_{i}")
    if target == "canary": inc canaryCount
    else: inc stableCount
  
  echo fmt"\nAfter promotion to 5%:"
  echo fmt"  Stable: {stableCount}/100, Canary: {canaryCount}/100"
  
  # Simulate bad metrics -> rollback
  recordCanaryMetrics(serviceId, 60, 1000, 800.0)
  
  if not checkCanaryHealth(serviceId):
    rollbackCanary(serviceId, "High error rate detected")
  
  echo fmt"\nAfter rollback:"
  let config = canaryConfigs[serviceId]
  echo fmt"  Canary active: {config.canary.isSome}"
  echo fmt"  Stable weight: {config.stable.weight * 100:.0f}%"

demo()
```

---

## 📝 สรุป Part 48

| Steps | หัวข้อ |
|-------|--------|
| 691 | API gateway: routing, load balancing, versioning |
| 692-705 | Canary deployments, auto rollback, health checks |

---

**← [Part 47: Feature Flags](part_47_feature_flags.md) | [Part 49: Service Mesh →](part_49_service_mesh.md)**
