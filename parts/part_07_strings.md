# Part 07: Strings & String Operations
## Steps 71-85: การจัดการ String อย่างมืออาชีพ

---

## 🎯 เป้าหมายของ Part นี้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
- จัดการ strings ขั้นสูง
- ใช้ Regular Expressions
- Parse และ Format strings
- จัดการ Unicode/UTF-8
- สร้าง Template Engine ง่ายๆ
- Build string utilities สำหรับ backend

---

## Step 71: String Basics ทบทวน

```nim
import std/strutils, std/strformat, std/unicode

# String fundamentals
var s = "Hello, สวัสดี World!"

echo s.len          # byte length (not char count!)
echo s.runeLen()    # Unicode character count

# Byte access
echo s[0]           # 'H' (char)
echo s[0..4]        # "Hello" (substring by byte)

# Safe Unicode access
for r in s.runes:
  write(stdout, $r & "|")
echo ""

# String methods
var text = "  Hello World  "
echo text.strip()             # "Hello World"
echo text.strip(leading=true, trailing=false)  # "Hello World  "
echo text.toLower()           # "  hello world  "
echo text.toUpper()           # "  HELLO WORLD  "
echo text.capitalize()        # "  hello world  " (capitalize first only)
echo "hello world".capitalizeAscii()  # "Hello world"

# Split
var csv = "apple,banana,,cherry,  date  "
var parts = csv.split(",")      # includes empty strings
echo parts  # @["apple", "banana", "", "cherry", "  date  "]

var cleanParts = csv.split(",")
  .filterIt(it.strip().len > 0)
  .mapIt(it.strip())
echo cleanParts  # @["apple", "banana", "cherry", "date"]

# Split by whitespace
var sentence = "  hello   world   nim  "
var words = sentence.splitWhitespace()
echo words  # @["hello", "world", "nim"]

# Join
echo @["a", "b", "c"].join(", ")  # a, b, c
echo @["a", "b", "c"].join("")     # abc

# Replace
var str = "Hello World World"
echo str.replace("World", "Nim")  # Hello Nim Nim
echo str.replace("World", "Nim", maxReplacements = 1)  # Hello Nim World

# Count
echo "hello world hello".count("hello")  # 2

# Find/Contains
echo str.find("World")      # 6 (first occurrence index)
echo str.rfind("World")     # 12 (last occurrence index)
echo str.contains("World")  # true
echo str.startsWith("Hello") # true
echo str.endsWith("World")   # true
```

---

## Step 72: String Formatting

```nim
import std/strformat, std/strutils

# fmt string (f-string like)
let name = "Alice"
let age = 28
let score = 98.567

echo fmt"Name: {name}, Age: {age}"
echo fmt"Score: {score:.2f}"           # 98.57
echo fmt"Score: {score:10.2f}"         # right-aligned width 10
echo fmt"Score: {score:<10.2f}"        # left-aligned
echo fmt"Name: {name:>10}"            # right-align string
echo fmt"Name: {name:^10}"            # center-align string
echo fmt"Hex: {255:#x}"               # 0xff
echo fmt"Hex: {255:X}"                # FF
echo fmt"Bin: {42:b}"                 # 101010
echo fmt"Oct: {42:o}"                 # 52
echo fmt"Sci: {12345.678:e}"          # 1.234568e+04
echo fmt"Pct: {0.8567:.1%}"          # 85.7%
echo fmt"Fill: {'*':*>10}"           # ******* (fill with *)
echo fmt"Num: {1234567:,}"            # 1,234,567

# String format with $
echo "Value: " & $42            # Value: 42
echo "Pi: " & $3.14159          # Pi: 3.14159

# sprintf-like
import std/strformat
proc format(tmpl: string, args: varargs[string]): string =
  var result = tmpl
  var argIdx = 0
  while "{}" in result and argIdx < args.len:
    result = result.replace("{}", args[argIdx], 1)
    inc argIdx
  return result

echo format("Hello {} from {}!", "World", "Nim")
# Hello World from Nim!

# Multi-line format
var table = fmt"""
┌─────────────┬─────────┐
│ {"Name":^13} │ {"Score":^7} │
├─────────────┼─────────┤
│ {"Alice":^13} │ {98:^7} │
│ {"Bob":^13} │ {87:^7} │
│ {"Charlie":^13} │ {92:^7} │
└─────────────┴─────────┘"""

echo table
```

