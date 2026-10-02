# Part 08: Data Classes, Sealed Classes และ Enum
## เจาะลึก Data Class, Sealed Class และ Enum ใน Kotlin

---

## 🎯 เป้าหมายของ Part นี้

- เข้าใจ data class แบบลึก: copy, destructuring, componentN
- ใช้ sealed class patterns อย่างมืออาชีพ
- สร้าง enum class พร้อม properties และ functions
- ใช้ enum กับ `when` expression
- สร้าง Result/Either pattern ด้วย sealed class
- ออกแบบ State Machine ด้วย sealed class
- สร้าง API Response pattern

---

## 📦 1. Data Class แบบลึก

`data class` คือ class พิเศษที่ Kotlin generate methods ให้อัตโนมัติ: `equals()`, `hashCode()`, `toString()`, `copy()`, และ `componentN()` functions

### การสร้างและ Auto-generated Methods

```kotlin
data class Point(val x: Double, val y: Double)

fun main() {
    val p1 = Point(3.0, 4.0)
    val p2 = Point(3.0, 4.0)
    val p3 = Point(1.0, 2.0)
    
    // toString() - auto generated
    println(p1)         // Point(x=3.0, y=4.0)
    
    // equals() - compares by value (not reference)
    println(p1 == p2)   // true
    println(p1 == p3)   // false
    println(p1 === p2)  // false (different objects!)
    
    // hashCode() - consistent with equals
    val set = setOf(p1, p2, p3)
    println(set.size)   // 2 (p1 และ p2 เป็น "เหมือนกัน")
    
    val map = mapOf(p1 to "first", p3 to "third")
    println(map[p2])    // "first" (เพราะ p2 == p1)
}
```

### copy() - สร้าง Object ใหม่พร้อมเปลี่ยนบาง Fields

```kotlin
data class User(
    val id: Long,
    val name: String,
    val email: String,
    val age: Int,
    val isActive: Boolean = true,
    val roles: List<String> = listOf("user")
)

fun main() {
    val alice = User(1, "Alice", "alice@example.com", 30)
    println(alice)
    // User(id=1, name=Alice, email=alice@example.com, age=30, isActive=true, roles=[user])
    
    // copy() เปลี่ยนแค่ email
    val aliceNewEmail = alice.copy(email = "alice.new@example.com")
    println(aliceNewEmail)
    
    // copy() เปลี่ยนหลาย fields พร้อมกัน
    val aliceAdmin = alice.copy(
        age = 31,
        roles = listOf("user", "admin")
    )
    println(aliceAdmin)
    
    // Original ไม่เปลี่ยน!
    println(alice.email)  // alice@example.com (unchanged)
    
    // copy() useful มากใน immutable data patterns
    val users = listOf(alice)
    val updatedUsers = users.map { user ->
        if (user.id == 1L) user.copy(isActive = false) else user
    }
    println(updatedUsers[0].isActive)  // false
}
```

### Destructuring (การ Unpack)

```kotlin
data class Point3D(val x: Double, val y: Double, val z: Double)

data class Employee(
    val id: Int,
    val name: String,
    val department: String,
    val salary: Double
)

fun main() {
    val point = Point3D(1.0, 2.0, 3.0)
    
    // Destructuring declaration
    val (x, y, z) = point
    println("x=$x, y=$y, z=$z")  // x=1.0, y=2.0, z=3.0
    
    // ข้ามบาง fields ด้วย underscore
    val (_, _, zOnly) = point
    println("z=$zOnly")  // z=3.0
    
    // ใช้ใน for loop
    val employees = listOf(
        Employee(1, "Alice", "Engineering", 75000.0),
        Employee(2, "Bob", "Marketing", 65000.0),
        Employee(3, "Charlie", "Engineering", 80000.0)
    )
    
    for ((id, name, dept, salary) in employees) {
        println("[$id] $name ($dept): ${"$%.2f".format(salary)}")
    }
    
    // ใช้ใน lambda
    employees.forEach { (id, name, _, salary) ->
        if (salary > 70000) println("High earner: $name (#$id)")
    }
    
    // ใช้กับ Map.Entry
    val scores = mapOf("Alice" to 95, "Bob" to 87, "Charlie" to 92)
    for ((name, score) in scores) {
        println("$name: $score")
    }
}
```

### componentN() Functions

Destructuring ทำงานผ่าน `component1()`, `component2()`, ... ที่ data class generate ให้

```kotlin
data class RGB(val red: Int, val green: Int, val blue: Int) {
    val hex: String get() = "#%02X%02X%02X".format(red, green, blue)
    
    // Kotlin generates these automatically for data class:
    // fun component1() = red
    // fun component2() = green
    // fun component3() = blue
}

// Custom class ก็สามารถ support destructuring ได้ด้วย operator functions
class Pair2D(val first: Double, val second: Double) {
    operator fun component1() = first
    operator fun component2() = second
}

fun main() {
    val color = RGB(255, 128, 0)
    
    // ใช้ component functions โดยตรง
    println(color.component1())  // 255
    println(color.component2())  // 128
    println(color.component3())  // 0
    
    // หรือใช้ destructuring (สวยกว่า)
    val (r, g, b) = color
    println("RGB: $r, $g, $b")  // RGB: 255, 128, 0
    println("Hex: ${color.hex}")  // Hex: #FF8000
    
    // Custom class
    val pair = Pair2D(3.0, 4.0)
    val (x, y) = pair
    println("Distance from origin: ${"%.2f".format(Math.sqrt(x*x + y*y))}")
}
```

