# Part 11: Extension Functions ขั้นสูง

## ทำความรู้จักกับ Extension Functions

Extension Functions เป็นหนึ่งในฟีเจอร์ที่ทรงพลังที่สุดของ Kotlin ช่วยให้เราสามารถเพิ่มฟังก์ชันให้กับ class ที่มีอยู่แล้ว โดยไม่ต้องแก้ไข source code ของ class นั้น ไม่ว่าจะเป็น class จาก Standard Library, Third-party Library หรือแม้แต่ class ของเราเอง

### ทำไม Extension Functions ถึงสำคัญ?

ลองนึกภาพว่าเราต้องการเพิ่มเมธอดให้กับ `String` class เพื่อตรวจสอบว่า string นั้นเป็น email ที่ถูกต้องหรือไม่ ใน Java เราต้องสร้าง Utility class:

```java
// Java way
public class StringUtils {
    public static boolean isValidEmail(String str) {
        return str.matches("[a-zA-Z0-9+_.-]+@[a-zA-Z0-9.-]+");
    }
}

// การใช้งาน
StringUtils.isValidEmail("test@example.com"); // ไม่ natural
```

แต่ใน Kotlin เราสามารถทำได้แบบนี้:

```kotlin
// Kotlin way
fun String.isValidEmail(): Boolean {
    return this.matches(Regex("[a-zA-Z0-9+_.-]+@[a-zA-Z0-9.-]+"))
}

// การใช้งาน - natural มาก!
"test@example.com".isValidEmail() // true
"not-an-email".isValidEmail()     // false
```

---

## 11.1 พื้นฐาน Extension Functions

### Syntax พื้นฐาน

```kotlin
fun ReceiverType.functionName(parameters): ReturnType {
    // ใช้ this เพื่ออ้างอิง receiver object
    return someValue
}
```

### ตัวอย่างเบื้องต้น

```kotlin
// Extension function สำหรับ Int
fun Int.isEven(): Boolean = this % 2 == 0
fun Int.isOdd(): Boolean = !this.isEven()
fun Int.squared(): Int = this * this
fun Int.factorial(): Long {
    if (this < 0) throw IllegalArgumentException("Factorial ไม่รองรับจำนวนลบ")
    return if (this == 0) 1L else (1..this).fold(1L) { acc, n -> acc * n }
}

fun main() {
    println(4.isEven())      // true
    println(7.isOdd())       // true
    println(5.squared())     // 25
    println(6.factorial())   // 720
    
    // ใช้กับ range
    val evenNumbers = (1..20).filter { it.isEven() }
    println(evenNumbers) // [2, 4, 6, 8, 10, 12, 14, 16, 18, 20]
}
```

```kotlin
// Extension function สำหรับ String
fun String.capitalizeWords(): String {
    return this.split(" ")
        .joinToString(" ") { word ->
            word.replaceFirstChar { it.uppercase() }
        }
}

fun String.truncate(maxLength: Int, suffix: String = "..."): String {
    return if (this.length <= maxLength) this
    else this.substring(0, maxLength - suffix.length) + suffix
}

fun String.countWords(): Int = this.trim().split(Regex("\\s+")).size

fun String.isPalindrome(): Boolean {
    val cleaned = this.lowercase().filter { it.isLetterOrDigit() }
    return cleaned == cleaned.reversed()
}

fun main() {
    println("hello world kotlin".capitalizeWords())  // Hello World Kotlin
    println("This is a very long string".truncate(15))  // This is a ver...
    println("The quick brown fox".countWords())  // 4
    println("A man a plan a canal Panama".isPalindrome())  // true
    println("Race car".isPalindrome())  // true
}
```

---

## 11.2 Extension Properties

นอกจาก Extension Functions เรายังสามารถสร้าง Extension Properties ได้ด้วย อย่างไรก็ตาม Extension Properties ไม่สามารถมี backing field ได้

