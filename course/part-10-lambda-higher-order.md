# Part 10: Lambda และ Higher-Order Functions
## Lambda, Function Types, Closures และ Functional Programming

---

## 🎯 เป้าหมายของ Part นี้

- เข้าใจ lambda syntax ทุกรูปแบบ
- ใช้ function types ได้อย่างถูกต้อง
- เข้าใจ closures และ variable capture
- รู้จัก scope functions: let, run, with, apply, also
- สร้างและใช้ higher-order functions จริงๆ
- เขียน data processing pipeline ด้วย functional programming

---

## λ 1. Lambda Syntax

Lambda คือ anonymous function (ฟังก์ชันที่ไม่มีชื่อ) ที่เขียนได้กระชับ

### 1.1 รูปแบบพื้นฐาน

```kotlin
fun main() {
    // Lambda syntax: { parameters -> body }
    val greet = { name: String -> "Hello, $name!" }
    println(greet("Alice"))  // Hello, Alice!
    
    // Lambda ที่มีหลาย parameters
    val add = { a: Int, b: Int -> a + b }
    println(add(3, 5))  // 8
    
    // Lambda ไม่มี parameter
    val sayHello = { println("Hello!") }
    sayHello()  // Hello!
    
    // Lambda ที่มีหลาย statements (ค่าสุดท้ายคือค่าที่คืน)
    val processNumber = { x: Int ->
        val doubled = x * 2
        val result = doubled + 1
        result  // implicit return
    }
    println(processNumber(5))  // 11
    
    // Lambda ที่ไม่มีค่าคืน (Unit)
    val printSquare = { x: Int -> println("$x^2 = ${x * x}") }
    printSquare(4)  // 4^2 = 16
}
```

### 1.2 Trailing Lambda Syntax

```kotlin
fun applyTwice(n: Int, operation: (Int) -> Int): Int {
    return operation(operation(n))
}

fun withLogging(message: String, block: () -> Unit) {
    println(">>> $message")
    block()
    println("<<< Done")
}

fun buildString(builder: StringBuilder.() -> Unit): String {
    val sb = StringBuilder()
    sb.builder()
    return sb.toString()
}

fun main() {
    // เรียกปกติ
    val result1 = applyTwice(3, { x -> x * 2 })
    println(result1)  // 12
    
    // Trailing lambda - ถ้า lambda เป็น parameter สุดท้าย ย้ายออกนอก ()
    val result2 = applyTwice(3) { x -> x * 2 }
    println(result2)  // 12
    
    // ถ้ามีแค่ lambda ไม่ต้องมี () เลย
    withLogging("Processing data") {
        println("Doing work...")
    }
    
    // it - ชื่อ implicit parameter เมื่อมี parameter เดียว
    val numbers = listOf(1, 2, 3, 4, 5)
    val doubled = numbers.map { it * 2 }
    println(doubled)  // [2, 4, 6, 8, 10]
    
    // filter + map chain
    val result = numbers
        .filter { it > 2 }
        .map { it * it }
    println(result)  // [9, 16, 25]
    
    // Lambda receiver (extension lambda)
    val message = buildString {
        append("Hello")
        append(", ")
        append("World")
        append("!")
    }
    println(message)  // Hello, World!
}
```

### 1.3 it vs Named Parameters