### Data Class กับ Inheritance

```kotlin
// Data class ไม่สามารถ inherit จาก data class ได้
// แต่สามารถ inherit จาก open class หรือ implement interface ได้

interface Entity {
    val id: Long
    val createdAt: Long
}

abstract class BaseEntity(
    override val id: Long,
    override val createdAt: Long = System.currentTimeMillis()
) : Entity

data class Product(
    override val id: Long,
    val name: String,
    val price: Double,
    val category: String,
    override val createdAt: Long = System.currentTimeMillis()
) : BaseEntity(id, createdAt)

data class Order(
    override val id: Long,
    val userId: Long,
    val items: List<OrderItem>,
    val status: String = "PENDING",
    override val createdAt: Long = System.currentTimeMillis()
) : BaseEntity(id, createdAt) {
    val total: Double get() = items.sumOf { it.price * it.quantity }
}

data class OrderItem(
    val productId: Long,
    val name: String,
    val price: Double,
    val quantity: Int
)

fun main() {
    val items = listOf(
        OrderItem(1, "Laptop", 999.99, 1),
        OrderItem(2, "Mouse", 29.99, 2)
    )
    
    val order = Order(1001L, userId = 42L, items = items)
    println("Order #${order.id}: Total = ${"$%.2f".format(order.total)}")
    // Order #1001: Total = $1059.97
    
    // update status via copy
    val completedOrder = order.copy(status = "COMPLETED")
    println("Status: ${completedOrder.status}")
}
```

### ข้อควรระวังกับ data class

```kotlin
// ❌ Mutable collections ใน data class - อันตราย!
data class BadDesign(val items: MutableList<String>)

fun main() {
    val a = BadDesign(mutableListOf("apple"))
    val b = a.copy()  // b ชี้ไปที่ list เดียวกัน!
    
    b.items.add("banana")
    println(a.items)  // [apple, banana] - a เปลี่ยนไปด้วย!
    
    // ✅ ใช้ immutable list แทน
    data class GoodDesign(val items: List<String>)
    
    val c = GoodDesign(listOf("apple"))
    val d = c.copy(items = c.items + "banana")  // สร้าง list ใหม่
    println(c.items)  // [apple] - ไม่เปลี่ยน!
    println(d.items)  // [apple, banana]
}
```

---

## 🔒 2. Sealed Class Patterns แบบลึก

### Pattern 1: Result Pattern

```kotlin
sealed class Result<out T> {
    data class Success<T>(val data: T) : Result<T>()
    
    data class Failure(
        val message: String,
        val cause: Throwable? = null,
        val code: Int = -1
    ) : Result<Nothing>()
    
    object Empty : Result<Nothing>()
    
    // Helper functions บน sealed class
    val isSuccess: Boolean get() = this is Success
    val isFailure: Boolean get() = this is Failure
    
    fun getOrNull(): T? = when (this) {
        is Success -> data
        else -> null
    }
    
    fun getOrDefault(default: @UnsafeVariance T): T = when (this) {
        is Success -> data
        else -> default
    }
    
    fun getOrThrow(): T = when (this) {
        is Success -> data
        is Failure -> throw RuntimeException(message, cause)
        Empty -> throw NoSuchElementException("Result is empty")
    }
    
    fun <R> map(transform: (T) -> R): Result<R> = when (this) {
        is Success -> try {
            Success(transform(data))
        } catch (e: Exception) {
            Failure(e.message ?: "Transform failed", e)
        }
        is Failure -> this
        Empty -> Empty
    }
    
    fun <R> flatMap(transform: (T) -> Result<R>): Result<R> = when (this) {
        is Success -> try {
            transform(data)
        } catch (e: Exception) {
            Failure(e.message ?: "FlatMap failed", e)
        }
        is Failure -> this
        Empty -> Empty
    }
    
    fun onSuccess(action: (T) -> Unit): Result<T> {
        if (this is Success) action(data)
        return this
    }
    
    fun onFailure(action: (Failure) -> Unit): Result<T> {
        if (this is Failure) action(this)
        return this
    }
}

// ฟังก์ชัน helper สำหรับ try-catch
inline fun <T> runCatching(block: () -> T): Result<T> {
    return try {
        Result.Success(block())
    } catch (e: Exception) {
        Result.Failure(e.message ?: "Unknown error", e)
    }
}

// ตัวอย่างการใช้งาน
data class UserProfile(val id: Int, val name: String, val email: String)

object UserRepository {
    private val users = mapOf(
        1 to UserProfile(1, "Alice", "alice@example.com"),
        2 to UserProfile(2, "Bob", "bob@example.com")
    )
    
    fun findById(id: Int): Result<UserProfile> {
        return if (id <= 0) {
            Result.Failure("Invalid ID: $id", code = 400)
        } else {
            users[id]?.let { Result.Success(it) }
                ?: Result.Failure("User not found: $id", code = 404)
        }
    }
    
    fun findAll(): Result<List<UserProfile>> = Result.Success(users.values.toList())
}

fun main() {
    // Chain operations
    UserRepository.findById(1)
        .map { it.copy(name = it.name.uppercase()) }
        .onSuccess { println("Found: $it") }
        .onFailure { println("Error: ${it.message}") }
    
    UserRepository.findById(99)
        .onSuccess { println("Found: $it") }
        .onFailure { println("Error ${it.code}: ${it.message}") }
    
    // getOrDefault
    val name = UserRepository.findById(1)
        .map { it.name }
        .getOrDefault("Unknown")
    println("Name: $name")
    
    // flatMap - chain multiple operations that can fail
    val result = UserRepository.findById(1)
        .flatMap { user ->
            if (user.email.contains("@")) {
                Result.Success(user.email.split("@")[1])  // get domain
            } else {
                Result.Failure("Invalid email format")
            }
        }
    println("Email domain: ${result.getOrNull()}")  // example.com
}
```

