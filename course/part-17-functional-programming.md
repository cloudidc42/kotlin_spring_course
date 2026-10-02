# Part 17: Functional Programming
## Functional Programming Patterns in Kotlin

---

## 🎯 เป้าหมายของ Part นี้

- เข้าใจ Functional Programming (FP) concepts
- Pure functions และ immutability
- Function composition
- Currying และ Partial application
- Functors, Monads (ผ่าน Kotlin idioms)
- Sequence vs Eager evaluation
- ตัวอย่างจริง: Data pipeline

---

## 🌟 1. Pure Functions

```kotlin
// Pure function: output ขึ้นอยู่กับ input เท่านั้น, ไม่มี side effects
fun add(a: Int, b: Int) = a + b
fun square(n: Int) = n * n
fun toUpperCase(s: String) = s.uppercase()

// ❌ Impure: มี side effects
var globalCount = 0
fun impureIncrement(): Int {
    globalCount++   // side effect!
    return globalCount
}

// ✅ Pure: ไม่มี side effects
fun pureIncrement(count: Int) = count + 1

// ตัวอย่าง: คำนวณโดยไม่แก้ state
data class ShoppingCart(
    val items: List<CartItem>,
    val discountPercent: Double = 0.0
)

data class CartItem(val name: String, val price: Double, val qty: Int)

// Pure functions สำหรับ cart
fun totalPrice(cart: ShoppingCart): Double =
    cart.items.sumOf { it.price * it.qty }

fun afterDiscount(cart: ShoppingCart): Double =
    totalPrice(cart) * (1 - cart.discountPercent / 100)

fun addItem(cart: ShoppingCart, item: CartItem): ShoppingCart =
    cart.copy(items = cart.items + item)  // ไม่แก้ cart เดิม

fun removeItem(cart: ShoppingCart, itemName: String): ShoppingCart =
    cart.copy(items = cart.items.filter { it.name != itemName })

fun applyDiscount(cart: ShoppingCart, percent: Double): ShoppingCart =
    cart.copy(discountPercent = percent)

fun main() {
    val cart = ShoppingCart(emptyList())
    val cart1 = addItem(cart, CartItem("Kotlin Book", 599.0, 1))
    val cart2 = addItem(cart1, CartItem("Spring Course", 1200.0, 1))
    val cart3 = applyDiscount(cart2, 10.0)
    
    println("Total: ${totalPrice(cart3)}")
    println("After discount: ${afterDiscount(cart3)}")
}
```

---

## 🔗 2. Function Composition

```kotlin
// compose two functions: f(g(x))
fun <A, B, C> compose(f: (B) -> C, g: (A) -> B): (A) -> C = { a -> f(g(a)) }

// pipe: g(f(x)) - readable left-to-right
fun <A, B, C> pipe(f: (A) -> B, g: (B) -> C): (A) -> C = { a -> g(f(a)) }

// infix compose operators
infix fun <A, B, C> ((B) -> C).compose(g: (A) -> B): (A) -> C = { a -> this(g(a)) }
infix fun <A, B, C> ((A) -> B).andThen(g: (B) -> C): (A) -> C = { a -> g(this(a)) }

fun main() {
    val double = { x: Int -> x * 2 }
    val addOne = { x: Int -> x + 1 }
    val square = { x: Int -> x * x }
    
    // compose: right-to-left
    val doubleAndAddOne = compose(addOne, double)
    println(doubleAndAddOne(5))   // (5*2)+1 = 11
    
    // pipe / andThen: left-to-right
    val pipeline = double andThen addOne andThen square
    println(pipeline(3))  // ((3*2)+1)^2 = 49
    
    // String processing pipeline
    val processName = String::trim andThen String::lowercase andThen 
                      { it.replace(" ", "_") } andThen
                      { it.replaceFirstChar { c -> c.uppercase() } }
    
    println(processName("  Hello World  "))  // Hello_world
    
    // Data transformation pipeline
    val processNumbers = { nums: List<Int> ->
        nums.filter { it > 0 }
            .map { it * 2 }
            .filter { it < 20 }
            .sorted()
    }
    
    println(processNumbers(listOf(-1, 2, 5, 3, -3, 8, 11, 15)))
    // [4, 6, 10, 16]
}
```

---

## 🍛 3. Currying และ Partial Application

```kotlin
// Currying: แปลง f(a, b) เป็น f(a)(b)
fun <A, B, C> curry(f: (A, B) -> C): (A) -> (B) -> C = { a -> { b -> f(a, b) } }

fun add(a: Int, b: Int) = a + b
fun multiply(a: Int, b: Int) = a * b

val curriedAdd = curry(::add)
val add5 = curriedAdd(5)          // Partial application
println(add5(3))    // 8
println(add5(10))   // 15

val double = curry(::multiply)(2)  // Partial application
val triple = curry(::multiply)(3)

println(double(7))  // 14
println(triple(7))  // 21

// ตัวอย่างจริง: Discount calculator
fun applyDiscount(discount: Double, price: Double) = price * (1 - discount)
val tenPercentOff = curry(::applyDiscount)(0.10)
val twentyPercentOff = curry(::applyDiscount)(0.20)

val prices = listOf(100.0, 250.0, 599.0, 1200.0)
println(prices.map(tenPercentOff))    // [90.0, 225.0, 539.1, 1080.0]
println(prices.map(twentyPercentOff)) // [80.0, 200.0, 479.2, 960.0]

// Partial application แบบ Kotlin
fun <A, B, C> partial(f: (A, B) -> C, a: A): (B) -> C = { b -> f(a, b) }

val greet = { greeting: String, name: String -> "$greeting, $name!" }
val hello = partial(greet, "Hello")
val hi = partial(greet, "Hi")

println(hello("Alice"))  // Hello, Alice!
println(hi("Bob"))       // Hi, Bob!
```