```kotlin
fun main() {
    val numbers = listOf(1, 2, 3, 4, 5)
    
    // ใช้ it (implicit parameter - parameter เดียว)
    val doubled = numbers.map { it * 2 }
    
    // ตั้งชื่อ parameter (ชัดเจนกว่า)
    val tripled = numbers.map { number -> number * 3 }
    
    // เมื่อมี lambda ซ้อนกัน ควรตั้งชื่อ (it จะสับสน)
    val matrix = listOf(listOf(1, 2, 3), listOf(4, 5, 6))
    
    // ไม่ดี - it ซ้อนกัน สับสน
    val flat1 = matrix.flatMap { it.map { it * 2 } }
    
    // ดีกว่า - ตั้งชื่อชัดเจน
    val flat2 = matrix.flatMap { row -> row.map { cell -> cell * 2 } }
    
    println(flat1)  // [2, 4, 6, 8, 10, 12]
    println(flat2)  // [2, 4, 6, 8, 10, 12]
    
    // _ สำหรับ parameter ที่ไม่ใช้
    val pairs = listOf(Pair("a", 1), Pair("b", 2), Pair("c", 3))
    val values = pairs.map { (_, value) -> value }
    println(values)  // [1, 2, 3]
}
```

---

## 📐 2. Function Types

Function type คือ type ที่แสดงลายเซ็นของฟังก์ชัน

### 2.1 Function Type พื้นฐาน

```kotlin
fun main() {
    // (ParameterTypes) -> ReturnType
    
    // ฟังก์ชันที่รับ Int คืน String
    val intToString: (Int) -> String = { it.toString() }
    println(intToString(42))  // 42
    
    // ฟังก์ชันที่รับ Int, String คืน Boolean
    val check: (Int, String) -> Boolean = { num, str ->
        str.length == num
    }
    println(check(5, "hello"))  // true
    println(check(3, "hello"))  // false
    
    // ฟังก์ชันที่ไม่มี parameter คืน Unit
    val doSomething: () -> Unit = { println("Doing...") }
    doSomething()
    
    // ฟังก์ชันที่ไม่มี parameter คืนค่า
    val getNumber: () -> Int = { 42 }
    println(getNumber())  // 42
    
    // Nullable function type
    val maybeFunction: ((Int) -> String)? = null
    val result = maybeFunction?.invoke(5) ?: "no function"
    println(result)  // no function
    
    // invoke() - เรียก function type อย่างชัดเจน
    val multiply: (Int, Int) -> Int = { a, b -> a * b }
    println(multiply(3, 4))        // 12
    println(multiply.invoke(3, 4)) // 12 (เหมือนกัน)
}
```

### 2.2 Function References

```kotlin
fun double(x: Int) = x * 2
fun isEven(x: Int) = x % 2 == 0
fun String.isPalindrome() = this == this.reversed()

class Calculator {
    fun add(a: Int, b: Int) = a + b
    fun multiply(a: Int, b: Int) = a * b
}

fun main() {
    val numbers = listOf(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)
    
    // :: - function reference (ไม่ต้องเขียน lambda)
    val doubled = numbers.map(::double)
    println(doubled)  // [2, 4, 6, 8, 10, 12, 14, 16, 18, 20]
    
    val evens = numbers.filter(::isEven)
    println(evens)  // [2, 4, 6, 8, 10]
    
    // Constructor reference
    val stringNumbers = numbers.map(::String)
    // หรือ numbers.map { it.toString() }
    
    // Member function reference
    val calc = Calculator()
    val adder: (Int, Int) -> Int = calc::add
    println(adder(3, 5))  // 8
    
    // Extension function reference
    val words = listOf("racecar", "hello", "level", "world")
    val palindromes = words.filter(String::isPalindrome)
    println(palindromes)  // [racecar, level]
    
    // Property reference
    data class Person(val name: String, val age: Int)
    val people = listOf(Person("Alice", 30), Person("Bob", 25), Person("Charlie", 35))
    
    val names = people.map(Person::name)
    println(names)  // [Alice, Bob, Charlie]
    
    val sortedByAge = people.sortedBy(Person::age)
    println(sortedByAge)
    
    // println reference
    numbers.forEach(::println)
}
```

### 2.3 Function Type กับ Generics

