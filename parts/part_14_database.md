# Part 14: Database - SQLite & PostgreSQL
## Steps 176-190: การทำงานกับ Database

---

## 🎯 เป้าหมายของ Part นี้

- ใช้ SQLite กับ Nim
- เชื่อมต่อ PostgreSQL
- Query Builder
- Connection Pooling
- Database migrations
- ORM patterns

---

## Step 176: SQLite พื้นฐาน

```nim
# nimble install db_connector
import db_connector/db_sqlite, std/strformat, std/times

# Open database
let db = open("app.db", "", "", "")
defer: db.close()

# Create table
db.exec(sql"""
  CREATE TABLE IF NOT EXISTS users (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT NOT NULL,
    email TEXT UNIQUE NOT NULL,
    password_hash TEXT NOT NULL,
    created_at TEXT DEFAULT (datetime('now')),
    is_active INTEGER DEFAULT 1
  )
""")

# Insert
proc insertUser(db: DbConn, name, email, passwordHash: string): int64 =
  db.insertID(sql"""
    INSERT INTO users (name, email, password_hash) 
    VALUES (?, ?, ?)
  """, name, email, passwordHash)

let id1 = insertUser(db, "Alice", "alice@example.com", "hash_alice")
let id2 = insertUser(db, "Bob", "bob@example.com", "hash_bob")

# Query all
echo "All users:"
for row in db.fastRows(sql"SELECT id, name, email FROM users"):
  echo fmt"  [{row[0]}] {row[1]} ({row[2]})"

# Query single
let user = db.getRow(sql"SELECT id, name, email FROM users WHERE id = ?", id1)
echo fmt"User: {user[1]} ({user[2]})"

# Update
db.exec(sql"UPDATE users SET name = ? WHERE id = ?", "Alice Smith", id1)

# Delete
db.exec(sql"DELETE FROM users WHERE id = ?", id2)

# Count
let count = db.getValue(sql"SELECT COUNT(*) FROM users")
echo "User count: " & count

# Transaction
db.exec(sql"BEGIN")
try:
  discard insertUser(db, "Charlie", "charlie@example.com", "hash_charlie")
  discard insertUser(db, "Diana", "diana@example.com", "hash_diana")
  db.exec(sql"COMMIT")
  echo "Transaction committed"
except:
  db.exec(sql"ROLLBACK")
  echo "Transaction rolled back: " & getCurrentExceptionMsg()
```

---

## Step 177: SQLite ORM Pattern

```nim
import db_connector/db_sqlite, std/strformat, std/tables,
       std/sequtils, std/options, std/times

type
  Model = object of RootObj
    id: int
    createdAt: string
    updatedAt: string

  UserRecord = object of Model
    name: string
    email: string
    passwordHash: string
    isActive: bool

  PostRecord = object of Model
    title: string
    content: string
    authorId: int
    published: bool

# Database abstraction
type
  Database = object
    conn: DbConn
    path: string

proc newDatabase(path: string): Database =
  let conn = open(path, "", "", "")
  
  # Enable WAL mode for better concurrency
  conn.exec(sql"PRAGMA journal_mode=WAL")
  conn.exec(sql"PRAGMA foreign_keys=ON")
  
  return Database(conn: conn, path: path)

proc close(db: Database) = db.conn.close()

# Migration system
type
  Migration = object
    version: int
    description: string
    up: string
    down: string

proc runMigrations(db: Database, migrations: seq[Migration]) =
  # Create migrations table
  db.conn.exec(sql"""
    CREATE TABLE IF NOT EXISTS migrations (
      version INTEGER PRIMARY KEY,
      description TEXT,
      applied_at TEXT
    )
  """)
  
  # Find applied migrations
  var applied: seq[int] = @[]
  for row in db.conn.fastRows(sql"SELECT version FROM migrations ORDER BY version"):
    applied.add(parseInt(row[0]))
  
  # Apply pending migrations
  for migration in migrations:
    if migration.version notin applied:
      echo fmt"  Applying migration {migration.version}: {migration.description}"
      
      db.conn.exec(sql"BEGIN")
      try:
        db.conn.exec(SqlQuery(migration.up))
        db.conn.exec(sql"""
          INSERT INTO migrations (version, description, applied_at) 
          VALUES (?, ?, datetime('now'))
        """, $migration.version, migration.description)
        db.conn.exec(sql"COMMIT")
        echo "  ✅ Applied"
      except:
        db.conn.exec(sql"ROLLBACK")
        echo "  ❌ Failed: " & getCurrentExceptionMsg()
        break

# Migrations
let migrations = @[
  Migration(
    version: 1,
    description: "Create users table",
    up: """
      CREATE TABLE users (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        name TEXT NOT NULL,
        email TEXT UNIQUE NOT NULL,
        password_hash TEXT NOT NULL,
        is_active INTEGER DEFAULT 1,
        created_at TEXT DEFAULT (datetime('now')),
        updated_at TEXT DEFAULT (datetime('now'))
      )
    """,
    down: "DROP TABLE users"
  ),
  Migration(
    version: 2,
    description: "Create posts table",
    up: """
      CREATE TABLE posts (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        title TEXT NOT NULL,
        content TEXT,
        author_id INTEGER NOT NULL REFERENCES users(id),
        published INTEGER DEFAULT 0,
        created_at TEXT DEFAULT (datetime('now')),
        updated_at TEXT DEFAULT (datetime('now'))
      )
    """,
    down: "DROP TABLE posts"
  ),
  Migration(
    version: 3,
    description: "Add index on posts.author_id",
    up: "CREATE INDEX idx_posts_author_id ON posts(author_id)",
    down: "DROP INDEX idx_posts_author_id"
  ),
]

# Usage
var database = newDatabase("app.db")
defer: database.close()

echo "Running migrations..."
database.runMigrations(migrations)
echo "Migrations complete!"
```

