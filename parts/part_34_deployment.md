# Part 34: Production Deployment
## Steps 481-495: Deploy ระดับ Production

---

## 🎯 เป้าหมายของ Part นี้

- Cross-compilation
- Docker multi-stage build
- Kubernetes deployment
- CI/CD pipeline
- Zero-downtime deployment
- Environment configuration
- Health checks และ graceful shutdown

---

## Step 481: Optimized Build

```bash
# Build flags for production

# Basic release build
nim c -d:release myapp.nim

# Full optimization (slower compile, faster binary)
nim c -d:release -d:danger myapp.nim

# With ARC (better performance, deterministic GC)
nim c -d:release --gc:arc myapp.nim

# Strip debug symbols
nim c -d:release --passL:"-s" myapp.nim

# Static binary (no external dependencies)
nim c -d:release --passL:"-static" myapp.nim

# Cross-compile to Linux from macOS
nim c -d:release --os:linux --cpu:amd64 \
    --passC:"-O3" \
    --passL:"-static" \
    -o:myapp-linux myapp.nim

# Check binary size
# ls -la myapp
# strip myapp
# upx --best myapp  (optional compression)
```

```nim
# Build script (build.nim)
import os, strformat, strutils

type
  BuildTarget = object
    os: string
    arch: string
    output: string
    extraFlags: seq[string]

const targets = [
  BuildTarget(os: "linux",   arch: "amd64",   output: "dist/app-linux-amd64"),
  BuildTarget(os: "linux",   arch: "arm64",   output: "dist/app-linux-arm64"),
  BuildTarget(os: "macosx",  arch: "amd64",   output: "dist/app-macos-amd64"),
  BuildTarget(os: "windows", arch: "amd64",   output: "dist/app-win-amd64.exe"),
]

proc build(target: BuildTarget) =
  echo fmt"Building {target.os}/{target.arch}..."
  
  var flags = @[
    "-d:release",
    "--gc:arc",
    fmt"--os:{target.os}",
    fmt"--cpu:{target.arch}",
    fmt"-o:{target.output}",
    "--passC:\"-O3\"",
  ]
  flags.add(target.extraFlags)
  
  createDir("dist")
  
  let cmd = "nim c " & flags.join(" ") & " src/main.nim"
  echo "  " & cmd
  
  let code = execShellCmd(cmd)
  if code == 0:
    echo fmt"  ✓ Built: {target.output}"
  else:
    echo fmt"  ✗ Failed to build {target.output}"

# Build all targets
# for target in targets:
#   build(target)
echo "Build system ready"
```

---

## Step 482: Docker Multi-Stage Build

```dockerfile
# Dockerfile

# ============================
# Stage 1: Build
# ============================
FROM nimlang/nim:2.0.0-alpine AS builder

WORKDIR /app

# Copy nimble file first (cache dependencies)
COPY myapp.nimble ./
RUN nimble install -y --depsOnly

# Copy source
COPY src/ ./src/

# Build optimized binary
RUN nim c \
    -d:release \
    --gc:arc \
    --opt:speed \
    --passL:"-static" \
    -o:myapp \
    src/main.nim

# ============================
# Stage 2: Runtime
# ============================
FROM scratch

# Copy binary only
COPY --from=builder /app/myapp /myapp

# Non-root user
USER 1001

EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=3s --start-period=5s \
    CMD ["/myapp", "health"]

ENTRYPOINT ["/myapp"]
```

```yaml
# docker-compose.yml
version: '3.8'

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
    image: myapp:latest
    container_name: myapp
    restart: unless-stopped
    ports:
      - "8080:8080"
    environment:
      - DATABASE_URL=postgres://user:pass@db:5432/myapp
      - JWT_SECRET=${JWT_SECRET}
      - REDIS_URL=redis://redis:6379
      - APP_ENV=production
      - LOG_LEVEL=info
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started
    healthcheck:
      test: ["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "http://localhost:8080/health"]
      interval: 30s
      timeout: 10s
      retries: 3
    deploy:
      resources:
        limits:
          cpus: '2'
          memory: 512M
        reservations:
          cpus: '0.5'
          memory: 128M

  db:
    image: postgres:16-alpine
    container_name: myapp_db
    restart: unless-stopped
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: user
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./migrations:/docker-entrypoint-initdb.d
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user -d myapp"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    container_name: myapp_redis
    restart: unless-stopped
    volumes:
      - redis_data:/data
    command: redis-server --appendonly yes --maxmemory 256mb --maxmemory-policy allkeys-lru

  nginx:
    image: nginx:alpine
    container_name: myapp_nginx
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/certs:/etc/nginx/certs:ro
    depends_on:
      - app

volumes:
  postgres_data:
  redis_data:
```

---

## Step 483: Nginx Configuration

