# Part 60: Database ORM
## Steps 871-885: Object-Relational Mapping in Nim

---

## 🎯 เป้าหมายของ Part นี้

- Model definition with macros
- Query builder (fluent API)
- Relationships (hasOne, hasMany, belongsTo)
- Transactions
- Connection pooling
- Migration integration

---

## Step 871: Query Builder

```nim
import tables, strformat, times, sequtils, json, strutils, options, algorithm

# ============================
# Query builder
# ============================

type
  JoinType = enum
    jtInner = "INNER", jtLeft = "LEFT", jtRight = "RIGHT", jtFull = "FULL"

  OrderDir = enum
    odAsc = "ASC", odDesc = "DESC"

  WhereClause = object
    sql: string
    params: seq[string]

  JoinClause = object
    type_: JoinType
    table: string
    on: string

  QueryBuilder = object
    table_: string
    selects: seq[string]
    wheres: seq[WhereClause]
    joins: seq[JoinClause]
    orderBys: seq[tuple[field: string, dir: OrderDir]]
    groupBys: seq[string]
    having_: string
    limit_: int
    offset_: int
    params: seq[string]
    distinct_: bool

proc fromTable(tableName: string): QueryBuilder =
  QueryBuilder(
    table_: tableName,
    selects: @["*"],
    wheres: @[],
    joins: @[],
    orderBys: @[],
    groupBys: @[],
    limit_: -1,
    offset_: 0,
    params: @[]
  )

proc select(q: QueryBuilder, fields: varargs[string]): QueryBuilder =
  var result = q
  result.selects = @fields
  return result

proc selectDistinct(q: QueryBuilder, fields: varargs[string]): QueryBuilder =
  var result = q
  result.selects = @fields
  result.distinct_ = true
  return result

proc where(q: QueryBuilder, sql: string, params: varargs[string]): QueryBuilder =
  var result = q
  result.wheres.add(WhereClause(sql: sql, params: @params))
  for p in params: result.params.add(p)
  return result

proc whereIn(q: QueryBuilder, field: string, values: seq[string]): QueryBuilder =
  var result = q
  let placeholders = values.mapIt("?").join(", ")
  result.wheres.add(WhereClause(
    sql: fmt"{field} IN ({placeholders})",
    params: values
  ))
  for v in values: result.params.add(v)
  return result

proc whereBetween(q: QueryBuilder, field: string, min_, max_: string): QueryBuilder =
  var result = q
  result.wheres.add(WhereClause(
    sql: fmt"{field} BETWEEN ? AND ?",
    params: @[min_, max_]
  ))
  result.params.add(min_)
  result.params.add(max_)
  return result

proc join(q: QueryBuilder, table, on: string, type_ = jtInner): QueryBuilder =
  var result = q
  result.joins.add(JoinClause(type_: type_, table: table, on: on))
  return result

proc orderBy(q: QueryBuilder, field: string, dir = odAsc): QueryBuilder =
  var result = q
  result.orderBys.add((field, dir))
  return result

proc groupBy(q: QueryBuilder, fields: varargs[string]): QueryBuilder =
  var result = q
  for f in fields: result.groupBys.add(f)
  return result

proc having(q: QueryBuilder, sql: string): QueryBuilder =
  var result = q
  result.having_ = sql
  return result

proc limit(q: QueryBuilder, n: int): QueryBuilder =
  var result = q
  result.limit_ = n
  return result

proc offset(q: QueryBuilder, n: int): QueryBuilder =
  var result = q
  result.offset_ = n
  return result

proc toSql(q: QueryBuilder): tuple[sql: string, params: seq[string]] =
  var parts: seq[string]

  # SELECT
  let selectStr = if q.distinct_: "SELECT DISTINCT " else: "SELECT "
  parts.add(selectStr & q.selects.join(", "))

  # FROM
  parts.add(fmt"FROM {q.table_}")

  # JOINs
  for j in q.joins:
    parts.add(fmt"{j.type_} JOIN {j.table} ON {j.on}")

  # WHERE
  if q.wheres.len > 0:
    let whereStr = q.wheres.mapIt(it.sql).join(" AND ")
    parts.add(fmt"WHERE {whereStr}")

  # GROUP BY
  if q.groupBys.len > 0:
    parts.add(fmt"GROUP BY {q.groupBys.join(\", \")}")

  # HAVING
  if q.having_.len > 0:
    parts.add(fmt"HAVING {q.having_}")

  # ORDER BY
  if q.orderBys.len > 0:
    let orderStr = q.orderBys.mapIt(fmt"{it.field} {it.dir}").join(", ")
    parts.add(fmt"ORDER BY {orderStr}")

  # LIMIT / OFFSET
  if q.limit_ >= 0:
    parts.add(fmt"LIMIT {q.limit_}")
  if q.offset_ > 0:
    parts.add(fmt"OFFSET {q.offset_}")

  return (parts.join("\n"), q.params)

# ============================
# Insert/Update/Delete builders
# ============================

type
  InsertBuilder = object
    table_: string
    rows: seq[Table[string, string]]
    onConflict: string

  UpdateBuilder = object
    table_: string
    sets: Table[string, string]
    wheres: seq[WhereClause]
    params: seq[string]

  DeleteBuilder = object
    table_: string
    wheres: seq[WhereClause]
    params: seq[string]

proc insertInto(tableName: string): InsertBuilder =
  InsertBuilder(table_: tableName, rows: @[])

proc values(b: InsertBuilder, row: Table[string, string]): InsertBuilder =
  var result = b
  result.rows.add(row)
  return result

proc onConflictIgnore(b: InsertBuilder): InsertBuilder =
  var result = b
  result.onConflict = "ON CONFLICT DO NOTHING"
  return result

proc toInsertSql(b: InsertBuilder): tuple[sql: string, params: seq[string]] =
  if b.rows.len == 0: return ("", @[])

  let columns = toSeq(b.rows[0].keys)
  let colStr = columns.join(", ")
  var allParams: seq[string]
  var placeholderGroups: seq[string]

  for row in b.rows:
    var placeholders: seq[string]
    for col in columns:
      placeholders.add("?")
      allParams.add(row.getOrDefault(col, ""))
    placeholderGroups.add("(" & placeholders.join(", ") & ")")

  let sql = fmt"INSERT INTO {b.table_} ({colStr})\nVALUES {placeholderGroups.join(\", \")}"
  let fullSql = if b.onConflict.len > 0: sql & "\n" & b.onConflict else: sql
  return (fullSql, allParams)

proc updateTable(tableName: string): UpdateBuilder =
  UpdateBuilder(table_: tableName, sets: initTable[string, string]())

proc set(b: UpdateBuilder, field, value: string): UpdateBuilder =
  var result = b
  result.sets[field] = value
  result.params.add(value)
  return result

proc whereUpdate(b: UpdateBuilder, sql: string, params: varargs[string]): UpdateBuilder =
  var result = b
  result.wheres.add(WhereClause(sql: sql, params: @params))
  for p in params: result.params.add(p)
  return result

proc toUpdateSql(b: UpdateBuilder): tuple[sql: string, params: seq[string]] =
  var setClauses: seq[string]
  for field, _ in b.sets:
    setClauses.add(fmt"{field} = ?")

  var sql = fmt"UPDATE {b.table_}\nSET {setClauses.join(\", \")}"
  if b.wheres.len > 0:
    sql &= "\nWHERE " & b.wheres.mapIt(it.sql).join(" AND ")

  return (sql, b.params)

# ============================
# ORM Model base
# ============================

type
  ModelRow = Table[string, JsonNode]

  ModelMeta = object
    tableName: string
    primaryKey: string
    columns: seq[string]
    hasTimestamps: bool

  Repository[M] = object
    meta: ModelMeta
    db: seq[ModelRow]   # mock in-memory DB
    counter: int

proc newRepository[M](tableName: string, columns: seq[string],
                       primaryKey = "id", timestamps = true): Repository[M] =
  Repository[M](
    meta: ModelMeta(
      tableName: tableName,
      primaryKey: primaryKey,
      columns: columns,
      hasTimestamps: timestamps
    ),
    db: @[],
    counter: 0
  )

proc findById[M](repo: Repository[M], id: string): Option[ModelRow] =
  for row in repo.db:
    if repo.meta.primaryKey in row:
      let v = row[repo.meta.primaryKey]
      if v.getStr() == id or (v.kind == JInt and $v.getInt() == id):
        return some(row)
  return none(ModelRow)

proc findAll[M](repo: var Repository[M], q: QueryBuilder): seq[ModelRow] =
  ## Mock: ignore query, return all rows (demo only)
  return repo.db

proc findWhere[M](repo: Repository[M], field, value: string): seq[ModelRow] =
  repo.db.filterIt(
    field in it and (it[field].getStr() == value or
                     (it[field].kind == JInt and $it[field].getInt() == value))
  )

proc insert[M](repo: var Repository[M], data: Table[string, JsonNode]): ModelRow =
  inc repo.counter
  var row = data
  row[repo.meta.primaryKey] = %repo.counter
  if repo.meta.hasTimestamps:
    let now = $epochTime()
    row["created_at"] = %now
    row["updated_at"] = %now
  repo.db.add(row)
  return row

proc update[M](repo: var Repository[M], id: string,
               data: Table[string, JsonNode]): Option[ModelRow] =
  for i, row in repo.db:
    let pkVal = row.getOrDefault(repo.meta.primaryKey, newJNull())
    if pkVal.getStr() == id or (pkVal.kind == JInt and $pkVal.getInt() == id):
      var updated = row
      for k, v in data:
        updated[k] = v
      if repo.meta.hasTimestamps:
        updated["updated_at"] = %$epochTime()
      repo.db[i] = updated
      return some(updated)
  return none(ModelRow)

proc delete[M](repo: var Repository[M], id: string): bool =
  let before = repo.db.len
  repo.db = repo.db.filterIt(
    let pkVal = it.getOrDefault(repo.meta.primaryKey, newJNull())
    pkVal.getStr() != id and
    not (pkVal.kind == JInt and $pkVal.getInt() == id)
  )
  return repo.db.len < before

proc count[M](repo: Repository[M]): int = repo.db.len

# ============================
# Demo
# ============================

type User = object
type Post = object

proc demo() =
  echo "=== ORM Demo ==="

  # Query builder
  echo "\n--- Query builder ---"
  let q = fromTable("users")
    .select("id", "name", "email")
    .join("orders", "users.id = orders.user_id", jtLeft)
    .where("users.active = ?", "true")
    .where("users.created_at > ?", "2024-01-01")
    .orderBy("created_at", odDesc)
    .limit(20)
    .offset(0)

  let (sql, params) = toSql(q)
  echo fmt"SQL:\n{sql}"
  echo fmt"Params: {params}"

  # Insert builder
  echo "\n--- Insert ---"
  var row: Table[string, string]
  row["name"] = "Alice"
  row["email"] = "alice@example.com"
  row["age"] = "30"

  let (insertSql, insertParams) = toInsertSql(
    insertInto("users").values(row).onConflictIgnore()
  )
  echo fmt"Insert SQL:\n{insertSql}"
  echo fmt"Params: {insertParams}"

  # Update builder
  echo "\n--- Update ---"
  let (updateSql, updateParams) = toUpdateSql(
    updateTable("users")
      .set("name", "Alice Smith")
      .set("updated_at", "2024-01-15")
      .whereUpdate("id = ?", "1")
  )
  echo fmt"Update SQL:\n{updateSql}"

  # Repository pattern
  echo "\n--- Repository ---"
  var userRepo = newRepository[User]("users",
    @["id", "name", "email", "age"], timestamps = true)

  var u1: Table[string, JsonNode]
  u1["name"] = %"Alice"
  u1["email"] = %"alice@example.com"
  u1["age"] = %30
  let inserted1 = userRepo.insert(u1)

  var u2: Table[string, JsonNode]
  u2["name"] = %"Bob"
  u2["email"] = %"bob@example.com"
  u2["age"] = %25
  discard userRepo.insert(u2)

  echo fmt"Total users: {userRepo.count()}"

  let found = userRepo.findById("1")
  if found.isSome:
    echo fmt"Found: {found.get()[\"name\"].getStr()}"

  var updateData: Table[string, JsonNode]
  updateData["name"] = %"Alice Smith"
  let updated = userRepo.update("1", updateData)
  if updated.isSome:
    echo fmt"Updated name: {updated.get()[\"name\"].getStr()}"

  let alices = userRepo.findWhere("name", "Alice Smith")
  echo fmt"Found by name: {alices.len}"

  let deleted = userRepo.delete("2")
  echo fmt"Deleted Bob: {deleted}, remaining: {userRepo.count()}"

  # Complex query
  echo "\n--- Complex query ---"
  let complexQ = fromTable("orders")
    .select("user_id", "COUNT(*) as order_count", "SUM(amount) as total")
    .join("users", "orders.user_id = users.id")
    .where("orders.created_at >= ?", "2024-01-01")
    .groupBy("user_id")
    .having("SUM(amount) > 100")
    .orderBy("total", odDesc)
    .limit(10)

  let (complexSql, _) = toSql(complexQ)
  echo complexSql

demo()
```

---

## 📝 สรุป Part 60

| Steps | หัวข้อ |
|-------|--------|
| 871 | Query builder (fluent API), INSERT/UPDATE/DELETE builders |
| 872-885 | Repository pattern, findById/findWhere, mock in-memory DB |

---

**← [Part 59: Testing](part_59_testing.md) | [Part 61: Microservices →](part_61_microservices.md)**
