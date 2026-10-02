# Part 12: Null Safety ใน Kotlin

## ทำไม Null Safety ถึงสำคัญ?

Tony Hoare ผู้คิดค้น null reference กล่าวว่ามันเป็น "Billion-dollar mistake" เพราะทำให้เกิด `NullPointerException` (NPE) นับไม่ถ้วนในระบบ Java, C#, และภาษาอื่นๆ

Kotlin แก้ปัญหานี้โดยแยก nullable types ออกจาก non-nullable types ในระดับ type system ทำให้ compiler สามารถตรวจจับ null ได้ตั้งแต่เวลา compile

```kotlin
// Java - อาจเกิด NPE ตอน runtime
String name = null;
System.out.println(name.length()); // NullPointerException!

// Kotlin - compiler จะไม่ยอม compile
val name: String = null  // Compilation Error!

// Kotlin - ถ้าต้องการ nullable ต้องประกาศด้วย ?
val name: String? = null
println(name.length)  // Compilation Error! - ต้องจัดการ null ก่อน
```

---

## 12.1 Nullable Types อย่างละเอียด

### การประกาศ Nullable Types

```kotlin
// Non-nullable types
val name: String = "Alice"
val age: Int = 25
val isActive: Boolean = true

// Nullable types (เพิ่ม ? หลัง type)
val nullableName: String? = null
val nullableAge: Int? = null
val nullableList: List<String>? = null

// Nullable generic
val nullableItems: List<String?> = listOf("a", null, "b")  // List ที่มี null ใน items
val nullableList2: List<String>? = null  // nullable List ที่ไม่มี null ใน items
val bothNullable: List<String?>? = null  // ทั้ง list และ items nullable
```

### Nullable กับ Type Hierarchy

```kotlin
// String? เป็น supertype ของ String
val nonNull: String = "Hello"
val nullable: String? = nonNull  // OK: String เป็น String?

// String? ไม่ใช่ subtype ของ String
// val nonNull2: String = nullable  // Error!
```

### การเข้าถึง Nullable Types

```kotlin
fun demonstrateNullAccess() {
    val name: String? = "Alice"
    
    // 1. if-null check (เช็คก่อนใช้)
    if (name != null) {
        println(name.length)  // Smart cast: name เป็น String ใน block นี้
    }
    
    // 2. safe call (?.)
    println(name?.length)  // 5 หรือ null ถ้า name เป็น null
    
    // 3. Elvis operator (?:)
    val length = name?.length ?: 0  // 5 หรือ 0 ถ้า null
    
    // 4. Non-null assertion (!!)
    val forcedLength = name!!.length  // 5 หรือ NullPointerException ถ้า null
}
```

---

## 12.2 Smart Casts

Kotlin มี Smart Cast ที่ทำให้เราไม่ต้อง cast ด้วยตนเองหลังจากตรวจสอบ null แล้ว

```kotlin
fun processName(name: String?) {
    // หลังจาก null check, Kotlin รู้ว่า name ไม่ใช่ null
    if (name != null) {
        // ใน block นี้ name เป็น String (non-nullable)
        println(name.uppercase())
        println(name.length)
        println(name.reversed())
    }
}

fun processWithReturn(name: String?): String {
    // Early return pattern
    name ?: return "Unknown"
    // หลังจาก return แล้ว name ไม่ใช่ null
    return name.uppercase()
}

// Smart cast กับ is operator
fun processAny(value: Any?) {
    when (value) {
        null -> println("null")
        is String -> println("String: ${value.uppercase()}")  // value เป็น String
        is Int -> println("Int: ${value * 2}")                // value เป็น Int
        is List<*> -> println("List size: ${value.size}")     // value เป็น List
        else -> println("Other: $value")
    }
}

fun main() {
    processName("Alice")
    processName(null)
    
    println(processWithReturn("Bob"))   // BOB
    println(processWithReturn(null))    // Unknown
    
    processAny("Hello")     // String: HELLO
    processAny(42)          // Int: 84
    processAny(null)        // null
    processAny(listOf(1,2)) // List size: 2
}
```