```kotlin
// Generic function type
fun <T, R> transform(value: T, transformer: (T) -> R): R {
    return transformer(value)
}

fun <T> applyAll(value: T, vararg operations: (T) -> T): T {
    return operations.fold(value) { acc, op -> op(acc) }
}

fun <T, R> List<T>.customMap(transform: (T) -> R): List<R> {
    val result = mutableListOf<R>()
    for (item in this) {
        result.add(transform(item))
    }
    return result
}

fun main() {
    val num = transform(42) { it * 2 }
    println(num)  // 84
    
    val str = transform("hello") { it.uppercase() }
    println(str)  // HELLO
    
    val result = applyAll(
        5,
        { it * 2 },
        { it + 3 },
        { it * it }
    )
    println(result)  // ((5*2)+3)^2 = 169
    
    val numbers = listOf(1, 2, 3, 4, 5)
    val squared = numbers.customMap { it * it }
    println(squared)  // [1, 4, 9, 16, 25]
}
```

---

## 🔒 3. Closures

Closure คือ lambda ที่ "จับ" (capture) ตัวแปรจาก scope รอบนอก

### 3.1 Variable Capture

```kotlin
fun main() {
    // Capture immutable variable
    val prefix = "Hello"
    val greet = { name: String -> "$prefix, $name!" }
    println(greet("Alice"))  // Hello, Alice!
    println(greet("Bob"))    // Hello, Bob!
    
    // Capture mutable variable
    var counter = 0
    val increment = { counter++ }
    val getCount = { counter }
    
    increment()
    increment()
    increment()
    println(getCount())  // 3
    println(counter)     // 3 - ตัวแปรนอกถูกเปลี่ยนด้วย!
    
    // Closure ใน loop
    val adders = mutableListOf<(Int) -> Int>()
    for (i in 1..3) {
        adders.add { x -> x + i }  // แต่ละ closure capture ค่า i ที่ต่างกัน
    }
    println(adders[0](10))  // 11
    println(adders[1](10))  // 12
    println(adders[2](10))  // 13
    
    // Counter factory
    fun makeCounter(start: Int = 0): () -> Int {
        var count = start
        return { count++ }
    }
    
    val counter1 = makeCounter()
    val counter2 = makeCounter(100)
    
    println(counter1())  // 0
    println(counter1())  // 1
    println(counter2())  // 100
    println(counter1())  // 2
    println(counter2())  // 101
}
```

### 3.2 Practical Closure Examples

```kotlin
fun createValidator(minLength: Int, maxLength: Int): (String) -> Boolean {
    return { input -> input.length in minLength..maxLength }
}

fun createMultiplier(factor: Int): (Int) -> Int {
    return { number -> number * factor }
}

fun memoize(fn: (Int) -> Long): (Int) -> Long {
    val cache = mutableMapOf<Int, Long>()
    return { n ->
        cache.getOrPut(n) { fn(n) }
    }
}

fun main() {
    // Validator factory
    val validatePassword = createValidator(8, 20)
    val validateUsername = createValidator(3, 15)
    
    println(validatePassword("short"))      // false
    println(validatePassword("validPass1")) // true
    println(validateUsername("ab"))         // false
    println(validateUsername("alice"))      // true
    
    // Multiplier factory
    val double = createMultiplier(2)
    val triple = createMultiplier(3)
    val times10 = createMultiplier(10)
    
    println(listOf(1, 2, 3, 4, 5).map(double))  // [2, 4, 6, 8, 10]
    println(listOf(1, 2, 3, 4, 5).map(triple))  // [3, 6, 9, 12, 15]
    
    // Memoized fibonacci
    var callCount = 0
    val fibonacci = memoize { n ->
        callCount++
        if (n <= 1) n.toLong()
        else {
            // สำหรับตัวอย่างนี้ใช้ iterative
            var a = 0L
            var b = 1L
            repeat(n - 1) { val tmp = b; b = a + b; a = tmp }
            b
        }
    }
    
    println(fibonacci(10))  // 55
    println(fibonacci(10))  // 55 (จาก cache)
    println(fibonacci(15))  // 610
    println("Total calls: $callCount")  // 2 (ไม่ใช่ 3)
}
```

