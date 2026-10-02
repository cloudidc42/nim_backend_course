# Part 08: File I/O & System Interaction
## Steps 86-100: การทำงานกับ Files และ System

---

## 🎯 เป้าหมายของ Part นี้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
- อ่านและเขียนไฟล์ทุกรูปแบบ
- ทำงานกับ Directory และ Path
- ใช้ Environment Variables
- รัน System Commands
- จัดการ File Watchers
- อ่านและเขียน JSON, CSV, INI files

---

## Step 86: File Operations พื้นฐาน

```nim
import std/os, std/io

# Write to file
writeFile("hello.txt", "Hello, World!\nนี่คือทดสอบ")

# Read entire file
let content = readFile("hello.txt")
echo content

# File existence
echo fileExists("hello.txt")   # true
echo dirExists("/tmp")          # true

# Open file for reading
var f = open("hello.txt")
defer: close(f)  # จะ close อัตโนมัติเมื่อออกจาก scope

# Read line by line
var line = ""
while readLine(f, line):
  echo "  > " & line

# Or use lines iterator
for line in lines("hello.txt"):
  echo line

# Write with append
var fw = open("log.txt", fmAppend)
defer: close(fw)
fw.writeLine("New log entry")
fw.write("Data without newline")

# File modes
# fmRead    - read only
# fmWrite   - write (create/truncate)
# fmAppend  - append
# fmReadWrite - read and write
# fmReadWriteExisting - read/write existing file
```

---

## Step 87: Working with Paths

```nim
import std/os, std/paths

# Path operations
let basePath = getCurrentDir()
echo "Current dir: " & basePath

# Join paths
let dataPath = basePath / "data" / "users.json"
echo dataPath

# Path components
echo dataPath.splitPath()      # (dir, file)
echo dataPath.extractFilename() # users.json
echo dataPath.splitFile()       # (dir, name, ext)
echo dataPath.parentDir()       # parent directory

# Create directories
createDir("/tmp/testdir")
createDir("/tmp/testdir/sub/sub2")  # create recursively

# List directory
for entry in walkDir("/tmp"):
  echo entry.path & " (" & $entry.kind & ")"

# Recursive walk
for path in walkDirRec("/tmp/testdir"):
  echo path

# Delete
removeFile("hello.txt")
removeDir("/tmp/testdir")

# Copy/Move
copyFile("source.txt", "dest.txt")
moveFile("old.txt", "new.txt")
copyDir("src_dir", "dst_dir")

# Temp files/dirs
let tmpDir = getTempDir()
echo "Temp dir: " & tmpDir

# Get absolute path
echo absolutePath("../relative/path")

# Normalize path
echo normalizedPath("/a/b/../c/./d")  # /a/c/d

# File info
let info = getFileInfo("myfile.txt")
echo info.size
echo info.lastWriteTime
echo info.permissions
```

---

## Step 88: File Streaming

```nim
import std/streams, std/strutils

# String stream (in-memory)
var ss = newStringStream("Hello World\nLine 2\nLine 3")

# Read line
var line = ""
while ss.readLine(line):
  echo line

# Seek
ss.setPosition(0)  # go back to start
echo ss.readStr(5)  # "Hello"

# File stream
var fs = newFileStream("data.bin", fmWrite)
defer: fs.close()

# Write various types
fs.write(42'i32)       # write int32
fs.write(3.14'f64)     # write float64
fs.write("Hello")      # write string (no null terminator)
fs.write(true)         # write bool

# Binary file reading
var fsr = newFileStream("data.bin", fmRead)
defer: fsr.close()

let intVal = fsr.readInt32()
let floatVal = fsr.readFloat64()
echo intVal    # 42
echo floatVal  # 3.14

# Buffered writing for performance
proc writeLotsOfData(filename: string, data: seq[string]) =
  var buf = newStringStream()
  
  for line in data:
    buf.writeLine(line)
  
  writeFile(filename, buf.data)

# Large file processing
proc countLinesInFile(filename: string): int =
  result = 0
  for line in lines(filename):
    inc result

# Process CSV file
proc processCSV(filename: string): seq[seq[string]] =
  result = @[]
  for line in lines(filename):
    if line.len > 0:
      result.add(line.split(','))

# Write CSV
proc writeCSV(filename: string, data: seq[seq[string]]) =
  var content = ""
  for row in data:
    content &= row.join(",") & "\n"
  writeFile(filename, content)
```

