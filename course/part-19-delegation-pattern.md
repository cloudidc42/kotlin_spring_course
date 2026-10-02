# Part 19: Delegation Pattern
## Delegation by Keyword - Interface & Property Delegation

---

## 🎯 เป้าหมายของ Part นี้

- เข้าใจ Delegation pattern
- ใช้ `by` keyword สำหรับ interface delegation
- Property delegation (lazy, observable, vetoable)
- สร้าง custom property delegates
- Map-backed properties

---

## 🔑 1. Interface Delegation (by keyword)

```kotlin
interface Printer {
    fun print(text: String)
    fun printLine(text: String) = println(text)
}

interface Scanner {
    fun scan(): String
}

// Implementation
class ConsolePrinter : Printer {
    override fun print(text: String) = kotlin.io.print(text)
}

class KeyboardScanner : Scanner {
    override fun scan() = readLine() ?: ""
}

// Delegation: PrinterScanner ใช้งาน ConsolePrinter และ KeyboardScanner
// โดยไม่ต้อง implement เอง!
class PrinterScanner(
    printer: Printer = ConsolePrinter(),
    scanner: Scanner = KeyboardScanner()
) : Printer by printer, Scanner by scanner {
    fun printAndScan(prompt: String): String {
        print(prompt)
        return scan()
    }
}

fun main() {
    val device = PrinterScanner()
    device.printLine("=== Printer/Scanner Device ===")
    // พิมพ์ผ่าน ConsolePrinter
}
```

### Delegation เพื่อ compose behaviors

```kotlin
interface Logger {
    fun log(message: String)
    fun logError(message: String) = log("[ERROR] $message")
    fun logInfo(message: String) = log("[INFO] $message")
}

interface Cache<K, V> {
    fun get(key: K): V?
    fun put(key: K, value: V)
    fun invalidate(key: K)
}

class ConsoleLogger : Logger {
    override fun log(message: String) = println(message)
}

class InMemoryCache<K, V> : Cache<K, V> {
    private val store = HashMap<K, V>()
    override fun get(key: K) = store[key]
    override fun put(key: K, value: V) { store[key] = value }
    override fun invalidate(key: K) { store.remove(key) }
}

// UserRepository ใช้ทั้ง Logger และ Cache ผ่าน delegation
class UserRepository(
    private val logger: Logger = ConsoleLogger(),
    private val cache: Cache<Int, String> = InMemoryCache()
) : Logger by logger, Cache<Int, String> by cache {
    
    fun findUser(id: Int): String? {
        val cached = get(id)
        if (cached != null) {
            logInfo("Cache hit for user $id")
            return cached
        }
        
        logInfo("Loading user $id from database...")
        // simulate DB load
        val user = "User-$id"
        put(id, user)
        return user
    }
}

fun main() {
    val repo = UserRepository()
    println(repo.findUser(1))  // โหลดจาก DB
    println(repo.findUser(1))  // จาก Cache
    println(repo.findUser(2))  // โหลดจาก DB
}
```

---

## 🏠 2. Property Delegation พื้นฐาน

### lazy - โหลดครั้งแรกเท่านั้น

```kotlin
class HeavyResource {
    val data: List<Int> by lazy {
        println("Loading heavy data... (only once)")
        (1..1_000_000).toList()
    }
    
    val summary: String by lazy {
        "Count: ${data.size}, Sum: ${data.sum()}"
    }
}

fun main() {
    val resource = HeavyResource()
    println("Resource created, not loaded yet")
    
    println(resource.summary)  // ตรงนี้จึงโหลด
    // Loading heavy data... (only once)
    // Count: 1000000, Sum: 500000500000
    
    println(resource.summary)  // ไม่โหลดซ้ำ
    // Count: 1000000, Sum: 500000500000
}
```

### lazy modes

```kotlin
// Thread-safe (default)
val safeData by lazy(LazyThreadSafetyMode.SYNCHRONIZED) {
    loadFromDatabase()
}

// Not thread-safe (faster, single-threaded)
val fastData by lazy(LazyThreadSafetyMode.NONE) {
    computeLocalValue()
}

// Published (safe, multiple threads can initialize)
val publishedData by lazy(LazyThreadSafetyMode.PUBLICATION) {
    loadConfig()
}
```

### observable - สังเกตการเปลี่ยนแปลง