---

## Step 73: Regular Expressions

```nim
import std/re

# Basic matching
let pattern = re"[A-Z][a-z]+"
let text = "Hello World Nim Programming"

# Check if matches
echo text.match(pattern) != -1   # depends on position

# Find first match
let m = text.find(pattern)
if m >= 0:
  echo text[m..^1]  # match

# findAll - find all matches
for match in text.findAll(pattern):
  write(stdout, match & " ")
echo ""  # Hello World Nim Programming

# Groups
var email = "contact: alice@example.com, bob@test.org"
var emailPattern = re"(\w+)@(\w+\.\w+)"

var matches: array[3, string]
if email.match(emailPattern, matches):
  echo "Full: " & matches[0]   # alice@example.com
  echo "User: " & matches[1]   # alice
  echo "Domain: " & matches[2] # example.com

# findAllBounds
for (start, stop) in email.findAllBounds(emailPattern):
  echo email[start..stop]

# Replace
var cleaned = "Hello   World   Nim"
echo cleaned.replace(re"\s+", " ")  # Hello World Nim

# Replace with capture groups
var date = "2024-01-15"
echo date.replace(re"(\d{4})-(\d{2})-(\d{2})", "$3/$2/$1")
# 15/01/2024

# Split by pattern
var parts = "one1two2three3four".split(re"\d")
echo parts  # @["one", "two", "three", "four"]

# Validation patterns
proc validateEmail(email: string): bool =
  email.match(re"^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$") >= 0

proc validatePhone(phone: string): bool =
  phone.match(re"^[\+]?[(]?[0-9]{3}[)]?[-\s\.]?[0-9]{3}[-\s\.]?[0-9]{4,6}$") >= 0

proc validateUrl(url: string): bool =
  url.match(re"^https?://[^\s/$.?#].[^\s]*$") >= 0

echo validateEmail("alice@example.com")  # true
echo validateEmail("not-an-email")       # false
echo validatePhone("+1-555-1234")        # true
echo validateUrl("https://nim-lang.org") # true
```

---

## Step 74: String Parsing

```nim
import std/strutils, std/strscans, std/parseutils

# Manual parsing
proc parseKeyValue(line: string): (string, string) =
  let pos = line.find('=')
  if pos < 0:
    return ("", "")
  return (line[0..<pos].strip(), line[pos+1..^1].strip())

let kv = parseKeyValue("name = Alice Smith")
echo kv  # ("name", "Alice Smith")

# parseutils
var s = "42 hello 3.14"
var intVal: int
var floatVal: float
var strVal: string

var i = 0
i += parseInt(s, intVal, i)  # parse int starting at i=0
i += skipWhitespace(s, i)
i += parseIdent(s, strVal, i)  # parse identifier
i += skipWhitespace(s, i)
i += parseFloat(s, floatVal, i)

echo intVal    # 42
echo strVal    # hello
echo floatVal  # 3.14

# strscans - scanner
import std/strscans

var input = "John 28 engineer"
var name2: string
var age2: int
var occupation: string

if input.scanf("%s %i %s", name2, age2, occupation):
  echo fmt"Name: {name2}, Age: {age2}, Job: {occupation}"

# CSV parser
proc parseCSVLine(line: string): seq[string] =
  result = @[]
  var current = ""
  var inQuotes = false
  
  for i, c in line:
    if c == '"':
      inQuotes = not inQuotes
    elif c == ',' and not inQuotes:
      result.add(current.strip(chars = {'"'}))
      current = ""
    else:
      current.add(c)
  
  result.add(current.strip(chars = {'"'}))

let csvLine = """John,"Smith, Jr.",28,"New York, NY""""
let fields = parseCSVLine(csvLine)
for f in fields:
  echo "  Field: '" & f & "'"

# JSON-like value parser (simplified)
type
  JsonValueKind = enum
    jNull, jBool, jInt, jFloat, jString

  JsonValue = object
    case kind: JsonValueKind
    of jNull: discard
    of jBool: boolVal: bool
    of jInt: intVal: int
    of jFloat: floatVal: float
    of jString: strVal: string

proc parseSimpleValue(s: string): JsonValue =
  let trimmed = s.strip()
  
  if trimmed == "null":
    return JsonValue(kind: jNull)
  
  if trimmed == "true":
    return JsonValue(kind: jBool, boolVal: true)
  
  if trimmed == "false":
    return JsonValue(kind: jBool, boolVal: false)
  
  if trimmed.startsWith('"') and trimmed.endsWith('"'):
    return JsonValue(kind: jString, strVal: trimmed[1..^2])
  
  try:
    let intVal = parseInt(trimmed)
    return JsonValue(kind: jInt, intVal: intVal)
  except ValueError:
    discard
  
  try:
    let floatVal = parseFloat(trimmed)
    return JsonValue(kind: jFloat, floatVal: floatVal)
  except ValueError:
    discard
  
  return JsonValue(kind: jString, strVal: trimmed)
```

