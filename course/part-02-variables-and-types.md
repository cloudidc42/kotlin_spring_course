# Part 02: ตัวแปร, ชนิดข้อมูล และ Type System
## Variables, Data Types & Type System

---

## 🎯 เป้าหมายของ Part นี้

- เข้าใจ `val` vs `var` และเมื่อไหรควรใช้อะไร
- รู้จักชนิดข้อมูลทั้งหมดใน Kotlin
- เข้าใจ Type Inference
- แปลงชนิดข้อมูล (Type Conversion)
- เข้าใจ Nullable Types

---

## 📖 1. val vs var

### `val` - Immutable (ไม่เปลี่ยนค่าได้)

```kotlin
val name = "Alice"
// name = "Bob"  // ❌ Error! Val cannot be reassigned

val pi = 3.14159
val maxSize = 100
val greeting = "Hello, World!"
```

### `var` - Mutable (เปลี่ยนค่าได้)

```kotlin
var score = 0
score = 100       // ✅ OK
score += 50       // ✅ OK, score = 150

var message = "Hello"
message = "Goodbye"  // ✅ OK

var counter = 0
counter++         // ✅ OK
counter--         // ✅ OK
```

### 💡 Best Practice: ใช้ `val` เสมอ ยกเว้นจำเป็นต้องเปลี่ยนค่า

```kotlin
// ✅ แนะนำ
val firstName = "Alice"
val lastName = "Smith"
val fullName = "$firstName $lastName"

// ใช้ var เฉพาะเมื่อจำเป็น
var attempts = 0
while (attempts < 3) {
    attempts++
}
```

### ความแตกต่างระหว่าง val และ const val

```kotlin
// val - computed at runtime
val currentTime = System.currentTimeMillis()  // คำนวณตอนรัน

// const val - computed at compile time (ต้องอยู่ใน top-level หรือ object)
const val MAX_COUNT = 100          // คำนวณตอน compile
const val APP_NAME = "MyApp"
const val PI = 3.14159

// object สำหรับ constants
object Constants {
    const val BASE_URL = "https://api.example.com"
    const val TIMEOUT = 30000L
    const val MAX_RETRY = 3
}

fun main() {
    println(Constants.BASE_URL)
    println(Constants.TIMEOUT)
}
```

---

## 📊 2. ชนิดข้อมูลพื้นฐาน (Basic Types)

### 2.1 ตัวเลข (Numbers)

```kotlin
// Integer types (จำนวนเต็ม)
val byteVal: Byte = 127              // -128 ถึง 127 (8 bits)
val shortVal: Short = 32767          // -32,768 ถึง 32,767 (16 bits)
val intVal: Int = 2_147_483_647      // -2B ถึง 2B (32 bits) - ใช้บ่อยที่สุด
val longVal: Long = 9_223_372_036_854_775_807L  // (64 bits) ต้องมี L

// Floating point types (ทศนิยม)
val floatVal: Float = 3.14f          // Single precision (32 bits) ต้องมี f
val doubleVal: Double = 3.14159265358979  // Double precision (64 bits) - ใช้บ่อยที่สุด

// Unsigned integers (ไม่มีลบ) - Kotlin 1.5+
val uByte: UByte = 255u
val uShort: UShort = 65535u
val uInt: UInt = 4294967295u
val uLong: ULong = 18446744073709551615uL

println("Byte: $byteVal")
println("Int: $intVal")
println("Long: $longVal")
println("Float: $floatVal")
println("Double: $doubleVal")
```

### ตัวคั่นหลัก (Digit Separators)

```kotlin
// ใช้ _ เพื่อให้อ่านง่าย
val population = 8_000_000_000L      // 8 พันล้าน
val creditCard = 1234_5678_9012_3456L
val byteMask = 0xFF_FF_FF_FF
val hexColor = 0xFF_AA_00
val binaryData = 0b1111_0000_1010_0101

println("Population: $population")
println("Hex color: #${hexColor.toString(16).uppercase()}")
```

### 2.2 Boolean

```kotlin
val isKotlinFun: Boolean = true
val isJavaBetter = false  // Type inference

println(isKotlinFun)          // true
println(!isKotlinFun)         // false (negation)
println(isKotlinFun && false) // false (AND)
println(isKotlinFun || false) // true (OR)
println(isKotlinFun xor true) // false (XOR)
```

