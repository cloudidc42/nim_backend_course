# Part 18: Docker สำหรับ Nim

## Steps 241-255

Docker ช่วยให้เราสามารถ package แอพพลิเคชัน Nim พร้อม dependencies ทั้งหมดเพื่อให้รันได้ทุกที่ ในบทนี้เราจะเรียนรู้การสร้าง Dockerfile ที่ดี ใช้ docker-compose และ deploy production stack ที่สมบูรณ์

---

## Step 241: Dockerfile พื้นฐานสำหรับ Nim

```dockerfile
# file: Dockerfile.simple
# ใช้ official Nim image
FROM nimlang/nim:2.0.0

# Working directory
WORKDIR /app

# Copy source files
COPY . .

# Install dependencies
RUN nimble install -d -y

# Build
RUN nimble build -d:release

# รัน binary
CMD ["./myapp"]
```

ปัญหาของ Dockerfile ข้างต้น: image ขนาดใหญ่มาก (รวม Nim compiler ด้วย)

---

## Step 242: Multi-Stage Build (Best Practice)

```dockerfile
# file: Dockerfile
# Stage 1: Build
FROM nimlang/nim:2.0.0-alpine AS builder

WORKDIR /build

# Copy nimble files ก่อน (leverage Docker cache)
COPY *.nimble ./
COPY nimble.lock* ./

# Install dependencies
RUN nimble install -d -y

# Copy source code
COPY src/ ./src/

# Build optimized binary
RUN nimble build \
    -d:release \
    -d:ssl \
    --opt:speed \
    --passC:"-march=native" \
    -o:app

# Stage 2: Runtime (ขนาดเล็กมาก)
FROM alpine:3.18

# Install runtime dependencies เท่านั้น
RUN apk add --no-cache \
    libssl3 \
    libcrypto3 \
    pcre \
    libc6-compat

WORKDIR /app

# Copy เฉพาะ binary
COPY --from=builder /build/app ./app

# Non-root user
RUN adduser -D -u 1001 appuser
USER appuser

EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD wget -q -O- http://localhost:8080/health || exit 1

CMD ["./app"]
```

ผลลัพธ์: image ขนาดประมาณ 15-20MB แทน 500MB+

---

## Step 243: Nim Application พร้อม Docker

```nim
# file: src/server.nim
# Application ที่รองรับ Docker environment

import asyncdispatch, asynchttpserver, httpcore
import strformat, json, os, strutils, times

type
  Config = object
    host: string
    port: int
    dbUrl: string
    redisUrl: string
    logLevel: string
    environment: string
    maxConnections: int

proc loadConfig(): Config =
  # อ่านค่าจาก environment variables
  Config(
    host: getEnv("HOST", "0.0.0.0"),
    port: parseInt(getEnv("PORT", "8080")),
    dbUrl: getEnv("DATABASE_URL", "postgres://localhost/mydb"),
    redisUrl: getEnv("REDIS_URL", "redis://localhost:6379"),
    logLevel: getEnv("LOG_LEVEL", "info"),
    environment: getEnv("ENVIRONMENT", "development"),
    maxConnections: parseInt(getEnv("MAX_CONNECTIONS", "100"))
  )

proc log(level, message: string) =
  let timestamp = now().format("yyyy-MM-dd HH:mm:ss")
  echo &"[{timestamp}] [{level.toUpperAscii()}] {message}"

proc handleRequest(req: Request, config: Config) {.async.} =
  let startTime = epochTime()
  
  var statusCode = Http200
  var responseBody = ""
  
  case req.url.path
  of "/health":
    responseBody = $(%*{
      "status": "healthy",
      "environment": config.environment,
      "timestamp": $now(),
      "uptime": "ok"
    })
  
  of "/ready":
    # Readiness probe - ตรวจสอบ dependencies
    responseBody = $(%*{
      "ready": true,
      "checks": {
        "database": "ok",
        "redis": "ok"
      }
    })
  
  of "/api/info":
    responseBody = $(%*{
      "app": "nim-backend",
      "version": "1.0.0",
      "environment": config.environment,
      "port": config.port
    })
  
  of "/api/config":
    if config.environment != "production":
      responseBody = $(%*{
        "host": config.host,
        "port": config.port,
        "environment": config.environment,
        "logLevel": config.logLevel
      })
    else:
      statusCode = Http403
      responseBody = $(%*{"error": "Config not available in production"})
  
  else:
    statusCode = Http404
    responseBody = $(%*{"error": "Not Found"})
  
  let duration = int((epochTime() - startTime) * 1000)
  
  await req.respond(statusCode, responseBody,
    newHttpHeaders([
      ("Content-Type", "application/json"),
      ("X-Response-Time", &"{duration}ms")
    ]))
  
  log("info", &"{req.reqMethod} {req.url.path} {statusCode} {duration}ms")

proc main() {.async.} =
  let config = loadConfig()
  
  log("info", &"Starting server in {config.environment} mode")
  log("info", &"Listening on {config.host}:{config.port}")
  
  var server = newAsyncHttpServer()
  
  proc cb(req: Request) {.async.} =
    await handleRequest(req, config)
  
  await server.serve(Port(config.port), cb, address = config.host)

waitFor main()
```