---

## Step 75: String Builder Pattern

```nim
import std/strutils, std/strformat

# String Builder
type
  StringBuilder = object
    parts: seq[string]
    totalLen: int

proc newStringBuilder(capacity: int = 16): StringBuilder =
  StringBuilder(parts: newSeqOfCap[string](capacity), totalLen: 0)

proc add(sb: var StringBuilder, s: string) =
  sb.parts.add(s)
  sb.totalLen += s.len

proc addLine(sb: var StringBuilder, s: string = "") =
  sb.add(s & "\n")

proc addFmt(sb: var StringBuilder, fmt: string) =
  sb.add(fmt)

proc toString(sb: StringBuilder): string =
  result = newStringOfCap(sb.totalLen)
  for part in sb.parts:
    result.add(part)

# Usage - Building HTML
proc buildHtmlTable(headers: seq[string], rows: seq[seq[string]]): string =
  var sb = newStringBuilder()
  
  sb.addLine("<table>")
  sb.addLine("  <thead>")
  sb.addLine("    <tr>")
  for h in headers:
    sb.add(fmt"      <th>{h}</th>\n")
  sb.addLine("    </tr>")
  sb.addLine("  </thead>")
  sb.addLine("  <tbody>")
  
  for row in rows:
    sb.addLine("    <tr>")
    for cell in row:
      sb.add(fmt"      <td>{cell}</td>\n")
    sb.addLine("    </tr>")
  
  sb.addLine("  </tbody>")
  sb.addLine("</table>")
  
  return sb.toString()

let headers = @["Name", "Age", "City"]
let rows = @[
  @["Alice", "28", "Bangkok"],
  @["Bob", "32", "Chiang Mai"],
  @["Charlie", "25", "Phuket"],
]

echo buildHtmlTable(headers, rows)
```

---

## Step 76: Unicode & Internationalization

```nim
import std/unicode, std/strutils

# UTF-8 basics
var thai = "สวัสดีครับ"
var emoji = "Hello 😊 World 🌍"
var mixed = "Nim ภาษา Programming"

# Length differences
echo thai.len          # bytes
echo thai.runeLen()    # characters (runes)
echo emoji.len         # bytes
echo emoji.runeLen()   # characters

# Rune operations
for i, rune in thai.toRunes():
  echo fmt"  [{i}] U+{int(rune):04X} = '{rune}'"

# String manipulation with Unicode
proc reverseString(s: string): string =
  var runes = s.toRunes()
  runes.reverse()
  return $runes

echo reverseString("Hello สวัสดี")  # สวัสดี olleH (reversed runes)

# Unicode normalization
proc normalizeUsername(name: string): string =
  result = name.toLower()
  # Remove accents (simplified)
  result = result.replace("é", "e")
           .replace("è", "e")
           .replace("ê", "e")
           .replace("ñ", "n")
           .replace("ü", "u")

echo normalizeUsername("Ünder")  # under (simplified)

# Count by Unicode categories
proc countByType(s: string): (int, int, int) =
  var letters = 0
  var digits = 0
  var others = 0
  
  for r in s.runes:
    if isAlpha(r): inc letters
    elif isNumber(r): inc digits
    else: inc others
  
  return (letters, digits, others)

let (l, d, o) = countByType("Hello 123 สวัสดี!")
echo fmt"Letters: {l}, Digits: {d}, Others: {o}"

# Thai text utilities
proc countThaiChars(s: string): int =
  result = 0
  for r in s.runes:
    let code = int(r)
    if code >= 0x0E00 and code <= 0x0E7F:
      inc result

echo countThaiChars("Hello สวัสดี World")  # 6
```