### 2.3 Characters

```kotlin
val letter: Char = 'A'
val digit: Char = '9'
val thai: Char = 'ก'
val emoji: Char = '😊'  // ⚠️ บางตัวอาจมีปัญหา

// Character operations
println(letter.code)           // 65 (ASCII/Unicode value)
println(letter.lowercaseChar()) // a
println(letter.uppercaseChar()) // A
println(letter.isLetter())     // true
println(digit.isDigit())       // true
println(digit.digitToInt())    // 9

// Char arithmetic
val nextLetter = letter + 1    // 'B'
println(nextLetter)            // B
```

### 2.4 Strings

```kotlin
// การสร้าง String
val simple = "Hello, World!"
val withEscape = "Line 1\nLine 2\tTabbed"
val withQuote = "He said \"Hello\""

// Raw String (Multiline, no escaping)
val multiline = """
    Hello,
    World!
    This is a multiline string.
""".trimIndent()

println(multiline)
// Hello,
// World!
// This is a multiline string.

// String Template
val name = "Alice"
val age = 25
val intro = "My name is $name and I'm $age years old."
val calc = "2 + 3 = ${2 + 3}"
val upper = "Name in uppercase: ${name.uppercase()}"

println(intro)
println(calc)
println(upper)
```

### String Operations

```kotlin
val text = "Hello, Kotlin World!"

// ความยาว
println(text.length)              // 20

// การค้นหา
println(text.contains("Kotlin"))  // true
println(text.startsWith("Hello")) // true
println(text.endsWith("World!"))  // true
println(text.indexOf("Kotlin"))   // 7

// การแปลง
println(text.uppercase())         // HELLO, KOTLIN WORLD!
println(text.lowercase())         // hello, kotlin world!
println(text.reversed())          // !dlroW niltoK ,olleH

// การตัด
println(text.substring(7, 13))   // Kotlin
println(text.take(5))             // Hello
println(text.drop(7))             // Kotlin World!
println(text.trim())              // ตัด whitespace หัว-ท้าย

// การแทนที่
println(text.replace("Kotlin", "Java"))  // Hello, Java World!
println(text.replace(Regex("[aeiou]"), "*"))  // H*ll*, K*tl*n W*rld!

// การแยก
val csv = "apple,banana,cherry"
val fruits = csv.split(",")
println(fruits)    // [apple, banana, cherry]
println(fruits[0]) // apple

// การรวม
val words = listOf("Hello", "Kotlin", "World")
println(words.joinToString(" "))           // Hello Kotlin World
println(words.joinToString(", "))         // Hello, Kotlin, World
println(words.joinToString(prefix = "[", postfix = "]")) // [Hello, Kotlin, World]

// การตรวจสอบ
println("  ".isBlank())      // true (ว่างเปล่า หรือ whitespace)
println("".isEmpty())        // true (ว่างเปล่าล้วน)
println("Hello".isNotEmpty()) // true

// Padding
println("42".padStart(5, '0'))   // 00042
println("hi".padEnd(10, '.'))    // hi........
```

---

## 🔄 3. Type Inference (การอนุมานชนิดข้อมูล)

```kotlin
// Kotlin อนุมาน type อัตโนมัติ
val name = "Alice"          // String
val age = 25                // Int
val height = 175.5          // Double
val active = true           // Boolean
val initial = 'A'           // Char

// เทียบกับการระบุ type ชัดเจน
val name2: String = "Alice"
val age2: Int = 25
val height2: Double = 175.5

// ใช้งานได้เหมือนกัน แต่แบบแรกกระชับกว่า
```

### เมื่อไหรควรระบุ type ชัดเจน?

```kotlin
// 1. เมื่อต้องการ type ที่ต่างจาก default
val score: Long = 100          // default จะเป็น Int
val price: Float = 9.99f      // default จะเป็น Double (ต้องมี f ด้วย)
val smallNum: Byte = 10        // default จะเป็น Int

// 2. เมื่อประกาศโดยยังไม่ assign ค่า
var result: String             // ต้องระบุ type
result = "Success"

// 3. เพื่อความชัดเจนใน API public
fun calculateArea(width: Double, height: Double): Double {
    return width * height
}
```

