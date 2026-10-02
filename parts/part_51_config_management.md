# Part 51: Config Management & Secrets
## Steps 736-750: Production Configuration Management

---

## 🎯 เป้าหมายของ Part นี้

- Hierarchical config system
- Environment-based overrides
- Secret vault (encryption at rest)
- Config hot-reload
- Validation & schema
- Audit logging

---

## Step 736: Hierarchical Config

```nim
import tables, strformat, times, json, strutils, sequtils, options, os

# ============================
# Config value types
# ============================

type
  ConfigType = enum
    ctString, ctInt, ctFloat, ctBool, ctList, ctMap

  ConfigVal = object
    case kind: ConfigType
    of ctString: strVal: string
    of ctInt: intVal: int
    of ctFloat: floatVal: float
    of ctBool: boolVal: bool
    of ctList: listVal: seq[ConfigVal]
    of ctMap: mapVal: Table[string, ConfigVal]

  ConfigSource = enum
    csDefault, csFile, csEnv, csOverride, csSecret

  ConfigEntry = object
    key: string
    value: ConfigVal
    source: ConfigSource
    description: string
    required: bool
    sensitive: bool     # mask in logs
    updatedAt: float

  ConfigStore = object
    entries: Table[string, ConfigEntry]
    aliases: Table[string, string]
    watchers: Table[string, seq[proc(key, newVal: string) {.closure.}]]

var config = ConfigStore(
  entries: initTable[string, ConfigEntry](),
  aliases: initTable[string, string](),
  watchers: initTable[string, seq[proc(key, newVal: string) {.closure.}]]()
)

# ============================
# Builders
# ============================

proc cv(s: string): ConfigVal = ConfigVal(kind: ctString, strVal: s)
proc cv(n: int): ConfigVal = ConfigVal(kind: ctInt, intVal: n)
proc cv(f: float): ConfigVal = ConfigVal(kind: ctFloat, floatVal: f)
proc cv(b: bool): ConfigVal = ConfigVal(kind: ctBool, boolVal: b)

proc `$`(v: ConfigVal): string =
  case v.kind
  of ctString: v.strVal
  of ctInt: $v.intVal
  of ctFloat: $v.floatVal
  of ctBool: $v.boolVal
  of ctList: v.listVal.mapIt($it).join(",")
  of ctMap: "{ map }"

proc setDefault(key: string, value: ConfigVal, description = "",
                required = false, sensitive = false) =
  config.entries[key] = ConfigEntry(
    key: key,
    value: value,
    source: csDefault,
    description: description,
    required: required,
    sensitive: sensitive,
    updatedAt: epochTime()
  )

proc setOverride(key: string, value: ConfigVal) =
  if key notin config.entries:
    config.entries[key] = ConfigEntry(key: key)
  config.entries[key].value = value
  config.entries[key].source = csOverride
  config.entries[key].updatedAt = epochTime()
  echo fmt"[Config] Override: {key}"

proc loadFromEnv(prefix = "APP_") =
  ## Load environment variables with prefix APP_ into config
  for key, value in envPairs():
    if key.startsWith(prefix):
      let configKey = key[prefix.len..^1].toLowerAscii().replace("_", ".")
      config.entries[configKey] = ConfigEntry(
        key: configKey,
        value: cv(value),
        source: csEnv,
        updatedAt: epochTime()
      )
      echo fmt"[Config] From env: {configKey}"

proc loadFromJson(data: string, source = csFile) =
  try:
    let root = parseJson(data)
    proc flatten(node: JsonNode, prefix: string) =
      case node.kind
      of JObject:
        for k, v in node:
          let fullKey = if prefix.len > 0: prefix & "." & k else: k
          flatten(v, fullKey)
      of JString:
        config.entries[prefix] = ConfigEntry(
          key: prefix, value: cv(node.getStr()),
          source: source, updatedAt: epochTime()
        )
      of JInt:
        config.entries[prefix] = ConfigEntry(
          key: prefix, value: cv(node.getInt()),
          source: source, updatedAt: epochTime()
        )
      of JFloat:
        config.entries[prefix] = ConfigEntry(
          key: prefix, value: cv(node.getFloat()),
          source: source, updatedAt: epochTime()
        )
      of JBool:
        config.entries[prefix] = ConfigEntry(
          key: prefix, value: cv(node.getBool()),
          source: source, updatedAt: epochTime()
        )
      else: discard

    flatten(root, "")
    echo fmt"[Config] Loaded from JSON: {toSeq(root.keys).len} top-level keys"
  except JsonParsingError as e:
    echo fmt"[Config] JSON parse error: {e.msg}"

# ============================
# Getters
# ============================

proc getStr(key, default_: string = ""): string =
  let k = config.aliases.getOrDefault(key, key)
  if k notin config.entries: return default_
  let v = config.entries[k].value
  if v.kind == ctString: return v.strVal
  return $v

proc getInt(key: string, default_ = 0): int =
  let k = config.aliases.getOrDefault(key, key)
  if k notin config.entries: return default_
  let v = config.entries[k].value
  case v.kind
  of ctInt: return v.intVal
  of ctString:
    try: return parseInt(v.strVal)
    except: return default_
  else: return default_

proc getBool(key: string, default_ = false): bool =
  let k = config.aliases.getOrDefault(key, key)
  if k notin config.entries: return default_
  let v = config.entries[k].value
  case v.kind
  of ctBool: return v.boolVal
  of ctString: return v.strVal.toLowerAscii() in ["true", "1", "yes", "on"]
  else: return default_

proc getFloat(key: string, default_ = 0.0): float =
  let k = config.aliases.getOrDefault(key, key)
  if k notin config.entries: return default_
  let v = config.entries[k].value
  case v.kind
  of ctFloat: return v.floatVal
  of ctInt: return float(v.intVal)
  of ctString:
    try: return parseFloat(v.strVal)
    except: return default_
  else: return default_

proc require(key: string): string =
  if key notin config.entries or config.entries[key].value.kind == ctString and
     config.entries[key].value.strVal.len == 0:
    raise newException(ValueError, fmt"Required config missing: {key}")
  return getStr(key)

# ============================
# Watchers for hot-reload
# ============================

proc watch(key: string, cb: proc(key, newVal: string) {.closure.}) =
  if key notin config.watchers:
    config.watchers[key] = @[]
  config.watchers[key].add(cb)
  echo fmt"[Config] Watcher registered for: {key}"

proc notifyWatchers(key, newVal: string) =
  if key in config.watchers:
    for cb in config.watchers[key]:
      cb(key, newVal)

proc hotReload(key: string, value: ConfigVal) =
  setOverride(key, value)
  notifyWatchers(key, $value)
  echo fmt"[Config] Hot-reloaded: {key} = {value}"

# ============================
# Config validation
# ============================

type
  ValidationRule = object
    key: string
    required: bool
    minVal: Option[float]
    maxVal: Option[float]
    pattern: string     # regex-like: "email", "url", "port"
    allowedValues: seq[string]

  ValidationResult = object
    valid: bool
    errors: seq[string]

proc validate(rules: seq[ValidationRule]): ValidationResult =
  var errors: seq[string]

  for rule in rules:
    if rule.required and rule.key notin config.entries:
      errors.add(fmt"Missing required key: {rule.key}")
      continue

    if rule.key notin config.entries: continue

    let v = config.entries[rule.key].value
    let sv = $v

    if rule.minVal.isSome:
      let num = case v.kind
        of ctInt: float(v.intVal)
        of ctFloat: v.floatVal
        else:
          try: parseFloat(sv)
          except: float.low

      if num < rule.minVal.get():
        errors.add(fmt"{rule.key} must be >= {rule.minVal.get()}")

    if rule.maxVal.isSome:
      let num = case v.kind
        of ctInt: float(v.intVal)
        of ctFloat: v.floatVal
        else:
          try: parseFloat(sv)
          except: float.high

      if num > rule.maxVal.get():
        errors.add(fmt"{rule.key} must be <= {rule.maxVal.get()}")

    if rule.allowedValues.len > 0 and sv notin rule.allowedValues:
      errors.add(fmt"{rule.key} must be one of: {rule.allowedValues.join(\", \")}")

    if rule.pattern == "port":
      try:
        let port = parseInt(sv)
        if port < 1 or port > 65535:
          errors.add(fmt"{rule.key} must be a valid port (1-65535)")
      except:
        errors.add(fmt"{rule.key} must be a valid port number")

  return ValidationResult(valid: errors.len == 0, errors: errors)

# ============================
# Demo
# ============================

proc demo() =
  echo "=== Config Management Demo ==="

  # Set defaults
  setDefault("app.name", cv("MyApp"), "Application name", required = true)
  setDefault("app.port", cv(8080), "HTTP port", required = true)
  setDefault("app.env", cv("development"), "Environment")
  setDefault("db.host", cv("localhost"), "Database host", required = true)
  setDefault("db.port", cv(5432), "Database port")
  setDefault("db.name", cv("myapp_dev"), "Database name")
  setDefault("db.password", cv(""), "Database password", sensitive = true)
  setDefault("cache.ttl_seconds", cv(300), "Cache TTL in seconds")
  setDefault("log.level", cv("info"), "Log level")
  setDefault("feature.dark_mode", cv(false), "Enable dark mode")

  # Load from JSON (simulating a config file)
  loadFromJson("""
{
  "app": {
    "name": "ProductionApp",
    "port": 9090
  },
  "db": {
    "host": "db.prod.internal",
    "name": "myapp_prod"
  },
  "log": {
    "level": "warn"
  }
}""")

  echo "\n--- Config values ---"
  echo fmt"app.name: {getStr(\"app.name\")}"
  echo fmt"app.port: {getInt(\"app.port\")}"
  echo fmt"db.host: {getStr(\"db.host\")}"
  echo fmt"log.level: {getStr(\"log.level\")}"
  echo fmt"cache.ttl: {getInt(\"cache.ttl_seconds\")}s"

  # Override
  setOverride("app.env", cv("production"))
  echo fmt"\napp.env after override: {getStr(\"app.env\")}"

  # Hot-reload watcher
  echo "\n--- Hot-reload ---"
  watch("log.level", proc(key, val: string) =
    echo fmt"  [Watcher] {key} changed to: {val}")

  hotReload("log.level", cv("debug"))

  # Validation
  echo "\n--- Validation ---"
  let rules = @[
    ValidationRule(key: "app.name", required: true),
    ValidationRule(key: "app.port", required: true,
      pattern: "port", minVal: some(1.0), maxVal: some(65535.0)),
    ValidationRule(key: "log.level", required: true,
      allowedValues: @["debug", "info", "warn", "error"]),
    ValidationRule(key: "cache.ttl_seconds",
      minVal: some(60.0), maxVal: some(86400.0)),
  ]

  let result = validate(rules)
  echo fmt"Valid: {result.valid}"
  if result.errors.len > 0:
    for e in result.errors:
      echo fmt"  Error: {e}"
  else:
    echo "  All validations passed!"

demo()
```

