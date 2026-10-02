# Part 37: Database Migrations
## Steps 526-540: Migration System ระดับ Production

---

## 🎯 เป้าหมายของ Part นี้

- Migration system แบบ version-controlled
- Up/down migrations
- Schema diff & auto-generate
- Seeding
- Multi-database support
- CI/CD integration

---

## Step 526: Migration Framework

```nim
import os, strformat, strutils, times, sequtils, algorithm, json

# ============================
# Migration types
# ============================

type
  MigrationDirection = enum
    mdUp, mdDown

  Migration = object
    version: int       # timestamp e.g. 20240101120000
    name: string
    upSql: string
    downSql: string
    checksum: string

  MigrationRecord = object
    version: int
    name: string
    appliedAt: float
    checksum: string
    executionMs: int

  MigrationError = object of CatchableError

proc parseMigrationFile(path: string): Migration =
  let content = readFile(path)
  let lines = content.splitLines()
  
  var upLines, downLines: seq[string]
  var inUp = false
  var inDown = false
  
  for line in lines:
    if line.strip() == "-- +migrate Up":
      inUp = true
      inDown = false
    elif line.strip() == "-- +migrate Down":
      inUp = false
      inDown = true
    elif inUp:
      upLines.add(line)
    elif inDown:
      downLines.add(line)
  
  # Extract version and name from filename: 20240101120000_create_users.sql
  let filename = path.extractFilename()
  let parts = filename.splitFile().name.split('_', 1)
  
  let version = parts[0].parseInt()
  let name = if parts.len > 1: parts[1] else: ""
  
  # Simple checksum
  let checksum = $content.len  # in production use SHA-256
  
  return Migration(
    version: version,
    name: name,
    upSql: upLines.join("\n"),
    downSql: downLines.join("\n"),
    checksum: checksum
  )

proc loadMigrations(dir: string): seq[Migration] =
  var migrations: seq[Migration]
  
  if not dirExists(dir):
    return migrations
  
  for file in walkFiles(dir / "*.sql"):
    try:
      let m = parseMigrationFile(file)
      migrations.add(m)
    except:
      echo fmt"Warning: Could not parse {file}: {getCurrentExceptionMsg()}"
  
  # Sort by version ascending
  migrations.sort(proc(a, b: Migration): int = cmp(a.version, b.version))
  return migrations

# ============================
# Migration runner (abstract DB)
# ============================

type
  DbConn = object
    url: string
    # In real implementation this would be a db handle

proc execSql(conn: DbConn, sql: string): bool =
  echo fmt"[SQL] Executing: {sql[0..min(60, sql.len-1)]}..."
  true  # mock

proc queryRows(conn: DbConn, sql: string): seq[seq[string]] =
  # mock - in real implementation use db.getAllRows
  @[]

proc createMigrationsTable(conn: DbConn) =
  discard conn.execSql("""
    CREATE TABLE IF NOT EXISTS schema_migrations (
      version      BIGINT PRIMARY KEY,
      name         TEXT NOT NULL,
      applied_at   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
      checksum     TEXT NOT NULL,
      execution_ms INT NOT NULL DEFAULT 0
    )
  """)

proc getAppliedVersions(conn: DbConn): seq[int] =
  let rows = conn.queryRows("SELECT version FROM schema_migrations ORDER BY version")
  rows.mapIt(it[0].parseInt())

proc recordMigration(conn: DbConn, m: Migration, ms: int) =
  discard conn.execSql(fmt"""
    INSERT INTO schema_migrations (version, name, checksum, execution_ms)
    VALUES ({m.version}, '{m.name}', '{m.checksum}', {ms})
  """)

proc removeMigrationRecord(conn: DbConn, version: int) =
  discard conn.execSql(fmt"DELETE FROM schema_migrations WHERE version = {version}")

# ============================
# Migrator
# ============================

type
  Migrator = object
    conn: DbConn
    migrationsDir: string
    dryRun: bool

proc newMigrator(url, dir: string, dryRun = false): Migrator =
  Migrator(conn: DbConn(url: url), migrationsDir: dir, dryRun: dryRun)

proc migrate(m: Migrator, target: int = -1) =
  m.conn.createMigrationsTable()
  
  let migrations = loadMigrations(m.migrationsDir)
  let applied = m.conn.getAppliedVersions()
  
  let pending = migrations.filterIt(it.version notin applied)
  
  if pending.len == 0:
    echo "✓ Database is up to date"
    return
  
  echo fmt"Running {pending.len} migration(s)..."
  
  for migration in pending:
    if target >= 0 and migration.version > target:
      break
    
    echo fmt"\n→ Migrating: {migration.version}_{migration.name}"
    
    if m.dryRun:
      echo "  [DRY RUN] Would execute:"
      echo migration.upSql.indent(4)
      continue
    
    let startMs = int(epochTime() * 1000)
    let ok = m.conn.execSql(migration.upSql)
    let endMs = int(epochTime() * 1000)
    
    if ok:
      m.conn.recordMigration(migration, endMs - startMs)
      echo fmt"  ✓ Applied in {endMs - startMs}ms"
    else:
      echo fmt"  ✗ Failed: {migration.name}"
      raise newException(MigrationError, fmt"Migration {migration.version} failed")

proc rollback(m: Migrator, steps: int = 1) =
  m.conn.createMigrationsTable()
  
  let migrations = loadMigrations(m.migrationsDir)
  let applied = m.conn.getAppliedVersions()
  
  let toRollback = migrations
    .filterIt(it.version in applied)
    .reversed()
    .take(steps)
  
  if toRollback.len == 0:
    echo "Nothing to rollback"
    return
  
  for migration in toRollback:
    echo fmt"← Rolling back: {migration.version}_{migration.name}"
    
    if m.dryRun:
      echo "  [DRY RUN] Would execute:"
      echo migration.downSql.indent(4)
      continue
    
    if m.conn.execSql(migration.downSql):
      m.conn.removeMigrationRecord(migration.version)
      echo fmt"  ✓ Rolled back"
    else:
      raise newException(MigrationError, fmt"Rollback {migration.version} failed")

proc status(m: Migrator) =
  m.conn.createMigrationsTable()
  
  let migrations = loadMigrations(m.migrationsDir)
  let applied = m.conn.getAppliedVersions()
  
  echo "\nMigration Status:"
  echo "=".repeat(60)
  
  for migration in migrations:
    let state = if migration.version in applied: "✓ Applied" else: "○ Pending"
    echo fmt"  [{state:10}] {migration.version}_{migration.name}"
  
  echo "=".repeat(60)
  echo fmt"  Applied: {applied.len} / {migrations.len}"

# ============================
# Demo
# ============================

proc demo() =
  echo "=== Migration Framework Demo ==="
  
  let migrator = newMigrator("postgres://localhost/myapp", "migrations/", dryRun = true)
  
  # Show status
  migrator.status()
  
  # Simulate migrations being present
  echo "\nExample migration file content:"
  echo """
-- +migrate Up
CREATE TABLE users (
  id         SERIAL PRIMARY KEY,
  email      TEXT NOT NULL UNIQUE,
  username   TEXT NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_users_email ON users(email);

-- +migrate Down
DROP TABLE IF EXISTS users;
"""

demo()
```