### ข้อจำกัดของ Smart Cast

```kotlin
class Holder(var value: String?)

fun main() {
    val holder = Holder("Hello")
    
    // var properties ไม่สามารถ smart cast ได้
    // เพราะค่าอาจเปลี่ยนโดย thread อื่น
    if (holder.value != null) {
        // Compilation Error! holder.value อาจเป็น null ได้
        // println(holder.value.length)
        
        // ต้องใช้ local variable แทน
        val v = holder.value
        if (v != null) {
            println(v.length)  // Smart cast works for local val
        }
    }
}
```

---

## 12.3 Safe Call Chain (?.)

Safe call เป็น pattern ที่ทรงพลังมาก ช่วยให้เขียน code ที่จัดการ null ได้สั้นและกระชับ

```kotlin
data class Address(
    val street: String?,
    val city: String?,
    val country: String?
)

data class Person(
    val name: String,
    val address: Address?
)

fun main() {
    val person = Person(
        name = "Alice",
        address = Address(
            street = "123 Main St",
            city = "Bangkok",
            country = "Thailand"
        )
    )
    
    val personNoAddress = Person("Bob", null)
    
    // Safe call chain
    println(person.address?.city)          // Bangkok
    println(personNoAddress.address?.city) // null
    
    // Chain หลายระดับ
    println(person.address?.city?.uppercase())          // BANGKOK
    println(personNoAddress.address?.city?.uppercase()) // null
    
    // กับ let
    person.address?.city?.let { city ->
        println("City found: $city")
    }
    personNoAddress.address?.city?.let { city ->
        println("This won't print")
    }
    
    // กับ Elvis
    val city = person.address?.city ?: "Unknown City"
    val cityNoAddress = personNoAddress.address?.city ?: "Unknown City"
    println(city)          // Bangkok
    println(cityNoAddress) // Unknown City
}
```

### Safe Call กับ Collections

```kotlin
fun processUsers(users: List<User>?) {
    // Safe call บน collections
    val count = users?.size ?: 0
    val firstUser = users?.firstOrNull()
    val userNames = users?.map { it.name } ?: emptyList()
    
    println("Count: $count")
    println("First: ${firstUser?.name}")
    println("Names: $userNames")
}

data class User(val id: Int, val name: String, val email: String?)

fun main() {
    val users = listOf(
        User(1, "Alice", "alice@example.com"),
        User(2, "Bob", null),
        User(3, "Charlie", "charlie@example.com")
    )
    
    processUsers(users)
    processUsers(null)
    
    // กรอง null emails
    val validEmails = users.mapNotNull { it.email }
    println(validEmails)  // [alice@example.com, charlie@example.com]
    
    // ส่ง email ที่ valid
    users.forEach { user ->
        user.email?.let { email ->
            println("Sending to: $email")
        } ?: println("No email for ${user.name}")
    }
}
```

---

## 12.4 Elvis Operator (?:) Patterns

Elvis operator (`?:`) คืนค่า default เมื่อ expression ทางซ้ายเป็น null