---

## Step 89: JSON File Operations

```nim
import std/json, std/strformat

# JSON parsing
let jsonStr = """
{
  "name": "Alice",
  "age": 28,
  "email": "alice@example.com",
  "hobbies": ["reading", "coding", "hiking"],
  "address": {
    "city": "Bangkok",
    "country": "Thailand"
  },
  "active": true,
  "score": 98.5
}
"""

# Parse JSON
let jsonNode = parseJson(jsonStr)

# Access fields
echo jsonNode["name"].getStr()           # Alice
echo jsonNode["age"].getInt()            # 28
echo jsonNode["score"].getFloat()        # 98.5
echo jsonNode["active"].getBool()        # true
echo jsonNode["hobbies"][0].getStr()     # reading
echo jsonNode["address"]["city"].getStr() # Bangkok

# Safe access with default
echo jsonNode{"nickname"}.getStr("Unknown")  # Unknown (field doesn't exist)

# Iterate array
for hobby in jsonNode["hobbies"]:
  echo "  - " & hobby.getStr()

# Iterate object
for key, val in jsonNode.pairs:
  echo fmt"  {key}: {val.kind}"

# Create JSON
var newJson = %* {
  "id": 1,
  "name": "Bob",
  "scores": @[95, 87, 92],
  "metadata": {
    "createdAt": "2024-01-15",
    "version": "1.0"
  }
}

echo newJson.pretty()  # pretty-printed JSON

# Modify JSON
newJson["name"] = %"Robert"
newJson["scores"].add(%100)
newJson["metadata"]["updatedAt"] = %"2024-02-01"

# Convert to string
echo $newJson
echo newJson.pretty(indent = 4)

# Write JSON to file
writeFile("user.json", newJson.pretty())

# Read JSON from file
let loaded = parseFile("user.json")
echo loaded["name"].getStr()

# JSON arrays
let jsonArray = parseJson("""[1, 2, 3, 4, 5]""")
for item in jsonArray:
  write(stdout, $item.getInt() & " ")
echo ""

# Type-safe JSON using objects
type
  UserJson = object
    name: string
    age: int
    email: string

# Using jsony (faster JSON library)
# import jsony
# let user = parseJson[UserJson](jsonStr)
```

---

## Step 90: CSV Operations

```nim
import std/strutils, std/sequtils, std/tables

type
  CsvRow = seq[string]
  CsvFile = object
    headers: CsvRow
    rows: seq[CsvRow]

proc readCSV(filename: string, hasHeader: bool = true, delimiter: char = ','): CsvFile =
  result.headers = @[]
  result.rows = @[]
  
  var isFirst = true
  for line in lines(filename):
    if line.len == 0: continue
    
    # Parse CSV line (handles quoted fields)
    var fields: seq[string] = @[]
    var current = ""
    var inQuotes = false
    
    for c in line:
      if c == '"':
        inQuotes = not inQuotes
      elif c == delimiter and not inQuotes:
        fields.add(current.strip())
        current = ""
      else:
        current.add(c)
    fields.add(current.strip())
    
    if isFirst and hasHeader:
      result.headers = fields
      isFirst = false
    else:
      result.rows.add(fields)
      isFirst = false

proc writeCSV(filename: string, csv: CsvFile, delimiter: char = ',') =
  var lines: seq[string] = @[]
  
  if csv.headers.len > 0:
    lines.add(csv.headers.join($delimiter))
  
  for row in csv.rows:
    lines.add(row.join($delimiter))
  
  writeFile(filename, lines.join("\n"))

proc toTable(csv: CsvFile, rowIndex: int): Table[string, string] =
  result = initTable[string, string]()
  if rowIndex >= csv.rows.len: return
  
  for i, header in csv.headers:
    if i < csv.rows[rowIndex].len:
      result[header] = csv.rows[rowIndex][i]

# Demo usage
proc main() =
  # Create sample CSV
  var csv = CsvFile(
    headers: @["id", "name", "email", "score"],
    rows: @[
      @["1", "Alice", "alice@example.com", "95"],
      @["2", "Bob", "bob@example.com", "87"],
      @["3", "Charlie", "charlie@example.com", "92"],
    ]
  )
  
  writeCSV("users.csv", csv)
  
  # Read it back
  let loaded = readCSV("users.csv")
  echo "Headers: " & loaded.headers.join(", ")
  
  for row in loaded.rows:
    let record = toTable(loaded, loaded.rows.find(row))
    echo fmt"  {record[\"name\"]} ({record[\"email\"]}): {record[\"score\"]}"

main()
```

