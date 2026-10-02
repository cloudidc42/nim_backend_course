# Part 32: Security Best Practices
## Steps 451-465: Security ระดับ Production

---

## 🎯 เป้าหมายของ Part นี้

- Input validation and sanitization
- SQL injection prevention
- XSS prevention
- CSRF protection
- Rate limiting
- Security headers
- Secrets management
- Cryptography basics

---

## Step 451: Input Validation

```nim
import strutils, re, strformat, options, json, sequtils

# ==============================
# Validators
# ==============================

type
  ValidationResult = object
    valid: bool
    error: string

proc ok(): ValidationResult = ValidationResult(valid: true)
proc err(msg: string): ValidationResult = ValidationResult(valid: false, error: msg)

# Email validation
proc validateEmail(email: string): ValidationResult =
  if email.len == 0:
    return err("Email is required")
  if email.len > 255:
    return err("Email is too long (max 255 chars)")
  
  let parts = email.split('@')
  if parts.len != 2:
    return err("Email must contain exactly one @")
  
  let local = parts[0]
  let domain = parts[1]
  
  if local.len == 0 or local.len > 64:
    return err("Email local part must be 1-64 chars")
  
  if domain.len < 4 or '.' notin domain:
    return err("Email domain is invalid")
  
  if not email.match(re"^[a-zA-Z0-9._%+\-]+@[a-zA-Z0-9.\-]+\.[a-zA-Z]{2,}$"):
    return err("Email format is invalid")
  
  return ok()

# Password validation
proc validatePassword(password: string): ValidationResult =
  if password.len < 8:
    return err("Password must be at least 8 characters")
  if password.len > 100:
    return err("Password is too long (max 100 chars)")
  
  var hasUpper = false
  var hasLower = false
  var hasDigit = false
  
  for c in password:
    if c.isUpperAscii(): hasUpper = true
    elif c.isLowerAscii(): hasLower = true
    elif c.isDigit(): hasDigit = true
  
  if not hasUpper:
    return err("Password must contain at least one uppercase letter")
  if not hasLower:
    return err("Password must contain at least one lowercase letter")
  if not hasDigit:
    return err("Password must contain at least one digit")
  
  # Common passwords check
  let common = ["password", "12345678", "qwerty123", "admin123", "password1"]
  if password.toLowerAscii() in common:
    return err("Password is too common")
  
  return ok()

# Username validation
proc validateUsername(username: string): ValidationResult =
  if username.len < 3:
    return err("Username must be at least 3 characters")
  if username.len > 30:
    return err("Username is too long (max 30 chars)")
  
  if not username.match(re"^[a-zA-Z0-9_\-]+$"):
    return err("Username can only contain letters, numbers, _ and -")
  
  if username[0] in {'_', '-'}:
    return err("Username cannot start with _ or -")
  
  let reserved = ["admin", "root", "system", "api", "www", "mail", "support"]
  if username.toLowerAscii() in reserved:
    return err("Username is reserved")
  
  return ok()

# URL validation
proc validateUrl(url: string): ValidationResult =
  if url.len == 0:
    return err("URL is required")
  
  if not (url.startsWith("http://") or url.startsWith("https://")):
    return err("URL must start with http:// or https://")
  
  if url.len > 2048:
    return err("URL is too long")
  
  # Prevent open redirects to dangerous protocols
  let dangerousPrefixes = ["javascript:", "data:", "vbscript:", "file:"]
  for prefix in dangerousPrefixes:
    if url.toLowerAscii().contains(prefix):
      return err("URL contains dangerous protocol")
  
  return ok()

# Test
echo "=== Input Validation ==="
echo validateEmail("user@example.com").valid      # true
echo validateEmail("invalid-email").error          # bad format
echo validateEmail("a" & repeat("x", 65) & "@b.com").error  # too long

echo validatePassword("SecureP@ss1").valid         # true
echo validatePassword("weak").error                # too short
echo validatePassword("PASSWORD123").error         # no lowercase

echo validateUsername("alice_123").valid           # true
echo validateUsername("ad").error                  # too short
echo validateUsername("admin").error               # reserved
```

---

## Step 452: SQL Injection Prevention

