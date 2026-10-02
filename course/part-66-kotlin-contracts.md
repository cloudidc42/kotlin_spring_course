# Part 66: Kotlin Contracts

## Kotlin Contracts — ช่วย Compiler วิเคราะห์โค้ดได้แม่นยำขึ้น

---

## 🎯 เป้าหมายของ Part นี้

- เข้าใจ Kotlin Contracts คืออะไรและทำงานอย่างไร
- ใช้ `contract {}` block เพื่อให้ข้อมูลกับ compiler
- เรียนรู้ `Returns`, `CallsInPlace` effects
- สร้าง custom assertion functions ที่ทำงานร่วมกับ smart cast
- ทำความเข้าใจ implicit contracts

---

## 📖 1. Kotlin Contracts คืออะไร?

**Kotlin Contracts** เป็นกลไกที่ช่วยให้นักพัฒนาสามารถบอก compiler ว่า function ของเราทำงานอย่างไร เพื่อให้ compiler วิเคราะห์ได้แม่นยำและถูกต้องมากขึ้น

โดยปกติ compiler ไม่รู้ว่า function ภายนอกทำอะไร แต่ด้วย contracts เราสามารถบอกได้ว่า:
- ถ้า function return ค่าหนึ่ง แสดงว่า argument มีค่า non-null
- function จะเรียก lambda กี่ครั้งและในลำดับใด

---

## 🔧 2. contract {} Block

Contract ถูกเพิ่มเข้ามาใน Kotlin 1.3 และยังอยู่ในสถานะ `@ExperimentalContracts`

```kotlin
import kotlin.contracts.*

@OptIn(ExperimentalContracts::class)
fun String?.isNotNullOrEmpty(): Boolean {
    contract {
        returns(true) implies (this@isNotNullOrEmpty != null)
    }
    return this != null && this.isNotEmpty()
}

fun main() {
    val name: String? = "Hello"
    if (name.isNotNullOrEmpty()) {
        // compiler รู้ว่า name ไม่ใช่ null ที่นี่
        println(name.length) // ไม่ต้อง !! หรือ ?.
    }
}
```

---

## 📋 3. Returns Effect

`returns()` บอก compiler ว่าเมื่อ function return ค่านี้ เงื่อนไขอะไรเป็นจริง

### returns(value) implies condition

```kotlin
import kotlin.contracts.*

@OptIn(ExperimentalContracts::class)
fun requireNotNull(value: Any?): Boolean {
    contract {
        returns(true) implies (value != null)
        returns(false) implies (value == null)
    }
    return value != null
}

fun processValue(input: Any?) {
    if (requireNotNull(input)) {
        // compiler รู้ว่า input != null ที่นี่
        println("Value: $input")
    }
}
```

### returnsNotNull() implies condition

```kotlin
import kotlin.contracts.*

@OptIn(ExperimentalContracts::class)
fun <T : Any> nullableToString(value: T?): String? {
    contract {
        returnsNotNull() implies (value != null)
    }
    return value?.toString()
}

fun example(obj: Any?) {
    val result = nullableToString(obj)
    if (result != null) {
        // compiler รู้ว่า obj != null ที่นี่
        println("Original type: ${obj::class.simpleName}")
    }
}
```

---

## ⚡ 4. callsInPlace Effect

`callsInPlace` บอก compiler ว่า lambda จะถูกเรียกกี่ครั้งและ inline หรือไม่

```kotlin
import kotlin.contracts.*

@OptIn(ExperimentalContracts::class)
inline fun <R> runSafe(block: () -> R): R {
    contract {
        callsInPlace(block, InvocationKind.EXACTLY_ONCE)
    }
    return block()
}

fun main() {
    val name: String
    runSafe {
        name = "Kotlin" // compiler อนุญาตให้ assign val ได้ เพราะรู้ว่าจะเรียกครั้งเดียว
    }
    println(name) // compiler รู้ว่า name ถูก initialize แล้ว
}
```

### InvocationKind ทั้งหมด

