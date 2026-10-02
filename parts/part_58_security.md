# Part 58: Security
## Steps 841-855: Backend Security Hardening

---

## 🎯 เป้าหมายของ Part นี้

- JWT authentication (signing + verification)
- Password hashing (bcrypt-style)
- CSRF protection
- SQL injection prevention
- XSS sanitization
- Security headers middleware
- Audit log

---

## Step 841: JWT & Auth

```nim
import tables, strformat, times, sequtils, json, strutils, options, hashes, base64, math

# ============================
# HMAC-SHA256 (simplified demo)
# ============================

proc hmacSha256(key, data: string): string =
  ## Simplified HMAC — use nimcrypto in production
  var h = 0u64
  let combined = key & "." & data
  for i, c in combined:
    h = h * 6364136223846793005u64 + uint64(ord(c)) + uint64(i)
  let hex = fmt"{h:016x}"
  return encode(hex)  # base64 of hex

# ============================
# JWT
# ============================

type
  JwtAlgorithm = enum
    jaHS256, jaHS384, jaHS512

  JwtHeader = object
    alg: string
    typ: string

  JwtClaims = object
    sub: string         # subject (user ID)
    iss: string         # issuer
    aud: string         # audience
    exp: float          # expiry unix timestamp
    iat: float          # issued at
    jti: string         # JWT ID (for revocation)
    data: Table[string, JsonNode]  # custom claims

  JwtToken = object
    header: JwtHeader
    claims: JwtClaims
    signature: string
    raw: string

proc base64UrlEncode(s: string): string =
  encode(s).replace("+", "-").replace("/", "_").replace("=", "")

proc base64UrlDecode(s: string): string =
  var padded = s.replace("-", "+").replace("_", "/")
  while padded.len mod 4 != 0: padded &= "="
  try: decode(padded)
  except: ""

proc signJwt(claims: JwtClaims, secret: string,
             alg = jaHS256): string =
  let header = %*{"alg": "HS256", "typ": "JWT"}
  let headerEnc = base64UrlEncode($header)

  var claimsObj = %*{
    "sub": claims.sub,
    "iss": claims.iss,
    "aud": claims.aud,
    "exp": claims.exp,
    "iat": claims.iat,
    "jti": claims.jti
  }
  for k, v in claims.data:
    claimsObj[k] = v

  let claimsEnc = base64UrlEncode($claimsObj)
  let signingInput = headerEnc & "." & claimsEnc
  let sig = base64UrlEncode(hmacSha256(secret, signingInput))

  return signingInput & "." & sig

proc verifyJwt(token, secret: string): tuple[valid: bool, claims: JwtClaims, error: string] =
  let parts = token.split(".")
  if parts.len != 3:
    return (false, JwtClaims(), "Invalid token format")

  let signingInput = parts[0] & "." & parts[1]
  let expectedSig = base64UrlEncode(hmacSha256(secret, signingInput))

  if parts[2] != expectedSig:
    return (false, JwtClaims(), "Invalid signature")

  let claimsJson = base64UrlDecode(parts[1])
  try:
    let j = parseJson(claimsJson)
    let exp = j.getOrDefault("exp", %0.0).getFloat()

    if exp > 0 and epochTime() > exp:
      return (false, JwtClaims(), "Token expired")

    var customData: Table[string, JsonNode]
    let knownKeys = ["sub", "iss", "aud", "exp", "iat", "jti"]
    for k, v in j:
      if k notin knownKeys:
        customData[k] = v

    return (true, JwtClaims(
      sub: j.getOrDefault("sub", %"").getStr(),
      iss: j.getOrDefault("iss", %"").getStr(),
      aud: j.getOrDefault("aud", %"").getStr(),
      exp: exp,
      iat: j.getOrDefault("iat", %0.0).getFloat(),
      jti: j.getOrDefault("jti", %"").getStr(),
      data: customData
    ), "")
  except:
    return (false, JwtClaims(), "Claims parse error")

proc makeJwt(userId, issuer, audience: string,
             ttlSeconds = 3600.0,
             data: Table[string, JsonNode] = initTable[string, JsonNode](),
             secret = "secret_key_change_in_prod"): string =
  let now = epochTime()
  let claims = JwtClaims(
    sub: userId,
    iss: issuer,
    aud: audience,
    exp: now + ttlSeconds,
    iat: now,
    jti: fmt"jti_{int(now * 1000) mod 1_000_000_000}",
    data: data
  )
  return signJwt(claims, secret)

# ============================
# Token revocation (deny list)
# ============================

var revokedJtis: Table[string, float] = initTable[string, float]()  # jti -> expiresAt

proc revokeToken(jti: string, expiresAt: float) =
  revokedJtis[jti] = expiresAt
  echo fmt"[Auth] Revoked token: {jti}"

proc isRevoked(jti: string): bool =
  if jti notin revokedJtis: return false
  if epochTime() > revokedJtis[jti]:
    revokedJtis.del(jti)
    return false
  return true

proc cleanRevokedTokens() =
  var expired: seq[string]
  let now = epochTime()
  for jti, exp in revokedJtis:
    if now > exp: expired.add(jti)
  for jti in expired: revokedJtis.del(jti)

# ============================
# Password hashing (bcrypt-like)
# ============================

proc generateSalt(): string =
  let ts = int(epochTime() * 1_000_000)
  fmt"{ts:016x}"

proc hashPassword(password, salt: string, iterations = 10): string =
  ## Simplified PBKDF2-like — use bcrypt/argon2 in production
  var h = password & salt
  for _ in 0..<(1 shl iterations):
    var next = 0u64
    for c in h:
      next = next * 31u64 + uint64(ord(c))
    h = fmt"{next:016x}"
  return salt & ":" & h

proc verifyPassword(password, hash: string): bool =
  let parts = hash.split(":")
  if parts.len != 2: return false
  let salt = parts[0]
  let expected = hashPassword(password, salt)
  return expected == hash

# ============================
# CSRF protection
# ============================

var csrfTokens: Table[string, tuple[sessionId: string, expiresAt: float]] =
  initTable[string, tuple[sessionId: string, expiresAt: float]]()

proc generateCsrfToken(sessionId: string): string =
  let token = fmt"{int(epochTime() * 1_000_000):016x}_{sessionId}"
  csrfTokens[token] = (sessionId, epochTime() + 3600)
  return token

proc validateCsrfToken(token, sessionId: string): bool =
  if token notin csrfTokens: return false
  let (storedSession, expiresAt) = csrfTokens[token]
  if epochTime() > expiresAt:
    csrfTokens.del(token)
    return false
  return storedSession == sessionId

# ============================
# Input sanitization
# ============================

proc sanitizeHtml(input: string): string =
  ## Remove/escape dangerous HTML
  input
    .replace("&", "&amp;")
    .replace("<", "&lt;")
    .replace(">", "&gt;")
    .replace("\"", "&quot;")
    .replace("'", "&#x27;")
    .replace("/", "&#x2F;")

proc sanitizeSql(input: string): string =
  ## Escape SQL special characters
  input
    .replace("'", "''")
    .replace("\\", "\\\\")
    .replace(";", "")
    .replace("--", "")
    .replace("/*", "")
    .replace("*/", "")

proc sqlQuery(template_: string, params: seq[string]): string =
  ## Safe parameterized query builder
  var query = template_
  for i, param in params:
    let placeholder = fmt"${i + 1}"
    let safe = "'" & sanitizeSql(param) & "'"
    query = query.replace(placeholder, safe)
  return query

proc isValidEmail(email: string): bool =
  let parts = email.split("@")
  if parts.len != 2: return false
  if parts[0].len == 0 or parts[1].len == 0: return false
  if "." notin parts[1]: return false
  return true

proc isStrongPassword(password: string): tuple[valid: bool, reason: string] =
  if password.len < 8:
    return (false, "Password must be at least 8 characters")
  var hasUpper, hasLower, hasDigit, hasSpecial = false
  for c in password:
    if c in 'A'..'Z': hasUpper = true
    if c in 'a'..'z': hasLower = true
    if c in '0'..'9': hasDigit = true
    if c in "!@#$%^&*()-_=+[]{}|;:,.<>?": hasSpecial = true
  if not hasUpper: return (false, "Must contain uppercase letter")
  if not hasLower: return (false, "Must contain lowercase letter")
  if not hasDigit: return (false, "Must contain digit")
  if not hasSpecial: return (false, "Must contain special character")
  return (true, "")

# ============================
# Security headers
# ============================

type
  HttpResponse = object
    status: int
    headers: Table[string, string]
    body: string

proc addSecurityHeaders(resp: var HttpResponse) =
  resp.headers["X-Content-Type-Options"] = "nosniff"
  resp.headers["X-Frame-Options"] = "DENY"
  resp.headers["X-XSS-Protection"] = "1; mode=block"
  resp.headers["Strict-Transport-Security"] = "max-age=31536000; includeSubDomains"
  resp.headers["Content-Security-Policy"] = "default-src 'self'; script-src 'self'"
  resp.headers["Referrer-Policy"] = "strict-origin-when-cross-origin"
  resp.headers["Permissions-Policy"] = "camera=(), microphone=(), geolocation=()"
  resp.headers["Cache-Control"] = "no-store"

# ============================
# Audit log
# ============================

type
  AuditEvent = object
    id: string
    userId: string
    action: string
    resource: string
    resourceId: string
    ipAddress: string
    userAgent: string
    result: string   # "success" | "failure"
    details: JsonNode
    timestamp: float

var auditLog: seq[AuditEvent] = @[]
var auditCounter = 0

proc audit(userId, action, resource, resourceId: string,
           result_ = "success", details: JsonNode = newJNull(),
           ip = "", userAgent = "") =
  inc auditCounter
  auditLog.add(AuditEvent(
    id: fmt"audit_{auditCounter}",
    userId: userId,
    action: action,
    resource: resource,
    resourceId: resourceId,
    ipAddress: ip,
    userAgent: userAgent,
    result: result_,
    details: details,
    timestamp: epochTime()
  ))
  echo fmt"[Audit] {userId} {action} {resource}/{resourceId}: {result_}"

proc queryAudit(userId = "", action = "", since = 0.0): seq[AuditEvent] =
  auditLog.filterIt(
    (userId.len == 0 or it.userId == userId) and
    (action.len == 0 or it.action == action) and
    it.timestamp >= since
  )

# ============================
# Demo
# ============================

proc demo() =
  echo "=== Security Demo ==="

  # JWT
  echo "\n--- JWT ---"
  var customData: Table[string, JsonNode]
  customData["role"] = %"admin"
  customData["plan"] = %"enterprise"

  let token = makeJwt("user_1", "my-app", "api", data = customData)
  echo fmt"Token (first 50): {token[0..min(50, token.len-1)]}..."

  let (valid, claims, err) = verifyJwt(token, "secret_key_change_in_prod")
  echo fmt"Valid: {valid}"
  echo fmt"Subject: {claims.sub}"
  echo fmt"Role: {claims.data.getOrDefault(\"role\", %\"?\").getStr()}"

  # Revoke
  revokeToken(claims.jti, claims.exp)
  echo fmt"Is revoked: {isRevoked(claims.jti)}"

  # Password hashing
  echo "\n--- Password hashing ---"
  let salt = generateSalt()
  let hash = hashPassword("MyP@ss123!", salt)
  echo fmt"Hash length: {hash.len}"
  echo fmt"Verify correct: {verifyPassword(\"MyP@ss123!\", hash)}"
  echo fmt"Verify wrong: {verifyPassword(\"wrongpass\", hash)}"

  let (strongOk, _) = isStrongPassword("MyP@ss123!")
  let (weakOk, weakReason) = isStrongPassword("weak")
  echo fmt"Strong password: {strongOk}"
  echo fmt"Weak password: {weakOk} ({weakReason})"

  # CSRF
  echo "\n--- CSRF ---"
  let csrfToken = generateCsrfToken("session_abc")
  echo fmt"CSRF valid: {validateCsrfToken(csrfToken, \"session_abc\")}"
  echo fmt"CSRF wrong session: {validateCsrfToken(csrfToken, \"session_xyz\")}"

  # Sanitization
  echo "\n--- Input sanitization ---"
  let xssInput = """<script>alert('XSS')</script>"""
  echo fmt"Sanitized HTML: {sanitizeHtml(xssInput)}"

  let sqlInput = "'; DROP TABLE users; --"
  let safeQuery = sqlQuery("SELECT * FROM users WHERE email = $1", @[sqlInput])
  echo fmt"Safe query: {safeQuery}"

  # Security headers
  echo "\n--- Security headers ---"
  var resp = HttpResponse(status: 200, headers: initTable[string, string]())
  addSecurityHeaders(resp)
  echo fmt"Headers count: {resp.headers.len}"
  for k, v in resp.headers:
    echo fmt"  {k}: {v[0..min(40, v.len-1)]}"

  # Audit log
  echo "\n--- Audit log ---"
  audit("user_1", "login", "session", "sess_123", ip = "192.168.1.1")
  audit("user_1", "view", "order", "order_456")
  audit("user_2", "delete", "user", "user_99", result_ = "failure",
    details = %*{"reason": "insufficient_permissions"})

  let userAudit = queryAudit(userId = "user_1")
  echo fmt"Audit entries for user_1: {userAudit.len}"

demo()
```

---

## 📝 สรุป Part 58

| Steps | หัวข้อ |
|-------|--------|
| 841 | JWT sign/verify, token revocation, password hashing |
| 842-855 | CSRF protection, SQL injection prevention, XSS sanitization, security headers, audit log |

---

**← [Part 57: Observability](part_57_observability.md) | [Part 59: Testing Framework →](part_59_testing.md)**