---

## Step 77: Template Engine

```nim
import std/strutils, std/tables, std/re

# Simple template engine
type
  TemplateEngine = object
    templates: Table[string, string]

proc newTemplateEngine(): TemplateEngine =
  TemplateEngine(templates: initTable[string, string]())

proc registerTemplate(engine: var TemplateEngine, name, tmpl: string) =
  engine.templates[name] = tmpl

proc render(engine: TemplateEngine, name: string, vars: Table[string, string]): string =
  if name notin engine.templates:
    raise newException(KeyError, "Template not found: " & name)
  
  result = engine.templates[name]
  
  # Replace {{variable}} patterns
  for varName, varValue in vars:
    result = result.replace("{{" & varName & "}}", varValue)
  
  # Remove unreplaced variables
  result = result.replace(re"\{\{[^}]+\}\}", "")

proc renderInline(tmpl: string, vars: Table[string, string]): string =
  result = tmpl
  for varName, varValue in vars:
    result = result.replace("{{" & varName & "}}", varValue)

# Usage
var engine = newTemplateEngine()

engine.registerTemplate("welcome_email", """
Subject: Welcome to {{app_name}}!

Dear {{user_name}},

Welcome to {{app_name}}! Your account has been created.

Username: {{username}}
Email: {{email}}

Please login at: {{login_url}}

Best regards,
The {{app_name}} Team
""")

engine.registerTemplate("password_reset", """
Subject: Password Reset Request

Dear {{user_name}},

You requested a password reset. Click the link below:
{{reset_link}}

This link expires in {{expiry}} minutes.

If you didn't request this, ignore this email.
""")

let emailVars = {
  "app_name": "MyApp",
  "user_name": "Alice",
  "username": "alice123",
  "email": "alice@example.com",
  "login_url": "https://myapp.com/login"
}.toTable()

echo engine.render("welcome_email", emailVars)

# HTML template with conditionals (simple version)
proc renderConditional(tmpl: string, vars: Table[string, string], flags: Table[string, bool]): string =
  result = tmpl
  
  # Process {{#if flag}}...{{/if}} blocks
  var processed = true
  while processed:
    processed = false
    let pattern = re"\{\{#if (\w+)\}\}(.*?)\{\{/if\}\}"
    var matches: array[3, string]
    
    if result.match(pattern, matches) >= 0:
      let flagName = matches[1]
      let content = matches[2]
      let isTrue = flags.getOrDefault(flagName, false)
      
      let full = "{{#if " & flagName & "}}" & content & "{{/if}}"
      result = result.replace(full, if isTrue: content else: "")
      processed = true
  
  # Replace variables
  for varName, varValue in vars:
    result = result.replace("{{" & varName & "}}", varValue)
```

---

## Step 78: String Security

```nim
import std/strutils, std/htmlparser, std/xmltree

# HTML Escaping - prevent XSS
proc escapeHtml(s: string): string =
  result = s
    .replace("&", "&amp;")   # Must be first!
    .replace("<", "&lt;")
    .replace(">", "&gt;")
    .replace('"', "&quot;")
    .replace("'", "&#39;")

echo escapeHtml("<script>alert('XSS')</script>")
# &lt;script&gt;alert(&#39;XSS&#39;)&lt;/script&gt;

# SQL Escaping (use parameterized queries instead!)
proc escapeSql(s: string): string =
  result = s
    .replace("\\", "\\\\")
    .replace("'", "''")
    .replace("\"", "\\\"")
    .replace("\0", "\\0")
    .replace("\n", "\\n")
    .replace("\r", "\\r")
    .replace("\x1a", "\\Z")

# URL encoding
proc encodeUrl(s: string): string =
  result = ""
  for c in s:
    if c.isAlpha() or c.isDigit() or c in "-._~":
      result.add(c)
    else:
      result.add('%')
      result.add(toHex(ord(c), 2))

proc decodeUrl(s: string): string =
  result = ""
  var i = 0
  while i < s.len:
    if s[i] == '%' and i + 2 < s.len:
      let hex = s[i+1..i+2]
      result.add(chr(fromHex[int](hex)))
      i += 3
    elif s[i] == '+':
      result.add(' ')
      inc i
    else:
      result.add(s[i])
      inc i

echo encodeUrl("hello world & nim = awesome!")
# hello%20world%20%26%20nim%20%3D%20awesome%21

echo decodeUrl("hello%20world%20%26%20nim%20%3D%20awesome%21")
# hello world & nim = awesome!

# Input sanitization
proc sanitizeInput(input: string, maxLen: int = 255): string =
  result = input
    .strip()                    # Remove leading/trailing whitespace
    .replace("\x00", "")        # Remove null bytes
    .replace("\r\n", "\n")      # Normalize line endings
    .replace("\r", "\n")
  
  if result.len > maxLen:
    result = result[0..<maxLen]

# Mask sensitive data
proc maskCreditCard(card: string): string =
  let digits = card.filterIt(it.isDigit())
  if digits.len < 4: return "****"
  return "*".repeat(digits.len - 4) & digits[^4..^1]

proc maskPhone(phone: string): string =
  let digits = phone.filterIt(it.isDigit())
  if digits.len < 4: return "****"
  return digits[0..2] & "-****-" & digits[^4..^1]

echo maskCreditCard("4111 1111 1111 1234")  # ************1234
echo maskPhone("+66-81-234-5678")           # 668-****-5678
```

