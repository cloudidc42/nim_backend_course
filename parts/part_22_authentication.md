# Part 22: Authentication & Authorization
## Steps 301-315: ระบบ Auth แบบ Production

---

## 🎯 เป้าหมายของ Part นี้

- JWT Authentication สมบูรณ์
- Refresh Token rotation
- Password hashing (bcrypt)
- Role-Based Access Control (RBAC)
- OAuth2 integration
- Session management

---

## Step 301: Password Hashing

```bash
# nimble install bcrypt
```

```nim
import bcrypt, std/strformat, std/times

# Hash password
proc hashPassword(password: string): string =
  let salt = genSalt(12)  # cost factor 12
  return hash(password, salt)

proc verifyPassword(password, hashedPassword: string): bool =
  return compare(password, hashedPassword)

# Test
let password = "SecurePass123!"
let hashed = hashPassword(password)

echo "Password: " & password
echo "Hashed: " & hashed[0..30] & "..."
echo "Valid: " & $verifyPassword(password, hashed)        # true
echo "Invalid: " & $verifyPassword("WrongPass", hashed)  # false

# Cost factor benchmark
proc benchmarkHash(cost: int) =
  let start = epochTime()
  let salt = genSalt(cost)
  discard hash("test", salt)
  let elapsed = epochTime() - start
  echo fmt"  Cost {cost:2}: {elapsed:.3f}s"

echo "\nBcrypt cost benchmark:"
for cost in 8..14:
  benchmarkHash(cost)
# Cost  8: ~0.05s
# Cost 10: ~0.2s
# Cost 12: ~0.8s (recommended for production)
```

---

## Step 302: JWT Complete Implementation