```kotlin
// Extension property สำหรับ String
val String.lastChar: Char
    get() = this[this.length - 1]

val String.firstChar: Char
    get() = this[0]

val String.wordCount: Int
    get() = this.trim().split(Regex("\\s+")).size

val String.isBlankOrEmpty: Boolean
    get() = this.isBlank() || this.isEmpty()

// Extension property กับ getter และ setter
var StringBuilder.lastChar: Char
    get() = this[this.length - 1]
    set(value) {
        this.setCharAt(this.length - 1, value)
    }

fun main() {
    val text = "Hello, World!"
    println(text.lastChar)  // !
    println(text.firstChar) // H
    println("Hello World".wordCount)  // 2
    
    val sb = StringBuilder("Hello")
    sb.lastChar = '!'
    println(sb)  // Hell!
}
```

### Extension Properties ที่มีประโยชน์สำหรับ Collections

```kotlin
val <T> List<T>.secondOrNull: T?
    get() = if (this.size >= 2) this[1] else null

val <T> List<T>.thirdOrNull: T?
    get() = if (this.size >= 3) this[2] else null

val <T> Collection<T>.isNotEmpty: Boolean
    get() = !this.isEmpty()

val <T> Collection<T>.size: Int
    get() = this.size

// Extension property สำหรับ Map
val <K, V> Map<K, V>.isNotEmpty: Boolean
    get() = !this.isEmpty()

fun main() {
    val list = listOf(1, 2, 3, 4, 5)
    println(list.secondOrNull)  // 2
    println(list.thirdOrNull)   // 3
    
    val emptyList = emptyList<Int>()
    println(emptyList.secondOrNull)  // null
}
```

---

## 11.3 Extension Functions ใน Companion Objects

เราสามารถสร้าง Extension Functions สำหรับ Companion Objects ได้ ซึ่งทำให้เราสามารถเรียกใช้เหมือน static method

```kotlin
data class User(
    val id: Long,
    val name: String,
    val email: String,
    val age: Int
) {
    companion object {
        // factory method เบื้องต้น
        fun create(name: String, email: String, age: Int): User {
            return User(System.currentTimeMillis(), name, email, age)
        }
    }
}

// Extension function บน companion object
fun User.Companion.fromMap(map: Map<String, Any>): User {
    return User(
        id = (map["id"] as? Long) ?: System.currentTimeMillis(),
        name = map["name"] as String,
        email = map["email"] as String,
        age = (map["age"] as? Int) ?: 0
    )
}

fun User.Companion.anonymous(): User {
    return User(0L, "Anonymous", "anonymous@example.com", 0)
}

fun User.Companion.fromCsv(csvLine: String): User {
    val parts = csvLine.split(",")
    return User(
        id = parts[0].trim().toLong(),
        name = parts[1].trim(),
        email = parts[2].trim(),
        age = parts[3].trim().toInt()
    )
}

fun main() {
    // ใช้ companion object extensions เหมือน static methods
    val userFromMap = User.fromMap(mapOf(
        "name" to "Alice",
        "email" to "alice@example.com",
        "age" to 25
    ))
    println(userFromMap)
    
    val anonymous = User.anonymous()
    println(anonymous)
    
    val fromCsv = User.fromCsv("1, Bob, bob@example.com, 30")
    println(fromCsv)
}
```

```kotlin
// ตัวอย่างกับ Result type
sealed class ApiResult<out T> {
    data class Success<T>(val data: T) : ApiResult<T>()
    data class Error(val message: String, val code: Int = 0) : ApiResult<Nothing>()
    object Loading : ApiResult<Nothing>()
    
    companion object
}

// Extensions บน companion
fun <T> ApiResult.Companion.success(data: T): ApiResult<T> = ApiResult.Success(data)
fun ApiResult.Companion.error(message: String, code: Int = 0): ApiResult<Nothing> = 
    ApiResult.Error(message, code)
fun ApiResult.Companion.loading(): ApiResult<Nothing> = ApiResult.Loading

// Extensions บน type เอง
fun <T> ApiResult<T>.isSuccess(): Boolean = this is ApiResult.Success
fun <T> ApiResult<T>.isError(): Boolean = this is ApiResult.Error
fun <T> ApiResult<T>.isLoading(): Boolean = this is ApiResult.Loading

fun <T> ApiResult<T>.getOrNull(): T? = when (this) {
    is ApiResult.Success -> data
    else -> null
}

fun <T, R> ApiResult<T>.map(transform: (T) -> R): ApiResult<R> = when (this) {
    is ApiResult.Success -> ApiResult.success(transform(data))
    is ApiResult.Error -> this
    is ApiResult.Loading -> ApiResult.loading()
}

fun main() {
    val result: ApiResult<List<User>> = ApiResult.success(listOf(
        User(1, "Alice", "alice@example.com", 25)
    ))
    
    println(result.isSuccess())  // true
    println(result.getOrNull())  // [User...]
    
    val mapped = result.map { users -> users.size }
    println(mapped)  // Success(data=1)
}
```

