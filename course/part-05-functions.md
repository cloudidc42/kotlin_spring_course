# Part 05: Functions และ Default Parameters
## Functions, Parameters & Functional Features

---

## 🎯 เป้าหมายของ Part นี้

- เขียน functions ทุกรูปแบบ
- ใช้ default parameters และ named arguments
- ใช้ varargs
- เขียน local functions
- เข้าใจ tail recursion
- เข้าใจ function types

---

## 📌 1. Function พื้นฐาน

```kotlin
// รูปแบบทั่วไป
fun functionName(param1: Type1, param2: Type2): ReturnType {
    // body
    return value
}

// ตัวอย่าง
fun greet(name: String): String {
    return "Hello, $name!"
}

fun add(a: Int, b: Int): Int {
    return a + b
}

// Unit function (ไม่คืนค่า - เหมือน void)
fun printMessage(message: String) {  // : Unit สามารถละได้
    println(message)
}

fun main() {
    println(greet("Alice"))  // Hello, Alice!
    println(add(5, 3))       // 8
    printMessage("Hello!")   // Hello!
}
```

### Single-Expression Function (กระชับมาก!)

```kotlin
// แบบปกติ
fun double(x: Int): Int {
    return x * 2
}

// แบบ single-expression (= แทน { return })
fun double(x: Int): Int = x * 2

// Type inference ได้เลย
fun double(x: Int) = x * 2

// ตัวอย่างมากขึ้น
fun square(n: Int) = n * n
fun isEven(n: Int) = n % 2 == 0
fun max(a: Int, b: Int) = if (a > b) a else b
fun greet(name: String) = "Hello, $name!"
fun celsiusToFahrenheit(c: Double) = c * 9 / 5 + 32

// ซับซ้อนขึ้น
fun getGrade(score: Int) = when {
    score >= 90 -> "A"
    score >= 80 -> "B"
    score >= 70 -> "C"
    score >= 60 -> "D"
    else -> "F"
}

println(getGrade(85))  // B
```

---

## 🔧 2. Default Parameters

```kotlin
// กำหนดค่า default
fun createProfile(
    name: String,
    age: Int = 0,
    city: String = "ไม่ระบุ",
    isStudent: Boolean = false
) {
    println("Name: $name, Age: $age, City: $city, Student: $isStudent")
}

// เรียกใช้หลายแบบ
createProfile("Alice")
// Name: Alice, Age: 0, City: ไม่ระบุ, Student: false

createProfile("Bob", 25)
// Name: Bob, Age: 25, City: ไม่ระบุ, Student: false

createProfile("Charlie", 20, "Bangkok", true)
// Name: Charlie, Age: 20, City: Bangkok, Student: true
```

### ตัวอย่างจริง

```kotlin
fun sendEmail(
    to: String,
    subject: String = "No Subject",
    body: String = "",
    cc: String = "",
    isHtml: Boolean = false
) {
    println("To: $to")
    println("Subject: $subject")
    if (cc.isNotEmpty()) println("CC: $cc")
    println("HTML: $isHtml")
    println("Body: ${body.take(50)}...")
    println()
}

sendEmail("alice@example.com")
sendEmail("bob@example.com", "Hello!")
sendEmail("charlie@example.com", "Newsletter", "Content here...", isHtml = true)
```

---

## 🏷️ 3. Named Arguments

```kotlin
fun createUser(
    name: String,
    email: String,
    age: Int,
    role: String = "user",
    isActive: Boolean = true
) {
    println("Creating user: $name ($email) age=$age role=$role active=$isActive")
}

// Named arguments - ลำดับไม่สำคัญ!
createUser(
    name = "Alice",
    email = "alice@example.com",
    age = 25
)

createUser(
    email = "bob@example.com",
    name = "Bob",         // สลับลำดับได้
    age = 30,
    role = "admin"
)

// Mix: positional + named
createUser("Charlie", "charlie@example.com", 22, isActive = false)
// ⚠️ ตำแหน่งก่อน named เสมอ
```

### Named arguments กับ Default

```kotlin
fun connectDB(
    host: String = "localhost",
    port: Int = 5432,
    database: String,
    username: String = "postgres",
    password: String = "",
    ssl: Boolean = false
) {
    println("Connecting to $host:$port/$database as $username (SSL: $ssl)")
}

// ข้าม parameter กลาง
connectDB(database = "myapp")
connectDB(host = "prod-server.com", database = "production", password = "secret")
connectDB(database = "dev", ssl = true)
```

