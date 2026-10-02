# Part 03: Operators และ String Templates
## Operators & String Templates

---

## 🎯 เป้าหมายของ Part นี้

- เข้าใจ operators ทุกประเภทใน Kotlin
- ใช้ String Templates อย่างเชี่ยวชาญ
- เข้าใจ operator precedence
- รู้จัก Bitwise operations
- เข้าใจ operator overloading เบื้องต้น

---

## ➕ 1. Arithmetic Operators (ตัวดำเนินการทางคณิตศาสตร์)

```kotlin
val a = 10
val b = 3

println(a + b)    // 13 (บวก)
println(a - b)    // 7  (ลบ)
println(a * b)    // 30 (คูณ)
println(a / b)    // 3  (หาร - ผลลัพธ์เป็น Int เพราะ a,b เป็น Int)
println(a % b)    // 1  (modulo - เศษ)

// Double division
println(a.toDouble() / b)  // 3.3333333333333335
println(10.0 / 3)          // 3.3333333333333335
println(10 / 3.0)          // 3.3333333333333335

// Integer division vs floating point
println(7 / 2)    // 3 (Integer - ปัดเศษทิ้ง)
println(7.0 / 2)  // 3.5 (Double)
println(7 / 2.0)  // 3.5 (Double)

// Modulo with negative numbers
println(-7 % 3)   // -1 (sign follows dividend)
println(7 % -3)   // 1  (sign follows dividend)
```

### Augmented Assignment Operators

```kotlin
var x = 10

x += 5    // x = x + 5 → 15
x -= 3    // x = x - 3 → 12
x *= 2    // x = x * 2 → 24
x /= 4    // x = x / 4 → 6
x %= 4    // x = x % 4 → 2

println(x)  // 2
```

### Increment/Decrement

```kotlin
var count = 5

// Prefix - เพิ่มก่อน ใช้ทีหลัง
println(++count)  // 6 (count เป็น 6)
println(--count)  // 5 (count เป็น 5)

// Postfix - ใช้ก่อน เพิ่มทีหลัง
println(count++)  // 5 (count เป็น 5 ก่อน แล้วค่อยเป็น 6)
println(count--)  // 6 (count เป็น 6 ก่อน แล้วค่อยเป็น 5)
println(count)    // 5

// ตัวอย่างความแตกต่าง
var a = 5
var b = a++   // b = 5, a = 6
var c = ++a   // a = 7, c = 7
println("a=$a, b=$b, c=$c")  // a=7, b=5, c=7
```

---

## 🔍 2. Comparison Operators (ตัวเปรียบเทียบ)

```kotlin
val a = 5
val b = 10

println(a == b)   // false (เท่ากัน?)
println(a != b)   // true  (ไม่เท่ากัน?)
println(a < b)    // true  (น้อยกว่า?)
println(a > b)    // false (มากกว่า?)
println(a <= b)   // true  (น้อยกว่าหรือเท่ากัน?)
println(a >= b)   // false (มากกว่าหรือเท่ากัน?)

// String comparison
val str1 = "Hello"
val str2 = "Hello"
val str3 = "World"

println(str1 == str2)     // true (เปรียบเทียบ content)
println(str1 === str2)    // true/false (เปรียบเทียบ reference - ขึ้นอยู่กับ JVM)
println(str1.equals(str2))  // true
println(str1 < str3)      // true (alphabetical order)
println(str1.compareTo(str2))  // 0 (เท่ากัน)
println(str1.compareTo(str3))  // ค่าลบ (str1 มาก่อน str3)

// Structural equality vs referential equality
data class Point(val x: Int, val y: Int)

val p1 = Point(1, 2)
val p2 = Point(1, 2)
val p3 = p1

println(p1 == p2)   // true (structural - content เหมือนกัน)
println(p1 === p2)  // false (referential - คนละ object)
println(p1 === p3)  // true (referential - object เดียวกัน)
```

---

## 🔀 3. Logical Operators (ตัวดำเนินการตรรกะ)