---

## 🔀 4. Type Conversion (การแปลงชนิดข้อมูล)

### Explicit Conversion (Kotlin ไม่ทำ implicit conversion!)

```kotlin
val intNum: Int = 42

// ต้องแปลงชัดเจน
val toLong: Long = intNum.toLong()
val toDouble: Double = intNum.toDouble()
val toFloat: Float = intNum.toFloat()
val toByte: Byte = intNum.toByte()
val toShort: Short = intNum.toShort()
val toString: String = intNum.toString()

println(toLong)    // 42
println(toDouble)  // 42.0
println(toString)  // "42"

// ❌ ใน Java ทำได้ แต่ Kotlin ไม่อนุญาต
// val longNum: Long = intNum  // Error!

// ✅ ต้องแปลงเสมอ
val longNum: Long = intNum.toLong()
```

### การแปลง String เป็นตัวเลข

```kotlin
// String → Number
val strNum = "42"
val intFromStr = strNum.toInt()         // 42
val doubleFromStr = "3.14".toDouble()   // 3.14
val longFromStr = "100000000".toLong()  // 100000000

// Safe conversion (ถ้าแปลงไม่ได้จะได้ null)
val safeInt = "abc".toIntOrNull()     // null (ไม่ใช่ตัวเลข)
val safeInt2 = "42".toIntOrNull()     // 42

// ใช้กับ Elvis operator
val value = "not a number".toIntOrNull() ?: 0  // 0 (ค่า default)
println(value)  // 0

// หลายรูปแบบ
println("FF".toInt(16))    // 255 (hex)
println("1111".toInt(2))   // 15 (binary)
println("17".toInt(8))     // 15 (octal)
```

### Number Formatting

```kotlin
val bigNumber = 1234567.89

// ใช้ String.format
println(String.format("%.2f", bigNumber))  // 1234567.89
println(String.format("%,.2f", bigNumber)) // 1,234,567.89
println(String.format("%10.2f", bigNumber)) // "1234567.89" (width 10)

// ใช้ Kotlin way
println("%.2f".format(bigNumber))          // 1234567.89

// ใช้ NumberFormat (Java)
import java.text.NumberFormat
import java.util.Locale

val formatter = NumberFormat.getNumberInstance(Locale("th", "TH"))
println(formatter.format(bigNumber))       // 1,234,567.89 (Thai format)
```

---

## ❓ 5. Nullable Types (ชนิดข้อมูลที่รับ null ได้)

### ปัญหาใน Java

```java
// Java - เสี่ยง NullPointerException
String name = null;
System.out.println(name.length()); // 💥 NullPointerException!
```

### Kotlin's Null Safety

```kotlin
// ✅ Non-nullable (ห้าม null - default)
var name: String = "Alice"
// name = null  // ❌ Compilation Error!

// ✅ Nullable (อนุญาต null - ต้องใส่ ?)
var nickname: String? = null
nickname = "Ali"
nickname = null  // ✅ OK

// ขนาด int
var count: Int = 0
var maybeCount: Int? = null
```

### การใช้งาน Nullable Types

```kotlin
var text: String? = null

// 1. Safe call operator (?.)
println(text?.length)        // null (ไม่ crash)
println(text?.uppercase())   // null

// 2. Elvis operator (?:) - ค่า default ถ้าเป็น null
val length = text?.length ?: 0
println(length)              // 0

// 3. Not-null assertion (!!) - ถ้า null จะ crash
// ใช้เมื่อแน่ใจ 100% ว่าไม่ null
text = "Hello"
println(text!!.length)       // 5

// 4. Safe cast (as?)
val obj: Any = "Hello"
val str = obj as? String     // "Hello"
val num = obj as? Int        // null (ไม่ใช่ Int)

// 5. let function
text?.let { nonNullText ->
    println("Text is: $nonNullText")
    println("Length: ${nonNullText.length}")
}
// ถ้า text เป็น null โค้ดใน let จะไม่ทำงาน
```

### Null Checks

