# Part 01: แนะนำ Kotlin และการติดตั้ง
## Introduction to Kotlin & Setup

---

## 🎯 เป้าหมายของ Part นี้

- เข้าใจว่า Kotlin คืออะไร และทำไมต้องใช้
- ติดตั้ง JDK, IntelliJ IDEA และ Kotlin
- เขียนโปรแกรม "Hello World" แรก
- เข้าใจโครงสร้างพื้นฐานของ Kotlin program

---

## 📖 1. Kotlin คืออะไร?

Kotlin เป็นภาษาโปรแกรมที่พัฒนาโดย **JetBrains** (บริษัทที่ทำ IntelliJ IDEA) ปล่อยตัวเมื่อปี 2011 และกลายเป็น **official language ของ Android** ในปี 2017 โดย Google

### ✅ จุดเด่นของ Kotlin

```
1. Concise (กระชับ)
   - เขียนโค้ดน้อยกว่า Java มาก
   - ลด boilerplate code ลงอย่างมาก

2. Safe (ปลอดภัย)
   - Null Safety built-in
   - ลด NullPointerException

3. Interoperable (ทำงานร่วมกับ Java ได้)
   - ใช้ Java libraries ทั้งหมดได้
   - เรียก Kotlin จาก Java ได้

4. Pragmatic (ใช้งานจริง)
   - ออกแบบสำหรับ production use
   - เรียนรู้ง่าย

5. Multi-platform
   - JVM, Android, JavaScript, Native
```

### 📊 เปรียบเทียบ Kotlin vs Java

```kotlin
// Java แบบเดิม (verbose มาก)
public class Person {
    private String name;
    private int age;
    
    public Person(String name, int age) {
        this.name = name;
        this.age = age;
    }
    
    public String getName() { return name; }
    public int getAge() { return age; }
    
    @Override
    public String toString() {
        return "Person(name=" + name + ", age=" + age + ")";
    }
    
    @Override
    public boolean equals(Object o) { ... }
    
    @Override
    public int hashCode() { ... }
}
```

```kotlin
// Kotlin แบบใหม่ (1 บรรทัด!)
data class Person(val name: String, val age: Int)
```

**ผลลัพธ์เหมือนกัน แต่ Kotlin กระชับกว่าหลายเท่า!**

---

## 💻 2. การติดตั้ง Environment

### Step 1: ติดตั้ง JDK 17+

**macOS (ใช้ Homebrew):**
```bash
# ติดตั้ง Homebrew ถ้ายังไม่มี
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# ติดตั้ง JDK 17
brew install openjdk@17

# เพิ่ม PATH
echo 'export PATH="/opt/homebrew/opt/openjdk@17/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc

# ตรวจสอบ
java -version
```

**Windows (ใช้ Chocolatey):**
```powershell
# ติดตั้ง Chocolatey ก่อน (run as Admin)
Set-ExecutionPolicy Bypass -Scope Process -Force
[System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072
iex ((New-Object System.Net.WebClient).DownloadString('https://community.chocolatey.org/install.ps1'))

# ติดตั้ง JDK 17
choco install openjdk17

# ตรวจสอบ
java -version
```

**Linux (Ubuntu/Debian):**
```bash
sudo apt update
sudo apt install openjdk-17-jdk

# ตรวจสอบ
java -version
javac -version
```

**ผลลัพธ์ที่ควรได้:**
```
openjdk version "17.0.x" 2023-xx-xx
OpenJDK Runtime Environment (build 17.0.x+x)
OpenJDK 64-Bit Server VM (build 17.0.x+x, mixed mode, sharing)
```

### Step 2: ติดตั้ง IntelliJ IDEA

1. ไปที่ https://www.jetbrains.com/idea/download/
2. เลือก **Community Edition** (ฟรี) หรือ **Ultimate** (มีฟีเจอร์ Spring เพิ่ม)
3. ดาวน์โหลดและติดตั้งตามระบบปฏิบัติการ

> 💡 **แนะนำ:** ใช้ **Ultimate Edition** เพราะมี Spring Boot support ครบ ถ้าเป็นนักศึกษาขอ free license ได้ที่ https://www.jetbrains.com/student/

### Step 3: ติดตั้ง Kotlin Plugin