---

## Step 527: Migration Generator

```nim
import os, strformat, times

# ============================
# Migration file generator
# ============================

type
  MigrationType = enum
    mtCreate, mtAlter, mtDrop, mtIndex, mtData, mtCustom

  ColumnDef = object
    name: string
    type_: string
    nullable: bool
    default_: string
    unique: bool
    references: string

proc nimTypeToSql(nimType: string): string =
  case nimType.toLowerAscii()
  of "int", "integer": "INTEGER"
  of "int64", "bigint": "BIGINT"
  of "string", "text": "TEXT"
  of "float", "double": "DOUBLE PRECISION"
  of "bool", "boolean": "BOOLEAN"
  of "datetime": "TIMESTAMPTZ NOT NULL DEFAULT NOW()"
  of "uuid": "UUID NOT NULL DEFAULT gen_random_uuid()"
  else: "TEXT"

proc generateVersion(): int =
  let now = now()
  return int(now.year) * 10000000000 +
         int(now.month) * 100000000 +
         int(now.monthday) * 1000000 +
         int(now.hour) * 10000 +
         int(now.minute) * 100 +
         int(now.second)

proc generateCreateTable(tableName: string, columns: seq[ColumnDef]): tuple[up, down: string] =
  var colDefs: seq[string] = @["id SERIAL PRIMARY KEY"]
  
  for col in columns:
    var def_ = fmt"{col.name} {col.type_}"
    if not col.nullable:
      def_ &= " NOT NULL"
    if col.unique:
      def_ &= " UNIQUE"
    if col.default_.len > 0:
      def_ &= fmt" DEFAULT {col.default_}"
    if col.references.len > 0:
      def_ &= fmt" REFERENCES {col.references}"
    colDefs.add(def_)
  
  colDefs.add("created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()")
  colDefs.add("updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()")
  
  let up = fmt"""
-- +migrate Up
CREATE TABLE {tableName} (
  {colDefs.join(",\n  ")}
);

CREATE INDEX idx_{tableName}_created_at ON {tableName}(created_at);

-- +migrate Down
DROP TABLE IF EXISTS {tableName};
""".strip()
  
  let down = fmt"DROP TABLE IF EXISTS {tableName};"
  
  return (up, down)

proc generateAddColumn(tableName, colName, colType: string, nullable = false): tuple[up, down: string] =
  let notNull = if not nullable: " NOT NULL" else: ""
  return (
    fmt"""-- +migrate Up
ALTER TABLE {tableName} ADD COLUMN {colName} {colType}{notNull};

-- +migrate Down
ALTER TABLE {tableName} DROP COLUMN IF EXISTS {colName};""",
    fmt"ALTER TABLE {tableName} DROP COLUMN IF EXISTS {colName};"
  )

proc generateIndex(tableName: string, columns: seq[string], unique = false): tuple[up, down: string] =
  let indexName = fmt"idx_{tableName}_{columns.join(\"_\")}"
  let uniqueStr = if unique: "UNIQUE " else: ""
  let colList = columns.join(", ")
  
  return (
    fmt"""-- +migrate Up
CREATE {uniqueStr}INDEX {indexName} ON {tableName}({colList});

-- +migrate Down
DROP INDEX IF EXISTS {indexName};""",
    fmt"DROP INDEX IF EXISTS {indexName};"
  )

proc writeMigration(dir, name, content: string): string =
  let version = generateVersion()
  let filename = fmt"{version}_{name}.sql"
  let path = dir / filename
  
  createDir(dir)
  writeFile(path, content)
  
  return path

# ============================
# Schema diff (simplified)
# ============================

type
  TableSchema = object
    name: string
    columns: seq[tuple[name, type_: string]]
    indexes: seq[string]

proc diffSchemas(old, new_: TableSchema): seq[string] =
  var changes: seq[string]
  
  # Find new columns
  let oldCols = old.columns.mapIt(it.name)
  for col in new_.columns:
    if col.name notin oldCols:
      changes.add(fmt"ADD COLUMN {col.name} {col.type_}")
  
  # Find removed columns
  let newCols = new_.columns.mapIt(it.name)
  for col in old.columns:
    if col.name notin newCols:
      changes.add(fmt"DROP COLUMN {col.name}")
  
  return changes

# ============================
# Demo
# ============================

proc demo() =
  echo "=== Migration Generator Demo ==="
  
  # Generate CREATE TABLE migration
  let cols = @[
    ColumnDef(name: "email", type_: "TEXT", nullable: false, unique: true),
    ColumnDef(name: "username", type_: "TEXT", nullable: false),
    ColumnDef(name: "password_hash", type_: "TEXT", nullable: false),
    ColumnDef(name: "role", type_: "TEXT", nullable: false, default_: "'user'"),
    ColumnDef(name: "is_active", type_: "BOOLEAN", nullable: false, default_: "TRUE"),
  ]
  
  let (createUp, _) = generateCreateTable("users", cols)
  echo "\nGenerated CREATE TABLE migration:"
  echo createUp
  
  # Generate ADD COLUMN migration
  let (addUp, _) = generateAddColumn("users", "last_login_at", "TIMESTAMPTZ", nullable = true)
  echo "\nGenerated ADD COLUMN migration:"
  echo addUp
  
  # Generate index migration
  let (indexUp, _) = generateIndex("users", @["email", "is_active"])
  echo "\nGenerated INDEX migration:"
  echo indexUp
  
  echo fmt"\nVersion: {generateVersion()}"

demo()
```