---

## 🔄 4. Sequence (Lazy Evaluation)

```kotlin
fun main() {
    // ❌ Eager (ทำทุก step ก่อน ค่อย step ถัดไป)
    val eagerResult = (1..1_000_000)
        .filter { it % 2 == 0 }    // สร้าง list 500,000 ตัว
        .map { it * 3 }             // สร้าง list 500,000 ตัว
        .take(5)                    // เอาแค่ 5 ตัว
        .toList()
    println(eagerResult)  // [6, 12, 18, 24, 30]
    
    // ✅ Lazy (ทำ element ละครั้ง หยุดเมื่อได้พอ)
    val lazyResult = (1..1_000_000).asSequence()
        .filter { it % 2 == 0 }    // lazy
        .map { it * 3 }             // lazy
        .take(5)                    // lazy
        .toList()                   // terminal (เรียกใช้จริง)
    println(lazyResult)  // [6, 12, 18, 24, 30]
    
    // Infinite sequence
    val naturals = generateSequence(1) { it + 1 }  // ไม่มีที่สิ้นสุด
    val first10Squares = naturals.map { it * it }.take(10).toList()
    println(first10Squares)  // [1, 4, 9, 16, 25, 36, 49, 64, 81, 100]
    
    // Fibonacci sequence (lazy)
    val fibonacci = generateSequence(Pair(0L, 1L)) { (a, b) -> Pair(b, a + b) }
        .map { it.first }
    
    println(fibonacci.take(15).toList())
    // [0, 1, 1, 2, 3, 5, 8, 13, 21, 34, 55, 89, 144, 233, 377]
    
    // Performance comparison
    fun timeIt(block: () -> Unit): Long {
        val start = System.currentTimeMillis()
        block()
        return System.currentTimeMillis() - start
    }
    
    val eagerTime = timeIt {
        (1..100_000).filter { it % 3 == 0 }.map { it * 2 }.first()
    }
    
    val lazyTime = timeIt {
        (1..100_000).asSequence().filter { it % 3 == 0 }.map { it * 2 }.first()
    }
    
    println("Eager: ${eagerTime}ms, Lazy: ${lazyTime}ms")
}
```

---

## 🎯 5. Functional Data Structures

```kotlin
// Immutable Stack
sealed class Stack<out T> {
    object Empty : Stack<Nothing>()
    data class Cons<T>(val head: T, val tail: Stack<T>) : Stack<T>()
    
    companion object {
        fun <T> empty(): Stack<T> = Empty
        fun <T> of(vararg elements: T): Stack<T> =
            elements.foldRight(empty()) { elem, acc -> Cons(elem, acc) }
    }
}

fun <T> Stack<T>.push(value: T): Stack<T> = Stack.Cons(value, this)
fun <T> Stack<T>.pop(): Pair<T, Stack<T>> = when (this) {
    is Stack.Empty -> throw EmptyStackException()
    is Stack.Cons -> Pair(head, tail)
}
fun <T> Stack<T>.peek(): T = when (this) {
    is Stack.Empty -> throw EmptyStackException()
    is Stack.Cons -> head
}
fun <T> Stack<T>.isEmpty() = this is Stack.Empty

fun main() {
    val stack = Stack.of(1, 2, 3)
    val stack2 = stack.push(4)
    val (top, rest) = stack2.pop()
    
    println(top)    // 4
    println(stack2) // Cons(head=4, tail=Cons(head=1, tail=Cons(head=2, tail=Cons(head=3, tail=Empty))))
}

// Option/Maybe type (แทน Nullable)
sealed class Option<out T> {
    object None : Option<Nothing>()
    data class Some<T>(val value: T) : Option<T>()
    
    companion object {
        fun <T> of(value: T?): Option<T> = 
            if (value != null) Some(value) else None
    }
}

fun <T, R> Option<T>.map(f: (T) -> R): Option<R> = when (this) {
    is Option.None -> Option.None
    is Option.Some -> Option.Some(f(value))
}

fun <T, R> Option<T>.flatMap(f: (T) -> Option<R>): Option<R> = when (this) {
    is Option.None -> Option.None
    is Option.Some -> f(value)
}

fun <T> Option<T>.getOrElse(default: T): T = when (this) {
    is Option.None -> default
    is Option.Some -> value
}

fun findUser(id: Int): Option<String> = when (id) {
    1 -> Option.Some("Alice")
    2 -> Option.Some("Bob")
    else -> Option.None
}

fun findEmail(username: String): Option<String> = when (username) {
    "Alice" -> Option.Some("alice@example.com")
    else -> Option.None
}

fun main2() {
    val email = findUser(1)
        .flatMap { findEmail(it) }
        .map { it.uppercase() }
        .getOrElse("no email")
    
    println(email)  // ALICE@EXAMPLE.COM
    
    val email2 = findUser(99)
        .flatMap { findEmail(it) }
        .getOrElse("no email")
    
    println(email2) // no email
}
```

