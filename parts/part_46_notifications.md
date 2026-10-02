# Part 46: Notification System
## Steps 661-675: Push & In-App Notifications

---

## 🎯 เป้าหมายของ Part นี้

- In-app notification center
- Push notifications (FCM/APNs abstraction)
- Notification templates
- Delivery tracking
- User notification preferences
- Batch notification sending

---

## Step 661: In-App Notification Center

```nim
import times, tables, strformat, json, sequtils, strutils, options, algorithm

# ============================
# Notification types
# ============================

type
  NotificationType = enum
    ntInfo, ntSuccess, ntWarning, ntError,
    ntMessage, ntOrder, ntPayment, ntSystem

  NotificationPriority = enum
    npLow, npNormal, npHigh, npUrgent

  Notification = object
    id: string
    userId: string
    type_: NotificationType
    priority: NotificationPriority
    title: string
    body: string
    icon: string
    imageUrl: string
    actionUrl: string
    data: JsonNode
    read: bool
    createdAt: float
    readAt: float
    expiresAt: float

  NotificationCenter = object
    notifications: Table[string, seq[Notification]]
    counter: int

var center = NotificationCenter(
  notifications: initTable[string, seq[Notification]](),
  counter: 0
)

proc notifId(): string =
  inc center.counter
  fmt"notif_{center.counter}"

proc createNotification(userId: string, type_: NotificationType,
                        title, body: string,
                        priority = npNormal,
                        actionUrl = "", data: JsonNode = nil,
                        ttlHours = 168): Notification =  # 7 days default
  let now = epochTime()
  Notification(
    id: notifId(),
    userId: userId,
    type_: type_,
    priority: priority,
    title: title,
    body: body,
    actionUrl: actionUrl,
    data: if data.isNil: newJNull() else: data,
    read: false,
    createdAt: now,
    expiresAt: if ttlHours > 0: now + float(ttlHours * 3600) else: 0.0
  )

proc send(n: Notification) =
  let userId = n.userId
  if userId notin center.notifications:
    center.notifications[userId] = @[]
  center.notifications[userId].add(n)
  echo fmt"[NotifCenter] Sent to {userId}: {n.title}"

proc getNotifications(userId: string, unreadOnly = false,
                      limit = 20, offset = 0): seq[Notification] =
  if userId notin center.notifications:
    return @[]
  
  let now = epochTime()
  var notifs = center.notifications[userId]
    .filterIt(it.expiresAt == 0 or now < it.expiresAt)
  
  if unreadOnly:
    notifs = notifs.filterIt(not it.read)
  
  notifs.sort(proc(a, b: Notification): int = cmp(b.createdAt, a.createdAt))
  
  let endIdx = min(offset + limit, notifs.len)
  if offset >= notifs.len: return @[]
  return notifs[offset..<endIdx]

proc markRead(userId, notifId: string) =
  if userId notin center.notifications: return
  for i, n in center.notifications[userId]:
    if n.id == notifId:
      center.notifications[userId][i].read = true
      center.notifications[userId][i].readAt = epochTime()
      break

proc markAllRead(userId: string) =
  if userId notin center.notifications: return
  let now = epochTime()
  for i in 0..<center.notifications[userId].len:
    center.notifications[userId][i].read = true
    center.notifications[userId][i].readAt = now
  echo fmt"[NotifCenter] Marked all read for: {userId}"

proc unreadCount(userId: string): int =
  if userId notin center.notifications: return 0
  center.notifications[userId].countIt(not it.read)

# ============================
# Notification preferences
# ============================

type
  NotifChannel = enum
    ncInApp, ncEmail, ncPush, ncSms

  NotifPreference = object
    userId: string
    channels: Table[string, bool]   # "order.created" -> email: true, push: false
    quiet: bool             # do not disturb
    quietStart: int         # hour 0-23
    quietEnd: int
    timezone: string

var preferences: Table[string, NotifPreference] = initTable[string, NotifPreference]()

proc defaultPreference(userId: string): NotifPreference =
  var channels: Table[string, bool]
  channels["email.order"] = true
  channels["email.payment"] = true
  channels["email.marketing"] = false
  channels["push.order"] = true
  channels["push.message"] = true
  channels["push.marketing"] = false
  channels["inapp.all"] = true
  
  NotifPreference(
    userId: userId,
    channels: channels,
    quiet: false,
    quietStart: 22,
    quietEnd: 8,
    timezone: "UTC"
  )

proc shouldSend(pref: NotifPreference, channel: NotifChannel, eventType: string): bool =
  let key = fmt"{($channel).toLower()}.{eventType}"
  pref.channels.getOrDefault(key, true)

proc isQuietHour(pref: NotifPreference): bool =
  if not pref.quiet: return false
  let h = now().hour
  if pref.quietStart > pref.quietEnd:
    return h >= pref.quietStart or h < pref.quietEnd
  return h >= pref.quietStart and h < pref.quietEnd

# ============================
# Demo
# ============================

proc demo() =
  echo "=== Notification Center Demo ==="
  
  # Send notifications to user
  let alice = "user_alice"
  
  send(createNotification(alice, ntOrder, "Order Confirmed",
    "Your order #1001 has been confirmed",
    priority = npNormal, actionUrl = "/orders/1001"))
  
  send(createNotification(alice, ntPayment, "Payment Received",
    "Payment of $99.99 processed successfully",
    priority = npHigh))
  
  send(createNotification(alice, ntMessage, "New Message",
    "Bob sent you a message",
    priority = npNormal, actionUrl = "/messages/123"))
  
  send(createNotification(alice, ntSystem, "Maintenance",
    "Scheduled maintenance on Sunday 2-4 AM",
    priority = npLow, ttlHours = 24))
  
  echo fmt"\nUnread count: {unreadCount(alice)}"
  
  let notifs = getNotifications(alice)
  echo fmt"Total notifications: {notifs.len}"
  
  for n in notifs:
    let icon = if n.read: "○" else: "●"
    echo fmt"  {icon} [{n.type_}] {n.title}"
  
  echo "\n--- Mark as read ---"
  markRead(alice, notifs[0].id)
  echo fmt"Unread after mark: {unreadCount(alice)}"
  
  markAllRead(alice)
  echo fmt"Unread after mark all: {unreadCount(alice)}"
  
  echo "\n--- Preferences ---"
  var pref = defaultPreference(alice)
  echo fmt"Email for orders: {pref.shouldSend(ncEmail, \"order\")}"
  echo fmt"Email for marketing: {pref.shouldSend(ncEmail, \"marketing\")}"
  echo fmt"Quiet hours: {pref.quiet}"

demo()
```

