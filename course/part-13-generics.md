# Part 13: Generics ใน Kotlin

## ทำความรู้จักกับ Generics

Generics ช่วยให้เราเขียน code ที่ทำงานกับ types หลากหลาย โดยยังคงมี type safety ขณะ compile เป็น tool ที่สำคัญอย่างยิ่งในการสร้าง reusable code ที่ปลอดภัย

```kotlin
// ปัญหาโดยไม่ใช้ generics
class IntBox(var value: Int)
class StringBox(var value: String)
class AnyBox(var value: Any)  // ใช้ได้แต่ไม่ type-safe

// แก้ด้วย generics
class Box<T>(var value: T)

fun main() {
    val intBox = Box(42)           // Box<Int>
    val strBox = Box("Hello")      // Box<String>
    
    println(intBox.value * 2)      // 84 - type-safe
    println(strBox.value.uppercase()) // HELLO - type-safe
    
    // ไม่สามารถทำผิด type ได้
    // intBox.value = "string"  // Compilation Error!
}
```

---

## 13.1 Generic Functions

```kotlin
// Generic function พื้นฐาน
fun <T> identity(value: T): T = value

fun <T> List<T>.secondOrNull(): T? = if (size >= 2) this[1] else null

fun <T, R> List<T>.mapToList(transform: (T) -> R): List<R> {
    val result = mutableListOf<R>()
    for (item in this) {
        result.add(transform(item))
    }
    return result
}

// Multiple type parameters
fun <K, V> Map<K, V>.reverseMap(): Map<V, K> {
    return entries.associate { (k, v) -> v to k }
}

fun <A, B, C> Pair<A, B>.mapFirst(transform: (A) -> C): Pair<C, B> {
    return Pair(transform(first), second)
}

fun <A, B, C> Pair<A, B>.mapSecond(transform: (B) -> C): Pair<A, C> {
    return Pair(first, transform(second))
}

// Generic function ที่ return generic type
fun <T> List<T>.partition(predicate: (T) -> Boolean): Pair<List<T>, List<T>> {
    val matching = mutableListOf<T>()
    val notMatching = mutableListOf<T>()
    for (item in this) {
        if (predicate(item)) matching.add(item) else notMatching.add(item)
    }
    return Pair(matching, notMatching)
}

fun main() {
    println(identity(42))          // 42
    println(identity("Hello"))     // Hello
    println(identity(listOf(1,2))) // [1, 2]
    
    val list = listOf(10, 20, 30, 40)
    println(list.secondOrNull())   // 20
    
    val doubled = list.mapToList { it * 2 }
    println(doubled)  // [20, 40, 60, 80]
    
    val map = mapOf("a" to 1, "b" to 2, "c" to 3)
    println(map.reverseMap())  // {1=a, 2=b, 3=c}
    
    val (evens, odds) = listOf(1, 2, 3, 4, 5, 6).partition { it % 2 == 0 }
    println("Evens: $evens, Odds: $odds")  // Evens: [2, 4, 6], Odds: [1, 3, 5]
}
```

---

## 13.2 Generic Classes