---

## Step 79: String Utilities สำหรับ API

```nim
import std/strutils, std/strformat, std/sequtils, std/tables, std/re

# Query string parser
proc parseQueryString(qs: string): Table[string, string] =
  result = initTable[string, string]()
  let parts = qs.strip(chars = {'?'}).split('&')
  
  for part in parts:
    let kv = part.split('=', maxsplit = 1)
    if kv.len == 2:
      result[kv[0]] = decodeUrl(kv[1])
    elif kv.len == 1 and kv[0].len > 0:
      result[kv[0]] = ""

proc buildQueryString(params: Table[string, string]): string =
  var parts: seq[string] = @[]
  for k, v in params:
    parts.add(encodeUrl(k) & "=" & encodeUrl(v))
  return parts.join("&")

let qs = "name=Alice+Smith&age=28&city=Bangkok&debug"
let params = parseQueryString(qs)
echo params  # {"name": "Alice Smith", "age": "28", ...}
echo buildQueryString(params)

# Path pattern matching
proc matchPath(pattern, path: string): Option[Table[string, string]] =
  let patternParts = pattern.strip(chars = {'/'}).split('/')
  let pathParts = path.strip(chars = {'/'}).split('/')
  
  if patternParts.len != pathParts.len:
    return none(Table[string, string])
  
  var params = initTable[string, string]()
  
  for i in 0..<patternParts.len:
    if patternParts[i].startsWith(':'):
      # Dynamic segment
      params[patternParts[i][1..^1]] = pathParts[i]
    elif patternParts[i] != pathParts[i]:
      return none(Table[string, string])
  
  return some(params)

let routes = [
  "/users/:id",
  "/users/:id/posts/:postId",
  "/products/:category/:slug",
]

let testPaths = [
  "/users/123",
  "/users/456/posts/789",
  "/products/electronics/laptop-pro",
  "/unknown/path",
]

for path in testPaths:
  var matched = false
  for route in routes:
    let result = matchPath(route, path)
    if result.isSome():
      echo fmt"✅ {path}"
      echo fmt"   Route: {route}"
      echo fmt"   Params: {result.get()}"
      matched = true
      break
  if not matched:
    echo fmt"❌ {path} - no match"

# JWT token parsing (simplified, no verification)
proc parseJwtPayload(token: string): Table[string, string] =
  result = initTable[string, string]()
  let parts = token.split('.')
  if parts.len != 3:
    return
  
  # Decode base64url (simplified)
  var b64 = parts[1]
    .replace("-", "+")
    .replace("_", "/")
  
  # Pad if necessary
  while b64.len mod 4 != 0:
    b64.add('=')
  
  # Would need proper base64 decode and JSON parse here
  # This is simplified for illustration
  echo "JWT parts: header, payload, signature"
  echo "Payload (base64): " & b64
```

---

## Step 80: Advanced String Patterns