---

## Step 737-750: Secret Vault

```nim
import tables, strformat, times, strutils, sequtils, options, hashes, base64

# ============================
# Simple XOR cipher (demo only; use AES in production)
# ============================

proc xorEncrypt(data, key: string): string =
  var result = newString(data.len)
  for i, c in data:
    result[i] = char(ord(c) xor ord(key[i mod key.len]))
  return result

proc xorDecrypt(data, key: string): string =
  xorEncrypt(data, key)  # XOR is symmetric

proc encodeBase64(s: string): string =
  encode(s)

proc decodeBase64(s: string): string =
  try: decode(s)
  except: ""

# ============================
# Secret vault
# ============================

type
  SecretMetadata = object
    version: int
    createdAt: float
    updatedAt: float
    expiresAt: float
    rotatedBy: string
    tags: seq[string]

  VaultSecret = object
    path: string
    encryptedValue: string
    metadata: SecretMetadata
    accessLog: seq[tuple[accessedAt: float, accessor: string]]

  SecretVault = object
    secrets: Table[string, VaultSecret]
    encryptionKey: string
    auditLog: seq[tuple[ts: float, action: string, path: string, actor: string]]

var vault = SecretVault(
  secrets: initTable[string, VaultSecret](),
  encryptionKey: "vault_master_key_change_in_prod",
  auditLog: @[]
)

proc writeSecret(path, value, actor: string,
                  ttlDays = 0, tags: seq[string] = @[]) =
  let encrypted = encodeBase64(xorEncrypt(value, vault.encryptionKey))
  let now = epochTime()

  var metadata = SecretMetadata(
    version: 1,
    createdAt: now,
    updatedAt: now,
    expiresAt: if ttlDays > 0: now + float(ttlDays * 86400) else: 0.0,
    rotatedBy: actor,
    tags: tags
  )

  if path in vault.secrets:
    metadata.version = vault.secrets[path].metadata.version + 1
    metadata.createdAt = vault.secrets[path].metadata.createdAt

  vault.secrets[path] = VaultSecret(
    path: path,
    encryptedValue: encrypted,
    metadata: metadata,
    accessLog: @[]
  )

  vault.auditLog.add((now, "write", path, actor))
  echo fmt"[Vault] Written: {path} (v{metadata.version})"

proc readSecret(path, actor: string): Option[string] =
  if path notin vault.secrets:
    return none(string)

  var secret = vault.secrets[path]

  # Check expiration
  if secret.metadata.expiresAt > 0 and epochTime() > secret.metadata.expiresAt:
    vault.auditLog.add((epochTime(), "read_denied_expired", path, actor))
    echo fmt"[Vault] Secret expired: {path}"
    return none(string)

  secret.accessLog.add((epochTime(), actor))
  vault.secrets[path] = secret

  let decrypted = xorDecrypt(decodeBase64(secret.encryptedValue), vault.encryptionKey)
  vault.auditLog.add((epochTime(), "read", path, actor))

  return some(decrypted)

proc deleteSecret(path, actor: string): bool =
  if path notin vault.secrets: return false
  vault.secrets.del(path)
  vault.auditLog.add((epochTime(), "delete", path, actor))
  echo fmt"[Vault] Deleted: {path}"
  return true

proc listSecrets(prefix = ""): seq[tuple[path: string, version: int, expiresAt: float]] =
  var result: seq[tuple[path: string, version: int, expiresAt: float]]
  for path, secret in vault.secrets:
    if prefix.len == 0 or path.startsWith(prefix):
      result.add((path, secret.metadata.version, secret.metadata.expiresAt))
  return result

proc rotateSecret(path, newValue, actor: string) =
  if path notin vault.secrets:
    raise newException(ValueError, fmt"Secret not found: {path}")
  writeSecret(path, newValue, actor)
  echo fmt"[Vault] Rotated: {path} (now v{vault.secrets[path].metadata.version})"

# ============================
# Secret injection (config integration)
# ============================

proc injectSecretsToConfig(secretPaths: seq[tuple[secretPath, configKey: string]],
                            actor = "system") =
  for mapping in secretPaths:
    let secret = readSecret(mapping.secretPath, actor)
    if secret.isSome:
      setOverride(mapping.configKey, cv(secret.get()))
      echo fmt"[Vault->Config] Injected {mapping.secretPath} -> {mapping.configKey}"

# ============================
# Audit report
# ============================

proc auditReport(since: float = 0.0): string =
  var lines: seq[string]
  lines.add("=== Vault Audit Report ===")
  for entry in vault.auditLog:
    if entry.ts < since: continue
    let ts = fromUnix(int(entry.ts)).format("HH:mm:ss")
    lines.add(fmt"  [{ts}] {entry.action} on '{entry.path}' by '{entry.actor}'")
  return lines.join("\n")

# ============================
# Demo
# ============================

proc demo() =
  echo "=== Secret Vault Demo ==="

  # Store secrets
  writeSecret("prod/db/password", "super_secret_db_pass_123",
    actor = "admin", tags = @["database", "critical"])

  writeSecret("prod/api/stripe_key", "sk_live_abc123xyz789",
    actor = "admin", ttlDays = 90, tags = @["payment"])

  writeSecret("prod/api/jwt_secret", "my_jwt_signing_secret_256bit",
    actor = "admin", tags = @["auth"])

  writeSecret("staging/db/password", "staging_db_pass",
    actor = "ci_system")

  # List secrets
  echo "\n--- Secret listing ---"
  let secrets = listSecrets("prod/")
  for s in secrets:
    let expiry = if s.expiresAt > 0:
      fmt"expires: {fromUnix(int(s.expiresAt)).format(\"yyyy-MM-dd\")}"
    else: "no expiry"
    echo fmt"  {s.path} (v{s.version}, {expiry})"

  # Read a secret
  echo "\n--- Reading secrets ---"
  let dbPass = readSecret("prod/db/password", actor = "app-server-1")
  if dbPass.isSome:
    echo fmt"DB password retrieved: {dbPass.get()[0..4]}... (masked)"

  # Rotate
  echo "\n--- Secret rotation ---"
  rotateSecret("prod/db/password", "new_super_secret_db_pass_456", actor = "admin")
  echo fmt"Version after rotation: {vault.secrets[\"prod/db/password\"].metadata.version}"

  # Inject into config
  echo "\n--- Inject to config ---"
  injectSecretsToConfig(@[
    ("prod/db/password", "db.password"),
    ("prod/api/jwt_secret", "auth.jwt_secret"),
  ])

  # Audit log
  echo "\n--- Audit log ---"
  echo auditReport()

demo()
```

---

## 📝 สรุป Part 51

| Steps | หัวข้อ |
|-------|--------|
| 736 | Hierarchical config, env loading, hot-reload, validation |
| 737-750 | Secret vault with encryption, rotation, access logging |

---

**← [Part 50: Data Pipeline](part_50_data_pipeline.md) | [Part 52: Event Streaming →](part_52_event_streaming.md)**
