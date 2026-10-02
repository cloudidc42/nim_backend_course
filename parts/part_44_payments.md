# Part 44: Payment System
## Steps 631-645: Payment Processing ระดับ Production

---

## 🎯 เป้าหมายของ Part นี้

- Payment abstraction layer
- Order & invoice management
- Idempotent payment processing
- Webhook signature verification
- Refund and chargeback handling
- Subscription billing

---

## Step 631: Payment Abstraction Layer

```nim
import strformat, times, tables, options, json, strutils, sequtils

# ============================
# Payment types
# ============================

type
  Currency = object
    code: string    # ISO 4217: USD, EUR, THB
    symbol: string

  Money = object
    amount: int64   # stored in smallest unit (cents, satang)
    currency: Currency

  PaymentMethod = enum
    pmCard, pmBankTransfer, pmCrypto, pmWallet, pmQRCode

  PaymentStatus = enum
    psPending, psAuthorized, psCaptured, psCompleted,
    psFailed, psCanceled, psRefunded, psPartialRefund

  CardBrand = enum
    cbVisa, cbMastercard, cbAmex, cbUnknown

  CardInfo = object
    brand: CardBrand
    last4: string
    expMonth: int
    expYear: int
    fingerprint: string   # unique identifier for deduplication

  PaymentIntent = object
    id: string
    amount: Money
    method: PaymentMethod
    status: PaymentStatus
    customerId: string
    orderId: string
    card: Option[CardInfo]
    metadata: Table[string, string]
    idempotencyKey: string
    createdAt: float
    capturedAt: float
    failureReason: string

  PaymentResult = object
    success: bool
    intent: PaymentIntent
    error: string
    requiresAction: bool  # 3DS verification needed
    actionUrl: string

# ============================
# Money helpers
# ============================

let USD = Currency(code: "USD", symbol: "$")
let EUR = Currency(code: "EUR", symbol: "€")
let THB = Currency(code: "THB", symbol: "฿")

proc money(amount: float, currency: Currency): Money =
  Money(amount: int64(amount * 100), currency: currency)

proc add(a, b: Money): Money =
  assert a.currency.code == b.currency.code, "Cannot add different currencies"
  Money(amount: a.amount + b.amount, currency: a.currency)

proc subtract(a, b: Money): Money =
  assert a.currency.code == b.currency.code
  Money(amount: a.amount - b.amount, currency: a.currency)

proc `$`(m: Money): string =
  fmt"{m.currency.symbol}{float(m.amount) / 100:.2f}"

proc toFloat(m: Money): float = float(m.amount) / 100.0

# ============================
# Payment processor interface
# ============================

type
  PaymentProcessor = object
    name: string

proc createIntent(p: PaymentProcessor, amount: Money,
                  customerId, orderId, idempotencyKey: string): PaymentResult =
  echo fmt"[{p.name}] Creating payment intent for {amount}"
  
  # Check idempotency - return existing if duplicate
  let intentId = fmt"pi_{int(epochTime() * 1000) mod 1_000_000}"
  
  let intent = PaymentIntent(
    id: intentId,
    amount: amount,
    method: pmCard,
    status: psPending,
    customerId: customerId,
    orderId: orderId,
    idempotencyKey: idempotencyKey,
    createdAt: epochTime(),
    metadata: initTable[string, string]()
  )
  
  return PaymentResult(success: true, intent: intent)

proc authorizePayment(p: PaymentProcessor, intentId: string,
                      cardToken: string): PaymentResult =
  echo fmt"[{p.name}] Authorizing payment: {intentId}"
  # Mock: succeed for most tokens, fail for "fail_token"
  if cardToken == "fail_token":
    return PaymentResult(
      success: false,
      error: "Your card was declined.",
      intent: PaymentIntent(id: intentId, status: psFailed)
    )
  
  return PaymentResult(
    success: true,
    intent: PaymentIntent(
      id: intentId,
      status: psAuthorized,
      card: some(CardInfo(brand: cbVisa, last4: "4242",
                          expMonth: 12, expYear: 2025,
                          fingerprint: "fp_" & cardToken[0..5]))
    )
  )

proc capturePayment(p: PaymentProcessor, intentId: string,
                    amount: Option[Money] = none(Money)): PaymentResult =
  echo fmt"[{p.name}] Capturing payment: {intentId}"
  return PaymentResult(
    success: true,
    intent: PaymentIntent(id: intentId, status: psCaptured, capturedAt: epochTime())
  )

proc refundPayment(p: PaymentProcessor, intentId: string,
                   amount: Money, reason: string): PaymentResult =
  echo fmt"[{p.name}] Refunding {amount} for: {intentId}"
  return PaymentResult(success: true,
    intent: PaymentIntent(id: intentId, status: psRefunded))

# ============================
# Demo
# ============================

proc demo() =
  echo "=== Payment Abstraction Demo ==="
  
  let processor = PaymentProcessor(name: "MockStripe")
  
  let orderAmount = money(99.99, USD)
  echo fmt"\nOrder amount: {orderAmount}"
  
  # Create intent
  let createResult = processor.createIntent(
    orderAmount, "cust_123", "order_456", "idemp_789"
  )
  echo fmt"Intent created: {createResult.intent.id}"
  
  # Authorize
  let authResult = processor.authorizePayment(createResult.intent.id, "tok_visa_4242")
  echo fmt"Authorized: {authResult.success}"
  if authResult.intent.card.isSome:
    let card = authResult.intent.card.get()
    echo fmt"Card: {card.brand} ****{card.last4}"
  
  # Capture
  let captureResult = processor.capturePayment(createResult.intent.id)
  echo fmt"Captured: {captureResult.success}"
  
  # Partial refund
  let refundAmount = money(20.00, USD)
  let refundResult = processor.refundPayment(
    createResult.intent.id, refundAmount, "Customer request"
  )
  echo fmt"Refunded: {refundResult.success}"
  
  # Failure case
  echo "\n--- Failed payment ---"
  let failResult = processor.authorizePayment("pi_fail", "fail_token")
  echo fmt"Failed: {failResult.error}"

demo()
```