```kotlin
// Generic class พื้นฐาน
class Stack<T> {
    private val elements = mutableListOf<T>()
    
    fun push(element: T) = elements.add(element)
    
    fun pop(): T {
        if (elements.isEmpty()) throw NoSuchElementException("Stack is empty")
        return elements.removeAt(elements.size - 1)
    }
    
    fun peek(): T {
        if (elements.isEmpty()) throw NoSuchElementException("Stack is empty")
        return elements.last()
    }
    
    fun isEmpty(): Boolean = elements.isEmpty()
    val size: Int get() = elements.size
    
    override fun toString(): String = "Stack$elements"
}

// Generic data class
data class Pair2<A, B>(val first: A, val second: B) {
    fun swap(): Pair2<B, A> = Pair2(second, first)
    fun <C> mapFirst(transform: (A) -> C): Pair2<C, B> = Pair2(transform(first), second)
    fun <C> mapSecond(transform: (B) -> C): Pair2<A, C> = Pair2(first, transform(second))
}

// Generic wrapper
class Lazy<T>(private val initializer: () -> T) {
    private var _value: T? = null
    private var initialized = false
    
    fun get(): T {
        if (!initialized) {
            _value = initializer()
            initialized = true
        }
        @Suppress("UNCHECKED_CAST")
        return _value as T
    }
    
    val isInitialized: Boolean get() = initialized
}

// Generic sealed class (Result type)
sealed class Either<out L, out R> {
    data class Left<L>(val value: L) : Either<L, Nothing>()
    data class Right<R>(val value: R) : Either<Nothing, R>()
    
    fun isLeft() = this is Left
    fun isRight() = this is Right
    
    fun leftOrNull(): L? = (this as? Left)?.value
    fun rightOrNull(): R? = (this as? Right)?.value
    
    fun <T> fold(onLeft: (L) -> T, onRight: (R) -> T): T = when (this) {
        is Left -> onLeft(value)
        is Right -> onRight(value)
    }
}

fun main() {
    // Stack usage
    val stack = Stack<Int>()
    stack.push(1)
    stack.push(2)
    stack.push(3)
    println(stack)         // Stack[1, 2, 3]
    println(stack.pop())   // 3
    println(stack.peek())  // 2
    println(stack.size)    // 2
    
    // Pair2 usage
    val pair = Pair2("Alice", 25)
    println(pair)           // Pair2(first=Alice, second=25)
    println(pair.swap())    // Pair2(first=25, second=Alice)
    println(pair.mapSecond { it + 1 })  // Pair2(first=Alice, second=26)
    
    // Lazy usage
    var initCount = 0
    val lazyValue = Lazy {
        initCount++
        "Expensive computation result"
    }
    println(lazyValue.isInitialized)  // false
    println(lazyValue.get())           // Expensive computation result
    println(lazyValue.get())           // Expensive computation result (cached)
    println("Init count: $initCount") // Init count: 1
    
    // Either usage
    val success: Either<String, Int> = Either.Right(42)
    val failure: Either<String, Int> = Either.Left("Error occurred")
    
    println(success.fold(
        onLeft = { "Error: $it" },
        onRight = { "Value: $it" }
    ))  // Value: 42
    
    println(failure.fold(
        onLeft = { "Error: $it" },
        onRight = { "Value: $it" }
    ))  // Error: Error occurred
}
```

---

## 13.3 Type Bounds

Type bounds กำหนดข้อจำกัดว่า type parameter ต้องเป็น subtype ของ type ใด

### Upper Bounds

```kotlin
// Upper bound: T ต้องเป็น Comparable<T>
fun <T : Comparable<T>> max(a: T, b: T): T = if (a > b) a else b
fun <T : Comparable<T>> min(a: T, b: T): T = if (a < b) a else b
fun <T : Comparable<T>> clamp(value: T, min: T, max: T): T {
    return when {
        value < min -> min
        value > max -> max
        else -> value
    }
}

// Multiple bounds ด้วย where clause
fun <T> processItem(item: T) where T : Comparable<T>, T : Cloneable {
    println("Processing: $item")
}

// Upper bound กับ custom interface
interface Printable {
    fun print()
}

interface Saveable {
    fun save()
}

fun <T> processDocument(doc: T) where T : Printable, T : Saveable {
    doc.print()
    doc.save()
}

// ตัวอย่างกับ Number
fun <T : Number> sum(numbers: List<T>): Double {
    return numbers.sumOf { it.toDouble() }
}

fun <T : Number> average(numbers: List<T>): Double {
    if (numbers.isEmpty()) return 0.0
    return sum(numbers) / numbers.size
}

fun main() {
    println(max(3, 7))      // 7
    println(max("apple", "banana"))  // banana
    println(min(3.14, 2.71))  // 2.71
    
    println(clamp(15, 0, 10))   // 10
    println(clamp(-5, 0, 10))   // 0
    println(clamp(5, 0, 10))    // 5
    
    println(sum(listOf(1, 2, 3, 4, 5)))        // 15.0
    println(average(listOf(10.0, 20.0, 30.0))) // 20.0
}
```

### ตัวอย่าง Upper Bounds ที่ซับซ้อน