---

## Step 91: INI/Config Files

```nim
import std/parsecfg, std/strutils, std/tables

# Parse .ini/.cfg files
# Example config file content:
# [database]
# host = localhost
# port = 5432
# name = mydb
#
# [server]
# port = 8080
# debug = true

proc loadConfig(filename: string): Table[string, Table[string, string]] =
  result = initTable[string, Table[string, string]]()
  
  if not fileExists(filename):
    return
  
  var cfg = loadConfig(filename)
  
  for section, keys in cfg:
    result[section] = initTable[string, string]()
    for key, value in keys:
      result[section][key] = value

# Using std/parsecfg directly
var cfg = newConfig()
cfg.setSectionKey("database", "host", "localhost")
cfg.setSectionKey("database", "port", "5432")
cfg.setSectionKey("server", "port", "8080")
cfg.setSectionKey("server", "debug", "true")

cfg.writeConfig("app.cfg")

# Read config
let loadedCfg = loadConfig("app.cfg")
echo loadedCfg.getSectionValue("database", "host")  # localhost
echo loadedCfg.getSectionValue("server", "port")    # 8080

# Custom config parser (simpler format)
type
  AppConfig = object
    dbHost: string
    dbPort: int
    dbName: string
    dbUser: string
    dbPassword: string
    serverPort: int
    debug: bool
    logLevel: string

proc loadAppConfig(filename: string): AppConfig =
  result = AppConfig(
    dbHost: "localhost",
    dbPort: 5432,
    dbName: "mydb",
    dbUser: "admin",
    dbPassword: "",
    serverPort: 8080,
    debug: false,
    logLevel: "info"
  )
  
  if not fileExists(filename):
    return
  
  for line in lines(filename):
    let stripped = line.strip()
    if stripped.len == 0 or stripped.startsWith('#'):
      continue
    
    let eq = stripped.find('=')
    if eq < 0: continue
    
    let key = stripped[0..<eq].strip()
    let value = stripped[eq+1..^1].strip()
    
    case key.toLower()
    of "db_host":     result.dbHost = value
    of "db_port":     result.dbPort = parseInt(value)
    of "db_name":     result.dbName = value
    of "db_user":     result.dbUser = value
    of "db_password": result.dbPassword = value
    of "server_port": result.serverPort = parseInt(value)
    of "debug":       result.debug = parseBool(value)
    of "log_level":   result.logLevel = value
    else: discard

# Write config
proc saveAppConfig(config: AppConfig, filename: string) =
  var lines: seq[string] = @[
    "# Application Configuration",
    "# Generated automatically",
    "",
    "# Database Settings",
    fmt"db_host = {config.dbHost}",
    fmt"db_port = {config.dbPort}",
    fmt"db_name = {config.dbName}",
    fmt"db_user = {config.dbUser}",
    fmt"db_password = {config.dbPassword}",
    "",
    "# Server Settings",
    fmt"server_port = {config.serverPort}",
    fmt"debug = {config.debug}",
    fmt"log_level = {config.logLevel}",
  ]
  writeFile(filename, lines.join("\n"))
```

---

## Step 92: Environment Variables