```kotlin
// Pattern พื้นฐาน
val name: String? = null
val displayName = name ?: "Guest"  // "Guest"

// Elvis กับ throw
fun getUser(id: Int): User {
    val user: User? = findUser(id)
    return user ?: throw IllegalArgumentException("User $id not found")
}

// Elvis กับ return (early return)
fun processUser(id: Int): String {
    val user = findUser(id) ?: return "User not found"
    val email = user.email ?: return "Email not provided"
    return "Processing: ${user.name} <$email>"
}

// Elvis chain
fun getDisplayInfo(user: User?): String {
    return user?.name ?: user?.email?.substringBefore("@") ?: "Unknown"
}

// Elvis กับ functions ที่ return null
fun String?.toIntOrDefault(default: Int = 0): Int {
    return this?.toIntOrNull() ?: default
}

fun main() {
    println("42".toIntOrDefault())    // 42
    println("abc".toIntOrDefault())   // 0
    println(null.toIntOrDefault(99))  // 99
    
    // Nested Elvis
    val config: Map<String, String?> = mapOf(
        "host" to "localhost",
        "port" to null,
        "db" to "mydb"
    )
    
    val host = config["host"] ?: "defaulthost"
    val port = config["port"]?.toInt() ?: 5432
    val db = config["db"] ?: "defaultdb"
    val missing = config["timeout"]?.toInt() ?: 30
    
    println("$host:$port/$db (timeout: $missing)")
    // localhost:5432/mydb (timeout: 30)
}

fun findUser(id: Int): User? {
    return if (id == 1) User(1, "Alice", "alice@example.com") else null
}
```

---

## 12.5 Non-null Assertion (!!) และเมื่อไหรควรใช้

Non-null assertion (`!!`) บอก Kotlin ว่า "ฉันมั่นใจว่าค่านี้ไม่เป็น null" ซึ่งอาจทำให้เกิด `KotlinNullPointerException` ถ้าค่าเป็น null จริงๆ

```kotlin
// เมื่อไหรควรใช้ !!

// 1. เมื่อ logic ของโปรแกรมรับประกันว่าไม่เป็น null
//    แต่ compiler ไม่สามารถพิสูจน์ได้
fun processInput(userInput: String) {
    // user input validated ก่อนแล้ว ไม่มีทางเป็น null
    val trimmed = userInput.trim()
    if (trimmed.isNotEmpty()) {
        val firstChar = trimmed.firstOrNull()!! // รู้ว่าไม่ null เพราะ isNotEmpty()
        println("First char: $firstChar")
    }
}

// 2. ใน test code ที่ต้องการ fail fast
class UserRepositoryTest {
    fun testGetUser() {
        val user = repository.findById(1)!!  // ควรเป็น null ใน test นี้
        assertEquals("Alice", user.name)
    }
}

// ❌ ไม่ควรใช้ !! แบบนี้
fun badPractice(name: String?) {
    println(name!!.length)  // อาจเกิด NPE ถ้า name เป็น null
}

// ✅ ควรใช้แบบนี้แทน
fun goodPractice(name: String?) {
    println(name?.length ?: 0)
}

// เมื่อมีหลาย !! ให้แยก expression
fun avoidChaining(a: User?, b: Address?, c: String?) {
    // ❌ ยากที่จะรู้ว่า NPE เกิดที่ไหน
    // val result = a!!.address!!.city!!.uppercase()
    
    // ✅ แยกออกมาเพื่อ debug ง่าย
    val user = a ?: return
    val address = user.address ?: return
    val city = address.city ?: "Unknown"
    val result = city.uppercase()
    println(result)
}
```

---

## 12.6 let, run กับ Nullable Types

`let` และ `run` เป็น scope functions ที่ใช้บ่อยมากกับ nullable types