```nim
import std/strutils, std/sequtils, std/re, std/tables

# Levenshtein distance (edit distance)
proc levenshtein(a, b: string): int =
  let m = a.len
  let n = b.len
  var dp = newSeqWith(m + 1, newSeq[int](n + 1))
  
  for i in 0..m: dp[i][0] = i
  for j in 0..n: dp[0][j] = j
  
  for i in 1..m:
    for j in 1..n:
      if a[i-1] == b[j-1]:
        dp[i][j] = dp[i-1][j-1]
      else:
        dp[i][j] = 1 + min(min(dp[i-1][j], dp[i][j-1]), dp[i-1][j-1])
  
  return dp[m][n]

echo levenshtein("kitten", "sitting")  # 3
echo levenshtein("nim", "nim")         # 0
echo levenshtein("hello", "world")     # 4

# Fuzzy search
proc fuzzyMatch(pattern, text: string, threshold: float = 0.6): bool =
  if pattern.len == 0: return true
  if text.len == 0: return false
  
  let dist = levenshtein(pattern.toLower(), text.toLower())
  let similarity = 1.0 - float(dist) / float(max(pattern.len, text.len))
  return similarity >= threshold

proc fuzzySearch(query: string, items: seq[string]): seq[string] =
  result = @[]
  for item in items:
    if fuzzyMatch(query, item):
      result.add(item)

let products2 = @["Laptop Pro", "Gaming Laptop", "USB Hub", "Mouse", "Keyboard", "Monitor"]
echo fuzzySearch("laptp", products2)  # ["Laptop Pro", "Gaming Laptop"]

# Text statistics
proc textStats(text: string): Table[string, int] =
  result = initTable[string, int]()
  
  result["chars"] = text.len
  result["lines"] = text.count("\n") + 1
  
  let words = text.splitWhitespace()
  result["words"] = words.len
  
  result["sentences"] = text.count('.') + text.count('!') + text.count('?')
  
  var uniqueWords = initHashSet[string]()
  for w in words:
    uniqueWords.incl(w.toLower().strip(chars = {'.',',','!','?','"','\''}))
  result["unique_words"] = uniqueWords.len
  
  return result

let article = """
Nim is a statically typed compiled programming language.
It combines successful concepts from mature languages like Python, Ada and Modula.
Nim is powerful, expressive, and efficient!
"""

let stats = textStats(article)
for key, val in stats:
  echo fmt"  {key}: {val}"

# Word frequency analysis
proc wordFrequency(text: string): seq[(string, int)] =
  var freq = initTable[string, int]()
  
  for word in text.splitWhitespace():
    let clean = word.toLower().strip(chars = {'.',',','!','?','"','\''})
    if clean.len > 2:  # ignore short words
      freq[clean] = freq.getOrDefault(clean, 0) + 1
  
  result = freq.pairs.toSeq()
  result.sort(proc(a, b: (string, int)): int = b[1] - a[1])

echo "\nTop 5 words:"
for i, (word, count) in wordFrequency(article):
  if i >= 5: break
  echo fmt"  {i+1}. {word}: {count}"
```

---

## Step 81-85: Real-World App - Content Processor

```nim
# content_processor.nim
# A text content processing system for a blog/CMS

import std/strutils, std/strformat, std/re, std/tables, std/sequtils

type
  HeadingLevel = 1..6
  
  ContentBlock = object
    case kind: string
    of "heading":
      level: HeadingLevel
      text: string
    of "paragraph":
      content: string
    of "code":
      language: string
      code: string
    of "list":
      items: seq[string]
      ordered: bool
    else:
      raw: string

proc parseMarkdownSimple(md: string): seq[ContentBlock] =
  result = @[]
  var lines = md.split("\n")
  var i = 0
  
  while i < lines.len:
    let line = lines[i]
    
    # Headings
    if line.startsWith("#"):
      var level = 0
      var j = 0
      while j < line.len and line[j] == '#':
        inc level
        inc j
      let text = line[level..^1].strip()
      result.add(ContentBlock(kind: "heading", level: HeadingLevel(min(level, 6)), text: text))
    
    # Code blocks
    elif line.startsWith("```"):
      let lang = line[3..^1].strip()
      var code = ""
      inc i
      while i < lines.len and not lines[i].startsWith("```"):
        code &= lines[i] & "\n"
        inc i
      result.add(ContentBlock(kind: "code", language: lang, code: code.strip()))
    
    # Lists
    elif line.startsWith("- ") or line.startsWith("* "):
      var items: seq[string] = @[]
      while i < lines.len and (lines[i].startsWith("- ") or lines[i].startsWith("* ")):
        items.add(lines[i][2..^1].strip())
        inc i
      dec i  # back up one
      result.add(ContentBlock(kind: "list", items: items, ordered: false))
    
    # Paragraphs
    elif line.strip().len > 0:
      var para = line
      while i + 1 < lines.len and lines[i+1].strip().len > 0 and
            not lines[i+1].startsWith("#") and
            not lines[i+1].startsWith("```") and
            not lines[i+1].startsWith("- "):
        inc i
        para &= " " & lines[i]
      result.add(ContentBlock(kind: "paragraph", content: para.strip()))
    
    inc i

