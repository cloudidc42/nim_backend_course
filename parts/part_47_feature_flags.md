# Part 47: Feature Flags
## Steps 676-690: Feature Flag System ระดับ Production

---

## 🎯 เป้าหมายของ Part นี้

- Feature flag types (boolean, percentage, targeting)
- A/B testing framework
- Gradual rollout
- Kill switch
- Analytics integration
- Remote config

---

## Step 676: Feature Flag Core

```nim
import tables, strformat, times, json, strutils, sequtils, hashes, math

# ============================
# Feature flag types
# ============================

type
  FlagType = enum
    ftBoolean, ftPercentage, ftTargeting, ftVariant, ftScheduled

  TargetRule = object
    attribute: string     # "country", "plan", "userId"
    operator: string      # "in", "not_in", "equals", "starts_with"
    values: seq[string]

  Variant = object
    name: string
    weight: float   # 0.0 - 1.0, sum of all variants = 1.0
    value: JsonNode

  Schedule = object
    startAt: float
    endAt: float

  FeatureFlag = object
    id: string
    name: string
    type_: FlagType
    enabled: bool
    defaultValue: JsonNode
    percentage: float           # for ftPercentage: 0-100
    targetRules: seq[TargetRule]
    variants: seq[Variant]      # for A/B testing
    schedule: Option[Schedule]
    tags: seq[string]
    createdAt: float
    updatedAt: float
    description: string

  EvaluationContext = object
    userId: string
    country: string
    plan: string
    email: string
    groups: seq[string]
    customAttrs: Table[string, string]

  EvaluationResult = object
    flagId: string
    enabled: bool
    value: JsonNode
    variant: string
    reason: string     # "default", "targeting", "percentage", "disabled"

# ============================
# Flag storage
# ============================

var flags: Table[string, FeatureFlag] = initTable[string, FeatureFlag]()

proc registerFlag(flag: FeatureFlag) =
  flags[flag.id] = flag
  echo fmt"[Flags] Registered: {flag.id} ({flag.type_})"

proc updateFlag(id: string, enabled: bool) =
  if id in flags:
    flags[id].enabled = enabled
    flags[id].updatedAt = epochTime()
    echo fmt"[Flags] Updated: {id} -> enabled={enabled}"

# ============================
# Evaluation engine
# ============================

proc matchesRule(rule: TargetRule, ctx: EvaluationContext): bool =
  let value = case rule.attribute
    of "country": ctx.country
    of "plan": ctx.plan
    of "email": ctx.email
    of "userId": ctx.userId
    else: ctx.customAttrs.getOrDefault(rule.attribute, "")
  
  case rule.operator
  of "in":
    return value in rule.values
  of "not_in":
    return value notin rule.values
  of "equals":
    return value == rule.values.getOrDefault(0, "")
  of "starts_with":
    return value.startsWith(rule.values.getOrDefault(0, ""))
  of "ends_with":
    return value.endsWith(rule.values.getOrDefault(0, ""))
  else:
    return false

proc userBucket(userId, flagId: string): float =
  ## Deterministic bucket assignment (0-100) based on userId + flagId
  let combined = userId & ":" & flagId
  var h = 0u32
  for c in combined:
    h = h * 31 + uint32(ord(c))
  return float(h mod 100)

proc selectVariant(variants: seq[Variant], bucket: float): Variant =
  var cumulative = 0.0
  for v in variants:
    cumulative += v.weight * 100.0
    if bucket < cumulative:
      return v
  return variants[^1]

proc evaluate(flagId: string, ctx: EvaluationContext): EvaluationResult =
  if flagId notin flags:
    return EvaluationResult(
      flagId: flagId, enabled: false,
      value: newJBool(false), variant: "", reason: "not_found"
    )
  
  let flag = flags[flagId]
  
  # Check if globally disabled
  if not flag.enabled:
    return EvaluationResult(
      flagId: flagId, enabled: false,
      value: flag.defaultValue, variant: "", reason: "disabled"
    )
  
  # Check schedule
  if flag.schedule.isSome:
    let sched = flag.schedule.get()
    let now = epochTime()
    if now < sched.startAt or now > sched.endAt:
      return EvaluationResult(
        flagId: flagId, enabled: false,
        value: flag.defaultValue, variant: "", reason: "schedule"
      )
  
  case flag.type_
  of ftBoolean:
    return EvaluationResult(
      flagId: flagId, enabled: true,
      value: newJBool(true), variant: "", reason: "enabled"
    )
  
  of ftPercentage:
    let bucket = userBucket(ctx.userId, flagId)
    let inGroup = bucket < flag.percentage
    return EvaluationResult(
      flagId: flagId, enabled: inGroup,
      value: newJBool(inGroup), variant: "",
      reason: if inGroup: "percentage_in" else: "percentage_out"
    )
  
  of ftTargeting:
    for rule in flag.targetRules:
      if matchesRule(rule, ctx):
        return EvaluationResult(
          flagId: flagId, enabled: true,
          value: newJBool(true), variant: "", reason: "targeting"
        )
    return EvaluationResult(
      flagId: flagId, enabled: false,
      value: flag.defaultValue, variant: "", reason: "no_match"
    )
  
  of ftVariant:
    let bucket = userBucket(ctx.userId, flagId)
    let variant = selectVariant(flag.variants, bucket)
    return EvaluationResult(
      flagId: flagId, enabled: true,
      value: variant.value, variant: variant.name, reason: "variant"
    )
  
  of ftScheduled:
    return EvaluationResult(
      flagId: flagId, enabled: true,
      value: newJBool(true), variant: "", reason: "scheduled"
    )

# ============================
# Helpers
# ============================

proc isEnabled(flagId: string, ctx: EvaluationContext): bool =
  evaluate(flagId, ctx).enabled

proc getVariant(flagId: string, ctx: EvaluationContext): string =
  evaluate(flagId, ctx).variant

proc getValue(flagId: string, ctx: EvaluationContext): JsonNode =
  evaluate(flagId, ctx).value

# ============================
# Demo
# ============================

proc demo() =
  echo "=== Feature Flags Demo ==="
  
  # Register flags
  registerFlag(FeatureFlag(
    id: "new_checkout", name: "New Checkout UI",
    type_: ftBoolean, enabled: true,
    defaultValue: newJBool(false)
  ))
  
  registerFlag(FeatureFlag(
    id: "dark_mode", name: "Dark Mode",
    type_: ftPercentage, enabled: true,
    percentage: 50.0,
    defaultValue: newJBool(false)
  ))
  
  registerFlag(FeatureFlag(
    id: "beta_features", name: "Beta Features",
    type_: ftTargeting, enabled: true,
    targetRules: @[
      TargetRule(attribute: "plan", operator: "in", values: @["enterprise", "pro"]),
    ],
    defaultValue: newJBool(false)
  ))
  
  registerFlag(FeatureFlag(
    id: "homepage_hero", name: "Homepage Hero A/B",
    type_: ftVariant, enabled: true,
    variants: @[
      Variant(name: "control", weight: 0.5, value: %"blue_hero"),
      Variant(name: "variant_a", weight: 0.3, value: %"green_hero"),
      Variant(name: "variant_b", weight: 0.2, value: %"purple_hero"),
    ],
    defaultValue: %"blue_hero"
  ))
  
  # Evaluate for different users
  echo "\n--- Evaluations ---"
  let users = [
    EvaluationContext(userId: "user_1", country: "US", plan: "free"),
    EvaluationContext(userId: "user_2", country: "TH", plan: "pro"),
    EvaluationContext(userId: "user_3", country: "UK", plan: "enterprise"),
  ]
  
  for ctx in users:
    echo fmt"\nUser {ctx.userId} (plan: {ctx.plan}):"
    echo fmt"  new_checkout: {isEnabled(\"new_checkout\", ctx)}"
    echo fmt"  dark_mode: {isEnabled(\"dark_mode\", ctx)}"
    echo fmt"  beta_features: {isEnabled(\"beta_features\", ctx)}"
    echo fmt"  homepage_hero variant: {getVariant(\"homepage_hero\", ctx)}"
  
  # Kill switch
  echo "\n--- Kill switch ---"
  updateFlag("new_checkout", false)
  let result = evaluate("new_checkout", users[0])
  echo fmt"new_checkout after disable: {result.enabled} (reason: {result.reason})"

demo()
```