```kotlin
import kotlin.contracts.*

// EXACTLY_ONCE - เรียกครั้งเดียวเสมอ
@OptIn(ExperimentalContracts::class)
inline fun <T, R> T.letExact(block: (T) -> R): R {
    contract { callsInPlace(block, InvocationKind.EXACTLY_ONCE) }
    return block(this)
}

// AT_LEAST_ONCE - เรียกอย่างน้อยครั้งเดียว
@OptIn(ExperimentalContracts::class)
inline fun repeatAtLeastOnce(times: Int, block: () -> Unit) {
    contract { callsInPlace(block, InvocationKind.AT_LEAST_ONCE) }
    repeat(maxOf(1, times)) { block() }
}

// AT_MOST_ONCE - เรียกไม่เกินครั้งเดียว
@OptIn(ExperimentalContracts::class)
inline fun <T> runIfNotNull(value: Any?, block: () -> T): T? {
    contract { callsInPlace(block, InvocationKind.AT_MOST_ONCE) }
    return if (value != null) block() else null
}

// UNKNOWN - ไม่แน่ใจจำนวนครั้ง (default)
@OptIn(ExperimentalContracts::class)
inline fun maybeRun(condition: Boolean, block: () -> Unit) {
    contract { callsInPlace(block, InvocationKind.UNKNOWN) }
    if (condition) block()
}
```

---

## 🏗️ 5. ตัวอย่าง: Custom Assertion Functions

สร้าง assertion functions ที่ทำงานร่วมกับ smart cast ได้

```kotlin
import kotlin.contracts.*

object Assertions {
    
    @OptIn(ExperimentalContracts::class)
    fun assertTrue(condition: Boolean, message: String = "Assertion failed") {
        contract {
            returns() implies condition
        }
        if (!condition) throw AssertionError(message)
    }
    
    @OptIn(ExperimentalContracts::class)
    fun <T : Any> assertNotNull(value: T?, message: String = "Value must not be null"): T {
        contract {
            returns() implies (value != null)
        }
        return value ?: throw AssertionError(message)
    }
    
    @OptIn(ExperimentalContracts::class)
    fun assertIs<reified T>(value: Any?, message: String = "Expected ${T::class.simpleName}"): T {
        contract {
            returns() implies (value is T)
        }
        return value as? T ?: throw AssertionError(message)
    }
}
```

### Spring Boot Integration

```kotlin
import kotlin.contracts.*
import org.springframework.stereotype.Service
import org.springframework.web.server.ResponseStatusException
import org.springframework.http.HttpStatus

@Service
class UserService(private val userRepository: UserRepository) {

    @OptIn(ExperimentalContracts::class)
    fun requireUser(userId: Long): User {
        contract {
            returnsNotNull()
        }
        return userRepository.findById(userId)
            .orElseThrow { ResponseStatusException(HttpStatus.NOT_FOUND, "User $userId not found") }
    }

    @OptIn(ExperimentalContracts::class)
    fun requireAdminAccess(user: User?): Boolean {
        contract {
            returns(true) implies (user != null)
        }
        return user != null && user.role == Role.ADMIN
    }

    fun updateProfile(userId: Long, profile: ProfileRequest): UserResponse {
        val user = requireUser(userId)
        // compiler รู้ว่า user ไม่ใช่ null
        user.apply {
            name = profile.name
            email = profile.email
        }
        return userRepository.save(user).toResponse()
    }

    fun performAdminAction(currentUser: User?, action: AdminAction): ActionResult {
        if (requireAdminAccess(currentUser)) {
            // compiler รู้ว่า currentUser != null
            return action.execute(currentUser)
        }
        throw ResponseStatusException(HttpStatus.FORBIDDEN, "Admin access required")
    }
}
```

---

## 🔍 6. Implicit Contracts (Standard Library)

Kotlin standard library มี implicit contracts ที่เราใช้ทุกวันโดยไม่รู้ตัว

```kotlin
fun main() {
    val name: String? = "Hello"
    
    // checkNotNull มี contract: returns() implies (value != null)
    checkNotNull(name) { "Name cannot be null" }
    println(name.length) // smart cast ทำงานได้
    
    // require มี contract: returns() implies value
    val age = -5
    // require(age >= 0) { "Age must be non-negative" }
    
    // let มี contract: callsInPlace(block, EXACTLY_ONCE)
    val result: String
    name.let {
        result = it.uppercase()
    }
    println(result) // compiler รู้ว่า result ถูก initialize
}
```

### Standard Library Examples

```kotlin
// also มี contract: callsInPlace(block, EXACTLY_ONCE)
val initialized: Int
"test".also {
    initialized = it.length
}
println(initialized) // ไม่มี error

// run มี contract: callsInPlace(block, EXACTLY_ONCE)
val computed: String
"hello".run {
    computed = this.uppercase()
}
println(computed) // ไม่มี error

// with มี contract: callsInPlace(block, EXACTLY_ONCE)
val processed: Int
with(listOf(1, 2, 3)) {
    processed = this.sum()
}
println(processed) // ไม่มี error
```

---

## 🧪 7. Advanced Contract Patterns