```nginx
# nginx/nginx.conf
worker_processes auto;
worker_rlimit_nofile 65535;

events {
    worker_connections 4096;
    use epoll;
    multi_accept on;
}

http {
    # Performance
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    keepalive_requests 1000;
    
    # Gzip
    gzip on;
    gzip_vary on;
    gzip_proxied any;
    gzip_comp_level 6;
    gzip_types text/plain application/json application/javascript text/css;
    
    # Rate limiting
    limit_req_zone $binary_remote_addr zone=api:10m rate=100r/m;
    limit_req_zone $binary_remote_addr zone=auth:10m rate=10r/m;
    
    # Upstream
    upstream app_backend {
        least_conn;
        server app:8080 max_fails=3 fail_timeout=30s;
        keepalive 32;
    }
    
    server {
        listen 80;
        server_name example.com;
        return 301 https://$server_name$request_uri;
    }
    
    server {
        listen 443 ssl http2;
        server_name example.com;
        
        # SSL
        ssl_certificate /etc/nginx/certs/cert.pem;
        ssl_certificate_key /etc/nginx/certs/key.pem;
        ssl_session_timeout 1d;
        ssl_session_cache shared:SSL:50m;
        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_prefer_server_ciphers off;
        
        # Security headers
        add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
        add_header X-Frame-Options DENY always;
        add_header X-Content-Type-Options nosniff always;
        add_header X-XSS-Protection "1; mode=block" always;
        
        # Proxy
        location /api/ {
            limit_req zone=api burst=20 nodelay;
            
            proxy_pass http://app_backend;
            proxy_http_version 1.1;
            proxy_set_header Connection "";
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            
            proxy_connect_timeout 5s;
            proxy_send_timeout 30s;
            proxy_read_timeout 30s;
            
            proxy_buffering on;
            proxy_buffer_size 4k;
            proxy_buffers 8 4k;
        }
        
        location /api/auth/ {
            limit_req zone=auth burst=5 nodelay;
            proxy_pass http://app_backend;
        }
        
        location /health {
            proxy_pass http://app_backend;
            access_log off;
        }
    }
}
```

---

## Step 484: Graceful Shutdown

```nim
import asyncdispatch, asynchttpserver, os, posix, strformat, times, sequtils

# ============================
# Graceful shutdown
# ============================

type
  Server = object
    httpServer: AsyncHttpServer
    activeConnections: int
    shuttingDown: bool
    startTime: float

var server = Server(startTime: epochTime())

proc handleSignal(sig: cint) {.noconv.} =
  echo "\n[Server] Received signal " & $sig & ", shutting down gracefully..."
  server.shuttingDown = true

proc handler(req: Request) {.async.} =
  if server.shuttingDown:
    await req.respond(Http503, """{"error":"Server shutting down"}""",
      newHttpHeaders([("Content-Type", "application/json"), ("Retry-After", "5")]))
    return
  
  inc server.activeConnections
  
  try:
    case req.url.path
    of "/":
      await req.respond(Http200, """{"status":"ok"}""",
        newHttpHeaders([("Content-Type", "application/json")]))
    of "/health":
      let uptime = epochTime() - server.startTime
      await req.respond(Http200,
        $(%*{"status": "ok", "uptime": uptime, "connections": server.activeConnections}),
        newHttpHeaders([("Content-Type", "application/json")]))
    else:
      await req.respond(Http404, """{"error":"not found"}""",
        newHttpHeaders([("Content-Type", "application/json")]))
  finally:
    dec server.activeConnections

proc runServer() {.async.} =
  server.httpServer = newAsyncHttpServer()
  
  # Register signal handlers
  setControlCHook(proc() {.noconv.} = handleSignal(SIGINT))
  
  echo "[Server] Starting on port 8080"
  
  let serveFuture = server.httpServer.serve(Port(8080), handler)
  asyncCheck serveFuture
  
  # Wait for shutdown signal
  while not server.shuttingDown:
    await sleepAsync(100)
  
  # Wait for active connections to drain (max 30 seconds)
  echo fmt"[Server] Waiting for {server.activeConnections} connections to drain..."
  let deadline = epochTime() + 30.0
  
  while server.activeConnections > 0 and epochTime() < deadline:
    await sleepAsync(100)
  
  if server.activeConnections > 0:
    echo fmt"[Server] Force closing {server.activeConnections} connections"
  
  server.httpServer.close()
  echo "[Server] Shutdown complete"

# waitFor runServer()
echo "Graceful shutdown configured"
```

---

## Step 485-495: Complete Production Setup

