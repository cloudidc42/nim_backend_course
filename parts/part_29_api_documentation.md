# Part 29: API Documentation (OpenAPI/Swagger)
## Steps 406-420: สร้าง API Docs ระดับ Professional

---

## 🎯 เป้าหมายของ Part นี้

- OpenAPI 3.0 specification
- Swagger UI integration
- Auto-generate docs from code
- Request/Response examples
- API versioning
- Postman collection generation

---

## Step 406: OpenAPI 3.0 Structure

```nim
import json, strformat, tables, sequtils

# OpenAPI 3.0 specification builder

type
  SchemaType = enum
    stString, stInteger, stNumber, stBoolean, stObject, stArray

  SchemaProperty = object
    name: string
    schemaType: SchemaType
    description: string
    required: bool
    format: string
    example: JsonNode
    enumValues: seq[string]
    minimum: Option[float]
    maximum: Option[float]

  Schema = object
    name: string
    description: string
    properties: seq[SchemaProperty]
    requiredFields: seq[string]

  Parameter = object
    name: string
    in_: string  # path, query, header, cookie
    description: string
    required: bool
    schema: JsonNode
    example: JsonNode

  RequestBody = object
    description: string
    required: bool
    contentType: string
    schema: string  # schema name reference

  Response = object
    statusCode: int
    description: string
    contentType: string
    schema: string

  Endpoint = object
    path: string
    httpMethod: string
    summary: string
    description: string
    tags: seq[string]
    parameters: seq[Parameter]
    requestBody: Option[RequestBody]
    responses: seq[Response]
    security: seq[string]

  OpenApiSpec = object
    title: string
    version: string
    description: string
    baseUrl: string
    schemas: seq[Schema]
    endpoints: seq[Endpoint]

proc newSpec(title, version, description, baseUrl: string): OpenApiSpec =
  OpenApiSpec(
    title: title,
    version: version,
    description: description,
    baseUrl: baseUrl,
    schemas: @[],
    endpoints: @[]
  )

proc addSchema(spec: var OpenApiSpec, schema: Schema) =
  spec.schemas.add(schema)

proc addEndpoint(spec: var OpenApiSpec, ep: Endpoint) =
  spec.endpoints.add(ep)

proc toJson(spec: OpenApiSpec): JsonNode =
  var schemas = newJObject()
  for schema in spec.schemas:
    var props = newJObject()
    var required = newJArray()
    
    for prop in schema.properties:
      var propObj = %*{
        "type": case prop.schemaType
          of stString: "string"
          of stInteger: "integer"
          of stNumber: "number"
          of stBoolean: "boolean"
          of stObject: "object"
          of stArray: "array"
      }
      
      if prop.description.len > 0:
        propObj["description"] = %prop.description
      
      if prop.format.len > 0:
        propObj["format"] = %prop.format
      
      if not prop.example.isNil:
        propObj["example"] = prop.example
      
      if prop.enumValues.len > 0:
        var enumArr = newJArray()
        for v in prop.enumValues:
          enumArr.add(%v)
        propObj["enum"] = enumArr
      
      if prop.required:
        required.add(%prop.name)
      
      props[prop.name] = propObj
    
    schemas[schema.name] = %*{
      "type": "object",
      "description": schema.description,
      "required": required,
      "properties": props
    }
  
  var paths = newJObject()
  for ep in spec.endpoints:
    if ep.path notin paths:
      paths[ep.path] = newJObject()
    
    var opObj = newJObject()
    opObj["summary"] = %ep.summary
    if ep.description.len > 0:
      opObj["description"] = %ep.description
    
    var tagsArr = newJArray()
    for tag in ep.tags:
      tagsArr.add(%tag)
    opObj["tags"] = tagsArr
    
    var paramsArr = newJArray()
    for param in ep.parameters:
      paramsArr.add(%*{
        "name": param.name,
        "in": param.in_,
        "description": param.description,
        "required": param.required,
        "schema": param.schema
      })
    if paramsArr.len > 0:
      opObj["parameters"] = paramsArr
    
    if ep.requestBody.isSome:
      let rb = ep.requestBody.get()
      opObj["requestBody"] = %*{
        "description": rb.description,
        "required": rb.required,
        "content": {
          rb.contentType: {
            "schema": {"$ref": "#/components/schemas/" & rb.schema}
          }
        }
      }
    
    var responsesObj = newJObject()
    for resp in ep.responses:
      var respObj = %*{"description": resp.description}
      if resp.schema.len > 0:
        respObj["content"] = %*{
          resp.contentType: {
            "schema": {"$ref": "#/components/schemas/" & resp.schema}
          }
        }
      responsesObj[$resp.statusCode] = respObj
    opObj["responses"] = responsesObj
    
    if ep.security.len > 0:
      var secArr = newJArray()
      for sec in ep.security:
        secArr.add(%*{sec: []})
      opObj["security"] = secArr
    
    paths[ep.path][ep.httpMethod.toLowerAscii()] = opObj
  
  return %*{
    "openapi": "3.0.0",
    "info": {
      "title": spec.title,
      "version": spec.version,
      "description": spec.description
    },
    "servers": [{"url": spec.baseUrl}],
    "components": {
      "schemas": schemas,
      "securitySchemes": {
        "BearerAuth": {
          "type": "http",
          "scheme": "bearer",
          "bearerFormat": "JWT"
        }
      }
    },
    "paths": paths
  }

# Build a sample API spec
var spec = newSpec(
  "Blog API",
  "1.0.0",
  "REST API for blog platform with authentication",
  "https://api.example.com/v1"
)

# Define schemas
spec.addSchema(Schema(
  name: "CreateUserRequest",
  description: "Request body for user registration",
  properties: @[
    SchemaProperty(name: "name", schemaType: stString, description: "Full name",
      required: true, example: %"สมชาย ใจดี"),
    SchemaProperty(name: "email", schemaType: stString, description: "Email address",
      required: true, format: "email", example: %"user@example.com"),
    SchemaProperty(name: "password", schemaType: stString, description: "Password (min 8 chars)",
      required: true, format: "password", example: %"SecureP@ss123"),
  ]
))

spec.addSchema(Schema(
  name: "UserResponse",
  description: "User data response",
  properties: @[
    SchemaProperty(name: "id", schemaType: stInteger, description: "User ID", example: %42),
    SchemaProperty(name: "name", schemaType: stString, description: "Full name", example: %"สมชาย"),
    SchemaProperty(name: "email", schemaType: stString, description: "Email", example: %"user@example.com"),
    SchemaProperty(name: "createdAt", schemaType: stString, format: "date-time",
      description: "Registration timestamp"),
  ]
))

spec.addSchema(Schema(
  name: "PostRequest",
  description: "Create/update blog post",
  properties: @[
    SchemaProperty(name: "title", schemaType: stString, required: true, example: %"My Post"),
    SchemaProperty(name: "content", schemaType: stString, required: true, example: %"Post content..."),
    SchemaProperty(name: "tags", schemaType: stArray, description: "Post tags"),
    SchemaProperty(name: "published", schemaType: stBoolean, description: "Publish immediately", example: %false),
  ]
))

# Define endpoints
spec.addEndpoint(Endpoint(
  path: "/auth/register",
  httpMethod: "POST",
  summary: "Register new user",
  description: "Creates a new user account",
  tags: @["auth"],
  requestBody: some(RequestBody(
    description: "User registration data",
    required: true,
    contentType: "application/json",
    schema: "CreateUserRequest"
  )),
  responses: @[
    Response(statusCode: 201, description: "User created successfully",
             contentType: "application/json", schema: "UserResponse"),
    Response(statusCode: 400, description: "Validation error"),
    Response(statusCode: 409, description: "Email already exists"),
  ]
))

spec.addEndpoint(Endpoint(
  path: "/posts",
  httpMethod: "GET",
  summary: "List blog posts",
  tags: @["posts"],
  parameters: @[
    Parameter(name: "page", in_: "query", description: "Page number",
              required: false, schema: %*{"type": "integer", "default": 1}),
    Parameter(name: "limit", in_: "query", description: "Items per page",
              required: false, schema: %*{"type": "integer", "default": 10, "maximum": 100}),
    Parameter(name: "tag", in_: "query", description: "Filter by tag",
              required: false, schema: %*{"type": "string"}),
  ],
  responses: @[
    Response(statusCode: 200, description: "Posts list"),
  ]
))

spec.addEndpoint(Endpoint(
  path: "/posts",
  httpMethod: "POST",
  summary: "Create blog post",
  tags: @["posts"],
  security: @["BearerAuth"],
  requestBody: some(RequestBody(
    description: "Post data",
    required: true,
    contentType: "application/json",
    schema: "PostRequest"
  )),
  responses: @[
    Response(statusCode: 201, description: "Post created"),
    Response(statusCode: 401, description: "Unauthorized"),
    Response(statusCode: 400, description: "Validation error"),
  ]
))

spec.addEndpoint(Endpoint(
  path: "/posts/{id}",
  httpMethod: "GET",
  summary: "Get post by ID",
  tags: @["posts"],
  parameters: @[
    Parameter(name: "id", in_: "path", description: "Post ID",
              required: true, schema: %*{"type": "integer"}),
  ],
  responses: @[
    Response(statusCode: 200, description: "Post data"),
    Response(statusCode: 404, description: "Post not found"),
  ]
))

# Output spec
let specJson = spec.toJson()
echo specJson.pretty()
```