---

## Step 244: .nimble File สำหรับ Docker Build

```nim
# file: myapp.nimble
# Package
version       = "1.0.0"
author        = "Your Name"
description   = "Production Nim Backend"
license       = "MIT"
srcDir        = "src"
bin           = @["server"]

# Dependencies
requires "nim >= 2.0.0"
requires "asyncdispatch"
requires "ws >= 0.5.0"
requires "db_postgres >= 0.1.0"

# Build tasks
task build_prod, "Build for production":
  exec "nim c -d:release -d:ssl --opt:speed -o:bin/server src/server.nim"

task build_debug, "Build for debugging":
  exec "nim c --debuginfo --lineDir:on -o:bin/server src/server.nim"

task test, "Run tests":
  exec "nim c -r tests/test_all.nim"

task docker, "Build Docker image":
  exec "docker build -t myapp:latest ."
```

---

## Step 245: docker-compose.yml พื้นฐาน

```yaml
# file: docker-compose.yml
version: '3.8'

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: nim_app
    restart: unless-stopped
    ports:
      - "8080:8080"
    environment:
      - PORT=8080
      - HOST=0.0.0.0
      - ENVIRONMENT=production
      - DATABASE_URL=postgres://nim_user:nim_pass@postgres:5432/nim_db
      - REDIS_URL=redis://redis:6379
      - LOG_LEVEL=info
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    networks:
      - backend

  postgres:
    image: postgres:15-alpine
    container_name: nim_postgres
    restart: unless-stopped
    environment:
      POSTGRES_DB: nim_db
      POSTGRES_USER: nim_user
      POSTGRES_PASSWORD: nim_pass
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./db/init.sql:/docker-entrypoint-initdb.d/init.sql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U nim_user -d nim_db"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - backend
    ports:
      - "5432:5432"  # ลบออกใน production

  redis:
    image: redis:7-alpine
    container_name: nim_redis
    restart: unless-stopped
    command: redis-server --appendonly yes --requirepass redis_secret
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "--no-auth-warning", "-a", "redis_secret", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - backend
    ports:
      - "6379:6379"  # ลบออกใน production

volumes:
  postgres_data:
    driver: local
  redis_data:
    driver: local

networks:
  backend:
    driver: bridge
```

---

## Step 246: docker-compose กับ Nginx Reverse Proxy

```yaml
# file: docker-compose.prod.yml
version: '3.8'

services:
  nginx:
    image: nginx:alpine
    container_name: nim_nginx
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/sites:/etc/nginx/sites-enabled:ro
      - ./ssl:/etc/nginx/ssl:ro
      - nginx_logs:/var/log/nginx
    depends_on:
      - app
    networks:
      - frontend
      - backend

  app:
    build:
      context: .
      target: production
    container_name: nim_app
    restart: unless-stopped
    # ไม่ expose port ออกมา - ผ่าน nginx เท่านั้น
    expose:
      - "8080"
    environment:
      - PORT=8080
      - ENVIRONMENT=production
    env_file:
      - .env.prod
    depends_on:
      - postgres
      - redis
    networks:
      - backend
    deploy:
      replicas: 2
      resources:
        limits:
          cpus: '1.0'
          memory: 256M
        reservations:
          cpus: '0.25'
          memory: 64M

  # ... postgres, redis, etc.

volumes:
  nginx_logs:

networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
    internal: true  # ไม่ให้เข้าถึงจากภายนอก
```