---

## Step 178: Query Builder

```nim
import db_connector/db_sqlite, std/strformat, std/sequtils, std/strutils

type
  WhereClause = object
    clause: string
    params: seq[string]

  SelectQuery = object
    table: string
    columns: seq[string]
    wheres: seq[WhereClause]
    orderBys: seq[string]
    limitVal: int
    offsetVal: int
    joins: seq[string]

  InsertQuery = object
    table: string
    columns: seq[string]
    values: seq[string]

  UpdateQuery = object
    table: string
    sets: seq[(string, string)]
    wheres: seq[WhereClause]

proc select(table: string, cols: varargs[string]): SelectQuery =
  SelectQuery(
    table: table,
    columns: if cols.len == 0: @["*"] else: @cols,
    wheres: @[],
    orderBys: @[],
    limitVal: -1,
    offsetVal: -1,
    joins: @[]
  )

proc where(q: SelectQuery, clause: string, params: varargs[string]): SelectQuery =
  result = q
  result.wheres.add(WhereClause(clause: clause, params: @params))

proc orderBy(q: SelectQuery, col: string): SelectQuery =
  result = q
  result.orderBys.add(col)

proc limit(q: SelectQuery, n: int): SelectQuery =
  result = q
  result.limitVal = n

proc offset(q: SelectQuery, n: int): SelectQuery =
  result = q
  result.offsetVal = n

proc join(q: SelectQuery, clause: string): SelectQuery =
  result = q
  result.joins.add(clause)

proc build(q: SelectQuery): (string, seq[string]) =
  var sql = "SELECT " & q.columns.join(", ")
  sql &= " FROM " & q.table
  
  for j in q.joins:
    sql &= " " & j
  
  var params: seq[string] = @[]
  if q.wheres.len > 0:
    let clauses = q.wheres.mapIt(it.clause)
    sql &= " WHERE " & clauses.join(" AND ")
    for w in q.wheres:
      params.add(w.params)
  
  if q.orderBys.len > 0:
    sql &= " ORDER BY " & q.orderBys.join(", ")
  
  if q.limitVal > 0:
    sql &= " LIMIT " & $q.limitVal
  
  if q.offsetVal > 0:
    sql &= " OFFSET " & $q.offsetVal
  
  return (sql, params)

proc exec(db: DbConn, q: SelectQuery): seq[Row] =
  let (sqlStr, params) = q.build()
  return db.getAllRows(SqlQuery(sqlStr), params)

# Usage
let db = open("app.db", "", "", "")
defer: db.close()

let (sql, params) = select("users", "id", "name", "email")
  .where("is_active = 1")
  .where("name LIKE ?", "%Ali%")
  .orderBy("name ASC")
  .limit(10)
  .offset(0)
  .build()

echo "SQL: " & sql
echo "Params: " & params.join(", ")

# Execute
let rows = db.exec(
  select("users", "id", "name", "email")
    .where("is_active = 1")
    .limit(5)
)

for row in rows:
  echo row.join(" | ")
```

---

## Step 179: PostgreSQL Connection

