# Part 23: Advanced Database Patterns
## Steps 316-330: Database ระดับ Production

---

## 🎯 เป้าหมายของ Part นี้

- Database transactions
- Optimistic locking
- Full-text search
- Database migrations
- Query optimization
- N+1 query prevention
- Connection pooling

---

## Step 316: Transactions ขั้นสูง

```nim
import db_connector/db_sqlite, std/strformat, std/times

let db = open("shop.db", "", "", "")

# Setup tables
db.exec(sql"""
  CREATE TABLE IF NOT EXISTS accounts (
    id INTEGER PRIMARY KEY,
    owner TEXT NOT NULL,
    balance DECIMAL(10,2) DEFAULT 0,
    version INTEGER DEFAULT 1
  )
""")

db.exec(sql"""
  CREATE TABLE IF NOT EXISTS transactions (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    from_account INTEGER,
    to_account INTEGER,
    amount DECIMAL(10,2) NOT NULL,
    type TEXT NOT NULL,
    created_at TEXT DEFAULT (datetime('now'))
  )
""")

# Insert initial data
db.exec(sql"INSERT OR IGNORE INTO accounts VALUES (1, 'Alice', 10000, 1)")
db.exec(sql"INSERT OR IGNORE INTO accounts VALUES (2, 'Bob', 5000, 1)")

# ==============================
# Transaction Types
# ==============================

type
  TransferError = object of CatchableError
  InsufficientFundsError = object of TransferError
  AccountNotFoundError = object of TransferError
  StaleVersionError = object of TransferError

# ACID Transfer with optimistic locking
proc transfer(
  fromId, toId: int,
  amount: float,
  maxRetries: int = 3
): bool =
  var attempts = 0
  
  while attempts < maxRetries:
    inc attempts
    
    db.exec(sql"BEGIN")
    
    try:
      # Get accounts with version (optimistic locking)
      let fromRow = db.getRow(sql"""
        SELECT balance, version FROM accounts WHERE id = ? FOR UPDATE
      """, $fromId)
      
      let toRow = db.getRow(sql"""
        SELECT balance, version FROM accounts WHERE id = ? FOR UPDATE
      """, $toId)
      
      if fromRow[0].len == 0:
        raise newException(AccountNotFoundError, fmt"Account {fromId} not found")
      if toRow[0].len == 0:
        raise newException(AccountNotFoundError, fmt"Account {toId} not found")
      
      let fromBalance = parseFloat(fromRow[0])
      let fromVersion = parseInt(fromRow[1])
      
      if fromBalance < amount:
        raise newException(InsufficientFundsError,
          fmt"Insufficient funds. Balance: {fromBalance:.2f}, Required: {amount:.2f}")
      
      # Update balances (with version check)
      let updated = db.execAffectedRows(sql"""
        UPDATE accounts 
        SET balance = balance - ?, version = version + 1
        WHERE id = ? AND version = ?
      """, $amount, $fromId, $fromVersion)
      
      if updated == 0:
        raise newException(StaleVersionError, "Concurrent modification detected")
      
      db.exec(sql"""
        UPDATE accounts SET balance = balance + ?, version = version + 1
        WHERE id = ?
      """, $amount, $toId)
      
      # Record transaction
      db.exec(sql"""
        INSERT INTO transactions (from_account, to_account, amount, type)
        VALUES (?, ?, ?, 'transfer')
      """, $fromId, $toId, $amount)
      
      db.exec(sql"COMMIT")
      return true
    
    except StaleVersionError:
      db.exec(sql"ROLLBACK")
      echo fmt"  Retry {attempts}/{maxRetries} - stale version"
      sleep(10 * attempts)  # exponential backoff
    
    except TransferError as e:
      db.exec(sql"ROLLBACK")
      echo "Transfer failed: " & e.msg
      return false
    
    except:
      db.exec(sql"ROLLBACK")
      echo "Unexpected error: " & getCurrentExceptionMsg()
      return false
  
  return false

# Test transfer
echo "=== Transfer Test ==="

let showBalance = proc() =
  for row in db.fastRows(sql"SELECT id, owner, balance FROM accounts ORDER BY id"):
    echo fmt"  Account {row[0]} ({row[1]}): ฿{parseFloat(row[2]):.2f}"

echo "Before:"
showBalance()

discard transfer(1, 2, 2500.0)

echo "After ฿2500 transfer:"
showBalance()

discard transfer(1, 2, 99999.0)  # Should fail: insufficient funds
```

