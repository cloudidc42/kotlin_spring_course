# Part 15: Exception Handling
## การจัดการ Exceptions ใน Kotlin อย่างมืออาชีพ

---

## 🎯 เป้าหมายของ Part นี้

- ใช้ try-catch-finally ได้ถูกต้อง
- ใช้ multiple catch blocks
- สร้าง custom exception classes
- เข้าใจ exception hierarchy ใน Kotlin
- ใช้ `Result<T>` type
- ใช้ `runCatching { }`
- เข้าใจ Checked vs Unchecked exceptions
- จัดการ exceptions ใน Coroutines ด้วย CoroutineExceptionHandler
- สร้างระบบ error handling สำหรับ HTTP client และ database

---

## 🚨 1. try-catch-finally

### 1.1 พื้นฐาน

```kotlin
fun main() {
    // try-catch พื้นฐาน
    try {
        val result = 10 / 0
        println(result)
    } catch (e: ArithmeticException) {
        println("Error: ${e.message}")  // Error: / by zero
    }
    
    // try-catch เป็น Expression (คืนค่าได้!)
    val value = try {
        "42".toInt()
    } catch (e: NumberFormatException) {
        -1
    }
    println(value)  // 42
    
    val badValue = try {
        "abc".toInt()
    } catch (e: NumberFormatException) {
        -1
    }
    println(badValue)  // -1
    
    // try-catch-finally
    var connection: String? = null
    try {
        connection = "Connected"
        println(connection)
        throw RuntimeException("Something went wrong!")
    } catch (e: RuntimeException) {
        println("Caught: ${e.message}")
    } finally {
        // finally ทำงานเสมอ ไม่ว่าจะ exception หรือไม่
        connection = null
        println("Connection closed (finally block always runs)")
    }
    
    // finally กับ return
    fun riskyFunction(): String {
        try {
            return "success"
        } catch (e: Exception) {
            return "error"
        } finally {
            println("Finally runs even after return!")
            // ห้าม return ใน finally! จะทำให้ exception หาย
        }
    }
    println(riskyFunction())
}
```

### 1.2 Exception เป็น Expression

```kotlin
fun parseAge(ageStr: String): Int {
    return try {
        val age = ageStr.toInt()
        if (age < 0 || age > 150) throw IllegalArgumentException("Age out of range: $age")
        age
    } catch (e: NumberFormatException) {
        throw IllegalArgumentException("Invalid age format: $ageStr", e)
    }
}

fun safeDivide(a: Int, b: Int): Int? {
    return try {
        a / b
    } catch (e: ArithmeticException) {
        null  // คืน null แทน throw exception
    }
}

fun main() {
    // Valid age
    println(parseAge("25"))   // 25
    
    // Invalid format
    try {
        println(parseAge("abc"))
    } catch (e: IllegalArgumentException) {
        println("Error: ${e.message}")
        println("Cause: ${e.cause?.message}")
    }
    
    // Safe divide
    println(safeDivide(10, 2))  // 5
    println(safeDivide(10, 0))  // null
    
    // Elvis operator กับ exception
    val result = safeDivide(10, 0) ?: throw ArithmeticException("Cannot divide by zero")
}
```

---

## 🎯 2. Multiple Catch Blocks

### 2.1 หลาย Exception Types

```kotlin
import java.io.IOException
import java.sql.SQLException

fun processUserInput(input: String, divisor: String): String {
    return try {
        val number = input.toInt()          // อาจ throw NumberFormatException
        val div = divisor.toInt()           // อาจ throw NumberFormatException
        val result = number / div           // อาจ throw ArithmeticException
        "Result: $result"
    } catch (e: NumberFormatException) {
        "Error: '$input' or '$divisor' is not a valid number"
    } catch (e: ArithmeticException) {
        "Error: Cannot divide by zero"
    }
}

fun readAndProcessFile(path: String): String {
    return try {
        val content = java.io.File(path).readText()
        content.split("\n").first()
    } catch (e: IOException) {
        "Error reading file: ${e.message}"
    } catch (e: NoSuchElementException) {
        "Error: File is empty"
    } catch (e: SecurityException) {
        "Error: No permission to read file"
    }
}

fun connectToDatabase(host: String, port: Int) {
    try {
        if (host.isEmpty()) throw IllegalArgumentException("Host cannot be empty")
        if (port !in 1..65535) throw IllegalArgumentException("Invalid port: $port")
        // simulate connection
        println("Connected to $host:$port")
    } catch (e: IllegalArgumentException) {
        println("Configuration error: ${e.message}")
        throw e  // rethrow ให้ caller จัดการ
    } catch (e: IOException) {
        println("Connection failed: ${e.message}")
    }
}

// Catch หลาย exception types พร้อมกัน (multi-catch)
fun parseData(input: String): Int {
    return try {
        input.trim().toInt()
    } catch (e: NumberFormatException) {
        -1
    } catch (e: NullPointerException) {
        -2
    }
}

// หรือใช้ | (union catch)
fun parseDataClean(input: String?): Int {
    return try {
        input!!.trim().toInt()
    } catch (e: Exception) {
        when (e) {
            is NumberFormatException -> -1
            is NullPointerException  -> -2
            else                     -> throw e
        }
    }
}

fun main() {
    println(processUserInput("10", "3"))   // Result: 3
    println(processUserInput("abc", "3"))  // Error: 'abc'...
    println(processUserInput("10", "0"))   // Error: Cannot divide by zero
    
    println(parseData("42"))      // 42
    println(parseData("  17  "))  // 17
    println(parseData("xyz"))     // -1
    println(parseData(""))        // -1
}
```