---

## 🔧 4. Scope Functions (สรุปย่อ)

Scope functions ใช้เมื่อต้องการทำงานกับ object ใน scope พิเศษ
> ดูรายละเอียดเพิ่มเติมที่ **Part 18: Scope Functions**

### 4.1 สรุปตาราง

| Function | Object reference | Return value | Use case |
|----------|-----------------|--------------|----------|
| `let` | `it` | Lambda result | Null checks, transform |
| `run` | `this` | Lambda result | Initialize + compute |
| `with` | `this` | Lambda result | Group calls on object |
| `apply` | `this` | Object itself | Configure object |
| `also` | `it` | Object itself | Side effects |

```kotlin
data class Config(
    var host: String = "",
    var port: Int = 0,
    var timeout: Int = 0,
    var retries: Int = 0
)

fun main() {
    // apply - configure object, returns object
    val config = Config().apply {
        host = "localhost"
        port = 8080
        timeout = 5000
        retries = 3
    }
    println(config)
    
    // let - transform nullable, returns result
    val name: String? = "Alice"
    val length = name?.let { it.length } ?: 0
    println(length)  // 5
    
    // run - execute block on object, returns result
    val greeting = config.run {
        "Connecting to $host:$port (timeout: ${timeout}ms)"
    }
    println(greeting)
    
    // with - group calls, returns result
    val summary = with(config) {
        "Host: $host, Port: $port, Timeout: $timeout, Retries: $retries"
    }
    println(summary)
    
    // also - side effect, returns object
    val numbers = mutableListOf(1, 2, 3)
        .also { println("Before: $it") }
        .also { it.add(4) }
        .also { println("After: $it") }
    
    // Chain ด้วย scope functions
    val result = "  Hello, World!  "
        .let { it.trim() }
        .let { it.lowercase() }
        .let { it.replace(",", "") }
    println(result)  // hello world!
}
```

---

## 🏗️ 5. Higher-Order Functions

Higher-order function คือฟังก์ชันที่รับ function เป็น parameter หรือคืน function เป็นผลลัพธ์

### 5.1 รับ Function เป็น Parameter

```kotlin
// Timing function
fun <T> measureTime(label: String, block: () -> T): T {
    val start = System.currentTimeMillis()
    val result = block()
    val elapsed = System.currentTimeMillis() - start
    println("$label took ${elapsed}ms")
    return result
}

// Retry logic
fun <T> retry(times: Int, block: () -> T): T {
    var lastException: Exception? = null
    repeat(times) { attempt ->
        try {
            return block()
        } catch (e: Exception) {
            lastException = e
            println("Attempt ${attempt + 1} failed: ${e.message}")
        }
    }
    throw lastException!!
}

// Conditional execution
fun <T> runIf(condition: Boolean, value: T, block: (T) -> T): T =
    if (condition) block(value) else value

// Transform pipeline
fun <T> T.pipe(vararg transforms: (T) -> T): T =
    transforms.fold(this) { acc, transform -> transform(acc) }

fun main() {
    // measureTime
    val result = measureTime("Sorting") {
        (1..10000).toList().shuffled().sorted()
    }
    println("Sorted ${result.size} items")
    
    // retry
    var attempts = 0
    val value = retry(3) {
        attempts++
        if (attempts < 3) throw RuntimeException("Not ready yet")
        "Success!"
    }
    println(value)  // Success!
    
    // runIf
    val input = "  hello world  "
    val trimmed = runIf(input.contains(" "), input) { it.trim() }
    println(trimmed)  // hello world
    
    // pipe
    val processed = "  Hello, World!  "
        .pipe(
            { it.trim() },
            { it.lowercase() },
            { it.replace(",", "") },
            { it.replace(" ", "_") }
        )
    println(processed)  // hello_world!
}
```