```nim
import std/json, std/base64, std/hmac, std/times, std/strutils,
       std/options, std/crypto/sha, std/strformat, std/tables

type
  JwtAlgorithm = enum
    HS256, HS384, HS512

  JwtClaims = object
    sub: string       # subject (user ID)
    iss: string       # issuer
    aud: string       # audience
    exp: int64        # expiration
    nbf: int64        # not before
    iat: int64        # issued at
    jti: string       # JWT ID
    # Custom claims
    role: string
    email: string

  JwtToken = object
    header: string
    payload: string
    signature: string
    raw: string

  JwtConfig = object
    secret: string
    algorithm: JwtAlgorithm
    issuer: string
    audience: string
    accessTtl: int    # seconds
    refreshTtl: int   # seconds

# Token blacklist (in production: use Redis)
var blacklistedTokens: Table[string, float] = initTable[string, float]()

proc base64UrlEncode2(s: string): string =
  encode(s).replace("=", "").replace("+", "-").replace("/", "_")

proc base64UrlDecode2(s: string): string =
  var padded = s.replace("-", "+").replace("_", "/")
  while padded.len mod 4 != 0: padded.add('=')
  decode(padded)

proc signHmac(data, secret: string, alg: JwtAlgorithm): string =
  case alg
  of HS256: base64UrlEncode2(hmac_sha256(secret, data))
  of HS384: base64UrlEncode2(hmac_sha384(secret, data))
  of HS512: base64UrlEncode2(hmac_sha512(secret, data))

proc generateJti(): string =
  base64UrlEncode2($epochTime() & $rand(99999))

proc createToken(claims: JwtClaims, config: JwtConfig): string =
  let header = base64UrlEncode2($ %*{
    "alg": $config.algorithm,
    "typ": "JWT"
  })
  
  let payload = base64UrlEncode2($ %*{
    "sub": claims.sub,
    "iss": if claims.iss.len > 0: claims.iss else: config.issuer,
    "aud": if claims.aud.len > 0: claims.aud else: config.audience,
    "exp": claims.exp,
    "nbf": claims.nbf,
    "iat": claims.iat,
    "jti": claims.jti,
    "role": claims.role,
    "email": claims.email
  })
  
  let signingInput = header & "." & payload
  let signature = signHmac(signingInput, config.secret, config.algorithm)
  
  return signingInput & "." & signature

proc verifyToken(token: string, config: JwtConfig): Option[JwtClaims] =
  let parts = token.split('.')
  if parts.len != 3:
    return none(JwtClaims)
  
  # Verify signature
  let signingInput = parts[0] & "." & parts[1]
  let expectedSig = signHmac(signingInput, config.secret, config.algorithm)
  
  if parts[2] != expectedSig:
    return none(JwtClaims)  # invalid signature
  
  let payload = try: parseJson(base64UrlDecode2(parts[1]))
  except: return none(JwtClaims)
  
  let now = int64(epochTime())
  
  # Check expiration
  let exp = payload{"exp"}.getInt(0)
  if exp > 0 and int64(exp) < now:
    return none(JwtClaims)  # expired
  
  # Check not-before
  let nbf = payload{"nbf"}.getInt(0)
  if nbf > 0 and int64(nbf) > now:
    return none(JwtClaims)  # not valid yet
  
  # Check blacklist
  let jti = payload{"jti"}.getStr("")
  if jti in blacklistedTokens:
    return none(JwtClaims)  # revoked
  
  return some(JwtClaims(
    sub:   payload{"sub"}.getStr(),
    iss:   payload{"iss"}.getStr(),
    aud:   payload{"aud"}.getStr(),
    exp:   int64(payload{"exp"}.getInt()),
    iat:   int64(payload{"iat"}.getInt()),
    jti:   jti,
    role:  payload{"role"}.getStr("user"),
    email: payload{"email"}.getStr()
  ))

proc revokeToken(token: string) =
  let parts = token.split('.')
  if parts.len != 3: return
  
  let payload = try: parseJson(base64UrlDecode2(parts[1]))
  except: return
  
  let jti = payload{"jti"}.getStr("")
  let exp = epochTime() + float(payload{"exp"}.getInt(0))
  
  if jti.len > 0:
    blacklistedTokens[jti] = exp

# Token pair (access + refresh)
type TokenPair2 = object
  accessToken: string
  refreshToken: string
  tokenType: string
  expiresIn: int

proc generateTokenPair(
  userId, email, role: string,
  config: JwtConfig
): TokenPair2 =
  let now = int64(epochTime())
  
  let accessClaims = JwtClaims(
    sub: userId,
    email: email,
    role: role,
    exp: now + int64(config.accessTtl),
    iat: now,
    nbf: now,
    jti: "acc_" & generateJti()
  )
  
  let refreshClaims = JwtClaims(
    sub: userId,
    email: email,
    role: role,
    exp: now + int64(config.refreshTtl),
    iat: now,
    nbf: now,
    jti: "ref_" & generateJti()
  )
  
  TokenPair2(
    accessToken: createToken(accessClaims, config),
    refreshToken: createToken(refreshClaims, JwtConfig(
      secret: config.secret & "_refresh",
      algorithm: config.algorithm,
      issuer: config.issuer,
      audience: config.audience
    )),
    tokenType: "Bearer",
    expiresIn: config.accessTtl
  )

# Test
let jwtConfig = JwtConfig(
  secret: "my-super-secret-key-at-least-32-chars-long",
  algorithm: HS256,
  issuer: "myapp",
  audience: "myapp-api",
  accessTtl: 3600,    # 1 hour
  refreshTtl: 2592000 # 30 days
)

let tokens = generateTokenPair("user123", "alice@example.com", "admin", jwtConfig)
echo "Access: " & tokens.accessToken[0..60] & "..."

let claims = verifyToken(tokens.accessToken, jwtConfig)
if claims.isSome:
  let c = claims.get()
  echo fmt"Valid! User: {c.sub}, Role: {c.role}, Email: {c.email}"

# Revoke
revokeToken(tokens.accessToken)
let claimsAfterRevoke = verifyToken(tokens.accessToken, jwtConfig)
echo fmt"After revoke: {claimsAfterRevoke.isSome}"  # false
```

---

## Step 303: RBAC - Role-Based Access Control