---

## 📦 4. Varargs

```kotlin
// รับ argument ได้หลายตัว
fun sum(vararg numbers: Int): Int {
    var total = 0
    for (n in numbers) total += n
    return total
}

println(sum(1))                // 1
println(sum(1, 2, 3))          // 6
println(sum(1, 2, 3, 4, 5))    // 15
println(sum())                  // 0

// ส่ง array ด้วย spread operator (*)
val nums = intArrayOf(1, 2, 3, 4, 5)
println(sum(*nums))            // 15

// Varargs กับ parameter อื่น
fun formatMessage(prefix: String, vararg messages: String, suffix: String = ""): String {
    return "$prefix ${messages.joinToString(" ")} $suffix".trim()
}

println(formatMessage(">>>", "Hello", "World", "!"))
println(formatMessage("LOG:", "Error occurred", suffix = "[ERROR]"))
```

### ตัวอย่าง printf-like

```kotlin
fun printTable(header: String, vararg rows: Pair<String, Any>) {
    val width = 40
    println("=".repeat(width))
    println(header.padStart((width + header.length) / 2))
    println("=".repeat(width))
    for ((label, value) in rows) {
        println("%-20s: %s".format(label, value))
    }
    println("=".repeat(width))
}

printTable(
    "User Information",
    "Name" to "Alice Smith",
    "Age" to 25,
    "Email" to "alice@example.com",
    "City" to "Bangkok",
    "Status" to "Active"
)
```

---

## 🏠 5. Local Functions

```kotlin
// Function ภายใน function
fun processOrder(orderId: Int, items: List<String>): String {
    // local function - เข้าถึง outer variables ได้
    fun validate(): Boolean {
        return orderId > 0 && items.isNotEmpty()
    }
    
    fun formatItem(item: String): String {
        return "• $item"
    }
    
    if (!validate()) return "Invalid order"
    
    val formatted = items.map { formatItem(it) }.joinToString("\n")
    return "Order #$orderId:\n$formatted"
}

println(processOrder(1001, listOf("Kotlin Book", "Spring Course", "Java Book")))
// Order #1001:
// • Kotlin Book
// • Spring Course
// • Java Book

// Local function ที่ซับซ้อนกว่า
fun calculateTax(income: Double): Double {
    fun taxBracket(amount: Double, rate: Double, max: Double): Double {
        return minOf(amount, max) * rate
    }
    
    return when {
        income <= 150_000 -> 0.0
        income <= 300_000 -> taxBracket(income - 150_000, 0.05, 150_000)
        income <= 500_000 -> 7_500 + taxBracket(income - 300_000, 0.10, 200_000)
        income <= 750_000 -> 27_500 + taxBracket(income - 500_000, 0.15, 250_000)
        income <= 1_000_000 -> 65_000 + taxBracket(income - 750_000, 0.20, 250_000)
        else -> 115_000 + (income - 1_000_000) * 0.35
    }
}

println("Tax: %.2f".format(calculateTax(500_000.0)))  // Tax: 27500.00
println("Tax: %.2f".format(calculateTax(1_000_000.0))) // Tax: 115000.00
```

---

## 🔄 6. Recursion และ Tail Recursion

### Recursion ทั่วไป

```kotlin
// Factorial
fun factorial(n: Long): Long {
    if (n <= 1) return 1
    return n * factorial(n - 1)
}

println(factorial(5))   // 120
println(factorial(10))  // 3628800
println(factorial(20))  // 2432902008176640000

// Fibonacci
fun fibonacci(n: Int): Long {
    if (n <= 1) return n.toLong()
    return fibonacci(n - 1) + fibonacci(n - 2)
}

println(fibonacci(10))  // 55
```

### Tail Recursion (ป้องกัน Stack Overflow!)