---

## Step 247: Nginx Configuration

```nginx
# file: nginx/nginx.conf
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log warn;
pid /var/run/nginx.pid;

events {
    worker_connections 1024;
    use epoll;
    multi_accept on;
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    # Logging
    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for" '
                    'rt=$request_time urt=$upstream_response_time';

    access_log /var/log/nginx/access.log main;

    # Performance
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    types_hash_max_size 2048;
    client_max_body_size 10M;

    # Gzip
    gzip on;
    gzip_vary on;
    gzip_proxied any;
    gzip_comp_level 6;
    gzip_types text/plain text/css application/json application/javascript
               text/xml application/xml application/xml+rss text/javascript;

    # Security headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header Referrer-Policy "no-referrer-when-downgrade" always;

    # Hide nginx version
    server_tokens off;

    # Rate limiting
    limit_req_zone $binary_remote_addr zone=api:10m rate=30r/m;
    limit_conn_zone $binary_remote_addr zone=conn:10m;

    # Upstream - load balance หลาย app instances
    upstream nim_app {
        least_conn;
        server app:8080 max_fails=3 fail_timeout=30s;
        keepalive 32;
    }

    include /etc/nginx/sites-enabled/*.conf;
}
```

```nginx
# file: nginx/sites/app.conf
server {
    listen 80;
    server_name myapp.example.com www.myapp.example.com;

    # Redirect HTTP to HTTPS
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name myapp.example.com www.myapp.example.com;

    # SSL Configuration
    ssl_certificate /etc/nginx/ssl/fullchain.pem;
    ssl_certificate_key /etc/nginx/ssl/privkey.pem;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers on;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384;

    # Health check (no rate limiting)
    location = /health {
        proxy_pass http://nim_app;
        access_log off;
    }

    # API endpoints
    location /api/ {
        limit_req zone=api burst=10 nodelay;
        limit_conn conn 10;

        proxy_pass http://nim_app;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_connect_timeout 5s;
        proxy_send_timeout 30s;
        proxy_read_timeout 30s;

        proxy_http_version 1.1;
        proxy_set_header Connection "";
    }

    # WebSocket
    location /ws {
        proxy_pass http://nim_app;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_read_timeout 3600s;
        proxy_send_timeout 3600s;
    }

    # Static files
    location /static/ {
        alias /var/www/static/;
        expires 1d;
        add_header Cache-Control "public, immutable";
    }
}
```

---

## Step 248: Environment Variables Management

```bash
# file: .env.example
# App
PORT=8080
HOST=0.0.0.0
ENVIRONMENT=production
LOG_LEVEL=info
MAX_CONNECTIONS=200

# Database
DATABASE_URL=postgres://nim_user:CHANGE_ME@postgres:5432/nim_db
DB_MAX_CONNECTIONS=20
DB_IDLE_TIMEOUT=300

# Redis
REDIS_URL=redis://:CHANGE_ME@redis:6379/0
REDIS_MAX_CONNECTIONS=50

# Security
JWT_SECRET=CHANGE_ME_USE_RANDOM_256BIT_KEY
SESSION_SECRET=CHANGE_ME_USE_RANDOM_256BIT_KEY
ALLOWED_ORIGINS=https://myapp.example.com

# External services (อย่า commit ค่าจริง!)
SMTP_HOST=smtp.example.com
SMTP_PORT=587
SMTP_USER=notifications@example.com
SMTP_PASS=CHANGE_ME
```

