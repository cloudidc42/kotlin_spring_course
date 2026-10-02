# Part 90: Interview Preparation
## คำถาม Interview และวิธีตอบ

---

## 🎯 เป้าหมายของ Part นี้

- Kotlin interview questions พร้อมคำตอบ
- Spring Boot interview questions
- System Design interview tips
- Coding problems ใน Kotlin
- Behavioral interview tips

---

## 📖 1. Kotlin Interview Questions

### Q1: Kotlin Null Safety คืออะไร?

```kotlin
// คำตอบ: Kotlin แยก nullable และ non-nullable types อย่างชัดเจน
// Non-nullable: ไม่สามารถเป็น null ได้
val name: String = "Alice"
// name = null  // ❌ Compile error

// Nullable: ต้อง handle null
val nickname: String? = null

// Safe call operator ?.
val length = nickname?.length  // null ถ้า nickname เป็น null

// Elvis operator ?:
val displayName = nickname ?: "Unknown"  // "Unknown" ถ้า null

// Non-null assertion !!
val forceLength = nickname!!.length  // throw NullPointerException ถ้า null

// Smart cast
if (nickname != null) {
    println(nickname.length)  // ใน if block, nickname ถือว่าเป็น non-null
}
```

### Q2: val กับ var ต่างกันยังไง?

```kotlin
// val = immutable reference (like final in Java)
val pi = 3.14159
// pi = 3.14  // ❌ Cannot reassign

// แต่ object ที่ val ชี้ไปยังเปลี่ยนได้
val list = mutableListOf(1, 2, 3)
list.add(4)  // ✅ OK, เปลี่ยน list ได้

// var = mutable reference
var counter = 0
counter = 1  // ✅ OK

// Best practice: ใช้ val ทุกครั้งที่ทำได้
```

### Q3: Data Class คืออะไร?

```kotlin
// Data class auto-generates:
// equals(), hashCode(), toString(), copy()
data class User(
    val id: Long,
    val name: String,
    val email: String
)

val user1 = User(1, "Alice", "alice@example.com")
val user2 = User(1, "Alice", "alice@example.com")

println(user1 == user2)  // true (structural equality)
println(user1.toString())  // User(id=1, name=Alice, email=alice@example.com)

val updated = user1.copy(name = "Alicia")  // copy with changes
```

### Q4: Coroutines คืออะไร?

```kotlin
// Coroutines = lightweight threads สำหรับ async programming
// ไม่ block thread, ใช้ memory น้อยกว่า threads มาก

import kotlinx.coroutines.*

// launch: fire and forget (ไม่คืนค่า)
fun example1() = runBlocking {
    launch {
        delay(1000L)
        println("World!")
    }
    println("Hello,")
}

// async/await: parallel + return value
suspend fun fetchData(): String {
    delay(1000L)  // simulate network call
    return "Data"
}

fun example2() = runBlocking {
    val result1 = async { fetchData() }
    val result2 = async { fetchData() }

    // ทำงาน parallel! รวม ~1 second ไม่ใช่ 2 seconds
    println("${result1.await()} + ${result2.await()}")
}

// Dispatchers
// Dispatchers.Main = UI thread (Android)
// Dispatchers.IO = network, file I/O
// Dispatchers.Default = CPU intensive
withContext(Dispatchers.IO) {
    // database or network call
}
```

### Q5: Extension Functions

```kotlin
// เพิ่ม function ให้ class ที่เราไม่ได้เขียน
fun String.isPalindrome(): Boolean {
    val cleaned = this.lowercase().replace(" ", "")
    return cleaned == cleaned.reversed()
}

println("racecar".isPalindrome())  // true
println("hello".isPalindrome())   // false

// Extension ใน Spring
fun HttpServletRequest.getBearerToken(): String? =
    getHeader("Authorization")?.removePrefix("Bearer ")?.trim()
```

---

## 🌱 2. Spring Boot Interview Questions

### Q1: Spring Boot Auto-configuration คืออะไร?

```
Spring Boot สแกน classpath หา @Configuration classes ที่มี @Conditional annotations
ตัวอย่าง:
- ถ้ามี H2 dependency → configure H2 DataSource อัตโนมัติ
- ถ้ามี Spring Security → configure security filter chain

@ConditionalOnClass: เปิดใช้เมื่อ class นี้อยู่ใน classpath
@ConditionalOnMissingBean: เปิดใช้เมื่อยังไม่มี bean นี้
@ConditionalOnProperty: เปิดใช้เมื่อ property มีค่าที่กำหนด
```