```nim
# nimble install db_connector
import db_connector/db_postgres, std/strformat

# Connect to PostgreSQL
let db = open(
  "host=localhost port=5432 dbname=myapp",
  "postgres",    # user
  "password",    # password
  ""
)
defer: db.close()

# Test connection
let version = db.getValue(sql"SELECT version()")
echo "Connected: " & version

# Create table
db.exec(sql"""
  CREATE TABLE IF NOT EXISTS products (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    price DECIMAL(10,2) NOT NULL,
    stock INTEGER NOT NULL DEFAULT 0,
    category VARCHAR(100),
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
  )
""")

# Insert with returning
proc insertProduct(
  db: DbConn,
  name, description: string,
  price: float,
  stock: int,
  category: string
): int =
  let row = db.getRow(sql"""
    INSERT INTO products (name, description, price, stock, category)
    VALUES ($1, $2, $3, $4, $5)
    RETURNING id
  """, name, description, $price, $stock, category)
  return parseInt(row[0])

let productId = insertProduct(
  db,
  "Laptop Pro",
  "High-performance laptop",
  45000.00,
  10,
  "Electronics"
)

echo "Created product ID: " & $productId

# Full text search
for row in db.fastRows(sql"""
  SELECT id, name, price
  FROM products
  WHERE to_tsvector('english', name || ' ' || description) @@ to_tsquery($1)
""", "laptop"):
  echo fmt"  [{row[0]}] {row[1]}: ฿{row[2]}"

# JSON operations (PostgreSQL specific)
db.exec(sql"""
  ALTER TABLE products ADD COLUMN IF NOT EXISTS metadata JSONB DEFAULT '{}'
""")

db.exec(sql"""
  UPDATE products 
  SET metadata = $1::jsonb
  WHERE id = $2
""", """{"warranty": "1 year", "weight": "2.5kg"}""", $productId)

# Query JSONB
for row in db.fastRows(sql"""
  SELECT id, name, metadata->>'warranty' as warranty
  FROM products
  WHERE metadata->>'warranty' IS NOT NULL
"""):
  echo fmt"  {row[1]}: warranty = {row[2]}"
```

---

## Step 180-190: Full Database Layer