IntelliJ IDEA มี Kotlin built-in อยู่แล้ว แต่ถ้าใช้ VS Code:

```bash
# ติดตั้ง Kotlin extension
code --install-extension fwcd.kotlin
code --install-extension mathiasfrohlich.Kotlin
```

### Step 4: ติดตั้ง Kotlin Compiler (Optional)

```bash
# macOS
brew install kotlin

# ตรวจสอบ
kotlinc -version
```

---

## 🖊️ 3. Hello World แรก

### วิธีที่ 1: ใช้ IntelliJ IDEA

1. เปิด IntelliJ IDEA
2. คลิก **New Project**
3. เลือก **Kotlin** → **Kotlin/JVM**
4. ตั้งชื่อ Project: `kotlin-basics`
5. เลือก **Gradle** เป็น Build System
6. คลิก **Create**

สร้างไฟล์ `src/main/kotlin/Main.kt`:

```kotlin
fun main() {
    println("Hello, World!")
    println("ยินดีต้อนรับสู่โลกของ Kotlin!")
}
```

รันโปรแกรม:
- กด **Shift + F10** หรือ
- คลิกปุ่ม **▶** สีเขียว

**ผลลัพธ์:**
```
Hello, World!
ยินดีต้อนรับสู่โลกของ Kotlin!
```

### วิธีที่ 2: ใช้ Command Line

```bash
# สร้างไฟล์
cat > Hello.kt << 'EOF'
fun main() {
    println("Hello, World!")
}
EOF

# Compile
kotlinc Hello.kt -include-runtime -d Hello.jar

# รัน
java -jar Hello.jar
```

### วิธีที่ 3: Kotlin REPL (Interactive Mode)

```bash
# เปิด REPL
kotlinc

# พิมพ์คำสั่ง
>>> println("Hello, World!")
Hello, World!

>>> 1 + 2
res0: kotlin.Int = 3

>>> "Kotlin" + " is " + "awesome!"
res1: kotlin.String = Kotlin is awesome!

>>> :quit
```

### วิธีที่ 4: Kotlin Playground Online

ไปที่ https://play.kotlinlang.org/ แล้วพิมพ์โค้ดได้เลย ไม่ต้องติดตั้งอะไร

---

## 🔍 4. ทำความเข้าใจโครงสร้างโปรแกรม

```kotlin
// 1. Comment (ความคิดเห็น) - ไม่มีผลต่อการทำงาน
// นี่คือ single-line comment

/*
 นี่คือ
 multi-line comment
*/

// 2. Function หลัก - จุดเริ่มต้นของโปรแกรม
fun main() {
    // 3. Statement - คำสั่งที่โปรแกรมต้องทำ
    println("Hello, World!")  // พิมพ์ข้อความ + ขึ้นบรรทัดใหม่
    print("Hello ")           // พิมพ์ข้อความ ไม่ขึ้นบรรทัดใหม่
    print("World!")
    println()                 // ขึ้นบรรทัดใหม่เปล่าๆ
}
```

### เจาะลึก `main` function

```kotlin
// รูปแบบที่ 1: ไม่มี args
fun main() {
    println("No arguments")
}

// รูปแบบที่ 2: มี args (รับ command line arguments)
fun main(args: Array<String>) {
    println("Number of args: ${args.size}")
    for (arg in args) {
        println("Arg: $arg")
    }
}
```

รันด้วย arguments:
```bash
java -jar MyApp.jar arg1 arg2 arg3
# Output:
# Number of args: 3
# Arg: arg1
# Arg: arg2
# Arg: arg3
```

---

## 📦 5. โครงสร้าง Project

```
my-kotlin-project/
├── build.gradle.kts          # Build configuration (Kotlin DSL)
├── settings.gradle.kts       # Project settings
├── src/
│   ├── main/
│   │   ├── kotlin/           # Kotlin source files
│   │   │   └── com/
│   │   │       └── example/
│   │   │           └── Main.kt
│   │   └── resources/        # Configuration files
│   └── test/
│       ├── kotlin/           # Test files
│       └── resources/
└── gradle/
    └── wrapper/
        ├── gradle-wrapper.jar
        └── gradle-wrapper.properties
```

### build.gradle.kts พื้นฐาน

