# Part 35: Advanced Design Patterns
## Steps 496-510: Design Patterns ระดับ World-Class

---

## 🎯 เป้าหมายของ Part นี้

- Repository + Unit of Work pattern
- CQRS (Command Query Responsibility Segregation)
- Domain-Driven Design (DDD) concepts
- Event Sourcing basics
- Hexagonal Architecture
- Clean Architecture in Nim

---

## Step 496: Repository Pattern (Production)

```nim
import asyncdispatch, options, sequtils, tables, strformat, times, json

# ============================
# Domain Model
# ============================

type
  UserId = distinct int
  Email = distinct string

  UserStatus = enum
    Active, Suspended, Deleted

  User = object
    id: UserId
    name: string
    email: Email
    status: UserStatus
    createdAt: float
    updatedAt: float

  # Filter/query object
  UserFilter = object
    status: Option[UserStatus]
    searchTerm: Option[string]
    limit: int
    offset: int

  PageResult[T] = object
    items: seq[T]
    total: int
    page: int
    limit: int

# ============================
# Repository Interface
# ============================

type
  IUserRepository = ref object of RootObj

method findById(repo: IUserRepository, id: UserId): Future[Option[User]] {.base, async.} =
  return none(User)

method findAll(repo: IUserRepository, filter: UserFilter): Future[PageResult[User]] {.base, async.} =
  return PageResult[User]()

method findByEmail(repo: IUserRepository, email: Email): Future[Option[User]] {.base, async.} =
  return none(User)

method save(repo: IUserRepository, user: User): Future[User] {.base, async.} =
  return user

method delete(repo: IUserRepository, id: UserId): Future[bool] {.base, async.} =
  return false

method count(repo: IUserRepository, filter: UserFilter): Future[int] {.base, async.} =
  return 0

# ============================
# In-Memory Implementation (for testing)
# ============================

type
  InMemoryUserRepo = ref object of IUserRepository
    store: Table[int, User]
    nextId: int

proc newInMemoryUserRepo(): InMemoryUserRepo =
  InMemoryUserRepo(store: initTable[int, User](), nextId: 1)

method findById(repo: InMemoryUserRepo, id: UserId): Future[Option[User]] {.async.} =
  let key = int(id)
  if key in repo.store:
    return some(repo.store[key])
  return none(User)

method findByEmail(repo: InMemoryUserRepo, email: Email): Future[Option[User]] {.async.} =
  for _, user in repo.store:
    if string(user.email) == string(email):
      return some(user)
  return none(User)

method save(repo: InMemoryUserRepo, user: User): Future[User] {.async.} =
  var u = user
  if int(u.id) == 0:
    u.id = UserId(repo.nextId)
    inc repo.nextId
    u.createdAt = epochTime()
  u.updatedAt = epochTime()
  repo.store[int(u.id)] = u
  return u

method delete(repo: InMemoryUserRepo, id: UserId): Future[bool] {.async.} =
  let key = int(id)
  if key in repo.store:
    repo.store[int(id)].status = Deleted
    return true
  return false

method findAll(repo: InMemoryUserRepo, filter: UserFilter): Future[PageResult[User]] {.async.} =
  var all = repo.store.values.toSeq()
  
  if filter.status.isSome:
    all = all.filterIt(it.status == filter.status.get())
  
  if filter.searchTerm.isSome:
    let term = filter.searchTerm.get().toLowerAscii()
    all = all.filterIt(it.name.toLowerAscii().contains(term))
  
  let total = all.len
  let start = min(filter.offset, total)
  let finish = min(filter.offset + filter.limit, total)
  
  return PageResult[User](
    items: all[start ..< finish],
    total: total,
    page: filter.offset div filter.limit + 1,
    limit: filter.limit
  )

# ============================
# Service Layer (uses repository)
# ============================

type
  UserService = object
    users: IUserRepository

proc newUserService(repo: IUserRepository): UserService =
  UserService(users: repo)

proc createUser(svc: UserService, name, email: string): Future[User] {.async.} =
  # Check if email already exists
  let existing = await svc.users.findByEmail(Email(email))
  if existing.isSome:
    raise newException(ValueError, "Email already registered")
  
  let user = User(
    id: UserId(0),
    name: name,
    email: Email(email),
    status: Active
  )
  return await svc.users.save(user)

proc suspendUser(svc: UserService, id: int): Future[bool] {.async.} =
  let userOpt = await svc.users.findById(UserId(id))
  if userOpt.isNone:
    return false
  
  var user = userOpt.get()
  user.status = Suspended
  discard await svc.users.save(user)
  return true

# Demo
proc repoDemo() {.async.} =
  echo "=== Repository Pattern Demo ==="
  
  let repo = newInMemoryUserRepo()
  let svc = newUserService(repo)
  
  let alice = await svc.createUser("Alice Smith", "alice@example.com")
  echo fmt"Created: {alice.name} (id={int(alice.id)})"
  
  let bob = await svc.createUser("Bob Jones", "bob@example.com")
  echo fmt"Created: {bob.name} (id={int(bob.id)})"
  
  # Try duplicate email
  try:
    discard await svc.createUser("Another Alice", "alice@example.com")
  except ValueError as e:
    echo fmt"Expected error: {e.msg}"
  
  let page = await repo.findAll(UserFilter(
    status: some(Active),
    limit: 10,
    offset: 0
  ))
  echo fmt"Active users: {page.total}"

waitFor repoDemo()
```