---

## Step 407: Swagger UI HTML Generator

```nim
proc generateSwaggerHtml(specUrl: string): string =
  return """<!DOCTYPE html>
<html>
<head>
  <title>API Documentation</title>
  <meta charset="utf-8"/>
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <link rel="stylesheet" type="text/css" href="https://unpkg.com/swagger-ui-dist@5/swagger-ui.css">
</head>
<body>
<div id="swagger-ui"></div>
<script src="https://unpkg.com/swagger-ui-dist@5/swagger-ui-bundle.js"></script>
<script>
  SwaggerUIBundle({
    url: '""" & specUrl & """',
    dom_id: '#swagger-ui',
    presets: [SwaggerUIBundle.presets.apis, SwaggerUIBundle.SwaggerUIStandalonePreset],
    layout: "BaseLayout",
    deepLinking: true,
    showExtensions: true,
    showCommonExtensions: true,
    tryItOutEnabled: true,
    requestInterceptor: (req) => {
      const token = localStorage.getItem('jwt_token');
      if (token) req.headers['Authorization'] = 'Bearer ' + token;
      return req;
    }
  })
</script>
</body>
</html>"""

echo "Swagger UI HTML:"
echo generateSwaggerHtml("/api/openapi.json")
```

---

## Step 408: API Versioning