```kotlin
plugins {
    kotlin("jvm") version "1.9.22"
    application
}

group = "com.example"
version = "1.0-SNAPSHOT"

repositories {
    mavenCentral()
}

dependencies {
    // Kotlin standard library
    implementation(kotlin("stdlib"))
    
    // Testing
    testImplementation(kotlin("test"))
}

application {
    mainClass.set("MainKt")  // ชี้ไปที่ไฟล์ Main.kt
}

tasks.test {
    useJUnitPlatform()
}

kotlin {
    jvmToolchain(17)
}
```

---

## 🎮 6. โปรแกรมตัวอย่างเพิ่มเติม

### ตัวอย่างที่ 1: รับ Input จากผู้ใช้

```kotlin
fun main() {
    print("กรุณาใส่ชื่อของคุณ: ")
    val name = readLine()  // อ่าน input จาก keyboard
    
    if (name != null && name.isNotEmpty()) {
        println("สวัสดี, $name!")
        println("ยินดีต้อนรับสู่หลักสูตร Kotlin")
    } else {
        println("สวัสดี, ผู้มาเยือน!")
    }
}
```

**ผลลัพธ์:**
```
กรุณาใส่ชื่อของคุณ: สมชาย
สวัสดี, สมชาย!
ยินดีต้อนรับสู่หลักสูตร Kotlin
```

### ตัวอย่างที่ 2: คำนวณง่ายๆ

```kotlin
fun main() {
    println("=== เครื่องคิดเลขง่ายๆ ===")
    
    print("ใส่ตัวเลขแรก: ")
    val num1 = readLine()?.toDoubleOrNull() ?: 0.0
    
    print("ใส่ตัวเลขที่สอง: ")
    val num2 = readLine()?.toDoubleOrNull() ?: 0.0
    
    println("\nผลการคำนวณ:")
    println("$num1 + $num2 = ${num1 + num2}")
    println("$num1 - $num2 = ${num1 - num2}")
    println("$num1 × $num2 = ${num1 * num2}")
    
    if (num2 != 0.0) {
        println("$num1 ÷ $num2 = ${num1 / num2}")
    } else {
        println("ไม่สามารถหารด้วย 0 ได้!")
    }
}
```

**ผลลัพธ์:**
```
=== เครื่องคิดเลขง่ายๆ ===
ใส่ตัวเลขแรก: 10
ใส่ตัวเลขที่สอง: 3

ผลการคำนวณ:
10.0 + 3.0 = 13.0
10.0 - 3.0 = 7.0
10.0 × 3.0 = 30.0
10.0 ÷ 3.0 = 3.3333333333333335
```

### ตัวอย่างที่ 3: ข้อมูลส่วนตัว

```kotlin
fun main() {
    // ข้อมูลส่วนตัว
    val name = "สมชาย ใจดี"
    val age = 25
    val height = 175.5
    val isStudent = true
    
    // แสดงผล
    println("========== บัตรประจำตัว ==========")
    println("ชื่อ: $name")
    println("อายุ: $age ปี")
    println("ส่วนสูง: $height ซม.")
    println("เป็นนักศึกษา: ${if (isStudent) "ใช่" else "ไม่ใช่"}")
    println("====================================")
}
```

**ผลลัพธ์:**
```
========== บัตรประจำตัว ==========
ชื่อ: สมชาย ใจดี
อายุ: 25 ปี
ส่วนสูง: 175.5 ซม.
เป็นนักศึกษา: ใช่
====================================
```

---

## 🔧 7. Kotlin vs Java: เปรียบเทียบโดยละเอียด

### 7.1 การประกาศตัวแปร

```kotlin
// Kotlin - กระชับ
val name = "Alice"           // Immutable (ไม่เปลี่ยนได้)
var age = 25                 // Mutable (เปลี่ยนได้)

// Java
final String name = "Alice"; // Immutable
int age = 25;                // Mutable
```

### 7.2 String Interpolation

```kotlin
// Kotlin - ง่ายและสวย
val name = "Bob"
val greeting = "Hello, $name! You are ${name.length} chars long."

// Java - ยุ่งยาก
String name = "Bob";
String greeting = "Hello, " + name + "! You are " + name.length() + " chars long.";
```

### 7.3 Null Safety

