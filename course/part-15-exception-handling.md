# Part 15: Exception Handling
## การจัดการ Exception ใน Kotlin

---

## 🎯 เป้าหมายของ Part นี้

- เข้าใจ try/catch/finally
- รู้ว่า Kotlin ไม่มี checked exceptions
- ใช้ Elvis operator สำหรับ exception handling
- สร้าง Custom Exceptions
- ใช้ Result type

---

## 1. try/catch พื้นฐาน

```kotlin
// try/catch เหมือน Java แต่เป็น expression
fun divide(a: Int, b: Int): Int {
    return try {
        a / b
    } catch (e: ArithmeticException) {
        println("Cannot divide by zero!")
        -1
    } finally {
        println("Division attempted")  // ทำเสมอ
    }
}

println(divide(10, 2))   // Division attempted \n 5
println(divide(10, 0))   // Cannot divide by zero! \n Division attempted \n -1

// try เป็น expression
val result = try {
    "123".toInt()
} catch (e: NumberFormatException) {
    0
}
println(result)  // 123
```

---

## 2. ไม่มี Checked Exceptions

```kotlin
// Java ต้องประกาศ throws:
// void readFile() throws IOException { }

// Kotlin ไม่บังคับ - ทุก exception เป็น unchecked
fun readFile(path: String): String {
    return java.io.File(path).readText()  // ไม่ต้อง declare throws
}

// ถ้าต้องการบอก Java code - ใช้ @Throws
@Throws(java.io.IOException::class)
fun readFileForJava(path: String): String {
    return java.io.File(path).readText()
}
```

---

## 3. Multiple catch blocks

```kotlin
fun parseData(input: String): Int {
    return try {
        when {
            input.startsWith("0x") -> input.substring(2).toInt(16)
            input.startsWith("0b") -> input.substring(2).toInt(2)
            else -> input.toInt()
        }
    } catch (e: NumberFormatException) {
        println("Invalid number format: $input")
        0
    } catch (e: IllegalArgumentException) {
        println("Invalid argument: ${e.message}")
        -1
    } catch (e: Exception) {
        println("Unexpected error: ${e.message}")
        throw e  // rethrow
    }
}

println(parseData("42"))     // 42
println(parseData("0xFF"))   // 255
println(parseData("abc"))    // Invalid number format: abc \n 0
```

---

## 4. Custom Exceptions

```kotlin
// Custom exception classes
class ValidationException(
    message: String,
    val field: String,
    val rejectedValue: Any? = null
) : RuntimeException(message)

class NotFoundException(
    message: String,
    val resourceId: Any
) : RuntimeException(message)

class BusinessException(
    message: String,
    val errorCode: String,
    cause: Throwable? = null
) : RuntimeException(message, cause)

// ใช้งาน
fun validateAge(age: Int) {
    if (age < 0) throw ValidationException(
        message = "Age cannot be negative",
        field = "age",
        rejectedValue = age
    )
    if (age > 150) throw ValidationException(
        message = "Age is unrealistic",
        field = "age",
        rejectedValue = age
    )
}

fun findUser(id: Int): String {
    val users = mapOf(1 to "Alice", 2 to "Bob")
    return users[id] ?: throw NotFoundException("User not found", id)
}

// Catching custom exceptions
fun main() {
    try {
        validateAge(-5)
    } catch (e: ValidationException) {
        println("Validation error on '${e.field}': ${e.message} (value: ${e.rejectedValue})")
    }
    
    try {
        findUser(999)
    } catch (e: NotFoundException) {
        println("Not found: ${e.message} (id: ${e.resourceId})")
    }
}
```

---

## 5. Result Type