### Pattern 2: Either Pattern

```kotlin
sealed class Either<out L, out R> {
    data class Left<L>(val value: L) : Either<L, Nothing>()
    data class Right<R>(val value: R) : Either<Nothing, R>()
    
    val isLeft: Boolean get() = this is Left
    val isRight: Boolean get() = this is Right
    
    fun <T> fold(onLeft: (L) -> T, onRight: (R) -> T): T = when (this) {
        is Left -> onLeft(value)
        is Right -> onRight(value)
    }
    
    fun <T> mapRight(transform: (R) -> T): Either<L, T> = when (this) {
        is Left -> this
        is Right -> Right(transform(value))
    }
}

// ใช้ Left สำหรับ error, Right สำหรับ success (convention)
typealias AppError = Either.Left<String>
typealias Success<T> = Either.Right<T>

fun divide(a: Double, b: Double): Either<String, Double> {
    return if (b == 0.0) {
        Either.Left("Division by zero!")
    } else {
        Either.Right(a / b)
    }
}

fun sqrt(n: Double): Either<String, Double> {
    return if (n < 0) {
        Either.Left("Cannot take sqrt of negative number")
    } else {
        Either.Right(Math.sqrt(n))
    }
}

fun main() {
    divide(10.0, 2.0).fold(
        onLeft = { println("Error: $it") },
        onRight = { println("Result: $it") }
    )  // Result: 5.0
    
    divide(10.0, 0.0).fold(
        onLeft = { println("Error: $it") },
        onRight = { println("Result: $it") }
    )  // Error: Division by zero!
    
    // Chain operations
    divide(16.0, 4.0)
        .mapRight { it - 1.0 }   // 4.0 - 1.0 = 3.0
        .mapRight { it * it }     // 3.0 * 3.0 = 9.0
        .fold(
            onLeft = { println("Error: $it") },
            onRight = { println("Final: $it") }
        )  // Final: 9.0
}
```

---

## 🎯 3. Enum Class พร้อม Properties และ Functions

### Enum พื้นฐาน

```kotlin
enum class Direction {
    NORTH, SOUTH, EAST, WEST;
    
    // Method ใน enum
    fun opposite(): Direction = when (this) {
        NORTH -> SOUTH
        SOUTH -> NORTH
        EAST -> WEST
        WEST -> EAST
    }
    
    fun rotate90Clockwise(): Direction = when (this) {
        NORTH -> EAST
        EAST -> SOUTH
        SOUTH -> WEST
        WEST -> NORTH
    }
}

fun main() {
    val dir = Direction.NORTH
    println(dir)                    // NORTH
    println(dir.name)               // NORTH
    println(dir.ordinal)            // 0
    println(dir.opposite())         // SOUTH
    println(dir.rotate90Clockwise()) // EAST
    
    // Iterate ทุก enum values
    Direction.values().forEach { println("${it.ordinal}: $it") }
    
    // หา enum จาก string
    val south = Direction.valueOf("SOUTH")
    println(south)  // SOUTH
    
    // enumValues<>() และ enumValueOf<>()
    val all = enumValues<Direction>()
    println(all.toList())  // [NORTH, SOUTH, EAST, WEST]
}
```

### Enum กับ Properties