```nim
import strutils, strformat, sequtils

# ==============================
# The Problem
# ==============================

# NEVER DO THIS:
proc unsafeSearch(name: string): string =
  # SQL injection: name = "' OR '1'='1"
  # Results in: SELECT * FROM users WHERE name = '' OR '1'='1'
  return fmt"SELECT * FROM users WHERE name = '{name}'"

# ==============================
# Safe: Parameterized Queries
# ==============================

type
  QueryParam = object
    value: string
    paramType: string  # text, int, bool

  SafeQuery = object
    sql: string
    params: seq[QueryParam]

proc newQuery(sql: string): SafeQuery =
  SafeQuery(sql: sql, params: @[])

proc addParam(query: var SafeQuery, value: string, kind: string = "text"): SafeQuery =
  query.params.add(QueryParam(value: value, paramType: kind))
  return query

# Query builder with parameterization
type
  WhereClause = object
    condition: string
    params: seq[QueryParam]

proc eq(field, value: string): WhereClause =
  WhereClause(condition: fmt"{field} = $?", params: @[QueryParam(value: value, paramType: "text")])

proc like(field, pattern: string): WhereClause =
  WhereClause(condition: fmt"{field} ILIKE $?", params: @[QueryParam(value: pattern, paramType: "text")])

proc gt(field: string, value: int): WhereClause =
  WhereClause(condition: fmt"{field} > $?", params: @[QueryParam(value: $value, paramType: "int")])

proc buildSelectQuery(table: string, fields: seq[string], wheres: seq[WhereClause],
                      limit: int = 100, offset: int = 0): SafeQuery =
  var paramIndex = 1
  var conditions: seq[string] = @[]
  var allParams: seq[QueryParam] = @[]
  
  for w in wheres:
    var condition = w.condition
    for _ in w.params:
      condition = condition.replace("$?", "$" & $paramIndex, 1)
      inc paramIndex
    conditions.add(condition)
    allParams.add(w.params)
  
  let fieldList = fields.join(", ")
  var sql = fmt"SELECT {fieldList} FROM {table}"
  
  if conditions.len > 0:
    sql &= " WHERE " & conditions.join(" AND ")
  
  sql &= fmt" LIMIT ${paramIndex} OFFSET ${paramIndex + 1}"
  allParams.add(QueryParam(value: $limit, paramType: "int"))
  allParams.add(QueryParam(value: $offset, paramType: "int"))
  
  return SafeQuery(sql: sql, params: allParams)

# Test
let attackInput = "'; DROP TABLE users; --"
let safeQuery = buildSelectQuery("users",
  @["id", "name", "email"],
  @[eq("name", attackInput), gt("age", 18)],
  limit = 10, offset = 0
)

echo "SQL: " & safeQuery.sql
echo "Params:"
for i, p in safeQuery.params:
  echo fmt"  ${i+1} = '{p.value}' (type: {p.paramType})"
# The attack string goes into params, not the SQL string itself
# PostgreSQL driver sends them separately, so injection is impossible

# ==============================
# Input sanitization for display
# ==============================

proc sanitizeForSql(s: string): string =
  # Only use this when parameterized queries aren't available
  s.replace("'", "''").replace("\\", "\\\\")

proc stripNullBytes(s: string): string =
  s.replace("\x00", "")

echo "\nSanitized: " & sanitizeForSql(attackInput)
```

---

## Step 453: XSS Prevention

