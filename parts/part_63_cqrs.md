# Part 63: CQRS & Event Sourcing
## Steps 916-930: Command Query Responsibility Segregation in Nim

---

## 🎯 เป้าหมายของ Part นี้

- Command/Query separation
- Event store (append-only log)
- Aggregate root with event replay
- Projection builders (read models)
- Snapshot strategy
- Eventual consistency patterns

---

## Step 916: Event Store & Aggregates

```nim
import tables, strformat, times, sequtils, json, strutils, options, algorithm, hashes

# ============================
# Domain events
# ============================

type
  EventType = string

  DomainEvent = object
    id: string
    aggregateId: string
    aggregateType: string
    eventType: EventType
    version: int
    payload: JsonNode
    metadata: JsonNode
    occurredAt: float

  EventStore = object
    events: seq[DomainEvent]
    counter: int

var store = EventStore(events: @[])

proc appendEvent(es: var EventStore, aggregateId, aggregateType, eventType: string,
                 payload, metadata: JsonNode, version: int): DomainEvent =
  inc es.counter
  let ev = DomainEvent(
    id: fmt"evt_{es.counter}",
    aggregateId: aggregateId,
    aggregateType: aggregateType,
    eventType: eventType,
    version: version,
    payload: payload,
    metadata: metadata,
    occurredAt: epochTime()
  )
  es.events.add(ev)
  echo fmt"[EventStore] Appended {eventType} v{version} for {aggregateId}"
  return ev

proc getEvents(es: EventStore, aggregateId: string): seq[DomainEvent] =
  es.events.filterIt(it.aggregateId == aggregateId)
    .sortedByIt(it.version)

proc getEventsSince(es: EventStore, aggregateId: string,
                    fromVersion: int): seq[DomainEvent] =
  es.events.filterIt(
    it.aggregateId == aggregateId and it.version > fromVersion
  ).sortedByIt(it.version)

proc getEventsByType(es: EventStore, eventType: string): seq[DomainEvent] =
  es.events.filterIt(it.eventType == eventType)

# ============================
# Order aggregate
# ============================

type
  OrderStatus = enum
    osPending, osConfirmed, osShipped, osDelivered, osCancelled

  OrderItem = object
    productId: string
    name: string
    quantity: int
    unitPrice: float

  Order = object
    id: string
    customerId: string
    items: seq[OrderItem]
    status: OrderStatus
    total: float
    version: int
    createdAt: float
    updatedAt: float

# Event types
const
  OrderCreated = "OrderCreated"
  OrderItemAdded = "OrderItemAdded"
  OrderConfirmed = "OrderConfirmed"
  OrderShipped = "OrderShipped"
  OrderDelivered = "OrderDelivered"
  OrderCancelled = "OrderCancelled"
  OrderItemRemoved = "OrderItemRemoved"

proc applyEvent(order: var Order, ev: DomainEvent) =
  case ev.eventType
  of OrderCreated:
    order.id = ev.payload["orderId"].getStr()
    order.customerId = ev.payload["customerId"].getStr()
    order.status = osPending
    order.items = @[]
    order.createdAt = ev.occurredAt

  of OrderItemAdded:
    order.items.add(OrderItem(
      productId: ev.payload["productId"].getStr(),
      name: ev.payload["name"].getStr(),
      quantity: ev.payload["quantity"].getInt(),
      unitPrice: ev.payload["unitPrice"].getFloat()
    ))
    order.total = order.items.foldl(
      a + float(b.quantity) * b.unitPrice, 0.0)

  of OrderItemRemoved:
    let pid = ev.payload["productId"].getStr()
    order.items = order.items.filterIt(it.productId != pid)
    order.total = order.items.foldl(
      a + float(b.quantity) * b.unitPrice, 0.0)

  of OrderConfirmed:
    order.status = osConfirmed

  of OrderShipped:
    order.status = osShipped

  of OrderDelivered:
    order.status = osDelivered

  of OrderCancelled:
    order.status = osCancelled

  else: discard

  order.version = ev.version
  order.updatedAt = ev.occurredAt

proc rehydrate(es: EventStore, orderId: string): Option[Order] =
  let events = es.getEvents(orderId)
  if events.len == 0: return none(Order)

  var order = Order()
  for ev in events:
    order.applyEvent(ev)
  return some(order)

# ============================
# Command handlers
# ============================

type
  CommandResult = tuple[ok: bool, error: string]

proc createOrder(es: var EventStore, orderId, customerId: string): CommandResult =
  let existing = es.getEvents(orderId)
  if existing.len > 0:
    return (false, "Order already exists")

  discard es.appendEvent(orderId, "Order", OrderCreated, %*{
    "orderId": orderId,
    "customerId": customerId
  }, %*{}, 1)
  return (true, "")

proc addItemToOrder(es: var EventStore, orderId, productId, name: string,
                    quantity: int, unitPrice: float): CommandResult =
  let order = rehydrate(es, orderId)
  if order.isNone: return (false, "Order not found")
  if order.get().status != osPending: return (false, "Order not in pending state")

  let existing = order.get().items.filterIt(it.productId == productId)
  if existing.len > 0: return (false, "Item already in order")

  discard es.appendEvent(orderId, "Order", OrderItemAdded, %*{
    "productId": productId,
    "name": name,
    "quantity": quantity,
    "unitPrice": unitPrice
  }, %*{}, order.get().version + 1)
  return (true, "")

proc confirmOrder(es: var EventStore, orderId: string): CommandResult =
  let order = rehydrate(es, orderId)
  if order.isNone: return (false, "Order not found")
  if order.get().status != osPending: return (false, "Order not pending")
  if order.get().items.len == 0: return (false, "Order has no items")

  discard es.appendEvent(orderId, "Order", OrderConfirmed,
    %*{"orderId": orderId}, %*{}, order.get().version + 1)
  return (true, "")

proc shipOrder(es: var EventStore, orderId, trackingNumber: string): CommandResult =
  let order = rehydrate(es, orderId)
  if order.isNone: return (false, "Order not found")
  if order.get().status != osConfirmed: return (false, "Order not confirmed")

  discard es.appendEvent(orderId, "Order", OrderShipped, %*{
    "orderId": orderId,
    "trackingNumber": trackingNumber
  }, %*{}, order.get().version + 1)
  return (true, "")

proc cancelOrder(es: var EventStore, orderId, reason: string): CommandResult =
  let order = rehydrate(es, orderId)
  if order.isNone: return (false, "Order not found")
  if order.get().status in [osShipped, osDelivered, osCancelled]:
    return (false, fmt"Cannot cancel order in {order.get().status} state")

  discard es.appendEvent(orderId, "Order", OrderCancelled, %*{
    "orderId": orderId,
    "reason": reason
  }, %*{}, order.get().version + 1)
  return (true, "")

# ============================
# Projections (read models)
# ============================

type
  OrderSummary = object
    id: string
    customerId: string
    status: string
    total: float
    itemCount: int
    createdAt: float

  CustomerOrders = object
    customerId: string
    orderCount: int
    totalSpent: float
    orders: seq[string]

  OrderProjection = object
    summaries: Table[string, OrderSummary]
    customerOrders: Table[string, CustomerOrders]
    lastProcessedVersion: Table[string, int]

var projection = OrderProjection(
  summaries: initTable[string, OrderSummary](),
  customerOrders: initTable[string, CustomerOrders]()
)

proc rebuildProjection(es: EventStore) =
  ## Full rebuild from all events
  projection = OrderProjection(
    summaries: initTable[string, OrderSummary](),
    customerOrders: initTable[string, CustomerOrders]()
  )

  for ev in es.events:
    case ev.eventType
    of OrderCreated:
      let orderId = ev.payload["orderId"].getStr()
      let customerId = ev.payload["customerId"].getStr()
      projection.summaries[orderId] = OrderSummary(
        id: orderId,
        customerId: customerId,
        status: "pending",
        total: 0.0,
        itemCount: 0,
        createdAt: ev.occurredAt
      )
      if customerId notin projection.customerOrders:
        projection.customerOrders[customerId] = CustomerOrders(
          customerId: customerId)
      projection.customerOrders[customerId].orderCount += 1
      projection.customerOrders[customerId].orders.add(orderId)

    of OrderItemAdded:
      if ev.aggregateId in projection.summaries:
        let qty = ev.payload["quantity"].getInt()
        let price = ev.payload["unitPrice"].getFloat()
        projection.summaries[ev.aggregateId].total += float(qty) * price
        inc projection.summaries[ev.aggregateId].itemCount

    of OrderConfirmed:
      if ev.aggregateId in projection.summaries:
        projection.summaries[ev.aggregateId].status = "confirmed"

    of OrderShipped:
      if ev.aggregateId in projection.summaries:
        projection.summaries[ev.aggregateId].status = "shipped"

    of OrderDelivered:
      if ev.aggregateId in projection.summaries:
        let orderId = ev.aggregateId
        let customerId = projection.summaries[orderId].customerId
        projection.summaries[orderId].status = "delivered"
        if customerId in projection.customerOrders:
          projection.customerOrders[customerId].totalSpent +=
            projection.summaries[orderId].total

    of OrderCancelled:
      if ev.aggregateId in projection.summaries:
        projection.summaries[ev.aggregateId].status = "cancelled"
        let customerId = projection.summaries[ev.aggregateId].customerId
        if customerId in projection.customerOrders:
          dec projection.customerOrders[customerId].orderCount

    else: discard

# ============================
# Snapshot strategy
# ============================

type
  Snapshot[T] = object
    aggregateId: string
    version: int
    state: T
    snapshotAt: float

var orderSnapshots: Table[string, Snapshot[Order]]

proc saveSnapshot(order: Order) =
  orderSnapshots[order.id] = Snapshot[Order](
    aggregateId: order.id,
    version: order.version,
    state: order,
    snapshotAt: epochTime()
  )
  echo fmt"[Snapshot] Saved order {order.id} at v{order.version}"

proc loadFromSnapshot(es: EventStore, orderId: string): Option[Order] =
  if orderId notin orderSnapshots: return rehydrate(es, orderId)

  let snap = orderSnapshots[orderId]
  var order = snap.state
  let events = es.getEventsSince(orderId, snap.version)
  for ev in events:
    order.applyEvent(ev)

  echo fmt"[Snapshot] Loaded order {orderId} from snapshot v{snap.version} + {events.len} events"
  return some(order)

# ============================
# Demo
# ============================

proc demo() =
  echo "=== CQRS / Event Sourcing Demo ==="

  # Create orders
  echo "\n--- Commands ---"
  discard createOrder(store, "order_1", "customer_alice")
  discard createOrder(store, "order_2", "customer_bob")

  discard addItemToOrder(store, "order_1", "prod_1", "Laptop", 1, 1299.99)
  discard addItemToOrder(store, "order_1", "prod_2", "Mouse", 2, 29.99)
  discard addItemToOrder(store, "order_2", "prod_3", "Keyboard", 1, 89.99)

  let (confirmOk, confirmErr) = confirmOrder(store, "order_1")
  echo fmt"Confirm order_1: {confirmOk} {confirmErr}"

  let (shipOk, shipErr) = shipOrder(store, "order_1", "TRACK123")
  echo fmt"Ship order_1: {shipOk} {shipErr}"

  # Try invalid transition
  let (badCancel, cancelErr) = cancelOrder(store, "order_1", "changed mind")
  echo fmt"Cancel shipped order: {badCancel} ({cancelErr})"

  # Query (rehydrate)
  echo "\n--- Query (rehydrate from events) ---"
  let order1 = rehydrate(store, "order_1")
  if order1.isSome:
    let o = order1.get()
    echo fmt"Order 1: status={o.status} items={o.items.len} total={o.total:.2f}"
    for item in o.items:
      echo fmt"  {item.name}: {item.quantity} x ${item.unitPrice:.2f}"

  # Projections
  echo "\n--- Projections ---"
  rebuildProjection(store)

  echo fmt"Summaries: {projection.summaries.len}"
  for id, s in projection.summaries:
    echo fmt"  {id}: status={s.status} total={s.total:.2f} items={s.itemCount}"

  for cid, c in projection.customerOrders:
    echo fmt"  Customer {cid}: {c.orderCount} orders, spent ${c.totalSpent:.2f}"

  # Snapshot
  echo "\n--- Snapshot ---"
  if order1.isSome:
    saveSnapshot(order1.get())

  # Add more events after snapshot
  discard appendEvent(store, "order_1", "Order", OrderDelivered,
    %*{"orderId": "order_1"}, %*{}, order1.get().version + 2)

  let fromSnap = loadFromSnapshot(store, "order_1")
  if fromSnap.isSome:
    echo fmt"Status after snapshot+events: {fromSnap.get().status}"

  echo fmt"\nTotal events in store: {store.events.len}"

demo()
```

---

## 📝 สรุป Part 63

| Steps | หัวข้อ |
|-------|--------|
| 916 | Event store, domain events, aggregate rehydration |
| 917-925 | Command handlers, validation, state transitions |
| 926-930 | Projections (read models), snapshot strategy |

---

**← [Part 62: Performance](part_62_performance.md) | [Part 64: Rate Limiting →](part_64_rate_limiting.md)**
