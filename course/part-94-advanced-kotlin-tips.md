# Part 94: Advanced Kotlin Tips
## เทคนิค Kotlin ระดับสูงจากประสบการณ์จริง

---

## 🎯 เป้าหมายของ Part นี้

- Performance optimization tips
- Idiomatic Kotlin patterns
- Common pitfalls to avoid
- Kotlin 2.0 features
- Production experience tips

---

## ⚡ 1. Performance Tips

### Avoid Unnecessary Object Creation

```kotlin
// ❌ สร้าง object ใหม่ทุก call
fun processItems(items: List<String>): List<String> {
    return items.filter { it.isNotBlank() }
               .map { it.trim() }
               .map { it.uppercase() }
               // สร้าง intermediate List ทุก step!
}

// ✅ ใช้ sequence เพื่อ lazy evaluation
fun processItemsLazy(items: List<String>): List<String> {
    return items.asSequence()
               .filter { it.isNotBlank() }
               .map { it.trim() }
               .map { it.uppercase() }
               .toList()  // สร้าง List แค่ครั้งเดียวตอนสิ้นสุด
}

// เมื่อไหร่ใช้ sequence?
// - list มี 1000+ items
// - มี multiple operations (filter + map + filter...)
// - ไม่ต้องการ intermediate results
```

### Inline Functions

```kotlin
// inline ลด overhead ของ lambda calls
inline fun <T> measureTime(block: () -> T): Pair<T, Long> {
    val start = System.currentTimeMillis()
    val result = block()
    val elapsed = System.currentTimeMillis() - start
    return result to elapsed
}

// reified type parameter - ใช้ type ใน runtime
inline fun <reified T> parseJson(json: String): T {
    return objectMapper.readValue(json, T::class.java)
}

// ใช้งาน
val user: User = parseJson("""{"id":1,"name":"Alice"}""")
// ไม่ต้องระบุ type explicitly ใน argument
```

### Lazy Initialization

```kotlin
class ExpensiveService {
    // ✅ สร้างเมื่อ access ครั้งแรกเท่านั้น
    private val heavyCache: Map<String, Data> by lazy {
        println("Loading heavy cache...")
        loadFromDatabase()
    }

    // Thread-safe lazy (default)
    private val config by lazy(LazyThreadSafetyMode.SYNCHRONIZED) {
        loadConfiguration()
    }

    // None mode สำหรับ single-threaded (เร็วกว่า)
    private val cache by lazy(LazyThreadSafetyMode.NONE) {
        HashMap<String, String>()
    }
}
```

---

## 🎨 2. Idiomatic Kotlin

### Sealed Classes กับ When

```kotlin
// ✅ exhaustive when - compiler บังคับ handle ทุก case
sealed class ApiResult<out T> {
    data class Success<T>(val data: T) : ApiResult<T>()
    data class Error(val message: String, val code: Int) : ApiResult<Nothing>()
    data object Loading : ApiResult<Nothing>()
}

fun handleResult(result: ApiResult<User>) = when (result) {
    is ApiResult.Success -> println("User: ${result.data}")
    is ApiResult.Error -> println("Error ${result.code}: ${result.message}")
    is ApiResult.Loading -> println("Loading...")
    // ไม่ต้อง else เพราะ sealed class exhaustive!
}
```

### Extension Functions ที่ elegant

```kotlin
// ✅ Fluent API กับ Extension Functions
fun String.toSlug(): String =
    this.lowercase()
        .replace(Regex("[^a-z0-9\\s-]"), "")
        .replace(Regex("\\s+"), "-")
        .trim('-')

fun LocalDate.isWeekend(): Boolean =
    dayOfWeek == DayOfWeek.SATURDAY || dayOfWeek == DayOfWeek.SUNDAY

fun <T> List<T>.second(): T {
    if (size < 2) throw IndexOutOfBoundsException("List has less than 2 elements")
    return this[1]
}

// Extension properties
val String.words: List<String>
    get() = trim().split(Regex("\\s+")).filter { it.isNotEmpty() }

val <T> List<T>.penultimate: T
    get() = this[size - 2]
```

### Scope Functions อย่างถูกต้อง