```kotlin
// Generic sorting algorithm
fun <T : Comparable<T>> MutableList<T>.bubbleSort() {
    val n = size
    for (i in 0 until n - 1) {
        for (j in 0 until n - i - 1) {
            if (this[j] > this[j + 1]) {
                val temp = this[j]
                this[j] = this[j + 1]
                this[j + 1] = temp
            }
        }
    }
}

fun <T : Comparable<T>> List<T>.binarySearch(target: T): Int {
    var left = 0
    var right = size - 1
    while (left <= right) {
        val mid = (left + right) / 2
        when {
            this[mid] == target -> return mid
            this[mid] < target -> left = mid + 1
            else -> right = mid - 1
        }
    }
    return -1
}

// Generic min-heap
class MinHeap<T : Comparable<T>> {
    private val heap = mutableListOf<T>()
    
    fun insert(value: T) {
        heap.add(value)
        heapifyUp(heap.size - 1)
    }
    
    fun extractMin(): T {
        if (heap.isEmpty()) throw NoSuchElementException("Heap is empty")
        val min = heap[0]
        heap[0] = heap.last()
        heap.removeAt(heap.size - 1)
        if (heap.isNotEmpty()) heapifyDown(0)
        return min
    }
    
    fun peek(): T = heap.firstOrNull() ?: throw NoSuchElementException("Heap is empty")
    val size: Int get() = heap.size
    fun isEmpty() = heap.isEmpty()
    
    private fun heapifyUp(index: Int) {
        var i = index
        while (i > 0) {
            val parent = (i - 1) / 2
            if (heap[i] >= heap[parent]) break
            val temp = heap[i]; heap[i] = heap[parent]; heap[parent] = temp
            i = parent
        }
    }
    
    private fun heapifyDown(index: Int) {
        var i = index
        while (true) {
            val left = 2 * i + 1
            val right = 2 * i + 2
            var smallest = i
            if (left < heap.size && heap[left] < heap[smallest]) smallest = left
            if (right < heap.size && heap[right] < heap[smallest]) smallest = right
            if (smallest == i) break
            val temp = heap[i]; heap[i] = heap[smallest]; heap[smallest] = temp
            i = smallest
        }
    }
}

fun main() {
    val list = mutableListOf(5, 3, 1, 4, 2)
    list.bubbleSort()
    println(list)  // [1, 2, 3, 4, 5]
    
    val sorted = listOf(1, 3, 5, 7, 9, 11)
    println(sorted.binarySearch(7))   // 3
    println(sorted.binarySearch(6))   // -1
    
    val heap = MinHeap<Int>()
    heap.insert(5); heap.insert(3); heap.insert(1); heap.insert(4); heap.insert(2)
    while (!heap.isEmpty()) print("${heap.extractMin()} ")  // 1 2 3 4 5
}
```

---

## 13.4 Variance: Covariant (out) และ Contravariant (in)

Variance เกี่ยวข้องกับ subtype relationships ระหว่าง generic types

### Covariance (out)

Covariant ใช้ `out` keyword บอกว่า generic type จะถูก "ผลิต" (produce) เท่านั้น ไม่ถูก "บริโภค" (consume)

```kotlin
// ปัญหาโดยไม่มี variance
class Box<T>(var value: T)

fun main() {
    val intBox = Box(42)
    // val anyBox: Box<Any> = intBox  // Error! แม้ Int เป็น subtype ของ Any
}

// Covariant (out) - producer
interface Producer<out T> {
    fun produce(): T
}

// Kotlin built-in: List<out E>
val intList: List<Int> = listOf(1, 2, 3)
val anyList: List<Any> = intList  // OK! List เป็น covariant

// ตัวอย่างจริง
class ReadOnlyBox<out T>(private val value: T) {
    fun get(): T = value
    // fun set(value: T) = ...  // Error! ไม่สามารถรับ T เป็น input
}

fun printAny(box: ReadOnlyBox<Any>) {
    println(box.get())
}

fun main() {
    val intBox = ReadOnlyBox(42)
    val strBox = ReadOnlyBox("Hello")
    
    printAny(intBox)  // OK! ReadOnlyBox<Int> เป็น subtype ของ ReadOnlyBox<Any>
    printAny(strBox)  // OK!
}
```

