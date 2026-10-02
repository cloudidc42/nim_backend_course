# Part 43: Email System
## Steps 616-630: Email Service ระดับ Production

---

## 🎯 เป้าหมายของ Part นี้

- SMTP client
- Email templates (HTML + text)
- Queue-based sending
- Bounce/unsubscribe handling
- Email verification tokens
- Bulk email with throttling

---

## Step 616: SMTP Client

```nim
import asyncdispatch, asyncnet, strutils, strformat, base64, times, tables

# ============================
# SMTP Client
# ============================

type
  SmtpConfig = object
    host: string
    port: int
    useTls: bool
    username: string
    password: string
    fromEmail: string
    fromName: string

  EmailAddress = object
    email: string
    name: string

  EmailAttachment = object
    filename: string
    contentType: string
    data: string      # base64 encoded

  Email = object
    to: seq[EmailAddress]
    cc: seq[EmailAddress]
    bcc: seq[EmailAddress]
    replyTo: string
    subject: string
    textBody: string
    htmlBody: string
    attachments: seq[EmailAttachment]
    headers: Table[string, string]

  SmtpClient = object
    config: SmtpConfig

proc newSmtpClient(config: SmtpConfig): SmtpClient =
  SmtpClient(config: config)

proc formatAddress(addr: EmailAddress): string =
  if addr.name.len > 0:
    fmt"{addr.name} <{addr.email}>"
  else:
    addr.email

proc generateMessageId(domain: string): string =
  fmt"<{int(epochTime() * 1000)}.{domain}>"

proc buildMimeMessage(client: SmtpClient, email: Email): string =
  let boundary = fmt"----=_Part_{int(epochTime() * 1000)}"
  let msgId = generateMessageId("example.com")
  let dateStr = now().format("ddd, dd MMM yyyy HH:mm:ss")
  
  var msg = ""
  
  # Headers
  msg &= fmt"Message-ID: {msgId}\r\n"
  msg &= fmt"Date: {dateStr} +0000\r\n"
  msg &= fmt"From: {formatAddress(EmailAddress(name: client.config.fromName, email: client.config.fromEmail))}\r\n"
  msg &= "To: " & email.to.mapIt(formatAddress(it)).join(", ") & "\r\n"
  
  if email.cc.len > 0:
    msg &= "Cc: " & email.cc.mapIt(formatAddress(it)).join(", ") & "\r\n"
  
  msg &= fmt"Subject: {email.subject}\r\n"
  msg &= "MIME-Version: 1.0\r\n"
  
  # Custom headers
  for k, v in email.headers:
    msg &= fmt"{k}: {v}\r\n"
  
  # Body
  let hasHtml = email.htmlBody.len > 0
  let hasText = email.textBody.len > 0
  let hasAttachments = email.attachments.len > 0
  
  if hasAttachments:
    msg &= fmt"Content-Type: multipart/mixed; boundary=\"{boundary}\"\r\n"
    msg &= "\r\n"
    msg &= fmt"--{boundary}\r\n"
  
  if hasHtml and hasText:
    let altBoundary = boundary & "_alt"
    msg &= fmt"Content-Type: multipart/alternative; boundary=\"{altBoundary}\"\r\n"
    msg &= "\r\n"
    msg &= fmt"--{altBoundary}\r\n"
    msg &= "Content-Type: text/plain; charset=UTF-8\r\n"
    msg &= "Content-Transfer-Encoding: quoted-printable\r\n"
    msg &= "\r\n"
    msg &= email.textBody & "\r\n"
    msg &= fmt"--{altBoundary}\r\n"
    msg &= "Content-Type: text/html; charset=UTF-8\r\n"
    msg &= "Content-Transfer-Encoding: quoted-printable\r\n"
    msg &= "\r\n"
    msg &= email.htmlBody & "\r\n"
    msg &= fmt"--{altBoundary}--\r\n"
  elif hasHtml:
    msg &= "Content-Type: text/html; charset=UTF-8\r\n"
    msg &= "Content-Transfer-Encoding: quoted-printable\r\n"
    msg &= "\r\n"
    msg &= email.htmlBody & "\r\n"
  else:
    msg &= "Content-Type: text/plain; charset=UTF-8\r\n"
    msg &= "\r\n"
    msg &= email.textBody & "\r\n"
  
  # Attachments
  for att in email.attachments:
    msg &= fmt"--{boundary}\r\n"
    msg &= fmt"Content-Type: {att.contentType}\r\n"
    msg &= "Content-Transfer-Encoding: base64\r\n"
    msg &= fmt"Content-Disposition: attachment; filename=\"{att.filename}\"\r\n"
    msg &= "\r\n"
    msg &= att.data & "\r\n"
  
  if hasAttachments:
    msg &= fmt"--{boundary}--\r\n"
  
  return msg

proc sendEmail(client: SmtpClient, email: Email) =
  ## Mock send - in production: use asyncnet + SMTP protocol
  echo fmt"[SMTP] Sending to: {email.to.mapIt(it.email).join(\", \")}"
  echo fmt"[SMTP] Subject: {email.subject}"
  
  let msg = client.buildMimeMessage(email)
  echo fmt"[SMTP] Message size: {msg.len} bytes"
  echo "[SMTP] ✓ Sent successfully"

# ============================
# Demo
# ============================

proc demo() =
  echo "=== SMTP Client Demo ==="
  
  let config = SmtpConfig(
    host: "smtp.example.com",
    port: 587,
    useTls: true,
    username: "noreply@example.com",
    password: "secret",
    fromEmail: "noreply@example.com",
    fromName: "My App"
  )
  
  let client = newSmtpClient(config)
  
  let email = Email(
    to: @[EmailAddress(email: "alice@example.com", name: "Alice")],
    subject: "Welcome to My App!",
    textBody: "Hi Alice,\n\nWelcome to My App!\n\nBest,\nThe Team",
    htmlBody: "<h1>Welcome, Alice!</h1><p>We're glad you're here.</p>",
    headers: {"X-Mailer": "MyApp v1.0"}.toTable()
  )
  
  client.sendEmail(email)

demo()
```