```kotlin
enum class Planet(
    val mass: Double,        // kg (× 10²⁴)
    val radius: Double,      // km
    val distanceFromSun: Double  // AU (Astronomical Units)
) {
    MERCURY(0.330, 2_439.7, 0.387),
    VENUS(4.868, 6_051.8, 0.723),
    EARTH(5.972, 6_371.0, 1.000),
    MARS(0.642, 3_389.5, 1.524),
    JUPITER(1898.19, 69_911.0, 5.203),
    SATURN(568.34, 58_232.0, 9.537),
    URANUS(86.813, 25_362.0, 19.191),
    NEPTUNE(102.413, 24_622.0, 30.069);
    
    // Derived properties
    val surfaceGravity: Double
        get() {
            val G = 6.67300E-11
            return G * mass * 1e24 / (radius * 1000).let { it * it }
        }
    
    val isInnerPlanet: Boolean
        get() = distanceFromSun < 2.0
    
    // Method
    fun surfaceWeight(earthWeight: Double): Double {
        val earthGravity = 9.81  // m/s²
        return earthWeight * surfaceGravity / earthGravity
    }
    
    fun distanceCategory(): String = when {
        distanceFromSun < 1.5 -> "Inner Solar System"
        distanceFromSun < 5.0 -> "Asteroid Belt Region"
        else -> "Outer Solar System"
    }
}

fun main() {
    val earthWeight = 75.0  // kg
    
    println("=== Weight on Planets (Earth weight: ${earthWeight}kg) ===")
    Planet.values().forEach { planet ->
        println("${planet.name}: ${"%.1f".format(planet.surfaceWeight(earthWeight))} kg - ${planet.distanceCategory()}")
    }
    
    println("\n=== Inner Planets ===")
    Planet.values()
        .filter { it.isInnerPlanet }
        .forEach { println(it.name) }
}
```

### Enum กับ Abstract Methods

```kotlin
enum class Operation(val symbol: String) {
    ADD("+") {
        override fun apply(a: Double, b: Double) = a + b
    },
    SUBTRACT("-") {
        override fun apply(a: Double, b: Double) = a - b
    },
    MULTIPLY("×") {
        override fun apply(a: Double, b: Double) = a * b
    },
    DIVIDE("÷") {
        override fun apply(a: Double, b: Double): Double {
            require(b != 0.0) { "Cannot divide by zero" }
            return a / b
        }
    },
    POWER("^") {
        override fun apply(a: Double, b: Double) = Math.pow(a, b)
    },
    MODULO("%") {
        override fun apply(a: Double, b: Double): Double {
            require(b != 0.0) { "Cannot modulo by zero" }
            return a % b
        }
    };
    
    abstract fun apply(a: Double, b: Double): Double
    
    fun format(a: Double, b: Double): String {
        return "$a $symbol $b = ${apply(a, b)}"
    }
}

fun calculate(a: Double, symbol: String, b: Double): Double {
    val op = Operation.values().find { it.symbol == symbol }
        ?: throw IllegalArgumentException("Unknown operator: $symbol")
    return op.apply(a, b)
}

fun main() {
    Operation.values().forEach { op ->
        println(op.format(10.0, 3.0))
    }
    
    println("\n--- Calculator ---")
    println(calculate(15.0, "+", 7.0))
    println(calculate(15.0, "÷", 4.0))
    println(calculate(2.0, "^", 10.0))
}
```

---

## 🔄 4. Enum กับ When Expression

```kotlin
enum class Season {
    SPRING, SUMMER, FALL, WINTER
}

enum class DayOfWeek {
    MONDAY, TUESDAY, WEDNESDAY, THURSDAY, FRIDAY, SATURDAY, SUNDAY;
    
    val isWeekend: Boolean get() = this == SATURDAY || this == SUNDAY
    val isWeekday: Boolean get() = !isWeekend
}

enum class HttpStatus(val code: Int, val description: String) {
    OK(200, "OK"),
    CREATED(201, "Created"),
    NO_CONTENT(204, "No Content"),
    BAD_REQUEST(400, "Bad Request"),
    UNAUTHORIZED(401, "Unauthorized"),
    FORBIDDEN(403, "Forbidden"),
    NOT_FOUND(404, "Not Found"),
    INTERNAL_SERVER_ERROR(500, "Internal Server Error"),
    SERVICE_UNAVAILABLE(503, "Service Unavailable");
    
    val isSuccess: Boolean get() = code in 200..299
    val isClientError: Boolean get() = code in 400..499
    val isServerError: Boolean get() = code in 500..599
    
    fun toResponse(body: Any? = null): String {
        return "$code $description${if (body != null) ": $body" else ""}"
    }
    
    companion object {
        fun fromCode(code: Int): HttpStatus? = values().find { it.code == code }
        
        fun fromCodeOrDefault(code: Int, default: HttpStatus = INTERNAL_SERVER_ERROR): HttpStatus {
            return fromCode(code) ?: default
        }
    }
}

fun getSeasonDescription(season: Season): String = when (season) {
    Season.SPRING -> "Flowers blooming, moderate temperature 🌸"
    Season.SUMMER -> "Hot and sunny, perfect for beach ☀️"
    Season.FALL   -> "Leaves falling, cool breeze 🍂"
    Season.WINTER -> "Cold and possibly snowy ❄️"
}

fun getWorkSchedule(day: DayOfWeek): String = when {
    day == DayOfWeek.MONDAY  -> "Weekly planning meeting at 9am"
    day == DayOfWeek.FRIDAY  -> "Sprint review at 3pm, early finish!"
    day.isWeekend            -> "Day off! Enjoy your weekend 🎉"
    else                     -> "Regular work day"
}

fun handleHttpStatus(status: HttpStatus) = when {
    status.isSuccess     -> println("✅ ${status.toResponse()}")
    status.isClientError -> println("⚠️ Client error: ${status.toResponse()}")
    status.isServerError -> println("🔥 Server error: ${status.toResponse()}")
    else                 -> println("ℹ️ ${status.toResponse()}")
}

fun main() {
    println("=== Seasons ===")
    Season.values().forEach { season ->
        println("${season.name}: ${getSeasonDescription(season)}")
    }
    
    println("\n=== Work Week ===")
    DayOfWeek.values().forEach { day ->
        println("${day.name}: ${getWorkSchedule(day)}")
    }
    
    println("\n=== HTTP Status Handling ===")
    listOf(HttpStatus.OK, HttpStatus.NOT_FOUND, HttpStatus.INTERNAL_SERVER_ERROR)
        .forEach { handleHttpStatus(it) }
    
    println("\n=== Find by code ===")
    println(HttpStatus.fromCode(404))            // NOT_FOUND
    println(HttpStatus.fromCode(999))            // null
    println(HttpStatus.fromCodeOrDefault(999))   // INTERNAL_SERVER_ERROR
}
```