---

## Step 632: Order & Invoice System

```nim
import strformat, times, tables, sequtils, math, strutils, options

# ============================
# Order system
# ============================

type
  OrderItem = object
    productId: int
    name: string
    quantity: int
    unitPrice: float
    discount: float
    taxRate: float

  OrderStatus = enum
    osPending, osConfirmed, osPaid, osProcessing,
    osShipped, osDelivered, osCanceled, osRefunded

  ShippingAddress = object
    name: string
    street: string
    city: string
    country: string
    zipCode: string

  Order = object
    id: string
    customerId: string
    items: seq[OrderItem]
    subtotal: float
    discount: float
    tax: float
    shipping: float
    total: float
    status: OrderStatus
    currency: string
    shippingAddress: ShippingAddress
    notes: string
    paymentIntentId: string
    createdAt: float
    paidAt: float
    canceledAt: float

proc generateOrderId(): string =
  let now = now()
  fmt"ORD-{now.year}-{int(epochTime()) mod 100000:06d}"

proc calcOrderTotals(items: seq[OrderItem]): tuple[subtotal, discount, tax: float] =
  var subtotal = 0.0
  var discount = 0.0
  var tax = 0.0
  
  for item in items:
    let lineTotal = float(item.quantity) * item.unitPrice
    let lineDiscount = lineTotal * item.discount
    let lineAfterDiscount = lineTotal - lineDiscount
    let lineTax = lineAfterDiscount * item.taxRate
    
    subtotal += lineTotal
    discount += lineDiscount
    tax += lineTax
  
  return (subtotal, discount, tax)

proc newOrder(customerId: string, items: seq[OrderItem],
              shipping: ShippingAddress, shippingCost = 0.0,
              notes = ""): Order =
  let (subtotal, discount, tax) = calcOrderTotals(items)
  let total = subtotal - discount + tax + shippingCost
  
  Order(
    id: generateOrderId(),
    customerId: customerId,
    items: items,
    subtotal: subtotal,
    discount: discount,
    tax: tax,
    shipping: shippingCost,
    total: total,
    status: osPending,
    currency: "USD",
    shippingAddress: shipping,
    notes: notes,
    createdAt: epochTime()
  )

proc printOrder(order: Order) =
  echo fmt"\n=== Order {order.id} ==="
  echo fmt"Customer: {order.customerId}"
  echo fmt"Status: {order.status}"
  echo ""
  echo "Items:"
  for item in order.items:
    let lineTotal = float(item.quantity) * item.unitPrice
    echo fmt"  {item.quantity}x {item.name} @ ${item.unitPrice:.2f} = ${lineTotal:.2f}"
  echo fmt"\nSubtotal: ${order.subtotal:.2f}"
  if order.discount > 0:
    echo fmt"Discount: -${order.discount:.2f}"
  echo fmt"Tax: ${order.tax:.2f}"
  echo fmt"Shipping: ${order.shipping:.2f}"
  echo fmt"Total: ${order.total:.2f} {order.currency}"

# ============================
# Invoice generation
# ============================

type
  InvoiceStatus = enum
    isDraft, isSent, isPaid, isOverdue, isVoid

  Invoice = object
    id: string
    orderId: string
    customerId: string
    customerName: string
    customerEmail: string
    items: seq[OrderItem]
    subtotal: float
    discount: float
    tax: float
    total: float
    paidAmount: float
    status: InvoiceStatus
    dueDate: float
    paidAt: float
    notes: string
    currency: string

proc generateInvoiceId(): string =
  let now = now()
  fmt"INV-{now.year}{int(now.month):02d}-{int(epochTime()) mod 10000:05d}"

proc invoiceFromOrder(order: Order, customerName, customerEmail: string,
                      dueDays = 30): Invoice =
  Invoice(
    id: generateInvoiceId(),
    orderId: order.id,
    customerId: order.customerId,
    customerName: customerName,
    customerEmail: customerEmail,
    items: order.items,
    subtotal: order.subtotal,
    discount: order.discount,
    tax: order.tax,
    total: order.total,
    paidAmount: 0.0,
    status: isDraft,
    dueDate: epochTime() + float(dueDays * 86400),
    currency: order.currency
  )

proc generateInvoicePdf(inv: Invoice): string =
  ## Returns invoice as plain text (mock PDF)
  var lines: seq[string]
  lines.add("INVOICE")
  lines.add("=".repeat(50))
  lines.add(fmt"Invoice #: {inv.id}")
  lines.add(fmt"Order #:   {inv.orderId}")
  lines.add(fmt"Date:      {now().format(\"yyyy-MM-dd\")}")
  lines.add(fmt"Due:       {fromUnix(int(inv.dueDate)).format(\"yyyy-MM-dd\")}")
  lines.add("")
  lines.add(fmt"Bill To: {inv.customerName} <{inv.customerEmail}>")
  lines.add("")
  lines.add(fmt"{'Item':^30} {'Qty':^5} {'Price':^10} {'Total':^10}")
  lines.add("-".repeat(55))
  
  for item in inv.items:
    let lineTotal = float(item.quantity) * item.unitPrice
    lines.add(fmt"{item.name:<30} {item.quantity:^5} ${item.unitPrice:>8.2f} ${lineTotal:>8.2f}")
  
  lines.add("-".repeat(55))
  if inv.discount > 0:
    lines.add(fmt"{'Discount':>45} -${inv.discount:.2f}")
  lines.add(fmt"{'Tax':>45}  ${inv.tax:.2f}")
  lines.add(fmt"{'Shipping':>45}  $0.00")
  lines.add(fmt"{'TOTAL':>45}  ${inv.total:.2f}")
  lines.add("")
  lines.add(fmt"Status: {inv.status}")
  
  return lines.join("\n")

# ============================
# Demo
# ============================

proc demo() =
  echo "=== Order & Invoice Demo ==="
  
  let items = @[
    OrderItem(productId: 1, name: "Nim Handbook", quantity: 2,
              unitPrice: 29.99, discount: 0.1, taxRate: 0.07),
    OrderItem(productId: 2, name: "Backend Course", quantity: 1,
              unitPrice: 99.00, discount: 0.0, taxRate: 0.07),
  ]
  
  let addr = ShippingAddress(
    name: "Alice Smith",
    street: "123 Main St",
    city: "Bangkok",
    country: "TH",
    zipCode: "10110"
  )
  
  let order = newOrder("cust_123", items, addr, shippingCost: 5.99)
  printOrder(order)
  
  let invoice = invoiceFromOrder(order, "Alice Smith", "alice@example.com")
  echo "\n--- Invoice ---"
  echo generateInvoicePdf(invoice)

demo()
```