```nim
# database_layer.nim - Production database layer

import db_connector/db_sqlite, std/strformat, std/tables,
       std/options, std/times, std/sequtils, std/strutils

# ==============================
# Connection Pool (simplified)
# ==============================

type
  PooledConnection = object
    conn: DbConn
    inUse: bool
    lastUsed: float

  ConnectionPool = object
    connections: seq[PooledConnection]
    dbPath: string
    maxSize: int

proc newPool(dbPath: string, size: int = 5): ConnectionPool =
  var pool = ConnectionPool(
    connections: @[],
    dbPath: dbPath,
    maxSize: size
  )
  
  for i in 0..<size:
    let conn = open(dbPath, "", "", "")
    conn.exec(sql"PRAGMA journal_mode=WAL")
    conn.exec(sql"PRAGMA foreign_keys=ON")
    pool.connections.add(PooledConnection(conn: conn, inUse: false, lastUsed: epochTime()))
  
  return pool

proc acquire(pool: var ConnectionPool): DbConn =
  for i in 0..<pool.connections.len:
    if not pool.connections[i].inUse:
      pool.connections[i].inUse = true
      pool.connections[i].lastUsed = epochTime()
      return pool.connections[i].conn
  
  raise newException(IOError, "No available database connections")

proc release(pool: var ConnectionPool, conn: DbConn) =
  for i in 0..<pool.connections.len:
    if pool.connections[i].conn == conn:
      pool.connections[i].inUse = false
      return

template withDb(pool: var ConnectionPool, db, body: untyped) =
  let db = pool.acquire()
  try:
    body
  finally:
    pool.release(db)

# ==============================
# User Repository
# ==============================

type
  UserRow = object
    id: int
    name: string
    email: string
    passwordHash: string
    isActive: bool
    createdAt: string

proc rowToUser(row: Row): UserRow =
  UserRow(
    id: parseInt(row[0]),
    name: row[1],
    email: row[2],
    passwordHash: row[3],
    isActive: row[4] == "1",
    createdAt: row[5]
  )

type
  UserRepo = object
    pool: ptr ConnectionPool

proc findAll(repo: UserRepo, activeOnly: bool = true): seq[UserRow] =
  withDb(repo.pool[]):
    let sql = if activeOnly:
      "SELECT id, name, email, password_hash, is_active, created_at FROM users WHERE is_active = 1 ORDER BY id"
    else:
      "SELECT id, name, email, password_hash, is_active, created_at FROM users ORDER BY id"
    
    return db.getAllRows(SqlQuery(sql)).mapIt(rowToUser(it))

proc findById(repo: UserRepo, id: int): Option[UserRow] =
  withDb(repo.pool[]):
    let rows = db.getAllRows(sql"""
      SELECT id, name, email, password_hash, is_active, created_at 
      FROM users WHERE id = ?
    """, $id)
    
    if rows.len > 0:
      return some(rowToUser(rows[0]))
    return none(UserRow)

proc findByEmail(repo: UserRepo, email: string): Option[UserRow] =
  withDb(repo.pool[]):
    let rows = db.getAllRows(sql"""
      SELECT id, name, email, password_hash, is_active, created_at 
      FROM users WHERE LOWER(email) = LOWER(?)
    """, email)
    
    if rows.len > 0:
      return some(rowToUser(rows[0]))
    return none(UserRow)

proc create(repo: UserRepo, name, email, passwordHash: string): UserRow =
  withDb(repo.pool[]):
    let id = db.insertID(sql"""
      INSERT INTO users (name, email, password_hash) VALUES (?, ?, ?)
    """, name, email, passwordHash)
    
    return repo.findById(int(id)).get()

proc update(repo: UserRepo, id: int, name: string): bool =
  withDb(repo.pool[]):
    db.exec(sql"""
      UPDATE users SET name = ?, updated_at = datetime('now') WHERE id = ?
    """, name, $id)
    return db.changes() > 0

proc softDelete(repo: UserRepo, id: int): bool =
  withDb(repo.pool[]):
    db.exec(sql"""
      UPDATE users SET is_active = 0, updated_at = datetime('now') WHERE id = ?
    """, $id)
    return db.changes() > 0

# ==============================
# Demo
# ==============================

proc main() =
  echo "=== Database Layer Demo ==="
  
  var pool = newPool("demo.db")
  
  # Setup schema
  withDb(pool):
    db.exec(sql"""
      CREATE TABLE IF NOT EXISTS users (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        name TEXT NOT NULL,
        email TEXT UNIQUE NOT NULL,
        password_hash TEXT NOT NULL,
        is_active INTEGER DEFAULT 1,
        created_at TEXT DEFAULT (datetime('now')),
        updated_at TEXT DEFAULT (datetime('now'))
      )
    """)
  
  let repo = UserRepo(pool: addr pool)
  
  # Create users
  echo "\n1. Creating users..."
  let alice = repo.create("Alice Smith", "alice@example.com", "hash_alice")
  let bob = repo.create("Bob Jones", "bob@example.com", "hash_bob")
  let charlie = repo.create("Charlie Brown", "charlie@example.com", "hash_charlie")
  
  echo fmt"   ✅ Created: {alice.name} (ID: {alice.id})"
  echo fmt"   ✅ Created: {bob.name} (ID: {bob.id})"
  echo fmt"   ✅ Created: {charlie.name} (ID: {charlie.id})"
  
  # List all
  echo "\n2. Listing all users..."
  for user in repo.findAll():
    echo fmt"   [{user.id}] {user.name} ({user.email})"
  
  # Find by email
  echo "\n3. Finding by email..."
  let found = repo.findByEmail("alice@example.com")
  if found.isSome:
    echo fmt"   Found: {found.get().name}"
  
  # Update
  echo "\n4. Updating Bob..."
  discard repo.update(bob.id, "Robert Jones")
  let updated = repo.findById(bob.id)
  if updated.isSome:
    echo fmt"   Updated name: {updated.get().name}"
  
  # Soft delete
  echo "\n5. Deleting Charlie (soft)..."
  discard repo.softDelete(charlie.id)
  
  echo "\n6. Active users only:"
  for user in repo.findAll(activeOnly = true):
    echo fmt"   [{user.id}] {user.name}"
  
  echo "\n✅ Done!"
  
  # Cleanup
  for pc in pool.connections:
    pc.conn.close()

main()
```

---

## 📝 สรุป Part 14

| Steps | หัวข้อ |
|-------|--------|
| 176 | SQLite basics |
| 177 | ORM pattern, migrations |
| 178 | Query builder |
| 179 | PostgreSQL connection |
| 180-190 | Full database layer (connection pool, repository) |

---

**← [Part 13: Async](part_13_async.md) | [Part 15: REST API →](part_15_rest_api.md)**
