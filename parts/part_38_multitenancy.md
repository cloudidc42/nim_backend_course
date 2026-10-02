# Part 38: Multi-Tenancy Patterns
## Steps 541-555: Multi-Tenant ระดับ Enterprise

---

## 🎯 เป้าหมายของ Part นี้

- Tenant isolation strategies
- Row-level security (RLS)
- Schema-per-tenant
- Tenant middleware
- Cross-tenant queries
- Tenant provisioning

---

## Step 541: Multi-Tenancy Strategies

```nim
import tables, strformat, options, json, strutils

# ============================
# Strategy 1: Shared schema (Row-Level)
# ============================
# All tenants in same table, tenant_id column everywhere

type
  TenantId = distinct string

  User = object
    id: int
    tenantId: TenantId
    email: string
    name: string

  TenantContext = object
    tenantId: TenantId
    tenantName: string
    plan: string
    features: seq[string]

var currentTenant {.threadvar.}: Option[TenantContext]

proc withTenant[T](ctx: TenantContext, body: proc(): T): T =
  currentTenant = some(ctx)
  defer: currentTenant = none(TenantContext)
  body()

proc requireTenant(): TenantContext =
  if currentTenant.isNone:
    raise newException(ValueError, "No tenant context set")
  currentTenant.get()

# All queries are automatically scoped to tenant
proc findUsers(db: auto, email = ""): seq[User] =
  let tenant = requireTenant()
  var sql = fmt"SELECT * FROM users WHERE tenant_id = '{string(tenant.tenantId)}'"
  if email.len > 0:
    sql &= fmt" AND email = '{email}'"
  echo fmt"[SQL] {sql}"
  return @[]  # mock

# ============================
# Strategy 2: Schema per tenant
# ============================
# Each tenant has own PostgreSQL schema: tenant_abc.users, tenant_xyz.users

type
  SchemaManager = object
    schemas: Table[TenantId, string]

proc tenantSchema(tenantId: TenantId): string =
  "tenant_" & string(tenantId)

proc setSearchPath(conn: auto, tenantId: TenantId) =
  let schema = tenantSchema(tenantId)
  echo fmt"[SQL] SET search_path = '{schema}', public"

proc findUsersBySchema(conn: auto, email: string, tenantId: TenantId): seq[User] =
  setSearchPath(conn, tenantId)
  echo fmt"[SQL] SELECT * FROM users WHERE email = '{email}'"
  return @[]

# ============================
# Strategy 3: Database per tenant
# ============================
# Each tenant has own database (strongest isolation, highest cost)

type
  TenantDbPool = object
    connections: Table[TenantId, string]  # tenantId -> connectionString

var dbPool = TenantDbPool(connections: initTable[TenantId, string]())

proc getDb(tenantId: TenantId): string =
  if tenantId notin dbPool.connections:
    let connStr = fmt"postgres://localhost/tenant_{string(tenantId)}"
    dbPool.connections[tenantId] = connStr
    echo fmt"[Pool] New connection pool for tenant: {string(tenantId)}"
  return dbPool.connections[tenantId]

# ============================
# Comparison
# ============================

proc printStrategyComparison() =
  echo "\nMulti-Tenancy Strategy Comparison:"
  echo "=".repeat(70)
  echo fmt"{'Strategy':^25} {'Isolation':^12} {'Cost':^8} {'Best for':^20}"
  echo "=".repeat(70)
  echo fmt"  {'Shared schema':^23} {'Low':^12} {'$':^8} {'Many small tenants':^20}"
  echo fmt"  {'Schema per tenant':^23} {'Medium':^12} {'$$':^8} {'Mid-size tenants':^20}"
  echo fmt"  {'DB per tenant':^23} {'High':^12} {'$$$':^8} {'Enterprise tenants':^20}"
  echo "=".repeat(70)

# ============================
# Demo
# ============================

proc demo() =
  echo "=== Multi-Tenancy Demo ==="
  printStrategyComparison()
  
  let tenant1 = TenantContext(
    tenantId: TenantId("acme"),
    tenantName: "ACME Corp",
    plan: "enterprise",
    features: @["api", "analytics", "sso"]
  )
  
  let tenant2 = TenantContext(
    tenantId: TenantId("startup42"),
    tenantName: "Startup 42",
    plan: "starter",
    features: @["api"]
  )
  
  echo "\n--- Row-Level Tenancy ---"
  withTenant(tenant1, proc(): void =
    let ctx = requireTenant()
    echo fmt"Tenant: {ctx.tenantName} ({ctx.plan})"
    discard findUsers(nil, "admin@acme.com")
  )
  
  echo "\n--- Schema-per-tenant ---"
  findUsersBySchema(nil, "admin@startup.com", tenant2.tenantId)
  
  echo "\n--- DB-per-tenant ---"
  let db1 = getDb(tenant1.tenantId)
  let db2 = getDb(tenant2.tenantId)
  echo fmt"Tenant 1 DB: {db1}"
  echo fmt"Tenant 2 DB: {db2}"

demo()
```