```kotlin
// Kotlin - ชัดเจนเรื่อง null
var name: String = "Alice"    // ห้าม null
var nickname: String? = null  // อนุญาต null

// เข้าถึงอย่างปลอดภัย
println(nickname?.length)    // ถ้า null จะได้ null
println(nickname ?: "No nickname") // Elvis operator

// Java - เสี่ยง NullPointerException
String name = "Alice";
String nickname = null;
System.out.println(nickname.length()); // 💥 NullPointerException!
```

### 7.4 Class และ Constructor

```kotlin
// Kotlin
class Person(val name: String, val age: Int) {
    fun greet() = "Hi, I'm $name, $age years old"
}

// Java
public class Person {
    private final String name;
    private final int age;
    
    public Person(String name, int age) {
        this.name = name;
        this.age = age;
    }
    
    public String getName() { return name; }
    public int getAge() { return age; }
    
    public String greet() {
        return "Hi, I'm " + name + ", " + age + " years old";
    }
}
```

---

## 🏋️ 8. แบบฝึกหัด

### แบบฝึกหัดที่ 1: Hello with Name
สร้างโปรแกรมที่ถามชื่อผู้ใช้แล้วแสดง greeting

```kotlin
// TODO: เขียนโค้ดของคุณที่นี่
fun main() {
    // 1. ถามชื่อ
    // 2. รับ input
    // 3. แสดง "สวัสดี, [ชื่อ]! ยินดีต้อนรับสู่ Kotlin"
}
```

**เฉลย:**
```kotlin
fun main() {
    print("ชื่อของคุณคือ: ")
    val name = readLine() ?: "ผู้มาเยือน"
    println("สวัสดี, $name! ยินดีต้อนรับสู่ Kotlin")
}
```

### แบบฝึกหัดที่ 2: อายุและปีเกิด
สร้างโปรแกรมที่รับอายุแล้วคำนวณปีเกิด

```kotlin
fun main() {
    // ปัจจุบัน 2026
    print("อายุของคุณ: ")
    val age = readLine()?.toIntOrNull() ?: 0
    val birthYear = 2026 - age
    println("คุณเกิดปี: $birthYear")
    println("ในระบบ AD: $birthYear")
    println("ในระบบ พ.ศ.: ${birthYear + 543}")
}
```

### แบบฝึกหัดที่ 3: รูปทรงสี่เหลี่ยม
สร้างโปรแกรมคำนวณพื้นที่และเส้นรอบรูปสี่เหลี่ยม

```kotlin
fun main() {
    print("กว้าง (เมตร): ")
    val width = readLine()?.toDoubleOrNull() ?: 0.0
    
    print("ยาว (เมตร): ")
    val length = readLine()?.toDoubleOrNull() ?: 0.0
    
    val area = width * length
    val perimeter = 2 * (width + length)
    
    println("\nผลการคำนวณ:")
    println("พื้นที่ = $area ตร.ม.")
    println("เส้นรอบรูป = $perimeter ม.")
}
```

---

## 📝 9. สรุปสิ่งที่เรียนใน Part นี้

| หัวข้อ | สิ่งที่เรียน |
|--------|-------------|
| Kotlin คืออะไร | ภาษา modern JVM ที่กระชับและปลอดภัย |
| ติดตั้ง | JDK 17, IntelliJ IDEA, Kotlin |
| Hello World | `fun main() { println("Hello") }` |
| รับ Input | `readLine()` |
| String Template | `"Hello, $name"` หรือ `"${expression}"` |
| Comments | `//` single-line, `/* */` multi-line |

---

## 🔗 แหล่งอ้างอิง

- [Kotlin Official Documentation](https://kotlinlang.org/docs/home.html)
- [Kotlin Playground](https://play.kotlinlang.org/)
- [JetBrains Academy - Kotlin](https://hyperskill.org/tracks/18)
- [Kotlin Koans](https://kotlinlang.org/docs/koans.html)

---

## ➡️ ถัดไป: Part 02 - ตัวแปร, ชนิดข้อมูล และ Type System

ใน Part ถัดไปเราจะเรียนรู้เกี่ยวกับ:
- ชนิดข้อมูลทั้งหมดใน Kotlin
- การประกาศตัวแปรด้วย `val` และ `var`
- Type inference และ explicit types
- การแปลงชนิดข้อมูล

---
*Part 01/100+ | Kotlin & Spring Boot Complete Course*