```kotlin
val isAdult = true
val hasId = false
val hasTicket = true

// AND (&&) - ทั้งคู่ต้องจริง
println(isAdult && hasId)        // false
println(isAdult && hasTicket)    // true

// OR (||) - อย่างน้อยหนึ่งอันต้องจริง
println(isAdult || hasId)        // true
println(hasId || false)          // false

// NOT (!) - กลับค่า
println(!isAdult)                // false
println(!hasId)                  // true

// Short-circuit evaluation (ประเมินแบบลัดวงจร)
var x = 0
val result = (x++ > 0) && (x++ > 0)  // ประเมินแค่ครั้งแรก
println(x)  // 1 (ไม่ใช่ 2 เพราะ && short-circuit)

var y = 0
val result2 = (y++ > 0) || (y++ > 0)  // ประเมินทั้งคู่
println(y)  // 2 (เพราะ || ต้องดูทั้งคู่ถ้าตัวแรก false)

// Complex conditions
val age = 20
val income = 50000
val hasJob = true

val canGetLoan = (age >= 18 && age <= 65) && 
                 (income >= 30000 || hasJob)
println("Can get loan: $canGetLoan")  // true
```

---

## 🔢 4. Bitwise Operations

```kotlin
val a = 0b1010  // 10 in decimal
val b = 0b1100  // 12 in decimal

// Bitwise AND (and)
println(a and b)          // 8  (0b1000)
println(a.and(b))         // 8  (method form)

// Bitwise OR (or)
println(a or b)           // 14 (0b1110)

// Bitwise XOR (xor)
println(a xor b)          // 6  (0b0110)

// Bitwise NOT (inv)
println(a.inv())          // -11 (inverts all bits)

// Shift left (shl) - คูณด้วย 2^n
println(1 shl 0)          // 1
println(1 shl 1)          // 2
println(1 shl 2)          // 4
println(1 shl 3)          // 8
println(5 shl 2)          // 20 (5 * 4)

// Shift right (shr) - หารด้วย 2^n
println(16 shr 1)         // 8
println(16 shr 2)         // 4
println(100 shr 2)        // 25

// Unsigned shift right (ushr)
println(-1 shr 1)         // -1 (sign preserved)
println(-1 ushr 1)        // 2147483647 (sign not preserved)

// ตัวอย่างการใช้งานจริง
// Permissions system (Unix-style)
val READ    = 0b100  // 4
val WRITE   = 0b010  // 2
val EXECUTE = 0b001  // 1

val userPerms = READ or WRITE   // 6 (rw-)
val hasRead    = userPerms and READ    != 0  // true
val hasExecute = userPerms and EXECUTE != 0  // false

println("Has read: $hasRead")
println("Has execute: $hasExecute")

// Flag operations
fun addPermission(perms: Int, newPerm: Int) = perms or newPerm
fun removePermission(perms: Int, perm: Int) = perms and perm.inv()
fun hasPermission(perms: Int, perm: Int) = (perms and perm) != 0

var permissions = READ or WRITE    // rw-
permissions = addPermission(permissions, EXECUTE)   // rwx
permissions = removePermission(permissions, WRITE)  // r-x
println("Can execute: ${hasPermission(permissions, EXECUTE)}")  // true
println("Can write: ${hasPermission(permissions, WRITE)}")      // false
```

---

## 📝 5. String Templates (เทมเพลตสตริง)

### พื้นฐาน

```kotlin
val name = "Alice"
val age = 25
val city = "Bangkok"

// Simple variable
println("Hello, $name!")         // Hello, Alice!

// Expression in braces
println("Age next year: ${age + 1}")  // Age next year: 26

// Property/method call
println("Name length: ${name.length}")  // Name length: 5
println("Upper: ${name.uppercase()}")   // Upper: ALICE

// Nested template
val items = listOf("apple", "banana", "cherry")
println("First item: ${items[0]}")  // First item: apple
println("Items: ${items.joinToString()}")  // Items: apple, banana, cherry
```