---

## 🏗️ 3. Custom Exception Classes

### 3.1 สร้าง Custom Exceptions

```kotlin
// Custom exception พื้นฐาน
class ValidationException(message: String) : Exception(message)

// Exception พร้อม cause
class DatabaseException(
    message: String,
    cause: Throwable? = null
) : Exception(message, cause)

// Exception ที่มี properties เพิ่มเติม
class ApiException(
    val statusCode: Int,
    val errorCode: String,
    message: String,
    cause: Throwable? = null
) : Exception(message, cause) {
    
    fun isClientError() = statusCode in 400..499
    fun isServerError() = statusCode in 500..599
    
    override fun toString(): String =
        "ApiException(status=$statusCode, code=$errorCode, message=$message)"
}

// Exception hierarchy
sealed class AppException(message: String, cause: Throwable? = null) :
    Exception(message, cause)

class NetworkException(
    message: String,
    val url: String,
    cause: Throwable? = null
) : AppException(message, cause)

class AuthException(
    message: String,
    val userId: Int? = null
) : AppException(message)

class NotFoundException(
    val resourceType: String,
    val resourceId: Any
) : AppException("$resourceType not found: $resourceId")

class ConflictException(
    val resourceType: String,
    message: String
) : AppException(message)

// ตัวอย่างการใช้งาน
class UserService {
    private val users = mutableMapOf(
        1 to "Alice",
        2 to "Bob"
    )
    
    fun getUser(id: Int): String {
        return users[id] ?: throw NotFoundException("User", id)
    }
    
    fun createUser(id: Int, name: String): String {
        if (users.containsKey(id)) {
            throw ConflictException("User", "User with id $id already exists")
        }
        if (name.isBlank()) {
            throw ValidationException("User name cannot be blank")
        }
        users[id] = name
        return name
    }
    
    fun authenticate(userId: Int, token: String): Boolean {
        val user = users[userId] ?: throw AuthException("User not found", userId)
        if (token != "valid_token") throw AuthException("Invalid token")
        return true
    }
}

fun main() {
    val service = UserService()
    
    // NotFoundException
    try {
        service.getUser(99)
    } catch (e: NotFoundException) {
        println("${e.resourceType} ${e.resourceId} not found")
    }
    
    // ConflictException
    try {
        service.createUser(1, "Alice2")
    } catch (e: ConflictException) {
        println("Conflict: ${e.message}")
    }
    
    // ValidationException
    try {
        service.createUser(5, "")
    } catch (e: ValidationException) {
        println("Validation: ${e.message}")
    }
    
    // Handle AppException hierarchy
    try {
        service.authenticate(1, "wrong_token")
    } catch (e: AppException) {
        when (e) {
            is AuthException -> println("Auth failed: ${e.message} (user: ${e.userId})")
            is NotFoundException -> println("Not found: ${e.message}")
            is NetworkException -> println("Network error: ${e.url}")
            else -> println("App error: ${e.message}")
        }
    }
    
    // ApiException
    val apiError = ApiException(404, "USER_NOT_FOUND", "User does not exist")
    println(apiError)
    println("Is client error: ${apiError.isClientError()}")
}
```

---

## 🌳 4. Exception Hierarchy ใน Kotlin

```
Throwable
├── Error (สำหรับ JVM errors - ไม่ควร catch)
│   ├── OutOfMemoryError
│   ├── StackOverflowError
│   └── AssertionError
└── Exception
    ├── RuntimeException (Unchecked)
    │   ├── IllegalArgumentException
    │   │   └── NumberFormatException
    │   ├── IllegalStateException
    │   ├── NullPointerException
    │   ├── IndexOutOfBoundsException
    │   │   └── ArrayIndexOutOfBoundsException
    │   ├── ArithmeticException
    │   ├── ClassCastException
    │   ├── UnsupportedOperationException
    │   └── CancellationException (Coroutines)
    ├── IOException (Checked ใน Java, Unchecked ใน Kotlin)
    │   ├── FileNotFoundException
    │   └── SocketException
    └── (Custom exceptions ของเรา)
```

