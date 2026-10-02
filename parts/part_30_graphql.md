# Part 30: GraphQL Basics
## Steps 421-435: GraphQL Server ด้วย Nim

---

## 🎯 เป้าหมายของ Part นี้

- GraphQL concepts vs REST
- Schema Definition Language (SDL)
- Resolvers
- Queries, Mutations, Subscriptions
- DataLoader pattern (N+1 prevention)
- GraphQL over HTTP

---

## Step 421: GraphQL Concepts

```
REST vs GraphQL:

REST:                           GraphQL:
GET /users/1                    query {
GET /users/1/posts                user(id: 1) {
GET /users/1/posts/5/comments       id
                                    name
Multiple requests!                  posts {
                                      title
                                      comments {
                                        content
                                      }
                                    }
                                  }
                                }
                                One request, exactly what you need!

GraphQL Schema (SDL):
type User {
  id: ID!
  name: String!
  email: String!
  posts: [Post!]!
}

type Post {
  id: ID!
  title: String!
  content: String!
  author: User!
  tags: [String!]!
}

type Query {
  user(id: ID!): User
  posts(limit: Int, offset: Int): [Post!]!
}

type Mutation {
  createPost(input: CreatePostInput!): Post!
  updatePost(id: ID!, input: UpdatePostInput!): Post!
}
```

```nim
import json, tables, strutils, sequtils, strformat, options

# ============================
# GraphQL Type System
# ============================

type
  GqlTypeKind = enum
    Scalar, ObjectType, InputType, EnumType, ListType, NonNull

  GqlScalar = enum
    GqlString, GqlInt, GqlFloat, GqlBoolean, GqlID

  GqlValue = ref object
    case kind: GqlTypeKind
    of Scalar:
      scalar: JsonNode
    of ObjectType:
      fields: Table[string, GqlValue]
    of ListType:
      items: seq[GqlValue]
    else:
      discard

  FieldDefinition = object
    name: string
    typeName: string
    isNonNull: bool
    isList: bool
    args: seq[(string, string)]

  TypeDefinition = object
    name: string
    kind: GqlTypeKind
    fields: seq[FieldDefinition]

  ResolverFn = proc(parent: JsonNode, args: JsonNode, ctx: JsonNode): JsonNode

  Schema = object
    types: Table[string, TypeDefinition]
    resolvers: Table[string, Table[string, ResolverFn]]

proc newSchema(): Schema =
  Schema(
    types: initTable[string, TypeDefinition](),
    resolvers: initTable[string, Table[string, ResolverFn]]()
  )

proc addType(schema: var Schema, typeDef: TypeDefinition) =
  schema.types[typeDef.name] = typeDef

proc addResolver(schema: var Schema, typeName, fieldName: string, resolver: ResolverFn) =
  if typeName notin schema.resolvers:
    schema.resolvers[typeName] = initTable[string, ResolverFn]()
  schema.resolvers[typeName][fieldName] = resolver

# ============================
# Mock Data
# ============================

var mockUsers: Table[string, JsonNode] = {
  "1": %*{"id": "1", "name": "Alice", "email": "alice@example.com"},
  "2": %*{"id": "2", "name": "Bob", "email": "bob@example.com"},
  "3": %*{"id": "3", "name": "Charlie", "email": "charlie@example.com"},
}.toTable()

var mockPosts: Table[string, JsonNode] = {
  "1": %*{"id": "1", "title": "Intro to Nim", "content": "Nim is awesome...", 
          "authorId": "1", "tags": ["nim", "programming"]},
  "2": %*{"id": "2", "title": "Async Nim", "content": "Async programming...",
          "authorId": "1", "tags": ["nim", "async"]},
  "3": %*{"id": "3", "title": "Nim vs Go", "content": "Comparison...",
          "authorId": "2", "tags": ["nim", "go", "comparison"]},
}.toTable()

# ============================
# Build Schema
# ============================

var schema = newSchema()

# Type definitions
schema.addType(TypeDefinition(
  name: "User",
  kind: ObjectType,
  fields: @[
    FieldDefinition(name: "id", typeName: "ID", isNonNull: true),
    FieldDefinition(name: "name", typeName: "String", isNonNull: true),
    FieldDefinition(name: "email", typeName: "String", isNonNull: true),
    FieldDefinition(name: "posts", typeName: "Post", isNonNull: true, isList: true),
  ]
))

schema.addType(TypeDefinition(
  name: "Post",
  kind: ObjectType,
  fields: @[
    FieldDefinition(name: "id", typeName: "ID", isNonNull: true),
    FieldDefinition(name: "title", typeName: "String", isNonNull: true),
    FieldDefinition(name: "content", typeName: "String", isNonNull: true),
    FieldDefinition(name: "author", typeName: "User", isNonNull: true),
    FieldDefinition(name: "tags", typeName: "String", isNonNull: true, isList: true),
  ]
))

# Resolvers
schema.addResolver("Query", "user", proc(parent, args, ctx: JsonNode): JsonNode =
  let id = args["id"].getStr()
  if id in mockUsers:
    return mockUsers[id]
  return newJNull()
)

schema.addResolver("Query", "users", proc(parent, args, ctx: JsonNode): JsonNode =
  var arr = newJArray()
  for _, user in mockUsers:
    arr.add(user)
  return arr
)

schema.addResolver("Query", "post", proc(parent, args, ctx: JsonNode): JsonNode =
  let id = args["id"].getStr()
  if id in mockPosts:
    return mockPosts[id]
  return newJNull()
)

schema.addResolver("Query", "posts", proc(parent, args, ctx: JsonNode): JsonNode =
  let limit = args.getOrDefault("limit").getInt(10)
  var arr = newJArray()
  var count = 0
  for _, post in mockPosts:
    if count >= limit: break
    arr.add(post)
    inc count
  return arr
)

schema.addResolver("User", "posts", proc(parent, args, ctx: JsonNode): JsonNode =
  let userId = parent["id"].getStr()
  var arr = newJArray()
  for _, post in mockPosts:
    if post["authorId"].getStr() == userId:
      arr.add(post)
  return arr
)

schema.addResolver("Post", "author", proc(parent, args, ctx: JsonNode): JsonNode =
  let authorId = parent["authorId"].getStr()
  if authorId in mockUsers:
    return mockUsers[authorId]
  return newJNull()
)

echo "Schema built with", schema.types.len, "types and",
     schema.resolvers.len, "resolver groups"
```