### Raw Strings (Triple-quoted)

```kotlin
// Raw string - ไม่ต้องใช้ escape characters
val poem = """
    Roses are red,
    Violets are blue,
    Kotlin is awesome,
    And so are you!
""".trimIndent()

println(poem)

// Raw string กับ template
val firstName = "Alice"
val lastName = "Smith"

val profile = """
    ============ User Profile ============
    Name: $firstName $lastName
    Email: ${firstName.lowercase()}@example.com
    Member since: 2026
    ======================================
""".trimIndent()

println(profile)

// Raw string กับ JSON (ไม่ต้อง escape quotes)
val json = """
    {
        "name": "$firstName",
        "age": $age,
        "active": true
    }
""".trimIndent()

println(json)
```

### Format Strings

```kotlin
val pi = 3.14159265358979
val price = 1234567.89
val percentage = 0.1234

// ทศนิยม n ตำแหน่ง
println("Pi: ${"%.2f".format(pi)}")     // Pi: 3.14
println("Pi: ${"%.4f".format(pi)}")     // Pi: 3.1416

// ความกว้างของตัวเลข
println("%10d".format(42))              // "        42"
println("%-10d|".format(42))            // "42        |"

// เลขฐาน 16 (hex)
println("%x".format(255))              // ff
println("%X".format(255))              // FF
println("%08X".format(255))            // 000000FF

// Scientific notation
println("%e".format(1234567.89))       // 1.234568e+06
println("%E".format(1234567.89))       // 1.234568E+06

// Percentage
println("%.1f%%".format(percentage * 100))  // 12.3%

// String width
println("%20s".format("Hello"))        // "               Hello"
println("%-20s|".format("Hello"))      // "Hello               |"

// ตัวอย่างตาราง
println("%-20s %10s %10s".format("Product", "Price", "Qty"))
println("%-20s %10s %10s".format("-".repeat(20), "-".repeat(10), "-".repeat(10)))
println("%-20s %10.2f %10d".format("Kotlin Book", 599.0, 5))
println("%-20s %10.2f %10d".format("Spring Course", 1200.0, 3))
```

**ผลลัพธ์:**
```
Product              Price        Qty
-------------------- ---------- ----------
Kotlin Book             599.00          5
Spring Course          1200.00          3
```

---

## 🎭 6. Operator Overloading

```kotlin
data class Vector(val x: Double, val y: Double) {
    // + operator
    operator fun plus(other: Vector) = Vector(x + other.x, y + other.y)
    
    // - operator
    operator fun minus(other: Vector) = Vector(x - other.x, y - other.y)
    
    // * operator (scalar multiplication)
    operator fun times(scalar: Double) = Vector(x * scalar, y * scalar)
    
    // unary minus
    operator fun unaryMinus() = Vector(-x, -y)
    
    // comparison
    operator fun compareTo(other: Vector): Int {
        val mag1 = Math.sqrt(x * x + y * y)
        val mag2 = Math.sqrt(other.x * other.x + other.y * other.y)
        return mag1.compareTo(mag2)
    }
    
    override fun toString() = "Vector($x, $y)"
}

fun main() {
    val v1 = Vector(1.0, 2.0)
    val v2 = Vector(3.0, 4.0)
    
    println(v1 + v2)     // Vector(4.0, 6.0)
    println(v2 - v1)     // Vector(2.0, 2.0)
    println(v1 * 2.0)    // Vector(2.0, 4.0)
    println(-v1)         // Vector(-1.0, -2.0)
    println(v1 < v2)     // true
}
```

### operator fun invoke

```kotlin
class Multiplier(private val factor: Int) {
    operator fun invoke(value: Int) = value * factor
}

val double = Multiplier(2)
val triple = Multiplier(3)

println(double(5))    // 10
println(triple(4))    // 12
println(double(triple(3)))  // 18
```

### Comparable interface