### 5.2 คืน Function เป็นผลลัพธ์

```kotlin
// Predicate factory
fun <T> and(vararg predicates: (T) -> Boolean): (T) -> Boolean =
    { value -> predicates.all { it(value) } }

fun <T> or(vararg predicates: (T) -> Boolean): (T) -> Boolean =
    { value -> predicates.any { it(value) } }

fun <T> not(predicate: (T) -> Boolean): (T) -> Boolean =
    { value -> !predicate(value) }

// Currying
fun <A, B, C> curry(fn: (A, B) -> C): (A) -> (B) -> C =
    { a -> { b -> fn(a, b) } }

// Partial application
fun <A, B, C> partial(fn: (A, B) -> C, a: A): (B) -> C =
    { b -> fn(a, b) }

fun main() {
    data class Person(val name: String, val age: Int, val isStudent: Boolean)
    
    val people = listOf(
        Person("Alice", 22, true),
        Person("Bob", 35, false),
        Person("Charlie", 19, true),
        Person("Dave", 45, false),
        Person("Eve", 28, false)
    )
    
    // Composing predicates
    val isAdult = { p: Person -> p.age >= 18 }
    val isYoung = { p: Person -> p.age < 30 }
    val isStudent = { p: Person -> p.isStudent }
    
    val youngAdults = people.filter(and(isAdult, isYoung))
    println("Young adults: ${youngAdults.map { it.name }}")
    // [Alice, Charlie, Eve]
    
    val studentsOrOld = people.filter(or(isStudent, { it.age > 40 }))
    println("Students or old: ${studentsOrOld.map { it.name }}")
    
    val nonStudents = people.filter(not(isStudent))
    println("Non-students: ${nonStudents.map { it.name }}")
    
    // Currying example
    val multiply = curry { a: Int, b: Int -> a * b }
    val double = multiply(2)
    val triple = multiply(3)
    
    println(listOf(1, 2, 3, 4, 5).map(double))  // [2, 4, 6, 8, 10]
    println(listOf(1, 2, 3, 4, 5).map(triple))  // [3, 6, 9, 12, 15]
    
    // Partial application
    fun greet(greeting: String, name: String) = "$greeting, $name!"
    val sayHello = partial(::greet, "Hello")
    val sayHi = partial(::greet, "Hi")
    
    println(sayHello("Alice"))  // Hello, Alice!
    println(sayHi("Bob"))       // Hi, Bob!
}
```

### 5.3 Function Composition

```kotlin
// Compose functions: (f ∘ g)(x) = f(g(x))
fun <A, B, C> compose(f: (B) -> C, g: (A) -> B): (A) -> C = { a -> f(g(a)) }

infix fun <A, B, C> ((B) -> C).after(g: (A) -> B): (A) -> C = { a -> this(g(a)) }
infix fun <A, B, C> ((A) -> B).andThen(f: (B) -> C): (A) -> C = { a -> f(this(a)) }

fun main() {
    val trim = { s: String -> s.trim() }
    val uppercase = { s: String -> s.uppercase() }
    val exclaim = { s: String -> "$s!" }
    
    // compose
    val shout = compose(compose(exclaim, uppercase), trim)
    println(shout("  hello world  "))  // HELLO WORLD!
    
    // infix after
    val shout2 = exclaim after uppercase after trim
    println(shout2("  hello world  "))  // HELLO WORLD!
    
    // infix andThen (pipe order)
    val shout3 = trim andThen uppercase andThen exclaim
    println(shout3("  hello world  "))  // HELLO WORLD!
    
    // Practical example
    fun parseNumber(s: String): Int? = s.trim().toIntOrNull()
    fun validate(n: Int): Int? = if (n > 0) n else null
    fun formatResult(n: Int): String = "Result: $n"
    
    val process: (String) -> String? = { input ->
        parseNumber(input)
            ?.let { validate(it) }
            ?.let { formatResult(it) }
    }
    
    println(process("  42  "))   // Result: 42
    println(process(" -5 "))     // null
    println(process("abc"))      // null
}
```