---

## Step 422: Query Execution Engine

```nim
import json, tables, strutils, strformat, options, sequtils

# Simplified GraphQL query parser and executor

type
  SelectionField = object
    name: string
    alias: string
    args: Table[string, JsonNode]
    subFields: seq[SelectionField]

  Operation = object
    kind: string  # query, mutation
    name: string
    variables: Table[string, JsonNode]
    selections: seq[SelectionField]

proc parseSelections(tokens: seq[string], pos: var int): seq[SelectionField]

proc parseField(tokens: seq[string], pos: var int): SelectionField =
  var field = SelectionField(
    name: tokens[pos],
    alias: "",
    args: initTable[string, JsonNode](),
    subFields: @[]
  )
  inc pos
  
  # Check for alias
  if pos < tokens.len and tokens[pos] == ":":
    field.alias = field.name
    inc pos
    field.name = tokens[pos]
    inc pos
  
  # Check for arguments
  if pos < tokens.len and tokens[pos] == "(":
    inc pos
    while pos < tokens.len and tokens[pos] != ")":
      let argName = tokens[pos]
      inc pos
      if pos < tokens.len and tokens[pos] == ":":
        inc pos
      let argVal = tokens[pos]
      inc pos
      
      # Parse value
      if argVal.startsWith("\""):
        field.args[argName] = %argVal.strip(chars = {'"'})
      elif argVal.all(c => c.isDigit or c == '-'):
        field.args[argName] = %parseInt(argVal)
      elif argVal == "true":
        field.args[argName] = %true
      elif argVal == "false":
        field.args[argName] = %false
      else:
        field.args[argName] = %argVal
      
      if pos < tokens.len and tokens[pos] == ",":
        inc pos
    
    if pos < tokens.len and tokens[pos] == ")":
      inc pos
  
  # Check for sub-fields
  if pos < tokens.len and tokens[pos] == "{":
    inc pos
    field.subFields = parseSelections(tokens, pos)
  
  return field

proc parseSelections(tokens: seq[string], pos: var int): seq[SelectionField] =
  result = @[]
  while pos < tokens.len and tokens[pos] != "}":
    if tokens[pos] in ["{", "}", "(", ")", ":"]:
      inc pos
      continue
    result.add(parseField(tokens, pos))
  
  if pos < tokens.len and tokens[pos] == "}":
    inc pos

proc tokenize(query: string): seq[string] =
  var tokens: seq[string] = @[]
  var current = ""
  var inString = false
  
  for c in query:
    if c == '"':
      inString = not inString
      current.add(c)
    elif inString:
      current.add(c)
    elif c in {' ', '\n', '\t', '\r', ','}:
      if current.len > 0:
        tokens.add(current)
        current = ""
    elif c in {'{', '}', '(', ')', ':'}:
      if current.len > 0:
        tokens.add(current)
        current = ""
      tokens.add($c)
    else:
      current.add(c)
  
  if current.len > 0:
    tokens.add(current)
  
  return tokens

# Simple executor
proc executeField(
  schema: Schema,
  typeName: string,
  fieldName: string,
  field: SelectionField,
  parent: JsonNode,
  ctx: JsonNode
): JsonNode =
  # Get resolver
  var value: JsonNode
  
  if typeName in schema.resolvers and fieldName in schema.resolvers[typeName]:
    let resolver = schema.resolvers[typeName][fieldName]
    let args = % field.args
    value = resolver(parent, args, ctx)
  elif not parent.isNil and parent.kind == JObject and fieldName in parent:
    value = parent[fieldName]
  else:
    return newJNull()
  
  if value.isNil or value.kind == JNull:
    return newJNull()
  
  # No sub-selections -> return scalar
  if field.subFields.len == 0:
    return value
  
  # Has sub-selections -> resolve each
  if value.kind == JArray:
    var arr = newJArray()
    for item in value:
      var obj = newJObject()
      for subField in field.subFields:
        let subTypeName = 
          if typeName in schema.types:
            let fieldDef = schema.types[typeName].fields.filterIt(it.name == fieldName)
            if fieldDef.len > 0: fieldDef[0].typeName else: ""
          else: ""
        
        let subVal = executeField(schema, subTypeName, subField.name, subField, item, ctx)
        obj[if subField.alias.len > 0: subField.alias else: subField.name] = subVal
      arr.add(obj)
    return arr
  
  elif value.kind == JObject:
    var obj = newJObject()
    let subTypeName = 
      if typeName in schema.types:
        let fieldDef = schema.types[typeName].fields.filterIt(it.name == fieldName)
        if fieldDef.len > 0: fieldDef[0].typeName else: ""
      else: ""
    
    for subField in field.subFields:
      let subVal = executeField(schema, subTypeName, subField.name, subField, value, ctx)
      obj[if subField.alias.len > 0: subField.alias else: subField.name] = subVal
    return obj
  
  return value

proc executeQuery(schema: Schema, queryStr: string, ctx: JsonNode = nil): JsonNode =
  let tokens = tokenize(queryStr)
  var pos = 0
  
  # Skip 'query' or 'mutation'
  var opType = "query"
  if pos < tokens.len and tokens[pos] in ["query", "mutation"]:
    opType = tokens[pos]
    inc pos
  
  # Skip optional operation name
  if pos < tokens.len and tokens[pos] != "{":
    inc pos
  
  # Expect {
  if pos < tokens.len and tokens[pos] == "{":
    inc pos
  
  let selections = parseSelections(tokens, pos)
  
  var data = newJObject()
  
  for sel in selections:
    let rootType = if opType == "mutation": "Mutation" else: "Query"
    let value = executeField(schema, rootType, sel.name, sel, nil, ctx)
    data[if sel.alias.len > 0: sel.alias else: sel.name] = value
  
  return %*{"data": data}

# Test queries
echo "=== GraphQL Query Execution ==="

echo "\nQuery 1: Get user with posts"
let q1 = """
query {
  user(id: "1") {
    id
    name
    email
    posts {
      title
      tags
    }
  }
}
"""
echo executeQuery(schema, q1).pretty()

echo "\nQuery 2: List posts"
let q2 = """
query {
  posts(limit: 2) {
    id
    title
    author {
      name
    }
    tags
  }
}
"""
echo executeQuery(schema, q2).pretty()
```