```kotlin
data class Temperature(val degrees: Double, val unit: Char = 'C') : Comparable<Temperature> {
    fun toCelsius(): Double = when (unit) {
        'F' -> (degrees - 32) * 5 / 9
        'K' -> degrees - 273.15
        else -> degrees
    }
    
    override fun compareTo(other: Temperature): Int {
        return this.toCelsius().compareTo(other.toCelsius())
    }
    
    override fun toString() = "$degrees°$unit"
}

fun main() {
    val temps = listOf(
        Temperature(100.0, 'C'),
        Temperature(212.0, 'F'),
        Temperature(373.15, 'K'),
        Temperature(0.0, 'C'),
        Temperature(37.0, 'C')
    )
    
    println(temps.sorted())
    // [0.0°C, 37.0°C, 100.0°C, 212.0°F, 373.15°K]
    
    println(temps.max())   // 100.0°C (or equivalent)
    println(temps.min())   // 0.0°C
}
```

---

## ⚡ 7. Operator Precedence (ลำดับความสำคัญ)

```kotlin
// จากสูงไปต่ำ:
// 1. Postfix: a++, a--, a.b, a[i], a()
// 2. Prefix: -a, +a, ++a, --a, !a
// 3. Type RHS: :, as, as?
// 4. Multiplication: *, /, %
// 5. Addition: +, -
// 6. Range: ..
// 7. Infix functions
// 8. Elvis: ?:
// 9. Named checks: in, !in, is, !is
// 10. Comparison: <, >, <=, >=
// 11. Equality: ==, !=
// 12. Conjunction: &&
// 13. Disjunction: ||
// 14. Spread: *
// 15. Assignment: =, +=, -=, *=, /=, %=

// ตัวอย่าง
println(2 + 3 * 4)         // 14 (ไม่ใช่ 20)
println((2 + 3) * 4)       // 20
println(10 - 3 + 2)        // 9  (ซ้าย → ขวา)
println(2.0.pow(3.0).pow(2.0))  // 512 (ขวา → ซ้าย สำหรับ power)

// Logical precedence
val a = true
val b = false
val c = true

println(a || b && c)        // true (AND ก่อน OR)
println((a || b) && c)      // true

println(!a || b)            // false (NOT ก่อน OR)
println(!(a || b))          // false
```

---

## 🔗 8. in, is, !in, !is operators

```kotlin
// in - ตรวจสอบว่าอยู่ใน range หรือ collection
val score = 75
println(score in 0..100)         // true (อยู่ใน range)
println(score in 70..79)         // true (เกรด C)
println(score !in 90..100)       // true (ไม่ได้อยู่ในช่วง A)

val vowels = setOf('a', 'e', 'i', 'o', 'u')
println('e' in vowels)           // true
println('z' !in vowels)          // true

val names = listOf("Alice", "Bob", "Charlie")
println("Bob" in names)          // true
println("David" !in names)       // true

// is - ตรวจสอบ type
val value: Any = "Hello, Kotlin!"

println(value is String)         // true
println(value is Int)            // false
println(value !is Int)           // true

// Smart cast หลัง is check
if (value is String) {
    println(value.length)        // ✅ Smart cast เป็น String
    println(value.uppercase())   // ✅
}

// ใน when
fun typeCheck(obj: Any): String = when (obj) {
    is Int    -> "Int: $obj"
    is String -> "String of length ${obj.length}"  // Smart cast!
    is List<*>-> "List with ${obj.size} items"
    is Boolean-> "Boolean: $obj"
    else      -> "Unknown type"
}

println(typeCheck(42))
println(typeCheck("Hello"))
println(typeCheck(listOf(1,2,3)))
```

---

## 🌊 9. Range Operators