```nim
# file: src/config.nim
import os, strutils, json

type
  DatabaseConfig = object
    url: string
    maxConnections: int
    idleTimeout: int

  RedisConfig = object
    url: string
    maxConnections: int

  SecurityConfig = object
    jwtSecret: string
    sessionSecret: string
    allowedOrigins: seq[string]

  AppConfig = object
    port: int
    host: string
    environment: string
    logLevel: string
    db: DatabaseConfig
    redis: RedisConfig
    security: SecurityConfig

proc requireEnv(key: string): string =
  let val = getEnv(key, "")
  if val.len == 0:
    raise newException(ValueError, &"Required environment variable '{key}' not set")
  return val

proc loadAppConfig*(): AppConfig =
  AppConfig(
    port: parseInt(getEnv("PORT", "8080")),
    host: getEnv("HOST", "0.0.0.0"),
    environment: getEnv("ENVIRONMENT", "development"),
    logLevel: getEnv("LOG_LEVEL", "info"),
    
    db: DatabaseConfig(
      url: requireEnv("DATABASE_URL"),
      maxConnections: parseInt(getEnv("DB_MAX_CONNECTIONS", "20")),
      idleTimeout: parseInt(getEnv("DB_IDLE_TIMEOUT", "300"))
    ),
    
    redis: RedisConfig(
      url: requireEnv("REDIS_URL"),
      maxConnections: parseInt(getEnv("REDIS_MAX_CONNECTIONS", "50"))
    ),
    
    security: SecurityConfig(
      jwtSecret: requireEnv("JWT_SECRET"),
      sessionSecret: requireEnv("SESSION_SECRET"),
      allowedOrigins: getEnv("ALLOWED_ORIGINS", "*").split(",")
    )
  )

proc isProduction*(config: AppConfig): bool =
  config.environment == "production"

proc isDevelopment*(config: AppConfig): bool =
  config.environment == "development"

# Safe config display (ไม่แสดง secrets)
proc toSafeJson*(config: AppConfig): JsonNode =
  %*{
    "port": config.port,
    "host": config.host,
    "environment": config.environment,
    "logLevel": config.logLevel,
    "db": {
      "url": config.db.url.split("@")[^1],  # ซ่อน credentials
      "maxConnections": config.db.maxConnections
    }
  }
```

---

## Step 249: Database Migrations ใน Docker

```sql
-- file: db/init.sql
-- สร้าง database schema เมื่อ container เริ่มต้น

CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "citext";

-- Users table
CREATE TABLE IF NOT EXISTS users (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    username CITEXT UNIQUE NOT NULL,
    email CITEXT UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    full_name VARCHAR(255),
    role VARCHAR(50) DEFAULT 'user',
    active BOOLEAN DEFAULT true,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Sessions table
CREATE TABLE IF NOT EXISTS sessions (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    token VARCHAR(255) UNIQUE NOT NULL,
    expires_at TIMESTAMPTZ NOT NULL,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    last_used TIMESTAMPTZ DEFAULT NOW()
);

-- Indexes
CREATE INDEX IF NOT EXISTS idx_users_email ON users(email);
CREATE INDEX IF NOT EXISTS idx_users_username ON users(username);
CREATE INDEX IF NOT EXISTS idx_sessions_token ON sessions(token);
CREATE INDEX IF NOT EXISTS idx_sessions_user_id ON sessions(user_id);
CREATE INDEX IF NOT EXISTS idx_sessions_expires ON sessions(expires_at);

-- Function to update updated_at
CREATE OR REPLACE FUNCTION update_updated_at()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Trigger for users
CREATE TRIGGER users_updated_at
    BEFORE UPDATE ON users
    FOR EACH ROW EXECUTE FUNCTION update_updated_at();

-- Insert admin user
INSERT INTO users (username, email, password_hash, full_name, role)
VALUES (
    'admin',
    'admin@example.com',
    '$2b$12$...',  -- bcrypt hash of 'admin_password'
    'System Admin',
    'admin'
) ON CONFLICT DO NOTHING;
```

```dockerfile
# file: db/Dockerfile.migrate
FROM golang:alpine AS migrate-builder
RUN go install -tags 'postgres' github.com/golang-migrate/migrate/v4/cmd/migrate@latest

FROM alpine:3.18
COPY --from=migrate-builder /go/bin/migrate /usr/local/bin/
COPY migrations/ /migrations/

CMD ["migrate", "-database", "$DATABASE_URL", "-path", "/migrations", "up"]
```

---

## Step 250: Health Checks ที่สมบูรณ์