```nim
import asyncdispatch, asynchttpserver, strutils, strformat, json

# API versioning strategies:
# 1. URL versioning: /v1/users, /v2/users
# 2. Header versioning: Accept: application/vnd.api+json;version=2
# 3. Query param: /users?version=2

# We'll use URL versioning (most common and clear)

type
  ApiVersion = enum
    V1, V2

  RouteHandler = proc(req: Request): Future[void]
  
  VersionedRouter = object
    routes: Table[string, Table[string, RouteHandler]]  # version -> path -> handler

var versionedRouter = VersionedRouter(
  routes: initTable[string, Table[string, RouteHandler]]()
)

proc addRoute(router: var VersionedRouter, version: ApiVersion,
              path: string, handler: RouteHandler) =
  let vStr = case version
    of V1: "v1"
    of V2: "v2"
  
  if vStr notin router.routes:
    router.routes[vStr] = initTable[string, RouteHandler]()
  
  router.routes[vStr][path] = handler

proc dispatch(router: VersionedRouter, req: Request): Future[bool] {.async.} =
  let parts = req.url.path.strip(chars = {'/'}).split('/')
  
  if parts.len < 2:
    return false
  
  let version = parts[0]
  let path = "/" & parts[1..^1].join("/")
  
  if version notin router.routes:
    return false
  
  if path notin router.routes[version]:
    return false
  
  await router.routes[version][path](req)
  return true

# V1 handlers
proc getUsersV1(req: Request) {.async.} =
  let resp = %*{
    "users": [
      {"id": 1, "name": "Alice"},
      {"id": 2, "name": "Bob"}
    ]
  }
  await req.respond(Http200, $resp, newHttpHeaders([("Content-Type", "application/json")]))

# V2 handlers (enhanced response)
proc getUsersV2(req: Request) {.async.} =
  let resp = %*{
    "data": [
      {"id": 1, "name": "Alice", "email": "alice@example.com", "avatar": "https://..."},
      {"id": 2, "name": "Bob", "email": "bob@example.com", "avatar": "https://..."}
    ],
    "meta": {"total": 2, "page": 1, "perPage": 10}
  }
  await req.respond(Http200, $resp, newHttpHeaders([("Content-Type", "application/json")]))

versionedRouter.addRoute(V1, "/users", getUsersV1)
versionedRouter.addRoute(V2, "/users", getUsersV2)

echo "API versioning configured"
echo "  v1: basic user response"
echo "  v2: enhanced user response with metadata"
```