---

## Step 542: Tenant Middleware

```nim
import asyncdispatch, asynchttpserver, strutils, strformat, tables, options, json

# ============================
# Tenant resolution strategies
# ============================

type
  TenantResolution = enum
    trSubdomain   # acme.myapp.com
    trHeader      # X-Tenant-ID: acme
    trJwt         # JWT claim: tenantId
    trPath        # /api/tenants/acme/users

  TenantInfo = object
    id: string
    name: string
    plan: string
    isActive: bool
    maxUsers: int
    features: seq[string]

  TenantCache = object
    cache: Table[string, TenantInfo]

var tenantCache = TenantCache(cache: initTable[string, TenantInfo]())

# Mock tenant database
let mockTenants = {
  "acme": TenantInfo(id: "acme", name: "ACME Corp", plan: "enterprise",
                     isActive: true, maxUsers: 1000,
                     features: @["api", "analytics", "sso", "audit_log"]),
  "startup42": TenantInfo(id: "startup42", name: "Startup 42", plan: "starter",
                          isActive: true, maxUsers: 10,
                          features: @["api"]),
  "disabled_co": TenantInfo(id: "disabled_co", name: "Disabled Co",
                             isActive: false, maxUsers: 0, plan: "none",
                             features: @[]),
}.toTable()

proc lookupTenant(tenantId: string): Option[TenantInfo] =
  # Check cache first
  if tenantId in tenantCache.cache:
    return some(tenantCache.cache[tenantId])
  
  # Look up from DB
  if tenantId in mockTenants:
    let info = mockTenants[tenantId]
    tenantCache.cache[tenantId] = info
    return some(info)
  
  return none(TenantInfo)

proc extractTenantId(req: Request, strategy: TenantResolution): string =
  case strategy
  of trHeader:
    return req.headers.getOrDefault("X-Tenant-ID", "")
  
  of trSubdomain:
    # Extract from Host: acme.myapp.com -> acme
    let host = req.headers.getOrDefault("Host", "")
    let parts = host.split('.')
    if parts.len >= 3:
      return parts[0]
    return ""
  
  of trPath:
    # /api/tenants/acme/users -> acme
    let path = req.url.path
    let parts = path.split('/')
    let tenantIdx = parts.find("tenants")
    if tenantIdx >= 0 and tenantIdx + 1 < parts.len:
      return parts[tenantIdx + 1]
    return ""
  
  of trJwt:
    # Would decode JWT and extract tenantId claim
    let auth = req.headers.getOrDefault("Authorization", "")
    if auth.startsWith("Bearer "):
      echo "[TenantMW] Would decode JWT claim tenantId"
      return "from_jwt"  # mock
    return ""

# ============================
# Tenant middleware
# ============================

type
  TenantMiddleware = object
    strategy: TenantResolution
    required: bool

proc tenantMiddleware(req: Request, mw: TenantMiddleware): Option[TenantInfo] =
  let tenantId = extractTenantId(req, mw.strategy)
  
  if tenantId.len == 0:
    if mw.required:
      echo "[TenantMW] ERROR: No tenant ID found"
    return none(TenantInfo)
  
  let tenant = lookupTenant(tenantId)
  
  if tenant.isNone:
    echo fmt"[TenantMW] ERROR: Tenant not found: {tenantId}"
    return none(TenantInfo)
  
  if not tenant.get().isActive:
    echo fmt"[TenantMW] ERROR: Tenant disabled: {tenantId}"
    return none(TenantInfo)
  
  echo fmt"[TenantMW] Resolved tenant: {tenantId} (plan: {tenant.get().plan})"
  return tenant

# ============================
# Feature flags per tenant
# ============================

proc hasFeature(tenant: TenantInfo, feature: string): bool =
  feature in tenant.features

proc requireFeature(tenant: TenantInfo, feature: string) =
  if not tenant.hasFeature(feature):
    raise newException(ValueError,
      fmt"Feature '{feature}' not available on plan '{tenant.plan}'")

# ============================
# Plan limits
# ============================

type
  PlanLimits = object
    maxUsers: int
    maxApiCalls: int
    maxStorage: int  # MB
    maxWebhooks: int

proc getPlanLimits(plan: string): PlanLimits =
  case plan
  of "starter":
    PlanLimits(maxUsers: 10, maxApiCalls: 10_000, maxStorage: 1_000, maxWebhooks: 1)
  of "growth":
    PlanLimits(maxUsers: 100, maxApiCalls: 100_000, maxStorage: 10_000, maxWebhooks: 10)
  of "enterprise":
    PlanLimits(maxUsers: 10_000, maxApiCalls: 10_000_000, maxStorage: 1_000_000, maxWebhooks: 100)
  else:
    PlanLimits()

# ============================
# Demo
# ============================

proc demo() =
  echo "=== Tenant Middleware Demo ==="
  
  let mw = TenantMiddleware(strategy: trHeader, required: true)
  
  # Simulate requests
  echo "\n--- Valid tenant ---"
  let tenant = lookupTenant("acme")
  if tenant.isSome:
    let t = tenant.get()
    echo fmt"Tenant: {t.name}"
    echo fmt"Plan: {t.plan}"
    echo fmt"Features: {t.features.join(\", \")}"
    
    let limits = getPlanLimits(t.plan)
    echo fmt"Max users: {limits.maxUsers}"
    echo fmt"Max API calls: {limits.maxApiCalls}"
    
    echo "\nFeature checks:"
    echo fmt"  analytics: {t.hasFeature(\"analytics\")}"
    echo fmt"  sso: {t.hasFeature(\"sso\")}"
    echo fmt"  custom_domain: {t.hasFeature(\"custom_domain\")}"
  
  echo "\n--- Invalid tenant ---"
  let unknown = lookupTenant("unknown_corp")
  echo fmt"Found: {unknown.isSome}"
  
  echo "\n--- Disabled tenant ---"
  let disabled = lookupTenant("disabled_co")
  if disabled.isSome:
    echo fmt"Active: {disabled.get().isActive}"

demo()
```