```kotlin
var email: String? = null

// วิธีที่ 1: if-null check
if (email != null) {
    // ใน block นี้ email ถูก smart cast เป็น String (non-nullable)
    println(email.length)    // ✅ ปลอดภัย
}

// วิธีที่ 2: Safe call
println(email?.length)      // null

// วิธีที่ 3: Elvis
val emailLength = email?.length ?: -1
println(emailLength)        // -1

// วิธีที่ 4: let
email?.let {
    println("Email: $it")
    println("Domain: ${it.substringAfter("@")}")
}

// assign แล้วใช้
email = "alice@example.com"
email?.let {
    println("Email: $it")
    // Email: alice@example.com
}
```

---

## 🎯 6. Any, Unit, Nothing

### Any - parent ของทุก type

```kotlin
val anything: Any = "Hello"   // String
val anything2: Any = 42       // Int
val anything3: Any = 3.14     // Double
val anything4: Any = true     // Boolean

fun printAnything(value: Any) {
    println(value)
}

printAnything("Hello")
printAnything(42)
printAnything(listOf(1, 2, 3))

// ตรวจสอบ type
println(anything is String)   // true
println(anything is Int)      // false
```

### Unit - เหมือน void ใน Java

```kotlin
// Function ที่ไม่คืนค่า return type คือ Unit
fun printMessage(message: String): Unit {
    println(message)
}

// Unit เป็น default return type จึงไม่ต้องเขียน
fun printMessage2(message: String) {
    println(message)
}

// Unit เป็น object จริงๆ (singleton)
val unit: Unit = Unit
println(unit)  // kotlin.Unit
```

### Nothing - ไม่มี instance เลย

```kotlin
// Function ที่ไม่มีทางคืนค่าได้ (throw หรือ infinite loop)
fun fail(message: String): Nothing {
    throw IllegalStateException(message)
}

fun infiniteLoop(): Nothing {
    while (true) {
        // never returns
    }
}

// ประโยชน์ใน type system
val value: String = null ?: fail("Value cannot be null")
// compiler รู้ว่า fail() ไม่คืนค่า ดังนั้น value จะเป็น String เสมอ
```

---

## 🔢 7. Arrays

```kotlin
// การสร้าง Array
val numbers = arrayOf(1, 2, 3, 4, 5)
val strings = arrayOf("apple", "banana", "cherry")
val mixed = arrayOf(1, "two", 3.0, true)

// Typed Arrays (ประสิทธิภาพดีกว่า)
val intArray = intArrayOf(1, 2, 3, 4, 5)
val doubleArray = doubleArrayOf(1.0, 2.0, 3.0)
val boolArray = booleanArrayOf(true, false, true)

// Array by size
val zeros = IntArray(5)          // [0, 0, 0, 0, 0]
val fiveOnes = IntArray(5) { 1 } // [1, 1, 1, 1, 1]
val squares = IntArray(5) { i -> (i + 1) * (i + 1) }  // [1, 4, 9, 16, 25]

// เข้าถึงข้อมูล
println(numbers[0])        // 1
println(numbers.last())    // 5
println(numbers.size)      // 5
println(numbers.first())   // 1

// แก้ไข (Array เป็น mutable)
numbers[0] = 10
println(numbers[0])        // 10

// วนซ้ำ
for (n in numbers) {
    print("$n ")
}
// 10 2 3 4 5

// Array operations
println(numbers.sum())        // 24
println(numbers.average())    // 4.8
println(numbers.max())        // 10
println(numbers.min())        // 2
println(numbers.contains(3))  // true
println(numbers.toList())     // [10, 2, 3, 4, 5]

// 2D Array
val matrix = Array(3) { row ->
    IntArray(3) { col -> row * 3 + col + 1 }
}
// [[1, 2, 3], [4, 5, 6], [7, 8, 9]]

for (row in matrix) {
    println(row.toList())
}
```

---

## 🧮 8. Type Checking และ Smart Casts