---

## 🏭 5. Result/Either Pattern ด้วย Sealed Class

### Validation Pattern

```kotlin
sealed class ValidationResult {
    object Valid : ValidationResult()
    data class Invalid(val errors: List<String>) : ValidationResult()
    
    val isValid: Boolean get() = this is Valid
    
    fun errorMessages(): List<String> = when (this) {
        is Valid -> emptyList()
        is Invalid -> errors
    }
}

// Validator functions
fun validateEmail(email: String): ValidationResult {
    val errors = mutableListOf<String>()
    if (email.isBlank()) errors.add("Email cannot be blank")
    if (!email.contains("@")) errors.add("Email must contain @")
    if (!email.contains(".")) errors.add("Email must contain a dot")
    if (email.length > 100) errors.add("Email too long (max 100 chars)")
    
    return if (errors.isEmpty()) ValidationResult.Valid
           else ValidationResult.Invalid(errors)
}

fun validatePassword(password: String): ValidationResult {
    val errors = mutableListOf<String>()
    if (password.length < 8) errors.add("Password must be at least 8 characters")
    if (!password.any { it.isUpperCase() }) errors.add("Password must contain uppercase letter")
    if (!password.any { it.isLowerCase() }) errors.add("Password must contain lowercase letter")
    if (!password.any { it.isDigit() }) errors.add("Password must contain a digit")
    if (!password.any { "!@#\$%^&*()".contains(it) }) errors.add("Password must contain special character")
    
    return if (errors.isEmpty()) ValidationResult.Valid
           else ValidationResult.Invalid(errors)
}

// Combined validation
data class RegistrationForm(val email: String, val password: String, val name: String)

fun validateRegistration(form: RegistrationForm): ValidationResult {
    val allErrors = mutableListOf<String>()
    
    if (form.name.isBlank()) allErrors.add("Name cannot be blank")
    
    validateEmail(form.email).let { result ->
        if (result is ValidationResult.Invalid) allErrors.addAll(result.errors)
    }
    
    validatePassword(form.password).let { result ->
        if (result is ValidationResult.Invalid) allErrors.addAll(result.errors)
    }
    
    return if (allErrors.isEmpty()) ValidationResult.Valid
           else ValidationResult.Invalid(allErrors)
}

fun main() {
    val forms = listOf(
        RegistrationForm("alice@example.com", "SecurePass1!", "Alice"),
        RegistrationForm("invalid-email", "weak", ""),
        RegistrationForm("bob@test.com", "NoSpecial1", "Bob")
    )
    
    forms.forEachIndexed { index, form ->
        println("=== Form ${index + 1}: ${form.name} ===")
        val result = validateRegistration(form)
        when (result) {
            is ValidationResult.Valid -> println("✅ Valid! Registration successful.")
            is ValidationResult.Invalid -> {
                println("❌ Invalid registration:")
                result.errors.forEach { println("  - $it") }
            }
        }
        println()
    }
}
```

### Error Hierarchy Pattern