```kotlin
// let: เมื่อต้องการ null check + transform
val result = maybeNull?.let { value ->
    transform(value)
}

// run: เมื่อต้องการ compute result บน object
val config = run {
    val base = loadBaseConfig()
    val env = loadEnvConfig()
    base.merge(env)  // returns merged config
}

// apply: เมื่อต้องการ configure object + return ตัวเอง
val request = HttpRequest.Builder()
    .apply {
        url("https://api.example.com")
        header("Authorization", "Bearer token")
        timeout(30, TimeUnit.SECONDS)
    }
    .build()

// also: side effect + return original
val user = createUser()
    .also { println("Created user: ${it.id}") }
    .also { emailService.sendWelcome(it) }

// with: เมื่อต้องการ call multiple methods บน non-null object
with(userRepository) {
    save(user1)
    save(user2)
    flush()
}
```

---

## 🚫 3. Common Pitfalls

### Mutable Shared State

```kotlin
// ❌ Mutable shared state - อันตรายใน concurrent environment
object CounterService {
    var count = 0  // race condition!
    fun increment() { count++ }
}

// ✅ Thread-safe alternatives
object SafeCounterService {
    private val count = AtomicLong(0)
    fun increment(): Long = count.incrementAndGet()
    fun get(): Long = count.get()
}

// ✅ หรือใช้ immutable data
data class AppState(val count: Int = 0) {
    fun increment() = copy(count = count + 1)
}
```

### Coroutine Context Mistakes

```kotlin
// ❌ blocking ใน coroutine
suspend fun fetchUserBad(id: Long): User {
    Thread.sleep(1000)  // blocks thread! ❌
    return repository.findById(id)
}

// ✅ ถูกต้อง
suspend fun fetchUser(id: Long): User {
    delay(1000)  // non-blocking ✅
    return withContext(Dispatchers.IO) {  // IO dispatcher สำหรับ blocking calls
        repository.findById(id)
    }
}

// ❌ ใช้ GlobalScope - memory leak!
fun startWork() {
    GlobalScope.launch {  // ❌ ไม่มี lifecycle management
        doWork()
    }
}

// ✅ ใช้ structured concurrency
class MyService(private val scope: CoroutineScope) {
    fun startWork() {
        scope.launch {  // ✅ tied to service lifecycle
            doWork()
        }
    }
}
```

### Data Class Gotchas

```kotlin
// ❌ Data class กับ mutable collections
data class UserBad(
    val name: String,
    val roles: MutableList<String>  // ❌ mutable!
)

val user1 = UserBad("Alice", mutableListOf("USER"))
val user2 = user1.copy()
user2.roles.add("ADMIN")  // user1 ก็ได้รับผลด้วย! (shared reference)

// ✅ ใช้ immutable collections
data class UserGood(
    val name: String,
    val roles: List<String>  // ✅ immutable
)

// ❌ Inheritance กับ data class
open class BaseBad(val id: Long)
data class UserDerived(val name: String) : BaseBad(1)  // ❌ ไม่แนะนำ

// ✅ ใช้ composition แทน
data class User(
    val base: BaseEntity,
    val name: String
)
```

---

## 🆕 4. Kotlin 2.0 Features

### Smart Casts improvements

```kotlin
// Kotlin 2.0: Smart cast ทำงานในกรณีที่ 1.x ทำไม่ได้
class Container {
    var value: Any = "Initial"
}

// Kotlin 1.x: ต้อง cast manually
fun processOld(container: Container) {
    val v = container.value
    if (v is String) {
        println(v.length)  // OK ใน local val
    }
    if (container.value is String) {
        // container.value.length  // ❌ ใน 1.x เพราะ var อาจเปลี่ยน
        (container.value as String).length  // ต้อง explicit cast
    }
}

// Kotlin 2.0: better smart cast
fun processNew(container: Container) {
    if (container.value is String) {
        println(container.value.length)  // ✅ สามารถ smart cast ได้ใน 2.0
    }
}
```

### K2 Compiler

```kotlin
// K2 compiler ใน Kotlin 2.0
// - เร็วกว่า K1 ประมาณ 2x
// - Better error messages
// - More stable

// build.gradle.kts
kotlin {
    // Enable K2 (default ใน Kotlin 2.0)
    compilerOptions {
        languageVersion.set(org.jetbrains.kotlin.gradle.dsl.KotlinVersion.KOTLIN_2_0)
    }
}
```