```kotlin
fun describe(value: Any): String {
    return when (value) {
        is Int -> "Integer: $value"
        is String -> "String of length ${value.length}"  // Smart cast!
        is Boolean -> "Boolean: $value"
        is List<*> -> "List with ${value.size} elements"
        is Double -> "Double: $value"
        else -> "Unknown: $value"
    }
}

println(describe(42))          // Integer: 42
println(describe("Hello"))     // String of length 5
println(describe(true))        // Boolean: true
println(describe(listOf(1,2))) // List with 2 elements

// Smart Cast ใน if
fun processInput(input: Any) {
    if (input is String) {
        // ใน block นี้ input ถูก smart cast เป็น String
        println("Upper: ${input.uppercase()}")    // ✅
        println("Length: ${input.length}")         // ✅
    }
    
    if (input is Int && input > 0) {
        println("Positive int: $input")
    }
}

processInput("hello")
processInput(42)
```

### Unsafe Cast vs Safe Cast

```kotlin
val obj: Any = "Hello, World!"

// Unsafe cast - ถ้าผิด type จะ ClassCastException
val str: String = obj as String  // OK ถ้า obj เป็น String
// val num: Int = obj as Int    // 💥 ClassCastException!

// Safe cast - ถ้าผิด type จะได้ null
val str2: String? = obj as? String  // "Hello, World!"
val num: Int? = obj as? Int         // null (ไม่ crash)

println(str2)  // Hello, World!
println(num)   // null
```

---

## 💡 9. Best Practices

### การตั้งชื่อตัวแปร

```kotlin
// ✅ ดี - camelCase, ชื่อบอกความหมาย
val firstName = "Alice"
val totalAmount = 1000.0
val isUserLoggedIn = false
val maxRetryCount = 3

// ❌ ไม่ดี
val fn = "Alice"           // ย่อเกินไป
val x = 1000.0             // ไม่บอกความหมาย
val flag = false           // คลุมเครือ
val MAX_RETRY_COUNT = 3    // ใช้ uppercase สำหรับ const เท่านั้น

// Constants - SCREAMING_SNAKE_CASE
const val MAX_CONNECTIONS = 100
const val DEFAULT_TIMEOUT = 30000L
const val APP_VERSION = "1.0.0"
```

### ตัวอย่างโปรแกรมจริง

```kotlin
import java.text.NumberFormat
import java.util.Locale

fun main() {
    // ข้อมูลสินค้า
    val productName = "Kotlin Programming Book"
    val originalPrice: Double = 599.0
    val discountPercent: Int = 20
    
    // คำนวณ
    val discountAmount = originalPrice * discountPercent / 100
    val finalPrice = originalPrice - discountAmount
    val vat = finalPrice * 0.07
    val totalPrice = finalPrice + vat
    
    // format ราคา
    val formatter = NumberFormat.getCurrencyInstance(Locale("th", "TH"))
    
    // แสดงผล
    println("╔══════════════════════════════════╗")
    println("║          ใบเสร็จรับเงิน          ║")
    println("╠══════════════════════════════════╣")
    println("║ สินค้า: $productName")
    println("║ ราคาปกติ: ${formatter.format(originalPrice)}")
    println("║ ส่วนลด $discountPercent%: -${formatter.format(discountAmount)}")
    println("║ ราคาหลังลด: ${formatter.format(finalPrice)}")
    println("║ VAT 7%: ${formatter.format(vat)}")
    println("╠══════════════════════════════════╣")
    println("║ รวมทั้งสิ้น: ${formatter.format(totalPrice)}")
    println("╚══════════════════════════════════╝")
}
```

**ผลลัพธ์:**
```
╔══════════════════════════════════╗
║          ใบเสร็จรับเงิน          ║
╠══════════════════════════════════╣
║ สินค้า: Kotlin Programming Book
║ ราคาปกติ: ฿599.00
║ ส่วนลด 20%: -฿119.80
║ ราคาหลังลด: ฿479.20
║ VAT 7%: ฿33.54
╠══════════════════════════════════╣
║ รวมทั้งสิ้น: ฿512.74
╚══════════════════════════════════╝
```

---

## 🏋️ 10. แบบฝึกหัด

### ข้อ 1: ประกาศตัวแปร
ประกาศตัวแปรต่อไปนี้ให้ถูกต้อง:
- ชื่อของคุณ (ไม่เปลี่ยน)
- อายุปัจจุบัน (เปลี่ยนได้)
- ส่วนสูง (ทศนิยม)
- ชอบ Kotlin หรือไม่ (boolean)