---

## Step 677-690: Remote Config & Gradual Rollout

```nim
import tables, strformat, times, json, strutils, sequtils, math

# ============================
# Remote config system
# ============================

type
  ConfigValue = object
    key: string
    value: JsonNode
    type_: string     # "string" | "int" | "float" | "bool" | "json"
    description: string
    environment: string   # "production" | "staging" | "development"
    updatedAt: float
    updatedBy: string

  RemoteConfig = object
    configs: Table[string, Table[string, ConfigValue]]
    environment: string

var remoteConfig = RemoteConfig(
  configs: initTable[string, Table[string, ConfigValue]](),
  environment: "production"
)

proc setConfig(env, key: string, value: JsonNode, description = "",
               updatedBy = "system") =
  if env notin remoteConfig.configs:
    remoteConfig.configs[env] = initTable[string, ConfigValue]()
  
  remoteConfig.configs[env][key] = ConfigValue(
    key: key,
    value: value,
    description: description,
    environment: env,
    updatedAt: epochTime(),
    updatedBy: updatedBy
  )
  echo fmt"[Config] Set {env}/{key} = {value}"

proc getConfig(key: string, default_: JsonNode = nil): JsonNode =
  let env = remoteConfig.environment
  if env in remoteConfig.configs and key in remoteConfig.configs[env]:
    return remoteConfig.configs[env][key].value
  # Fallback to "default" environment
  if "default" in remoteConfig.configs and key in remoteConfig.configs["default"]:
    return remoteConfig.configs["default"][key].value
  return if default_.isNil: newJNull() else: default_

proc getConfigStr(key, default_: string = ""): string =
  let v = getConfig(key)
  if v.kind == JString: return v.getStr()
  return default_

proc getConfigInt(key: string, default_ = 0): int =
  let v = getConfig(key)
  if v.kind == JInt: return v.getInt()
  return default_

proc getConfigBool(key: string, default_ = false): bool =
  let v = getConfig(key)
  if v.kind == JBool: return v.getBool()
  return default_

proc getConfigFloat(key: string, default_ = 0.0): float =
  let v = getConfig(key)
  if v.kind == JFloat: return v.getFloat()
  if v.kind == JInt: return float(v.getInt())
  return default_

# ============================
# Gradual rollout
# ============================

type
  RolloutStage = object
    percentage: float
    startedAt: float
    metrics: Table[string, float]
    status: string   # "running" | "paused" | "completed" | "rolled_back"

  GradualRollout = object
    featureId: string
    stages: seq[RolloutStage]
    currentStage: int
    autoAdvance: bool
    successThreshold: float   # error rate must be below this
    rollbackThreshold: float  # auto rollback if above this

var rollouts: Table[string, GradualRollout] = initTable[string, GradualRollout]()

proc startRollout(featureId: string, stages: seq[float],
                  autoAdvance = true): GradualRollout =
  let rolloutStages = stages.mapIt(RolloutStage(
    percentage: it,
    startedAt: 0.0,
    metrics: initTable[string, float](),
    status: "pending"
  ))
  
  let rollout = GradualRollout(
    featureId: featureId,
    stages: rolloutStages,
    currentStage: 0,
    autoAdvance: autoAdvance,
    successThreshold: 1.0,   # < 1% error rate
    rollbackThreshold: 5.0   # > 5% error rate
  )
  
  rollouts[featureId] = rollout
  echo fmt"[Rollout] Started: {featureId} with stages {stages}%"
  return rollout

proc advanceRollout(featureId: string) =
  if featureId notin rollouts: return
  var rollout = rollouts[featureId]
  
  if rollout.currentStage >= rollout.stages.len - 1:
    echo fmt"[Rollout] {featureId} already at max ({rollout.stages[^1].percentage}%)"
    return
  
  rollout.stages[rollout.currentStage].status = "completed"
  inc rollout.currentStage
  rollout.stages[rollout.currentStage].startedAt = epochTime()
  rollout.stages[rollout.currentStage].status = "running"
  
  let newPct = rollout.stages[rollout.currentStage].percentage
  rollouts[featureId] = rollout
  
  # Update feature flag percentage
  if featureId in flags:
    flags[featureId].percentage = newPct
  
  echo fmt"[Rollout] Advanced {featureId} to {newPct}%"

proc rollbackRollout(featureId: string) =
  if featureId notin rollouts: return
  var rollout = rollouts[featureId]
  
  rollout.stages[rollout.currentStage].status = "rolled_back"
  
  if rollout.currentStage > 0:
    dec rollout.currentStage
    let prevPct = rollout.stages[rollout.currentStage].percentage
    rollout.stages[rollout.currentStage].status = "running"
    rollouts[featureId] = rollout
    
    if featureId in flags:
      flags[featureId].percentage = prevPct
    
    echo fmt"[Rollout] Rolled back {featureId} to {prevPct}%"
  else:
    # Disable completely
    if featureId in flags:
      flags[featureId].enabled = false
    echo fmt"[Rollout] Disabled {featureId} (rolled back from stage 0)"

proc recordMetric(featureId: string, metric: string, value: float) =
  if featureId notin rollouts: return
  rollouts[featureId].stages[rollouts[featureId].currentStage].metrics[metric] = value

# ============================
# Demo
# ============================

proc demo() =
  echo "=== Remote Config & Gradual Rollout Demo ==="
  
  # Set remote config values
  echo "\n--- Remote Config ---"
  setConfig("production", "max_file_size_mb", %50, "Max upload file size")
  setConfig("production", "rate_limit_per_min", %100, "API rate limit")
  setConfig("production", "feature_timeout_ms", %5000.0, "Feature timeout")
  setConfig("production", "maintenance_mode", %false, "Kill switch")
  
  setConfig("default", "app_name", %"MyApp", "Application name")
  setConfig("default", "support_email", %"support@example.com")
  
  echo "\nConfig values:"
  echo fmt"  max_file_size: {getConfigInt(\"max_file_size_mb\")} MB"
  echo fmt"  rate_limit: {getConfigInt(\"rate_limit_per_min\")}/min"
  echo fmt"  timeout: {getConfigFloat(\"feature_timeout_ms\", 3000.0):.0f}ms"
  echo fmt"  maintenance: {getConfigBool(\"maintenance_mode\")}"
  echo fmt"  app_name: {getConfigStr(\"app_name\", \"Unknown\")}"
  
  # Gradual rollout
  echo "\n--- Gradual Rollout ---"
  
  # Register flag for rollout
  registerFlag(FeatureFlag(
    id: "new_search", name: "New Search Algorithm",
    type_: ftPercentage, enabled: true,
    percentage: 5.0,
    defaultValue: newJBool(false)
  ))
  
  let rollout = startRollout("new_search", @[5.0, 10.0, 25.0, 50.0, 100.0])
  echo fmt"Initial percentage: {flags[\"new_search\"].percentage}%"
  
  # Simulate health metrics OK -> advance
  recordMetric("new_search", "error_rate", 0.3)
  advanceRollout("new_search")
  echo fmt"After advance: {flags[\"new_search\"].percentage}%"
  
  advanceRollout("new_search")
  echo fmt"After 2nd advance: {flags[\"new_search\"].percentage}%"
  
  # Simulate high error rate -> rollback
  recordMetric("new_search", "error_rate", 8.5)
  rollbackRollout("new_search")
  echo fmt"After rollback: {flags[\"new_search\"].percentage}%"

demo()
```

---

## 📝 สรุป Part 47

| Steps | หัวข้อ |
|-------|--------|
| 676 | Feature flag engine (boolean, percentage, targeting, variant) |
| 677-690 | Remote config, gradual rollout, auto rollback |

---

**← [Part 46: Notifications](part_46_notifications.md) | [Part 48: API Gateway →](part_48_api_gateway.md)**