```kotlin
import kotlin.properties.Delegates

class ViewModel {
    var count: Int by Delegates.observable(0) { prop, old, new ->
        println("${prop.name}: $old → $new")
        onCountChanged(old, new)
    }
    
    var text: String by Delegates.observable("") { _, old, new ->
        if (old != new) notifyTextChanged()
    }
    
    private fun onCountChanged(old: Int, new: Int) {
        if (new > old * 2) println("Warning: count more than doubled!")
    }
    
    private fun notifyTextChanged() {
        println("Text updated: '$text'")
    }
}

fun main() {
    val vm = ViewModel()
    vm.count = 5
    vm.count = 6
    vm.count = 15  // triggers warning
    vm.text = "Hello"
    vm.text = "Hello"  // no change
    vm.text = "World"
}
```

### vetoable - ป้องกันการเปลี่ยนค่าที่ไม่ถูกต้อง

```kotlin
import kotlin.properties.Delegates

class FormField(val fieldName: String) {
    var value: String by Delegates.vetoable("") { _, _, new ->
        val isValid = new.isNotBlank() && new.length <= 100
        if (!isValid) println("Rejected: '$new' for $fieldName")
        isValid
    }
    
    var age: Int by Delegates.vetoable(0) { _, _, new ->
        new in 0..150
    }
}

fun main() {
    val field = FormField("username")
    field.value = "alice"
    println(field.value)   // alice
    
    field.value = ""       // Rejected
    println(field.value)   // alice (unchanged)
    
    field.value = "A".repeat(200)  // Rejected (too long)
    println(field.value)   // alice (unchanged)
    
    field.age = 25
    println(field.age)    // 25
    field.age = 200       // Rejected
    println(field.age)    // 25 (unchanged)
}
```

---

## 🛠️ 3. Custom Property Delegates

```kotlin
import kotlin.reflect.KProperty

// Simple delegate
class UppercaseDelegate {
    private var value = ""
    
    operator fun getValue(thisRef: Any?, property: KProperty<*>): String {
        return value
    }
    
    operator fun setValue(thisRef: Any?, property: KProperty<*>, newValue: String) {
        value = newValue.uppercase()
    }
}

class Person {
    var name: String by UppercaseDelegate()
    var city: String by UppercaseDelegate()
}

fun main() {
    val p = Person()
    p.name = "alice"
    p.city = "bangkok"
    println(p.name)   // ALICE
    println(p.city)   // BANGKOK
}

// Generic constraint delegate
class RangeDelegate<T : Comparable<T>>(
    private val range: ClosedRange<T>,
    private var default: T
) {
    private var value: T = default
    
    operator fun getValue(thisRef: Any?, property: KProperty<*>): T = value
    
    operator fun setValue(thisRef: Any?, property: KProperty<*>, newValue: T) {
        value = if (newValue in range) newValue else {
            println("${property.name}: $newValue out of range $range, keeping $value")
            value
        }
    }
}

class Sensor {
    var temperature: Double by RangeDelegate(-40.0..100.0, 25.0)
    var humidity: Int by RangeDelegate(0..100, 50)
    var pressure: Double by RangeDelegate(900.0..1100.0, 1013.25)
}

fun main2() {
    val sensor = Sensor()
    sensor.temperature = 30.0
    sensor.temperature = 150.0  // out of range
    println("Temp: ${sensor.temperature}")  // 30.0
    
    sensor.humidity = 65
    sensor.humidity = 120       // out of range
    println("Humidity: ${sensor.humidity}%")  // 65
}
```

### Logging Delegate

```kotlin
import kotlin.reflect.KProperty

class LoggingDelegate<T>(initialValue: T) {
    private var value: T = initialValue
    private val history = mutableListOf<Pair<T, Long>>()
    
    operator fun getValue(thisRef: Any?, property: KProperty<*>): T {
        println("[GET] ${property.name} = $value")
        return value
    }
    
    operator fun setValue(thisRef: Any?, property: KProperty<*>, newValue: T) {
        println("[SET] ${property.name}: $value → $newValue")
        history.add(Pair(value, System.currentTimeMillis()))
        value = newValue
    }
    
    fun getHistory() = history.toList()
}

class Config {
    var serverUrl: String by LoggingDelegate("http://localhost")
    var port: Int by LoggingDelegate(8080)
}

fun main3() {
    val config = Config()
    println(config.serverUrl)
    config.serverUrl = "https://api.example.com"
    config.port = 443
    println(config.port)
}
```