---

## Step 617: Email Templates

```nim
import strutils, tables, strformat, json

# ============================
# Template engine (simple mustache-like)
# ============================

type
  TemplateContext = Table[string, string]

proc renderTemplate(tmpl: string, ctx: TemplateContext): string =
  var result = tmpl
  
  for key, value in ctx:
    result = result.replace("{{" & key & "}}", value)
    result = result.replace("{{ " & key & " }}", value)
  
  # Handle conditionals: {{#if key}}...{{/if}}
  var output = result
  var pos = 0
  while pos < output.len:
    let ifStart = output.find("{{#if ", pos)
    if ifStart < 0: break
    
    let tagEnd = output.find("}}", ifStart)
    if tagEnd < 0: break
    
    let varName = output[ifStart + 6 .. tagEnd - 1].strip()
    let endTag = "{{/if}}"
    let bodyEnd = output.find(endTag, tagEnd)
    if bodyEnd < 0: break
    
    let body = output[tagEnd + 2 .. bodyEnd - 1]
    let replacement = if ctx.getOrDefault(varName, "") != "": body else: ""
    
    output = output[0..<ifStart] & replacement & output[bodyEnd + endTag.len .. ^1]
    pos = ifStart + replacement.len
  
  return output

# ============================
# Email templates
# ============================

const welcomeHtml = """
<!DOCTYPE html>
<html>
<head>
<style>
  body { font-family: Arial, sans-serif; max-width: 600px; margin: 0 auto; }
  .header { background: #2563eb; color: white; padding: 24px; text-align: center; }
  .body { padding: 24px; }
  .button { background: #2563eb; color: white; padding: 12px 24px;
            text-decoration: none; border-radius: 4px; display: inline-block; }
  .footer { color: #9ca3af; font-size: 12px; text-align: center; padding: 16px; }
</style>
</head>
<body>
<div class="header">
  <h1>Welcome to {{app_name}}!</h1>
</div>
<div class="body">
  <p>Hi {{user_name}},</p>
  <p>Thank you for signing up. We're excited to have you on board!</p>
  {{#if confirm_url}}
  <p>Please confirm your email address:</p>
  <p><a href="{{confirm_url}}" class="button">Confirm Email</a></p>
  {{/if}}
  <p>If you have any questions, reply to this email.</p>
  <p>Best,<br>The {{app_name}} Team</p>
</div>
<div class="footer">
  <p>{{app_name}} · {{app_address}}</p>
  <p><a href="{{unsubscribe_url}}">Unsubscribe</a></p>
</div>
</body>
</html>
"""

const passwordResetHtml = """
<!DOCTYPE html>
<html>
<head>
<style>
  body { font-family: Arial, sans-serif; max-width: 600px; margin: 0 auto; }
  .header { background: #dc2626; color: white; padding: 24px; text-align: center; }
  .body { padding: 24px; }
  .code { background: #f3f4f6; padding: 16px; font-size: 24px; letter-spacing: 8px;
          text-align: center; font-family: monospace; border-radius: 4px; }
  .button { background: #dc2626; color: white; padding: 12px 24px;
            text-decoration: none; border-radius: 4px; display: inline-block; }
</style>
</head>
<body>
<div class="header">
  <h1>Reset Your Password</h1>
</div>
<div class="body">
  <p>Hi {{user_name}},</p>
  <p>We received a request to reset your password.</p>
  {{#if reset_code}}
  <p>Your reset code:</p>
  <div class="code">{{reset_code}}</div>
  {{/if}}
  {{#if reset_url}}
  <p>Or click the link below:</p>
  <p><a href="{{reset_url}}" class="button">Reset Password</a></p>
  {{/if}}
  <p>This link expires in <strong>{{expires_in}}</strong>.</p>
  <p>If you didn't request this, ignore this email.</p>
</div>
</body>
</html>
"""

const orderConfirmHtml = """
<!DOCTYPE html>
<html>
<head>
<style>
  body { font-family: Arial, sans-serif; max-width: 600px; margin: 0 auto; }
  .header { background: #16a34a; color: white; padding: 24px; text-align: center; }
  .body { padding: 24px; }
  table { width: 100%; border-collapse: collapse; }
  td, th { border: 1px solid #e5e7eb; padding: 8px; text-align: left; }
  th { background: #f9fafb; }
  .total { font-weight: bold; font-size: 18px; }
</style>
</head>
<body>
<div class="header">
  <h1>Order Confirmed!</h1>
  <p>Order #{{order_id}}</p>
</div>
<div class="body">
  <p>Hi {{user_name}},</p>
  <p>Your order has been confirmed.</p>
  <table>
    <tr><th>Item</th><th>Qty</th><th>Price</th></tr>
    {{order_items}}
    <tr><td colspan="2" class="total">Total</td><td class="total">{{order_total}}</td></tr>
  </table>
  <p>Estimated delivery: <strong>{{delivery_date}}</strong></p>
</div>
</body>
</html>
"""

type
  EmailTemplate = object
    name: string
    subject: string
    htmlTemplate: string
    textTemplate: string

  TemplateRegistry = object
    templates: Table[string, EmailTemplate]

proc newTemplateRegistry(): TemplateRegistry =
  TemplateRegistry(templates: initTable[string, EmailTemplate]())

proc register(reg: var TemplateRegistry, tmpl: EmailTemplate) =
  reg.templates[tmpl.name] = tmpl

proc render(reg: TemplateRegistry, name: string, ctx: TemplateContext): tuple[subject, html, text: string] =
  if name notin reg.templates:
    raise newException(ValueError, fmt"Template not found: {name}")
  
  let tmpl = reg.templates[name]
  return (
    renderTemplate(tmpl.subject, ctx),
    renderTemplate(tmpl.htmlTemplate, ctx),
    renderTemplate(tmpl.textTemplate, ctx)
  )

# ============================
# Demo
# ============================

proc demo() =
  echo "=== Email Templates Demo ==="
  
  var registry = newTemplateRegistry()
  
  registry.register(EmailTemplate(
    name: "welcome",
    subject: "Welcome to {{app_name}}!",
    htmlTemplate: welcomeHtml,
    textTemplate: "Hi {{user_name}},\n\nWelcome to {{app_name}}!\n\nConfirm: {{confirm_url}}"
  ))
  
  registry.register(EmailTemplate(
    name: "password_reset",
    subject: "Reset your {{app_name}} password",
    htmlTemplate: passwordResetHtml,
    textTemplate: "Hi {{user_name}},\n\nReset code: {{reset_code}}\n\nExpires: {{expires_in}}"
  ))
  
  registry.register(EmailTemplate(
    name: "order_confirm",
    subject: "Order #{{order_id}} confirmed",
    htmlTemplate: orderConfirmHtml,
    textTemplate: "Order #{{order_id}} confirmed. Total: {{order_total}}"
  ))
  
  echo "\n--- Welcome email ---"
  let welcomeCtx: TemplateContext = {
    "app_name": "MyApp",
    "user_name": "Alice",
    "confirm_url": "https://myapp.com/confirm/token123",
    "app_address": "123 Main St",
    "unsubscribe_url": "https://myapp.com/unsubscribe/token456"
  }.toTable()
  
  let (subject, html, _) = registry.render("welcome", welcomeCtx)
  echo fmt"Subject: {subject}"
  echo fmt"HTML length: {html.len} bytes"
  
  echo "\n--- Password reset ---"
  let resetCtx: TemplateContext = {
    "app_name": "MyApp",
    "user_name": "Bob",
    "reset_code": "847291",
    "reset_url": "https://myapp.com/reset/tokenXYZ",
    "expires_in": "30 minutes"
  }.toTable()
  
  let (resetSubject, _, resetText) = registry.render("password_reset", resetCtx)
  echo fmt"Subject: {resetSubject}"
  echo fmt"Text:\n{resetText}"

demo()
```