```kotlin
// tailrec - Kotlin optimize เป็น loop อัตโนมัติ
tailrec fun factorial(n: Long, accumulator: Long = 1): Long {
    if (n <= 1) return accumulator
    return factorial(n - 1, n * accumulator)  // tail call
}

println(factorial(5))     // 120
println(factorial(100))   // จะ overflow ถ้าไม่ใช้ BigInteger แต่ไม่ stack overflow

// ตัวอย่างที่ดี: sum
tailrec fun sumTo(n: Long, acc: Long = 0): Long {
    if (n <= 0) return acc
    return sumTo(n - 1, acc + n)
}

println(sumTo(100))    // 5050
println(sumTo(10000))  // 50005000 (ทำงานได้ ไม่ stack overflow)

// กฎ: recursive call ต้องเป็น operation สุดท้าย
// ❌ ไม่ใช่ tail: return n * factorial(n-1)
// ✅ ใช่ tail: return factorial(n-1, n * acc)
```

---

## 🎯 7. Function Types และ Higher-Order Functions

```kotlin
// Function type: (ParamTypes) -> ReturnType
val add: (Int, Int) -> Int = { a, b -> a + b }
val greet: (String) -> String = { name -> "Hello, $name!" }
val printLine: (String) -> Unit = { println(it) }

println(add(3, 4))          // 7
println(greet("Alice"))     // Hello, Alice!
printLine("Hello World")    // Hello World

// Higher-order function รับ function เป็น parameter
fun operate(a: Int, b: Int, operation: (Int, Int) -> Int): Int {
    return operation(a, b)
}

println(operate(10, 5, { a, b -> a + b }))  // 15
println(operate(10, 5) { a, b -> a * b })   // 50 (trailing lambda)

// ส่ง function reference
fun multiply(a: Int, b: Int) = a * b
println(operate(10, 5, ::multiply))         // 50

// คืนค่าเป็น function
fun makeMultiplier(factor: Int): (Int) -> Int {
    return { number -> number * factor }
}

val double = makeMultiplier(2)
val triple = makeMultiplier(3)

println(double(5))   // 10
println(triple(5))   // 15
```

### Collection operations ใช้ higher-order functions

```kotlin
val numbers = listOf(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)

// filter - กรอง
val evens = numbers.filter { it % 2 == 0 }
println(evens)  // [2, 4, 6, 8, 10]

// map - แปลง
val doubled = numbers.map { it * 2 }
println(doubled)  // [2, 4, 6, 8, 10, 12, 14, 16, 18, 20]

// reduce - รวม
val sum = numbers.reduce { acc, n -> acc + n }
println(sum)  // 55

// fold - รวมพร้อม initial value
val product = numbers.fold(1) { acc, n -> acc * n }
println(product)  // 3628800

// forEach
numbers.filter { it > 5 }.forEach { println(it) }
```

---

## 🔀 8. Extension Functions

```kotlin
// เพิ่ม function ให้ existing type
fun String.isPalindrome(): Boolean {
    return this == this.reversed()
}

fun Int.factorial(): Long {
    var result = 1L
    for (i in 2..this) result *= i
    return result
}

fun Double.toBaht(): String = "฿${"%.2f".format(this)}"

// ใช้งาน
println("racecar".isPalindrome())  // true
println("hello".isPalindrome())    // false
println(5.factorial())             // 120
println(1234.56.toBaht())         // ฿1234.56

// Extension function ที่ซับซ้อน
fun String.titleCase(): String {
    return split(" ").joinToString(" ") { word ->
        word.replaceFirstChar { it.uppercase() }
    }
}

fun List<Int>.statistics(): Map<String, Double> {
    return mapOf(
        "sum" to sum().toDouble(),
        "average" to average(),
        "min" to min().toDouble(),
        "max" to max().toDouble()
    )
}

println("hello world kotlin".titleCase())  // Hello World Kotlin

val stats = listOf(5, 2, 8, 1, 9, 3, 7).statistics()
println(stats)
// {sum=35.0, average=5.0, min=1.0, max=9.0}
```

---

## 💎 9. Inline Functions

```kotlin
// inline - expand at call site (ประสิทธิภาพสูงขึ้น)
inline fun measure(block: () -> Unit): Long {
    val start = System.currentTimeMillis()
    block()
    return System.currentTimeMillis() - start
}

val time = measure {
    // code ที่ต้องการวัดเวลา
    Thread.sleep(100)
}
println("Time: ${time}ms")  // Time: ~100ms

// crossinline - ห้าม non-local return ใน lambda
inline fun runWithContext(crossinline block: () -> Unit) {
    Thread {
        block()
    }.start()
}

// noinline - ไม่ inline lambda นี้
inline fun process(
    noinline setup: () -> Unit,  // เก็บเป็น object
    execute: () -> Unit          // inline
) {
    setup()
    execute()
}
```