```nim
import strutils, strformat, tables

# ==============================
# HTML Escaping
# ==============================

proc escapeHtml(s: string): string =
  result = s
    .replace("&", "&amp;")    # MUST be first
    .replace("<", "&lt;")
    .replace(">", "&gt;")
    .replace("\"", "&quot;")
    .replace("'", "&#x27;")
    .replace("/", "&#x2F;")

proc escapeJs(s: string): string =
  # For embedding strings in JavaScript
  result = s
    .replace("\\", "\\\\")
    .replace("\"", "\\\"")
    .replace("'", "\\'")
    .replace("\n", "\\n")
    .replace("\r", "\\r")
    .replace("<", "\\x3C")    # Prevent </script>
    .replace(">", "\\x3E")
    .replace("&", "\\x26")

proc escapeUrl(s: string): string =
  # Percent-encode special characters
  result = ""
  for c in s:
    if c in {'a'..'z', 'A'..'Z', '0'..'9', '-', '_', '.', '~'}:
      result.add(c)
    else:
      result.add('%')
      result.add(toHex(ord(c).uint8, 2))

proc sanitizeHtml(html: string): string =
  # Remove script tags and event handlers
  var result = html
  let dangerous = [
    re"(?i)<script[^>]*>.*?</script>",
    re"(?i)on\w+\s*=\s*[\"'][^\"']*[\"']",
    re"(?i)javascript:",
    re"(?i)data:text/html",
  ]
  # In production, use a proper HTML sanitizer library
  return result

# ==============================
# Content Security Policy
# ==============================

proc cspHeader(
  defaultSrc: seq[string] = @["'self'"],
  scriptSrc: seq[string] = @["'self'"],
  styleSrc: seq[string] = @["'self'"],
  imgSrc: seq[string] = @["'self'", "data:"],
  connectSrc: seq[string] = @["'self'"]
): string =
  let directives = [
    "default-src " & defaultSrc.join(" "),
    "script-src " & scriptSrc.join(" "),
    "style-src " & styleSrc.join(" "),
    "img-src " & imgSrc.join(" "),
    "connect-src " & connectSrc.join(" "),
    "frame-ancestors 'none'",
    "form-action 'self'",
    "upgrade-insecure-requests",
  ]
  return directives.join("; ")

# Test
let userInput = "<script>alert('XSS!')</script><b>Hello</b>"
echo "Original: " & userInput
echo "Escaped HTML: " & escapeHtml(userInput)

let jsInput = "Hello \"World\"\nNew line"
echo "Escaped JS: " & escapeJs(jsInput)

let urlInput = "Hello World! (test)"
echo "URL encoded: " & escapeUrl(urlInput)

echo "\nCSP Header:"
echo cspHeader(
  scriptSrc = @["'self'", "https://cdn.example.com"],
  styleSrc = @["'self'", "https://fonts.googleapis.com"]
)
```

---

## Step 454: Security Headers

```nim
import asyncdispatch, asynchttpserver, strutils

proc addSecurityHeaders(headers: var HttpHeaders) =
  # Prevent clickjacking
  headers["X-Frame-Options"] = "DENY"
  
  # Prevent MIME sniffing
  headers["X-Content-Type-Options"] = "nosniff"
  
  # XSS protection (for older browsers)
  headers["X-XSS-Protection"] = "1; mode=block"
  
  # HTTPS only
  headers["Strict-Transport-Security"] = "max-age=31536000; includeSubDomains; preload"
  
  # Referrer policy
  headers["Referrer-Policy"] = "strict-origin-when-cross-origin"
  
  # Feature policy / Permissions policy
  headers["Permissions-Policy"] = "geolocation=(), microphone=(), camera=()"
  
  # Content Security Policy
  headers["Content-Security-Policy"] = """
    default-src 'self';
    script-src 'self' https://cdn.jsdelivr.net;
    style-src 'self' https://fonts.googleapis.com 'unsafe-inline';
    font-src https://fonts.gstatic.com;
    img-src 'self' data: https:;
    connect-src 'self' https://api.example.com;
    frame-ancestors 'none';
  """.replace("\n", " ").strip()
  
  # Remove sensitive server info
  headers["Server"] = "nim"
  headers.del("X-Powered-By")

proc secureMiddleware(req: Request, next: proc(req: Request): Future[void]): Future[void] {.async.} =
  # Check for suspicious patterns in URL
  let path = req.url.path
  
  # Path traversal check
  if ".." in path or path.contains("//"):
    await req.respond(Http400, """{"error":"Invalid path"}""",
      newHttpHeaders([("Content-Type", "application/json")]))
    return
  
  # Very long URLs (potential DoS or injection)
  if req.url.path.len > 2048:
    await req.respond(Http414, """{"error":"URI too long"}""",
      newHttpHeaders([("Content-Type", "application/json")]))
    return
  
  await next(req)

echo "Security headers configured"
```

---

## Step 455-465: Complete Security System

