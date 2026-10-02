# Part 18: Scope Functions
## let, run, with, apply, also

---

## 🎯 เป้าหมายของ Part นี้

- เข้าใจ scope functions ทั้ง 5 ตัว
- รู้ว่าเมื่อไหรควรใช้ตัวไหน
- ใช้ร่วมกับ nullable types
- เข้าใจ `this` vs `it` ใน scope functions
- ตัวอย่างจริงและ best practices

---

## 📊 1. ภาพรวม Scope Functions

| Function | Object ref | Return value | Use case |
|----------|-----------|-------------|----------|
| `let`   | `it`      | Lambda result | nullable check, transform |
| `run`   | `this`    | Lambda result | object config + result |
| `with`  | `this`    | Lambda result | สำหรับ non-extension |
| `apply` | `this`    | Object itself | object configuration |
| `also`  | `it`      | Object itself | side effects |

---

## 🔹 2. let

```kotlin
// let: object เข้าถึงผ่าน `it`, คืน lambda result

// ใช้บ่อยที่สุดกับ nullable
val name: String? = "Alice"

// แบบปกติ
if (name != null) {
    println(name.uppercase())
}

// แบบ let
name?.let { println(it.uppercase()) }

// let กับ chain
val result = name
    ?.let { it.trim() }
    ?.let { it.lowercase() }
    ?.let { "Hello, $it!" }
println(result)  // Hello, alice!

// let ใช้ transformer
val numbers = listOf(1, 2, 3, 4, 5)
val sum = numbers.let { list ->
    println("Processing ${list.size} numbers")
    list.sum()
}
println(sum)  // 15

// let สำหรับ temp variable ใน expression
val text = "  Hello, World!  "
val processed = text.let {
    val trimmed = it.trim()
    val lower = trimmed.lowercase()
    lower.replace(",", "")
}
println(processed)  // hello world!
```

---

## 🔹 3. run

```kotlin
// run: object เข้าถึงผ่าน `this`, คืน lambda result

class DatabaseConfig(
    var host: String = "localhost",
    var port: Int = 5432,
    var name: String = "mydb",
    var poolSize: Int = 10
) {
    fun connectionString() = "jdbc:postgresql://$host:$port/$name"
}

// run กับ object
val config = DatabaseConfig().run {
    host = "prod-server.com"
    port = 5433
    name = "production"
    poolSize = 50
    connectionString()  // return value
}
println(config)  // jdbc:postgresql://prod-server.com:5433/production

// run สำหรับ execute block และ return
val result = run {
    val a = 10
    val b = 20
    val c = a + b
    "Sum of $a and $b is $c"
}
println(result)  // Sum of 10 and 20 is 30

// run กับ nullable
val email: String? = "  ALICE@EXAMPLE.COM  "
val normalized = email?.run {
    trim().lowercase()
}
println(normalized)  // alice@example.com
```

---

## 🔹 4. with

```kotlin
// with: non-extension function, object เป็น argument, เข้าถึงผ่าน `this`

data class User(
    val name: String,
    val email: String,
    val age: Int,
    val city: String
)

val user = User("Alice Smith", "alice@example.com", 25, "Bangkok")

// with สำหรับ operations หลายอย่างบน object เดียว
val profile = with(user) {
    """
    === User Profile ===
    Name: $name
    Email: $email
    Age: $age
    City: $city
    ====================
    """.trimIndent()
}
println(profile)

// with ใน builder pattern
val sb = StringBuilder()
with(sb) {
    append("Hello, ")
    append(user.name)
    append("!\n")
    append("Welcome to our service.")
}
println(sb)

// with สำหรับ calculations
val stats = with(listOf(10, 20, 30, 40, 50)) {
    mapOf(
        "count" to size,
        "sum" to sum(),
        "avg" to average(),
        "min" to min(),
        "max" to max()
    )
}
println(stats)
```

---

## 🔹 5. apply

```kotlin
// apply: object เข้าถึงผ่าน `this`, คืน object เอง (สำหรับ configure)

data class HttpRequest(
    var method: String = "GET",
    var url: String = "",
    var headers: MutableMap<String, String> = mutableMapOf(),
    var body: String? = null,
    var timeout: Int = 30000
)

// apply ใช้สำหรับ configure object
val request = HttpRequest().apply {
    method = "POST"
    url = "https://api.example.com/users"
    headers["Content-Type"] = "application/json"
    headers["Authorization"] = "Bearer token123"
    body = """{"name": "Alice", "email": "alice@example.com"}"""
    timeout = 60000
}

println("${request.method} ${request.url}")

// apply กับ Android/UI pattern
class Button {
    var text = ""
    var color = "white"
    var fontSize = 14
    var isEnabled = true
    
    fun show() = println("Button: '$text' [$color, ${fontSize}px, enabled=$isEnabled]")
}

val button = Button().apply {
    text = "Click Me!"
    color = "blue"
    fontSize = 18
    isEnabled = true
}.also { it.show() }

// apply สำหรับ builder-like
class PersonBuilder {
    var name = ""
    var age = 0
    var email = ""
    
    fun build() = "Person($name, $age, $email)"
}

val person = PersonBuilder().apply {
    name = "Alice"
    age = 25
    email = "alice@example.com"
}.build()

println(person)
```

---

## 🔹 6. also

