# Part 26: File Uploads & Storage
## Steps 361-375: จัดการไฟล์อัปโหลดแบบ Production

---

## 🎯 เป้าหมายของ Part นี้

- Multipart form data parsing
- File validation (type, size)
- Local file storage
- Cloud storage (S3-compatible)
- Image processing (resize, convert)
- File download/streaming
- CDN integration pattern

---

## Step 361: Multipart Form Data Parsing

```nim
import asyncdispatch, asynchttpserver, strutils, strformat, os

# HTTP multipart/form-data format:
# --boundary
# Content-Disposition: form-data; name="field1"
#
# value1
# --boundary
# Content-Disposition: form-data; name="file"; filename="photo.jpg"
# Content-Type: image/jpeg
#
# <binary data>
# --boundary--

type
  UploadedFile = object
    fieldName: string
    filename: string
    contentType: string
    data: seq[byte]
    size: int

  FormPart = object
    name: string
    filename: string
    contentType: string
    value: string
    isFile: bool

proc extractBoundary(contentType: string): string =
  # Content-Type: multipart/form-data; boundary=----WebKitFormBoundary7MA4YWxkTrZu0gW
  let parts = contentType.split(';')
  for part in parts:
    let trimmed = part.strip()
    if trimmed.startsWith("boundary="):
      return trimmed[9..^1].strip(chars = {'"', '\''})
  return ""

proc parseMultipart(body: string, boundary: string): seq[FormPart] =
  result = @[]
  let delimiter = "--" & boundary
  let terminator = "--" & boundary & "--"
  
  var parts = body.split(delimiter)
  
  for part in parts:
    if part == "" or part == "--\r\n" or part == "--":
      continue
    
    let stripped = part.strip()
    if stripped == "--":
      continue
    
    # Split headers and body
    let headerBodySep = stripped.find("\r\n\r\n")
    if headerBodySep < 0:
      continue
    
    let headers = stripped[0 ..< headerBodySep]
    var value = stripped[headerBodySep + 4 .. ^1]
    
    # Remove trailing \r\n
    if value.endsWith("\r\n"):
      value = value[0 ..< ^2]
    
    var formPart = FormPart()
    
    for header in headers.split("\r\n"):
      if header.toLowerAscii().startsWith("content-disposition:"):
        let disp = header[20..^1].strip()
        
        for field in disp.split(';'):
          let f = field.strip()
          if f.startsWith("name="):
            formPart.name = f[5..^1].strip(chars = {'"'})
          elif f.startsWith("filename="):
            formPart.filename = f[9..^1].strip(chars = {'"'})
            formPart.isFile = true
      
      elif header.toLowerAscii().startsWith("content-type:"):
        formPart.contentType = header[13..^1].strip()
    
    formPart.value = value
    result.add(formPart)

# Test the parser
let testBody = "--boundary123\r\nContent-Disposition: form-data; name=\"title\"\r\n\r\nMy Photo\r\n--boundary123\r\nContent-Disposition: form-data; name=\"file\"; filename=\"photo.jpg\"\r\nContent-Type: image/jpeg\r\n\r\n<binary data here>\r\n--boundary123--"

let parts = parseMultipart(testBody, "boundary123")
for part in parts:
  if part.isFile:
    echo fmt"File: {part.filename} ({part.contentType})"
  else:
    echo fmt"Field: {part.name} = {part.value}"
```

---

## Step 362: File Validation