---

## Step 497: CQRS Pattern

```nim
import asyncdispatch, tables, times, strformat, sequtils, options, hashes

# ============================
# Commands (write side)
# ============================

type
  Command = ref object of RootObj
    commandId: string
    issuedAt: float

  CreatePostCommand = ref object of Command
    authorId: int
    title: string
    content: string
    tags: seq[string]

  PublishPostCommand = ref object of Command
    postId: int
    publishedBy: int

  DeletePostCommand = ref object of Command
    postId: int
    deletedBy: int
    reason: string

# ============================
# Queries (read side)
# ============================

type
  Query = ref object of RootObj

  GetPostQuery = ref object of Query
    postId: int

  ListPostsQuery = ref object of Query
    page: int
    limit: int
    tag: string
    authorId: int

# ============================
# Read Models (optimized for display)
# ============================

type
  PostSummary = object
    id: int
    title: string
    authorName: string
    tags: seq[string]
    publishedAt: string
    viewCount: int
    commentCount: int

  PostDetail = object
    id: int
    title: string
    content: string
    authorName: string
    authorAvatar: string
    tags: seq[string]
    publishedAt: string
    viewCount: int
    comments: seq[tuple[author, text: string, at: string]]

# ============================
# Command handlers
# ============================

type
  WriteDb = object
    posts: Table[int, JsonNode]
    nextId: int

var writeDb = WriteDb(posts: initTable[int, JsonNode](), nextId: 1)

# Read-optimized cache
var readPostsCache: Table[int, PostDetail]
var readPostListCache: seq[PostSummary]
var cacheInvalidated = true

proc handleCreatePost(cmd: CreatePostCommand): Future[int] {.async.} =
  let postId = writeDb.nextId
  inc writeDb.nextId
  
  writeDb.posts[postId] = %*{
    "id": postId,
    "authorId": cmd.authorId,
    "title": cmd.title,
    "content": cmd.content,
    "tags": cmd.tags,
    "status": "draft",
    "createdAt": epochTime()
  }
  
  cacheInvalidated = true
  echo fmt"[CMD] Created post {postId}: {cmd.title}"
  return postId

proc handlePublishPost(cmd: PublishPostCommand): Future[bool] {.async.} =
  if cmd.postId notin writeDb.posts:
    return false
  
  writeDb.posts[cmd.postId]["status"] = %"published"
  writeDb.posts[cmd.postId]["publishedAt"] = %epochTime()
  
  cacheInvalidated = true
  echo fmt"[CMD] Published post {cmd.postId}"
  return true

# ============================
# Query handlers (read side)
# ============================

proc rebuildReadModel() =
  readPostListCache = @[]
  readPostsCache.clear()
  
  for _, post in writeDb.posts:
    if post["status"].getStr() == "published":
      let summary = PostSummary(
        id: post["id"].getInt(),
        title: post["title"].getStr(),
        authorName: "Author " & $post["authorId"].getInt(),
        tags: post["tags"].getElems().mapIt(it.getStr()),
        publishedAt: $fromUnixFloat(post.getOrDefault("publishedAt").getFloat()),
        viewCount: 0,
        commentCount: 0
      )
      readPostListCache.add(summary)
      
      readPostsCache[post["id"].getInt()] = PostDetail(
        id: post["id"].getInt(),
        title: post["title"].getStr(),
        content: post["content"].getStr(),
        authorName: "Author " & $post["authorId"].getInt(),
        authorAvatar: "/avatars/default.jpg",
        tags: post["tags"].getElems().mapIt(it.getStr()),
        publishedAt: $fromUnixFloat(post.getOrDefault("publishedAt").getFloat()),
        comments: @[]
      )
  
  cacheInvalidated = false
  echo fmt"[Query] Read model rebuilt: {readPostListCache.len} posts"

proc handleListPosts(query: ListPostsQuery): seq[PostSummary] =
  if cacheInvalidated:
    rebuildReadModel()
  
  var posts = readPostListCache
  
  if query.tag.len > 0:
    posts = posts.filterIt(query.tag in it.tags)
  
  let start = (query.page - 1) * query.limit
  let finish = min(start + query.limit, posts.len)
  
  if start >= posts.len:
    return @[]
  return posts[start ..< finish]

proc handleGetPost(query: GetPostQuery): Option[PostDetail] =
  if cacheInvalidated:
    rebuildReadModel()
  
  if query.postId in readPostsCache:
    return some(readPostsCache[query.postId])
  return none(PostDetail)

# Demo
proc cqrsDemo() {.async.} =
  echo "\n=== CQRS Demo ==="
  
  # Commands
  let id1 = await handleCreatePost(CreatePostCommand(
    authorId: 1, title: "Intro to Nim", content: "Nim is great...",
    tags: @["nim", "intro"]
  ))
  
  let id2 = await handleCreatePost(CreatePostCommand(
    authorId: 1, title: "Async in Nim", content: "Async programming...",
    tags: @["nim", "async"]
  ))
  
  discard await handlePublishPost(PublishPostCommand(postId: id1, publishedBy: 1))
  discard await handlePublishPost(PublishPostCommand(postId: id2, publishedBy: 1))
  
  # Queries
  echo "\nQuery: List all posts"
  let posts = handleListPosts(ListPostsQuery(page: 1, limit: 10, tag: "", authorId: 0))
  for post in posts:
    echo fmt"  [{post.id}] {post.title} ({post.tags.join(\", \")})"
  
  echo "\nQuery: Get post detail"
  let detail = handleGetPost(GetPostQuery(postId: id1))
  if detail.isSome:
    echo fmt"  Title: {detail.get().title}"
    echo fmt"  Author: {detail.get().authorName}"

waitFor cqrsDemo()
```