---

## 11.4 Generic Extension Functions

Extension Functions สามารถเป็น Generic ได้ ทำให้มีความยืดหยุ่นสูงมาก

```kotlin
// Generic extension function พื้นฐาน
fun <T> T.also(block: (T) -> Unit): T {
    block(this)
    return this
}

fun <T, R> T.let(block: (T) -> R): R = block(this)

fun <T> T.takeIf(predicate: (T) -> Boolean): T? =
    if (predicate(this)) this else null

fun <T> T.takeUnless(predicate: (T) -> Boolean): T? =
    if (!predicate(this)) this else null
```

```kotlin
// Generic extensions ที่มีประโยชน์จริง
fun <T> List<T>.second(): T {
    if (this.size < 2) throw NoSuchElementException("List มี element น้อยกว่า 2")
    return this[1]
}

fun <T> List<T>.penultimate(): T {
    if (this.size < 2) throw NoSuchElementException("List มี element น้อยกว่า 2")
    return this[this.size - 2]
}

fun <T> Collection<T>.containsAll(vararg elements: T): Boolean {
    return elements.all { this.contains(it) }
}

fun <T> MutableList<T>.swap(index1: Int, index2: Int) {
    val tmp = this[index1]
    this[index1] = this[index2]
    this[index2] = tmp
}

fun <T : Comparable<T>> List<T>.isSorted(): Boolean {
    for (i in 0 until this.size - 1) {
        if (this[i] > this[i + 1]) return false
    }
    return true
}

fun <T : Comparable<T>> List<T>.isSortedDescending(): Boolean {
    for (i in 0 until this.size - 1) {
        if (this[i] < this[i + 1]) return false
    }
    return true
}

fun main() {
    val list = listOf(10, 20, 30, 40, 50)
    println(list.second())       // 20
    println(list.penultimate())  // 40
    println(list.containsAll(10, 30, 50))  // true
    println(list.isSorted())     // true
    
    val mutableList = mutableListOf(1, 2, 3, 4, 5)
    mutableList.swap(0, 4)
    println(mutableList)  // [5, 2, 3, 4, 1]
}
```

```kotlin
// Generic extension functions สำหรับการแปลงข้อมูล
fun <T, R> Iterable<T>.mapNotNull(transform: (T) -> R?): List<R> {
    return this.mapNotNull(transform)
}

fun <T> Iterable<T>.partitionBy(predicate: (T) -> Boolean): Pair<List<T>, List<T>> {
    val matching = mutableListOf<T>()
    val notMatching = mutableListOf<T>()
    for (element in this) {
        if (predicate(element)) matching.add(element)
        else notMatching.add(element)
    }
    return Pair(matching, notMatching)
}

fun <T, K> Iterable<T>.groupByAndCount(keySelector: (T) -> K): Map<K, Int> {
    return this.groupBy(keySelector).mapValues { it.value.size }
}

fun <T> Iterable<T>.firstOrDefault(default: T, predicate: (T) -> Boolean): T {
    return this.firstOrNull(predicate) ?: default
}

data class Product(val name: String, val category: String, val price: Double)

fun main() {
    val products = listOf(
        Product("Apple", "Fruit", 1.5),
        Product("Banana", "Fruit", 0.5),
        Product("Carrot", "Vegetable", 0.8),
        Product("Broccoli", "Vegetable", 1.2),
        Product("Cherry", "Fruit", 3.0)
    )
    
    val (expensive, cheap) = products.partitionBy { it.price > 1.0 }
    println("Expensive: ${expensive.map { it.name }}")  // [Apple, Carrot, Cherry]
    println("Cheap: ${cheap.map { it.name }}")          // [Banana, Broccoli]
    
    val countByCategory = products.groupByAndCount { it.category }
    println(countByCategory)  // {Fruit=3, Vegetable=2}
    
    val defaultProduct = Product("Unknown", "Unknown", 0.0)
    val found = products.firstOrDefault(defaultProduct) { it.name == "Apple" }
    println(found)  // Product(name=Apple, ...)
}
```