### Q2: Spring Bean Scope

```kotlin
// Singleton (default): 1 instance ต่อ Spring context
@Service  // singleton โดย default
class UserService

// Prototype: สร้าง instance ใหม่ทุกครั้งที่ inject
@Component
@Scope("prototype")
class OrderProcessor

// Request: 1 instance ต่อ HTTP request (Web apps)
@Component
@Scope("request")
class RequestContext

// Session: 1 instance ต่อ HTTP session
@Component
@Scope("session")
class ShoppingCart
```

### Q3: @Transactional ทำงานอย่างไร?

```kotlin
@Service
class OrderService(private val orderRepository: OrderRepository) {

    @Transactional  // Spring AOP สร้าง proxy รอบ method
    fun createOrder(order: Order): Order {
        val saved = orderRepository.save(order)
        // ถ้า exception ใดๆ ใน method นี้ → rollback ทุกอย่าง
        return saved
    }

    // Self-invocation ปัญหา!
    fun createOrderWithDiscount(order: Order): Order {
        // ❌ เรียก @Transactional method จากภายใน class เดิม
        // ไม่ผ่าน proxy → @Transactional ไม่ทำงาน!
        return createOrder(order)
    }
}

// Propagation
@Transactional(propagation = Propagation.REQUIRED)     // default: join existing TX
@Transactional(propagation = Propagation.REQUIRES_NEW) // always start new TX
@Transactional(propagation = Propagation.NEVER)        // throw if TX exists
```

### Q4: Spring Security Filter Chain

```
Request → [Security Filter Chain]
           ├── CorsFilter
           ├── CsrfFilter
           ├── UsernamePasswordAuthenticationFilter
           ├── JwtAuthenticationFilter (custom)
           ├── ExceptionTranslationFilter
           └── FilterSecurityInterceptor
          → Controller
```

---

## 💻 3. Coding Problems

### Problem 1: Two Sum (Kotlin)

```kotlin
// หา indices ของ 2 numbers ที่รวมกันได้ target
fun twoSum(nums: IntArray, target: Int): IntArray {
    val map = HashMap<Int, Int>()  // value → index

    for (i in nums.indices) {
        val complement = target - nums[i]
        if (map.containsKey(complement)) {
            return intArrayOf(map[complement]!!, i)
        }
        map[nums[i]] = i
    }

    throw IllegalArgumentException("No solution found")
}

// Test
println(twoSum(intArrayOf(2, 7, 11, 15), 9).toList())  // [0, 1]
println(twoSum(intArrayOf(3, 2, 4), 6).toList())        // [1, 2]
// Time: O(n), Space: O(n)
```

### Problem 2: Valid Parentheses

```kotlin
fun isValid(s: String): Boolean {
    val stack = ArrayDeque<Char>()
    val matching = mapOf(')' to '(', ']' to '[', '}' to '{')

    for (char in s) {
        when (char) {
            '(', '[', '{' -> stack.addLast(char)
            ')', ']', '}' -> {
                if (stack.isEmpty() || stack.last() != matching[char]) {
                    return false
                }
                stack.removeLast()
            }
        }
    }

    return stack.isEmpty()
}

// Test
println(isValid("()[]{}"))  // true
println(isValid("([)]"))    // false
println(isValid("{[]}"))    // true
```

### Problem 3: Fibonacci ด้วย Memoization

```kotlin
// Recursive with memoization
fun fibonacci(n: Int, memo: MutableMap<Int, Long> = mutableMapOf()): Long {
    if (n <= 1) return n.toLong()
    return memo.getOrPut(n) {
        fibonacci(n - 1, memo) + fibonacci(n - 2, memo)
    }
}

// Iterative (better for large n)
fun fibIterative(n: Int): Long {
    if (n <= 1) return n.toLong()
    var prev = 0L
    var curr = 1L
    repeat(n - 1) {
        val next = prev + curr
        prev = curr
        curr = next
    }
    return curr
}

println(fibonacci(50))    // 12586269025
println(fibIterative(50)) // 12586269025
```

### Problem 4: Flatten Nested List