```nim
# file: src/health.nim
import asyncdispatch, asynchttpserver, json, strformat
import times, os

type
  HealthStatus = enum
    hsHealthy = "healthy"
    hsDegraded = "degraded"
    hsUnhealthy = "unhealthy"

  CheckResult = object
    name: string
    status: HealthStatus
    message: string
    durationMs: float

  HealthReport = object
    status: HealthStatus
    checks: seq[CheckResult]
    timestamp: DateTime
    version: string
    uptime: float

var startTime = epochTime()

proc checkDatabase(dbUrl: string): Future[CheckResult] {.async.} =
  let start = epochTime()
  
  try:
    # ทดสอบ connection
    # let db = await openDatabase(dbUrl)
    # await db.query("SELECT 1")
    # db.close()
    
    # Simulate check
    await sleepAsync(5)
    
    return CheckResult(
      name: "database",
      status: hsHealthy,
      message: "Connection OK",
      durationMs: (epochTime() - start) * 1000
    )
  except Exception as e:
    return CheckResult(
      name: "database",
      status: hsUnhealthy,
      message: &"Connection failed: {e.msg}",
      durationMs: (epochTime() - start) * 1000
    )

proc checkRedis(redisUrl: string): Future[CheckResult] {.async.} =
  let start = epochTime()
  
  try:
    # ทดสอบ Redis connection
    # let r = await openRedis(redisUrl)
    # discard await r.ping()
    # r.close()
    
    await sleepAsync(2)
    
    return CheckResult(
      name: "redis",
      status: hsHealthy,
      message: "PONG",
      durationMs: (epochTime() - start) * 1000
    )
  except Exception as e:
    return CheckResult(
      name: "redis",
      status: hsDegraded,  # Redis failure = degraded, not unhealthy
      message: &"Redis unavailable: {e.msg}",
      durationMs: (epochTime() - start) * 1000
    )

proc checkDiskSpace(): CheckResult =
  # ตรวจสอบพื้นที่ disk
  let result = execProcess("df -h /")
  # Parse output...
  return CheckResult(
    name: "disk",
    status: hsHealthy,
    message: "Disk space OK",
    durationMs: 0.0
  )

proc checkMemory(): CheckResult =
  let result = execProcess("free -m")
  return CheckResult(
    name: "memory",
    status: hsHealthy,
    message: "Memory OK",
    durationMs: 0.0
  )

proc getHealthReport(dbUrl, redisUrl: string): Future[HealthReport] {.async.} =
  let dbCheck = await checkDatabase(dbUrl)
  let redisCheck = await checkRedis(redisUrl)
  let diskCheck = checkDiskSpace()
  let memCheck = checkMemory()
  
  let checks = @[dbCheck, redisCheck, diskCheck, memCheck]
  
  # Determine overall status
  var overallStatus = hsHealthy
  for check in checks:
    if check.status == hsUnhealthy:
      overallStatus = hsUnhealthy
      break
    elif check.status == hsDegraded and overallStatus == hsHealthy:
      overallStatus = hsDegraded
  
  return HealthReport(
    status: overallStatus,
    checks: checks,
    timestamp: now(),
    version: "1.0.0",
    uptime: epochTime() - startTime
  )

proc healthToJson(report: HealthReport): JsonNode =
  var checksJson = newJArray()
  for check in report.checks:
    checksJson.add(%*{
      "name": check.name,
      "status": $check.status,
      "message": check.message,
      "durationMs": check.durationMs
    })
  
  %*{
    "status": $report.status,
    "timestamp": report.timestamp.format("yyyy-MM-dd'T'HH:mm:ss'Z'"),
    "version": report.version,
    "uptime": report.uptime,
    "checks": checksJson
  }

proc handleHealth*(req: Request, dbUrl, redisUrl: string) {.async.} =
  if req.url.path == "/health":
    # Simple liveness probe
    await req.respond(Http200, $(%*{"status": "ok"}),
      newHttpHeaders([("Content-Type", "application/json")]))
  
  elif req.url.path == "/health/detailed":
    # Detailed health with all checks
    let report = await getHealthReport(dbUrl, redisUrl)
    let httpStatus = case report.status
      of hsHealthy: Http200
      of hsDegraded: Http200  # Still returning 200 for degraded
      of hsUnhealthy: Http503
    
    await req.respond(httpStatus, $healthToJson(report),
      newHttpHeaders([("Content-Type", "application/json")]))
  
  elif req.url.path == "/ready":
    # Readiness probe - app ready to receive traffic?
    let report = await getHealthReport(dbUrl, redisUrl)
    
    if report.status == hsUnhealthy:
      await req.respond(Http503, $(%*{"ready": false}),
        newHttpHeaders([("Content-Type", "application/json")]))
    else:
      await req.respond(Http200, $(%*{"ready": true}),
        newHttpHeaders([("Content-Type", "application/json")]))
```

