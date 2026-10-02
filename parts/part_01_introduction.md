# Part 01: เริ่มต้น Nim - Installation & Hello World
## Steps 1-10: รู้จัก Nim และการติดตั้ง

---

## 🎯 เป้าหมายของ Part นี้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
- เข้าใจว่า Nim คืออะไรและทำไมต้องใช้
- ติดตั้ง Nim บน Linux, macOS, Windows
- เขียนโปรแกรม Hello World แรกของคุณ
- เข้าใจ Nim Compiler และ Build System
- รู้จัก Nimble (Package Manager ของ Nim)

---

## Step 1: Nim คืออะไร?

Nim เป็นภาษาโปรแกรมมิ่งที่ออกแบบมาเพื่อ:
- **ประสิทธิภาพสูง** เหมือน C/C++
- **Productivity สูง** เหมือน Python
- **Expressiveness** เหมือน Lisp

### ประวัติย่อของ Nim

```
2005 - Andreas Rumpf เริ่มพัฒนา (ชื่อเดิม: Nimrod)
2008 - เปิด Source Code ครั้งแรก
2014 - เปลี่ยนชื่อเป็น Nim
2019 - Nim 1.0.0 ออกมา (Stable)
2021 - Nim 1.6.x
2023 - Nim 2.0 (ปรับปรุงครั้งใหญ่)
2024 - Nim 2.2.x (ปัจจุบัน)
```

### Nim ถูกใช้ที่ไหน?

- **Backend APIs** - REST, GraphQL, gRPC
- **System Programming** - OS tools, Compilers
- **Game Development** - 2D/3D games
- **Embedded Systems** - IoT devices
- **WebAssembly** - Browser applications
- **Scientific Computing** - Data analysis

---

## Step 2: ทำไมต้องเลือก Nim สำหรับ Backend?

### เปรียบเทียบกับภาษาอื่น

| Feature | Nim | Go | Python | Node.js | Rust |
|---------|-----|-----|--------|---------|------|
| Speed | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| Ease of Learning | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐ |
| Memory Safety | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| Ecosystem | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| Syntax Beauty | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ |

### Benchmark: Nim vs ภาษาอื่น (Requests/Second)

```
httpbeast (Nim)    ~  500,000 req/s  ███████████████████████████████
Actix-web (Rust)   ~  450,000 req/s  ████████████████████████████
Fasthttp (Go)      ~  380,000 req/s  ████████████████████████
Express (Node.js)  ~   90,000 req/s  ██████
FastAPI (Python)   ~   50,000 req/s  ███
```

### ข้อดีของ Nim สำหรับ Backend

```nim
# 1. Syntax สวยงาม อ่านง่าย
proc greetUser(name: string): string =
  return "สวัสดี " & name & "!"

# 2. Type Safety แต่ไม่ verbose
var users: seq[string] = @["Alice", "Bob", "Charlie"]

# 3. Async/Await ในตัว
import asynchttpserver, asyncdispatch

proc handler(req: Request): Future[void] {.async.} =
  await req.respond(Http200, "Hello World!")

# 4. Compile ไปเป็น Native Code
# nim c -d:release myapp.nim  --> ได้ binary ที่เร็วมาก
```

---

## Step 3: ติดตั้ง Nim บน Linux (Ubuntu/Debian)

### วิธีที่ 1: ผ่าน choosenim (แนะนำ)

choosenim เป็น Version Manager ของ Nim คล้ายกับ nvm ของ Node.js

```bash
# 1. Download และ run choosenim installer
curl https://nim-lang.org/choosenim/init.sh -sSf | sh

# 2. เพิ่ม Nim ใน PATH (เพิ่มใน ~/.bashrc หรือ ~/.zshrc)
export PATH="$HOME/.nimble/bin:$PATH"

# 3. Reload shell
source ~/.bashrc

# 4. ตรวจสอบการติดตั้ง
nim --version
# Nim Compiler Version 2.2.x [Linux: amd64]
```