### Contravariance (in)

Contravariant ใช้ `in` keyword บอกว่า generic type จะถูก "บริโภค" (consume) เท่านั้น

```kotlin
// Contravariant (in) - consumer
interface Consumer<in T> {
    fun consume(value: T)
}

// ตัวอย่าง: Comparator เป็น contravariant
interface Comparator<in T> {
    fun compare(a: T, b: T): Int
}

val anyComparator = object : Comparator<Any> {
    override fun compare(a: Any, b: Any): Int = a.hashCode() - b.hashCode()
}

// ใช้ anyComparator กับ String ได้ เพราะ String เป็น subtype ของ Any
val strComparator: Comparator<String> = anyComparator  // OK!

// Writer เป็น contravariant
class Printer<in T> {
    fun print(value: T) = println(value)
}

fun main() {
    val anyPrinter = Printer<Any>()
    val intPrinter: Printer<Int> = anyPrinter  // OK! Printer<Any> เป็น subtype ของ Printer<Int>
    intPrinter.print(42)
}
```

### Invariant (ไม่มี modifier)

```kotlin
// Invariant - ทั้ง produce และ consume
class MutableBox<T>(var value: T)  // invariant

// ไม่สามารถ assign ระหว่าง types ได้
val intBox = MutableBox(42)
// val anyBox: MutableBox<Any> = intBox  // Error!
// val nothingBox: MutableBox<Nothing> = intBox  // Error!
```

### ตัวอย่าง Variance ในชีวิตจริง

```kotlin
// Reading pipeline (covariant)
interface DataSource<out T> {
    fun read(): T
    fun readAll(): List<T>
}

class NumberSource : DataSource<Int> {
    private val numbers = listOf(1, 2, 3, 4, 5)
    override fun read(): Int = numbers.random()
    override fun readAll(): List<Int> = numbers
}

fun processSource(source: DataSource<Number>) {
    val value: Number = source.read()
    println("Got: $value (${value::class.simpleName})")
}

// Writing pipeline (contravariant)
interface DataSink<in T> {
    fun write(value: T)
    fun writeAll(values: List<T>)
}

class NumberSink : DataSink<Number> {
    override fun write(value: Number) = println("Writing: $value")
    override fun writeAll(values: List<Number>) = values.forEach { write(it) }
}

fun writeInts(sink: DataSink<Int>, values: List<Int>) {
    values.forEach { sink.write(it) }
}

fun main() {
    val numSource = NumberSource()
    processSource(numSource)  // OK! DataSource<Int> เป็น subtype ของ DataSource<Number>
    
    val numSink = NumberSink()
    writeInts(numSink, listOf(1, 2, 3))  // OK! DataSink<Number> เป็น subtype ของ DataSink<Int>
}
```

---

## 13.5 Star Projection

Star projection (`*`) ใช้เมื่อต้องการอ้างถึง generic type แต่ไม่สนใจ type parameter

```kotlin
// Star projection พื้นฐาน
fun printListSize(list: List<*>) {
    println("Size: ${list.size}")
    // list.get(0)  // returns Any?
}

// ความแตกต่างระหว่าง List<*>, List<Any?>, List<Any>
val stars: List<*> = listOf(1, "hello", true)   // อ่านได้เป็น Any?
val anyList: List<Any?> = listOf(1, "hello", null)  // nullable
val nonNullList: List<Any> = listOf(1, "hello")  // non-null

// Star projection กับ function types
fun processCallback(callback: ((Any?) -> Unit)?) {
    callback?.invoke("Hello")
}

// ตัวอย่าง: type-safe heterogeneous container
class TypeSafeMap {
    private val map = mutableMapOf<Class<*>, Any>()
    
    fun <T : Any> put(type: Class<T>, value: T) {
        map[type] = value
    }
    
    @Suppress("UNCHECKED_CAST")
    fun <T : Any> get(type: Class<T>): T? {
        return map[type] as? T
    }
}

fun main() {
    printListSize(listOf(1, 2, 3))    // Size: 3
    printListSize(listOf("a", "b"))   // Size: 2
    
    val tsMap = TypeSafeMap()
    tsMap.put(String::class.java, "Hello")
    tsMap.put(Int::class.java, 42)
    tsMap.put(List::class.java, listOf(1, 2, 3))
    
    println(tsMap.get(String::class.java))  // Hello
    println(tsMap.get(Int::class.java))     // 42
}
```