---

## Step 251: Docker Secrets (Secure Configuration)

```yaml
# file: docker-compose.secrets.yml
version: '3.8'

secrets:
  db_password:
    file: ./secrets/db_password.txt
  jwt_secret:
    file: ./secrets/jwt_secret.txt
  redis_password:
    file: ./secrets/redis_password.txt

services:
  app:
    image: nim_app:latest
    secrets:
      - db_password
      - jwt_secret
      - redis_password
    environment:
      - DB_PASSWORD_FILE=/run/secrets/db_password
      - JWT_SECRET_FILE=/run/secrets/jwt_secret
      - REDIS_PASSWORD_FILE=/run/secrets/redis_password
    # ...

  postgres:
    image: postgres:15-alpine
    secrets:
      - db_password
    environment:
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
    # ...
```

```nim
# file: src/secrets.nim
import os, strutils, strformat

proc readSecretFile(path: string): string =
  ## อ่าน secret จาก file (Docker secrets pattern)
  if path.len == 0:
    return ""
  
  try:
    let content = readFile(path).strip()
    return content
  except IOError:
    return ""

proc getSecret*(envKey: string, defaultVal = ""): string =
  ## ดู secret จาก environment variable หรือ secret file
  
  # ตรวจสอบว่ามี _FILE suffix ไหม
  let fileEnvKey = envKey & "_FILE"
  let filePath = getEnv(fileEnvKey, "")
  
  if filePath.len > 0:
    let secret = readSecretFile(filePath)
    if secret.len > 0:
      return secret
  
  # Fall back to direct env var
  let directVal = getEnv(envKey, defaultVal)
  return directVal

# Usage:
# let dbPassword = getSecret("DB_PASSWORD")
# หรือ
# let dbPassword = getSecret("DB_PASSWORD", "default_only_for_dev")
```

---

## Step 252: Logging ที่ดีสำหรับ Container

```nim
# file: src/logger.nim
import json, times, strformat, os

type
  LogLevel = enum
    llDebug = "debug"
    llInfo = "info"
    llWarn = "warn"
    llError = "error"

  Logger = ref object
    level: LogLevel
    service: string
    env: string

proc newLogger*(service: string): Logger =
  let levelStr = getEnv("LOG_LEVEL", "info")
  let level = case levelStr
    of "debug": llDebug
    of "warn": llWarn
    of "error": llError
    else: llInfo
  
  Logger(
    level: level,
    service: service,
    env: getEnv("ENVIRONMENT", "development")
  )

proc shouldLog(logger: Logger, level: LogLevel): bool =
  ord(level) >= ord(logger.level)

proc log(logger: Logger, level: LogLevel, message: string, 
         extra: JsonNode = nil) =
  if not logger.shouldLog(level):
    return
  
  # Structured JSON logging (สำหรับ container log aggregation)
  let logEntry = %*{
    "timestamp": now().format("yyyy-MM-dd'T'HH:mm:ss.fff'Z'"),
    "level": $level,
    "service": logger.service,
    "environment": logger.env,
    "message": message
  }
  
  if not extra.isNil:
    for key, val in extra:
      logEntry[key] = val
  
  # ส่ง stdout (Docker จะ collect ให้)
  echo $logEntry

proc debug*(logger: Logger, msg: string, extra: JsonNode = nil) =
  logger.log(llDebug, msg, extra)

proc info*(logger: Logger, msg: string, extra: JsonNode = nil) =
  logger.log(llInfo, msg, extra)

proc warn*(logger: Logger, msg: string, extra: JsonNode = nil) =
  logger.log(llWarn, msg, extra)

proc error*(logger: Logger, msg: string, extra: JsonNode = nil) =
  logger.log(llError, msg, extra)

proc withRequest*(logger: Logger, method, path: string, 
                  statusCode, durationMs: int) =
  logger.info("HTTP request", %*{
    "method": method,
    "path": path,
    "statusCode": statusCode,
    "durationMs": durationMs
  })

# Usage example
when isMainModule:
  let log = newLogger("nim-backend")
  
  log.info("Server starting", %*{"port": 8080})
  log.debug("Debug message")
  log.warn("High memory usage", %*{"usageMb": 450})
  log.error("Database connection failed", %*{
    "host": "postgres",
    "error": "connection refused"
  })
  log.withRequest("GET", "/api/users", 200, 45)
```