### วิธีที่ 2: ผ่าน Package Manager

```bash
# Ubuntu/Debian
sudo apt-get update
sudo apt-get install nim

# Fedora/RHEL
sudo dnf install nim

# Arch Linux
sudo pacman -S nim

# ตรวจสอบ version
nim --version
```

### วิธีที่ 3: Build จาก Source

```bash
# Clone Nim repository
git clone https://github.com/nim-lang/Nim.git
cd Nim

# Build
sh build_all.sh

# เพิ่ม PATH
export PATH="$PWD/bin:$PATH"
```

---

## Step 4: ติดตั้ง Nim บน macOS

### วิธีที่ 1: ผ่าน choosenim (แนะนำ)

```bash
# 1. ติดตั้ง choosenim
curl https://nim-lang.org/choosenim/init.sh -sSf | sh

# 2. เพิ่ม PATH ใน ~/.zshrc หรือ ~/.bash_profile
echo 'export PATH="$HOME/.nimble/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc

# 3. ตรวจสอบ
nim --version
```

### วิธีที่ 2: ผ่าน Homebrew

```bash
# ติดตั้ง Homebrew ก่อน (ถ้ายังไม่มี)
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# ติดตั้ง Nim
brew install nim

# ตรวจสอบ
nim --version
```

---

## Step 5: ติดตั้ง Nim บน Windows

### วิธีที่ 1: ผ่าน choosenim

```powershell
# 1. เปิด PowerShell ในฐานะ Administrator
# 2. ดาวน์โหลด choosenim
Invoke-WebRequest -Uri "https://nim-lang.org/choosenim/init.sh" -OutFile "choosenim-init.sh"

# หรือ download .exe โดยตรง
# ไปที่ https://nim-lang.org/install_windows.html
```

### วิธีที่ 2: Windows Installer

1. ไปที่ https://nim-lang.org/install_windows.html
2. Download `nim-x.x.x_x64.zip`
3. Extract ไปยัง `C:\Nim`
4. เพิ่ม `C:\Nim\bin` ใน PATH

```powershell
# ตรวจสอบการติดตั้ง
nim --version
nimble --version
```

### วิธีที่ 3: WSL2 (แนะนำสำหรับ Development)

```bash
# ติดตั้ง WSL2
wsl --install

# ใน WSL2 ทำตาม Linux steps
curl https://nim-lang.org/choosenim/init.sh -sSf | sh
```

---

## Step 6: ทำความรู้จัก Nim Compiler

### คำสั่งพื้นฐาน

```bash
# Compile และ Run
nim c myfile.nim        # compile เป็น binary
nim c -r myfile.nim     # compile และ run ทันที

# Build modes
nim c -d:debug myfile.nim    # Debug mode (default)
nim c -d:release myfile.nim  # Release mode (optimized)
nim c -d:danger myfile.nim   # Danger mode (fastest, no checks)

# Check syntax (ไม่ compile)
nim check myfile.nim

# Generate documentation
nim doc myfile.nim

# Compile to JavaScript
nim js myfile.nim

# Compile to C
nim compileToC myfile.nim
```

### Understanding Compilation Process

```
Source Code (.nim)
      │
      ▼
  Nim Compiler
      │
      ▼
  C/C++ Code     ← Nim generates C code first!
      │
      ▼
  C Compiler (gcc/clang/MSVC)
      │
      ▼
  Native Binary  ← Fast executable!
```

### Compiler Flags ที่สำคัญ

```bash
# Optimization flags
nim c -d:release -d:lto myfile.nim          # Link Time Optimization
nim c -d:release --opt:speed myfile.nim     # Optimize for speed
nim c -d:release --opt:size myfile.nim      # Optimize for size

# Debug flags
nim c --debugger:native myfile.nim          # Native debugger support
nim c --lineDir:on myfile.nim               # Include line directives

# Output
nim c -o:myapp myfile.nim                   # Custom output name

# Define constants
nim c -d:myConst=42 myfile.nim

# Show assembly
nim c --asm myfile.nim
```