---

## Step 409-420: Complete API Documentation System

```nim
# api_docs.nim - Auto-generate and serve API documentation

import asyncdispatch, asynchttpserver, json, tables, strutils, strformat

# ============================
# Route Registry (self-documenting)
# ============================

type
  EndpointDoc = object
    method: string
    path: string
    summary: string
    auth: bool
    params: seq[tuple[name, in_, typ, desc: string]]
    body: string
    responses: seq[tuple[code: int, desc: string]]

  ApiRegistry = object
    title: string
    version: string
    description: string
    endpoints: seq[EndpointDoc]

var apiRegistry = ApiRegistry(
  title: "Nim Blog API",
  version: "2.0.0",
  description: "Full-featured blog REST API built with Nim"
)

proc doc(registry: var ApiRegistry, ep: EndpointDoc) =
  registry.endpoints.add(ep)

# Register all endpoints
apiRegistry.doc(EndpointDoc(
  method: "POST", path: "/auth/register", summary: "Register new user",
  auth: false,
  body: """{"name":"string","email":"string","password":"string"}""",
  responses: @[
    (201, "User created"),
    (400, "Validation error"),
    (409, "Email taken")
  ]
))

apiRegistry.doc(EndpointDoc(
  method: "POST", path: "/auth/login", summary: "Login",
  auth: false,
  body: """{"email":"string","password":"string"}""",
  responses: @[
    (200, """{"token":"jwt","refreshToken":"jwt","expiresIn":3600}"""),
    (401, "Invalid credentials"),
    (423, "Account locked")
  ]
))

apiRegistry.doc(EndpointDoc(
  method: "POST", path: "/auth/refresh", summary: "Refresh access token",
  auth: false,
  body: """{"refreshToken":"string"}""",
  responses: @[(200, "New token pair"), (401, "Invalid refresh token")]
))

apiRegistry.doc(EndpointDoc(
  method: "GET", path: "/posts", summary: "List posts",
  auth: false,
  params: @[
    ("page", "query", "integer", "Page number (default: 1)"),
    ("limit", "query", "integer", "Items per page (max: 100)"),
    ("tag", "query", "string", "Filter by tag"),
    ("search", "query", "string", "Full-text search"),
  ],
  responses: @[(200, """{"data":[...],"total":100,"page":1}""")]
))

apiRegistry.doc(EndpointDoc(
  method: "POST", path: "/posts", summary: "Create post", auth: true,
  body: """{"title":"string","content":"string","tags":["string"],"published":false}""",
  responses: @[(201, "Created"), (401, "Unauthorized"), (400, "Validation error")]
))

apiRegistry.doc(EndpointDoc(
  method: "GET", path: "/posts/:id", summary: "Get post", auth: false,
  params: @[("id", "path", "integer", "Post ID")],
  responses: @[(200, "Post data"), (404, "Not found")]
))

apiRegistry.doc(EndpointDoc(
  method: "PUT", path: "/posts/:id", summary: "Update post", auth: true,
  params: @[("id", "path", "integer", "Post ID")],
  body: """{"title":"string","content":"string","tags":["string"]}""",
  responses: @[(200, "Updated"), (401, "Unauthorized"), (404, "Not found")]
))

apiRegistry.doc(EndpointDoc(
  method: "DELETE", path: "/posts/:id", summary: "Delete post", auth: true,
  params: @[("id", "path", "integer", "Post ID")],
  responses: @[(204, "Deleted"), (401, "Unauthorized"), (404, "Not found")]
))

apiRegistry.doc(EndpointDoc(
  method: "GET", path: "/users/:id", summary: "Get user profile", auth: false,
  params: @[("id", "path", "integer", "User ID")],
  responses: @[(200, "User profile"), (404, "Not found")]
))

# ============================
# Generate HTML docs
# ============================

proc generateHtmlDocs(registry: ApiRegistry): string =
  var html = fmt"""<!DOCTYPE html>
<html>
<head>
<meta charset="utf-8">
<title>{registry.title} - API Docs</title>
<style>
  body {{ font-family: -apple-system, sans-serif; max-width: 900px; margin: 40px auto; padding: 0 20px; }}
  h1 {{ color: #1a1a2e; }}
  .version {{ color: #666; font-size: 14px; }}
  .endpoint {{ border: 1px solid #e1e4e8; border-radius: 6px; margin: 16px 0; overflow: hidden; }}
  .endpoint-header {{ display: flex; align-items: center; padding: 12px 16px; cursor: pointer; }}
  .method {{ font-weight: bold; padding: 4px 8px; border-radius: 4px; font-size: 12px; min-width: 60px; text-align: center; }}
  .GET {{ background: #e3f2fd; color: #0d47a1; }}
  .POST {{ background: #e8f5e9; color: #1b5e20; }}
  .PUT {{ background: #fff8e1; color: #e65100; }}
  .DELETE {{ background: #ffebee; color: #b71c1c; }}
  .PATCH {{ background: #f3e5f5; color: #4a148c; }}
  .path {{ font-family: monospace; margin-left: 12px; font-size: 15px; }}
  .summary {{ margin-left: 16px; color: #586069; }}
  .auth-badge {{ margin-left: auto; background: #fff3e0; color: #e65100; padding: 2px 8px; border-radius: 12px; font-size: 11px; }}
  .endpoint-body {{ padding: 16px; border-top: 1px solid #e1e4e8; display: none; }}
  .section-title {{ font-weight: bold; color: #24292e; margin: 12px 0 6px; font-size: 13px; text-transform: uppercase; }}
  .param {{ display: flex; gap: 12px; margin: 4px 0; font-size: 14px; }}
  .param-name {{ font-family: monospace; color: #0d47a1; min-width: 120px; }}
  .param-in {{ color: #586069; font-size: 12px; }}
  code {{ background: #f6f8fa; padding: 12px; border-radius: 4px; display: block; font-size: 13px; white-space: pre-wrap; }}
  .response {{ display: flex; gap: 12px; margin: 4px 0; font-size: 14px; }}
  .status-200 {{ color: #28a745; }}
  .status-201 {{ color: #28a745; }}
  .status-400 {{ color: #dc3545; }}
  .status-401 {{ color: #dc3545; }}
  .status-404 {{ color: #dc3545; }}
</style>
</head>
<body>
<h1>{registry.title}</h1>
<p class="version">Version {registry.version} · {registry.description}</p>
"""
  
  # Group by tags (first path segment)
  var groups: OrderedTable[string, seq[EndpointDoc]]
  for ep in registry.endpoints:
    let parts = ep.path.strip(chars={'/'}).split('/')
    let group = if parts.len > 0: parts[0] else: "general"
    if group notin groups:
      groups[group] = @[]
    groups[group].add(ep)
  
  for group, endpoints in groups:
    html &= fmt"<h2 style='text-transform:capitalize'>{group}</h2>\n"
    
    for ep in endpoints:
      html &= fmt"""<div class="endpoint">
  <div class="endpoint-header" onclick="this.nextElementSibling.style.display=this.nextElementSibling.style.display=='none'?'block':'none'">
    <span class="method {ep.method}">{ep.method}</span>
    <code class="path" style="background:none;padding:0">{ep.path}</code>
    <span class="summary">{ep.summary}</span>
    {if ep.auth: "<span class='auth-badge'>🔒 JWT</span>" else: ""}
  </div>
  <div class="endpoint-body">
"""
      
      if ep.params.len > 0:
        html &= """    <div class="section-title">Parameters</div>"""
        for p in ep.params:
          html &= fmt"""    <div class="param">
      <span class="param-name">{p.name}</span>
      <span class="param-in">{p.in_}</span>
      <span>{p.typ}</span>
      <span style="color:#586069">{p.desc}</span>
    </div>
"""
      
      if ep.body.len > 0:
        html &= fmt"""    <div class="section-title">Request Body</div>
    <code>{ep.body}</code>
"""
      
      if ep.responses.len > 0:
        html &= """    <div class="section-title">Responses</div>"""
        for r in ep.responses:
          html &= fmt"""    <div class="response">
      <span class="status-{r.code}" style="min-width:40px;font-weight:bold">{r.code}</span>
      <span>{r.desc}</span>
    </div>
"""
      
      html &= "  </div>\n</div>\n"
  
  html &= "</body></html>"
  return html

# ============================
# Serve docs
# ============================

proc handleDocs(req: Request) {.async.} =
  let path = req.url.path
  
  if path == "/docs" or path == "/docs/":
    let html = generateHtmlDocs(apiRegistry)
    await req.respond(Http200, html,
      newHttpHeaders([("Content-Type", "text/html; charset=utf-8")]))
    return
  
  if path == "/docs/openapi.json":
    # Generate OpenAPI JSON
    var pathsObj = newJObject()
    for ep in apiRegistry.endpoints:
      if ep.path notin pathsObj:
        pathsObj[ep.path] = newJObject()
      
      pathsObj[ep.path][ep.method.toLowerAscii()] = %*{
        "summary": ep.summary,
        "security": if ep.auth: [{ep.method: []}] else: []
      }
    
    let spec = %*{
      "openapi": "3.0.0",
      "info": {
        "title": apiRegistry.title,
        "version": apiRegistry.version,
        "description": apiRegistry.description
      },
      "paths": pathsObj
    }
    
    await req.respond(Http200, spec.pretty(),
      newHttpHeaders([("Content-Type", "application/json")]))
    return
  
  await req.respond(Http404, """{"error":"not found"}""",
    newHttpHeaders([("Content-Type", "application/json")]))

# Demo
proc main() =
  echo fmt"API: {apiRegistry.title} v{apiRegistry.version}"
  echo fmt"Documented endpoints: {apiRegistry.endpoints.len}"
  echo ""
  for ep in apiRegistry.endpoints:
    let auth = if ep.auth: " 🔒" else: ""
    echo fmt"  {ep.method:<7} {ep.path}{auth}"
  
  echo "\nHTML docs generated (would be served at /docs)"
  let html = generateHtmlDocs(apiRegistry)
  echo fmt"HTML size: {html.len} bytes"

main()
```

---

## 📝 สรุป Part 29

| Steps | หัวข้อ |
|-------|--------|
| 406 | OpenAPI 3.0 spec builder |
| 407 | Swagger UI HTML generator |
| 408 | API versioning patterns |
| 409-420 | Complete self-documenting API with HTML docs |

---

**← [Part 28: Logging & Monitoring](part_28_logging_monitoring.md) | [Part 30: GraphQL →](part_30_graphql.md)**