---

## 11.5 Extension Functions กับ Scope Functions

ใน Kotlin มี scope functions ที่ทรงพลังคือ `let`, `run`, `with`, `apply`, `also` ซึ่งล้วนเป็น Extension Functions

```kotlin
// ทำความเข้าใจ scope functions

data class Person(
    var name: String = "",
    var age: Int = 0,
    var email: String = ""
)

fun demonstrateScopeFunctions() {
    // let - ใช้เมื่อต้องการทำงานกับผลลัพธ์ของ expression
    val nameLength = "Hello, World!".let { str ->
        println("Processing: $str")
        str.length  // return value
    }
    println("Length: $nameLength")  // 13
    
    // run - ใช้เมื่อต้องการ compute value ภายใน object context
    val person = Person().run {
        name = "Alice"
        age = 25
        email = "alice@example.com"
        this  // return the object itself
    }
    println(person)
    
    // with - คล้าย run แต่รับ object เป็น argument
    val description = with(person) {
        "Name: $name, Age: $age, Email: $email"
    }
    println(description)
    
    // apply - ใช้สำหรับ configuration/initialization
    val configuredPerson = Person().apply {
        name = "Bob"
        age = 30
        email = "bob@example.com"
    }
    println(configuredPerson)
    
    // also - ใช้เมื่อต้องการ side effect โดยไม่เปลี่ยน value
    val numbers = mutableListOf(1, 2, 3)
        .also { println("Original: $it") }
        .apply { add(4) }
        .also { println("After add: $it") }
    println("Final: $numbers")
}

fun main() {
    demonstrateScopeFunctions()
}
```

---

## 11.6 String Extensions Library

ตัวอย่างจริงของการสร้าง String Extensions Library ที่ครบถ้วน