```nim
import std/os, std/strutils, std/options

# Read environment variables
echo getEnv("HOME")           # /home/username
echo getEnv("PATH")           # system PATH
echo getEnv("UNKNOWN", "default")  # "default" (with fallback)

# Check if exists
echo existsEnv("HOME")  # true
echo existsEnv("FAKE_VAR")  # false

# Set/Delete (current process only)
putEnv("MY_VAR", "my_value")
echo getEnv("MY_VAR")  # my_value
delEnv("MY_VAR")
echo getEnv("MY_VAR")  # "" (deleted)

# dotenv loader
proc loadDotenv(filename: string = ".env") =
  if not fileExists(filename):
    return
  
  for line in lines(filename):
    let stripped = line.strip()
    
    # Skip comments and empty lines
    if stripped.len == 0 or stripped.startsWith('#'):
      continue
    
    # Skip lines without '='
    let eq = stripped.find('=')
    if eq < 0: continue
    
    let key = stripped[0..<eq].strip()
    var value = stripped[eq+1..^1].strip()
    
    # Remove quotes
    if value.startsWith('"') and value.endsWith('"'):
      value = value[1..^2]
    elif value.startsWith("'") and value.endsWith("'"):
      value = value[1..^2]
    
    # Don't override existing env vars
    if not existsEnv(key):
      putEnv(key, value)

# Config from environment
type
  DatabaseConfig = object
    host: string
    port: int
    name: string
    user: string
    password: string
    maxConnections: int

proc databaseConfigFromEnv(): DatabaseConfig =
  DatabaseConfig(
    host:           getEnv("DB_HOST", "localhost"),
    port:           parseInt(getEnv("DB_PORT", "5432")),
    name:           getEnv("DB_NAME", "mydb"),
    user:           getEnv("DB_USER", "postgres"),
    password:       getEnv("DB_PASSWORD", ""),
    maxConnections: parseInt(getEnv("DB_MAX_CONNECTIONS", "10"))
  )

# Required environment variable check
proc requireEnv(key: string): string =
  let value = getEnv(key)
  if value.len == 0:
    raise newException(ValueError, "Required environment variable not set: " & key)
  return value

# Application config from environment
type
  AppConfig2 = object
    port: int
    host: string
    debug: bool
    secretKey: string
    databaseUrl: string
    redisUrl: string
    logLevel: string

proc loadConfig2(): AppConfig2 =
  loadDotenv()  # Load .env file first
  
  return AppConfig2(
    port:        parseInt(getEnv("PORT", "8080")),
    host:        getEnv("HOST", "0.0.0.0"),
    debug:       getEnv("DEBUG", "false").parseBool(),
    secretKey:   requireEnv("SECRET_KEY"),
    databaseUrl: getEnv("DATABASE_URL", "sqlite://app.db"),
    redisUrl:    getEnv("REDIS_URL", "redis://localhost:6379"),
    logLevel:    getEnv("LOG_LEVEL", "info")
  )
```

---

## Step 93: System Commands

```nim
import std/osproc, std/os, std/strutils

# Run a command and get output
let (output, exitCode) = execCmdEx("echo Hello from shell")
echo "Output: " & output
echo "Exit code: " & $exitCode

# Run command and check success
proc runCommand(cmd: string): bool =
  let code = execCmd(cmd)
  return code == 0

discard runCommand("ls -la")

# Get output of command
let nimVersion = execProcess("nim --version")
echo nimVersion.splitLines()[0]  # First line only

# Run command in different directory
var process = startProcess(
  "ls",
  workingDir = "/tmp",
  args = @["-la"],
  options = {poUsePath}
)

defer: process.close()

echo process.outputStream().readAll()
echo "Exit code: " & $process.waitForExit()

# Pipe input/output
let (output2, errOutput, code) = execCmdEx("echo 'input' | rev")
echo output2  # tupni

# Environment for subprocess
var env = newStringTable()
env["MY_VAR"] = "hello"
env["PATH"] = getEnv("PATH")

var envProcess = startProcess(
  "bash",
  args = @["-c", "echo $MY_VAR"],
  env = env,
  options = {poUsePath}
)

defer: envProcess.close()
echo envProcess.outputStream().readAll()

# Timeout handling
proc runWithTimeout(cmd: string, timeout: int = 30): (string, bool) =
  var process = startProcess(cmd, options = {poUsePath, poEvalCommand})
  defer: process.close()
  
  var elapsed = 0
  while elapsed < timeout * 1000:
    if not process.running():
      return (process.outputStream().readAll(), true)
    sleep(100)
    elapsed += 100
  
  process.terminate()
  return ("", false)

let (output3, success) = runWithTimeout("sleep 1 && echo done", 5)
echo "Success: " & $success
echo "Output: " & output3
```

---

## Step 94: File Watching