---

## Step 7: Hello World! โปรแกรมแรก

### สร้างไฟล์แรก

```bash
# สร้าง directory สำหรับ projects
mkdir -p ~/nim_projects/hello_world
cd ~/nim_projects/hello_world

# สร้างไฟล์
touch hello.nim
```

### Hello World ง่ายๆ

```nim
# hello.nim
echo "สวัสดี, โลก!"
echo "Hello, World!"
echo "Bonjour, Monde!"
```

```bash
# Compile และ Run
nim c -r hello.nim

# Output:
# สวัสดี, โลก!
# Hello, World!
# Bonjour, Monde!
```

### Hello World แบบใช้ Procedure

```nim
# hello_proc.nim

proc sayHello(name: string) =
  echo "สวัสดี, " & name & "!"
  echo "Hello, " & name & "!"

proc main() =
  sayHello("Nim Learner")
  sayHello("World")
  
  let version = NimVersion
  echo "คุณกำลังใช้ Nim เวอร์ชัน: " & version

main()
```

```bash
nim c -r hello_proc.nim
# สวัสดี, Nim Learner!
# Hello, Nim Learner!
# สวัสดี, World!
# Hello, World!
# คุณกำลังใช้ Nim เวอร์ชัน: 2.2.0
```

### Hello World แบบ Interactive

```nim
# hello_interactive.nim

import std/strutils

proc main() =
  echo "กรุณาป้อนชื่อของคุณ:"
  let name = readLine(stdin).strip()
  
  if name.len == 0:
    echo "สวัสดี, ผู้ไม่ระบุตัวตน!"
  else:
    echo "สวัสดี, " & name & "!"
    echo "ยินดีต้อนรับสู่โลกของ Nim!"

main()
```

---

## Step 8: รู้จัก Nimble (Package Manager)

Nimble คือ Package Manager ของ Nim เหมือน npm ของ Node.js หรือ pip ของ Python

### คำสั่ง Nimble พื้นฐาน

```bash
# สร้าง project ใหม่
nimble init myproject

# ติดตั้ง package
nimble install jester        # ติดตั้ง Jester web framework
nimble install jsony         # ติดตั้ง JSON library
nimble install @[jester, jsony]  # ติดตั้งหลายอัน

# Update packages
nimble update
nimble upgrade

# ดู packages ที่ติดตั้ง
nimble list --installed

# ค้นหา packages
nimble search "web"
nimble search "json"

# รัน project
nimble run

# Build project
nimble build

# รัน tests
nimble test
```

### ไฟล์ .nimble (Project Config)

เมื่อสร้าง project ด้วย `nimble init`, จะได้ไฟล์ `.nimble`:

```nim
# myproject.nimble

# Package information
version       = "0.1.0"
author        = "Your Name"
description   = "โปรแกรม Nim แรกของฉัน"
license       = "MIT"
srcDir        = "src"
bin           = @["myproject"]

# Dependencies
requires "nim >= 2.0.0"
requires "jester >= 0.5.0"
requires "jsony >= 1.1.3"
```

### สร้าง Project ด้วย Nimble

```bash
# สร้าง project
nimble init my_backend
cd my_backend

# โครงสร้างที่ได้
my_backend/
├── my_backend.nimble   # Project config
├── src/
│   └── my_backend.nim  # Main source file
└── tests/
    └── test1.nim       # Test file

# แก้ไข src/my_backend.nim
cat > src/my_backend.nim << 'EOF'
proc main() =
  echo "My Backend Application"
  echo "Nim Version: " & NimVersion

main()
EOF

# รัน
nimble run
```

---

## Step 9: Text Editor & IDE Setup

### VS Code (แนะนำ)

```bash
# 1. ติดตั้ง VS Code
# https://code.visualstudio.com/

# 2. ติดตั้ง Extension "Nim" โดย Pietro Peterlongo
# หรือใช้ Command Palette: Ext: Install Extensions -> ค้นหา "Nim"

# 3. ติดตั้ง nimlsp (Language Server)
nimble install nimlsp

# 4. ติดตั้ง nimpretty (Code Formatter)
# มาพร้อมกับ Nim แล้ว
nimpretty --help
```