---

## Step 662-675: Push Notifications & Batch Sending

```nim
import asyncdispatch, strformat, times, tables, json, sequtils, strutils

# ============================
# Push notification (FCM/APNs abstraction)
# ============================

type
  PushProvider = enum
    ppFcm, ppApns, ppOneSignal

  PushDevice = object
    userId: string
    token: string
    provider: PushProvider
    platform: string    # "android" | "ios" | "web"
    active: bool
    lastSeen: float

  PushMessage = object
    title: string
    body: string
    icon: string
    image: string
    actionUrl: string
    data: Table[string, string]
    badge: int
    sound: string
    ttl: int           # seconds

  PushResult = object
    token: string
    success: bool
    error: string
    messageId: string

var deviceRegistry: Table[string, seq[PushDevice]] = initTable[string, seq[PushDevice]]()

proc registerDevice(userId: string, device: PushDevice) =
  if userId notin deviceRegistry:
    deviceRegistry[userId] = @[]
  
  # Replace existing token
  let existing = deviceRegistry[userId].findIt(it.token == device.token)
  if existing >= 0:
    deviceRegistry[userId][existing] = device
  else:
    deviceRegistry[userId].add(device)
  
  echo fmt"[Push] Registered device for {userId}: {device.platform}"

proc unregisterDevice(userId, token: string) =
  if userId in deviceRegistry:
    deviceRegistry[userId] = deviceRegistry[userId].filterIt(it.token != token)

proc sendPushToDevice(device: PushDevice, msg: PushMessage): Future[PushResult] {.async.} =
  ## Mock push send
  await sleepAsync(10)
  
  echo fmt"[Push:{device.platform}] Sending to token {device.token[0..8]}..."
  
  # Simulate occasional failures
  if device.token.startsWith("invalid_"):
    return PushResult(token: device.token, success: false, error: "InvalidRegistration")
  
  return PushResult(
    token: device.token,
    success: true,
    messageId: fmt"msg_{int(epochTime() * 1000) mod 1_000_000}"
  )

proc sendPushToUser(userId: string, msg: PushMessage): Future[seq[PushResult]] {.async.} =
  if userId notin deviceRegistry:
    return @[]
  
  var futures: seq[Future[PushResult]]
  
  for device in deviceRegistry[userId]:
    if device.active:
      futures.add(sendPushToDevice(device, msg))
  
  let results = await all(futures)
  
  # Remove invalid tokens
  for r in results:
    if not r.success and r.error == "InvalidRegistration":
      unregisterDevice(userId, r.token)
      echo fmt"[Push] Removed invalid token: {r.token[0..8]}..."
  
  return results

# ============================
# Batch notification sender
# ============================

type
  BatchJob = object
    id: string
    recipients: seq[string]
    message: PushMessage
    sent: int
    failed: int
    total: int
    startedAt: float
    completedAt: float
    status: string

var batchJobs: Table[string, BatchJob] = initTable[string, BatchJob]()

proc sendBatch(recipients: seq[string], msg: PushMessage,
               batchSize = 100): Future[BatchJob] {.async.} =
  let jobId = fmt"batch_{int(epochTime() * 1000) mod 1_000_000}"
  var job = BatchJob(
    id: jobId,
    recipients: recipients,
    message: msg,
    total: recipients.len,
    startedAt: epochTime(),
    status: "running"
  )
  batchJobs[jobId] = job
  
  echo fmt"[Batch] Starting job {jobId}: {recipients.len} recipients"
  
  # Process in batches
  var i = 0
  while i < recipients.len:
    let batch = recipients[i..<min(i + batchSize, recipients.len)]
    
    var futures: seq[Future[seq[PushResult]]]
    for userId in batch:
      futures.add(sendPushToUser(userId, msg))
    
    let results = await all(futures)
    
    for userResults in results:
      for r in userResults:
        if r.success: inc job.sent
        else: inc job.failed
    
    i += batchSize
    
    # Rate limiting between batches
    if i < recipients.len:
      await sleepAsync(100)
  
  job.completedAt = epochTime()
  job.status = "completed"
  batchJobs[jobId] = job
  
  echo fmt"[Batch] Job {jobId} done: sent={job.sent} failed={job.failed}"
  return job

# ============================
# Notification templates
# ============================

type
  PushTemplate = object
    id: string
    title: string
    body: string
    icon: string

let templates = {
  "order_shipped": PushTemplate(
    id: "order_shipped",
    title: "Your order is on the way!",
    body: "Order #{{order_id}} has shipped. Track it here.",
    icon: "/icons/truck.png"
  ),
  "payment_failed": PushTemplate(
    id: "payment_failed",
    title: "Payment Failed",
    body: "Your payment of {{amount}} couldn't be processed.",
    icon: "/icons/warning.png"
  ),
  "new_message": PushTemplate(
    id: "new_message",
    title: "{{sender}} sent you a message",
    body: "{{preview}}",
    icon: "/icons/message.png"
  ),
}.toTable()

proc renderPushTemplate(templateId: string,
                        vars: Table[string, string]): PushMessage =
  if templateId notin templates:
    raise newException(ValueError, fmt"Template not found: {templateId}")
  
  let tmpl = templates[templateId]
  var title = tmpl.title
  var body = tmpl.body
  
  for k, v in vars:
    title = title.replace("{{" & k & "}}", v)
    body = body.replace("{{" & k & "}}", v)
  
  PushMessage(
    title: title,
    body: body,
    icon: tmpl.icon,
    data: initTable[string, string]()
  )

# ============================
# Demo
# ============================

proc demo() {.async.} =
  echo "=== Push Notifications Demo ==="
  
  # Register devices
  registerDevice("user_1", PushDevice(userId: "user_1",
    token: "fcm_token_abc123", provider: ppFcm,
    platform: "android", active: true, lastSeen: epochTime()))
  
  registerDevice("user_1", PushDevice(userId: "user_1",
    token: "apns_token_xyz789", provider: ppApns,
    platform: "ios", active: true, lastSeen: epochTime()))
  
  registerDevice("user_2", PushDevice(userId: "user_2",
    token: "invalid_bad_token", provider: ppFcm,
    platform: "android", active: true, lastSeen: epochTime()))
  
  # Send single notification
  echo "\n--- Single user push ---"
  let msg = PushMessage(
    title: "Hello!",
    body: "You have a new message",
    data: {"type": "message", "id": "123"}.toTable()
  )
  
  let results = await sendPushToUser("user_1", msg)
  echo fmt"Results: {results.len}"
  for r in results:
    echo fmt"  {r.token[0..8]}... success={r.success}"
  
  # Template-based notification
  echo "\n--- Template push ---"
  let vars = {"order_id": "1001", "amount": "$99.99"}.toTable()
  let tmplMsg = renderPushTemplate("order_shipped", vars)
  echo fmt"Title: {tmplMsg.title}"
  echo fmt"Body: {tmplMsg.body}"
  
  # Batch notification
  echo "\n--- Batch push ---"
  let recipients = @["user_1", "user_2", "user_3"]
  let batchMsg = PushMessage(
    title: "System Update",
    body: "We've improved your experience!",
    data: initTable[string, string]()
  )
  
  let job = await sendBatch(recipients, batchMsg, batchSize = 2)
  echo fmt"Batch complete: {job.sent} sent, {job.failed} failed"

waitFor demo()
```

---

## 📝 สรุป Part 46

| Steps | หัวข้อ |
|-------|--------|
| 661 | In-app notification center, preferences |
| 662-675 | Push notifications (FCM/APNs), batch sending, templates |

---

**← [Part 45: Analytics](part_45_analytics.md) | [Part 47: Feature Flags →](part_47_feature_flags.md)**