```kotlin
sealed class AppError {
    abstract val message: String
    abstract val code: String
    
    // Network errors
    sealed class NetworkError : AppError() {
        data class Timeout(override val message: String = "Request timed out", val timeoutMs: Long = 30000) : NetworkError() {
            override val code = "NET_001"
        }
        data class ConnectionRefused(val host: String) : NetworkError() {
            override val message = "Connection refused to $host"
            override val code = "NET_002"
        }
        object NoInternet : NetworkError() {
            override val message = "No internet connection"
            override val code = "NET_003"
        }
    }
    
    // Business errors
    sealed class BusinessError : AppError() {
        data class NotFound(val resource: String, val id: Any) : BusinessError() {
            override val message = "$resource with id '$id' not found"
            override val code = "BIZ_001"
        }
        data class Unauthorized(val reason: String = "Authentication required") : BusinessError() {
            override val message = reason
            override val code = "BIZ_002"
        }
        data class ValidationFailed(val field: String, override val message: String) : BusinessError() {
            override val code = "BIZ_003"
        }
        data class DuplicateEntry(val resource: String, val field: String) : BusinessError() {
            override val message = "$resource with same $field already exists"
            override val code = "BIZ_004"
        }
    }
    
    // System errors
    sealed class SystemError : AppError() {
        data class DatabaseError(override val message: String, val cause: Throwable? = null) : SystemError() {
            override val code = "SYS_001"
        }
        data class ConfigurationError(override val message: String) : SystemError() {
            override val code = "SYS_002"
        }
    }
    
    fun toUserMessage(): String = when (this) {
        is NetworkError.Timeout -> "The request took too long. Please try again."
        is NetworkError.ConnectionRefused -> "Unable to connect. Please try again later."
        is NetworkError.NoInternet -> "Please check your internet connection."
        is BusinessError.NotFound -> message
        is BusinessError.Unauthorized -> "You don't have permission to perform this action."
        is BusinessError.ValidationFailed -> "Invalid input: $message"
        is BusinessError.DuplicateEntry -> message
        is SystemError.DatabaseError -> "A system error occurred. Please try again."
        is SystemError.ConfigurationError -> "System configuration error. Contact support."
    }
    
    fun toHttpStatus(): Int = when (this) {
        is NetworkError -> 503
        is BusinessError.NotFound -> 404
        is BusinessError.Unauthorized -> 401
        is BusinessError.ValidationFailed -> 400
        is BusinessError.DuplicateEntry -> 409
        is SystemError -> 500
    }
}

fun main() {
    val errors: List<AppError> = listOf(
        AppError.NetworkError.NoInternet,
        AppError.BusinessError.NotFound("User", 42),
        AppError.BusinessError.ValidationFailed("email", "must be a valid email"),
        AppError.BusinessError.DuplicateEntry("User", "email"),
        AppError.SystemError.DatabaseError("Connection pool exhausted")
    )
    
    println("=== Error Handling ===")
    errors.forEach { error ->
        println("[${error.code}] HTTP ${error.toHttpStatus()}")
        println("  Technical: ${error.message}")
        println("  User-facing: ${error.toUserMessage()}")
        println()
    }
}
```

---

## 🔄 6. ตัวอย่าง: State Machine

```kotlin
sealed class OrderStatus {
    abstract val displayName: String
    abstract val allowedTransitions: Set<String>
    
    object Pending : OrderStatus() {
        override val displayName = "Pending"
        override val allowedTransitions = setOf("CONFIRM", "CANCEL")
    }
    
    data class Confirmed(val confirmedAt: Long = System.currentTimeMillis()) : OrderStatus() {
        override val displayName = "Confirmed"
        override val allowedTransitions = setOf("SHIP", "CANCEL")
    }
    
    data class Shipped(
        val trackingNumber: String,
        val carrier: String,
        val shippedAt: Long = System.currentTimeMillis()
    ) : OrderStatus() {
        override val displayName = "Shipped"
        override val allowedTransitions = setOf("DELIVER")
    }
    
    data class Delivered(val deliveredAt: Long = System.currentTimeMillis()) : OrderStatus() {
        override val displayName = "Delivered"
        override val allowedTransitions = setOf("RETURN")
    }
    
    data class Cancelled(val reason: String, val cancelledAt: Long = System.currentTimeMillis()) : OrderStatus() {
        override val displayName = "Cancelled"
        override val allowedTransitions = emptySet<String>()
    }
    
    data class Returned(val reason: String, val returnedAt: Long = System.currentTimeMillis()) : OrderStatus() {
        override val displayName = "Returned"
        override val allowedTransitions = emptySet<String>()
    }
    
    fun canTransition(action: String) = action in allowedTransitions
}

data class Order(
    val id: String,
    val items: List<String>,
    val customerName: String,
    val status: OrderStatus = OrderStatus.Pending
) {
    val statusHistory = mutableListOf(status)
    
    fun confirm(): Order {
        check(status.canTransition("CONFIRM")) {
            "Cannot confirm order in status: ${status.displayName}"
        }
        val newStatus = OrderStatus.Confirmed()
        val newOrder = copy(status = newStatus)
        newOrder.statusHistory.addAll(statusHistory)
        newOrder.statusHistory.add(newStatus)
        return newOrder
    }
    
    fun ship(trackingNumber: String, carrier: String): Order {
        check(status.canTransition("SHIP")) {
            "Cannot ship order in status: ${status.displayName}"
        }
        val newStatus = OrderStatus.Shipped(trackingNumber, carrier)
        return copy(status = newStatus)
    }
    
    fun deliver(): Order {
        check(status.canTransition("DELIVER")) {
            "Cannot deliver order in status: ${status.displayName}"
        }
        return copy(status = OrderStatus.Delivered())
    }
    
    fun cancel(reason: String): Order {
        check(status.canTransition("CANCEL")) {
            "Cannot cancel order in status: ${status.displayName}"
        }
        return copy(status = OrderStatus.Cancelled(reason))
    }
    
    fun `return`(reason: String): Order {
        check(status.canTransition("RETURN")) {
            "Cannot return order in status: ${status.displayName}"
        }
        return copy(status = OrderStatus.Returned(reason))
    }
    
    fun describeStatus(): String = when (val s = status) {
        is OrderStatus.Pending -> "Order #$id is awaiting confirmation"
        is OrderStatus.Confirmed -> "Order #$id confirmed"
        is OrderStatus.Shipped -> "Order #$id shipped via ${s.carrier} (${s.trackingNumber})"
        is OrderStatus.Delivered -> "Order #$id delivered"
        is OrderStatus.Cancelled -> "Order #$id cancelled: ${s.reason}"
        is OrderStatus.Returned -> "Order #$id returned: ${s.reason}"
    }
}

fun main() {
    var order = Order(
        id = "ORD-001",
        items = listOf("Laptop", "Mouse"),
        customerName = "Alice"
    )
    
    println("Initial: ${order.describeStatus()}")
    
    order = order.confirm()
    println("After confirm: ${order.describeStatus()}")
    
    order = order.ship("TH123456789", "Kerry Express")
    println("After ship: ${order.describeStatus()}")
    
    order = order.deliver()
    println("After deliver: ${order.describeStatus()}")
    
    // Try invalid transition
    try {
        order = order.confirm()  // Can't confirm delivered order!
    } catch (e: IllegalStateException) {
        println("Error: ${e.message}")
    }
    
    // Return
    order = order.`return`("Product defective")
    println("After return: ${order.describeStatus()}")
    
    println("\n--- Another order: Cancel scenario ---")
    var order2 = Order("ORD-002", listOf("Book"), "Bob")
    order2 = order2.confirm()
    order2 = order2.cancel("Customer changed mind")
    println("Final: ${order2.describeStatus()}")
}
```