```kotlin
// StringExtensions.kt

/**
 * ตรวจสอบว่า String เป็น email ที่ถูกต้อง
 */
fun String.isValidEmail(): Boolean {
    val emailRegex = Regex("^[A-Za-z0-9+_.-]+@[A-Za-z0-9.-]+\\.[A-Za-z]{2,}$")
    return emailRegex.matches(this.trim())
}

/**
 * ตรวจสอบว่า String เป็นเบอร์โทรศัพท์ไทย
 */
fun String.isValidThaiPhone(): Boolean {
    val phoneRegex = Regex("^(0[689]\\d{8}|\\+66[689]\\d{8})$")
    return phoneRegex.matches(this.replace("-", "").replace(" ", ""))
}

/**
 * ตรวจสอบว่า String เป็น URL ที่ถูกต้อง
 */
fun String.isValidUrl(): Boolean {
    return try {
        java.net.URL(this)
        this.startsWith("http://") || this.startsWith("https://")
    } catch (e: Exception) {
        false
    }
}

/**
 * แปลง String เป็น camelCase
 */
fun String.toCamelCase(): String {
    return this.split(Regex("[_\\s-]+"))
        .mapIndexed { index, word ->
            if (index == 0) word.lowercase()
            else word.replaceFirstChar { it.uppercase() }
        }
        .joinToString("")
}

/**
 * แปลง String เป็น PascalCase
 */
fun String.toPascalCase(): String {
    return this.split(Regex("[_\\s-]+"))
        .joinToString("") { word ->
            word.replaceFirstChar { it.uppercase() }
        }
}

/**
 * แปลง String เป็น snake_case
 */
fun String.toSnakeCase(): String {
    return this.replace(Regex("([a-z])([A-Z])"), "$1_$2")
        .replace(Regex("[\\s-]+"), "_")
        .lowercase()
}

/**
 * แปลง String เป็น kebab-case
 */
fun String.toKebabCase(): String {
    return this.replace(Regex("([a-z])([A-Z])"), "$1-$2")
        .replace(Regex("[\\s_]+"), "-")
        .lowercase()
}

/**
 * ลบ HTML tags ออกจาก String
 */
fun String.stripHtml(): String {
    return this.replace(Regex("<[^>]*>"), "")
}

/**
 * แปลง HTML entities
 */
fun String.unescapeHtml(): String {
    return this
        .replace("&amp;", "&")
        .replace("&lt;", "<")
        .replace("&gt;", ">")
        .replace("&quot;", "\"")
        .replace("&#39;", "'")
        .replace("&nbsp;", " ")
}

/**
 * Mask ข้อมูลสำคัญ
 */
fun String.maskEmail(): String {
    val parts = this.split("@")
    if (parts.size != 2) return this
    val localPart = parts[0]
    val domain = parts[1]
    val masked = when {
        localPart.length <= 2 -> "*".repeat(localPart.length)
        else -> localPart.first() + "*".repeat(localPart.length - 2) + localPart.last()
    }
    return "$masked@$domain"
}

fun String.maskPhone(): String {
    return if (this.length < 4) this
    else "*".repeat(this.length - 4) + this.takeLast(4)
}

/**
 * นับจำนวนครั้งที่ substring ปรากฏ
 */
fun String.countOccurrences(substring: String, ignoreCase: Boolean = false): Int {
    if (substring.isEmpty()) return 0
    var count = 0
    var index = 0
    while (true) {
        index = this.indexOf(substring, index, ignoreCase)
        if (index == -1) break
        count++
        index += substring.length
    }
    return count
}

/**
 * แทนที่ตัวแรก n ตัว
 */
fun String.replaceFirst(n: Int, replacement: String): String {
    return if (this.length <= n) replacement
    else replacement + this.substring(n)
}

/**
 * ตัด whitespace ระหว่างคำออก (normalize)
 */
fun String.normalizeWhitespace(): String {
    return this.trim().replace(Regex("\\s+"), " ")
}

/**
 * ตรวจสอบว่า String เป็นตัวเลขล้วน
 */
fun String.isNumeric(): Boolean = this.all { it.isDigit() }

/**
 * ตรวจสอบว่า String เป็น alpha เท่านั้น
 */
fun String.isAlpha(): Boolean = this.all { it.isLetter() }

/**
 * ตรวจสอบว่า String เป็น alphanumeric
 */
fun String.isAlphaNumeric(): Boolean = this.all { it.isLetterOrDigit() }

/**
 * แปลง String เป็น List ของ chars ไม่ซ้ำ
 */
fun String.uniqueChars(): List<Char> = this.toCharArray().toSet().toList()

/**
 * ดึง substring ระหว่าง delimiters
 */
fun String.substringBetween(start: String, end: String): String? {
    val startIndex = this.indexOf(start)
    if (startIndex == -1) return null
    val endIndex = this.indexOf(end, startIndex + start.length)
    if (endIndex == -1) return null
    return this.substring(startIndex + start.length, endIndex)
}

// ทดสอบ String Extensions Library
fun main() {
    // Validation
    println("test@example.com".isValidEmail())    // true
    println("not-an-email".isValidEmail())         // false
    println("0812345678".isValidThaiPhone())       // true
    println("https://google.com".isValidUrl())     // true
    
    // Case conversion
    println("hello_world_kotlin".toCamelCase())    // helloWorldKotlin
    println("hello world kotlin".toPascalCase())   // HelloWorldKotlin
    println("helloWorldKotlin".toSnakeCase())       // hello_world_kotlin
    println("helloWorldKotlin".toKebabCase())       // hello-world-kotlin
    
    // Data masking
    println("john.doe@example.com".maskEmail())    // j******e@example.com
    println("0812345678".maskPhone())              // ******5678
    
    // String operations
    println("<p>Hello <b>World</b></p>".stripHtml())  // Hello World
    println("The cat sat on the mat".countOccurrences("at"))  // 3
    println("  Hello   World  ".normalizeWhitespace())  // Hello World
    
    // Extraction
    println("<title>My Page</title>".substringBetween("<title>", "</title>"))  // My Page
}
```

---

## 11.7 Collection Extensions Library