---

## Step 528-540: Complete Migration CLI

```nim
# migrate_cli.nim - Full migration CLI tool

import os, strutils, strformat, times, json, tables, sequtils, algorithm

# ============================
# Configuration
# ============================

type
  Config = object
    databaseUrl: string
    migrationsDir: string
    seedsDir: string
    verbose: bool
    dryRun: bool

proc loadConfig(): Config =
  Config(
    databaseUrl: getEnv("DATABASE_URL", "postgres://localhost/myapp_dev"),
    migrationsDir: getEnv("MIGRATIONS_DIR", "migrations"),
    seedsDir: getEnv("SEEDS_DIR", "seeds"),
    verbose: getEnv("VERBOSE", "false") == "true",
    dryRun: getEnv("DRY_RUN", "false") == "true"
  )

# ============================
# Seed data management
# ============================

type
  Seed = object
    name: string
    sql: string
    order: int

proc loadSeeds(dir: string): seq[Seed] =
  if not dirExists(dir):
    return @[]
  
  var seeds: seq[Seed]
  var order = 0
  
  for file in walkFiles(dir / "*.sql"):
    inc order
    seeds.add(Seed(
      name: file.extractFilename(),
      sql: readFile(file),
      order: order
    ))
  
  seeds.sort(proc(a, b: Seed): int = cmp(a.order, b.order))
  return seeds

# ============================
# Mock DB for demo
# ============================

var executedSql: seq[string] = @[]

proc mockExec(sql: string): bool =
  executedSql.add(sql)
  true

# ============================
# Migration manager (full)
# ============================

type
  MigrationStatus = enum
    msPending, msApplied, msReverted, msFailed

  MigrationEntry = object
    version: int
    name: string
    upSql: string
    downSql: string
    status: MigrationStatus
    appliedAt: string
    durationMs: int

proc formatVersion(v: int): string =
  let s = $v
  if s.len == 14:
    fmt"{s[0..3]}-{s[4..5]}-{s[6..7]} {s[8..9]}:{s[10..11]}:{s[12..13]}"
  else:
    s

proc printStatusTable(entries: seq[MigrationEntry]) =
  echo "\n" & "=".repeat(80)
  echo fmt"{'Version':^20} {'Name':^30} {'Status':^10} {'Applied At':^15}"
  echo "=".repeat(80)
  
  for e in entries:
    let statusIcon = case e.status
      of msPending: "○"
      of msApplied: "✓"
      of msReverted: "←"
      of msFailed: "✗"
    
    let appliedStr = if e.appliedAt.len > 0: e.appliedAt[0..9] else: "-"
    echo fmt"  {e.version:<18} {e.name:<30} {statusIcon & \" \" & $e.status:<10} {appliedStr:<15}"
  
  echo "=".repeat(80)

# ============================
# CLI commands
# ============================

proc cmdStatus(config: Config) =
  echo fmt"[migrate] Checking database: {config.databaseUrl[0..20]}..."
  echo fmt"[migrate] Migrations dir: {config.migrationsDir}"
  
  # In real implementation, load from DB and filesystem
  let entries = @[
    MigrationEntry(version: 20240101000000, name: "create_users", status: msApplied,
                   appliedAt: "2024-01-01", durationMs: 12),
    MigrationEntry(version: 20240115000000, name: "add_user_roles", status: msApplied,
                   appliedAt: "2024-01-15", durationMs: 8),
    MigrationEntry(version: 20240201000000, name: "create_products", status: msPending,
                   durationMs: 0),
    MigrationEntry(version: 20240210000000, name: "add_product_tags", status: msPending,
                   durationMs: 0),
  ]
  
  printStatusTable(entries)
  
  let pending = entries.countIt(it.status == msPending)
  let applied = entries.countIt(it.status == msApplied)
  echo fmt"\n  Total: {entries.len} | Applied: {applied} | Pending: {pending}"

proc cmdUp(config: Config, steps: int = -1) =
  echo "[migrate] Running migrations UP..."
  
  if config.dryRun:
    echo "[migrate] DRY RUN MODE - no changes will be made"
  
  # Simulate running 2 pending migrations
  let pendingMigrations = @[
    ("20240201000000_create_products", """
CREATE TABLE products (
  id          SERIAL PRIMARY KEY,
  name        TEXT NOT NULL,
  price       NUMERIC(10,2) NOT NULL,
  stock       INT NOT NULL DEFAULT 0,
  created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
"""),
    ("20240210000000_add_product_tags", """
ALTER TABLE products ADD COLUMN tags TEXT[] NOT NULL DEFAULT '{}';
CREATE INDEX idx_products_tags ON products USING GIN(tags);
"""),
  ]
  
  let toRun = if steps > 0: pendingMigrations[0..<min(steps, pendingMigrations.len)]
              else: pendingMigrations
  
  for (name, sql) in toRun:
    echo fmt"\n→ {name}"
    if not config.dryRun:
      let ok = mockExec(sql)
      if ok:
        echo "  ✓ OK"
      else:
        echo "  ✗ FAILED"
        return
    else:
      echo "  [DRY RUN]"
      for line in sql.strip().splitLines():
        echo "    " & line
  
  echo fmt"\n✓ Applied {toRun.len} migration(s)"

proc cmdDown(config: Config, steps: int = 1) =
  echo fmt"[migrate] Rolling back {steps} migration(s)..."
  
  if config.dryRun:
    echo "[migrate] DRY RUN MODE"
  
  echo "\n← 20240115000000_add_user_roles"
  if not config.dryRun:
    discard mockExec("ALTER TABLE users DROP COLUMN IF EXISTS role;")
    echo "  ✓ Rolled back"
  else:
    echo "  [DRY RUN] Would execute: ALTER TABLE users DROP COLUMN IF EXISTS role;"

proc cmdSeed(config: Config) =
  echo "[migrate] Running seeds..."
  
  let seeds = @[
    ("01_users.sql", "INSERT INTO users (email, username) VALUES ('admin@example.com', 'admin');"),
    ("02_products.sql", "INSERT INTO products (name, price) VALUES ('Test Product', 99.99);"),
  ]
  
  for (name, sql) in seeds:
    echo fmt"→ {name}"
    if not config.dryRun:
      discard mockExec(sql)
      echo "  ✓ Seeded"
  
  echo fmt"\n✓ Ran {seeds.len} seed file(s)"

proc cmdCreate(config: Config, name: string) =
  let version = block:
    let now = now()
    int(now.year) * 10000000000 +
    int(now.month) * 100000000 +
    int(now.monthday) * 1000000 +
    int(now.hour) * 10000 +
    int(now.minute) * 100 +
    int(now.second)
  
  let filename = fmt"{version}_{name}.sql"
  let path = config.migrationsDir / filename
  
  let template_ = fmt"""-- Migration: {name}
-- Created: {now().format("yyyy-MM-dd HH:mm:ss")}

-- +migrate Up


-- +migrate Down

"""
  
  createDir(config.migrationsDir)
  writeFile(path, template_)
  echo fmt"✓ Created migration: {path}"

# ============================
# Main CLI
# ============================

proc printHelp() =
  echo """
Usage: migrate <command> [options]

Commands:
  status          Show migration status
  up [n]          Run pending migrations (n = number of steps)
  down [n]        Rollback migrations (default: 1 step)
  seed            Run seed files
  create <name>   Create new migration file

Options:
  --dry-run       Preview changes without applying
  --verbose       Show detailed output

Environment:
  DATABASE_URL    Database connection URL
  MIGRATIONS_DIR  Migrations directory (default: migrations/)
  SEEDS_DIR       Seeds directory (default: seeds/)
"""

proc main() =
  echo "=== Migration CLI Demo ==="
  
  let config = loadConfig()
  
  echo "\n--- migrate status ---"
  cmdStatus(config)
  
  echo "\n--- migrate up ---"
  cmdUp(config)
  
  echo "\n--- migrate down 1 ---"
  cmdDown(config, 1)
  
  echo "\n--- migrate seed ---"
  cmdSeed(config)
  
  echo "\n--- migrate create add_user_avatar ---"
  # cmdCreate(config, "add_user_avatar")  # would write to disk
  echo "Would create: migrations/20241001120000_add_user_avatar.sql"
  
  echo fmt"\n✓ SQL statements executed: {executedSql.len}"

main()
```

---

## 📝 สรุป Part 37

| Steps | หัวข้อ |
|-------|--------|
| 526 | Migration framework (up/down/status) |
| 527 | Migration file generator |
| 528-540 | Complete migration CLI with seeding |

---

**← [Part 36: Real-Time](part_36_realtime.md) | [Part 38: Multi-Tenancy →](part_38_multitenancy.md)**
