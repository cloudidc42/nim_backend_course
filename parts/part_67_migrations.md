# Part 67: Database Migrations
## Steps 976-990: Schema Evolution with Nim

---

## 🎯 เป้าหมายของ Part นี้

- Migration versioning (sequential + timestamp)
- Up/down migration pairs
- Migration runner with transaction support
- Rollback on failure
- Migration status tracking
- Seed data management
- Schema diff detection

---

## Step 976: Migration System

```nim
import tables, strformat, times, sequtils, json, strutils, options, algorithm, os

# ============================
# Migration types
# ============================

type
  MigrationDirection = enum
    mdUp, mdDown

  SqlStatement = object
    sql: string
    description: string

  Migration = object
    version: string      # e.g. "20240115_001" or "001"
    name: string
    up: seq[SqlStatement]
    down: seq[SqlStatement]
    checksum: string
    author: string
    createdAt: float

  MigrationRecord = object
    version: string
    name: string
    appliedAt: float
    checksum: string
    executionMs: float
    success: bool
    error: string

  MigrationsTable = object
    records: seq[MigrationRecord]

  MigrationRunner = object
    migrations: seq[Migration]
    history: MigrationsTable
    dryRun: bool
    verbose: bool

# ============================
# Migration builder
# ============================

proc migration(version, name: string, author = "unknown"): Migration =
  Migration(
    version: version,
    name: name,
    author: author,
    createdAt: epochTime(),
    up: @[],
    down: @[]
  )

proc addUp(m: var Migration, sql, description: string) =
  m.up.add(SqlStatement(sql: sql, description: description))

proc addDown(m: var Migration, sql, description: string) =
  m.down.add(SqlStatement(sql: sql, description: description))

proc withChecksum(m: var Migration): Migration =
  var content = m.version & m.name
  for s in m.up: content &= s.sql
  var h = 0u64
  for c in content:
    h = h * 31u64 + uint64(ord(c))
  m.checksum = fmt"{h:016x}"
  return m

# ============================
# Define migrations
# ============================

proc allMigrations(): seq[Migration] =
  var migrations: seq[Migration]

  block:
    var m = migration("001", "create_users_table", "alice")
    m.addUp(
      "CREATE TABLE users (id SERIAL PRIMARY KEY, email VARCHAR(255) UNIQUE NOT NULL, name VARCHAR(255) NOT NULL, created_at TIMESTAMP NOT NULL DEFAULT NOW(), updated_at TIMESTAMP NOT NULL DEFAULT NOW())",
      "Create users table"
    )
    m.addUp(
      "CREATE INDEX idx_users_email ON users(email)",
      "Index on email"
    )
    m.addDown(
      "DROP INDEX IF EXISTS idx_users_email",
      "Drop email index"
    )
    m.addDown(
      "DROP TABLE IF EXISTS users",
      "Drop users table"
    )
    migrations.add(withChecksum(m))

  block:
    var m = migration("002", "create_posts_table", "bob")
    m.addUp(
      "CREATE TABLE posts (id SERIAL PRIMARY KEY, user_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE, title VARCHAR(500) NOT NULL, content TEXT, status VARCHAR(50) NOT NULL DEFAULT 'draft', created_at TIMESTAMP NOT NULL DEFAULT NOW(), updated_at TIMESTAMP NOT NULL DEFAULT NOW())",
      "Create posts table"
    )
    m.addUp(
      "CREATE INDEX idx_posts_user_id ON posts(user_id)",
      "Index on user_id"
    )
    m.addUp(
      "CREATE INDEX idx_posts_status ON posts(status)",
      "Index on status"
    )
    m.addDown("DROP INDEX IF EXISTS idx_posts_status", "Drop status index")
    m.addDown("DROP INDEX IF EXISTS idx_posts_user_id", "Drop user_id index")
    m.addDown("DROP TABLE IF EXISTS posts", "Drop posts table")
    migrations.add(withChecksum(m))

  block:
    var m = migration("003", "add_user_profile_fields", "charlie")
    m.addUp(
      "ALTER TABLE users ADD COLUMN bio TEXT",
      "Add bio column"
    )
    m.addUp(
      "ALTER TABLE users ADD COLUMN avatar_url VARCHAR(1000)",
      "Add avatar_url column"
    )
    m.addUp(
      "ALTER TABLE users ADD COLUMN role VARCHAR(50) NOT NULL DEFAULT 'user'",
      "Add role column"
    )
    m.addDown("ALTER TABLE users DROP COLUMN IF EXISTS role", "Drop role")
    m.addDown("ALTER TABLE users DROP COLUMN IF EXISTS avatar_url", "Drop avatar_url")
    m.addDown("ALTER TABLE users DROP COLUMN IF EXISTS bio", "Drop bio")
    migrations.add(withChecksum(m))

  block:
    var m = migration("004", "create_sessions_table", "alice")
    m.addUp("""
CREATE TABLE sessions (
  id VARCHAR(128) PRIMARY KEY,
  user_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  data JSONB,
  ip_address VARCHAR(45),
  user_agent TEXT,
  last_activity TIMESTAMP NOT NULL DEFAULT NOW(),
  expires_at TIMESTAMP NOT NULL,
  created_at TIMESTAMP NOT NULL DEFAULT NOW()
)""", "Create sessions table")
    m.addUp(
      "CREATE INDEX idx_sessions_user_id ON sessions(user_id)",
      "Index on user_id"
    )
    m.addUp(
      "CREATE INDEX idx_sessions_expires_at ON sessions(expires_at)",
      "Index on expires_at for cleanup"
    )
    m.addDown("DROP INDEX IF EXISTS idx_sessions_expires_at", "Drop expires index")
    m.addDown("DROP INDEX IF EXISTS idx_sessions_user_id", "Drop user_id index")
    m.addDown("DROP TABLE IF EXISTS sessions", "Drop sessions table")
    migrations.add(withChecksum(m))

  block:
    var m = migration("005", "add_soft_delete", "diana")
    m.addUp(
      "ALTER TABLE users ADD COLUMN deleted_at TIMESTAMP",
      "Add soft delete to users"
    )
    m.addUp(
      "ALTER TABLE posts ADD COLUMN deleted_at TIMESTAMP",
      "Add soft delete to posts"
    )
    m.addUp(
      "CREATE INDEX idx_users_deleted_at ON users(deleted_at) WHERE deleted_at IS NOT NULL",
      "Partial index for deleted users"
    )
    m.addDown("DROP INDEX IF EXISTS idx_users_deleted_at", "Drop deleted index")
    m.addDown("ALTER TABLE posts DROP COLUMN IF EXISTS deleted_at", "Drop deleted_at from posts")
    m.addDown("ALTER TABLE users DROP COLUMN IF EXISTS deleted_at", "Drop deleted_at from users")
    migrations.add(withChecksum(m))

  return migrations

# ============================
# Migration runner
# ============================

proc newMigrationRunner(dryRun = false, verbose = true): MigrationRunner =
  MigrationRunner(
    migrations: allMigrations(),
    history: MigrationsTable(records: @[]),
    dryRun: dryRun,
    verbose: verbose
  )

proc isApplied(runner: MigrationRunner, version: string): bool =
  runner.history.records.anyIt(it.version == version and it.success)

proc pendingMigrations(runner: MigrationRunner): seq[Migration] =
  runner.migrations.filterIt(not runner.isApplied(it.version))

proc appliedMigrations(runner: MigrationRunner): seq[MigrationRecord] =
  runner.history.records.filterIt(it.success)

proc executeStatement(runner: MigrationRunner, stmt: SqlStatement): tuple[ok: bool, error: string] =
  if runner.verbose:
    echo fmt"    SQL: {stmt.sql[0..min(60, stmt.sql.len-1)]}{'...' if stmt.sql.len > 60 else ''}"
  if runner.dryRun:
    echo fmt"    [DRY RUN] Would execute: {stmt.description}"
    return (true, "")

  # Mock execution — in real code: db.exec(stmt.sql)
  return (true, "")

proc runMigration(runner: var MigrationRunner, m: Migration,
                  direction: MigrationDirection): tuple[ok: bool, error: string] =
  let start = epochTime()
  let stmts = if direction == mdUp: m.up else: m.down
  let dirStr = if direction == mdUp: "UP" else: "DOWN"

  echo fmt"\n[Migration {dirStr}] {m.version}: {m.name}"

  for stmt in stmts:
    if runner.verbose:
      echo fmt"  > {stmt.description}"
    let (ok, err) = executeStatement(runner, stmt)
    if not ok:
      let record = MigrationRecord(
        version: m.version,
        name: m.name,
        appliedAt: epochTime(),
        checksum: m.checksum,
        executionMs: (epochTime() - start) * 1000,
        success: false,
        error: err
      )
      runner.history.records.add(record)
      return (false, err)

  let duration = (epochTime() - start) * 1000
  if direction == mdUp:
    runner.history.records.add(MigrationRecord(
      version: m.version,
      name: m.name,
      appliedAt: epochTime(),
      checksum: m.checksum,
      executionMs: duration,
      success: true
    ))
  else:
    runner.history.records = runner.history.records.filterIt(it.version != m.version)

  echo fmt"  Done ({duration:.1f}ms)"
  return (true, "")

proc migrate(runner: var MigrationRunner, target = ""): int =
  let pending = pendingMigrations(runner)
  if pending.len == 0:
    echo "No pending migrations."
    return 0

  echo fmt"Running {pending.len} migration(s)..."
  var count = 0
  for m in pending:
    if target.len > 0 and m.version > target: break
    let (ok, err) = runMigration(runner, m, mdUp)
    if not ok:
      echo fmt"FAILED: {err}"
      break
    inc count

  echo fmt"\nApplied {count} migration(s)."
  return count

proc rollback(runner: var MigrationRunner, steps = 1): int =
  let applied = appliedMigrations(runner)
  if applied.len == 0:
    echo "No migrations to roll back."
    return 0

  let toRollback = applied[max(0, applied.len - steps)..^1].reversed()
  echo fmt"Rolling back {toRollback.len} migration(s)..."

  var count = 0
  for record in toRollback:
    let m = runner.migrations.filterIt(it.version == record.version)
    if m.len == 0: continue
    let (ok, err) = runMigration(runner, m[0], mdDown)
    if ok: inc count
    else: echo fmt"Rollback FAILED: {err}"; break

  echo fmt"Rolled back {count} migration(s)."
  return count

proc status(runner: MigrationRunner) =
  echo "\nMigration status:"
  echo fmt"  {'Version':<20} {'Name':<35} {'Status':<12} {'Applied At'}"
  echo "  " & "-".repeat(80)

  let appliedVersions = runner.history.records.filterIt(it.success)
    .mapIt(it.version)

  for m in runner.migrations:
    let applied = m.version in appliedVersions
    let statusStr = if applied: "applied" else: "pending"
    let rec = runner.history.records.filterIt(it.version == m.version)
    let appliedAt = if rec.len > 0 and rec[0].success:
      let t = fromUnix(int64(rec[0].appliedAt))
      format(t, "yyyy-MM-dd HH:mm")
    else: ""
    echo fmt"  {m.version:<20} {m.name:<35} {statusStr:<12} {appliedAt}"

# ============================
# Seed data
# ============================

type
  SeedData = object
    name: string
    table: string
    rows: seq[JsonNode]
    runOrder: int

proc seed(name, table: string, rows: seq[JsonNode], order = 0): SeedData =
  SeedData(name: name, table: table, rows: rows, runOrder: order)

proc runSeed(runner: MigrationRunner, data: SeedData) =
  echo fmt"[Seed] {data.name}: inserting {data.rows.len} rows into {data.table}"
  if runner.dryRun:
    echo fmt"  [DRY RUN] Would insert {data.rows.len} rows"
    return
  for row in data.rows:
    echo fmt"  INSERT INTO {data.table}: {($row)[0..min(60, ($row).len-1)]}..."

proc baseSeeds(): seq[SeedData] =
  @[
    seed("admin_user", "users", @[
      %*{"name": "Admin", "email": "admin@example.com", "role": "admin"},
    ], order = 1),
    seed("sample_users", "users", @[
      %*{"name": "Alice", "email": "alice@example.com", "role": "user"},
      %*{"name": "Bob", "email": "bob@example.com", "role": "user"},
      %*{"name": "Charlie", "email": "charlie@example.com", "role": "moderator"},
    ], order = 2),
    seed("sample_posts", "posts", @[
      %*{"user_id": 2, "title": "Hello World", "content": "First post!", "status": "published"},
      %*{"user_id": 2, "title": "Nim is Fast", "content": "Learning Nim...", "status": "published"},
      %*{"user_id": 3, "title": "Draft Post", "content": "Work in progress", "status": "draft"},
    ], order = 3),
  ]

# ============================
# Demo
# ============================

proc demo() =
  echo "=== Database Migrations Demo ==="

  var runner = newMigrationRunner(dryRun = false, verbose = true)

  # Show initial status
  echo "\n--- Initial status ---"
  status(runner)

  # Run all migrations
  echo "\n--- Running migrations ---"
  let applied = migrate(runner)
  echo fmt"Applied: {applied}"

  # Show status after
  echo "\n--- Status after migrate ---"
  status(runner)

  # Try to migrate again (should be no-op)
  echo "\n--- Re-run (should be no-op) ---"
  let reapplied = migrate(runner)
  echo fmt"Applied again: {reapplied}"

  # Rollback 2 steps
  echo "\n--- Rollback 2 steps ---"
  let rolledBack = rollback(runner, 2)
  echo fmt"Rolled back: {rolledBack}"

  echo "\n--- Status after rollback ---"
  status(runner)

  # Re-apply
  echo "\n--- Re-migrate ---"
  discard migrate(runner)

  # Dry run
  echo "\n--- Dry run ---"
  var dryRunner = newMigrationRunner(dryRun = true)
  discard migrate(dryRunner)

  # Seeds
  echo "\n--- Seed data ---"
  let seeds = baseSeeds().sortedByIt(it.runOrder)
  for s in seeds:
    runSeed(runner, s)

demo()
```

---

## 📝 สรุป Part 67

| Steps | หัวข้อ |
|-------|--------|
| 976 | Migration types, up/down SQL statements |
| 977-985 | Migration runner, apply/rollback, checksums |
| 986-990 | Migration status, seed data management, dry run |

---

**← [Part 66: Tracing](part_66_tracing.md) | [Part 68: Background Jobs →](part_68_background_jobs.md)**