```kotlin
// let - ทำงานกับ non-null value
fun processEmail(email: String?) {
    // ถ้า email ไม่เป็น null จะเข้า block
    email?.let { nonNullEmail ->
        println("Sending to: $nonNullEmail")
        sendEmail(nonNullEmail)
    }
    
    // สั้นกว่าด้วย it
    email?.let { sendEmail(it) }
}

// let กับ chain
fun processUser(userId: Int) {
    findUser(userId)
        ?.let { user ->
            println("Found: ${user.name}")
            user.email
        }
        ?.let { email ->
            println("Email: $email")
            sendWelcomeEmail(email)
        }
        ?: println("User not found or no email")
}

// run - ทำงานใน context ของ object
data class Config(
    var host: String = "localhost",
    var port: Int = 8080,
    var dbUrl: String = ""
)

fun loadConfig(configMap: Map<String, String>?): Config {
    return configMap?.run {
        Config(
            host = get("host") ?: "localhost",
            port = get("port")?.toInt() ?: 8080,
            dbUrl = get("db_url") ?: "jdbc:h2:mem:test"
        )
    } ?: Config()
}

// let สำหรับ transformation chain
fun transformData(input: String?): Int? {
    return input
        ?.trim()
        ?.takeIf { it.isNotEmpty() }
        ?.let { it.toIntOrNull() }
        ?.let { it * 2 }
        ?.takeIf { it > 0 }
}

fun main() {
    println(transformData("  21  "))  // 42
    println(transformData(""))        // null
    println(transformData("abc"))     // null
    println(transformData("-5"))      // null (ลบ > 0 check)
    println(transformData(null))      // null
}

fun sendEmail(email: String) = println("Email sent to $email")
fun sendWelcomeEmail(email: String) = println("Welcome email sent to $email")
```

### also กับ Nullable

```kotlin
// also - side effect โดยไม่เปลี่ยนค่า
fun loadData(id: Int): List<String>? {
    return fetchFromDatabase(id)
        ?.also { data ->
            println("Loaded ${data.size} items")
            logAccess(id)
        }
}

fun processOptionalList(items: List<String>?): String {
    return items
        ?.also { println("Processing ${it.size} items") }
        ?.filter { it.isNotEmpty() }
        ?.also { println("After filter: ${it.size} items") }
        ?.joinToString(", ")
        ?: "No items"
}

fun fetchFromDatabase(id: Int): List<String>? {
    return if (id > 0) listOf("item1", "item2", "item3") else null
}

fun logAccess(id: Int) = println("Accessed: $id")
```

---

## 12.7 requireNotNull และ checkNotNull

ใน Kotlin มีฟังก์ชันมาตรฐานสำหรับ validation ที่ช่วยให้ code อ่านง่ายกว่า if-null checks

```kotlin
fun processOrder(orderId: Int?, userId: Int?, amount: Double?) {
    // requireNotNull - throw IllegalArgumentException ถ้า null
    val id = requireNotNull(orderId) { "Order ID cannot be null" }
    val user = requireNotNull(userId) { "User ID cannot be null" }
    val price = requireNotNull(amount) { "Amount cannot be null" }
    
    println("Processing order $id for user $user: $$price")
}

fun processState(data: String?) {
    // checkNotNull - throw IllegalStateException ถ้า null
    // ใช้เมื่อ null หมายความว่า state ของโปรแกรมผิดปกติ
    val value = checkNotNull(data) { "Data must be initialized before processing" }
    println("Processing: $value")
}

// ตัวอย่าง: Spring Service
class OrderService {
    private var repository: OrderRepository? = null
    
    fun initialize(repo: OrderRepository) {
        repository = repo
    }
    
    fun createOrder(customerId: Int, items: List<OrderItem>): Order {
        // checkNotNull - repository ต้องถูก initialize ก่อน
        val repo = checkNotNull(repository) { 
            "OrderService must be initialized with a repository" 
        }
        
        // requireNotNull - input validation
        requireNotNull(customerId.takeIf { it > 0 }) { 
            "Customer ID must be positive" 
        }
        require(items.isNotEmpty()) { "Order must have at least one item" }
        
        return repo.save(Order(customerId = customerId, items = items))
    }
}

// ความแตกต่างระหว่าง require, check, error
fun validateAge(age: Int?) {
    // require - precondition (argument validation)
    val validAge = requireNotNull(age) { "Age cannot be null" }
    require(validAge >= 0) { "Age cannot be negative: $validAge" }
    require(validAge <= 150) { "Age seems unrealistic: $validAge" }
    
    println("Valid age: $validAge")
}

fun main() {
    // ทดสอบ requireNotNull
    try {
        processOrder(null, 1, 100.0)
    } catch (e: IllegalArgumentException) {
        println("Caught: ${e.message}")  // Caught: Order ID cannot be null
    }
    
    processOrder(1, 2, 99.99)  // Processing order 1 for user 2: $99.99
    
    // ทดสอบ validateAge
    try {
        validateAge(-5)
    } catch (e: IllegalArgumentException) {
        println("Caught: ${e.message}")  // Age cannot be negative: -5
    }
}
```