```nim
import std/tables, std/sequtils, std/strformat, std/sets

type
  Permission = enum
    pReadUsers, pWriteUsers, pDeleteUsers,
    pReadPosts, pWritePosts, pDeletePosts,
    pReadAdminPanel, pManageSystem

  Role = enum
    rGuest, rUser, rModerator, rAdmin, rSuperAdmin

# Define permissions per role
const RolePermissions: array[Role, set[Permission]] = [
  # Guest
  {pReadPosts},
  # User
  {pReadPosts, pWritePosts},
  # Moderator
  {pReadPosts, pWritePosts, pDeletePosts, pReadUsers},
  # Admin
  {pReadPosts, pWritePosts, pDeletePosts, pReadUsers, pWriteUsers, pReadAdminPanel},
  # SuperAdmin
  {pReadPosts, pWritePosts, pDeletePosts, pReadUsers, pWriteUsers, pDeleteUsers, pReadAdminPanel, pManageSystem},
]

proc hasPermission(role: Role, permission: Permission): bool =
  permission in RolePermissions[role]

proc hasAnyPermission(role: Role, perms: set[Permission]): bool =
  (RolePermissions[role] * perms).len > 0

proc hasAllPermissions(role: Role, perms: set[Permission]): bool =
  perms <= RolePermissions[role]

# Resource ownership
type
  OwnedResource = object
    id: int
    ownerId: string

proc canAccess(userId, resourceOwnerId: string, role: Role, perm: Permission): bool =
  # Super admin can do anything
  if role == rSuperAdmin:
    return true
  
  # Owner can access their own resource
  if userId == resourceOwnerId:
    return true
  
  # Check role permission
  return hasPermission(role, perm)

# Middleware factory
proc requirePermission(permission: Permission): proc() =
  return proc() =
    # Would check JWT here and extract role
    # then verify permission
    discard

# Test
echo "=== RBAC Test ==="
for role in Role:
  let perms = RolePermissions[role]
  echo fmt"\n{role}:"
  echo fmt"  Can read posts:    {hasPermission(role, pReadPosts)}"
  echo fmt"  Can delete posts:  {hasPermission(role, pDeletePosts)}"
  echo fmt"  Can manage system: {hasPermission(role, pManageSystem)}"
```

---

## Step 304-315: Complete Auth System