---

## 13.6 Reified Type Parameters

`reified` ช่วยให้เราเข้าถึง type parameter ตอน runtime ในฟังก์ชัน inline เท่านั้น

```kotlin
// ปัญหา: ปกติ type parameters ถูกลบออก (type erasure)
fun <T> isType(value: Any): Boolean {
    // return value is T  // Error! Cannot check for erased type T
    return false
}

// แก้ด้วย reified
inline fun <reified T> isType(value: Any): Boolean {
    return value is T
}

inline fun <reified T> castOrNull(value: Any?): T? {
    return value as? T
}

inline fun <reified T> List<*>.filterIsInstanceOf(): List<T> {
    return this.filterIsInstance<T>()
}

// ตัวอย่างที่ใช้จริง
inline fun <reified T : Any> String.toObjectOrNull(): T? {
    return try {
        // JSON parsing
        jacksonObjectMapper().readValue(this, T::class.java)
    } catch (e: Exception) {
        null
    }
}

// reified กับ Kotlin reflection
inline fun <reified T : Any> createInstance(): T {
    return T::class.java.getDeclaredConstructor().newInstance()
}

inline fun <reified T : Any> getTypeName(): String {
    return T::class.simpleName ?: "Unknown"
}

// reified สำหรับ Android/Spring (ตัวอย่าง pattern)
inline fun <reified T : Any> inject(): T {
    // ปกติจะ lookup จาก DI container
    return T::class.java.getDeclaredConstructor().newInstance()
}

fun main() {
    println(isType<String>("Hello"))  // true
    println(isType<Int>("Hello"))     // false
    println(isType<Int>(42))          // true
    
    val mixed: List<Any> = listOf(1, "hello", 2.0, "world", 3)
    val strings = mixed.filterIsInstanceOf<String>()
    val ints = mixed.filterIsInstanceOf<Int>()
    println("Strings: $strings")  // [hello, world]
    println("Ints: $ints")        // [1, 3]
    
    println(getTypeName<String>())     // String
    println(getTypeName<List<Int>>())  // List
    
    val cast: Int? = castOrNull<Int>(42)
    val fail: Int? = castOrNull<Int>("not int")
    println(cast)  // 42
    println(fail)  // null
}

// ต้อง import จริง
fun jacksonObjectMapper() = com.fasterxml.jackson.module.kotlin.jacksonObjectMapper()
```

---

## 13.7 Generic Repository Pattern

ตัวอย่าง: Generic Repository สำหรับ Spring Boot