```kotlin
fun demonstrateExceptions() {
    // NullPointerException
    try {
        val s: String? = null
        val len = s!!.length  // !! force unwrap
    } catch (e: NullPointerException) {
        println("NPE: ${e.message}")
    }
    
    // IndexOutOfBoundsException
    try {
        val list = listOf(1, 2, 3)
        println(list[10])
    } catch (e: IndexOutOfBoundsException) {
        println("Index out of bounds: ${e.message}")
    }
    
    // ClassCastException
    try {
        val obj: Any = "string"
        val num = obj as Int
    } catch (e: ClassCastException) {
        println("Cast failed: ${e.message}")
    }
    
    // StackOverflowError - ห้าม catch! เป็น Error ไม่ใช่ Exception
    fun infiniteRecursion(): Int = 1 + infiniteRecursion()
    try {
        // infiniteRecursion()  // จะ crash
    } catch (e: StackOverflowError) {
        // ไม่ควรทำแบบนี้!
        println("Stack overflow!")
    }
    
    // Safe alternatives
    val str: String? = null
    val safeLength = str?.length ?: 0
    
    val list = listOf(1, 2, 3)
    val safeElement = list.getOrNull(10) ?: -1
    
    val obj: Any = "string"
    val safeNum = obj as? Int  // null ถ้า cast ไม่ได้
    
    println("Safe: $safeLength, $safeElement, $safeNum")
}

fun main() {
    demonstrateExceptions()
}
```

---

## ✅ 5. Result<T> Type

`Result<T>` เป็น type ที่ห่อหุ้ม ความสำเร็จ (Success) หรือ ความล้มเหลว (Failure)

### 5.1 Result พื้นฐาน

```kotlin
fun divide(a: Int, b: Int): Result<Int> {
    return if (b == 0) {
        Result.failure(ArithmeticException("Division by zero"))
    } else {
        Result.success(a / b)
    }
}

fun parseNumber(s: String): Result<Int> {
    return try {
        Result.success(s.toInt())
    } catch (e: NumberFormatException) {
        Result.failure(e)
    }
}

fun main() {
    // สร้าง Result
    val success = Result.success(42)
    val failure = Result.failure<Int>(RuntimeException("Something went wrong"))
    
    // ตรวจสอบ
    println(success.isSuccess)   // true
    println(success.isFailure)   // false
    println(failure.isSuccess)   // false
    println(failure.isFailure)   // true
    
    // ดึงค่า
    println(success.getOrNull())          // 42
    println(failure.getOrNull())          // null
    println(failure.getOrDefault(-1))     // -1
    println(failure.getOrElse { -999 })  // -999
    
    // exceptionOrNull
    println(failure.exceptionOrNull()?.message)  // Something went wrong
    
    // ทดสอบกับฟังก์ชัน
    val r1 = divide(10, 2)
    val r2 = divide(10, 0)
    val r3 = parseNumber("42")
    val r4 = parseNumber("abc")
    
    println(r1.getOrNull())  // 5
    println(r2.getOrNull())  // null
    println(r3.getOrNull())  // 42
    println(r4.getOrNull())  // null
    
    // fold - จัดการทั้ง success และ failure
    r1.fold(
        onSuccess = { println("Success: $it") },
        onFailure = { println("Failure: ${it.message}") }
    )
    
    r2.fold(
        onSuccess = { println("Success: $it") },
        onFailure = { println("Failure: ${it.message}") }
    )
}
```

### 5.2 Result Chaining

```kotlin
fun fetchUserFromDb(userId: Int): Result<String> {
    return if (userId > 0) Result.success("User_$userId")
    else Result.failure(IllegalArgumentException("Invalid user ID: $userId"))
}

fun fetchUserOrders(userName: String): Result<List<String>> {
    return Result.success(listOf("Order_1", "Order_2"))
}

fun calculateDiscount(orders: List<String>): Result<Double> {
    val discount = if (orders.size >= 2) 0.1 else 0.0
    return Result.success(discount)
}

fun main() {
    // map - แปลง value ถ้า success
    val doubled = Result.success(5).map { it * 2 }
    println(doubled)  // Success(10)
    
    val failedMap = Result.failure<Int>(RuntimeException("error")).map { it * 2 }
    println(failedMap)  // Failure(java.lang.RuntimeException: error)
    
    // mapCatching - map แต่ catch exception
    val parsed = Result.success("42").mapCatching { it.toInt() * 2 }
    println(parsed)  // Success(84)
    
    val badParse = Result.success("abc").mapCatching { it.toInt() }
    println(badParse)  // Failure(NumberFormatException)
    
    // Chaining ด้วย fold + let
    val result = fetchUserFromDb(1)
        .mapCatching { user -> fetchUserOrders(user).getOrThrow() }
        .mapCatching { orders -> calculateDiscount(orders).getOrThrow() }
    
    result.fold(
        onSuccess = { discount -> println("Discount: ${discount * 100}%") },
        onFailure = { error -> println("Error: ${error.message}") }
    )
    
    // recover - แปลง failure เป็น success
    val recovered = Result.failure<Int>(RuntimeException("error"))
        .recover { 0 }  // คืน 0 ถ้า fail
    println(recovered)  // Success(0)
    
    // recoverCatching
    val recovered2 = Result.failure<Int>(RuntimeException("error"))
        .recoverCatching { "42".toInt() }
    println(recovered2)  // Success(42)
}
```