---

## Step 253: Production Dockerfile ที่สมบูรณ์

```dockerfile
# file: Dockerfile.production
# ============ Build Stage ============
FROM nimlang/nim:2.0.0-alpine AS builder

LABEL stage=builder

WORKDIR /build

# Install build dependencies
RUN apk add --no-cache \
    musl-dev \
    openssl-dev \
    pcre-dev \
    postgresql-dev

# Copy dependency files first (cache optimization)
COPY myapp.nimble ./
COPY nimble.lock* ./

# Install Nim dependencies
RUN nimble install -d -y 2>&1

# Copy source
COPY src/ ./src/
COPY config/ ./config/

# Build static binary (for minimal runtime)
RUN nim c \
    -d:release \
    -d:ssl \
    -d:danger \
    --opt:speed \
    --gc:arc \
    --mm:arc \
    -d:lto \
    --passL:"-static" \
    -o:/build/server \
    src/server.nim

# Verify binary
RUN /build/server --version || true

# ============ Final Stage ============
FROM scratch

# Copy CA certificates for HTTPS
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/

# Copy binary
COPY --from=builder /build/server /app/server

# Copy config files
COPY --from=builder /build/config /app/config

WORKDIR /app

# Non-root user (in scratch, we use numeric UID)
USER 65534:65534

EXPOSE 8080

# Minimal health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 \
    CMD ["/app/server", "--health-check"]

CMD ["/app/server"]
```

---

## Step 254: CI/CD Pipeline (GitHub Actions)

```yaml
# file: .github/workflows/docker.yml
name: Build and Deploy

on:
  push:
    branches: [main, develop]
    tags: ['v*']
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:15-alpine
        env:
          POSTGRES_DB: test_db
          POSTGRES_USER: test_user
          POSTGRES_PASSWORD: test_pass
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
      redis:
        image: redis:7-alpine
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Nim
        uses: jiro4989/setup-nim-action@v1
        with:
          nim-version: '2.0.0'
      
      - name: Install dependencies
        run: nimble install -d -y
      
      - name: Run tests
        run: nimble test
        env:
          DATABASE_URL: postgres://test_user:test_pass@localhost:5432/test_db
          REDIS_URL: redis://localhost:6379
  
  build:
    needs: test
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Log in to Container Registry
        uses: docker/login-action@v2
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v4
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=ref,event=branch
            type=ref,event=pr
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
      
      - name: Build and push
        uses: docker/build-push-action@v4
        with:
          context: .
          push: ${{ github.event_name != 'pull_request' }}
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
  
  deploy:
    needs: build
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    environment: production
    
    steps:
      - name: Deploy to server
        uses: appleboy/ssh-action@master
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SERVER_SSH_KEY }}
          script: |
            cd /opt/myapp
            docker-compose pull
            docker-compose up -d --no-build
            docker image prune -f
```

---

## Step 255: Complete Dockerized Stack