---

## 🌐 7. ตัวอย่าง: API Response Pattern

```kotlin
sealed class ApiResponse<out T> {
    data class Success<T>(
        val data: T,
        val statusCode: Int = 200,
        val message: String = "OK"
    ) : ApiResponse<T>()
    
    data class Error(
        val statusCode: Int,
        val message: String,
        val details: List<String> = emptyList(),
        val requestId: String = java.util.UUID.randomUUID().toString()
    ) : ApiResponse<Nothing>()
    
    data class Paginated<T>(
        val data: List<T>,
        val page: Int,
        val pageSize: Int,
        val total: Int,
        val statusCode: Int = 200
    ) : ApiResponse<List<T>>() {
        val totalPages: Int get() = Math.ceil(total.toDouble() / pageSize).toInt()
        val hasNext: Boolean get() = page < totalPages
        val hasPrev: Boolean get() = page > 1
    }
    
    // Convenience methods
    fun isSuccess() = this is Success || this is Paginated
    
    fun <R> map(transform: (T) -> R): ApiResponse<R> = when (this) {
        is Success -> Success(transform(data), statusCode, message)
        is Error -> this
        is Paginated -> throw UnsupportedOperationException("Use mapList for Paginated")
    }
    
    companion object {
        fun <T> ok(data: T) = Success(data, 200, "OK")
        fun <T> created(data: T) = Success(data, 201, "Created")
        fun notFound(message: String) = Error(404, message)
        fun badRequest(message: String, details: List<String> = emptyList()) = Error(400, message, details)
        fun unauthorized() = Error(401, "Unauthorized")
        fun serverError(message: String) = Error(500, message)
        
        fun <T> paginate(
            data: List<T>, 
            page: Int, 
            pageSize: Int, 
            total: Int
        ) = Paginated(data, page, pageSize, total)
    }
}

// Simulated service and controller
data class Article(val id: Int, val title: String, val author: String, val published: Boolean)

object ArticleService {
    private val articles = (1..25).map { i ->
        Article(i, "Article #$i", "Author ${i % 5 + 1}", i % 3 != 0)
    }
    
    fun getById(id: Int): ApiResponse<Article> {
        if (id <= 0) return ApiResponse.badRequest("ID must be positive")
        return articles.find { it.id == id }?.let { ApiResponse.ok(it) }
            ?: ApiResponse.notFound("Article with id $id not found")
    }
    
    fun getAll(page: Int = 1, pageSize: Int = 10): ApiResponse<List<Article>> {
        if (page <= 0) return ApiResponse.badRequest("Page must be positive")
        if (pageSize !in 1..50) return ApiResponse.badRequest("Page size must be 1-50")
        
        val published = articles.filter { it.published }
        val startIndex = (page - 1) * pageSize
        val pageData = published.drop(startIndex).take(pageSize)
        
        return ApiResponse.paginate(pageData, page, pageSize, published.size)
    }
    
    fun create(title: String, author: String): ApiResponse<Article> {
        val errors = mutableListOf<String>()
        if (title.isBlank()) errors.add("Title cannot be blank")
        if (title.length > 200) errors.add("Title too long")
        if (author.isBlank()) errors.add("Author cannot be blank")
        
        if (errors.isNotEmpty()) return ApiResponse.badRequest("Validation failed", errors)
        
        val newArticle = Article(articles.size + 1, title, author, false)
        return ApiResponse.created(newArticle)
    }
}

fun handleResponse(response: ApiResponse<*>) {
    when (response) {
        is ApiResponse.Success -> {
            println("✅ [${response.statusCode}] ${response.message}")
            println("   Data: ${response.data}")
        }
        is ApiResponse.Error -> {
            println("❌ [${response.statusCode}] ${response.message}")
            if (response.details.isNotEmpty()) {
                response.details.forEach { println("   - $it") }
            }
            println("   RequestId: ${response.requestId}")
        }
        is ApiResponse.Paginated<*> -> {
            println("📄 [${response.statusCode}] Page ${response.page}/${response.totalPages}")
            println("   Total: ${response.total} items, ${response.data.size} on this page")
            println("   Has next: ${response.hasNext}, Has prev: ${response.hasPrev}")
            response.data.forEach { println("   - $it") }
        }
    }
}

fun main() {
    println("=== Article API Demo ===\n")
    
    println("--- Get by ID ---")
    handleResponse(ArticleService.getById(1))
    println()
    handleResponse(ArticleService.getById(999))
    println()
    handleResponse(ArticleService.getById(-1))
    
    println("\n--- Get All (paginated) ---")
    handleResponse(ArticleService.getAll(page = 1, pageSize = 5))
    println()
    handleResponse(ArticleService.getAll(page = 2, pageSize = 5))
    
    println("\n--- Create ---")
    handleResponse(ArticleService.create("New Kotlin Article", "Alice"))
    println()
    handleResponse(ArticleService.create("", ""))  // validation error
}
```