```kotlin
// CollectionExtensions.kt

/**
 * แยก list ออกเป็น chunks ขนาดที่กำหนด
 */
fun <T> List<T>.chunked(size: Int): List<List<T>> {
    require(size > 0) { "Chunk size must be positive" }
    return (0 until this.size step size).map { start ->
        this.subList(start, minOf(start + size, this.size))
    }
}

/**
 * หมุน list ไปทางซ้าย n ตำแหน่ง
 */
fun <T> List<T>.rotateLeft(n: Int): List<T> {
    if (this.isEmpty()) return this
    val shift = n % this.size
    return this.drop(shift) + this.take(shift)
}

/**
 * หมุน list ไปทางขวา n ตำแหน่ง
 */
fun <T> List<T>.rotateRight(n: Int): List<T> {
    return this.rotateLeft(this.size - (n % this.size))
}

/**
 * หาค่า median
 */
fun List<Double>.median(): Double {
    if (this.isEmpty()) throw NoSuchElementException("List is empty")
    val sorted = this.sorted()
    val mid = sorted.size / 2
    return if (sorted.size % 2 == 0) {
        (sorted[mid - 1] + sorted[mid]) / 2.0
    } else {
        sorted[mid]
    }
}

/**
 * หาค่า mode (ค่าที่ปรากฏบ่อยที่สุด)
 */
fun <T> List<T>.mode(): T? {
    return this.groupBy { it }
        .maxByOrNull { it.value.size }
        ?.key
}

/**
 * แบ่ง list ออกเป็นส่วนๆ โดย predicate
 */
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

/**
 * zip กับ index
 */
fun <T> List<T>.zipWithIndex(): List<Pair<Int, T>> {
    return this.mapIndexed { index, item -> Pair(index, item) }
}

/**
 * flatten แบบ deep
 */
fun <T> List<*>.deepFlatten(): List<T> {
    val result = mutableListOf<T>()
    for (item in this) {
        @Suppress("UNCHECKED_CAST")
        if (item is List<*>) {
            result.addAll(item.deepFlatten())
        } else {
            result.add(item as T)
        }
    }
    return result
}

/**
 * สร้าง sliding window
 */
fun <T> List<T>.windowed(size: Int, step: Int = 1): List<List<T>> {
    require(size > 0) { "Window size must be positive" }
    require(step > 0) { "Step must be positive" }
    return (0..this.size - size step step).map { start ->
        this.subList(start, start + size)
    }
}

/**
 * Map extension functions
 */
fun <K, V> Map<K, V>.getOrThrow(key: K): V {
    return this[key] ?: throw NoSuchElementException("Key '$key' not found")
}

fun <K, V> Map<K, V>.filterValues(predicate: (V) -> Boolean): Map<K, V> {
    return this.filter { (_, v) -> predicate(v) }
}

fun <K, V, R> Map<K, V>.mapValues(transform: (V) -> R): Map<K, R> {
    return this.mapValues { (_, v) -> transform(v) }
}

fun <K, V> Map<K, V>.toSortedByValue(): Map<K, V> where V : Comparable<V> {
    return this.entries.sortedBy { it.value }.associate { it.key to it.value }
}

// ทดสอบ Collection Extensions
fun main() {
    // List operations
    val numbers = listOf(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)
    
    println("Chunks: ${numbers.chunked(3)}")
    // [[1, 2, 3], [4, 5, 6], [7, 8, 9], [10]]
    
    println("Rotate left 2: ${numbers.rotateLeft(2)}")
    // [3, 4, 5, 6, 7, 8, 9, 10, 1, 2]
    
    println("Rotate right 3: ${numbers.rotateRight(3)}")
    // [8, 9, 10, 1, 2, 3, 4, 5, 6, 7]
    
    val doubles = listOf(1.0, 2.0, 3.0, 4.0, 5.0)
    println("Median: ${doubles.median()}")  // 3.0
    
    val words = listOf("a", "b", "a", "c", "b", "a")
    println("Mode: ${words.mode()}")  // a
    
    println("Windowed: ${listOf(1, 2, 3, 4, 5).windowed(3)}")
    // [[1, 2, 3], [2, 3, 4], [3, 4, 5]]
    
    // Map operations
    val scores = mapOf("Alice" to 95, "Bob" to 78, "Charlie" to 88)
    println("Sorted by value: ${scores.toSortedByValue()}")
    // {Bob=78, Charlie=88, Alice=95}
    
    val highScorers = scores.filterValues { it >= 85 }
    println("High scorers: $highScorers")  // {Alice=95, Charlie=88}
}
```

---