VS Code Settings สำหรับ Nim (`.vscode/settings.json`):

```json
{
  "nim.buildOnSave": true,
  "nim.lintOnSave": true,
  "nim.projectMapping": {
    "src/*.nim": "src/main.nim"
  },
  "[nim]": {
    "editor.tabSize": 2,
    "editor.formatOnSave": true
  }
}
```

### Neovim/Vim

```bash
# ติดตั้ง nim.vim
# ใช้ vim-plug:
# Plug 'alaviss/nim.nvim'  (Neovim)
# หรือ
# Plug 'zah/nim.vim'       (Vim/Neovim)

# ติดตั้ง LSP
nimble install nimlsp
```

### JetBrains (IntelliJ/CLion)

1. ไปที่ Settings → Plugins
2. ค้นหา "Nim"
3. ติดตั้ง "Nim" plugin โดย Dmitry Matveyev

### Emacs

```elisp
;; ใน .emacs หรือ init.el
(use-package nim-mode
  :ensure t
  :hook (nim-mode . lsp))
```

---

## Step 10: โปรแกรมแรก - Temperature Converter

มาสร้างโปรแกรมที่มีประโยชน์จริงๆ เป็นตัวแปลงอุณหภูมิ:

```nim
# temperature_converter.nim
# โปรแกรมแปลงอุณหภูมิ Celsius <-> Fahrenheit <-> Kelvin

import std/strutils, std/strformat, std/math

# Conversion procedures
proc celsiusToFahrenheit(celsius: float): float =
  return celsius * 9.0 / 5.0 + 32.0

proc fahrenheitToCelsius(fahrenheit: float): float =
  return (fahrenheit - 32.0) * 5.0 / 9.0

proc celsiusToKelvin(celsius: float): float =
  return celsius + 273.15

proc kelvinToCelsius(kelvin: float): float =
  return kelvin - 273.15

proc fahrenheitToKelvin(fahrenheit: float): float =
  return celsiusToKelvin(fahrenheitToCelsius(fahrenheit))

proc kelvinToFahrenheit(kelvin: float): float =
  return celsiusToFahrenheit(kelvinToCelsius(kelvin))

# Display results
proc displayConversions(value: float, unit: string) =
  echo "\n" & "=".repeat(40)
  echo fmt"อุณหภูมิ: {value:.2f} {unit}"
  echo "=".repeat(40)
  
  case unit.toLower():
  of "c", "celsius":
    echo fmt"  Celsius:    {value:.2f}°C"
    echo fmt"  Fahrenheit: {celsiusToFahrenheit(value):.2f}°F"
    echo fmt"  Kelvin:     {celsiusToKelvin(value):.2f}K"
  of "f", "fahrenheit":
    echo fmt"  Fahrenheit: {value:.2f}°F"
    echo fmt"  Celsius:    {fahrenheitToCelsius(value):.2f}°C"
    echo fmt"  Kelvin:     {fahrenheitToKelvin(value):.2f}K"
  of "k", "kelvin":
    echo fmt"  Kelvin:     {value:.2f}K"
    echo fmt"  Celsius:    {kelvinToCelsius(value):.2f}°C"
    echo fmt"  Fahrenheit: {kelvinToFahrenheit(value):.2f}°F"
  else:
    echo "Error: ไม่รู้จักหน่วยอุณหภูมิ"

# Main program
proc main() =
  echo "╔════════════════════════════════════╗"
  echo "║   โปรแกรมแปลงอุณหภูมิ (Nim)        ║"
  echo "╚════════════════════════════════════╝"
  
  # ตัวอย่างการแปลง
  displayConversions(0.0, "C")      # น้ำแข็งละลาย
  displayConversions(100.0, "C")    # น้ำเดือด
  displayConversions(37.0, "C")     # อุณหภูมิร่างกาย
  displayConversions(212.0, "F")    # น้ำเดือด (Fahrenheit)
  displayConversions(273.15, "K")   # จุดเยือกแข็งของน้ำ (Kelvin)
  
  echo "\n--- Interactive Mode ---"
  echo "ป้อนอุณหภูมิ (เช่น: 100 C หรือ 212 F หรือ 373 K):"
  
  let input = readLine(stdin).strip()
  let parts = input.split(" ")
  
  if parts.len >= 2:
    try:
      let value = parseFloat(parts[0])
      let unit = parts[1]
      displayConversions(value, unit)
    except ValueError:
      echo "Error: กรุณาป้อนตัวเลขที่ถูกต้อง"
  else:
    echo "Error: รูปแบบไม่ถูกต้อง กรุณาป้อนแบบ: <ตัวเลข> <หน่วย>"

main()
```