```nim
import std/os, std/times, std/tables

# Simple file watcher using polling
type
  FileWatcher = object
    watchedFiles: Table[string, Time]
    watchedDirs: Table[string, Time]

proc newFileWatcher(): FileWatcher =
  FileWatcher(
    watchedFiles: initTable[string, Time](),
    watchedDirs: initTable[string, Time]()
  )

proc watch(watcher: var FileWatcher, path: string) =
  if fileExists(path):
    watcher.watchedFiles[path] = getFileInfo(path).lastWriteTime
  elif dirExists(path):
    watcher.watchedDirs[path] = getFileInfo(path).lastWriteTime

proc checkChanges(watcher: var FileWatcher): seq[string] =
  result = @[]
  
  for path, oldTime in watcher.watchedFiles.mpairs:
    if not fileExists(path):
      result.add(path & " (deleted)")
      watcher.watchedFiles.del(path)
    else:
      let newTime = getFileInfo(path).lastWriteTime
      if newTime > oldTime:
        result.add(path & " (modified)")
        watcher.watchedFiles[path] = newTime
  
  for path, oldTime in watcher.watchedDirs.mpairs:
    let newTime = getFileInfo(path).lastWriteTime
    if newTime > oldTime:
      result.add(path & " (directory changed)")
      watcher.watchedDirs[path] = newTime

# Example: watch config file for hot reload
proc watchConfig(configFile: string, onReload: proc(config: string)) =
  var lastModified = getFileInfo(configFile).lastWriteTime
  
  echo fmt"Watching {configFile} for changes..."
  
  while true:
    sleep(1000)  # check every second
    
    if not fileExists(configFile):
      echo "Config file deleted!"
      break
    
    let currentModified = getFileInfo(configFile).lastWriteTime
    if currentModified > lastModified:
      echo "Config changed, reloading..."
      lastModified = currentModified
      onReload(readFile(configFile))
```

---

## Step 95: Log Management

```nim
import std/times, std/strformat, std/os, std/strutils

type
  LogLevel = enum
    Trace, Debug, Info, Warning, Error, Critical

  LogTarget = enum
    Console, File, Both

  Logger = object
    level: LogLevel
    target: LogTarget
    filename: string
    maxSize: int  # bytes
    backupCount: int

proc newLogger(
  level: LogLevel = Info,
  target: LogTarget = Console,
  filename: string = "app.log",
  maxSize: int = 10_000_000,  # 10MB
  backupCount: int = 5
): Logger =
  Logger(
    level: level,
    target: target,
    filename: filename,
    maxSize: maxSize,
    backupCount: backupCount
  )

proc shouldRotate(logger: Logger): bool =
  if not fileExists(logger.filename): return false
  return getFileSize(logger.filename) > logger.maxSize

proc rotate(logger: Logger) =
  # Rotate log files
  for i in countdown(logger.backupCount - 1, 1):
    let src = logger.filename & "." & $i
    let dst = logger.filename & "." & $(i + 1)
    if fileExists(src):
      moveFile(src, dst)
  
  if fileExists(logger.filename):
    moveFile(logger.filename, logger.filename & ".1")

proc log(logger: Logger, level: LogLevel, msg: string, context: string = "") =
  if level < logger.level: return
  
  let timestamp = now().format("yyyy-MM-dd HH:mm:ss")
  let levelStr = case level
    of Trace: "TRACE"
    of Debug: "DEBUG"
    of Info:  "INFO "
    of Warning: "WARN "
    of Error: "ERROR"
    of Critical: "CRIT "
  
  var logLine = fmt"[{timestamp}] [{levelStr}] {msg}"
  if context.len > 0:
    logLine &= fmt" | context={context}"
  
  if logger.target in {Console, Both}:
    let colorCode = case level
      of Trace, Debug: "\e[37m"    # Gray
      of Info:         "\e[32m"    # Green
      of Warning:      "\e[33m"    # Yellow
      of Error:        "\e[31m"    # Red
      of Critical:     "\e[35m"    # Magenta
    echo colorCode & logLine & "\e[0m"
  
  if logger.target in {File, Both}:
    if logger.shouldRotate():
      logger.rotate()
    
    let f = open(logger.filename, fmAppend)
    defer: close(f)
    f.writeLine(logLine)

# Usage
var appLogger = newLogger(
  level = Debug,
  target = Both,
  filename = "app.log"
)

proc trace(msg: string) = appLogger.log(Trace, msg)
proc debug(msg: string) = appLogger.log(Debug, msg)
proc info(msg: string)  = appLogger.log(Info, msg)
proc warn(msg: string)  = appLogger.log(Warning, msg)
proc error(msg: string) = appLogger.log(Error, msg)
proc critical(msg: string) = appLogger.log(Critical, msg)

info("Application started")
debug("Debug mode is on")
warn("High memory usage detected")
error("Database connection failed")
```