## 11.8 Operator Overloading ด้วย Extension Functions

```kotlin
// สร้าง Vector 2D class
data class Vector2D(val x: Double, val y: Double) {
    companion object
}

// Extension operators
operator fun Vector2D.plus(other: Vector2D) = Vector2D(x + other.x, y + other.y)
operator fun Vector2D.minus(other: Vector2D) = Vector2D(x - other.x, y - other.y)
operator fun Vector2D.times(scalar: Double) = Vector2D(x * scalar, y * scalar)
operator fun Vector2D.div(scalar: Double) = Vector2D(x / scalar, y / scalar)
operator fun Vector2D.unaryMinus() = Vector2D(-x, -y)

// Extension properties
val Vector2D.magnitude: Double
    get() = Math.sqrt(x * x + y * y)

val Vector2D.normalized: Vector2D
    get() {
        val mag = magnitude
        return if (mag == 0.0) Vector2D(0.0, 0.0) else this / mag
    }

// Extension functions
fun Vector2D.dot(other: Vector2D): Double = x * other.x + y * other.y
fun Vector2D.distanceTo(other: Vector2D): Double = (this - other).magnitude
fun Vector2D.angleTo(other: Vector2D): Double {
    return Math.atan2(other.y - y, other.x - x)
}

fun main() {
    val v1 = Vector2D(3.0, 4.0)
    val v2 = Vector2D(1.0, 2.0)
    
    println("v1 + v2 = ${v1 + v2}")          // (4.0, 6.0)
    println("v1 - v2 = ${v1 - v2}")          // (2.0, 2.0)
    println("v1 * 2 = ${v1 * 2.0}")          // (6.0, 8.0)
    println("magnitude of v1 = ${v1.magnitude}")  // 5.0
    println("normalized v1 = ${v1.normalized}")   // (0.6, 0.8)
    println("dot product = ${v1.dot(v2)}")    // 11.0
    println("distance = ${v1.distanceTo(v2)}")  // 2.828...
}
```

---

## 11.9 Extension Functions กับ Infix Notation

```kotlin
// Infix extension functions
infix fun Int.to(other: Int): IntRange = this..other
infix fun <T> T.into(list: MutableList<T>): MutableList<T> {
    list.add(this)
    return list
}

// Time DSL ด้วย infix functions
data class Duration(val milliseconds: Long) {
    operator fun plus(other: Duration) = Duration(milliseconds + other.milliseconds)
    override fun toString(): String {
        val seconds = milliseconds / 1000
        val minutes = seconds / 60
        val hours = minutes / 60
        return "${hours}h ${minutes % 60}m ${seconds % 60}s"
    }
}

val Int.seconds: Duration get() = Duration(this * 1000L)
val Int.minutes: Duration get() = Duration(this * 60 * 1000L)
val Int.hours: Duration get() = Duration(this * 60 * 60 * 1000L)
val Int.days: Duration get() = Duration(this * 24 * 60 * 60 * 1000L)

infix fun Duration.and(other: Duration): Duration = this + other

fun main() {
    // Infix usage
    val range = 1 to 10
    println(range)  // 1..10
    
    val list = mutableListOf<String>()
    "Hello" into list
    "World" into list
    println(list)  // [Hello, World]
    
    // Time DSL
    val duration = 2.hours and 30.minutes and 15.seconds
    println(duration)  // 2h 30m 15s
    
    val oneDay = 1.days
    println(oneDay)  // 24h 0m 0s
}
```

---

## 11.10 ข้อควรระวังใน Extension Functions

```kotlin
// 1. Extension function ไม่ได้เป็น polymorphic
open class Shape {
    open fun draw() = println("Drawing shape")
}

class Circle : Shape() {
    override fun draw() = println("Drawing circle")
}

fun Shape.describe() = "I am a Shape"
fun Circle.describe() = "I am a Circle"

fun main() {
    val shape: Shape = Circle()
    shape.draw()       // Drawing circle (polymorphic)
    shape.describe()   // I am a Shape (NOT polymorphic - ใช้ static type)
    
    val circle = Circle()
    circle.describe()  // I am a Circle
}
```