---

## Step 543-555: Complete Multi-Tenant Application

```nim
# multitenant_app.nim - Complete multi-tenant SaaS backend

import tables, strformat, times, json, options, sequtils, strutils

# ============================
# Domain types
# ============================

type
  TenantPlan = enum
    tpFree, tpStarter, tpGrowth, tpEnterprise

  Tenant = object
    id: string
    slug: string
    name: string
    plan: TenantPlan
    isActive: bool
    createdAt: float
    settings: Table[string, string]

  TenantUser = object
    id: int
    tenantId: string
    email: string
    role: string
    isActive: bool

  ApiKey = object
    id: string
    tenantId: string
    name: string
    keyHash: string
    scopes: seq[string]
    expiresAt: Option[float]
    lastUsedAt: float
    callCount: int

  TenantStore = object
    tenants: Table[string, Tenant]
    users: Table[int, TenantUser]
    apiKeys: Table[string, ApiKey]
    userCounter: int
    keyCounter: int

# ============================
# Tenant provisioning
# ============================

var store = TenantStore(
  tenants: initTable[string, Tenant](),
  users: initTable[int, TenantUser](),
  apiKeys: initTable[string, ApiKey](),
  userCounter: 0,
  keyCounter: 0
)

proc generateSlug(name: string): string =
  name.toLower().replace(" ", "-").replace("_", "-")
    .filterIt(it.isAlphaNumeric() or it == '-')
    .join()

proc generateApiKeyId(): string =
  fmt"key_{int(epochTime() * 1000) mod 100000}"

proc createTenant(name, adminEmail: string, plan = tpFree): Tenant =
  let slug = generateSlug(name)
  
  if slug in store.tenants:
    raise newException(ValueError, fmt"Tenant slug already exists: {slug}")
  
  let tenant = Tenant(
    id: fmt"t_{slug}",
    slug: slug,
    name: name,
    plan: plan,
    isActive: true,
    createdAt: epochTime(),
    settings: initTable[string, string]()
  )
  
  store.tenants[tenant.id] = tenant
  
  echo fmt"[Tenant] Created: {name} ({plan})"
  
  # Create admin user
  inc store.userCounter
  let adminUser = TenantUser(
    id: store.userCounter,
    tenantId: tenant.id,
    email: adminEmail,
    role: "admin",
    isActive: true
  )
  store.users[adminUser.id] = adminUser
  echo fmt"[Tenant] Admin user created: {adminEmail}"
  
  # Provision resources
  echo fmt"[Tenant] Provisioning schema: tenant_{slug}"
  echo fmt"[Tenant] Setting up defaults..."
  
  return tenant

proc getTenantBySlug(slug: string): Option[Tenant] =
  for _, t in store.tenants:
    if t.slug == slug:
      return some(t)
  return none(Tenant)

# ============================
# API key management
# ============================

proc createApiKey(tenantId, name: string, scopes: seq[string],
                  ttlDays: int = 0): ApiKey =
  let keyId = generateApiKeyId()
  let expiresAt = if ttlDays > 0:
    some(epochTime() + float(ttlDays * 86400))
  else:
    none(float)
  
  let apiKey = ApiKey(
    id: keyId,
    tenantId: tenantId,
    name: name,
    keyHash: "hash_" & keyId,  # in production: bcrypt hash
    scopes: scopes,
    expiresAt: expiresAt,
    lastUsedAt: 0.0,
    callCount: 0
  )
  
  store.apiKeys[keyId] = apiKey
  return apiKey

proc validateApiKey(keyId: string): Option[ApiKey] =
  if keyId notin store.apiKeys:
    return none(ApiKey)
  
  var key = store.apiKeys[keyId]
  
  if key.expiresAt.isSome and epochTime() > key.expiresAt.get():
    return none(ApiKey)  # expired
  
  # Update last used
  key.lastUsedAt = epochTime()
  inc key.callCount
  store.apiKeys[keyId] = key
  
  return some(key)

proc hasScope(key: ApiKey, scope: string): bool =
  "*" in key.scopes or scope in key.scopes

# ============================
# Tenant usage tracking
# ============================

type
  UsageMetrics = object
    tenantId: string
    period: string
    apiCalls: int
    users: int
    storageBytes: int
    webhookCalls: int

var usageStore: Table[string, UsageMetrics] = initTable[string, UsageMetrics]()

proc getCurrentPeriod(): string =
  let now = now()
  fmt"{now.year}-{int(now.month):02d}"

proc trackApiCall(tenantId: string) =
  let key = fmt"{tenantId}:{getCurrentPeriod()}"
  if key notin usageStore:
    usageStore[key] = UsageMetrics(tenantId: tenantId, period: getCurrentPeriod())
  usageStore[key].apiCalls += 1

proc getUsage(tenantId, period: string): UsageMetrics =
  let key = fmt"{tenantId}:{period}"
  if key in usageStore:
    return usageStore[key]
  return UsageMetrics(tenantId: tenantId, period: period)

proc checkUsageLimits(tenantId: string, plan: TenantPlan): bool =
  let period = getCurrentPeriod()
  let usage = getUsage(tenantId, period)
  
  let maxCalls = case plan
    of tpFree: 1_000
    of tpStarter: 50_000
    of tpGrowth: 500_000
    of tpEnterprise: high(int)
  
  if usage.apiCalls >= maxCalls:
    echo fmt"[Usage] Limit exceeded for tenant {tenantId}: {usage.apiCalls}/{maxCalls}"
    return false
  return true

# ============================
# Tenant settings
# ============================

proc getSetting(tenant: Tenant, key, default_: string = ""): string =
  if key in tenant.settings:
    return tenant.settings[key]
  return default_

proc setSetting(tenantId, key, value: string) =
  if tenantId in store.tenants:
    var t = store.tenants[tenantId]
    t.settings[key] = value
    store.tenants[tenantId] = t

# ============================
# Cross-tenant admin operations
# ============================

proc listAllTenants(activeOnly = true): seq[Tenant] =
  var result: seq[Tenant]
  for _, t in store.tenants:
    if not activeOnly or t.isActive:
      result.add(t)
  return result

proc suspendTenant(tenantId: string, reason: string) =
  if tenantId in store.tenants:
    var t = store.tenants[tenantId]
    t.isActive = false
    store.tenants[tenantId] = t
    echo fmt"[Admin] Tenant suspended: {tenantId} - {reason}"

proc upgradePlan(tenantId: string, newPlan: TenantPlan) =
  if tenantId in store.tenants:
    var t = store.tenants[tenantId]
    let oldPlan = t.plan
    t.plan = newPlan
    store.tenants[tenantId] = t
    echo fmt"[Admin] Plan upgraded: {tenantId} {oldPlan} -> {newPlan}"

# ============================
# Demo
# ============================

proc demo() =
  echo "=== Multi-Tenant SaaS Backend Demo ==="
  
  echo "\n1. Creating tenants..."
  let acme = createTenant("ACME Corp", "admin@acme.com", tpEnterprise)
  let startup = createTenant("Startup 42", "founder@startup42.io", tpStarter)
  let freeUser = createTenant("Free User", "user@example.com", tpFree)
  
  echo fmt"\n2. Total tenants: {store.tenants.len}"
  
  echo "\n3. API key management..."
  let key1 = createApiKey(acme.id, "Production Key",
    @["read:users", "write:users", "read:data"], ttlDays: 365)
  let key2 = createApiKey(acme.id, "Analytics Key",
    @["read:analytics"], ttlDays: 30)
  
  echo fmt"Created key: {key1.id} (scopes: {key1.scopes.join(\", \")})"
  
  let validated = validateApiKey(key1.id)
  echo fmt"Key valid: {validated.isSome}"
  if validated.isSome:
    echo fmt"Can write users: {validated.get().hasScope(\"write:users\")}"
    echo fmt"Can delete: {validated.get().hasScope(\"delete:*\")}"
  
  echo "\n4. Usage tracking..."
  for i in 0..<5:
    trackApiCall(acme.id)
  trackApiCall(startup.id)
  
  let acmeUsage = getUsage(acme.id, getCurrentPeriod())
  echo fmt"ACME API calls: {acmeUsage.apiCalls}"
  echo fmt"Free tier OK: {checkUsageLimits(freeUser.id, tpFree)}"
  
  echo "\n5. Tenant settings..."
  setSetting(acme.id, "theme", "dark")
  setSetting(acme.id, "timezone", "Asia/Bangkok")
  
  let freshAcme = store.tenants[acme.id]
  echo fmt"Theme: {getSetting(freshAcme, \"theme\", \"light\")}"
  echo fmt"Timezone: {getSetting(freshAcme, \"timezone\", \"UTC\")}"
  
  echo "\n6. Admin operations..."
  upgradePlan(startup.id, tpGrowth)
  
  let allTenants = listAllTenants()
  echo fmt"\nAll active tenants: {allTenants.len}"
  for t in allTenants:
    echo fmt"  - {t.name} ({t.plan}) [{t.id}]"

demo()
```

---

## 📝 สรุป Part 38

| Steps | หัวข้อ |
|-------|--------|
| 541 | Multi-tenancy strategies (shared, schema, DB) |
| 542 | Tenant middleware, feature flags, plan limits |
| 543-555 | Full SaaS backend: provisioning, API keys, usage |

---

**← [Part 37: Database Migrations](part_37_database_migrations.md) | [Part 39: Message Queues →](part_39_message_queues.md)**