---

## Step 317: Query Optimization

```nim
import db_connector/db_sqlite, std/strformat, std/times, std/sequtils

let db2 = open("perf.db", "", "", "")

# Setup
db2.exec(sql"PRAGMA journal_mode=WAL")
db2.exec(sql"PRAGMA synchronous=NORMAL")
db2.exec(sql"PRAGMA cache_size=10000")
db2.exec(sql"PRAGMA temp_store=MEMORY")

db2.exec(sql"""
  CREATE TABLE IF NOT EXISTS orders (
    id INTEGER PRIMARY KEY,
    user_id INTEGER NOT NULL,
    product_id INTEGER NOT NULL,
    quantity INTEGER NOT NULL,
    total_price DECIMAL(10,2),
    status TEXT DEFAULT 'pending',
    created_at TEXT DEFAULT (datetime('now'))
  )
""")

db2.exec(sql"""
  CREATE TABLE IF NOT EXISTS order_items (
    id INTEGER PRIMARY KEY,
    order_id INTEGER NOT NULL REFERENCES orders(id),
    product_name TEXT,
    price DECIMAL(10,2),
    qty INTEGER
  )
""")

# Create indexes for common queries
db2.exec(sql"CREATE INDEX IF NOT EXISTS idx_orders_user_id ON orders(user_id)")
db2.exec(sql"CREATE INDEX IF NOT EXISTS idx_orders_status ON orders(status)")
db2.exec(sql"CREATE INDEX IF NOT EXISTS idx_orders_created ON orders(created_at)")
db2.exec(sql"CREATE INDEX IF NOT EXISTS idx_items_order_id ON order_items(order_id)")

# Analyze query with EXPLAIN
proc explainQuery(db: DbConn, query: string) =
  echo "QUERY PLAN:"
  for row in db.fastRows(SqlQuery("EXPLAIN QUERY PLAN " & query)):
    echo "  " & row.join(" | ")

explainQuery(db2, "SELECT * FROM orders WHERE user_id = 1 AND status = 'pending'")
explainQuery(db2, "SELECT * FROM orders WHERE created_at > '2024-01-01' ORDER BY created_at")

# N+1 Query Prevention - use JOINs
echo "\n--- N+1 Problem ---"

# BAD: N+1 queries
proc getOrdersWithItemsBad(db: DbConn, userId: int): seq[tuple[order, items: string]] =
  result = @[]
  for orderRow in db.fastRows(sql"SELECT id, total_price FROM orders WHERE user_id = ?", $userId):
    # This causes N+1: one query per order!
    let items = db.getAllRows(sql"SELECT product_name, qty FROM order_items WHERE order_id = ?", orderRow[0])
    result.add((orderRow.join(","), items.mapIt(it.join(":")).join(", ")))

# GOOD: Single JOIN query
proc getOrdersWithItemsGood(db: DbConn, userId: int): string =
  let rows = db.getAllRows(sql"""
    SELECT o.id, o.total_price, oi.product_name, oi.qty
    FROM orders o
    LEFT JOIN order_items oi ON oi.order_id = o.id
    WHERE o.user_id = ?
    ORDER BY o.id
  """, $userId)
  
  var currentOrderId = ""
  var result = ""
  
  for row in rows:
    if row[0] != currentOrderId:
      result &= fmt"\nOrder #{row[0]}: ฿{row[1]}\n"
      currentOrderId = row[0]
    
    if row[2].len > 0:
      result &= fmt"  - {row[2]} x{row[3]}\n"
  
  return result

# Bulk insert (much faster than individual inserts)
proc bulkInsertOrders(db: DbConn, orders: seq[tuple[userId, productId, qty: int, price: float]]) =
  db.exec(sql"BEGIN")
  try:
    for (userId, productId, qty, price) in orders:
      db.exec(sql"""
        INSERT INTO orders (user_id, product_id, quantity, total_price)
        VALUES (?, ?, ?, ?)
      """, $userId, $productId, $qty, $price)
    db.exec(sql"COMMIT")
  except:
    db.exec(sql"ROLLBACK")
    raise

# Benchmark
let orders = (1..100).toSeq().mapIt((
  userId: rand(10) + 1,
  productId: rand(50) + 1,
  qty: rand(5) + 1,
  price: float(rand(10000)) / 100.0
))

let start = now()
bulkInsertOrders(db2, orders)
let elapsed = (now() - start).inMilliseconds
echo fmt"Inserted 100 orders in {elapsed}ms"
```