---

## 🎣 6. runCatching

`runCatching` เป็น shorthand สำหรับ `try-catch` ที่คืน `Result<T>`

```kotlin
fun main() {
    // runCatching - เรียบง่ายกว่า try-catch
    val result1 = runCatching { "42".toInt() }
    println(result1)  // Success(42)
    
    val result2 = runCatching { "abc".toInt() }
    println(result2)  // Failure(NumberFormatException)
    
    // ใช้กับ chain
    val processed = runCatching { "42".toInt() }
        .map { it * 2 }
        .map { "Result: $it" }
    println(processed.getOrDefault("Error"))  // Result: 84
    
    // Object.runCatching
    val str = "hello"
    val length = str.runCatching { length }
    println(length)  // Success(5)
    
    // Practical: safe JSON parsing
    data class Config(val host: String, val port: Int)
    
    fun parseConfig(json: String): Config? = runCatching {
        // simulate JSON parsing
        val parts = json.trim().removeSurrounding("{", "}")
            .split(",")
            .associate { part ->
                val (key, value) = part.split(":")
                key.trim().trim('"') to value.trim().trim('"')
            }
        Config(
            host = parts["host"] ?: throw IllegalArgumentException("Missing host"),
            port = parts["port"]?.toInt() ?: throw IllegalArgumentException("Missing port")
        )
    }.getOrNull()
    
    println(parseConfig("""{"host": "localhost", "port": "8080"}"""))
    println(parseConfig("""invalid json"""))
    
    // runCatching กับ coroutines
    suspend fun safeApiCall(): Result<String> = runCatching {
        // ถ้า exception ถูก throw, จะถูก wrap ใน Result.failure
        "response data"
    }
}
```

---

## ⚡ 7. Checked vs Unchecked Exceptions

### ใน Java: มี Checked Exceptions ที่ต้อง declare
### ใน Kotlin: ทุก exception เป็น Unchecked (ไม่บังคับ declare)

```kotlin
import java.io.IOException

// Kotlin ไม่มี checked exceptions
// แต่ถ้า call Java method ที่ throws checked exception
// Kotlin ไม่บังคับ catch แต่ควรจัดการ

// @Throws สำหรับ Java interoperability
@Throws(IOException::class)
fun readFileForJava(path: String): String {
    return java.io.File(path).readText()
}

// Kotlin style - ใช้ Result หรือ nullable
fun readFileKotlinStyle(path: String): String? {
    return runCatching { java.io.File(path).readText() }.getOrNull()
}

// ตัวอย่าง: Exception propagation
fun level3(): Int {
    throw RuntimeException("Error in level 3")
}

fun level2(): Int {
    return level3()  // Kotlin ไม่บังคับ declare throws
}

fun level1(): Int {
    return level2()
}

fun main() {
    // Exception propagates up the call stack
    try {
        level1()
    } catch (e: RuntimeException) {
        println("Caught at main: ${e.message}")
        // Stack trace
        e.printStackTrace()
    }
    
    // Exception chaining
    fun parseConfig(data: String): Map<String, String> {
        try {
            // parse logic
            throw IOException("File read error")
        } catch (e: IOException) {
            throw RuntimeException("Failed to parse config", e)  // wrap with context
        }
    }
    
    try {
        parseConfig("bad data")
    } catch (e: RuntimeException) {
        println("Error: ${e.message}")
        println("Caused by: ${e.cause?.message}")
    }
}
```

---

## 🔄 8. Exception Handling ใน Coroutines