```kotlin
// also: object เข้าถึงผ่าน `it`, คืน object เอง (สำหรับ side effects)

// also ดีสำหรับ logging / debugging
val numbers = mutableListOf(1, 2, 3)
    .also { println("Initial list: $it") }
    .also { it.add(4) }
    .also { println("After adding 4: $it") }
    .also { it.removeIf { n -> n % 2 == 0 } }
    .also { println("After removing evens: $it") }

// also สำหรับ validation chain
data class Registration(val email: String, val password: String)

fun validate(reg: Registration): Registration = reg
    .also { require(it.email.contains("@")) { "Invalid email" } }
    .also { require(it.password.length >= 8) { "Password too short" } }
    .also { println("Validation passed for ${it.email}") }

try {
    val reg = validate(Registration("alice@example.com", "password123"))
    println("Registered: $reg")
} catch (e: IllegalArgumentException) {
    println("Validation error: ${e.message}")
}

// also สำหรับ assign + log
class OrderService {
    private val orders = mutableListOf<String>()
    
    fun createOrder(item: String): String {
        val orderId = "ORD-${System.currentTimeMillis()}"
        return orderId
            .also { orders.add(it) }
            .also { println("Order created: $it for $item") }
    }
}
```

---

## 🔀 7. Combining Scope Functions

```kotlin
data class Address(
    var street: String = "",
    var city: String = "",
    var country: String = "Thailand",
    var zipCode: String = ""
)

data class Person(
    var name: String = "",
    var age: Int = 0,
    var email: String? = null,
    var address: Address = Address()
)

fun createPerson(
    name: String,
    age: Int,
    email: String?,
    street: String,
    city: String
): Person {
    return Person().apply {
        this.name = name
        this.age = age
        this.email = email
        address.apply {
            this.street = street
            this.city = city
        }
    }.also {
        println("Created person: ${it.name}")
    }
}

fun processPersonEmail(person: Person): String? {
    return person.email?.let {
        it.trim().lowercase()
    }?.run {
        if (contains("@")) this else null
    }?.also {
        println("Processing email: $it")
    }
}

fun main() {
    val person = createPerson(
        "Alice Smith", 25, "  ALICE@EXAMPLE.COM  ",
        "123 Main St", "Bangkok"
    )
    
    val email = processPersonEmail(person)
    println("Final email: $email")
    
    // Chain ยาว
    val report = person
        .let { p ->
            with(p) {
                "Name: $name, Age: $age, City: ${address.city}"
            }
        }
        .also { println("Report: $it") }
        .run { uppercase() }
    
    println(report)
}
```

---

## 💡 8. Best Practices

```kotlin
// ✅ ใช้ let กับ nullable
val user: User? = findUser(id)
user?.let { processUser(it) }

// ✅ ใช้ apply กับ object initialization
val config = Config().apply {
    timeout = 5000
    retries = 3
}

// ✅ ใช้ also กับ logging
val result = computeExpensiveValue()
    .also { log.debug("Computed: $it") }

// ✅ ใช้ run กับ computation ที่ต้องการ result
val message = connection.run {
    val data = fetch()
    process(data)
}

// ✅ ใช้ with กับ non-extension context
with(context) {
    startActivity(intent)
    overridePendingTransition(0, 0)
}

// ❌ อย่าซ้อน scope functions หลายชั้นโดยไม่จำเป็น
// ยากอ่าน!
val bad = obj.let { it.also { it.run { apply { } } } }

// ✅ แบ่งเป็นขั้นตอน
val step1 = obj.apply { configure() }
val step2 = step1.also { validate() }
val result2 = step2.let { transform(it) }
```

---

## 🏋️ 9. แบบฝึกหัด

### ข้อ 1: Builder pattern
```kotlin
data class Email(
    val to: String,
    val cc: List<String> = emptyList(),
    val subject: String = "",
    val body: String = "",
    val isHtml: Boolean = false
)

class EmailBuilder {
    private var to = ""
    private val cc = mutableListOf<String>()
    private var subject = ""
    private var body = ""
    private var isHtml = false
    
    fun to(address: String) = apply { to = address }
    fun cc(address: String) = apply { cc.add(address) }
    fun subject(text: String) = apply { subject = text }
    fun body(text: String) = apply { body = text }
    fun html(flag: Boolean = true) = apply { isHtml = flag }
    fun build() = Email(to, cc.toList(), subject, body, isHtml)
}

fun email(block: EmailBuilder.() -> Unit): Email {
    return EmailBuilder().apply(block).build()
}

fun main() {
    val mail = email {
        to("alice@example.com")
        cc("bob@example.com")
        subject("Hello!")
        body("<h1>Hello World</h1>")
        html()
    }
    println(mail)
}
```

### ข้อ 2: Logging with also
```kotlin
fun <T> T.log(tag: String = ""): T = also {
    val prefix = if (tag.isNotEmpty()) "[$tag] " else ""
    println("${prefix}$it")
}

fun processOrder(orderId: String, amount: Double): String {
    return orderId
        .log("INPUT")
        .also { require(amount > 0) { "Amount must be positive" } }
        .let { id -> "$id: ${"%.2f".format(amount)} THB" }
        .log("OUTPUT")
}

fun main() {
    println(processOrder("ORD-001", 1500.0))
    // [INPUT] ORD-001
    // [OUTPUT] ORD-001: 1500.00 THB
    // ORD-001: 1500.00 THB
}
```

---

## 📝 สรุป Part 18

| Function | `this`/`it` | Return | เมื่อไหร่ใช้ |
|----------|------------|--------|------------|
| `let` | `it` | lambda result | nullable check, transform |
| `run` | `this` | lambda result | config + compute result |
| `with` | `this` | lambda result | multiple ops (non-extension) |
| `apply` | `this` | receiver | object initialization |
| `also` | `it` | receiver | side effects, logging |

---

## ➡️ ถัดไป: Part 19 - Delegation Pattern

---
*Part 18/100+ | Kotlin & Spring Boot Complete Course*