```kotlin
// ช่วง (Range)
val range1 = 1..10        // 1 ถึง 10 (inclusive)
val range2 = 1 until 10   // 1 ถึง 9 (exclusive end)
val range3 = 10 downTo 1  // 10 ลงมา 1
val range4 = 1..10 step 2 // 1, 3, 5, 7, 9

for (i in range1) print("$i ")  // 1 2 3 4 5 6 7 8 9 10
println()
for (i in range2) print("$i ")  // 1 2 3 4 5 6 7 8 9
println()
for (i in range3) print("$i ")  // 10 9 8 7 6 5 4 3 2 1
println()
for (i in range4) print("$i ")  // 1 3 5 7 9
println()

// Char range
for (c in 'a'..'z') print(c)    // abcdefghijklmnopqrstuvwxyz
println()
for (c in 'A'..'Z' step 2) print(c)  // ACEGIKMOQSUWY
println()

// String range (alphabetical)
val letters = 'a'..'z'
println('c' in letters)  // true
println('1' in letters)  // false

// Range operations
val nums = 1..100
println(nums.count())    // 100
println(nums.sum())      // 5050
println(nums.average())  // 50.5
println(nums.first)      // 1
println(nums.last)       // 100
println(nums.contains(50))  // true
```

---

## 📊 10. Infix Functions

```kotlin
// สร้าง infix function
infix fun Int.plus_custom(other: Int): Int = this + other + 100

println(5 plus_custom 3)  // 108

// Built-in infix functions
println(1 to "one")          // (1, one)
println(listOf(1,2,3) zip listOf('a','b','c'))  // [(1, a), (2, b), (3, c)]

// สร้าง DSL-like code ด้วย infix
class Person(val name: String) {
    var salary = 0
    
    infix fun earns(amount: Int): Person {
        salary = amount
        return this
    }
    
    override fun toString() = "$name earns $salary THB/month"
}

val alice = Person("Alice") earns 100_000
val bob = Person("Bob") earns 80_000
println(alice)  // Alice earns 100000 THB/month
println(bob)    // Bob earns 80000 THB/month

// Bitwise infix
val flags = 0b1010 or 0b0101  // 0b1111 = 15
println(flags)  // 15
```

---

## 🏗️ 11. ตัวอย่างโปรแกรมจริง

### Calculator ครบฟีเจอร์

```kotlin
import kotlin.math.*

fun calculate(a: Double, operator: String, b: Double): String {
    return when (operator) {
        "+"  -> "${a + b}"
        "-"  -> "${a - b}"
        "*"  -> "${a * b}"
        "/"  -> if (b != 0.0) "${a / b}" else "Error: Division by zero"
        "%"  -> "${a % b}"
        "^"  -> "${a.pow(b)}"
        "log"-> if (a > 0 && b > 0 && b != 1.0) "${log(b, a)}" else "Error: Invalid log"
        else -> "Unknown operator"
    }
}

fun main() {
    val calculations = listOf(
        Triple(10.0, "+", 5.0),
        Triple(10.0, "-", 5.0),
        Triple(10.0, "*", 5.0),
        Triple(10.0, "/", 5.0),
        Triple(10.0, "/", 0.0),
        Triple(2.0, "^", 10.0),
        Triple(10.0, "log", 100.0)
    )
    
    println("=".repeat(40))
    println("%-15s %-5s %-15s = %s".format("A", "Op", "B", "Result"))
    println("=".repeat(40))
    
    for ((a, op, b) in calculations) {
        val result = calculate(a, op, b)
        println("%-15s %-5s %-15s = %s".format(a, op, b, result))
    }
}
```

### Grade Evaluator

```kotlin
fun getGrade(score: Int): Pair<String, String> {
    return when (score) {
        in 90..100 -> "A" to "ดีมาก"
        in 80 until 90 -> "B" to "ดี"
        in 70 until 80 -> "C" to "พอใช้"
        in 60 until 70 -> "D" to "ต้องปรับปรุง"
        in 0 until 60  -> "F" to "สอบตก"
        else -> "?" to "คะแนนไม่ถูกต้อง"
    }
}

fun main() {
    val scores = listOf(95, 85, 75, 65, 45, 101, -5)
    
    println("%-10s %-10s %s".format("คะแนน", "เกรด", "ความหมาย"))
    println("-".repeat(30))
    
    for (score in scores) {
        val (grade, meaning) = getGrade(score)
        println("%-10d %-10s %s".format(score, grade, meaning))
    }
}
```