```kotlin
// Kotlin มี Result<T> built-in (experimental → stable ใน 1.5+)
// ใช้แทน try/catch สำหรับ functional style

fun safeDivide(a: Int, b: Int): Result<Int> =
    runCatching { a / b }

fun safeParseInt(s: String): Result<Int> =
    runCatching { s.toInt() }

// ใช้งาน
val r1 = safeDivide(10, 2)
val r2 = safeDivide(10, 0)

println(r1.isSuccess)  // true
println(r1.getOrNull())  // 5

println(r2.isFailure)  // true
println(r2.exceptionOrNull()?.message)  // / by zero

// getOrDefault
println(r2.getOrDefault(-1))  // -1

// getOrElse
println(r2.getOrElse { e -> println("Error: ${e.message}"); -1 })  // -1

// map / flatMap
val result = safeParseInt("42")
    .map { it * 2 }
    .getOrDefault(0)
println(result)  // 84

// fold - handle both success and failure
safeParseInt("abc").fold(
    onSuccess = { println("Parsed: $it") },
    onFailure = { println("Failed: ${it.message}") }
)
// Failed: For input string: "abc"
```

---

## 6. Sealed Class สำหรับ Error Handling

```kotlin
// Pattern ที่นิยมมาก สำหรับ functional error handling
sealed class ApiResult<out T> {
    data class Success<T>(val data: T) : ApiResult<T>()
    data class Error(val message: String, val code: Int = 0) : ApiResult<Nothing>()
    object Loading : ApiResult<Nothing>()
}

fun fetchUser(id: Int): ApiResult<String> {
    if (id <= 0) return ApiResult.Error("Invalid ID", 400)
    if (id > 100) return ApiResult.Error("User not found", 404)
    return ApiResult.Success("User$id")
}

// ใช้งานด้วย when
fun handleResult(result: ApiResult<String>) = when (result) {
    is ApiResult.Success -> println("Got user: ${result.data}")
    is ApiResult.Error   -> println("Error ${result.code}: ${result.message}")
    is ApiResult.Loading -> println("Loading...")
}

handleResult(fetchUser(1))    // Got user: User1
handleResult(fetchUser(-1))   // Error 400: Invalid ID
handleResult(fetchUser(999))  // Error 404: User not found
```

---

## 7. ตัวอย่าง: Service Layer Exception Handling

```kotlin
// Exceptions สำหรับ Service layer
sealed class ServiceError {
    data class NotFound(val id: Any) : ServiceError()
    data class Validation(val errors: Map<String, String>) : ServiceError()
    data class Unauthorized(val reason: String) : ServiceError()
    data class Internal(val cause: Throwable) : ServiceError()
}

typealias ServiceResult<T> = Result<T>

class UserService {
    private val users = mutableMapOf<Int, String>()
    private var nextId = 1
    
    fun createUser(name: String): ServiceResult<Int> = runCatching {
        val errors = mutableMapOf<String, String>()
        if (name.isBlank()) errors["name"] = "Name cannot be blank"
        if (name.length > 50) errors["name"] = "Name too long"
        
        if (errors.isNotEmpty()) {
            throw ValidationException("Validation failed", "name")
        }
        
        val id = nextId++
        users[id] = name
        id
    }
    
    fun getUser(id: Int): ServiceResult<String> = runCatching {
        users[id] ?: throw NotFoundException("User not found", id)
    }
}

fun main() {
    val service = UserService()
    
    // Create user
    service.createUser("Alice").fold(
        onSuccess = { id -> println("Created user with id=$id") },
        onFailure = { e -> println("Create failed: ${e.message}") }
    )
    
    // Get user
    service.getUser(1).fold(
        onSuccess = { name -> println("Found: $name") },
        onFailure = { e -> println("Not found: ${e.message}") }
    )
    
    // Try invalid
    service.createUser("").fold(
        onSuccess = { println("Should not succeed") },
        onFailure = { e -> println("Validation: ${e.message}") }
    )
}
```

---

## 📝 สรุป Part 15

| แนวคิด | รายละเอียด |
|--------|-----------|
| `try/catch` | Expression, multiple catch |
| No checked exceptions | ไม่บังคับประกาศ throws |
| Custom exceptions | Extend RuntimeException |
| `Result<T>` | Functional error handling |
| `runCatching` | สร้าง Result จาก exception |
| Sealed class | Type-safe error types |

---

## ➡️ ถัดไป: Part 16 - File I/O

---
*Part 15/100+ | Kotlin & Spring Boot Complete Course*