```nim
import strutils, strformat, os, sequtils

type
  FileType = enum
    Image, Document, Video, Audio, Archive, Unknown

  ValidationError = object of CatchableError

  FileValidationRule = object
    maxSizeBytes: int
    allowedTypes: seq[string]   # MIME types
    allowedExtensions: seq[string]

  FileInfo = object
    filename: string
    contentType: string
    size: int
    extension: string
    detectedType: FileType

# MIME type detection by magic bytes
proc detectMimeType(data: seq[byte]): string =
  if data.len < 4:
    return "application/octet-stream"
  
  # JPEG: FF D8 FF
  if data[0] == 0xFF and data[1] == 0xD8 and data[2] == 0xFF:
    return "image/jpeg"
  
  # PNG: 89 50 4E 47
  if data[0] == 0x89 and data[1] == 0x50 and data[2] == 0x4E and data[3] == 0x47:
    return "image/png"
  
  # GIF: 47 49 46 38
  if data[0] == 0x47 and data[1] == 0x49 and data[2] == 0x46:
    return "image/gif"
  
  # PDF: 25 50 44 46
  if data[0] == 0x25 and data[1] == 0x50 and data[2] == 0x44 and data[3] == 0x46:
    return "application/pdf"
  
  # ZIP: 50 4B 03 04
  if data[0] == 0x50 and data[1] == 0x4B and data[2] == 0x03 and data[3] == 0x04:
    return "application/zip"
  
  # WebP: RIFF...WEBP
  if data.len >= 12 and data[0] == 0x52 and data[1] == 0x49 and 
     data[8] == 0x57 and data[9] == 0x45:
    return "image/webp"
  
  return "application/octet-stream"

proc getFileType(mimeType: string): FileType =
  if mimeType.startsWith("image/"):        return Image
  if mimeType.startsWith("video/"):        return Video
  if mimeType.startsWith("audio/"):        return Audio
  if mimeType == "application/pdf":        return Document
  if mimeType.contains("zip") or 
     mimeType.contains("tar") or
     mimeType.contains("rar"):             return Archive
  return Unknown

proc validateFile(filename: string, contentType: string, 
                  size: int, rule: FileValidationRule): string =
  # Check size
  if size > rule.maxSizeBytes:
    let maxMb = rule.maxSizeBytes div (1024 * 1024)
    let sizeMb = size div (1024 * 1024)
    return fmt"File too large: {sizeMb}MB (max {maxMb}MB)"
  
  # Check extension
  let ext = filename.splitFile().ext.toLowerAscii().strip(chars = {'.'})
  if rule.allowedExtensions.len > 0 and ext notin rule.allowedExtensions:
    return fmt"Extension .{ext} not allowed. Allowed: {rule.allowedExtensions.join(\", \")}"
  
  # Check content type
  if rule.allowedTypes.len > 0 and contentType notin rule.allowedTypes:
    return fmt"Content type '{contentType}' not allowed"
  
  return ""  # No error

# Predefined rules
let imageRule = FileValidationRule(
  maxSizeBytes: 5 * 1024 * 1024,  # 5MB
  allowedTypes: @["image/jpeg", "image/png", "image/gif", "image/webp"],
  allowedExtensions: @["jpg", "jpeg", "png", "gif", "webp"]
)

let documentRule = FileValidationRule(
  maxSizeBytes: 20 * 1024 * 1024,  # 20MB
  allowedTypes: @["application/pdf", "application/msword"],
  allowedExtensions: @["pdf", "doc", "docx"]
)

# Test
echo validateFile("photo.jpg", "image/jpeg", 2_000_000, imageRule)  # "" (valid)
echo validateFile("photo.jpg", "image/jpeg", 10_000_000, imageRule)  # too large
echo validateFile("virus.exe", "application/x-msdownload", 100, imageRule)  # bad ext
```

---

## Step 363: Secure File Storage

```nim
import os, strformat, times, random, strutils, hashes, sequtils

randomize()

type
  StorageConfig = object
    basePath: string
    urlPrefix: string
    maxDirFiles: int  # files per subdirectory

  StoredFile = object
    storageKey: string  # unique path on disk
    publicUrl: string
    originalName: string
    contentType: string
    size: int
    uploadedAt: float

proc newStorageConfig(basePath: string, urlPrefix: string): StorageConfig =
  StorageConfig(
    basePath: basePath,
    urlPrefix: urlPrefix,
    maxDirFiles: 1000
  )

proc generateStorageKey(originalFilename: string): string =
  let ext = originalFilename.splitFile().ext.toLowerAscii()
  let timestamp = toUnix(getTime()).int
  let random = rand(999999)
  
  # Use date-based directory: 2025/01/15/
  let dt = now()
  let dateDir = fmt"{dt.year:04d}/{dt.month.ord:02d}/{dt.monthday:02d}"
  
  return fmt"{dateDir}/{timestamp}_{random:06d}{ext}"

proc sanitizeFilename(filename: string): string =
  # Remove path traversal and dangerous chars
  let name = filename.extractFilename()
  
  result = ""
  for c in name:
    if c in {'a'..'z', 'A'..'Z', '0'..'9', '-', '_', '.'}:
      result.add(c)
    else:
      result.add('_')
  
  # Limit length
  if result.len > 100:
    result = result[0..99]

proc storeFile(config: StorageConfig, originalName: string,
               contentType: string, data: seq[byte]): StoredFile =
  let safeFilename = sanitizeFilename(originalName)
  let key = generateStorageKey(safeFilename)
  let fullPath = config.basePath / key
  
  # Create directory
  createDir(fullPath.parentDir())
  
  # Write file
  var f = open(fullPath, fmWrite)
  for b in data:
    f.write(b)
  f.close()
  
  return StoredFile(
    storageKey: key,
    publicUrl: config.urlPrefix & "/" & key,
    originalName: originalName,
    contentType: contentType,
    size: data.len,
    uploadedAt: epochTime()
  )

proc deleteFile(config: StorageConfig, storageKey: string) =
  let fullPath = config.basePath / storageKey
  if fileExists(fullPath):
    removeFile(fullPath)

# Test
let storage = newStorageConfig("/tmp/uploads", "https://files.example.com")

echo "File storage configured:"
echo fmt"  Base path: {storage.basePath}"
echo fmt"  URL prefix: {storage.urlPrefix}"

echo "\nFilename sanitization:"
echo sanitizeFilename("my photo (1).jpg")          # safe
echo sanitizeFilename("../../../etc/passwd")        # path traversal removed
echo sanitizeFilename("file; rm -rf /; echo.txt")  # injection removed

echo "\nStorage key example:"
echo generateStorageKey("photo.jpg")
```