```nim
# auth_system.nim - Complete authentication system

import std/json, std/tables, std/options, std/times, std/strformat,
       std/strutils, std/sequtils

# ==============================
# User Model
# ==============================

type
  UserId2 = distinct string  # UUID-like

  UserRecord2 = object
    id: UserId2
    name: string
    email: string
    passwordHash: string
    role: string
    isActive: bool
    isEmailVerified: bool
    loginAttempts: int
    lockedUntil: float
    lastLoginAt: string
    createdAt: string

  AuthResult = object
    success: bool
    message: string
    user: Option[UserRecord2]
    accessToken: string
    refreshToken: string
    expiresIn: int

# ==============================
# User Store (in production: database)
# ==============================

var userStore: Table[string, UserRecord2] = initTable[string, UserRecord2]()
var emailIndex: Table[string, string] = initTable[string, string]()

proc createUser2(name, email, password: string, role: string = "user"): UserRecord2 =
  let id = "user_" & $int(epochTime()) & "_" & $rand(9999)
  let user = UserRecord2(
    id: UserId2(id),
    name: name,
    email: email.toLower(),
    passwordHash: "bcrypt_" & password,  # Use real bcrypt in production
    role: role,
    isActive: true,
    isEmailVerified: false,
    loginAttempts: 0,
    lockedUntil: 0.0,
    createdAt: now().format("yyyy-MM-dd HH:mm:ss")
  )
  userStore[id] = user
  emailIndex[email.toLower()] = id
  return user

proc findByEmail2(email: string): Option[UserRecord2] =
  let id = emailIndex.getOrDefault(email.toLower(), "")
  if id.len > 0 and id in userStore:
    return some(userStore[id])
  return none(UserRecord2)

# Account lockout (10 attempts = 30 min lockout)
const MAX_LOGIN_ATTEMPTS = 10
const LOCKOUT_DURATION = 30 * 60  # 30 minutes

proc isLocked(user: UserRecord2): bool =
  user.loginAttempts >= MAX_LOGIN_ATTEMPTS and
  epochTime() < user.lockedUntil

proc recordFailedLogin(id: string) =
  if id in userStore:
    userStore[id].loginAttempts += 1
    if userStore[id].loginAttempts >= MAX_LOGIN_ATTEMPTS:
      userStore[id].lockedUntil = epochTime() + LOCKOUT_DURATION

proc recordSuccessfulLogin(id: string) =
  if id in userStore:
    userStore[id].loginAttempts = 0
    userStore[id].lockedUntil = 0.0
    userStore[id].lastLoginAt = now().format("yyyy-MM-dd HH:mm:ss")

# ==============================
# Auth Service
# ==============================

proc login(email, password: string): AuthResult =
  let userOpt = findByEmail2(email)
  
  if userOpt.isNone:
    # Don't reveal if email exists
    return AuthResult(success: false, message: "Invalid email or password")
  
  var user = userOpt.get()
  
  # Check if locked
  if user.isLocked():
    let remaining = int(user.lockedUntil - epochTime())
    return AuthResult(success: false,
      message: fmt"Account locked. Try again in {remaining div 60} minutes")
  
  # Verify password (simplified - use real bcrypt)
  let validPassword = user.passwordHash == "bcrypt_" & password
  
  if not validPassword:
    recordFailedLogin(string(user.id))
    let attempts = userStore[string(user.id)].loginAttempts
    let remaining = MAX_LOGIN_ATTEMPTS - attempts
    
    if remaining <= 3:
      return AuthResult(success: false,
        message: fmt"Invalid password. {remaining} attempt(s) remaining")
    
    return AuthResult(success: false, message: "Invalid email or password")
  
  if not user.isActive:
    return AuthResult(success: false, message: "Account is deactivated")
  
  recordSuccessfulLogin(string(user.id))
  
  # Generate tokens (simplified)
  let now2 = epochTime()
  let accessToken = "access_" & string(user.id) & "_" & $int(now2)
  let refreshToken = "refresh_" & string(user.id) & "_" & $int(now2)
  
  return AuthResult(
    success: true,
    message: "Login successful",
    user: some(user),
    accessToken: accessToken,
    refreshToken: refreshToken,
    expiresIn: 3600
  )

proc logout(accessToken: string): bool =
  # Revoke token (add to blacklist)
  echo fmt"Revoking token: {accessToken[0..20]}..."
  return true

proc changePassword(userId, currentPassword, newPassword: string): bool =
  if userId notin userStore:
    return false
  
  let user = userStore[userId]
  
  # Verify current password
  if user.passwordHash != "bcrypt_" & currentPassword:
    return false
  
  # Update password
  if newPassword.len < 8:
    return false
  
  userStore[userId].passwordHash = "bcrypt_" & newPassword
  return true

# ==============================
# Demo
# ==============================

proc main() =
  echo "=== Authentication System Demo ==="
  
  # Create users
  let alice = createUser2("Alice Smith", "alice@example.com", "Password123!", "admin")
  let bob   = createUser2("Bob Jones", "bob@example.com", "MyPass456@")
  
  echo fmt"\nCreated users: {alice.name}, {bob.name}"
  
  # Successful login
  echo "\n--- Login Tests ---"
  let result1 = login("alice@example.com", "Password123!")
  echo fmt"Alice login: {result1.success} - {result1.message}"
  if result1.success:
    echo fmt"  Token: {result1.accessToken[0..25]}..."
  
  # Wrong password
  let result2 = login("alice@example.com", "WrongPassword")
  echo fmt"Wrong pass: {result2.success} - {result2.message}"
  
  # Non-existent user
  let result3 = login("nobody@example.com", "password")
  echo fmt"Unknown user: {result3.success} - {result3.message}"
  
  # Test lockout
  echo "\n--- Lockout Test ---"
  for i in 1..12:
    let r = login("bob@example.com", "wrong" & $i)
    if not r.success:
      echo fmt"  Attempt {i:2}: {r.message}"
      if "locked" in r.message or "remaining" in r.message.toLower():
        break
  
  # Change password
  echo "\n--- Password Change ---"
  let changed = changePassword(string(alice.id), "Password123!", "NewSecurePass789!")
  echo fmt"Password changed: {changed}"
  
  let result4 = login("alice@example.com", "NewSecurePass789!")
  echo fmt"Login with new pass: {result4.success}"

main()
```

---

## 📝 สรุป Part 22

| Steps | หัวข้อ |
|-------|--------|
| 301 | Password hashing (bcrypt) |
| 302 | JWT complete implementation |
| 303 | RBAC - Role-Based Access Control |
| 304-315 | Complete auth system (login, logout, lockout, RBAC) |

---

**← [Part 21: Jester](part_21_jester_framework.md) | [Part 23: WebSockets Advanced →](part_23_websockets_advanced.md)**
