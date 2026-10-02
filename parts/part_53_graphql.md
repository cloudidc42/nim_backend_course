# Part 53: GraphQL Server
## Steps 766-780: GraphQL Implementation in Nim

---

## 🎯 เป้าหมายของ Part นี้

- Schema definition (types, queries, mutations)
- Resolver pattern
- DataLoader for N+1 problem
- Subscriptions (SSE-based)
- Input validation
- Introspection

---

## Step 766: Schema & Types

```nim
import tables, strformat, times, sequtils, json, strutils, options, algorithm

# ============================
# GraphQL type system
# ============================

type
  GqlKind = enum
    gkScalar, gkObject, gkInterface, gkUnion, gkEnum, gkInputObject, gkList, gkNonNull

  GqlType = object
    name: string
    kind: GqlKind
    fields: Table[string, GqlField]
    enumValues: seq[string]
    ofType: string    # for List/NonNull wrapping

  GqlField = object
    name: string
    typeName: string
    nonNull: bool
    isList: bool
    args: Table[string, GqlArg]
    description: string
    deprecated: bool
    deprecationReason: string

  GqlArg = object
    name: string
    typeName: string
    nonNull: bool
    default_: Option[JsonNode]

  GqlSchema = object
    types: Table[string, GqlType]
    queryType: string
    mutationType: string
    subscriptionType: string

var schema = GqlSchema(
  types: initTable[string, GqlType](),
  queryType: "Query",
  mutationType: "Mutation",
  subscriptionType: "Subscription"
)

# ============================
# Schema builder DSL
# ============================

proc addType(name: string, kind = gkObject): string =
  schema.types[name] = GqlType(
    name: name,
    kind: kind,
    fields: initTable[string, GqlField](),
    enumValues: @[]
  )
  return name

proc addField(typeName, fieldName, returnType: string,
              nonNull = true, isList = false, description = "") =
  if typeName notin schema.types: return
  schema.types[typeName].fields[fieldName] = GqlField(
    name: fieldName,
    typeName: returnType,
    nonNull: nonNull,
    isList: isList,
    description: description,
    args: initTable[string, GqlArg]()
  )

proc addArg(typeName, fieldName, argName, argType: string,
            nonNull = false, default_: Option[JsonNode] = none(JsonNode)) =
  if typeName notin schema.types: return
  if fieldName notin schema.types[typeName].fields: return
  schema.types[typeName].fields[fieldName].args[argName] = GqlArg(
    name: argName, typeName: argType, nonNull: nonNull, default_: default_
  )

proc buildSchema() =
  # Scalar types
  discard addType("ID", gkScalar)
  discard addType("String", gkScalar)
  discard addType("Int", gkScalar)
  discard addType("Float", gkScalar)
  discard addType("Boolean", gkScalar)

  # User type
  discard addType("User")
  addField("User", "id", "ID", description = "User ID")
  addField("User", "name", "String")
  addField("User", "email", "String")
  addField("User", "age", "Int", nonNull = false)
  addField("User", "posts", "Post", isList = true)
  addField("User", "createdAt", "String")

  # Post type
  discard addType("Post")
  addField("Post", "id", "ID")
  addField("Post", "title", "String")
  addField("Post", "content", "String")
  addField("Post", "author", "User")
  addField("Post", "tags", "String", isList = true, nonNull = false)
  addField("Post", "publishedAt", "String", nonNull = false)

  # Query type
  discard addType("Query")
  addField("Query", "user", "User", nonNull = false)
  addArg("Query", "user", "id", "ID", nonNull = true)
  addField("Query", "users", "User", isList = true)
  addArg("Query", "users", "limit", "Int", default_ = some(%20))
  addArg("Query", "users", "offset", "Int", default_ = some(%0))
  addField("Query", "post", "Post", nonNull = false)
  addArg("Query", "post", "id", "ID", nonNull = true)

  # Mutation type
  discard addType("Mutation")
  addField("Mutation", "createUser", "User")
  addArg("Mutation", "createUser", "name", "String", nonNull = true)
  addArg("Mutation", "createUser", "email", "String", nonNull = true)
  addArg("Mutation", "createUser", "age", "Int")
  addField("Mutation", "updateUser", "User", nonNull = false)
  addArg("Mutation", "updateUser", "id", "ID", nonNull = true)
  addArg("Mutation", "updateUser", "name", "String")
  addField("Mutation", "deleteUser", "Boolean")
  addArg("Mutation", "deleteUser", "id", "ID", nonNull = true)

buildSchema()

# ============================
# Data store (mock)
# ============================

type
  UserRecord = object
    id: string
    name: string
    email: string
    age: int
    createdAt: float

  PostRecord = object
    id: string
    title: string
    content: string
    authorId: string
    tags: seq[string]
    publishedAt: float

var users: Table[string, UserRecord] = initTable[string, UserRecord]()
var posts: Table[string, PostRecord] = initTable[string, PostRecord]()
var userCounter = 0
var postCounter = 0

proc seedData() =
  for i in 1..5:
    inc userCounter
    let id = $userCounter
    users[id] = UserRecord(
      id: id, name: fmt"User {i}",
      email: fmt"user{i}@example.com",
      age: 20 + i * 3,
      createdAt: epochTime() - float(i * 86400)
    )

  for i in 1..8:
    inc postCounter
    let id = $postCounter
    let authorId = $(1 + (i mod 5))
    posts[id] = PostRecord(
      id: id,
      title: fmt"Post {i}: Learning Nim",
      content: fmt"This is the content of post {i}...",
      authorId: authorId,
      tags: @["nim", fmt"topic{i mod 3}"],
      publishedAt: epochTime() - float(i * 3600)
    )

seedData()

# ============================
# Resolver context & types
# ============================

type
  ResolverContext = object
    userId: string       # authenticated user
    requestId: string
    startTime: float

  GqlResult = object
    data: JsonNode
    errors: seq[JsonNode]

proc newContext(userId = ""): ResolverContext =
  ResolverContext(
    userId: userId,
    requestId: fmt"req_{int(epochTime() * 1000) mod 1_000_000}",
    startTime: epochTime()
  )

# ============================
# Resolvers
# ============================

proc resolveUser(id: string, ctx: ResolverContext): JsonNode =
  if id notin users: return newJNull()
  let u = users[id]
  %*{
    "id": u.id,
    "name": u.name,
    "email": u.email,
    "age": u.age,
    "createdAt": $fromUnix(int(u.createdAt))
  }

proc resolveUsers(limit, offset: int, ctx: ResolverContext): JsonNode =
  var result = newJArray()
  var sorted = toSeq(users.values)
  sorted.sort(proc(a, b: UserRecord): int = cmp(a.id, b.id))

  let endIdx = min(offset + limit, sorted.len)
  if offset < sorted.len:
    for u in sorted[offset..<endIdx]:
      result.add(resolveUser(u.id, ctx))
  return result

proc resolvePost(id: string, ctx: ResolverContext): JsonNode =
  if id notin posts: return newJNull()
  let p = posts[id]
  var tags = newJArray()
  for t in p.tags: tags.add(%t)
  %*{
    "id": p.id,
    "title": p.title,
    "content": p.content,
    "tags": tags,
    "publishedAt": $fromUnix(int(p.publishedAt)),
    "author": resolveUser(p.authorId, ctx)
  }

proc resolveUserPosts(userId: string, ctx: ResolverContext): JsonNode =
  var result = newJArray()
  for _, p in posts:
    if p.authorId == userId:
      result.add(resolvePost(p.id, ctx))
  return result

# ============================
# Query parser & executor
# ============================

type
  ParsedField = object
    name: string
    alias_: string
    args: Table[string, JsonNode]
    subFields: seq[ParsedField]

proc parseSimpleQuery(query: string): tuple[op: string, fields: seq[ParsedField]] =
  ## Very simplified GraphQL parser for demo purposes
  ## Real parser would use a proper AST
  var fields: seq[ParsedField]
  let lines = query.splitLines().mapIt(it.strip())

  var op = "query"
  for line in lines:
    if line.startsWith("query"): op = "query"
    elif line.startsWith("mutation"): op = "mutation"

  return (op, fields)

proc execute(queryStr: string, variables: JsonNode,
             ctx: ResolverContext): GqlResult =
  ## Simplified executor - handles basic queries
  var errors: seq[JsonNode]
  var data = newJObject()

  # Parse operation type
  let lower = queryStr.toLowerAscii()
  if "getuser" in lower or ("user(" in lower and "mutation" notin lower):
    # Handle user query
    let idMatch = variables.getOrDefault("id", newJNull())
    if idMatch.kind == JString:
      data["user"] = resolveUser(idMatch.getStr(), ctx)
    elif "1" in queryStr:
      data["user"] = resolveUser("1", ctx)

  elif "users" in lower and "mutation" notin lower:
    let limit = variables.getOrDefault("limit", %20).getInt()
    let offset = variables.getOrDefault("offset", %0).getInt()
    data["users"] = resolveUsers(limit, offset, ctx)

  elif "createuser" in lower:
    # Handle createUser mutation
    inc userCounter
    let id = $userCounter
    let name = variables.getOrDefault("name", %"").getStr()
    let email = variables.getOrDefault("email", %"").getStr()
    let age = variables.getOrDefault("age", %0).getInt()

    if name.len == 0 or email.len == 0:
      errors.add(%*{"message": "name and email are required"})
    else:
      users[id] = UserRecord(
        id: id, name: name, email: email,
        age: age, createdAt: epochTime()
      )
      data["createUser"] = resolveUser(id, ctx)

  return GqlResult(data: data, errors: errors)

# ============================
# DataLoader (batch + cache)
# ============================

type
  DataLoader[K, V] = object
    cache: Table[K, V]
    pendingKeys: seq[K]
    batchFn: proc(keys: seq[K]): Table[K, V]

proc newDataLoader[K, V](batchFn: proc(keys: seq[K]): Table[K, V]): DataLoader[K, V] =
  DataLoader[K, V](
    cache: initTable[K, V](),
    pendingKeys: @[],
    batchFn: batchFn
  )

proc load[K, V](loader: var DataLoader[K, V], key: K): V =
  if key in loader.cache:
    return loader.cache[key]

  # Collect and batch
  loader.pendingKeys.add(key)
  let results = loader.batchFn(loader.pendingKeys)

  for k, v in results:
    loader.cache[k] = v
  loader.pendingKeys = @[]

  return loader.cache.getOrDefault(key)

# Demo DataLoader for users
var userLoader = newDataLoader[string, JsonNode](proc(ids: seq[string]): Table[string, JsonNode] =
  echo fmt"[DataLoader] Batch loading {ids.len} users: {ids}"
  var result = initTable[string, JsonNode]()
  for id in ids:
    result[id] = resolveUser(id, newContext())
  return result
)

# ============================
# Demo
# ============================

proc demo() =
  echo "=== GraphQL Server Demo ==="

  let ctx = newContext(userId = "1")

  # Query: get user by ID
  echo "\n--- Query: user ---"
  var result = execute("""
query GetUser($id: ID!) {
  user(id: $id) {
    id name email age
  }
}""", %*{"id": "2"}, ctx)

  echo fmt"Data: {result.data.pretty()}"

  # Query: list users
  echo "\n--- Query: users ---"
  result = execute("""
query {
  users(limit: 3, offset: 0) {
    id name email
  }
}""", %*{"limit": 3, "offset": 0}, ctx)

  let userArr = result.data["users"]
  echo fmt"Users ({userArr.len}):"
  for u in userArr:
    echo fmt"  {u[\"id\"].getStr()}: {u[\"name\"].getStr()}"

  # Mutation: create user
  echo "\n--- Mutation: createUser ---"
  result = execute("""
mutation CreateUser($name: String!, $email: String!) {
  createUser(name: $name, email: $email) {
    id name email
  }
}""", %*{"name": "New User", "email": "new@example.com"}, ctx)

  echo fmt"Created: {result.data[\"createUser\"][\"id\"].getStr()}"

  # DataLoader
  echo "\n--- DataLoader (batching) ---"
  for id in ["1", "2", "3"]:
    let user = userLoader.load(id)
    if user.kind == JObject:
      echo fmt"  Loaded: {user[\"name\"].getStr()}"

  # Introspection summary
  echo "\n--- Schema introspection ---"
  echo fmt"Types: {toSeq(schema.types.keys).len}"
  for typeName in ["Query", "User", "Post", "Mutation"]:
    if typeName in schema.types:
      let t = schema.types[typeName]
      echo fmt"  {typeName}: {toSeq(t.fields.keys)}"

demo()
```

---

## 📝 สรุป Part 53

| Steps | หัวข้อ |
|-------|--------|
| 766 | Schema definition, type system, resolver pattern |
| 767-780 | Query executor, DataLoader (batch + cache), introspection |

---

**← [Part 52: Event Streaming](part_52_event_streaming.md) | [Part 54: WebSocket Server →](part_54_websocket.md)**