---

## 🔄 6. Functional Programming Patterns

### 6.1 map, filter, reduce chain

```kotlin
data class Transaction(
    val id: Int,
    val amount: Double,
    val type: String,  // "income" or "expense"
    val category: String,
    val description: String
)

fun main() {
    val transactions = listOf(
        Transaction(1, 50000.0, "income", "salary", "Monthly salary"),
        Transaction(2, 3500.0, "expense", "food", "Groceries"),
        Transaction(3, 1200.0, "expense", "transport", "Taxi fare"),
        Transaction(4, 15000.0, "income", "freelance", "Web project"),
        Transaction(5, 5000.0, "expense", "rent", "Apartment rent"),
        Transaction(6, 800.0, "expense", "food", "Restaurant"),
        Transaction(7, 2500.0, "expense", "entertainment", "Concert ticket"),
        Transaction(8, 500.0, "expense", "food", "Coffee shop")
    )
    
    // Total income
    val totalIncome = transactions
        .filter { it.type == "income" }
        .map { it.amount }
        .reduce { sum, amount -> sum + amount }
    println("Total Income: ฿${totalIncome}")
    
    // Total expenses
    val totalExpenses = transactions
        .filter { it.type == "expense" }
        .sumOf { it.amount }
    println("Total Expenses: ฿${totalExpenses}")
    
    // Balance
    val balance = totalIncome - totalExpenses
    println("Balance: ฿${balance}")
    
    // Expenses by category
    val expenseByCategory = transactions
        .filter { it.type == "expense" }
        .groupBy { it.category }
        .mapValues { (_, txns) -> txns.sumOf { it.amount } }
        .entries
        .sortedByDescending { it.value }
    
    println("\nExpenses by Category:")
    expenseByCategory.forEach { (cat, amount) ->
        val percentage = (amount / totalExpenses * 100).toInt()
        println("  $cat: ฿$amount ($percentage%)")
    }
    
    // Top 3 expenses
    val top3 = transactions
        .filter { it.type == "expense" }
        .sortedByDescending { it.amount }
        .take(3)
    
    println("\nTop 3 Expenses:")
    top3.forEach { println("  ${it.description}: ฿${it.amount}") }
    
    // Category summary with fold
    val summary = transactions.fold(
        mutableMapOf<String, Double>()
    ) { map, transaction ->
        val sign = if (transaction.type == "income") 1.0 else -1.0
        map[transaction.category] = (map[transaction.category] ?: 0.0) + sign * transaction.amount
        map
    }
    println("\nNet by Category: $summary")
}
```

### 6.2 Data Processing Pipeline