```kotlin
// 2. Member function มี priority สูงกว่า extension function
class MyClass {
    fun greet() = "Hello from member function"
}

// Extension function นี้จะไม่ถูกเรียกใช้
fun MyClass.greet() = "Hello from extension function"

fun main() {
    val obj = MyClass()
    println(obj.greet())  // Hello from member function (member function wins)
}
```

```kotlin
// 3. Extension functions ที่ nullable receiver
fun String?.isNullOrBlankSafe(): Boolean {
    return this == null || this.isBlank()
}

fun <T> T?.orDefault(default: T): T = this ?: default

fun main() {
    val nullString: String? = null
    println(nullString.isNullOrBlankSafe())  // true
    println("  ".isNullOrBlankSafe())        // true
    println("hello".isNullOrBlankSafe())     // false
    
    val value: Int? = null
    println(value.orDefault(0))  // 0
    println(42.orDefault(0))     // 42
}
```

---

## สรุปบทที่ 11

Extension Functions เป็นเครื่องมือทรงพลังที่ทำให้ Kotlin มีความยืดหยุ่นสูง:

| Feature | ประโยชน์ |
|---------|----------|
| Extension Functions | เพิ่ม behavior ให้ class โดยไม่ต้องแก้ source code |
| Extension Properties | เพิ่ม properties (แต่ไม่มี backing field) |
| Companion Extensions | เพิ่ม static-like methods |
| Generic Extensions | ฟังก์ชันที่ทำงานกับหลาย type |
| Operator Extensions | Override operators สำหรับ custom types |
| Infix Extensions | สร้าง DSL ที่อ่านง่าย |
| Nullable Extensions | จัดการ null safety อย่างปลอดภัย |

**ข้อควรจำ:**
- Extension functions ไม่เป็น polymorphic
- Member functions มี priority สูงกว่า extension functions
- Extension functions ถูก compile เป็น static functions
- ไม่สามารถเข้าถึง private members ของ receiver class

---

## แบบฝึกหัดบทที่ 11

### ระดับง่าย

1. สร้าง extension function `String.isPalindrome()` ที่ตรวจสอบว่าคำนั้นเป็น palindrome หรือไม่ (เช่น "racecar", "level")

2. สร้าง extension functions สำหรับ `Int`:
   - `isPositive()`: ตรวจสอบว่าเป็นจำนวนบวก
   - `isNegative()`: ตรวจสอบว่าเป็นจำนวนลบ
   - `absoluteValue()`: คืนค่า absolute value

3. สร้าง extension property `List<Int>.sum` และ `List<Int>.average`

### ระดับกลาง

4. สร้าง extension function `List<T>.groupConsecutive()` ที่จัดกลุ่ม elements ที่ติดกันและเหมือนกัน:
   ```kotlin
   listOf(1, 1, 2, 3, 3, 3, 1).groupConsecutive()
   // [[1, 1], [2], [3, 3, 3], [1]]
   ```

5. สร้าง extension function `String.format(vararg args: Any)` ที่แทนที่ `{0}`, `{1}`, ... ด้วย arguments:
   ```kotlin
   "Hello {0}, you are {1} years old".format("Alice", 25)
   // "Hello Alice, you are 25 years old"
   ```

6. สร้าง String extension library สำหรับการจัดการ Thai text:
   - `hasThaiCharacters()`: ตรวจสอบว่ามีตัวอักษรไทย
   - `removeThaiToneMarks()`: ลบวรรณยุกต์
   - `thaiWordCount()`: นับคำภาษาไทย (แยกด้วยช่องว่าง)

### ระดับยาก

7. สร้าง DSL สำหรับสร้าง HTML ด้วย extension functions:
   ```kotlin
   val html = buildHtml {
       head {
           title("My Page")
       }
       body {
           h1("Hello World")
           p("This is a paragraph")
           ul {
               li("Item 1")
               li("Item 2")
           }
       }
   }
   ```

8. สร้าง `Table<T>` class และ extension functions สำหรับ:
   - `select(vararg columns)`: เลือกคอลัมน์
   - `where(predicate)`: กรองข้อมูล
   - `orderBy(column, ascending)`: เรียงข้อมูล
   - `limit(n)`: จำกัดจำนวน rows

---

[ไปต่อ Part 12: Null Safety →](part-12-null-safety.md)

---

*Part 11/100+ | Kotlin & Spring Boot Complete Course*