**ผลลัพธ์:**
```
คะแนน      เกรด       ความหมาย
------------------------------
95         A          ดีมาก
85         B          ดี
75         C          พอใช้
65         D          ต้องปรับปรุง
45         F          สอบตก
101        ?          คะแนนไม่ถูกต้อง
-5         ?          คะแนนไม่ถูกต้อง
```

---

## 🏋️ 12. แบบฝึกหัด

### ข้อ 1: Swap values โดยไม่ใช้ temp variable
```kotlin
fun main() {
    var a = 5
    var b = 10
    
    // วิธีที่ 1: ใช้ arithmetic
    a = a + b
    b = a - b
    a = a - b
    println("a=$a, b=$b")  // a=10, b=5
    
    // วิธีที่ 2: ใช้ destructuring (Kotlin way)
    var x = 5
    var y = 10
    val temp = x.also { x = y; y = it }
    println("x=$x, y=$y")  // x=10, y=5
    
    // วิธีที่ 3: Kotlin idiomatic
    var p = 5
    var q = 10
    p = q.also { q = p }
    println("p=$p, q=$q")  // p=10, q=5
}
```

### ข้อ 2: Check leap year
```kotlin
fun isLeapYear(year: Int): Boolean {
    return (year % 4 == 0 && year % 100 != 0) || (year % 400 == 0)
}

fun main() {
    for (year in listOf(2000, 1900, 2024, 2025, 2026, 2100)) {
        val status = if (isLeapYear(year)) "ปีอธิกสุรทิน" else "ปีปกติ"
        println("$year: $status")
    }
}
```

### ข้อ 3: String analysis
```kotlin
fun analyzeString(text: String) {
    println("ข้อความ: \"$text\"")
    println("ความยาว: ${text.length}")
    println("ตัวพิมพ์ใหญ่: ${text.count { it.isUpperCase() }}")
    println("ตัวพิมพ์เล็ก: ${text.count { it.isLowerCase() }}")
    println("ตัวเลข: ${text.count { it.isDigit() }}")
    println("ช่องว่าง: ${text.count { it.isWhitespace() }}")
    println("พยัญชนะ: ${text.count { it in "bcdfghjklmnpqrstvwxyzBCDFGHJKLMNPQRSTVWXYZ" }}")
    println("สระ: ${text.count { it in "aeiouAEIOU" }}")
    println()
}

fun main() {
    analyzeString("Hello, World 2026!")
    analyzeString("Kotlin is AWESOME")
}
```

---

## 📝 สรุป Part 03

| Operator | ตัวอย่าง | ความหมาย |
|----------|---------|---------|
| `+`, `-`, `*`, `/`, `%` | `5 % 2 = 1` | คณิตศาสตร์ |
| `++`, `--` | `x++` | เพิ่ม/ลด 1 |
| `==`, `!=` | `a == b` | เปรียบเทียบค่า |
| `===`, `!==` | `a === b` | เปรียบเทียบ reference |
| `<`, `>`, `<=`, `>=` | `a < b` | เปรียบเทียบขนาด |
| `&&`, `\|\|`, `!` | `a && b` | ตรรกะ |
| `and`, `or`, `xor` | `a and b` | bitwise |
| `shl`, `shr` | `1 shl 3` | shift |
| `in`, `!in` | `x in 1..10` | ตรวจสอบ range |
| `is`, `!is` | `x is String` | ตรวจสอบ type |
| `..` | `1..10` | range |
| `?:` | `x ?: 0` | elvis |

---

## ➡️ ถัดไป: Part 04 - Control Flow

ใน Part ถัดไปจะเรียนรู้:
- `if`, `else if`, `else`
- `when` expression (Kotlin's switch)
- `for`, `while`, `do-while` loops
- `break`, `continue`, `return`
- Labeled loops

---
*Part 03/100+ | Kotlin & Spring Boot Complete Course*