```kotlin
data class RawData(val line: String)
data class ParsedRecord(val name: String, val value: Double, val unit: String)
data class ValidRecord(val name: String, val valueInKg: Double)
data class Report(val name: String, val weight: Double, val category: String)

fun main() {
    val rawLines = listOf(
        "Apple,1.5,kg",
        "Banana,500,g",
        "Cherry,0.25,kg",
        "INVALID_DATA",
        "Date,800,g",
        "Elderberry,0.1,kg",
        "bad_format,abc,kg"
    )
    
    // Step 1: Parse
    val parsed: List<ParsedRecord?> = rawLines.map { line ->
        val parts = line.split(",")
        if (parts.size != 3) return@map null
        val value = parts[1].toDoubleOrNull() ?: return@map null
        ParsedRecord(parts[0], value, parts[2])
    }
    
    // Step 2: Filter valid records
    val validParsed = parsed.filterNotNull()
    println("Valid after parsing: ${validParsed.size}/${rawLines.size}")
    
    // Step 3: Normalize units to kg
    val normalized: List<ValidRecord> = validParsed.map { record ->
        val valueInKg = when (record.unit.lowercase()) {
            "kg" -> record.value
            "g"  -> record.value / 1000.0
            "lb" -> record.value * 0.453592
            else -> return@map null
        } ?: return@map null
        ValidRecord(record.name, valueInKg)
    }.filterNotNull()
    
    // Step 4: Categorize
    val report: List<Report> = normalized.map { record ->
        val category = when {
            record.valueInKg < 0.2  -> "Light"
            record.valueInKg < 0.5  -> "Medium"
            record.valueInKg < 1.0  -> "Heavy"
            else                     -> "Very Heavy"
        }
        Report(record.name, record.valueInKg, category)
    }
    
    // Step 5: Generate report
    println("\n=== Weight Report ===")
    report
        .sortedByDescending { it.weight }
        .forEach { r ->
            println("${r.name.padEnd(15)} ${String.format("%.3f", r.weight)} kg (${r.category})")
        }
    
    println("\nBy Category:")
    report
        .groupBy { it.category }
        .forEach { (cat, items) ->
            println("  $cat: ${items.map { it.name }}")
        }
    
    val totalWeight = report.sumOf { it.weight }
    println("\nTotal weight: ${String.format("%.3f", totalWeight)} kg")
}
```

---

## 🛠️ 7. Practical Higher-Order Functions

### 7.1 Custom Collection Operations

```kotlin
// Custom operations ที่มีประโยชน์
fun <T> List<T>.splitWhen(predicate: (T) -> Boolean): List<List<T>> {
    val result = mutableListOf<MutableList<T>>()
    var current = mutableListOf<T>()
    for (item in this) {
        if (predicate(item)) {
            if (current.isNotEmpty()) {
                result.add(current)
                current = mutableListOf()
            }
        } else {
            current.add(item)
        }
    }
    if (current.isNotEmpty()) result.add(current)
    return result
}

fun <T, K> List<T>.indexBy(keySelector: (T) -> K): Map<K, T> =
    associateBy(keySelector)

fun <T> List<T>.frequencies(): Map<T, Int> =
    groupingBy { it }.eachCount()

fun <T> List<T>.batch(size: Int, operation: (List<T>) -> Unit) {
    chunked(size).forEach(operation)
}

fun main() {
    // splitWhen
    val numbers = listOf(1, 2, 0, 3, 4, 0, 5, 6, 7)
    val groups = numbers.splitWhen { it == 0 }
    println(groups)  // [[1, 2], [3, 4], [5, 6, 7]]
    
    // indexBy
    data class Product(val id: Int, val name: String, val price: Double)
    val products = listOf(
        Product(1, "Apple", 30.0),
        Product(2, "Banana", 15.0),
        Product(3, "Cherry", 50.0)
    )
    val byId = products.indexBy { it.id }
    println(byId[2]?.name)  // Banana
    
    // frequencies
    val words = "the quick brown fox jumps over the lazy dog".split(" ")
    val freq = words.frequencies()
    println(freq.entries.sortedByDescending { it.value }.take(3))
    
    // batch processing
    val items = (1..100).toList()
    var batchCount = 0
    items.batch(10) { batch ->
        batchCount++
        // Process each batch of 10
    }
    println("Processed $batchCount batches")  // 10
}
```

### 7.2 Event System ด้วย Lambda