### Conditional Return Contract

```kotlin
import kotlin.contracts.*

sealed class ValidationResult {
    object Valid : ValidationResult()
    data class Invalid(val message: String) : ValidationResult()
}

@OptIn(ExperimentalContracts::class)
fun ValidationResult.isValid(): Boolean {
    contract {
        returns(true) implies (this@isValid is ValidationResult.Valid)
        returns(false) implies (this@isValid is ValidationResult.Invalid)
    }
    return this is ValidationResult.Valid
}

fun processValidation(result: ValidationResult) {
    if (result.isValid()) {
        // compiler รู้ว่า result เป็น ValidationResult.Valid
        println("Validation passed")
    } else {
        // compiler รู้ว่า result เป็น ValidationResult.Invalid
        println("Error: ${result.message}")
    }
}
```

### Builder Pattern with Contracts

```kotlin
import kotlin.contracts.*

class DatabaseConfig private constructor(
    val host: String,
    val port: Int,
    val database: String,
    val username: String,
    val password: String
) {
    companion object {
        @OptIn(ExperimentalContracts::class)
        inline fun build(init: Builder.() -> Unit): DatabaseConfig {
            contract {
                callsInPlace(init, InvocationKind.EXACTLY_ONCE)
            }
            return Builder().apply(init).build()
        }
    }

    class Builder {
        var host: String = "localhost"
        var port: Int = 5432
        var database: String = ""
        var username: String = ""
        var password: String = ""

        fun build(): DatabaseConfig {
            require(database.isNotBlank()) { "Database name is required" }
            require(username.isNotBlank()) { "Username is required" }
            return DatabaseConfig(host, port, database, username, password)
        }
    }
}

fun main() {
    val config = DatabaseConfig.build {
        host = "db.example.com"
        port = 5432
        database = "myapp"
        username = "admin"
        password = "secret"
    }
    println("Connected to ${config.host}:${config.port}/${config.database}")
}
```

---

## 🧪 8. Testing Contracts

```kotlin
import kotlin.contracts.*
import org.junit.jupiter.api.Test
import org.junit.jupiter.api.assertThrows
import org.assertj.core.api.Assertions.assertThat

class ContractTest {

    @OptIn(ExperimentalContracts::class)
    fun requirePositive(value: Int): Boolean {
        contract {
            returns(true) implies (value > 0)
        }
        return value > 0
    }

    @Test
    fun `test contract-based assertion`() {
        val value = 42
        assertThat(requirePositive(value)).isTrue()
    }

    @Test
    fun `test smart cast with contract`() {
        val text: String? = "Hello"
        
        @OptIn(ExperimentalContracts::class)
        fun isNonEmpty(s: String?): Boolean {
            contract {
                returns(true) implies (s != null)
            }
            return s != null && s.isNotEmpty()
        }

        if (isNonEmpty(text)) {
            // compiler รู้ว่า text ไม่ใช่ null
            assertThat(text.length).isGreaterThan(0)
        }
    }
}
```

---

## 📊 9. สรุปตาราง Kotlin Contracts

| Effect | คำอธิบาย | ใช้เมื่อ |
|--------|----------|---------|
| `returns()` | Function return ปกติ (ไม่ throw) | Assert-style functions |
| `returns(value)` | Function return ค่าเฉพาะ | Boolean checks |
| `returnsNotNull()` | Function return non-null | Nullable utilities |
| `returns() implies cond` | Return implies condition จริง | Type/null checks |
| `callsInPlace(EXACTLY_ONCE)` | Lambda เรียกครั้งเดียว | Builder, run, let |
| `callsInPlace(AT_LEAST_ONCE)` | Lambda เรียกอย่างน้อยครั้งเดียว | forEach variants |
| `callsInPlace(AT_MOST_ONCE)` | Lambda เรียกไม่เกินครั้งเดียว | Optional callbacks |
| `callsInPlace(UNKNOWN)` | ไม่แน่ใจจำนวน | General callbacks |

---

## 💡 Best Practices

1. **ใช้ contracts กับ inline functions** เพื่อประสิทธิภาพสูงสุด
2. **อย่าโกหก compiler** — contract ที่ผิดทำให้เกิด undefined behavior
3. **ใช้ `@OptIn(ExperimentalContracts::class)`** ครอบทุก function ที่มี contract
4. **Test contract-based functions อย่างละเอียด** เพราะ compiler เชื่อถือ contract
5. **เริ่มจาก standard library** ก่อนเขียน custom contracts

---

*Part 66/100+ | Kotlin & Spring Boot Complete Course*