---

## 12.8 Platform Types (Java Interop)

เมื่อเรียกใช้ Java code จาก Kotlin, Kotlin ไม่รู้ว่า Java types เป็น nullable หรือ non-nullable (เพราะ Java ไม่มี type system สำหรับ null)

```java
// Java class
public class JavaUser {
    private String name;
    private String email; // อาจเป็น null
    
    public JavaUser(String name, String email) {
        this.name = name;
        this.email = email;
    }
    
    public String getName() { return name; }
    public String getEmail() { return email; }
    
    // กับ @Nullable annotation
    @Nullable
    public String getNickname() { return null; }
    
    // กับ @NotNull annotation
    @NotNull
    public String getId() { return "user-123"; }
}
```

```kotlin
// Kotlin side - จัดการ platform types
fun processJavaUser(javaUser: JavaUser) {
    // name เป็น platform type String! (ไม่รู้ว่า nullable หรือ non-nullable)
    val name = javaUser.name  // String! - อาจเป็น null
    
    // ใช้อย่างปลอดภัย
    val safeName: String? = javaUser.name  // assign เป็น nullable
    val displayName = safeName ?: "Unknown"
    
    // หรือใช้ safe call
    println(javaUser.name?.uppercase())
    
    // กับ @Nullable annotation - Kotlin รู้ว่าเป็น nullable
    val nickname: String? = javaUser.nickname  // ชัดเจนว่า nullable
    
    // กับ @NotNull annotation - Kotlin รู้ว่าเป็น non-nullable
    val id: String = javaUser.id  // ชัดเจนว่า non-nullable
}

// Best practice: ห่อหุ้ม Java API
data class KotlinUser(
    val name: String,
    val email: String?,
    val nickname: String?,
    val id: String
)

fun JavaUser.toKotlinUser(): KotlinUser {
    return KotlinUser(
        name = this.name ?: "Unknown",  // จัดการ null อย่างชัดเจน
        email = this.email,              // nullable
        nickname = this.nickname,        // @Nullable
        id = this.id                     // @NotNull
    )
}
```

---

## 12.9 ตัวอย่างจริง: JSON Parsing