```kotlin
typealias EventHandler<T> = (T) -> Unit

class EventBus<T> {
    private val handlers = mutableListOf<EventHandler<T>>()
    
    fun subscribe(handler: EventHandler<T>) {
        handlers.add(handler)
    }
    
    fun unsubscribe(handler: EventHandler<T>) {
        handlers.remove(handler)
    }
    
    fun emit(event: T) {
        handlers.forEach { it(event) }
    }
}

data class UserEvent(val type: String, val userId: Int, val data: Map<String, Any>)

fun main() {
    val userEvents = EventBus<UserEvent>()
    
    // Register handlers
    userEvents.subscribe { event ->
        println("[Logger] Event: ${event.type} for user ${event.userId}")
    }
    
    userEvents.subscribe { event ->
        if (event.type == "login") {
            println("[Security] Login detected for user ${event.userId}")
        }
    }
    
    val analyticsHandler: EventHandler<UserEvent> = { event ->
        println("[Analytics] Tracking ${event.type}: ${event.data}")
    }
    userEvents.subscribe(analyticsHandler)
    
    // Emit events
    userEvents.emit(UserEvent("login", 1, mapOf("ip" to "192.168.1.1", "device" to "mobile")))
    println("---")
    userEvents.emit(UserEvent("purchase", 1, mapOf("productId" to 42, "amount" to 1500)))
    println("---")
    
    // Unsubscribe analytics
    userEvents.unsubscribe(analyticsHandler)
    userEvents.emit(UserEvent("logout", 1, emptyMap()))
}
```

---

## 🏋️ แบบฝึกหัด

### ระดับ 1 (ง่าย)

1. เขียนฟังก์ชัน `applyOperation(numbers: List<Int>, op: (Int) -> Int): List<Int>` และทดสอบกับ: double, square, และ `n * n + 1`

2. สร้าง higher-order function `createGreeter(greeting: String): (String) -> String` แล้วสร้าง `sayHello`, `sayHi`, `sayGoodbye` จาก factory นี้

### ระดับ 2 (กลาง)

3. เขียน `compose` function ที่รวม list ของ functions เข้าด้วยกัน: `compose(listOf(trim, lowercase, removeSpaces))` ควรคืน function ที่ทำ 3 operations ตามลำดับ

4. สร้างระบบ validation ด้วย function types:
   - `type Validator<T> = (T) -> ValidationResult`
   - `data class ValidationResult(val isValid: Boolean, val errors: List<String>)`
   - สร้าง validators สำหรับ email, password, username
   - เขียน `combineValidators` ที่รวม validators หลายตัวเข้าด้วยกัน

### ระดับ 3 (ท้าทาย)

5. สร้าง `Pipeline<T>` class ที่:
   - มี `pipe(transform: (T) -> T): Pipeline<T>` 
   - มี `filter(predicate: (T) -> Boolean): Pipeline<T>`
   - มี `execute(input: T): T?` ที่คืน null ถ้า filter ไม่ผ่าน
   - ทดสอบกับ text processing pipeline

6. Implement `memoize` function ที่ general กว่าตัวอย่าง ให้รองรับ:
   - ฟังก์ชันที่มี 1, 2 parameters
   - cache size limit (LRU cache)
   - cache invalidation

---

## 📊 สรุป

| Concept | Syntax | ตัวอย่าง |
|---------|--------|---------|
| Lambda | `{ params -> body }` | `{ x -> x * 2 }` |
| Trailing Lambda | `fn(arg) { body }` | `list.map { it * 2 }` |
| it | implicit single param | `list.filter { it > 0 }` |
| Function type | `(Params) -> Return` | `(Int) -> Boolean` |
| Function ref | `::functionName` | `list.map(::double)` |
| Closure | capture outer vars | `val add = { x: Int -> x + base }` |
| Higher-order | fn takes/returns fn | `fun apply(fn: (T) -> T)` |
| Compose | f after g | `trim andThen uppercase` |

| Scope Function | Reference | Returns | Best For |
|---------------|-----------|---------|----------|
| `let` | `it` | Lambda result | Nullable + transform |
| `run` | `this` | Lambda result | Init + compute |
| `with` | `this` | Lambda result | Group calls |
| `apply` | `this` | Object | Builder pattern |
| `also` | `it` | Object | Side effects |

---

## ➡️ ถัดไป: Part 11 - Extension Functions

---
*Part 10/100+ | Kotlin & Spring Boot Complete Course*