### 8.1 Exceptions ใน launch

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    // Exception ใน launch - ถ้าไม่จัดการ จะ crash
    val job = launch {
        try {
            delay(100)
            throw RuntimeException("Error in coroutine!")
        } catch (e: RuntimeException) {
            println("Caught inside coroutine: ${e.message}")
        }
    }
    job.join()
    
    // Exception ที่ไม่ได้ catch จะ propagate ขึ้น
    try {
        launch {
            throw RuntimeException("Uncaught!")
        }.join()
    } catch (e: RuntimeException) {
        println("Caught from join: ${e.message}")
    }
}
```

### 8.2 CoroutineExceptionHandler

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    // CoroutineExceptionHandler - จัดการ uncaught exceptions
    val handler = CoroutineExceptionHandler { context, exception ->
        println("Caught by handler: ${exception.message}")
        println("Coroutine context: $context")
    }
    
    // ใช้กับ launch (root coroutine เท่านั้น!)
    val job = launch(handler) {
        throw RuntimeException("Error in root coroutine")
    }
    job.join()
    
    println("Main continues after error")
    
    // handler ไม่ทำงานกับ async (ต้องใช้ try-catch หรือ .await())
    val deferred = async {
        throw RuntimeException("Error in async")
    }
    
    try {
        deferred.await()
    } catch (e: RuntimeException) {
        println("Caught from async: ${e.message}")
    }
}
```

### 8.3 supervisorScope และ SupervisorJob

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    // supervisorScope - child failures ไม่ cancel siblings
    supervisorScope {
        val child1 = launch {
            delay(100)
            throw RuntimeException("Child 1 failed!")
        }
        
        val child2 = launch {
            repeat(5) { i ->
                delay(50)
                println("Child 2 step $i")
            }
        }
        
        // child1 fail แต่ child2 ยังทำงานต่อ
    }
    
    println("After supervisorScope")
    
    // SupervisorJob ใน production
    val supervisor = SupervisorJob()
    val scope = CoroutineScope(Dispatchers.Default + supervisor)
    
    val handler = CoroutineExceptionHandler { _, e ->
        println("Supervisor caught: ${e.message}")
    }
    
    scope.launch(handler) {
        delay(100)
        throw RuntimeException("Task 1 failed")
    }
    
    scope.launch {
        delay(200)
        println("Task 2 completed successfully")
    }
    
    delay(500)
    supervisor.cancel()
}
```

### 8.4 Exception Handling Best Practices ใน Coroutines

```kotlin
import kotlinx.coroutines.*

// Pattern 1: Try-catch ใน coroutine body
suspend fun safeOperation(): String {
    return try {
        riskyApiCall()
    } catch (e: Exception) {
        if (e is CancellationException) throw e  // สำคัญ! rethrow cancellation
        "default value"
    }
}

suspend fun riskyApiCall(): String {
    delay(100)
    if (Math.random() > 0.5) throw RuntimeException("API Error")
    return "success"
}

// Pattern 2: Result return type
suspend fun safeApiCall(): Result<String> = runCatching {
    riskyApiCall()
}

// Pattern 3: Sealed class
sealed class ApiResult<out T> {
    data class Success<T>(val data: T) : ApiResult<T>()
    data class Error(val exception: Exception, val message: String) : ApiResult<Nothing>()
    object Loading : ApiResult<Nothing>()
}

suspend fun fetchData(id: Int): ApiResult<String> {
    return try {
        val data = riskyApiCall()
        ApiResult.Success(data)
    } catch (e: CancellationException) {
        throw e
    } catch (e: Exception) {
        ApiResult.Error(e, "Failed to fetch data for id: $id")
    }
}

fun main() = runBlocking {
    // Pattern 1
    println(safeOperation())
    
    // Pattern 2
    val result = safeApiCall()
    result.fold(
        onSuccess = { println("Success: $it") },
        onFailure = { println("Failure: ${it.message}") }
    )
    
    // Pattern 3
    when (val apiResult = fetchData(1)) {
        is ApiResult.Success -> println("Got: ${apiResult.data}")
        is ApiResult.Error -> println("Error: ${apiResult.message}")
        is ApiResult.Loading -> println("Loading...")
    }
}
```

---

## 🌐 9. ตัวอย่าง: HTTP Client Error Handling

```kotlin
import kotlinx.coroutines.*

// HTTP response types
data class HttpRequest(val method: String, val url: String, val body: String? = null)
data class HttpResponse(val statusCode: Int, val body: String, val headers: Map<String, String> = emptyMap())

// Exception hierarchy สำหรับ HTTP
sealed class HttpException(message: String, cause: Throwable? = null) : Exception(message, cause) {
    abstract val statusCode: Int
    abstract val errorBody: String
}

class ClientException(
    override val statusCode: Int,
    override val errorBody: String,
    message: String
) : HttpException(message) {
    val isBadRequest get() = statusCode == 400
    val isUnauthorized get() = statusCode == 401
    val isForbidden get() = statusCode == 403
    val isNotFound get() = statusCode == 404
    val isConflict get() = statusCode == 409
    val isUnprocessable get() = statusCode == 422
}

class ServerException(
    override val statusCode: Int,
    override val errorBody: String,
    message: String
) : HttpException(message) {
    val isInternalError get() = statusCode == 500
    val isServiceUnavailable get() = statusCode == 503
}