---

## Step 318-330: Full E-Commerce Database Layer

```nim
# ecommerce_db.nim - Production e-commerce database layer

import db_connector/db_sqlite, std/strformat, std/tables,
       std/options, std/times, std/sequtils, std/strutils, std/json

# ==============================
# Database Setup
# ==============================

type
  Db = object
    conn: DbConn

proc newDb(path: string): Db =
  let conn = open(path, "", "", "")
  conn.exec(sql"PRAGMA journal_mode=WAL")
  conn.exec(sql"PRAGMA foreign_keys=ON")
  conn.exec(sql"PRAGMA cache_size=10000")
  Db(conn: conn)

proc close(db: Db) = db.conn.close()

proc migrate(db: Db) =
  db.conn.exec(sql"""
    CREATE TABLE IF NOT EXISTS categories (
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      name TEXT UNIQUE NOT NULL,
      slug TEXT UNIQUE NOT NULL,
      parent_id INTEGER REFERENCES categories(id)
    )
  """)
  
  db.conn.exec(sql"""
    CREATE TABLE IF NOT EXISTS products2 (
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      sku TEXT UNIQUE NOT NULL,
      name TEXT NOT NULL,
      description TEXT,
      price DECIMAL(10,2) NOT NULL,
      cost DECIMAL(10,2),
      stock INTEGER DEFAULT 0,
      category_id INTEGER REFERENCES categories(id),
      is_active INTEGER DEFAULT 1,
      metadata JSON,
      created_at TEXT DEFAULT (datetime('now')),
      updated_at TEXT DEFAULT (datetime('now'))
    )
  """)
  
  db.conn.exec(sql"""
    CREATE TABLE IF NOT EXISTS customers (
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      name TEXT NOT NULL,
      email TEXT UNIQUE NOT NULL,
      phone TEXT,
      created_at TEXT DEFAULT (datetime('now'))
    )
  """)
  
  db.conn.exec(sql"""
    CREATE TABLE IF NOT EXISTS orders2 (
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      order_number TEXT UNIQUE NOT NULL,
      customer_id INTEGER NOT NULL REFERENCES customers(id),
      status TEXT DEFAULT 'pending',
      subtotal DECIMAL(10,2) DEFAULT 0,
      discount DECIMAL(10,2) DEFAULT 0,
      tax DECIMAL(10,2) DEFAULT 0,
      total DECIMAL(10,2) DEFAULT 0,
      notes TEXT,
      created_at TEXT DEFAULT (datetime('now')),
      updated_at TEXT DEFAULT (datetime('now'))
    )
  """)
  
  db.conn.exec(sql"""
    CREATE TABLE IF NOT EXISTS order_items2 (
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      order_id INTEGER NOT NULL REFERENCES orders2(id) ON DELETE CASCADE,
      product_id INTEGER NOT NULL REFERENCES products2(id),
      quantity INTEGER NOT NULL,
      unit_price DECIMAL(10,2) NOT NULL,
      total_price DECIMAL(10,2) NOT NULL
    )
  """)
  
  # Indexes
  for idx in [
    "CREATE INDEX IF NOT EXISTS idx_products_sku ON products2(sku)",
    "CREATE INDEX IF NOT EXISTS idx_products_category ON products2(category_id)",
    "CREATE INDEX IF NOT EXISTS idx_products_active ON products2(is_active)",
    "CREATE INDEX IF NOT EXISTS idx_orders_customer ON orders2(customer_id)",
    "CREATE INDEX IF NOT EXISTS idx_orders_status ON orders2(status)",
    "CREATE INDEX IF NOT EXISTS idx_items_order ON order_items2(order_id)",
  ]:
    db.conn.exec(SqlQuery(idx))

# ==============================
# Product Repository
# ==============================

type
  ProductFilter = object
    categoryId: Option[int]
    minPrice: Option[float]
    maxPrice: Option[float]
    search: Option[string]
    inStock: bool

  ProductRow = object
    id: int
    sku: string
    name: string
    description: string
    price: float
    stock: int
    categoryName: string
    isActive: bool

proc searchProducts(db: Db, filter: ProductFilter, limit: int = 20, offset: int = 0): seq[ProductRow] =
  var conditions = @["p.is_active = 1"]
  var params: seq[string] = @[]
  
  if filter.categoryId.isSome:
    conditions.add("p.category_id = ?")
    params.add($filter.categoryId.get())
  
  if filter.minPrice.isSome:
    conditions.add("p.price >= ?")
    params.add($filter.minPrice.get())
  
  if filter.maxPrice.isSome:
    conditions.add("p.price <= ?")
    params.add($filter.maxPrice.get())
  
  if filter.search.isSome:
    let q = "%" & filter.search.get() & "%"
    conditions.add("(p.name LIKE ? OR p.description LIKE ? OR p.sku LIKE ?)")
    params.add(q)
    params.add(q)
    params.add(q)
  
  if filter.inStock:
    conditions.add("p.stock > 0")
  
  let whereStr = conditions.join(" AND ")
  
  let rows = db.conn.getAllRows(SqlQuery(fmt"""
    SELECT p.id, p.sku, p.name, p.description, p.price, p.stock,
           COALESCE(c.name, '') as category
    FROM products2 p
    LEFT JOIN categories c ON c.id = p.category_id
    WHERE {whereStr}
    ORDER BY p.name ASC
    LIMIT ? OFFSET ?
  """), params & @[$limit, $offset])
  
  result = @[]
  for row in rows:
    result.add(ProductRow(
      id: parseInt(row[0]),
      sku: row[1],
      name: row[2],
      description: row[3],
      price: parseFloat(row[4]),
      stock: parseInt(row[5]),
      categoryName: row[6],
      isActive: true
    ))

# ==============================
# Order Service
# ==============================

type
  OrderItem = object
    productId: int
    quantity: int

  CreateOrderInput2 = object
    customerId: int
    items: seq[OrderItem]
    notes: string

  OrderResult = object
    success: bool
    message: string
    orderId: int
    orderNumber: string
    total: float

proc generateOrderNumber(): string =
  "ORD-" & now().format("yyyyMMdd") & "-" & $(rand(9999) + 1000)

proc createOrder(db: Db, input: CreateOrderInput2): OrderResult =
  db.conn.exec(sql"BEGIN")
  
  try:
    # Validate customer
    let custRow = db.conn.getRow(sql"SELECT id FROM customers WHERE id = ?", $input.customerId)
    if custRow[0].len == 0:
      raise newException(ValueError, "Customer not found")
    
    # Validate items and calculate totals
    var subtotal = 0.0
    var orderItems: seq[(int, int, float)] = @[]  # productId, qty, unitPrice
    
    for item in input.items:
      let prodRow = db.conn.getRow(sql"""
        SELECT id, name, price, stock FROM products2 
        WHERE id = ? AND is_active = 1
      """, $item.productId)
      
      if prodRow[0].len == 0:
        raise newException(ValueError, fmt"Product {item.productId} not found")
      
      let stock = parseInt(prodRow[3])
      if stock < item.quantity:
        raise newException(ValueError,
          fmt"Insufficient stock for '{prodRow[1]}'. Available: {stock}")
      
      let price = parseFloat(prodRow[2])
      subtotal += price * float(item.quantity)
      orderItems.add((item.productId, item.quantity, price))
    
    let tax = subtotal * 0.07  # 7% VAT
    let total = subtotal + tax
    let orderNumber = generateOrderNumber()
    
    # Create order
    let orderId = db.conn.insertID(sql"""
      INSERT INTO orders2 (order_number, customer_id, status, subtotal, tax, total, notes)
      VALUES (?, ?, 'confirmed', ?, ?, ?, ?)
    """, orderNumber, $input.customerId, $subtotal, $tax, $total, input.notes)
    
    # Create order items and update stock
    for (productId, qty, unitPrice) in orderItems:
      db.conn.exec(sql"""
        INSERT INTO order_items2 (order_id, product_id, quantity, unit_price, total_price)
        VALUES (?, ?, ?, ?, ?)
      """, $orderId, $productId, $qty, $unitPrice, $(unitPrice * float(qty)))
      
      # Deduct stock
      db.conn.exec(sql"""
        UPDATE products2 SET stock = stock - ?, updated_at = datetime('now')
        WHERE id = ?
      """, $qty, $productId)
    
    db.conn.exec(sql"COMMIT")
    
    return OrderResult(
      success: true,
      message: "Order created successfully",
      orderId: int(orderId),
      orderNumber: orderNumber,
      total: total
    )
  
  except ValueError as e:
    db.conn.exec(sql"ROLLBACK")
    return OrderResult(success: false, message: e.msg)
  
  except:
    db.conn.exec(sql"ROLLBACK")
    return OrderResult(success: false, message: "Unexpected error: " & getCurrentExceptionMsg())

# ==============================
# Demo
# ==============================

proc main() =
  var db = newDb("ecommerce.db")
  defer: db.close()
  
  db.migrate()
  
  echo "=== E-Commerce Database Demo ==="
  
  # Seed categories
  db.conn.exec(sql"INSERT OR IGNORE INTO categories (name, slug) VALUES ('Electronics', 'electronics')")
  db.conn.exec(sql"INSERT OR IGNORE INTO categories (name, slug) VALUES ('Stationery', 'stationery')")
  
  let elecId = parseInt(db.conn.getValue(sql"SELECT id FROM categories WHERE slug = 'electronics'"))
  
  # Seed products
  for (sku, name, price, stock) in [
    ("LAP001", "Laptop Pro 15", 45000.0, 10),
    ("MOU001", "Wireless Mouse", 1200.0, 50),
    ("KEY001", "Mechanical Keyboard", 3500.0, 25),
    ("HUB001", "USB-C Hub", 1800.0, 30),
  ]:
    db.conn.exec(sql"""
      INSERT OR IGNORE INTO products2 (sku, name, price, stock, category_id)
      VALUES (?, ?, ?, ?, ?)
    """, sku, name, $price, $stock, $elecId)
  
  # Seed customer
  db.conn.exec(sql"""
    INSERT OR IGNORE INTO customers (name, email) VALUES ('Alice Smith', 'alice@example.com')
  """)
  
  let customerId = parseInt(db.conn.getValue(sql"SELECT id FROM customers WHERE email = 'alice@example.com'"))
  let lapId = parseInt(db.conn.getValue(sql"SELECT id FROM products2 WHERE sku = 'LAP001'"))
  let mouId = parseInt(db.conn.getValue(sql"SELECT id FROM products2 WHERE sku = 'MOU001'"))
  
  # Search products
  echo "\n--- Product Search ---"
  let products = searchProducts(db, ProductFilter(
    search: some("mouse"),
    inStock: true
  ))
  for p in products:
    echo fmt"  [{p.sku}] {p.name}: ฿{p.price:.2f} (stock: {p.stock})"
  
  # Create order
  echo "\n--- Create Order ---"
  let orderResult = createOrder(db, CreateOrderInput2(
    customerId: customerId,
    items: @[
      OrderItem(productId: lapId, quantity: 1),
      OrderItem(productId: mouId, quantity: 2),
    ],
    notes: "Gift wrap please"
  ))
  
  echo fmt"Success: {orderResult.success}"
  echo fmt"Order: {orderResult.orderNumber}"
  echo fmt"Total: ฿{orderResult.total:.2f}"

main()
```

---

## 📝 สรุป Part 23

| Steps | หัวข้อ |
|-------|--------|
| 316 | Transactions, optimistic locking |
| 317 | Query optimization, N+1 prevention |
| 318-330 | E-Commerce database layer |

---

**← [Part 22: Authentication](part_22_authentication.md) | [Part 24: Caching →](part_24_caching.md)**