---

## 📝 สรุป Part 08

| แนวคิด | รายละเอียด |
|--------|-----------|
| `data class` | Auto-generated equals, hashCode, toString, copy, componentN |
| `copy()` | สร้าง object ใหม่โดยเปลี่ยนแค่บาง fields |
| Destructuring | Unpack data class ด้วย `val (a, b, c) = obj` |
| `componentN()` | Functions ที่ enable destructuring |
| `sealed class` | Restricted hierarchy ที่ compiler รู้ทุก subclass |
| `enum class` | Named constants พร้อม properties/methods |
| `enum.values()` | ได้ทุก enum constants |
| `enum.valueOf()` | หา enum จาก string |
| Result pattern | Sealed class สำหรับ success/failure |
| State machine | sealed class สำหรับ define states |

### เมื่อไหรใช้อะไร?

- **data class**: ใช้กับ data transfer objects (DTO), value objects, configuration
- **sealed class**: ใช้กับ states, events, results, algebraic data types
- **enum class**: ใช้กับ fixed set of constants ที่รู้แน่นอน เช่น colors, directions, statuses

---

## 🏋️ แบบฝึกหัด Part 08

### ระดับ 1 (พื้นฐาน)

1. สร้าง `data class Address` ที่มี `street`, `city`, `country`, `zipCode` จากนั้น:
   - สร้าง address แล้ว copy เปลี่ยน city
   - ใช้ destructuring เพื่อ extract city และ country
   - ทดสอบ equals กับ address ที่มีค่าเหมือนกัน

2. สร้าง `enum class Currency` ที่มี `USD`, `EUR`, `THB`, `JPY` พร้อม property `symbol` (เช่น "$", "€", "฿", "¥") และ method `format(amount: Double)` ที่ return string เช่น "$100.00"

3. สร้าง `sealed class Shape` ที่มี `Circle(radius)`, `Rectangle(w, h)`, `Triangle(base, height)` และเขียน `calculateArea()` function

### ระดับ 2 (กลาง)

4. สร้าง `sealed class LoginResult` ที่มี:
   - `Success(userId, token, expiresAt)`
   - `InvalidCredentials`
   - `AccountLocked(unlockAt)`
   - `RequiresTwoFactor(userId)`
   เขียนฟังก์ชัน `handleLogin()` ที่แสดงข้อความเหมาะสมสำหรับแต่ละ case

5. สร้าง `enum class Weekday` ที่มี properties: `isWorkday`, `nextDay` และ method `hoursWorked()` ที่คืนค่า hours ตาม type ของวัน (weekday=8, Saturday=4, Sunday=0)

6. Implement `Result<T>` sealed class พร้อม `map()`, `flatMap()`, `getOrElse()` และนำไปใช้กับ chain of operations ที่อาจ fail

### ระดับ 3 (ท้าทาย)

7. สร้าง State Machine สำหรับ ATM โดยใช้ sealed class:
   - States: `Idle`, `CardInserted(cardNumber)`, `PinEntered(cardNumber, attempts)`, `Authenticated(userId, balance)`, `Dispensing(amount)`, `Error(reason)`
   - Transitions: `insertCard()`, `enterPin()`, `selectAmount()`, `dispense()`, `eject()`

8. สร้าง `sealed class JsonValue` ที่ represent JSON types:
   - `JsonNull`, `JsonBoolean(value)`, `JsonNumber(value)`, `JsonString(value)`, `JsonArray(items)`, `JsonObject(fields)`
   - เขียน `stringify()` function ที่แปลง JsonValue เป็น JSON string

---

## ➡️ ถัดไป: Part 09 - Collections

---
*Part 08/100+ | Kotlin & Spring Boot Complete Course*