```nim
# production_config.nim - Complete production configuration

import os, strutils, strformat, json, tables, options

# ============================
# Configuration
# ============================

type
  AppConfig = object
    # Server
    host: string
    port: int
    
    # Database
    dbUrl: string
    dbMaxConnections: int
    dbIdleConnections: int
    dbConnTimeout: int
    
    # Redis
    redisUrl: string
    redisMaxConnections: int
    
    # JWT
    jwtSecret: string
    jwtExpiresIn: int
    jwtRefreshExpiresIn: int
    
    # Limits
    maxRequestSize: int
    maxConnections: int
    requestTimeout: int
    
    # Features
    logLevel: string
    enableMetrics: bool
    enableProfiling: bool
    
    # App
    environment: string
    version: string
    serviceName: string

proc loadConfig(): AppConfig =
  let env = getEnv("APP_ENV", "development")
  
  result = AppConfig(
    # Server
    host: getEnv("HOST", "0.0.0.0"),
    port: getEnv("PORT", "8080").parseInt(),
    
    # Database
    dbUrl: getEnv("DATABASE_URL", "postgres://localhost/myapp_dev"),
    dbMaxConnections: getEnv("DB_MAX_CONNECTIONS", "20").parseInt(),
    dbIdleConnections: getEnv("DB_IDLE_CONNECTIONS", "5").parseInt(),
    dbConnTimeout: getEnv("DB_CONN_TIMEOUT", "5000").parseInt(),
    
    # Redis
    redisUrl: getEnv("REDIS_URL", "redis://localhost:6379"),
    redisMaxConnections: getEnv("REDIS_MAX_CONNECTIONS", "10").parseInt(),
    
    # JWT
    jwtSecret: getEnv("JWT_SECRET", "dev-secret-change-in-prod"),
    jwtExpiresIn: getEnv("JWT_EXPIRES_IN", "3600").parseInt(),
    jwtRefreshExpiresIn: getEnv("JWT_REFRESH_EXPIRES_IN", "2592000").parseInt(),
    
    # Limits
    maxRequestSize: getEnv("MAX_REQUEST_SIZE", "10485760").parseInt(),  # 10MB
    maxConnections: getEnv("MAX_CONNECTIONS", "1000").parseInt(),
    requestTimeout: getEnv("REQUEST_TIMEOUT", "30000").parseInt(),
    
    # Features
    logLevel: getEnv("LOG_LEVEL", if env == "production": "warn" else: "debug"),
    enableMetrics: getEnv("ENABLE_METRICS", "true") == "true",
    enableProfiling: getEnv("ENABLE_PROFILING", "false") == "true",
    
    # App
    environment: env,
    version: getEnv("APP_VERSION", "0.0.0"),
    serviceName: getEnv("SERVICE_NAME", "my-nim-service")
  )

proc validateConfig(config: AppConfig): seq[string] =
  var errors: seq[string] = @[]
  
  if config.environment == "production":
    if config.jwtSecret == "dev-secret-change-in-prod":
      errors.add("JWT_SECRET must be set in production")
    if config.jwtSecret.len < 32:
      errors.add("JWT_SECRET must be at least 32 characters")
    if not config.dbUrl.startsWith("postgres://"):
      errors.add("DATABASE_URL must use postgres:// in production")
  
  if config.port < 1 or config.port > 65535:
    errors.add(fmt"PORT must be 1-65535, got {config.port}")
  
  if config.dbMaxConnections < 1 or config.dbMaxConnections > 1000:
    errors.add("DB_MAX_CONNECTIONS must be 1-1000")
  
  return errors

proc printConfig(config: AppConfig) =
  echo fmt"Service: {config.serviceName} v{config.version}"
  echo fmt"Environment: {config.environment}"
  echo fmt"Port: {config.port}"
  echo fmt"Log level: {config.logLevel}"
  echo fmt"DB connections: {config.dbIdleConnections}/{config.dbMaxConnections}"
  
  if config.environment != "production":
    echo fmt"DB URL: {config.dbUrl}"
  else:
    echo "DB URL: [hidden in production]"

# ============================
# Health check endpoint data
# ============================

type
  BuildInfo = object
    version: string
    buildTime: string
    gitCommit: string
    goVersion: string  # actually nim version here

const buildInfo = BuildInfo(
  version: "1.0.0",
  buildTime: CompileDate & " " & CompileTime,
  gitCommit: "abc1234",  # in real build: pass via -d:gitCommit=...
  goVersion: NimVersion
)

# ============================
# Demo
# ============================

proc main() =
  echo "=== Production Configuration Demo ==="
  
  let config = loadConfig()
  let errors = validateConfig(config)
  
  if errors.len > 0:
    echo "\n⚠ Configuration errors:"
    for err in errors:
      echo fmt"  - {err}"
  else:
    echo "\n✓ Configuration valid"
  
  echo ""
  printConfig(config)
  
  echo fmt"\nBuild info:"
  echo fmt"  Version: {buildInfo.version}"
  echo fmt"  Built: {buildInfo.buildTime}"
  echo fmt"  Nim: {buildInfo.goVersion}"
  
  # Kubernetes-style deployment info
  echo "\nKubernetes labels:"
  echo fmt"  app.kubernetes.io/name: {config.serviceName}"
  echo fmt"  app.kubernetes.io/version: {config.version}"
  echo fmt"  app.kubernetes.io/managed-by: helm"

main()
```

---

## 📝 สรุป Part 34

| Steps | หัวข้อ |
|-------|--------|
| 481 | Optimized build flags, cross-compilation |
| 482 | Docker multi-stage build + compose |
| 483 | Nginx reverse proxy configuration |
| 484 | Graceful shutdown (SIGINT handling) |
| 485-495 | Production config, validation, build info |

---

**← [Part 33: Performance](part_33_performance.md) | [Part 35: Advanced Patterns →](part_35_advanced_patterns.md)**