```yaml
# file: docker-compose.complete.yml
# Production-ready complete stack

version: '3.8'

x-app-common: &app-common
  image: ${REGISTRY:-ghcr.io}/${IMAGE_NAME:-myorg/myapp}:${TAG:-latest}
  restart: unless-stopped
  env_file: .env
  networks:
    - backend
  depends_on:
    postgres:
      condition: service_healthy
    redis:
      condition: service_healthy
  deploy:
    resources:
      limits:
        cpus: '1.0'
        memory: 256M
      reservations:
        cpus: '0.25'
        memory: 64M

services:
  # ============ Load Balancer ============
  nginx:
    image: nginx:1.25-alpine
    container_name: nginx
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/sites:/etc/nginx/conf.d:ro
      - ./ssl:/etc/nginx/ssl:ro
      - static_files:/var/www/static:ro
      - nginx_logs:/var/log/nginx
    depends_on:
      - app1
      - app2
    networks:
      - frontend
      - backend
    healthcheck:
      test: ["CMD", "nginx", "-t"]
      interval: 30s
      timeout: 10s
      retries: 3

  # ============ App Instances ============
  app1:
    <<: *app-common
    container_name: app1
    environment:
      - INSTANCE_ID=1

  app2:
    <<: *app-common
    container_name: app2
    environment:
      - INSTANCE_ID=2

  # ============ Database ============
  postgres:
    image: postgres:15-alpine
    container_name: postgres
    restart: unless-stopped
    environment:
      POSTGRES_DB: ${DB_NAME:-myapp}
      POSTGRES_USER: ${DB_USER:-myapp_user}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_INITDB_ARGS: "--encoding=UTF-8 --lc-collate=C --lc-ctype=C"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./db/init.sql:/docker-entrypoint-initdb.d/00-init.sql:ro
      - postgres_logs:/var/log/postgresql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER:-myapp_user}"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 20s
    networks:
      - backend
    shm_size: 256mb

  # ============ Cache ============
  redis:
    image: redis:7-alpine
    container_name: redis
    restart: unless-stopped
    command: >
      redis-server
      --appendonly yes
      --appendfsync everysec
      --maxmemory 256mb
      --maxmemory-policy allkeys-lru
      --requirepass ${REDIS_PASSWORD}
      --loglevel notice
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "--no-auth-warning", "-a", "${REDIS_PASSWORD}", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - backend

  # ============ Database Backups ============
  postgres_backup:
    image: prodrigestivill/postgres-backup-local
    container_name: postgres_backup
    restart: unless-stopped
    depends_on:
      postgres:
        condition: service_healthy
    environment:
      POSTGRES_HOST: postgres
      POSTGRES_DB: ${DB_NAME:-myapp}
      POSTGRES_USER: ${DB_USER:-myapp_user}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      SCHEDULE: "0 2 * * *"  # Daily at 2 AM
      BACKUP_KEEP_DAYS: 7
      BACKUP_KEEP_WEEKS: 4
      BACKUP_KEEP_MONTHS: 6
    volumes:
      - backups:/backups
    networks:
      - backend

  # ============ Monitoring ============
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    restart: unless-stopped
    volumes:
      - ./monitoring/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.retention.time=15d'
    networks:
      - backend
    ports:
      - "9090:9090"

volumes:
  postgres_data:
    driver: local
  postgres_logs:
    driver: local
  redis_data:
    driver: local
  nginx_logs:
    driver: local
  static_files:
    driver: local
  backups:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: /opt/backups
  prometheus_data:
    driver: local

networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
    internal: true
```

```bash
# file: scripts/deploy.sh
#!/bin/bash
set -euo pipefail

echo "=== Deploying Nim Backend ==="

# Load environment
source .env

# Pull latest images
echo "Pulling images..."
docker-compose -f docker-compose.complete.yml pull

# Run database migrations
echo "Running migrations..."
docker-compose -f docker-compose.complete.yml run --rm \
  -e DATABASE_URL="${DATABASE_URL}" \
  app1 ./migrate

# Rolling update
echo "Updating app instances..."
docker-compose -f docker-compose.complete.yml up -d --no-deps --scale app1=2 app1
sleep 10

docker-compose -f docker-compose.complete.yml up -d --no-deps --scale app2=2 app2
sleep 10

# Cleanup
docker image prune -f

echo "=== Deployment Complete ==="
docker-compose -f docker-compose.complete.yml ps
```

---

## 📝 สรุป Part 18

| Step | หัวข้อ | สิ่งที่เรียนรู้ |
|------|--------|----------------|
| 241 | Basic Dockerfile | FROM, COPY, RUN, CMD |
| 242 | Multi-stage Build | Builder + minimal runtime |
| 243 | App Configuration | Environment variables in Nim |
| 244 | Nimble for Docker | Build tasks, dependencies |
| 245 | docker-compose | App + Postgres + Redis |
| 246 | With Nginx | Reverse proxy, load balancing |
| 247 | Nginx Config | SSL, rate limiting, WebSocket |
| 248 | Env Management | .env files, required vs optional |
| 249 | DB Migrations | Init SQL, migration patterns |
| 250 | Health Checks | Liveness, readiness probes |
| 251 | Docker Secrets | Secure credential management |
| 252 | Structured Logging | JSON logs for aggregation |
| 253 | Production Dockerfile | Static binary, scratch image |
| 254 | CI/CD Pipeline | GitHub Actions automation |
| 255 | Complete Stack | Full production setup |

---

## Navigation

- [← Part 17: Redis](part_17_redis.md)
- [→ Part 19: Testing](part_19_testing.md)
- [กลับ README](../README.md)