```kotlin
// Generic Repository interface
interface Repository<T, ID> {
    fun findById(id: ID): T?
    fun findAll(): List<T>
    fun save(entity: T): T
    fun delete(id: ID): Boolean
    fun count(): Long
    fun existsById(id: ID): Boolean
}

// ในชีวิตจริงจะ extend JpaRepository
// interface UserRepository : JpaRepository<User, Long>

// In-memory implementation
abstract class InMemoryRepository<T, ID> : Repository<T, ID> {
    protected val store = mutableMapOf<ID, T>()
    
    abstract fun getId(entity: T): ID
    
    override fun findById(id: ID): T? = store[id]
    override fun findAll(): List<T> = store.values.toList()
    override fun save(entity: T): T {
        store[getId(entity)] = entity
        return entity
    }
    override fun delete(id: ID): Boolean = store.remove(id) != null
    override fun count(): Long = store.size.toLong()
    override fun existsById(id: ID): Boolean = store.containsKey(id)
}

// Specific repositories
data class User(val id: Long, val name: String, val email: String)
data class Product(val id: Long, val name: String, val price: Double)

class UserRepository : InMemoryRepository<User, Long>() {
    override fun getId(entity: User): Long = entity.id
    
    fun findByEmail(email: String): User? {
        return store.values.find { it.email == email }
    }
    
    fun findByNameContaining(name: String): List<User> {
        return store.values.filter { it.name.contains(name, ignoreCase = true) }
    }
}

class ProductRepository : InMemoryRepository<Product, Long>() {
    override fun getId(entity: Product): Long = entity.id
    
    fun findByPriceRange(min: Double, max: Double): List<Product> {
        return store.values.filter { it.price in min..max }
    }
    
    fun findCheapestN(n: Int): List<Product> {
        return store.values.sortedBy { it.price }.take(n)
    }
}

// Generic Service layer
abstract class CrudService<T, ID>(protected val repository: Repository<T, ID>) {
    fun getById(id: ID): T = repository.findById(id) 
        ?: throw NoSuchElementException("Entity with id $id not found")
    
    fun getAll(): List<T> = repository.findAll()
    fun save(entity: T): T = repository.save(entity)
    fun delete(id: ID): Boolean = repository.delete(id)
    fun count(): Long = repository.count()
    fun exists(id: ID): Boolean = repository.existsById(id)
}

class UserService(repo: UserRepository) : CrudService<User, Long>(repo) {
    private val userRepo = repo
    
    fun findByEmail(email: String): User? = userRepo.findByEmail(email)
    fun search(query: String): List<User> = userRepo.findByNameContaining(query)
}

fun main() {
    val userRepo = UserRepository()
    val userService = UserService(userRepo)
    
    // สร้าง users
    val alice = userService.save(User(1L, "Alice Smith", "alice@example.com"))
    val bob = userService.save(User(2L, "Bob Johnson", "bob@example.com"))
    val charlie = userService.save(User(3L, "Charlie Brown", "charlie@example.com"))
    
    println("Count: ${userService.count()}")  // 3
    println("All: ${userService.getAll().map { it.name }}")
    
    println("Alice: ${userService.getById(1L)}")
    println("By email: ${userService.findByEmail("bob@example.com")?.name}")
    println("Search 'john': ${userService.search("john").map { it.name }}")
    
    userService.delete(2L)
    println("After delete: ${userService.count()}")  // 2
    
    // Products
    val productRepo = ProductRepository()
    productRepo.save(Product(1L, "Apple", 1.5))
    productRepo.save(Product(2L, "Banana", 0.5))
    productRepo.save(Product(3L, "Cherry", 3.0))
    productRepo.save(Product(4L, "Date", 2.5))
    
    println("Price 1-3: ${productRepo.findByPriceRange(1.0, 3.0).map { it.name }}")
    println("Cheapest 2: ${productRepo.findCheapestN(2).map { it.name }}")
}
```

---

## 13.8 Result Type Pattern