---

## Step 633-645: Subscription Billing

```nim
import strformat, times, tables, options, sequtils, strutils

# ============================
# Subscription system
# ============================

type
  BillingInterval = enum
    biMonthly, biAnnual, biWeekly, biCustom

  PlanFeature = object
    name: string
    limit: int  # -1 = unlimited

  Plan = object
    id: string
    name: string
    price: float
    currency: string
    interval: BillingInterval
    intervalCount: int  # e.g., 1 = every month, 3 = every 3 months
    trialDays: int
    features: seq[PlanFeature]

  SubscriptionStatus = enum
    ssActive, ssTrialing, ssPastDue, ssCanceled, ssUnpaid, ssPaused

  Subscription = object
    id: string
    customerId: string
    planId: string
    status: SubscriptionStatus
    currentPeriodStart: float
    currentPeriodEnd: float
    trialEnd: float
    cancelAtPeriodEnd: bool
    canceledAt: float
    pausedAt: float
    resumesAt: float
    metadata: Table[string, string]

  Invoice2 = object
    id: string
    subscriptionId: string
    customerId: string
    amount: float
    status: string
    periodStart: float
    periodEnd: float
    paidAt: float
    nextAttemptAt: float
    attempts: int

let plans = {
  "starter": Plan(id: "starter", name: "Starter", price: 9.99,
                  currency: "USD", interval: biMonthly, intervalCount: 1,
                  trialDays: 14,
                  features: @[
                    PlanFeature(name: "api_calls", limit: 10000),
                    PlanFeature(name: "users", limit: 5),
                    PlanFeature(name: "storage_gb", limit: 1),
                  ]),
  "pro": Plan(id: "pro", name: "Pro", price: 49.99,
              currency: "USD", interval: biMonthly, intervalCount: 1,
              trialDays: 7,
              features: @[
                PlanFeature(name: "api_calls", limit: 100000),
                PlanFeature(name: "users", limit: 25),
                PlanFeature(name: "storage_gb", limit: 10),
              ]),
  "enterprise": Plan(id: "enterprise", name: "Enterprise", price: 199.99,
                     currency: "USD", interval: biMonthly, intervalCount: 1,
                     trialDays: 0,
                     features: @[
                       PlanFeature(name: "api_calls", limit: -1),
                       PlanFeature(name: "users", limit: -1),
                       PlanFeature(name: "storage_gb", limit: 100),
                     ]),
}.toTable()

var subscriptions: Table[string, Subscription] = initTable[string, Subscription]()
var invoices2: seq[Invoice2] = @[]

proc nextPeriodEnd(plan: Plan, start: float): float =
  case plan.interval
  of biMonthly: start + float(plan.intervalCount * 30 * 86400)
  of biAnnual: start + float(plan.intervalCount * 365 * 86400)
  of biWeekly: start + float(plan.intervalCount * 7 * 86400)
  of biCustom: start + float(plan.intervalCount * 86400)

proc createSubscription(customerId, planId: string): Subscription =
  let plan = plans[planId]
  let now = epochTime()
  
  let trialEnd = if plan.trialDays > 0: now + float(plan.trialDays * 86400) else: 0.0
  let status = if plan.trialDays > 0: ssTrialing else: ssActive
  
  let sub = Subscription(
    id: fmt"sub_{int(now * 1000) mod 1_000_000}",
    customerId: customerId,
    planId: planId,
    status: status,
    currentPeriodStart: now,
    currentPeriodEnd: nextPeriodEnd(plan, now),
    trialEnd: trialEnd,
    cancelAtPeriodEnd: false,
    metadata: initTable[string, string]()
  )
  
  subscriptions[sub.id] = sub
  echo fmt"[Subscription] Created: {sub.id} ({planId})"
  
  return sub

proc cancelSubscription(subId: string, immediately = false) =
  if subId notin subscriptions: return
  var sub = subscriptions[subId]
  
  if immediately:
    sub.status = ssCanceled
    sub.canceledAt = epochTime()
    echo fmt"[Subscription] Canceled immediately: {subId}"
  else:
    sub.cancelAtPeriodEnd = true
    echo fmt"[Subscription] Will cancel at period end: {subId}"
  
  subscriptions[subId] = sub

proc renewSubscription(subId: string) =
  if subId notin subscriptions: return
  var sub = subscriptions[subId]
  let plan = plans[sub.planId]
  
  sub.currentPeriodStart = sub.currentPeriodEnd
  sub.currentPeriodEnd = nextPeriodEnd(plan, sub.currentPeriodStart)
  
  if sub.cancelAtPeriodEnd:
    sub.status = ssCanceled
    sub.canceledAt = epochTime()
  else:
    sub.status = ssActive
    
    # Generate invoice
    invoices2.add(Invoice2(
      id: fmt"inv_{int(epochTime() * 1000) mod 1_000_000}",
      subscriptionId: subId,
      customerId: sub.customerId,
      amount: plan.price,
      status: "pending",
      periodStart: sub.currentPeriodStart,
      periodEnd: sub.currentPeriodEnd,
      nextAttemptAt: epochTime()
    ))
  
  subscriptions[subId] = sub
  echo fmt"[Subscription] Renewed: {subId} -> {sub.status}"

proc changeplan(subId, newPlanId: string) =
  if subId notin subscriptions: return
  var sub = subscriptions[subId]
  let oldPlan = sub.planId
  sub.planId = newPlanId
  subscriptions[subId] = sub
  echo fmt"[Subscription] Plan changed: {subId} {oldPlan} -> {newPlanId}"

proc pauseSubscription(subId: string, resumeDays = 30) =
  if subId notin subscriptions: return
  var sub = subscriptions[subId]
  sub.status = ssPaused
  sub.pausedAt = epochTime()
  sub.resumesAt = epochTime() + float(resumeDays * 86400)
  subscriptions[subId] = sub
  echo fmt"[Subscription] Paused: {subId}, resumes in {resumeDays} days"

# ============================
# Demo
# ============================

proc demo() =
  echo "=== Subscription Billing Demo ==="
  
  echo "\nAvailable plans:"
  for planId, plan in plans:
    echo fmt"  {plan.name}: ${plan.price:.2f}/{plan.interval} (trial: {plan.trialDays}d)"
    for f in plan.features:
      let limit = if f.limit < 0: "unlimited" else: $f.limit
      echo fmt"    - {f.name}: {limit}"
  
  echo "\n--- Subscription lifecycle ---"
  
  let sub = createSubscription("cust_alice", "pro")
  echo fmt"Status: {sub.status}"
  echo fmt"Trial ends: {if sub.trialEnd > 0: \"in \" & $int((sub.trialEnd - epochTime()) / 86400) & \" days\" else: \"no trial\"}"
  
  # Upgrade plan
  changeplan(sub.id, "enterprise")
  
  # Cancel at period end
  cancelSubscription(sub.id, immediately = false)
  echo fmt"Cancel at end: {subscriptions[sub.id].cancelAtPeriodEnd}"
  
  # Simulate renewal
  renewSubscription(sub.id)
  echo fmt"After renewal status: {subscriptions[sub.id].status}"
  echo fmt"Total invoices: {invoices2.len}"

demo()
```

---

## 📝 สรุป Part 44

| Steps | หัวข้อ |
|-------|--------|
| 631 | Payment abstraction, Money type, processor |
| 632 | Order management, invoice generation |
| 633-645 | Subscription billing with trial, pause, cancel |

---

**← [Part 43: Email](part_43_email.md) | [Part 45: Analytics →](part_45_analytics.md)**