class NetworkException(message: String, cause: Throwable? = null) :
    Exception(message, cause)

class TimeoutException(val timeoutMs: Long) :
    Exception("Request timed out after ${timeoutMs}ms")

// HTTP Client
class HttpClient(
    private val baseUrl: String,
    private val timeoutMs: Long = 5000
) {
    suspend fun get(path: String): HttpResponse = makeRequest(HttpRequest("GET", "$baseUrl$path"))
    suspend fun post(path: String, body: String): HttpResponse = makeRequest(HttpRequest("POST", "$baseUrl$path", body))
    suspend fun put(path: String, body: String): HttpResponse = makeRequest(HttpRequest("PUT", "$baseUrl$path", body))
    suspend fun delete(path: String): HttpResponse = makeRequest(HttpRequest("DELETE", "$baseUrl$path"))
    
    private suspend fun makeRequest(request: HttpRequest): HttpResponse {
        return withContext(Dispatchers.IO) {
            withTimeout(timeoutMs) {
                simulateRequest(request)
            }
        }
    }
    
    // Simulated request - ใน production จะใช้ OkHttp/Ktor
    private suspend fun simulateRequest(request: HttpRequest): HttpResponse {
        delay(200)  // simulate network latency
        
        return when {
            request.url.contains("/users/999") ->
                throw ClientException(404, """{"error":"User not found"}""", "Not Found")
            request.url.contains("/admin") ->
                throw ClientException(401, """{"error":"Unauthorized"}""", "Unauthorized")
            request.url.contains("/error") ->
                throw ServerException(500, """{"error":"Internal Server Error"}""", "Server Error")
            request.url.contains("/timeout") ->
                throw TimeoutCancellationException("Timeout")
            else ->
                HttpResponse(200, """{"data": "success"}""")
        }
    }
}

// Repository layer with error handling
class UserRepository(private val client: HttpClient) {
    
    suspend fun getUser(userId: Int): Result<HttpResponse> = runCatching {
        client.get("/users/$userId")
    }.mapCatching { response ->
        response  // additional validation here
    }
    
    suspend fun updateUser(userId: Int, data: String): Result<HttpResponse> {
        return try {
            val response = client.put("/users/$userId", data)
            Result.success(response)
        } catch (e: ClientException) {
            when {
                e.isNotFound -> Result.failure(NotFoundException("User", userId))
                e.isUnauthorized -> Result.failure(AuthException("Token expired"))
                e.isBadRequest -> Result.failure(ValidationException("Invalid user data: ${e.errorBody}"))
                else -> Result.failure(e)
            }
        } catch (e: ServerException) {
            Result.failure(RuntimeException("Server error, please try again"))
        } catch (e: TimeoutException) {
            Result.failure(RuntimeException("Request timed out"))
        } catch (e: NetworkException) {
            Result.failure(RuntimeException("Network unavailable"))
        }
    }
    
    suspend fun getUserSafely(userId: Int): User? {
        return getUser(userId)
            .map { response -> parseUser(response.body) }
            .getOrNull()
    }
    
    private fun parseUser(json: String): User = User(1, "Alice", "alice@example.com")
}

data class User(val id: Int, val name: String, val email: String)

// Error handling ระดับ Service
class UserService(private val repo: UserRepository) {
    
    suspend fun getUserProfile(userId: Int): UserProfileResult {
        return when {
            userId <= 0 -> UserProfileResult.ValidationError("Invalid user ID")
            else -> {
                val user = repo.getUserSafely(userId)
                if (user != null) {
                    UserProfileResult.Success(user)
                } else {
                    UserProfileResult.NotFound(userId)
                }
            }
        }
    }
}

sealed class UserProfileResult {
    data class Success(val user: User) : UserProfileResult()
    data class NotFound(val userId: Int) : UserProfileResult()
    data class ValidationError(val message: String) : UserProfileResult()
    data class ServerError(val message: String) : UserProfileResult()
}

fun main() = runBlocking {
    val client = HttpClient("https://api.example.com")
    val repo = UserRepository(client)
    val service = UserService(repo)
    
    println("=== HTTP Error Handling Demo ===\n")
    
    // Success case
    println("1. Fetch existing user:")
    val result1 = repo.getUser(1)
    result1.fold(
        onSuccess = { println("   Success: ${it.statusCode} - ${it.body}") },
        onFailure = { println("   Error: ${it.message}") }
    )
    
    // 404 case
    println("\n2. Fetch non-existent user:")
    val result2 = repo.getUser(999)
    result2.fold(
        onSuccess = { println("   Success: ${it.statusCode}") },
        onFailure = { println("   Error: ${it.message}") }
    )
    
    // Service level
    println("\n3. Service level handling:")
    listOf(0, 1, 999).forEach { userId ->
        when (val profile = service.getUserProfile(userId)) {
            is UserProfileResult.Success -> println("   User $userId: ${profile.user.name}")
            is UserProfileResult.NotFound -> println("   User $userId: not found")
            is UserProfileResult.ValidationError -> println("   User $userId: validation - ${profile.message}")
            is UserProfileResult.ServerError -> println("   User $userId: server error")
        }
    }
}
```

---

## 🗄️ 10. ตัวอย่าง: Database Error Handling

```kotlin
// Database exceptions
class DatabaseConnectionException(message: String, cause: Throwable? = null) :
    Exception(message, cause)