---

## 🏗️ 6. Data Pipeline Pattern

```kotlin
data class RawData(val id: Int, val name: String?, val value: String?, val date: String?)
data class ProcessedData(val id: Int, val name: String, val value: Double, val year: Int)

// Pipeline steps
fun validateData(raw: List<RawData>): List<RawData> =
    raw.filter { it.name != null && it.value != null && it.date != null }

fun parseData(raw: List<RawData>): List<ProcessedData> =
    raw.mapNotNull { r ->
        runCatching {
            ProcessedData(
                id = r.id,
                name = r.name!!.trim().titlecase(),
                value = r.value!!.toDouble(),
                year = r.date!!.substringBefore("-").toInt()
            )
        }.getOrNull()
    }

fun String.titlecase() = split(" ").joinToString(" ") { 
    it.replaceFirstChar { c -> c.uppercase() } 
}

fun enrichData(data: List<ProcessedData>): List<Map<String, Any>> =
    data.map { item ->
        mapOf(
            "id" to item.id,
            "name" to item.name,
            "value" to item.value,
            "year" to item.year,
            "category" to when {
                item.value < 100 -> "low"
                item.value < 1000 -> "medium"
                else -> "high"
            }
        )
    }

fun aggregateByYear(data: List<ProcessedData>): Map<Int, Summary> {
    data class Summary(val count: Int, val total: Double, val avg: Double)
    
    return data.groupBy { it.year }.mapValues { (_, items) ->
        val total = items.sumOf { it.value }
        Summary(items.size, total, total / items.size)
    }
}

fun main() {
    val rawData = listOf(
        RawData(1, "alice smith", "150.5", "2024-01-15"),
        RawData(2, null, "200.0", "2024-02-20"),      // invalid
        RawData(3, "bob jones", "not_a_number", "2024-03-10"),  // invalid
        RawData(4, "charlie brown", "2500.0", "2025-01-05"),
        RawData(5, "diana prince", "75.0", "2025-06-01")
    )
    
    // Pipeline
    val result = rawData
        .let(::validateData)   // Step 1: validate
        .let(::parseData)      // Step 2: parse
        .also { println("Valid records: ${it.size}") }
    
    // Enrich
    val enriched = enrichData(result)
    enriched.forEach { println(it) }
    
    // Aggregate
    val byYear = aggregateByYear(result)
    byYear.forEach { (year, summary) ->
        println("$year: count=${summary.count}, total=${summary.total}, avg=${summary.avg}")
    }
}
```

---

## 🏋️ 7. แบบฝึกหัด

### ข้อ 1: Function composition pipeline
```kotlin
// สร้าง text processing pipeline
fun normalize(text: String) = text.trim().lowercase()
fun removeSpecialChars(text: String) = text.replace(Regex("[^a-z0-9 ]"), "")
fun tokenize(text: String) = text.split(" ").filter { it.isNotEmpty() }
fun removeDuplicates(words: List<String>) = words.distinct()
fun sortWords(words: List<String>) = words.sorted()

fun processText(text: String): List<String> {
    return text
        .let(::normalize)
        .let(::removeSpecialChars)
        .let(::tokenize)
        .let(::removeDuplicates)
        .let(::sortWords)
}

fun main() {
    val text = "  Hello, World! Kotlin is GREAT. kotlin is awesome!  "
    println(processText(text))
    // [awesome, great, hello, is, kotlin, world]
}
```

### ข้อ 2: Sequence operations
```kotlin
// หาจำนวนเฉพาะ 100 ตัวแรก
fun isPrime(n: Int): Boolean {
    if (n < 2) return false
    if (n == 2) return true
    if (n % 2 == 0) return false
    for (i in 3..Math.sqrt(n.toDouble()).toInt() step 2) {
        if (n % i == 0) return false
    }
    return true
}

val first100Primes = generateSequence(2) { it + 1 }
    .filter { isPrime(it) }
    .take(100)
    .toList()

println("100th prime: ${first100Primes.last()}")
println("Sum of first 100 primes: ${first100Primes.sum()}")
```

---

## 📝 สรุป Part 17

| แนวคิด | ตัวอย่าง |
|--------|---------|
| Pure function | ไม่มี side effects |
| Function composition | `f andThen g` |
| Currying | `f(a)(b)` |
| Partial application | bind บาง args |
| Lazy sequence | `asSequence()` |
| Infinite sequence | `generateSequence` |
| Pipeline | chain of transformations |

---

## ➡️ ถัดไป: Part 18 - Scope Functions

---
*Part 17/100+ | Kotlin & Spring Boot Complete Course*