### Context Parameters (Kotlin 2.1+)

```kotlin
// Context parameters - ทดแทน extension receivers
context(Transaction)
fun saveUser(user: User) {
    // ใช้ transaction context โดยปริยาย
    execute("INSERT INTO users...")
}

// เรียกใช้
withTransaction {
    saveUser(user)  // context อยู่ใน scope อัตโนมัติ
}
```

---

## 💡 5. Production Tips

### Logging ที่ถูกต้อง

```kotlin
// ✅ ใช้ companion object สำหรับ logger
class UserService {
    companion object {
        private val log = LoggerFactory.getLogger(UserService::class.java)
    }

    fun createUser(request: CreateUserRequest): User {
        log.info("Creating user: email={}", request.email)  // ไม่ใช้ string interpolation!

        return try {
            val user = userRepository.save(request.toEntity())
            log.info("User created: id={}", user.id)
            user
        } catch (e: Exception) {
            log.error("Failed to create user: email={}", request.email, e)
            throw e
        }
    }
}

// ✅ Kotlin extension สำหรับ clean logging
private val Any.log get() = LoggerFactory.getLogger(this::class.java)

class SomeService {
    fun doWork() {
        log.info("Working...")
    }
}
```

### Configuration Properties ที่ clean

```kotlin
// ✅ Type-safe configuration
@ConfigurationProperties(prefix = "app")
@Validated
data class AppProperties(
    val database: DatabaseProperties,
    val cache: CacheProperties,
    val security: SecurityProperties
) {
    data class DatabaseProperties(
        @field:NotBlank val url: String,
        @field:Min(1) @field:Max(100) val poolSize: Int = 10,
        val slowQueryThresholdMs: Long = 1000
    )

    data class CacheProperties(
        val defaultTtlSeconds: Long = 300,
        val maxSize: Long = 1000
    )

    data class SecurityProperties(
        @field:NotBlank val jwtSecret: String,
        @field:Positive val jwtExpiryHours: Int = 24
    )
}
```

### Error Handling Pattern

```kotlin
// ✅ Result type สำหรับ expected failures
sealed class Result<out T> {
    data class Success<T>(val value: T) : Result<T>()
    data class Failure(val error: AppError) : Result<Nothing>()

    fun getOrNull(): T? = (this as? Success)?.value
    fun getOrElse(default: T): T = getOrNull() ?: default
    fun isSuccess(): Boolean = this is Success
}

// Custom errors
sealed class AppError {
    data class NotFound(val resource: String, val id: Any) : AppError()
    data class ValidationError(val field: String, val message: String) : AppError()
    data class Unauthorized(val reason: String) : AppError()
}

// ใช้งาน
fun findUser(id: Long): Result<User> {
    val user = userRepository.findById(id).orElse(null)
        ?: return Result.Failure(AppError.NotFound("User", id))
    return Result.Success(user)
}

// Call site ชัดเจน
when (val result = findUser(123)) {
    is Result.Success -> println("Found: ${result.value.name}")
    is Result.Failure -> when (val error = result.error) {
        is AppError.NotFound -> println("Not found: ${error.resource}")
        is AppError.ValidationError -> println("Validation: ${error.message}")
        is AppError.Unauthorized -> println("Unauthorized: ${error.reason}")
    }
}
```

---

## 📋 สรุป

| หัวข้อ | Tip |
|--------|-----|
| Performance | ใช้ sequence สำหรับ large collections + multiple ops |
| Inline | ลด lambda overhead สำหรับ high-frequency calls |
| Scope Functions | ใช้ให้ถูก purpose (let/run/apply/also/with) |
| Coroutines | ใช้ structured concurrency, ไม่ block ใน suspend |
| Data Classes | Immutable collections เท่านั้น |
| Kotlin 2.0 | K2 compiler เร็วกว่า 2x, better smart casts |
| Logging | lazy string, ไม่ใช้ string interpolation |
| Configuration | @ConfigurationProperties แทน @Value |

---

*Part 94/100+ | Kotlin & Spring Boot Complete Course*