---

## Step 364-375: Complete File Upload Server

```nim
# file_upload_server.nim - Production file upload service

import asyncdispatch, asynchttpserver, json, os, strutils, 
       strformat, times, tables, options, sequtils

# ============================
# Configuration
# ============================

const
  MAX_IMAGE_SIZE = 5 * 1024 * 1024    # 5MB
  MAX_DOC_SIZE   = 20 * 1024 * 1024   # 20MB
  UPLOAD_PATH    = "/tmp/test_uploads"
  BASE_URL       = "http://localhost:8080"

# ============================
# File Record (in-memory for demo)
# ============================

type
  FileRecord = object
    id: string
    storageKey: string
    originalName: string
    contentType: string
    sizeBytes: int
    uploadedBy: int
    publicUrl: string
    createdAt: string
    tags: seq[string]

var fileRecords: Table[string, FileRecord] = initTable[string, FileRecord]()

# ============================
# Upload Processing
# ============================

proc generateFileId(): string =
  let ts = toUnix(getTime())
  let rnd = rand(high(int))
  return fmt"{ts:x}{rnd:x}"

proc processUpload(
  filename: string,
  contentType: string,
  data: string,
  userId: int,
  tags: seq[string] = @[]
): FileRecord =
  # Create storage directory
  createDir(UPLOAD_PATH)
  
  # Generate unique path
  let dt = now()
  let dateDir = fmt"{dt.year:04d}/{dt.month.ord:02d}"
  let storageDir = UPLOAD_PATH / dateDir
  createDir(storageDir)
  
  let fileId = generateFileId()
  let ext = filename.splitFile().ext.toLowerAscii()
  let storageName = fileId & ext
  let storagePath = storageDir / storageName
  let storageKey = dateDir & "/" & storageName
  
  # Write file
  writeFile(storagePath, data)
  
  let record = FileRecord(
    id: fileId,
    storageKey: storageKey,
    originalName: filename,
    contentType: contentType,
    sizeBytes: data.len,
    uploadedBy: userId,
    publicUrl: BASE_URL & "/files/" & storageKey,
    createdAt: $now(),
    tags: tags
  )
  
  fileRecords[fileId] = record
  return record

# ============================
# HTTP Handler
# ============================

proc handleUpload(req: Request) {.async.} =
  let path = req.url.path
  let meth = req.reqMethod
  
  # POST /upload - Single file upload (JSON-based for simplicity)
  if meth == HttpPost and path == "/upload":
    try:
      let body = parseJson(req.body)
      
      let filename = body["filename"].getStr()
      let contentType = body["contentType"].getStr()
      let fileData = body["data"].getStr()  # base64 in real app
      let userId = body.getOrDefault("userId").getInt(1)
      
      var tags: seq[string] = @[]
      if "tags" in body:
        for tag in body["tags"]:
          tags.add(tag.getStr())
      
      # Validate
      if filename.len == 0:
        await req.respond(Http400, """{"error":"filename required"}""",
          newHttpHeaders([("Content-Type", "application/json")]))
        return
      
      if fileData.len > MAX_IMAGE_SIZE:
        await req.respond(Http413, """{"error":"File too large"}""",
          newHttpHeaders([("Content-Type", "application/json")]))
        return
      
      let record = processUpload(filename, contentType, fileData, userId, tags)
      
      let response = %*{
        "id": record.id,
        "url": record.publicUrl,
        "filename": record.originalName,
        "size": record.sizeBytes,
        "contentType": record.contentType,
        "tags": record.tags,
        "createdAt": record.createdAt
      }
      
      await req.respond(Http201, $response,
        newHttpHeaders([("Content-Type", "application/json")]))
    
    except JsonParsingError:
      await req.respond(Http400, """{"error":"Invalid JSON body"}""",
        newHttpHeaders([("Content-Type", "application/json")]))
    return
  
  # GET /files/:id - File info
  if meth == HttpGet and path.startsWith("/files/"):
    let fileId = path[7..^1].split('/')[0]
    
    if fileId in fileRecords:
      let record = fileRecords[fileId]
      let response = %*{
        "id": record.id,
        "url": record.publicUrl,
        "filename": record.originalName,
        "size": record.sizeBytes,
        "contentType": record.contentType,
        "tags": record.tags,
        "createdAt": record.createdAt
      }
      await req.respond(Http200, $response,
        newHttpHeaders([("Content-Type", "application/json")]))
    else:
      await req.respond(Http404, """{"error":"File not found"}""",
        newHttpHeaders([("Content-Type", "application/json")]))
    return
  
  # DELETE /files/:id
  if meth == HttpDelete and path.startsWith("/files/"):
    let fileId = path[7..^1]
    
    if fileId in fileRecords:
      let record = fileRecords[fileId]
      let fullPath = UPLOAD_PATH / record.storageKey
      
      if fileExists(fullPath):
        removeFile(fullPath)
      
      fileRecords.del(fileId)
      
      await req.respond(Http200, """{"message":"File deleted"}""",
        newHttpHeaders([("Content-Type", "application/json")]))
    else:
      await req.respond(Http404, """{"error":"File not found"}""",
        newHttpHeaders([("Content-Type", "application/json")]))
    return
  
  # GET /files - List files
  if meth == HttpGet and path == "/files":
    var files = newJArray()
    for _, record in fileRecords:
      files.add(%*{
        "id": record.id,
        "filename": record.originalName,
        "url": record.publicUrl,
        "size": record.sizeBytes,
        "contentType": record.contentType
      })
    await req.respond(Http200, $(%*{"files": files, "total": files.len}),
      newHttpHeaders([("Content-Type", "application/json")]))
    return
  
  # Health
  if path == "/health":
    await req.respond(Http200, """{"status":"ok","service":"file-upload"}""",
      newHttpHeaders([("Content-Type", "application/json")]))
    return
  
  await req.respond(Http404, """{"error":"Not found"}""",
    newHttpHeaders([("Content-Type", "application/json")]))

# ============================
# Test (without running server)
# ============================

proc demoUploads() =
  echo "=== File Upload Demo ==="
  
  # Simulate uploads
  let r1 = processUpload("photo.jpg", "image/jpeg", "fake-jpeg-data", 1, @["profile", "photo"])
  echo fmt"Uploaded: {r1.id} -> {r1.publicUrl}"
  
  let r2 = processUpload("document.pdf", "application/pdf", "fake-pdf-data", 1, @["document"])
  echo fmt"Uploaded: {r2.id} -> {r2.publicUrl}"
  
  let r3 = processUpload("report.pdf", "application/pdf", "fake-pdf-data-2", 2, @["report"])
  echo fmt"Uploaded: {r3.id} -> {r3.publicUrl}"
  
  echo fmt"\nTotal files: {fileRecords.len}"
  
  # List by tag
  echo "\nFiles with tag 'document':"
  for _, rec in fileRecords:
    if "document" in rec.tags:
      echo fmt"  - {rec.originalName} ({rec.sizeBytes} bytes)"
  
  # Cleanup
  for _, rec in fileRecords:
    let fullPath = UPLOAD_PATH / rec.storageKey
    if fileExists(fullPath):
      removeFile(fullPath)
  
  echo "\nCleanup done"

demoUploads()

# To run the server:
# proc main() {.async.} =
#   let server = newAsyncHttpServer()
#   echo "File upload service on port 8080"
#   await server.serve(Port(8080), handleUpload)
# waitFor main()
```

---

## 📝 สรุป Part 26

| Steps | หัวข้อ |
|-------|--------|
| 361 | Multipart form data parsing |
| 362 | File validation (type, size, MIME) |
| 363 | Secure file storage with sanitization |
| 364-375 | Complete file upload server (CRUD) |

---

**← [Part 25: Microservices](part_25_microservices.md) | [Part 27: Background Jobs →](part_27_background_jobs.md)**