**เฉลย:**
```kotlin
val myName = "Alice"        // val เพราะชื่อไม่เปลี่ยน
var myAge = 25              // var เพราะอายุเพิ่มได้
val myHeight = 165.5        // val เพราะส่วนสูงไม่ค่อยเปลี่ยน
val likesKotlin = true      // val
```

### ข้อ 2: Type Conversion
แปลง "12345" เป็น Int แล้วบวกกับ 100

**เฉลย:**
```kotlin
val numStr = "12345"
val num = numStr.toInt()
val result = num + 100
println(result)  // 12445

// หรือแบบ safe
val safeResult = numStr.toIntOrNull()?.plus(100) ?: 0
println(safeResult)  // 12445
```

### ข้อ 3: Nullable handling
รับ email จากผู้ใช้ ถ้าว่างให้แสดง "ไม่ได้ใส่ email"

**เฉลย:**
```kotlin
fun main() {
    print("ใส่ email (หรือกด Enter ข้าม): ")
    val email: String? = readLine()?.takeIf { it.isNotEmpty() }
    
    val message = email?.let {
        "Email ของคุณคือ: $it"
    } ?: "ไม่ได้ใส่ email"
    
    println(message)
}
```

### ข้อ 4: Temperature Converter
สร้างโปรแกรมแปลงอุณหภูมิ Celsius ↔ Fahrenheit

**เฉลย:**
```kotlin
fun celsiusToFahrenheit(celsius: Double): Double = celsius * 9 / 5 + 32
fun fahrenheitToCelsius(fahrenheit: Double): Double = (fahrenheit - 32) * 5 / 9

fun main() {
    val celsius = 100.0
    val fahrenheit = celsiusToFahrenheit(celsius)
    println("$celsius°C = ${"%.1f".format(fahrenheit)}°F")
    // 100.0°C = 212.0°F
    
    val temp = 98.6
    val inCelsius = fahrenheitToCelsius(temp)
    println("$temp°F = ${"%.1f".format(inCelsius)}°C")
    // 98.6°F = 37.0°C
}
```

### ข้อ 5: BMI Calculator
คำนวณ BMI และบอกว่าอยู่ในระดับไหน

```kotlin
fun calculateBMI(weight: Double, heightCm: Double): Double {
    val heightM = heightCm / 100
    return weight / (heightM * heightM)
}

fun bmiCategory(bmi: Double): String = when {
    bmi < 18.5 -> "น้ำหนักน้อย (Underweight)"
    bmi < 25.0 -> "ปกติ (Normal)"
    bmi < 30.0 -> "น้ำหนักเกิน (Overweight)"
    else       -> "อ้วน (Obese)"
}

fun main() {
    print("น้ำหนัก (กก.): ")
    val weight = readLine()?.toDoubleOrNull() ?: return
    
    print("ส่วนสูง (ซม.): ")
    val height = readLine()?.toDoubleOrNull() ?: return
    
    val bmi = calculateBMI(weight, height)
    val category = bmiCategory(bmi)
    
    println("\n=== ผลการคำนวณ BMI ===")
    println("BMI: ${"%.2f".format(bmi)}")
    println("ระดับ: $category")
}
```

---

## 📝 สรุป Part 02

| แนวคิด | รายละเอียด |
|--------|-----------|
| `val` | Immutable - ใช้เป็น default |
| `var` | Mutable - ใช้เมื่อจำเป็น |
| `const val` | Compile-time constant |
| Int, Long | จำนวนเต็ม |
| Float, Double | ทศนิยม |
| Boolean | true/false |
| Char | ตัวอักษรเดี่ยว |
| String | ข้อความ |
| Nullable `?` | อนุญาต null |
| `?.` | Safe call |
| `?:` | Elvis operator |
| `as?` | Safe cast |

---

## ➡️ ถัดไป: Part 03 - Operators และ String Templates

ใน Part ถัดไปเราจะเรียนรู้:
- Arithmetic, Comparison, Logical operators
- Bitwise operations
- String Templates แบบละเอียด
- Operator overloading เบื้องต้น

---
*Part 02/100+ | Kotlin & Spring Boot Complete Course*