```kotlin
fun flatten(list: List<Any>): List<Int> = buildList {
    for (item in list) {
        when (item) {
            is Int -> add(item)
            is List<*> -> @Suppress("UNCHECKED_CAST")
                addAll(flatten(item as List<Any>))
        }
    }
}

// Test
val nested = listOf(1, listOf(2, 3), listOf(4, listOf(5, 6)), 7)
println(flatten(nested))  // [1, 2, 3, 4, 5, 6, 7]
```

---

## 🏗️ 4. System Design Interview Tips

### Framework: STAR+C

```
S - Scope: Clarify requirements, scale, constraints
T - Think: High-level design first
A - Architecture: Choose components and explain why
R - Refined: Deep dive into critical components
C - Consider: Trade-offs, bottlenecks, improvements
```

### คำถามที่ต้องถาม

```
1. Scale:
   - How many users?
   - How many requests/second?
   - Data size?

2. Features:
   - MVP vs Full features?
   - Which features are critical?

3. Non-functional:
   - Latency requirements?
   - Availability (99.9%? 99.99%?)
   - Consistency vs Availability?

4. Constraints:
   - Budget?
   - Tech stack preferences?
   - Existing infrastructure?
```

### ตัวอย่าง: Design URL Shortener

```
1. Clarify: 100M URLs/day, read:write = 100:1, links never expire
2. Core service: shorten URL, redirect URL
3. Storage: ~100M * 365 = 36.5B records/year
4. Algorithm: Base62 encoding of sequential ID or MD5 hash

Key Components:
- Load Balancer
- API Servers (stateless, horizontal scale)
- Cache (Redis - most accessed URLs)
- Database (PostgreSQL for write, MySQL Cluster for read)
- CDN for redirects

Calculation:
- Write: 100M/day = ~1,160/sec
- Read: 116,000/sec
- Storage: 500 bytes/URL × 36.5B = ~18TB/year
```

---

## 🎭 5. Behavioral Interview Tips

### STAR Method

```
S - Situation: บริบทและ context
T - Task: หน้าที่ความรับผิดชอบ
A - Action: สิ่งที่คุณทำ (ใช้ "I" ไม่ใช่ "we")
R - Result: ผลลัพธ์ที่วัดได้

ตัวอย่าง:
Q: "Tell me about a time you handled a production incident"

S: "ตอน Black Friday ปีที่แล้ว API เราลงเพราะ database connection pool หมด"
T: "ผมเป็น on-call engineer ต้องแก้ไขภายใน SLA 30 นาที"
A: "ผม: 1) เปิด monitoring ดู metrics 2) พบ connection pool exhausted 
     3) เพิ่ม pool size ชั่วคราว 4) trace root cause ว่ามี slow queries"
R: "ระบบกลับมา online ใน 12 นาที, ไม่มี data loss
     สัปดาห์ถัดไป optimize queries และเพิ่ม connection pooling properly"
```

### คำถาม Behavioral ที่พบบ่อย

```
1. "Tell me about your biggest technical challenge"
   → เตรียม story เกี่ยวกับ complex problem ที่แก้ได้

2. "How do you handle disagreements with teammates?"
   → เน้น communication, data-driven decisions

3. "Tell me about a time you improved a process"
   → เตรียม quantifiable improvements

4. "How do you prioritize when everything is urgent?"
   → Framework: Impact vs Effort matrix

5. "What's your biggest failure and what did you learn?"
   → ตอบตรงๆ + เน้น learning
```

---

## 📋 สรุป

| หัวข้อ | สิ่งสำคัญ |
|--------|---------|
| Kotlin | Null safety, Coroutines, Extension functions |
| Spring Boot | Auto-config, Bean scope, @Transactional |
| Coding | Practice LeetCode medium, clean code |
| System Design | Clarify → HLD → Deep dive → Trade-offs |
| Behavioral | STAR method, genuine stories |

### แนะนำการเตรียมตัว

```
2 สัปดาห์ก่อน interview:
  - Review Kotlin fundamentals
  - Practice 3-5 coding problems/day
  - อ่าน system design resources

1 สัปดาห์ก่อน:
  - Mock interview กับเพื่อน
  - เตรียม behavioral stories

วันก่อน:
  - ทบทวน core concepts
  - นอนหลับให้พอ
  - เตรียม questions ถามกลับ
```

---

*Part 90/100+ | Kotlin & Spring Boot Complete Course*