class QueryException(
    val query: String,
    message: String,
    cause: Throwable? = null
) : Exception(message, cause)

class TransactionException(message: String, cause: Throwable? = null) :
    Exception(message, cause)

class DuplicateKeyException(
    val tableName: String,
    val keyValue: Any
) : Exception("Duplicate key in $tableName: $keyValue")

class OptimisticLockException(
    val entityType: String,
    val entityId: Any
) : Exception("Concurrent modification detected for $entityType($entityId)")

// Simulated Database
object Database {
    private val users = mutableMapOf(
        1 to mapOf("id" to 1, "name" to "Alice", "email" to "alice@example.com", "version" to 1),
        2 to mapOf("id" to 2, "name" to "Bob", "email" to "bob@example.com", "version" to 1)
    )
    
    fun findById(id: Int): Map<String, Any>? {
        Thread.sleep(50)  // simulate DB latency
        return users[id]
    }
    
    fun insert(data: Map<String, Any>): Map<String, Any> {
        val id = data["id"] as Int
        if (users.containsKey(id)) {
            throw DuplicateKeyException("users", id)
        }
        users[id] = data + mapOf("version" to 1)
        return users[id]!!
    }
    
    fun update(id: Int, data: Map<String, Any>, expectedVersion: Int): Map<String, Any> {
        val current = users[id] ?: throw QueryException("UPDATE users", "User $id not found")
        val currentVersion = current["version"] as Int
        
        if (currentVersion != expectedVersion) {
            throw OptimisticLockException("User", id)
        }
        
        users[id] = current + data + mapOf("version" to currentVersion + 1)
        return users[id]!!
    }
    
    fun <T> transaction(block: () -> T): T {
        println("[DB] Transaction started")
        return try {
            val result = block()
            println("[DB] Transaction committed")
            result
        } catch (e: Exception) {
            println("[DB] Transaction rolled back: ${e.message}")
            throw TransactionException("Transaction failed", e)
        }
    }
}

// Repository with proper error handling
class UserRepositoryDb {
    
    fun findById(id: Int): Result<Map<String, Any>?> = runCatching {
        Database.findById(id)
    }.mapCatching { user ->
        user  // validate here if needed
    }
    
    fun create(id: Int, name: String, email: String): Result<Map<String, Any>> {
        if (name.isBlank()) return Result.failure(ValidationException("Name is required"))
        if (!email.contains("@")) return Result.failure(ValidationException("Invalid email: $email"))
        
        return runCatching {
            Database.insert(mapOf("id" to id, "name" to name, "email" to email))
        }.recoverCatching { e ->
            when (e) {
                is DuplicateKeyException -> throw ConflictException("User", "User with id $id already exists")
                else -> throw e
            }
        }
    }
    
    fun updateWithRetry(
        id: Int,
        data: Map<String, Any>,
        maxRetries: Int = 3
    ): Result<Map<String, Any>> {
        var lastError: Exception? = null
        
        repeat(maxRetries) { attempt ->
            // Get current version
            val current = Database.findById(id)
                ?: return Result.failure(NotFoundException("User", id))
            
            val currentVersion = current["version"] as Int
            
            try {
                val updated = Database.update(id, data, currentVersion)
                return Result.success(updated)
            } catch (e: OptimisticLockException) {
                println("Retry ${attempt + 1}/$maxRetries due to concurrent modification")
                lastError = e
                Thread.sleep(100L * (attempt + 1))  // backoff
            } catch (e: Exception) {
                return Result.failure(e)
            }
        }
        
        return Result.failure(lastError ?: RuntimeException("Max retries exceeded"))
    }
    
    fun <T> inTransaction(block: () -> T): Result<T> = runCatching {
        Database.transaction(block)
    }.mapFailure { e ->
        when (e) {
            is TransactionException -> DatabaseConnectionException("Transaction failed", e)
            else -> e
        }
    }
}

// Extension for mapping failure
fun <T> Result<T>.mapFailure(transform: (Throwable) -> Throwable): Result<T> =
    if (isFailure) Result.failure(transform(exceptionOrNull()!!))
    else this