```kotlin
// Result type ที่สมบูรณ์ด้วย generics
sealed class Result<out T> {
    data class Success<T>(val data: T) : Result<T>()
    data class Error(
        val message: String,
        val code: Int = 0,
        val cause: Throwable? = null
    ) : Result<Nothing>()
    
    companion object {
        fun <T> success(data: T): Result<T> = Success(data)
        fun error(message: String, code: Int = 0, cause: Throwable? = null): Result<Nothing> =
            Error(message, code, cause)
        
        // สร้าง Result จาก lambda ที่อาจ throw
        inline fun <T> runCatching(block: () -> T): Result<T> {
            return try {
                success(block())
            } catch (e: Exception) {
                error(e.message ?: "Unknown error", cause = e)
            }
        }
    }
    
    val isSuccess: Boolean get() = this is Success
    val isError: Boolean get() = this is Error
    
    fun getOrNull(): T? = (this as? Success)?.data
    fun errorOrNull(): Error? = this as? Error
    
    fun getOrThrow(): T = when (this) {
        is Success -> data
        is Error -> throw cause ?: RuntimeException(message)
    }
    
    fun getOrDefault(default: @UnsafeVariance T): T = when (this) {
        is Success -> data
        is Error -> default
    }
    
    fun getOrElse(default: (Error) -> @UnsafeVariance T): T = when (this) {
        is Success -> data
        is Error -> default(this)
    }
    
    fun <R> map(transform: (T) -> R): Result<R> = when (this) {
        is Success -> runCatching { transform(data) }
        is Error -> this
    }
    
    fun <R> flatMap(transform: (T) -> Result<R>): Result<R> = when (this) {
        is Success -> try { transform(data) } catch (e: Exception) {
            error(e.message ?: "Unknown error", cause = e)
        }
        is Error -> this
    }
    
    fun onSuccess(action: (T) -> Unit): Result<T> {
        if (this is Success) action(data)
        return this
    }
    
    fun onError(action: (Error) -> Unit): Result<T> {
        if (this is Error) action(this)
        return this
    }
    
    fun recover(transform: (Error) -> @UnsafeVariance T): Result<T> = when (this) {
        is Success -> this
        is Error -> runCatching { transform(this) }
    }
}

// ตัวอย่าง: API Service ที่ใช้ Result
class ApiService {
    fun fetchUser(id: Int): Result<User> {
        return Result.runCatching {
            if (id <= 0) throw IllegalArgumentException("Invalid ID: $id")
            if (id == 999) throw RuntimeException("User not found")
            User(id.toLong(), "User $id", "user$id@example.com")
        }
    }
    
    fun fetchUserEmail(id: Int): Result<String> {
        return fetchUser(id).map { it.email }
    }
    
    fun fetchAndValidateUser(id: Int): Result<User> {
        return fetchUser(id).flatMap { user ->
            if (user.email.contains("@")) Result.success(user)
            else Result.error("Invalid email: ${user.email}")
        }
    }
}

fun main() {
    val service = ApiService()
    
    // Chain operations
    service.fetchUser(1)
        .onSuccess { println("Found: ${it.name}") }
        .onError { println("Error: ${it.message}") }
    // Found: User 1
    
    service.fetchUser(999)
        .onSuccess { println("Found: ${it.name}") }
        .onError { println("Error: ${it.message}") }
    // Error: User not found
    
    // map
    val emailResult = service.fetchUser(2).map { it.email }
    println(emailResult.getOrNull())  // user2@example.com
    
    // flatMap
    val validated = service.fetchAndValidateUser(3)
    println(validated.isSuccess)  // true
    
    // recover
    val withDefault = service.fetchUser(999)
        .recover { User(0L, "Default", "default@example.com") }
    println(withDefault.getOrNull()?.name)  // Default
    
    // chain
    val result = Result.runCatching { "42" }
        .map { it.toInt() }
        .map { it * 2 }
        .map { "Result is $it" }
    
    println(result.getOrNull())  // Result is 84
}
```

---

## 13.9 Generic Constraints ขั้นสูง

```kotlin
// Generic function กับ multiple constraints
fun <T> findBest(items: List<T>, comparator: Comparator<T>): T? {
    return items.minWithOrNull(comparator)
}

// Recursive generic types
data class TreeNode<T>(
    val value: T,
    val left: TreeNode<T>? = null,
    val right: TreeNode<T>? = null
)

fun <T : Comparable<T>> TreeNode<T>.insert(value: T): TreeNode<T> {
    return when {
        value < this.value -> copy(left = left?.insert(value) ?: TreeNode(value))
        value > this.value -> copy(right = right?.insert(value) ?: TreeNode(value))
        else -> this
    }
}

fun <T> TreeNode<T>.inOrder(): List<T> {
    return (left?.inOrder() ?: emptyList()) + listOf(value) + (right?.inOrder() ?: emptyList())
}

// Generic Builder pattern
class Builder<T : Any> {
    private val properties = mutableMapOf<String, Any?>()
    
    fun set(key: String, value: Any?): Builder<T> {
        properties[key] = value
        return this
    }
    
    @Suppress("UNCHECKED_CAST")
    fun <V> get(key: String): V? = properties[key] as? V
}

// Type-safe builder
@DslMarker
annotation class HtmlDsl

@HtmlDsl
class HtmlElement<T>(val tag: String) {
    val attributes = mutableMapOf<String, String>()
    val children = mutableListOf<HtmlElement<*>>()
    var text: String? = null
    
    fun attr(key: String, value: String) {
        attributes[key] = value
    }
    
    fun <C> child(tag: String, init: HtmlElement<C>.() -> Unit): HtmlElement<C> {
        val element = HtmlElement<C>(tag)
        element.init()
        children.add(element)
        return element
    }
    
    fun render(indent: Int = 0): String {
        val spaces = "  ".repeat(indent)
        val attrs = if (attributes.isEmpty()) ""
            else " " + attributes.entries.joinToString(" ") { "${it.key}=\"${it.value}\"" }
        return buildString {
            appendLine("$spaces<$tag$attrs>")
            text?.let { appendLine("$spaces  $it") }
            children.forEach { append(it.render(indent + 1)) }
            appendLine("$spaces</$tag>")
        }
    }
}

fun main() {
    // Binary search tree
    var tree: TreeNode<Int> = TreeNode(5)
    for (n in listOf(3, 7, 1, 4, 6, 8)) {
        tree = tree.insert(n)
    }
    println("In-order: ${tree.inOrder()}")  // [1, 3, 4, 5, 6, 7, 8]
}
```