```kotlin
import com.fasterxml.jackson.module.kotlin.jacksonObjectMapper
import com.fasterxml.jackson.module.kotlin.readValue

// Data classes สำหรับ API response
data class ApiResponse<T>(
    val status: String,
    val data: T?,
    val error: ErrorInfo?
)

data class ErrorInfo(
    val code: Int,
    val message: String,
    val details: String?
)

data class UserProfile(
    val id: Long,
    val username: String,
    val email: String?,
    val phone: String?,
    val avatar: String?,
    val address: UserAddress?
)

data class UserAddress(
    val street: String?,
    val city: String?,
    val country: String?,
    val zipCode: String?
)

// Service ที่จัดการ null safety อย่างถูกต้อง
class UserProfileService {
    
    fun parseUserProfile(json: String): UserProfile? {
        return try {
            val mapper = jacksonObjectMapper()
            val response: ApiResponse<UserProfile> = mapper.readValue(json)
            
            if (response.status == "success") {
                response.data
            } else {
                println("API Error: ${response.error?.message ?: "Unknown error"}")
                null
            }
        } catch (e: Exception) {
            println("Parse error: ${e.message}")
            null
        }
    }
    
    fun getDisplayName(profile: UserProfile): String {
        return profile.username
    }
    
    fun getContactInfo(profile: UserProfile): String {
        val email = profile.email ?: "No email"
        val phone = profile.phone ?: "No phone"
        return "Email: $email, Phone: $phone"
    }
    
    fun getLocationSummary(profile: UserProfile): String {
        return profile.address?.let { addr ->
            buildString {
                addr.street?.let { append("$it, ") }
                addr.city?.let { append("$it, ") }
                addr.country?.let { append(it) }
            }.trimEnd(',', ' ')
        } ?: "No address"
    }
    
    fun processProfile(json: String): String {
        val profile = parseUserProfile(json)
            ?: return "Failed to parse profile"
        
        return buildString {
            appendLine("=== User Profile ===")
            appendLine("Name: ${getDisplayName(profile)}")
            appendLine("Contact: ${getContactInfo(profile)}")
            appendLine("Location: ${getLocationSummary(profile)}")
            profile.avatar?.let { appendLine("Avatar: $it") }
        }
    }
}

fun main() {
    val service = UserProfileService()
    
    val jsonWithFullData = """
        {
            "status": "success",
            "data": {
                "id": 1,
                "username": "alice",
                "email": "alice@example.com",
                "phone": "0812345678",
                "avatar": "https://example.com/avatar.jpg",
                "address": {
                    "street": "123 Main St",
                    "city": "Bangkok",
                    "country": "Thailand",
                    "zipCode": "10100"
                }
            },
            "error": null
        }
    """.trimIndent()
    
    println(service.processProfile(jsonWithFullData))
    
    val jsonPartial = """
        {
            "status": "success",
            "data": {
                "id": 2,
                "username": "bob",
                "email": null,
                "phone": null,
                "avatar": null,
                "address": null
            },
            "error": null
        }
    """.trimIndent()
    
    println(service.processProfile(jsonPartial))
}
```

---

## 12.10 ตัวอย่างจริง: API Response Handling

```kotlin
// ตัวอย่างใน Spring Boot Controller

// Result wrapper type
sealed class Result<out T> {
    data class Success<T>(val data: T) : Result<T>()
    data class Failure(
        val error: String,
        val code: Int = 500,
        val cause: Throwable? = null
    ) : Result<Nothing>()
}

// Extension functions สำหรับ Result
fun <T> Result<T>.getOrNull(): T? = when (this) {
    is Result.Success -> data
    is Result.Failure -> null
}

fun <T> Result<T>.getOrThrow(): T = when (this) {
    is Result.Success -> data
    is Result.Failure -> throw RuntimeException(error, cause)
}

fun <T, R> Result<T>.map(transform: (T) -> R): Result<R> = when (this) {
    is Result.Success -> Result.Success(transform(data))
    is Result.Failure -> this
}

fun <T> Result<T>.onSuccess(action: (T) -> Unit): Result<T> {
    if (this is Result.Success) action(data)
    return this
}

fun <T> Result<T>.onFailure(action: (String) -> Unit): Result<T> {
    if (this is Result.Failure) action(error)
    return this
}

// Repository interface
interface UserRepository {
    fun findById(id: Long): User?
    fun findByEmail(email: String): User?
    fun save(user: User): User
}

// Service ที่ใช้ Result pattern
class UserService(private val repository: UserRepository) {
    
    fun getUserById(id: Long): Result<User> {
        if (id <= 0) {
            return Result.Failure("Invalid user ID: $id", code = 400)
        }
        
        val user = repository.findById(id)
            ?: return Result.Failure("User $id not found", code = 404)
        
        return Result.Success(user)
    }
    
    fun getUserByEmail(email: String): Result<User> {
        if (!email.contains("@")) {
            return Result.Failure("Invalid email format", code = 400)
        }
        
        val user = repository.findByEmail(email)
            ?: return Result.Failure("User with email $email not found", code = 404)
        
        return Result.Success(user)
    }
    
    fun updateUserEmail(userId: Long, newEmail: String): Result<User> {
        val existingResult = getUserById(userId)
        
        return existingResult.map { user ->
            val emailExists = repository.findByEmail(newEmail) != null
            if (emailExists) {
                throw IllegalArgumentException("Email already in use: $newEmail")
            }
            repository.save(user.copy(email = newEmail))
        }
    }
}

// Controller ที่ใช้ Result
class UserController(private val userService: UserService) {
    
    data class ApiResponse<T>(
        val success: Boolean,
        val data: T? = null,
        val error: String? = null,
        val code: Int = 200
    )
    
    fun getUser(id: Long): ApiResponse<User> {
        return userService.getUserById(id)
            .map { user -> ApiResponse(success = true, data = user) }
            .getOrElse { failure ->
                ApiResponse(success = false, error = failure.error, code = failure.code)
            }
    }
}

fun <T> Result<T>.getOrElse(default: (Result.Failure) -> T): T = when (this) {
    is Result.Success -> data
    is Result.Failure -> default(this)
}

// ตัวอย่างการใช้งาน
fun main() {
    val mockRepo = object : UserRepository {
        private val users = mutableMapOf(
            1L to User(1, "Alice", "alice@example.com"),
            2L to User(2, "Bob", "bob@example.com")
        )
        
        override fun findById(id: Long) = users[id]
        override fun findByEmail(email: String) = users.values.find { it.email == email }
        override fun save(user: User): User {
            users[user.id] = user
            return user
        }
    }
    
    val service = UserService(mockRepo)
    
    service.getUserById(1)
        .onSuccess { println("Found: ${it.name}") }
        .onFailure { println("Error: $it") }
    // Found: Alice
    
    service.getUserById(999)
        .onSuccess { println("Found: ${it.name}") }
        .onFailure { println("Error: $it") }
    // Error: User 999 not found
    
    val result = service.getUserById(1)
    println(result.getOrNull()?.name)   // Alice
    println(result.map { it.name })     // Success(data=Alice)
}
```