```nim
# security.nim - Complete security utilities

import strutils, strformat, times, tables, json, hashes, sequtils, options
import base64

# ==============================
# CSRF Token
# ==============================

type
  CsrfStore = object
    tokens: Table[string, float]  # token -> expiry
    ttl: float

proc newCsrfStore(ttl: float = 3600.0): CsrfStore =
  CsrfStore(tokens: initTable[string, float](), ttl: ttl)

proc generateCsrfToken(store: var CsrfStore, sessionId: string): string =
  let token = base64.encode(sessionId & $epochTime() & $rand(high(int)))
  store.tokens[token] = epochTime() + store.ttl
  return token

proc validateCsrfToken(store: var CsrfStore, token: string): bool =
  if token notin store.tokens:
    return false
  
  let expiry = store.tokens[token]
  if epochTime() > expiry:
    store.tokens.del(token)
    return false
  
  # Single-use: delete after validation
  store.tokens.del(token)
  return true

proc cleanupExpiredTokens(store: var CsrfStore) =
  let now = epochTime()
  var expired: seq[string] = @[]
  for token, expiry in store.tokens:
    if now > expiry:
      expired.add(token)
  for token in expired:
    store.tokens.del(token)

# ==============================
# Secrets Management
# ==============================

type
  SecretStore = object
    secrets: Table[string, string]
    encrypted: bool

proc loadSecretsFromEnv(): SecretStore =
  var store = SecretStore(secrets: initTable[string, string](), encrypted: false)
  
  # In production, use HashiCorp Vault, AWS Secrets Manager, etc.
  let envVars = [
    "DATABASE_URL",
    "JWT_SECRET",
    "REDIS_URL",
    "SMTP_PASSWORD",
    "API_KEY_STRIPE",
  ]
  
  for envVar in envVars:
    let value = getEnv(envVar, "")
    if value.len > 0:
      store.secrets[envVar] = value
  
  return store

proc getSecret(store: SecretStore, key: string): Option[string] =
  if key in store.secrets:
    return some(store.secrets[key])
  return none(string)

proc validateSecretNotEmpty(store: SecretStore, key: string) =
  let secret = store.getSecret(key)
  if secret.isNone or secret.get().len == 0:
    raise newException(ValueError, fmt"Required secret '{key}' is not set")

# ==============================
# Password Policy
# ==============================

type
  PasswordPolicy = object
    minLength: int
    maxLength: int
    requireUppercase: bool
    requireLowercase: bool
    requireDigit: bool
    requireSpecial: bool
    specialChars: string
    bannedPasswords: seq[string]
    maxRepeatedChars: int

let defaultPolicy = PasswordPolicy(
  minLength: 12,
  maxLength: 128,
  requireUppercase: true,
  requireLowercase: true,
  requireDigit: true,
  requireSpecial: true,
  specialChars: "!@#$%^&*()_+-=[]{}|;:,.<>?",
  bannedPasswords: @[
    "password123!", "Admin123!", "Welcome1!", "P@ssword1"
  ],
  maxRepeatedChars: 3
)

proc checkPasswordPolicy(password: string, policy: PasswordPolicy): seq[string] =
  var errors: seq[string] = @[]
  
  if password.len < policy.minLength:
    errors.add(fmt"Min length: {policy.minLength}")
  if password.len > policy.maxLength:
    errors.add(fmt"Max length: {policy.maxLength}")
  
  if policy.requireUppercase and not password.any(c => c.isUpperAscii()):
    errors.add("Must contain uppercase letter")
  if policy.requireLowercase and not password.any(c => c.isLowerAscii()):
    errors.add("Must contain lowercase letter")
  if policy.requireDigit and not password.any(c => c.isDigit()):
    errors.add("Must contain digit")
  if policy.requireSpecial and not password.any(c => c in policy.specialChars):
    errors.add(fmt"Must contain special char ({policy.specialChars[0..5]}...)")
  
  if password.toLowerAscii() in policy.bannedPasswords.mapIt(it.toLowerAscii()):
    errors.add("Password is too common")
  
  # Check repeated chars
  var maxRep = 1
  var curRep = 1
  for i in 1..<password.len:
    if password[i] == password[i-1]:
      inc curRep
      maxRep = max(maxRep, curRep)
    else:
      curRep = 1
  
  if maxRep > policy.maxRepeatedChars:
    errors.add(fmt"Max {policy.maxRepeatedChars} repeated chars in a row")
  
  return errors

# ==============================
# Rate Limiting
# ==============================

type
  RateLimit = object
    limit: int
    window: float  # seconds
    
  RateLimiter = object
    limits: Table[string, RateLimit]
    counters: Table[string, (int, float)]  # (count, window_start)

proc newRateLimiter(): RateLimiter =
  RateLimiter(
    limits: initTable[string, RateLimit](),
    counters: initTable[string, (int, float)]()
  )

proc setLimit(rl: var RateLimiter, key: string, limit: int, window: float) =
  rl.limits[key] = RateLimit(limit: limit, window: window)

proc isAllowed(rl: var RateLimiter, key: string, id: string): bool =
  if key notin rl.limits:
    return true
  
  let limit = rl.limits[key]
  let counterKey = key & ":" & id
  let now = epochTime()
  
  if counterKey in rl.counters:
    let (count, windowStart) = rl.counters[counterKey]
    
    if now - windowStart > limit.window:
      # Reset window
      rl.counters[counterKey] = (1, now)
      return true
    
    if count >= limit.limit:
      return false
    
    rl.counters[counterKey] = (count + 1, windowStart)
  else:
    rl.counters[counterKey] = (1, now)
  
  return true

proc getRemainingRequests(rl: RateLimiter, key: string, id: string): int =
  if key notin rl.limits:
    return high(int)
  
  let limit = rl.limits[key]
  let counterKey = key & ":" & id
  
  if counterKey notin rl.counters:
    return limit.limit
  
  let (count, windowStart) = rl.counters[counterKey]
  
  if epochTime() - windowStart > limit.window:
    return limit.limit
  
  return max(0, limit.limit - count)

# ==============================
# Demo
# ==============================

proc securityDemo() =
  echo "=== Security Demo ==="
  
  # CSRF tokens
  var csrfStore = newCsrfStore(ttl = 3600.0)
  let token1 = csrfStore.generateCsrfToken("session-abc")
  let token2 = csrfStore.generateCsrfToken("session-xyz")
  
  echo fmt"\nCSRF tokens generated: {csrfStore.tokens.len}"
  echo fmt"Token1 valid: {csrfStore.validateCsrfToken(token1)}"
  echo fmt"Token1 reuse: {csrfStore.validateCsrfToken(token1)}"  # single-use!
  
  # Password policy
  echo "\nPassword policy checks:"
  let passwords = [
    ("weak", "weak"),
    ("StrongP@ss123!", "strong"),
    ("aaaBBB111!!!", "repeats"),
    ("Admin123!", "common"),
  ]
  for (password, label) in passwords:
    let errors = checkPasswordPolicy(password, defaultPolicy)
    if errors.len == 0:
      echo fmt"  ✓ '{label}': valid"
    else:
      echo fmt"  ✗ '{label}': {errors[0]}"
  
  # Rate limiting
  var rl = newRateLimiter()
  rl.setLimit("login", 5, 60.0)  # 5 attempts per minute
  
  echo "\nRate limit test (5 per 60s):"
  let clientIp = "192.168.1.1"
  for i in 1..7:
    let allowed = rl.isAllowed("login", clientIp)
    let remaining = rl.getRemainingRequests("login", clientIp)
    echo fmt"  Attempt {i}: {if allowed: \"✓ allowed\" else: \"✗ blocked\"} (remaining: {remaining})"

securityDemo()
```

---

## 📝 สรุป Part 32

| Steps | หัวข้อ |
|-------|--------|
| 451 | Input validation (email, password, username) |
| 452 | SQL injection prevention (parameterized queries) |
| 453 | XSS prevention (HTML/JS/URL escaping) |
| 454 | Security headers (CSP, HSTS, etc.) |
| 455-465 | CSRF tokens, secrets management, password policy, rate limiting |

---

**← [Part 31: Advanced Testing](part_31_testing_advanced.md) | [Part 33: Performance Optimization →](part_33_performance.md)**