---

## Step 498: Event Sourcing

```nim
import times, strformat, json, tables, sequtils

# ============================
# Event Store
# ============================

type
  DomainEvent = object
    eventId: string
    aggregateId: string
    aggregateType: string
    eventType: string
    version: int
    payload: JsonNode
    occurredAt: float

  EventStore = object
    events: seq[DomainEvent]

var eventStore = EventStore(events: @[])

proc append(store: var EventStore, event: DomainEvent) =
  store.events.add(event)
  echo fmt"[EventStore] {event.aggregateType}/{event.aggregateId} v{event.version}: {event.eventType}"

proc getEvents(store: EventStore, aggregateId: string): seq[DomainEvent] =
  store.events.filterIt(it.aggregateId == aggregateId)

proc getEventsSince(store: EventStore, aggregateId: string, version: int): seq[DomainEvent] =
  store.events.filterIt(it.aggregateId == aggregateId and it.version > version)

# ============================
# Account Aggregate
# ============================

type
  AccountState = object
    id: string
    owner: string
    balance: float
    status: string
    version: int

proc applyEvent(state: var AccountState, event: DomainEvent) =
  case event.eventType
  of "AccountOpened":
    state.id = event.aggregateId
    state.owner = event.payload["owner"].getStr()
    state.balance = event.payload["initialBalance"].getFloat()
    state.status = "active"
    state.version = event.version
  
  of "MoneyDeposited":
    state.balance += event.payload["amount"].getFloat()
    state.version = event.version
  
  of "MoneyWithdrawn":
    state.balance -= event.payload["amount"].getFloat()
    state.version = event.version
  
  of "AccountClosed":
    state.status = "closed"
    state.version = event.version

proc rebuildAccount(store: EventStore, accountId: string): AccountState =
  var state = AccountState()
  let events = store.getEvents(accountId)
  for event in events:
    state.applyEvent(event)
  return state

# ============================
# Account Commands
# ============================

proc openAccount(accountId, owner: string, initialBalance: float) =
  eventStore.append(DomainEvent(
    eventId: fmt"e{epochTime().int}",
    aggregateId: accountId,
    aggregateType: "Account",
    eventType: "AccountOpened",
    version: 1,
    payload: %*{"owner": owner, "initialBalance": initialBalance},
    occurredAt: epochTime()
  ))

proc deposit(accountId: string, amount: float) =
  let state = rebuildAccount(eventStore, accountId)
  if state.status != "active":
    raise newException(ValueError, "Account is not active")
  
  eventStore.append(DomainEvent(
    eventId: fmt"e{epochTime().int}{hash(amount)}",
    aggregateId: accountId,
    aggregateType: "Account",
    eventType: "MoneyDeposited",
    version: state.version + 1,
    payload: %*{"amount": amount},
    occurredAt: epochTime()
  ))

proc withdraw(accountId: string, amount: float) =
  let state = rebuildAccount(eventStore, accountId)
  if state.status != "active":
    raise newException(ValueError, "Account not active")
  if state.balance < amount:
    raise newException(ValueError, fmt"Insufficient funds: {state.balance} < {amount}")
  
  eventStore.append(DomainEvent(
    eventId: fmt"e{epochTime().int}{hash(amount)}",
    aggregateId: accountId,
    aggregateType: "Account",
    eventType: "MoneyWithdrawn",
    version: state.version + 1,
    payload: %*{"amount": amount},
    occurredAt: epochTime()
  ))

# Demo
echo "=== Event Sourcing Demo ==="
openAccount("acc-001", "Alice", 1000.0)
deposit("acc-001", 500.0)
deposit("acc-001", 250.0)
withdraw("acc-001", 200.0)

try:
  withdraw("acc-001", 2000.0)
except ValueError as e:
  echo fmt"Expected error: {e.msg}"

# Rebuild state from events
let finalState = rebuildAccount(eventStore, "acc-001")
echo fmt"\nAccount: {finalState.id}"
echo fmt"Owner: {finalState.owner}"
echo fmt"Balance: ${finalState.balance:.2f}"
echo fmt"Events: {eventStore.getEvents(\"acc-001\").len}"

# Time-travel: what was balance after event 2?
var stateAtV2 = AccountState()
for event in eventStore.getEvents("acc-001"):
  if event.version <= 2:
    stateAtV2.applyEvent(event)
echo fmt"Balance after 2 events: ${stateAtV2.balance:.2f}"
```