```bash
# Compile และ Run
nim c -r temperature_converter.nim

# Output:
# ╔════════════════════════════════════╗
# ║   โปรแกรมแปลงอุณหภูมิ (Nim)        ║
# ╚════════════════════════════════════╝
# 
# ════════════════════════════════════════
# อุณหภูมิ: 0.00 C
# ════════════════════════════════════════
#   Celsius:    0.00°C
#   Fahrenheit: 32.00°F
#   Kelvin:     273.15K
# ...
```

---

## 📝 สรุป Part 01

ใน Part นี้คุณได้เรียนรู้:

| Step | เนื้อหา | ความสำเร็จ |
|------|---------|-----------|
| 1 | Nim คืออะไร? | ✅ |
| 2 | ทำไมต้องใช้ Nim สำหรับ Backend? | ✅ |
| 3 | ติดตั้งบน Linux | ✅ |
| 4 | ติดตั้งบน macOS | ✅ |
| 5 | ติดตั้งบน Windows | ✅ |
| 6 | Nim Compiler commands | ✅ |
| 7 | Hello World | ✅ |
| 8 | Nimble package manager | ✅ |
| 9 | IDE Setup | ✅ |
| 10 | โปรแกรมจริง (Temperature Converter) | ✅ |

---

## 🏋️ แบบฝึกหัด Part 01

### Exercise 1: ติดตั้งและทดสอบ
```bash
# 1. ติดตั้ง Nim ตามระบบปฏิบัติการของคุณ
# 2. ตรวจสอบ version
nim --version
nimble --version

# 3. เขียนและรัน Hello World
```

### Exercise 2: Calculator โปรแกรมแรก

เขียนโปรแกรม Calculator ที่:
- รับ input จาก user: `5 + 3`
- Parse input และคำนวณ
- แสดงผลลัพธ์

```nim
# calculator.nim
# TODO: เขียนโค้ดตรงนี้

import std/strutils, std/strformat

proc calculate(a: float, op: string, b: float): float =
  # TODO: implement this
  discard

proc main() =
  echo "Simple Calculator"
  echo "ป้อนสมการ (เช่น: 5 + 3):"
  let input = readLine(stdin)
  # TODO: parse และแสดงผล

main()
```

### Exercise 3: แก้ไขโค้ด

โค้ดต่อไปนี้มีข้อผิดพลาด จงหาและแก้ไข:

```nim
# buggy_code.nim
proc greet(name string) =  # Bug 1: missing colon
  echo "Hello " + name      # Bug 2: wrong operator

greet("World)               # Bug 3: missing quote
```

---

## 🔗 ทรัพยากรเพิ่มเติม

- [Nim Official Website](https://nim-lang.org)
- [Nim Documentation](https://nim-lang.org/documentation.html)
- [Nim by Example](https://nim-by-example.github.io)
- [Nim Forum](https://forum.nim-lang.org)
- [Nim on GitHub](https://github.com/nim-lang/Nim)
- [Nimble Packages](https://nimble.directory)

---

**ต่อไป → [Part 02: Variables, Data Types & Constants](part_02_variables_types.md)**