---

## สรุปบทที่ 13

| Concept | ใช้งาน |
|---------|--------|
| Generic Functions | สร้างฟังก์ชันที่ทำงานกับหลาย types |
| Generic Classes | สร้าง class ที่รับ type parameter |
| Upper Bounds | จำกัด type ให้ต้องเป็น subtype ของบางสิ่ง |
| Covariant (out) | อ่านได้ (producer) - List<out T> |
| Contravariant (in) | เขียนได้ (consumer) - Comparator<in T> |
| Star Projection (*) | ใช้เมื่อไม่สนใจ type parameter |
| Reified | เข้าถึง type ตอน runtime ใน inline functions |

**กฎสำคัญ:**
- `out` = Producer = อ่านได้ = Covariant
- `in` = Consumer = เขียนได้ = Contravariant
- ไม่มี modifier = ทั้งอ่านและเขียน = Invariant
- `reified` ใช้ได้เฉพาะกับ `inline` functions

---

## แบบฝึกหัดบทที่ 13

### ระดับง่าย

1. สร้าง generic class `Queue<T>` ที่มี `enqueue`, `dequeue`, `peek`, `isEmpty`, `size`

2. เขียน generic function `zip3` ที่รับ 3 lists และคืน list of Triple:
   ```kotlin
   zip3(listOf(1,2), listOf("a","b"), listOf(true,false))
   // [(1, "a", true), (2, "b", false)]
   ```

3. สร้าง generic extension `List<T>.groupByKey(keySelector: (T) -> K): Map<K, List<T>>`

### ระดับกลาง

4. Implement generic `Cache<K, V>` ที่มี:
   - `put(key, value, ttlMs)`: เก็บค่าพร้อม expiry
   - `get(key)`: คืนค่าถ้ายังไม่หมดอายุ
   - `evict()`: ลบ expired entries
   - `size`: จำนวน active entries

5. สร้าง covariant `ReadableList<out T>` และ contravariant `WritableList<in T>` จากนั้นสร้าง `MutableTypedList<T>` ที่ implement ทั้งสอง

6. เขียน generic `ObservableList<T>` ที่แจ้งเตือน listeners เมื่อ list เปลี่ยนแปลง:
   ```kotlin
   val list = ObservableList<String>()
   list.addListener { change -> println("Changed: $change") }
   list.add("Hello")  // แจ้ง listener
   ```

### ระดับยาก

7. Implement generic `Trie<V>` (prefix tree) สำหรับ String keys:
   - `put(key: String, value: V)`
   - `get(key: String): V?`
   - `startsWith(prefix: String): List<Pair<String, V>>`
   - `delete(key: String): Boolean`

8. สร้าง type-safe event system:
   ```kotlin
   class EventBus {
       fun <T : Event> subscribe(eventType: KClass<T>, handler: (T) -> Unit)
       fun <T : Event> publish(event: T)
   }
   ```

---

[ไปต่อ Part 14: Coroutines Basic →](part-14-coroutines-basic.md)

---

*Part 13/100+ | Kotlin & Spring Boot Complete Course*