---

## 🗺️ 4. Map-backed Properties

```kotlin
// Properties จาก Map - ดีสำหรับ JSON/config
class UserConfig(private val map: Map<String, Any>) {
    val name: String by map
    val age: Int by map
    val email: String by map
    val isAdmin: Boolean by map
}

class MutableUserConfig(private val map: MutableMap<String, Any> = mutableMapOf()) {
    var name: String by map
    var age: Int by map
    var email: String by map
    
    fun toMap() = map.toMap()
}

fun main() {
    // Immutable
    val config = UserConfig(
        mapOf(
            "name" to "Alice",
            "age" to 25,
            "email" to "alice@example.com",
            "isAdmin" to true
        )
    )
    println("${config.name}, ${config.age}, ${config.email}")
    
    // Mutable
    val mutableConfig = MutableUserConfig()
    mutableConfig.name = "Bob"
    mutableConfig.age = 30
    mutableConfig.email = "bob@example.com"
    
    println(mutableConfig.toMap())
    // {name=Bob, age=30, email=bob@example.com}
}
```

---

## 🏋️ 5. แบบฝึกหัด

### ข้อ 1: Cache delegate
```kotlin
import kotlin.reflect.KProperty

class CachedDelegate<T>(private val compute: () -> T, private val ttlMs: Long = 5000L) {
    private var cachedValue: T? = null
    private var lastComputed: Long = 0L
    
    operator fun getValue(thisRef: Any?, property: KProperty<*>): T {
        val now = System.currentTimeMillis()
        if (cachedValue == null || now - lastComputed > ttlMs) {
            println("Computing ${property.name}...")
            cachedValue = compute()
            lastComputed = now
        }
        return cachedValue!!
    }
}

fun <T> cached(ttlMs: Long = 5000L, compute: () -> T) = CachedDelegate(compute, ttlMs)

class DataService {
    val expensiveData: List<Int> by cached(ttlMs = 1000L) {
        Thread.sleep(100)  // simulate expensive computation
        (1..10).toList()
    }
}

fun main() {
    val service = DataService()
    println(service.expensiveData)  // Computing...
    println(service.expensiveData)  // From cache
    Thread.sleep(1100)
    println(service.expensiveData)  // Computing again (expired)
}
```

### ข้อ 2: Validated delegate
```kotlin
import kotlin.reflect.KProperty

fun validated(vararg validators: (String) -> String?): ReadWriteProperty<Any, String> {
    return object : ReadWriteProperty<Any, String> {
        private var value = ""
        
        override fun getValue(thisRef: Any, property: KProperty<*>) = value
        
        override fun setValue(thisRef: Any, property: KProperty<*>, newValue: String) {
            for (validate in validators) {
                val error = validate(newValue)
                if (error != null) throw IllegalArgumentException("${property.name}: $error")
            }
            value = newValue
        }
    }
}

import kotlin.properties.ReadWriteProperty

class RegistrationForm {
    var email: String by validated(
        { if (!it.contains("@")) "Invalid email" else null },
        { if (it.length > 200) "Email too long" else null }
    )
    
    var password: String by validated(
        { if (it.length < 8) "Password too short" else null },
        { if (!it.any { c -> c.isDigit() }) "Password must contain digit" else null }
    )
}

fun main() {
    val form = RegistrationForm()
    try {
        form.email = "alice@example.com"
        form.password = "password123"
        println("Form valid!")
    } catch (e: IllegalArgumentException) {
        println("Error: ${e.message}")
    }
}
```

---

## 📝 สรุป Part 19

| Delegation | ตัวอย่าง | ใช้เมื่อ |
|-----------|---------|---------|
| Interface delegation | `class X : I by impl` | Composition over inheritance |
| `lazy` | `val x by lazy { }` | โหลดครั้งแรก |
| `observable` | `var x by Delegates.observable` | React to changes |
| `vetoable` | `var x by Delegates.vetoable` | Validate changes |
| Custom delegate | `class D { getValue/setValue }` | Custom logic |
| Map-backed | `val x: Type by map` | Dynamic properties |

---

## ➡️ ถัดไป: Part 20 - ทบทวนและ Mini Project

---
*Part 19/100+ | Kotlin & Spring Boot Complete Course*