---

## 🏋️ 10. แบบฝึกหัด

### ข้อ 1: Calculator with operations
```kotlin
fun calculate(a: Double, b: Double, op: String): Double? = when (op) {
    "+" -> a + b
    "-" -> a - b
    "*" -> a * b
    "/" -> if (b != 0.0) a / b else null
    "%" -> a % b
    "^" -> Math.pow(a, b)
    else -> null
}

fun main() {
    val tests = listOf(
        Triple(10.0, 3.0, "+"),
        Triple(10.0, 3.0, "/"),
        Triple(10.0, 0.0, "/"),
        Triple(2.0, 8.0, "^")
    )
    
    for ((a, b, op) in tests) {
        val result = calculate(a, b, op)
        println("$a $op $b = ${result ?: "Error"}")
    }
}
```

### ข้อ 2: String utilities extension
```kotlin
fun String.wordCount(): Int = if (isBlank()) 0 else trim().split(Regex("\\s+")).size
fun String.truncate(maxLength: Int, suffix: String = "...") =
    if (length <= maxLength) this
    else take(maxLength - suffix.length) + suffix

fun main() {
    println("Hello World Kotlin".wordCount())  // 3
    println("".wordCount())                     // 0
    println("Hello, World!".truncate(10))       // Hello, ...
    println("Hi".truncate(10))                  // Hi
}
```

### ข้อ 3: Number utilities
```kotlin
fun Int.isPrime(): Boolean {
    if (this < 2) return false
    if (this == 2) return true
    if (this % 2 == 0) return false
    for (i in 3..Math.sqrt(toDouble()).toInt() step 2) {
        if (this % i == 0) return false
    }
    return true
}

fun Int.primeFactors(): List<Int> {
    val factors = mutableListOf<Int>()
    var n = this
    var d = 2
    while (n > 1) {
        while (n % d == 0) {
            factors.add(d)
            n /= d
        }
        d++
    }
    return factors
}

fun main() {
    println(17.isPrime())         // true
    println(100.isPrime())        // false
    println(360.primeFactors())   // [2, 2, 2, 3, 3, 5]
    println(2310.primeFactors())  // [2, 3, 5, 7, 11]
}
```

### ข้อ 4: Higher-order functions
```kotlin
fun applyTwice(f: (Int) -> Int, x: Int): Int = f(f(x))
fun compose(f: (Int) -> Int, g: (Int) -> Int): (Int) -> Int = { f(g(it)) }
fun repeat(n: Int, action: (Int) -> Unit) = (1..n).forEach(action)

fun main() {
    println(applyTwice({ it * 2 }, 3))      // 12 (3*2=6, 6*2=12)
    
    val addOne = { x: Int -> x + 1 }
    val timesThree = { x: Int -> x * 3 }
    val addOneThenTimesThree = compose(timesThree, addOne)
    
    println(addOneThenTimesThree(4))  // 15 ((4+1)*3)
    
    repeat(3) { i -> println("Line $i") }
    // Line 1
    // Line 2
    // Line 3
}
```

---

## 📝 สรุป Part 05

| แนวคิด | ตัวอย่าง |
|--------|---------|
| Regular function | `fun add(a: Int, b: Int): Int` |
| Single-expression | `fun add(a: Int, b: Int) = a + b` |
| Default params | `fun f(x: Int = 0)` |
| Named args | `f(x = 5)` |
| Varargs | `fun f(vararg args: String)` |
| Local function | function ภายใน function |
| Tail recursion | `tailrec fun f(...)` |
| Extension function | `fun String.myFun()` |
| Higher-order | รับ/คืน function |

---

## ➡️ ถัดไป: Part 06 - Classes และ Objects

ใน Part ถัดไปเราจะเรียนรู้:
- Class definition และ constructors
- Properties และ getters/setters
- Companion objects
- Object declarations (Singleton)
- Visibility modifiers

---
*Part 05/100+ | Kotlin & Spring Boot Complete Course*