proc renderHtml(blocks: seq[ContentBlock]): string =
  var sb = newStringBuilder()
  
  for blk in blocks:
    case blk.kind
    of "heading":
      let h = "h" & $blk.level
      sb.addLine(fmt"<{h}>{escapeHtml(blk.text)}</{h}>")
    of "paragraph":
      # Process inline markdown
      var text = blk.content
      text = text.replace(re"\*\*([^*]+)\*\*", "<strong>$1</strong>")  # bold
      text = text.replace(re"\*([^*]+)\*", "<em>$1</em>")              # italic
      text = text.replace(re"`([^`]+)`", "<code>$1</code>")            # inline code
      sb.addLine(fmt"<p>{text}</p>")
    of "code":
      sb.addLine(fmt"""<pre><code class="language-{blk.language}">""")
      sb.addLine(escapeHtml(blk.code))
      sb.addLine("</code></pre>")
    of "list":
      let tag = if blk.ordered: "ol" else: "ul"
      sb.addLine(fmt"<{tag}>")
      for item in blk.items:
        sb.addLine(fmt"  <li>{escapeHtml(item)}</li>")
      sb.addLine(fmt"</{tag}>")
    else:
      discard
  
  return sb.toString()

proc generateTableOfContents(blocks: seq[ContentBlock]): string =
  var toc: seq[(int, string)] = @[]
  
  for blk in blocks:
    if blk.kind == "heading":
      toc.add((blk.level, blk.text))
  
  if toc.len == 0: return ""
  
  var sb = newStringBuilder()
  sb.addLine("<nav class=\"toc\">")
  sb.addLine("  <h2>สารบัญ</h2>")
  sb.addLine("  <ul>")
  
  for (level, text) in toc:
    let indent = "  ".repeat(level)
    let anchor = text.toLower().replace(" ", "-")
    sb.addLine(fmt"{indent}  <li><a href=\"#{anchor}\">{text}</a></li>")
  
  sb.addLine("  </ul>")
  sb.addLine("</nav>")
  
  return sb.toString()

# Demo
proc main() =
  let article = """
# Introduction to Nim

Nim is a **statically typed** compiled programming language.

## Why Nim?

Nim combines successful concepts:
- Fast like C
- Readable like Python
- Powerful like Lisp

## Getting Started

Install with:

```bash
curl https://nim-lang.org/choosenim/init.sh -sSf | sh
```

Write your first program:

```nim
echo "Hello, World!"
```

## Conclusion

Nim is an *excellent* choice for backend development!
"""

  echo "=== Parsing Markdown ==="
  let blocks = parseMarkdownSimple(article)
  echo fmt"Found {blocks.len} content blocks"
  
  echo "\n=== Table of Contents ==="
  echo generateTableOfContents(blocks)
  
  echo "\n=== HTML Output ==="
  echo renderHtml(blocks)

main()
```

---

## 📝 สรุป Part 07

| Steps | หัวข้อ |
|-------|--------|
| 71 | String basics, methods |
| 72 | String formatting (fmt) |
| 73 | Regular expressions |
| 74 | String parsing |
| 75 | String builder pattern |
| 76 | Unicode, Thai text |
| 77 | Template engine |
| 78 | String security (XSS, SQL escape) |
| 79 | API string utilities |
| 80 | Advanced: edit distance, fuzzy search |
| 81-85 | Real-world: Content Processor |

---

**← [Part 06: Sequences](part_06_sequences.md) | [Part 08: File I/O →](part_08_file_io.md)**