---

## 12.11 Null Safety Patterns สรุป

```kotlin
// Pattern 1: Elvis with throw
fun requireUser(id: Long, repo: UserRepository): User {
    return repo.findById(id) ?: throw NoSuchElementException("User $id not found")
}

// Pattern 2: Early return
fun processUserData(id: Long, repo: UserRepository): String {
    val user = repo.findById(id) ?: return "Not found"
    val email = user.email ?: return "No email"
    return "Processing ${user.name} <$email>"
}

// Pattern 3: let chain
fun getUserSummary(id: Long, repo: UserRepository): String {
    return repo.findById(id)
        ?.let { user -> "${user.name} (${user.email ?: "no email"})" }
        ?: "User not found"
}

// Pattern 4: fold-like null handling
fun <T, R> T?.whenNotNull(
    ifNull: () -> R,
    ifNotNull: (T) -> R
): R = if (this != null) ifNotNull(this) else ifNull()

// Pattern 5: Collection null handling
fun processNullableList(items: List<String?>): List<String> {
    return items.filterNotNull()
}

// Pattern 6: Safe navigation with default
data class Configuration(
    val database: DatabaseConfig?,
    val server: ServerConfig?
)

data class DatabaseConfig(val host: String?, val port: Int?, val name: String?)
data class ServerConfig(val host: String?, val port: Int?)

fun getDbConnectionString(config: Configuration?): String {
    val host = config?.database?.host ?: "localhost"
    val port = config?.database?.port ?: 5432
    val name = config?.database?.name ?: "default"
    return "jdbc:postgresql://$host:$port/$name"
}

fun main() {
    // Pattern demos
    val fullConfig = Configuration(
        database = DatabaseConfig("prod-db.example.com", 5432, "myapp"),
        server = ServerConfig("0.0.0.0", 8080)
    )
    
    val partialConfig = Configuration(
        database = DatabaseConfig(null, null, "myapp"),
        server = null
    )
    
    println(getDbConnectionString(fullConfig))
    // jdbc:postgresql://prod-db.example.com:5432/myapp
    
    println(getDbConnectionString(partialConfig))
    // jdbc:postgresql://localhost:5432/myapp
    
    println(getDbConnectionString(null))
    // jdbc:postgresql://localhost:5432/default
    
    // whenNotNull usage
    val value: String? = "Hello"
    val result = value.whenNotNull(
        ifNull = { "was null" },
        ifNotNull = { it.uppercase() }
    )
    println(result)  // HELLO
}
```