---

## Step 96-100: Real-World App - Config Manager

```nim
# config_manager.nim
# Complete application configuration manager

import std/os, std/json, std/strutils, std/tables, std/options,
       std/strformat, std/times

type
  ConfigSource = enum
    Default, EnvVar, File, Override

  ConfigValue = object
    value: JsonNode
    source: ConfigSource
    lastUpdated: DateTime

  ConfigManager = object
    values: Table[string, ConfigValue]
    schema: Table[string, JsonNode]  # default values and types
    watchers: seq[proc(key, value: string)]
    filename: string
    loaded: bool

# ==============================
# Schema Definition
# ==============================

proc newConfigManager(filename: string = "config.json"): ConfigManager =
  result = ConfigManager(
    values: initTable[string, ConfigValue](),
    schema: initTable[string, JsonNode](),
    watchers: @[],
    filename: filename,
    loaded: false
  )
  
  # Define schema with defaults
  result.schema = {
    "server.host":       %"0.0.0.0",
    "server.port":       %8080,
    "server.debug":      %false,
    "server.workers":    %4,
    
    "database.host":     %"localhost",
    "database.port":     %5432,
    "database.name":     %"myapp",
    "database.user":     %"postgres",
    "database.password": %"",
    "database.pool_size": %10,
    
    "redis.host":        %"localhost",
    "redis.port":        %6379,
    "redis.db":          %0,
    
    "auth.secret_key":   %"",
    "auth.token_ttl":    %3600,
    "auth.refresh_ttl":  %86400,
    
    "log.level":         %"info",
    "log.file":          %"",
    "log.max_size":      %10485760,
    
    "cache.ttl":         %300,
    "cache.max_size":    %1000,
  }.toTable()

# ==============================
# Loading from different sources
# ==============================

proc loadDefaults(cm: var ConfigManager) =
  for key, defaultVal in cm.schema:
    cm.values[key] = ConfigValue(
      value: defaultVal,
      source: Default,
      lastUpdated: now()
    )

proc loadFromEnv(cm: var ConfigManager) =
  # Map env vars to config keys
  let envMapping = {
    "HOST":         "server.host",
    "PORT":         "server.port",
    "DEBUG":        "server.debug",
    "DB_HOST":      "database.host",
    "DB_PORT":      "database.port",
    "DB_NAME":      "database.name",
    "DB_USER":      "database.user",
    "DB_PASSWORD":  "database.password",
    "REDIS_HOST":   "redis.host",
    "REDIS_PORT":   "redis.port",
    "SECRET_KEY":   "auth.secret_key",
    "LOG_LEVEL":    "log.level",
  }.toTable()
  
  for envKey, configKey in envMapping:
    let envVal = getEnv(envKey)
    if envVal.len > 0:
      let jsonVal = try: parseJson(envVal) except: %envVal
      cm.values[configKey] = ConfigValue(
        value: jsonVal,
        source: EnvVar,
        lastUpdated: now()
      )

proc loadFromFile(cm: var ConfigManager) =
  if not fileExists(cm.filename):
    return
  
  try:
    let content = readFile(cm.filename)
    let jsonData = parseJson(content)
    
    proc flatten(node: JsonNode, prefix: string = "") =
      case node.kind
      of JObject:
        for key, val in node.pairs:
          let fullKey = if prefix.len > 0: prefix & "." & key else: key
          if val.kind == JObject:
            flatten(val, fullKey)
          else:
            cm.values[fullKey] = ConfigValue(
              value: val,
              source: File,
              lastUpdated: now()
            )
      else:
        discard
    
    flatten(jsonData)
    cm.loaded = true
  
  except JsonParsingError as e:
    echo fmt"Config file parse error: {e.msg}"

# ==============================
# Accessing Config Values
# ==============================

proc getString(cm: ConfigManager, key: string, default: string = ""): string =
  if key in cm.values:
    return cm.values[key].value.getStr(default)
  return default

proc getInt(cm: ConfigManager, key: string, default: int = 0): int =
  if key in cm.values:
    return cm.values[key].value.getInt(default)
  return default

proc getFloat(cm: ConfigManager, key: string, default: float = 0.0): float =
  if key in cm.values:
    return cm.values[key].value.getFloat(default)
  return default

proc getBool(cm: ConfigManager, key: string, default: bool = false): bool =
  if key in cm.values:
    return cm.values[key].value.getBool(default)
  return default

proc set(cm: var ConfigManager, key: string, value: JsonNode) =
  cm.values[key] = ConfigValue(
    value: value,
    source: Override,
    lastUpdated: now()
  )
  
  # Notify watchers
  for watcher in cm.watchers:
    watcher(key, $value)

proc onChanged(cm: var ConfigManager, handler: proc(key, value: string)) =
  cm.watchers.add(handler)

# ==============================
# Config Validation
# ==============================

proc validate(cm: ConfigManager): seq[string] =
  result = @[]
  
  if cm.getString("auth.secret_key").len < 32:
    result.add("auth.secret_key must be at least 32 characters")
  
  let port = cm.getInt("server.port")
  if port < 1 or port > 65535:
    result.add(fmt"server.port {port} is out of range (1-65535)")
  
  let dbPort = cm.getInt("database.port")
  if dbPort < 1 or dbPort > 65535:
    result.add(fmt"database.port {dbPort} is out of range")
  
  let logLevel = cm.getString("log.level")
  if logLevel notin @["trace", "debug", "info", "warning", "error", "critical"]:
    result.add(fmt"log.level '{logLevel}' is invalid")

# ==============================
# Display
# ==============================

proc printConfig(cm: ConfigManager) =
  echo "\n" & "═".repeat(60)
  echo "  Application Configuration"
  echo "═".repeat(60)
  
  var lastSection = ""
  
  for key in cm.values.keys.toSeq().sorted():
    let parts = key.split('.')
    let section = if parts.len > 1: parts[0] else: ""
    
    if section != lastSection:
      echo fmt"\n  [{section.toUpper()}]"
      lastSection = section
    
    let config = cm.values[key]
    let sourceStr = case config.source
      of Default: "default"
      of EnvVar:  "env"
      of File:    "file"
      of Override: "override"
    
    let valueStr = if key.contains("password") or key.contains("secret"):
      "***"
    else:
      $config.value
    
    echo fmt"  {key:<30} = {valueStr:<20} [{sourceStr}]"

# ==============================
# Main Demo
# ==============================

proc main() =
  echo "╔══════════════════════════════════════╗"
  echo "║       Config Manager Demo             ║"
  echo "╚══════════════════════════════════════╝"
  
  var config = newConfigManager("config.json")
  
  # Load from all sources (in priority order)
  config.loadDefaults()      # 1. Defaults (lowest priority)
  config.loadFromFile()      # 2. Config file
  config.loadFromEnv()       # 3. Environment variables (highest)
  
  # Set up change monitoring
  config.onChanged(proc(key, value: string) =
    echo fmt"Config changed: {key} = {value}"
  )
  
  # Override a value
  config.set("server.debug", %true)
  
  # Display config
  config.printConfig()
  
  # Validate
  let errors = config.validate()
  if errors.len > 0:
    echo "\n⚠️  Validation warnings:"
    for err in errors:
      echo fmt"  - {err}"
  else:
    echo "\n✅ Configuration is valid"
  
  # Access values
  echo "\n--- Configuration Summary ---"
  echo fmt"Server: {config.getString(\"server.host\")}:{config.getInt(\"server.port\")}"
  echo fmt"Database: {config.getString(\"database.host\")}:{config.getInt(\"database.port\")}/{config.getString(\"database.name\")}"
  echo fmt"Debug mode: {config.getBool(\"server.debug\")}"
  echo fmt"Log level: {config.getString(\"log.level\")}"

main()
```

---

## 📝 สรุป Part 08

| Step | หัวข้อ |
|------|--------|
| 86 | File read/write basics |
| 87 | Path operations |
| 88 | File streaming, binary files |
| 89 | JSON files |
| 90 | CSV operations |
| 91 | INI/Config files |
| 92 | Environment variables, dotenv |
| 93 | System commands |
| 94 | File watching |
| 95 | Log management |
| 96-100 | Real-world: Config Manager |

---

**← [Part 07: Strings](part_07_strings.md) | [Part 09: Error Handling →](part_09_error_handling.md)**