---

## Step 618-630: Email Queue and Verification

```nim
import asyncdispatch, strformat, times, tables, sequtils, random, strutils

# ============================
# Email queue
# ============================

type
  EmailJob = object
    id: string
    to: string
    subject: string
    html: string
    text: string
    templateName: string
    context: Table[string, string]
    attempts: int
    maxAttempts: int
    scheduledAt: float
    status: string  # pending | sent | failed

  EmailQueue = object
    pending: seq[EmailJob]
    sent: seq[EmailJob]
    failed: seq[EmailJob]
    rateLimitPerSec: float
    lastSentAt: float

var emailQueue = EmailQueue(
  pending: @[], sent: @[], failed: @[],
  rateLimitPerSec: 10.0, lastSentAt: 0.0
)

proc queueEmail(job: EmailJob) =
  var j = job
  if j.id.len == 0:
    j.id = fmt"email_{int(epochTime() * 1000) mod 1_000_000}"
  j.scheduledAt = epochTime()
  j.status = "pending"
  j.maxAttempts = 3
  emailQueue.pending.add(j)
  echo fmt"[EmailQueue] Queued: {j.id} to {j.to}"

proc sendEmailMock(job: EmailJob): bool =
  # Simulate occasional failures (10% chance)
  randomize()
  return rand(9) != 0

proc processQueue() {.async.} =
  while emailQueue.pending.len > 0:
    let now = epochTime()
    
    # Rate limiting
    let timeSinceLast = now - emailQueue.lastSentAt
    let minInterval = 1.0 / emailQueue.rateLimitPerSec
    
    if timeSinceLast < minInterval:
      await sleepAsync(int((minInterval - timeSinceLast) * 1000))
    
    let job = emailQueue.pending[0]
    emailQueue.pending.delete(0)
    
    echo fmt"[EmailQueue] Processing: {job.id}"
    
    let success = sendEmailMock(job)
    emailQueue.lastSentAt = epochTime()
    
    if success:
      var sentJob = job
      sentJob.status = "sent"
      emailQueue.sent.add(sentJob)
      echo fmt"[EmailQueue] ✓ Sent: {job.id}"
    else:
      var failedJob = job
      inc failedJob.attempts
      
      if failedJob.attempts < failedJob.maxAttempts:
        # Retry with backoff
        failedJob.status = "pending"
        failedJob.scheduledAt = epochTime() + float(2 ^ failedJob.attempts) * 10.0
        emailQueue.pending.add(failedJob)
        echo fmt"[EmailQueue] ↩ Retry {failedJob.attempts}/{failedJob.maxAttempts}: {job.id}"
      else:
        failedJob.status = "failed"
        emailQueue.failed.add(failedJob)
        echo fmt"[EmailQueue] ✗ Failed: {job.id}"

# ============================
# Email verification tokens
# ============================

type
  VerificationToken = object
    token: string
    userId: int
    email: string
    type_: string    # "email_verify" | "password_reset" | "magic_link"
    createdAt: float
    expiresAt: float
    usedAt: float    # 0 = not used

var tokenStore: Table[string, VerificationToken] = initTable[string, VerificationToken]()

proc generateToken(): string =
  randomize()
  var chars = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789"
  result = ""
  for _ in 0..<48:
    result &= chars[rand(chars.len - 1)]

proc createVerificationToken(userId: int, email, type_: string,
                              ttlMinutes: int = 30): VerificationToken =
  let token = generateToken()
  let now = epochTime()
  
  # Invalidate previous tokens of same type for this user
  for k, v in tokenStore.mpairs:
    if v.userId == userId and v.type_ == type_ and v.usedAt == 0:
      v.usedAt = now  # mark as invalidated
  
  let vt = VerificationToken(
    token: token,
    userId: userId,
    email: email,
    type_: type_,
    createdAt: now,
    expiresAt: now + float(ttlMinutes * 60),
    usedAt: 0.0
  )
  
  tokenStore[token] = vt
  return vt

proc validateToken(token, expectedType: string): tuple[valid: bool, vt: VerificationToken] =
  if token notin tokenStore:
    return (false, VerificationToken())
  
  let vt = tokenStore[token]
  
  if vt.type_ != expectedType:
    return (false, vt)
  
  if vt.usedAt > 0:
    return (false, vt)  # already used
  
  if epochTime() > vt.expiresAt:
    return (false, vt)  # expired
  
  return (true, vt)

proc useToken(token: string) =
  if token in tokenStore:
    tokenStore[token].usedAt = epochTime()

# ============================
# Unsubscribe management
# ============================

type
  UnsubscribeType = enum
    utAll, utMarketing, utTransactional, utNewsletter

  UnsubscribeEntry = object
    email: string
    type_: UnsubscribeType
    timestamp: float
    token: string

var unsubscribeList: Table[string, seq[UnsubscribeEntry]] = initTable[string, seq[UnsubscribeEntry]]()

proc generateUnsubToken(email: string): string =
  # In production: HMAC-signed token
  encode(email & ":" & $int(epochTime()))

proc unsubscribe(email: string, type_ = utAll) =
  if email notin unsubscribeList:
    unsubscribeList[email] = @[]
  unsubscribeList[email].add(UnsubscribeEntry(
    email: email, type_: type_,
    timestamp: epochTime(),
    token: generateUnsubToken(email)
  ))
  echo fmt"[Unsub] {email} unsubscribed from {type_}"

proc isUnsubscribed(email: string, type_ = utMarketing): bool =
  if email notin unsubscribeList: return false
  for entry in unsubscribeList[email]:
    if entry.type_ == utAll or entry.type_ == type_:
      return true
  return false

# ============================
# Demo
# ============================

proc demo() {.async.} =
  echo "=== Email System Demo ==="
  
  # Queue some emails
  for i in 0..<5:
    queueEmail(EmailJob(
      to: fmt"user{i}@example.com",
      subject: fmt"Hello User {i}",
      html: fmt"<p>Hello User {i}</p>",
      text: fmt"Hello User {i}"
    ))
  
  echo "\n--- Processing queue ---"
  # Process just a few
  for _ in 0..<3:
    if emailQueue.pending.len > 0:
      let job = emailQueue.pending[0]
      emailQueue.pending.delete(0)
      let success = sendEmailMock(job)
      if success:
        emailQueue.sent.add(job)
        echo fmt"✓ Sent: {job.to}"
      else:
        emailQueue.failed.add(job)
        echo fmt"✗ Failed: {job.to}"
  
  echo fmt"\nQueue stats: pending={emailQueue.pending.len} sent={emailQueue.sent.len} failed={emailQueue.failed.len}"
  
  echo "\n--- Email verification tokens ---"
  randomize()
  let vt = createVerificationToken(1, "alice@example.com", "email_verify", ttlMinutes = 30)
  echo fmt"Token: {vt.token[0..15]}..."
  echo fmt"Expires in: 30 minutes"
  
  let (valid, _) = validateToken(vt.token, "email_verify")
  echo fmt"Token valid: {valid}"
  
  useToken(vt.token)
  let (valid2, _) = validateToken(vt.token, "email_verify")
  echo fmt"Token valid after use: {valid2}"
  
  echo "\n--- Unsubscribe management ---"
  unsubscribe("bob@example.com", utMarketing)
  echo fmt"bob unsubscribed from marketing: {isUnsubscribed(\"bob@example.com\", utMarketing)}"
  echo fmt"bob unsubscribed from transactional: {isUnsubscribed(\"bob@example.com\", utTransactional)}"

waitFor demo()
```

---

## 📝 สรุป Part 43

| Steps | หัวข้อ |
|-------|--------|
| 616 | SMTP client, MIME message builder |
| 617 | Email template engine with conditionals |
| 618-630 | Email queue, verification tokens, unsubscribe |

---

**← [Part 42: Search](part_42_search.md) | [Part 44: Payment System →](part_44_payments.md)**