---

## สรุปบทที่ 12

| Pattern | เมื่อไหรใช้ |
|---------|------------|
| `?.` (safe call) | เมื่อต้องการ chain operations บน nullable |
| `?:` (Elvis) | เมื่อต้องการ default value หรือ early return |
| `!!` (non-null assert) | ใช้เมื่อมั่นใจ 100% ว่าไม่ null (ระวัง!) |
| `let` | ทำงานใน block เฉพาะเมื่อไม่เป็น null |
| `run` | คำนวณใน object context เมื่อไม่เป็น null |
| `requireNotNull` | Validate argument ที่ต้องไม่เป็น null |
| `checkNotNull` | Validate state ที่ต้องไม่เป็น null |
| `filterNotNull()` | กรอง null ออกจาก collection |

**หลักการสำคัญ:**
- ใช้ `?` เฉพาะเมื่อค่านั้นสามารถเป็น null จริงๆ
- หลีกเลี่ยง `!!` ให้มากที่สุด
- ใช้ early return pattern เพื่อลด nesting
- ห่อหุ้ม Java APIs ด้วย Kotlin types ที่ชัดเจน

---

## แบบฝึกหัดบทที่ 12

### ระดับง่าย

1. เขียนฟังก์ชัน `safeDiv(a: Int, b: Int): Int?` ที่คืน null เมื่อ b เป็น 0

2. เขียน extension function `String?.toIntSafe(): Int` ที่คืน 0 ถ้า string เป็น null หรือ convert ไม่ได้

3. เขียนฟังก์ชัน `firstPositive(numbers: List<Int?>): Int?` ที่คืนตัวเลขบวกตัวแรกที่ไม่ใช่ null

### ระดับกลาง

4. สร้าง `SafeConfig` class ที่อ่าน configuration จาก Map ที่อาจมี null values:
   ```kotlin
   class SafeConfig(private val map: Map<String, String?>) {
       fun getString(key: String, default: String = ""): String
       fun getInt(key: String, default: Int = 0): Int
       fun getBoolean(key: String, default: Boolean = false): Boolean
   }
   ```

5. เขียน function chain สำหรับ parse และ validate user input:
   - รับ `Map<String, String?>` เป็น input
   - Validate ว่า "name", "email", "age" มีค่า
   - Validate format ของ email
   - Parse age เป็น Int และตรวจสอบว่า >= 18
   - คืน `Result<User>` 

6. สร้าง deep null-safe accessor สำหรับ JSON-like structure:
   ```kotlin
   val data: Map<String, Any?> = mapOf(...)
   val city = data.getPath("address", "city") as? String
   ```

### ระดับยาก

7. Implement `Option<T>` type (คล้าย Haskell's Maybe) ด้วย Kotlin:
   - `Some<T>` และ `None`
   - `map`, `flatMap`, `getOrElse`, `filter`
   - Extension function `T?.toOption(): Option<T>`

8. สร้าง null-safe JSON builder:
   ```kotlin
   val json = jsonObject {
       "name" to user.name
       "email" to user.email  // ถ้า null จะไม่ใส่ field นี้
       "age" to user.age.takeIf { it > 0 }
   }
   ```

---

[ไปต่อ Part 13: Generics →](part-13-generics.md)

---

*Part 12/100+ | Kotlin & Spring Boot Complete Course*