fun main() {
    val repo = UserRepositoryDb()
    
    println("=== Database Error Handling ===\n")
    
    // Find user
    println("1. Find existing user:")
    repo.findById(1).fold(
        onSuccess = { println("   Found: $it") },
        onFailure = { println("   Error: ${it.message}") }
    )
    
    // Find non-existent
    println("\n2. Find non-existent user:")
    repo.findById(99).fold(
        onSuccess = { println("   Result: $it") },
        onFailure = { println("   Error: ${it.message}") }
    )
    
    // Create - validation error
    println("\n3. Create with invalid data:")
    repo.create(3, "", "invalid").fold(
        onSuccess = { println("   Created: $it") },
        onFailure = { println("   Validation: ${it.message}") }
    )
    
    // Create - success
    println("\n4. Create valid user:")
    repo.create(3, "Charlie", "charlie@example.com").fold(
        onSuccess = { println("   Created: $it") },
        onFailure = { println("   Error: ${it.message}") }
    )
    
    // Create - duplicate
    println("\n5. Create duplicate user:")
    repo.create(1, "Alice2", "alice2@example.com").fold(
        onSuccess = { println("   Created: $it") },
        onFailure = { println("   Conflict: ${it.message}") }
    )
    
    // Update with retry
    println("\n6. Update user:")
    repo.updateWithRetry(1, mapOf("name" to "Alice Updated")).fold(
        onSuccess = { println("   Updated: $it") },
        onFailure = { println("   Error: ${it.message}") }
    )
    
    // Transaction
    println("\n7. Transaction:")
    repo.inTransaction {
        Database.insert(mapOf("id" to 4, "name" to "David", "email" to "david@example.com"))
    }.fold(
        onSuccess = { println("   Transaction success: $it") },
        onFailure = { println("   Transaction failed: ${it.message}") }
    )
}
```

---

## 🏋️ แบบฝึกหัด

### ระดับ 1 (ง่าย)

1. เขียนฟังก์ชัน `safeParseDate(dateStr: String): LocalDate?` ที่ parse วันที่ในรูปแบบ "YYYY-MM-DD" แบบ safe โดยคืน null ถ้า format ไม่ถูกต้อง

2. สร้าง custom exception `InsufficientFundsException(balance: Double, amount: Double)` และเขียน class `BankAccount` ที่ใช้ exception นี้ใน `withdraw()` method

### ระดับ 2 (กลาง)

3. เขียน `retry` function:
   ```kotlin
   suspend fun <T> retry(
       times: Int,
       delay: Long = 1000,
       retryOn: (Exception) -> Boolean = { true },
       block: suspend () -> T
   ): T
   ```
   ทดสอบกับ function ที่ fail สุ่ม

4. สร้าง `ValidationBuilder` class ที่:
   - ใช้ builder pattern
   - เพิ่ม rules ได้: `notBlank()`, `minLength(n)`, `maxLength(n)`, `matches(regex)`
   - `validate(value)` คืน `Result<String>` พร้อม error messages

### ระดับ 3 (ท้าทาย)

5. สร้าง `ErrorBoundary` coroutine operator ที่:
   - Catch exceptions ใน coroutine scope
   - Retry N ครั้งด้วย exponential backoff
   - ส่ง error report เมื่อหมด retries
   - รองรับ fallback value

6. Implement `CircuitBreaker` pattern:
   - `CLOSED` state: ทำงานปกติ
   - `OPEN` state: fail fast ทันที (ไม่ call จริง)
   - `HALF_OPEN` state: ทดสอบ call 1 ครั้ง
   - transition logic ตาม failure rate

---

## 📊 สรุป

| Concept | ใช้เมื่อ | ตัวอย่าง |
|---------|---------|---------|
| `try-catch` | จัดการ exception ทั่วไป | IO, parsing |
| `try-catch-finally` | ต้องการ cleanup เสมอ | resource release |
| `try` as expression | คืนค่าจาก try block | `val x = try { ... }` |
| Custom exception | ต้องการ semantic ชัดเจน | `NotFoundException` |
| `Result<T>` | ส่ง error เป็นค่า | API return type |
| `runCatching` | shorthand Result | single operation |
| `CoroutineExceptionHandler` | handle coroutine errors | background tasks |
| `supervisorScope` | isolate child failures | independent tasks |

| Exception Type | เมื่อไหร่ | ตัวอย่าง |
|---------------|---------|---------|
| `IllegalArgumentException` | parameter ผิด | invalid input |
| `IllegalStateException` | object ในสถานะผิด | call order wrong |
| `NotFoundException` | resource ไม่พบ | 404 errors |
| `ValidationException` | data ไม่ valid | form validation |
| `NetworkException` | network error | connectivity issue |
| `TimeoutException` | เกิน time limit | slow API |

---

## ➡️ ถัดไป: Part 16 - File I/O

---
*Part 15/100+ | Kotlin & Spring Boot Complete Course*