---

## Step 499-510: Hexagonal Architecture

```nim
# hexagonal.nim - Clean/Hexagonal Architecture example

import asyncdispatch, options, tables, strformat, times

# ============================
# Domain (Core) - no external dependencies
# ============================

type
  # Value Objects
  Money = object
    amount: float
    currency: string

  ProductId = distinct int
  CategoryId = distinct int

  # Domain Entity
  Product = object
    id: ProductId
    name: string
    price: Money
    stock: int
    categoryId: CategoryId

  # Domain Events
  ProductCreated = object
    productId: ProductId
    name: string
    price: float

  StockUpdated = object
    productId: ProductId
    oldStock: int
    newStock: int
    reason: string

  # Domain Errors
  DomainError = object of CatchableError

proc newMoney(amount: float, currency: string = "THB"): Money =
  if amount < 0:
    raise newException(DomainError, "Amount cannot be negative")
  Money(amount: amount, currency: currency)

proc add(a, b: Money): Money =
  if a.currency != b.currency:
    raise newException(DomainError, "Cannot add different currencies")
  Money(amount: a.amount + b.amount, currency: a.currency)

proc isAvailable(product: Product): bool = product.stock > 0
proc canFulfill(product: Product, qty: int): bool = product.stock >= qty

# ============================
# Ports (interfaces)
# ============================

type
  # Driven ports (what domain needs from outside)
  IProductStorage = ref object of RootObj

method findProduct(s: IProductStorage, id: ProductId): Future[Option[Product]] {.base, async.} = none(Product)
method saveProduct(s: IProductStorage, p: Product): Future[Product] {.base, async.} = p
method listProducts(s: IProductStorage, limit, offset: int): Future[seq[Product]] {.base, async.} = @[]

type
  IEventPublisher = ref object of RootObj

method publish(pub: IEventPublisher, event: string, data: Product) {.base.} = discard

# ============================
# Application Services (Use Cases)
# ============================

type
  ProductService = object
    storage: IProductStorage
    events: IEventPublisher

proc createProduct(svc: ProductService, name: string, price: float,
                   stock: int, categoryId: int): Future[Product] {.async.} =
  # Validate domain rules
  if name.len < 2:
    raise newException(DomainError, "Product name too short")
  
  let product = Product(
    id: ProductId(0),  # will be assigned by storage
    name: name,
    price: newMoney(price),
    stock: stock,
    categoryId: CategoryId(categoryId)
  )
  
  let saved = await svc.storage.saveProduct(product)
  svc.events.publish("product.created", saved)
  return saved

proc adjustStock(svc: ProductService, productId: int, delta: int, reason: string): Future[Product] {.async.} =
  let pOpt = await svc.storage.findProduct(ProductId(productId))
  if pOpt.isNone:
    raise newException(DomainError, fmt"Product {productId} not found")
  
  var product = pOpt.get()
  let newStock = product.stock + delta
  
  if newStock < 0:
    raise newException(DomainError, fmt"Insufficient stock: {product.stock} + {delta} = {newStock}")
  
  product.stock = newStock
  let saved = await svc.storage.saveProduct(product)
  svc.events.publish("stock.updated", saved)
  return saved

# ============================
# Adapters (implementations)
# ============================

# Storage adapter: In-memory
type
  MemoryProductStorage = ref object of IProductStorage
    products: Table[int, Product]
    nextId: int

proc newMemoryProductStorage(): MemoryProductStorage =
  MemoryProductStorage(products: initTable[int, Product](), nextId: 1)

method findProduct(s: MemoryProductStorage, id: ProductId): Future[Option[Product]] {.async.} =
  let key = int(id)
  if key in s.products:
    return some(s.products[key])
  return none(Product)

method saveProduct(s: MemoryProductStorage, p: Product): Future[Product] {.async.} =
  var product = p
  if int(product.id) == 0:
    product.id = ProductId(s.nextId)
    inc s.nextId
  s.products[int(product.id)] = product
  return product

method listProducts(s: MemoryProductStorage, limit, offset: int): Future[seq[Product]] {.async.} =
  var result = s.products.values.toSeq()
  let start = min(offset, result.len)
  let finish = min(start + limit, result.len)
  return result[start ..< finish]

# Event publisher adapter: Console
type
  ConsoleEventPublisher = ref object of IEventPublisher

method publish(pub: ConsoleEventPublisher, event: string, data: Product) =
  echo fmt"[EVENT] {event}: {data.name} (stock={data.stock})"

# ============================
# Demo: wire everything together
# ============================

proc hexagonalDemo() {.async.} =
  echo "=== Hexagonal Architecture Demo ==="
  
  # Compose application with adapters
  let storage = newMemoryProductStorage()
  let events = ConsoleEventPublisher()
  let svc = ProductService(storage: storage, events: events)
  
  # Use cases
  echo "\nCreating products:"
  let laptop = await svc.createProduct("Laptop Pro", 45000.0, 10, 1)
  echo fmt"  Created: {laptop.name} (id={int(laptop.id)}, stock={laptop.stock})"
  
  let mouse = await svc.createProduct("Wireless Mouse", 990.0, 50, 1)
  echo fmt"  Created: {mouse.name} (id={int(mouse.id)}, stock={mouse.stock})"
  
  echo "\nAdjusting stock:"
  let updated = await svc.adjustStock(int(laptop.id), -3, "sold")
  echo fmt"  Laptop stock: {updated.stock}"
  
  try:
    discard await svc.adjustStock(int(mouse.id), -100, "sold")
  except DomainError as e:
    echo fmt"  Expected: {e.msg}"
  
  echo "\nListing products:"
  let products = await storage.listProducts(10, 0)
  for p in products:
    echo fmt"  {p.name}: ${p.price.amount:.0f} (stock: {p.stock})"

waitFor hexagonalDemo()
```

---

## 📝 สรุป Part 35

| Steps | หัวข้อ |
|-------|--------|
| 496 | Repository + Service Layer pattern |
| 497 | CQRS - write/read side separation |
| 498 | Event Sourcing - rebuild state from events |
| 499-510 | Hexagonal Architecture - ports and adapters |

---

**← [Part 34: Deployment](part_34_deployment.md) | [Part 36: Real-Time Features →](part_36_realtime.md)**