---

## Step 423-435: Complete GraphQL HTTP Server

```nim
# graphql_server.nim - GraphQL HTTP endpoint

import asyncdispatch, asynchttpserver, json, tables, strutils, strformat

# ============================
# Mutations
# ============================

var nextPostId = 4

schema.addResolver("Mutation", "createPost", proc(parent, args, ctx: JsonNode): JsonNode =
  let input = args["input"]
  let postId = $nextPostId
  inc nextPostId
  
  # Get userId from context (JWT)
  let userId = if not ctx.isNil and "userId" in ctx: ctx["userId"].getStr() else: "1"
  
  let post = %*{
    "id": postId,
    "title": input["title"].getStr(),
    "content": input["content"].getStr(),
    "authorId": userId,
    "tags": input.getOrDefault("tags")
  }
  
  mockPosts[postId] = post
  return post
)

schema.addResolver("Mutation", "deletePost", proc(parent, args, ctx: JsonNode): JsonNode =
  let id = args["id"].getStr()
  if id in mockPosts:
    mockPosts.del(id)
    return %true
  return %false
)

# ============================
# Introspection (basic)
# ============================

proc introspect(schema: Schema): JsonNode =
  var types = newJArray()
  
  for name, typeDef in schema.types:
    var fields = newJArray()
    for field in typeDef.fields:
      fields.add(%*{
        "name": field.name,
        "type": {
          "name": field.typeName,
          "kind": if field.isNonNull: "NON_NULL" else: "OBJECT"
        }
      })
    
    types.add(%*{
      "name": name,
      "kind": "OBJECT",
      "fields": fields
    })
  
  return %*{
    "data": {
      "__schema": {
        "types": types,
        "queryType": {"name": "Query"},
        "mutationType": {"name": "Mutation"}
      }
    }
  }

# ============================
# GraphQL HTTP Handler
# ============================

proc handleGraphQL(req: Request) {.async.} =
  let path = req.url.path
  
  # GraphQL Playground (GET /graphql)
  if req.reqMethod == HttpGet and path == "/graphql":
    let playground = """<!DOCTYPE html>
<html>
<head>
  <title>GraphQL Playground</title>
  <meta charset="utf-8">
  <link rel="stylesheet" href="https://unpkg.com/graphql-playground-react/build/static/css/index.css">
</head>
<body>
  <div id="root"></div>
  <script src="https://unpkg.com/graphql-playground-react/build/static/js/middleware.js"></script>
  <script>
    GraphQLPlayground.init(document.getElementById('root'), { endpoint: '/graphql' })
  </script>
</body>
</html>"""
    await req.respond(Http200, playground,
      newHttpHeaders([("Content-Type", "text/html")]))
    return
  
  # GraphQL API (POST /graphql)
  if req.reqMethod == HttpPost and path == "/graphql":
    try:
      let body = parseJson(req.body)
      let query = body["query"].getStr()
      
      # Handle introspection
      if "__schema" in query or "__type" in query:
        await req.respond(Http200, $introspect(schema),
          newHttpHeaders([("Content-Type", "application/json")]))
        return
      
      # Build context from headers
      var ctx = newJObject()
      if req.headers.hasKey("Authorization"):
        let token = req.headers["Authorization"].replace("Bearer ", "")
        # In real app, decode JWT
        ctx["userId"] = %"1"
        ctx["token"] = %token
      
      # Execute
      let result = executeQuery(schema, query, ctx)
      
      await req.respond(Http200, $result,
        newHttpHeaders([("Content-Type", "application/json")]))
    
    except JsonParsingError as e:
      await req.respond(Http400,
        $(%*{"errors": [{"message": "Invalid JSON: " & e.msg}]}),
        newHttpHeaders([("Content-Type", "application/json")]))
    
    except Exception as e:
      await req.respond(Http500,
        $(%*{"errors": [{"message": e.msg}]}),
        newHttpHeaders([("Content-Type", "application/json")]))
    return
  
  await req.respond(Http404, """{"error":"not found"}""",
    newHttpHeaders([("Content-Type", "application/json")]))

# ============================
# Demo — Test queries directly
# ============================

proc demoQueries() =
  echo "=== GraphQL Server Demo ==="
  
  echo "\n1. Introspection:"
  let types = introspect(schema)["data"]["__schema"]["types"]
  for t in types:
    echo fmt"   Type: {t[\"name\"].getStr()}"
  
  echo "\n2. Query all users:"
  let usersQ = """query { users { id name email } }"""
  let usersResult = executeQuery(schema, usersQ)
  echo usersResult.pretty()
  
  echo "\n3. Mutation - Create post:"
  let createQ = """
mutation {
  createPost(input: {
    title: "New GraphQL Post"
    content: "GraphQL in Nim works!"
    tags: ["nim", "graphql"]
  }) {
    id
    title
    author {
      name
    }
  }
}
"""
  let createResult = executeQuery(schema, createQ)
  echo createResult.pretty()

demoQueries()

# To run server:
# proc main() {.async.} =
#   let server = newAsyncHttpServer()
#   echo "GraphQL server on http://localhost:8080/graphql"
#   await server.serve(Port(8080), handleGraphQL)
# waitFor main()
```

---

## 📝 สรุป Part 30

| Steps | หัวข้อ |
|-------|--------|
| 421 | GraphQL concepts vs REST |
| 422 | Query parser and execution engine |
| 423-435 | Complete GraphQL HTTP server with playground |

---

**← [Part 29: API Documentation](part_29_api_documentation.md) | [Part 31: Testing →](part_31_testing_advanced.md)**
