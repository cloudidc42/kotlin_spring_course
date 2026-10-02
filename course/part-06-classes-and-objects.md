# Part 06: Classes และ Objects
## Classes, Objects & OOP in Kotlin

---

## 🎯 เป้าหมายของ Part นี้

- สร้าง class และ constructor
- ใช้ properties กับ custom getter/setter
- เข้าใจ companion objects
- ใช้ object declarations (Singleton)
- ควบคุม visibility ด้วย modifiers
- ใช้ init blocks

---

## 🏗️ 1. Class พื้นฐาน

```kotlin
// Class ง่ายๆ
class Person {
    var name = "Unknown"
    var age = 0
    
    fun greet() {
        println("Hello, I'm $name, age $age")
    }
}

// สร้าง instance
val person = Person()  // ไม่ต้องใช้ new!
person.name = "Alice"
person.age = 25
person.greet()  // Hello, I'm Alice, age 25
```

### Primary Constructor

```kotlin
// ประกาศ properties ใน constructor โดยตรง
class Person(val name: String, var age: Int) {
    fun greet() = "Hello, I'm $name, age $age"
    
    fun birthday() {
        age++
        println("Happy Birthday $name! Now $age years old.")
    }
}

val alice = Person("Alice", 25)
println(alice.greet())   // Hello, I'm Alice, age 25
alice.birthday()         // Happy Birthday Alice! Now 26 years old.
println(alice.age)       // 26

// val = read-only property
// var = read-write property
```

### Constructor กับ init block

```kotlin
class BankAccount(
    val owner: String,
    initialBalance: Double = 0.0  // constructor parameter (ไม่ใช่ property)
) {
    var balance: Double = initialBalance
        private set  // ตั้งค่าได้เฉพาะภายใน class
    
    val accountId: String
    
    init {
        // ทำงานตอนสร้าง object
        require(initialBalance >= 0) { "Initial balance cannot be negative" }
        accountId = "ACC-${System.currentTimeMillis()}"
        println("Account created for $owner with balance $initialBalance")
    }
    
    fun deposit(amount: Double) {
        require(amount > 0) { "Deposit amount must be positive" }
        balance += amount
        println("Deposited $amount. Balance: $balance")
    }
    
    fun withdraw(amount: Double) {
        require(amount > 0) { "Withdrawal amount must be positive" }
        require(amount <= balance) { "Insufficient funds" }
        balance -= amount
        println("Withdrew $amount. Balance: $balance")
    }
}

val account = BankAccount("Alice", 1000.0)
account.deposit(500.0)
account.withdraw(200.0)
// account.balance = 5000.0  // ❌ Error! private set
println(account.balance)  // 1300.0
```

### Secondary Constructors

```kotlin
class Student(val name: String, val age: Int, val grade: String) {
    var gpa: Double = 0.0
    var courses = mutableListOf<String>()
    
    // Secondary constructor ต้องเรียก primary ด้วย this()
    constructor(name: String, age: Int) : this(name, age, "ปี 1")
    
    constructor(name: String) : this(name, 18, "ปี 1")
    
    // Secondary constructor กับ init
    constructor(name: String, age: Int, grade: String, gpa: Double) : this(name, age, grade) {
        this.gpa = gpa
    }
}

val s1 = Student("Alice", 20, "ปี 3")
val s2 = Student("Bob", 19)           // เรียก secondary
val s3 = Student("Charlie")            // เรียก secondary
val s4 = Student("David", 21, "ปี 4", 3.8)
```

---

## 🏠 2. Properties

### Custom Getter และ Setter

```kotlin
class Rectangle(var width: Double, var height: Double) {
    // Computed property (ไม่มี backing field)
    val area: Double
        get() = width * height
    
    val perimeter: Double
        get() = 2 * (width + height)
    
    val isSquare: Boolean
        get() = width == height
    
    // Property กับ backing field
    var name: String = "Rectangle"
        get() = field.uppercase()   // field คือค่าจริง
        set(value) {
            field = value.trim()    // validate ก่อน set
        }
}

val rect = Rectangle(4.0, 6.0)
println(rect.area)       // 24.0
println(rect.perimeter)  // 20.0
println(rect.isSquare)   // false

rect.width = 5.0
println(rect.isSquare)   // false
rect.height = 5.0
println(rect.isSquare)   // true

rect.name = "  my rect  "
println(rect.name)  // MY RECT
```

### Late-initialized Properties

```kotlin
class UserService {
    // lateinit: กำหนดค่าภายหลัง (ไม่ต้องเป็น null)
    lateinit var database: String
    lateinit var cache: MutableMap<String, String>
    
    fun initialize() {
        database = "postgresql://localhost/mydb"
        cache = mutableMapOf()
    }
    
    fun getUser(id: String): String {
        // ตรวจสอบก่อน access
        if (!::database.isInitialized) {
            throw IllegalStateException("Service not initialized!")
        }
        return cache.getOrDefault(id, "User not found")
    }
}

val service = UserService()
// service.database  // ❌ UninitializedPropertyAccessException!
service.initialize()
println(service.database)  // postgresql://localhost/mydb
```

### Delegated Properties

```kotlin
import kotlin.properties.Delegates

class User {
    // Observable property - รู้เมื่อเปลี่ยนค่า
    var name: String by Delegates.observable("Unknown") { prop, old, new ->
        println("${prop.name} changed: '$old' → '$new'")
    }
    
    // Vetoable property - ป้องกันการเปลี่ยนค่า
    var age: Int by Delegates.vetoable(0) { _, old, new ->
        new >= 0  // ยอมรับเฉพาะค่า >= 0
    }
    
    // notNull property
    var email: String by Delegates.notNull()
}

val user = User()
user.name = "Alice"
// name changed: 'Unknown' → 'Alice'
user.name = "Bob"
// name changed: 'Alice' → 'Bob'

user.age = 25
println(user.age)   // 25
user.age = -1       // ถูก veto
println(user.age)   // 25 (ไม่เปลี่ยน)

user.email = "alice@example.com"
println(user.email) // alice@example.com
```

---

## 👥 3. Companion Objects

```kotlin
class Circle(val radius: Double) {
    // companion object - เหมือน static ใน Java
    companion object {
        const val PI = 3.14159265358979
        
        fun create(radius: Double): Circle {
            require(radius > 0) { "Radius must be positive" }
            return Circle(radius)
        }
        
        fun unitCircle() = Circle(1.0)
    }
    
    val area: Double get() = PI * radius * radius
    val circumference: Double get() = 2 * PI * radius
}

// เรียกผ่าน class name (เหมือน static)
println(Circle.PI)              // 3.14159265358979
val c1 = Circle.create(5.0)    // Factory method
val c2 = Circle.unitCircle()   // Factory method

println(c1.area)
println(c2.circumference)
```

### Named Companion Object

```kotlin
class HttpClient private constructor(
    val baseUrl: String,
    val timeout: Int
) {
    companion object Factory {
        fun create(baseUrl: String): HttpClient {
            return HttpClient(baseUrl, 30000)
        }
        
        fun createWithTimeout(baseUrl: String, timeout: Int): HttpClient {
            return HttpClient(baseUrl, timeout)
        }
    }
}

val client1 = HttpClient.create("https://api.example.com")
val client2 = HttpClient.Factory.create("https://api.test.com")
val client3 = HttpClient.createWithTimeout("https://api.prod.com", 60000)
```

### Companion + Interface

```kotlin
interface Serializable<T> {
    fun serialize(obj: T): String
    fun deserialize(data: String): T
}

class User(val name: String, val age: Int) {
    companion object : Serializable<User> {
        override fun serialize(obj: User) = "${obj.name}:${obj.age}"
        
        override fun deserialize(data: String): User {
            val parts = data.split(":")
            return User(parts[0], parts[1].toInt())
        }
    }
    
    override fun toString() = "User(name=$name, age=$age)"
}

val alice = User("Alice", 25)
val serialized = User.serialize(alice)      // "Alice:25"
val restored = User.deserialize(serialized) // User(name=Alice, age=25)
println(serialized)
println(restored)
```

---

## 🔒 4. Visibility Modifiers

```kotlin
class BankingSystem {
    public val bankName = "My Bank"     // ทุกคนเข้าถึงได้ (default)
    internal val internalCode = "XYZ"  // ใน module เดียวกัน
    protected val protectedData = 42   // class นี้ + subclasses
    private val secretKey = "abc123"   // class นี้เท่านั้น
    
    fun publicOperation() {
        println("Public: $bankName")
        println("Private: $secretKey")  // ✅ access จาก class เดียวกัน
    }
    
    private fun privateHelper() {
        // ใช้ได้เฉพาะภายใน class นี้
    }
}

// Top-level declarations
public fun publicFunction() = println("Public")
internal fun internalFunction() = println("Internal")
private fun privateFunction() = println("Private")  // เฉพาะไฟล์นี้

// Private constructor
class Singleton private constructor() {
    companion object {
        val instance = Singleton()
    }
}
```

---

## 🎭 5. Object Declaration (Singleton)

```kotlin
// Singleton pattern ด้วย object
object AppConfig {
    val version = "1.0.0"
    val maxConnections = 100
    var debugMode = false
    
    fun info() = "App v$version (debug=$debugMode)"
    
    fun configure(debug: Boolean) {
        debugMode = debug
    }
}

println(AppConfig.version)
AppConfig.configure(true)
println(AppConfig.info())

// Object ที่ implement interface
object Logger : Runnable {
    val logs = mutableListOf<String>()
    
    fun log(message: String) {
        logs.add("[${System.currentTimeMillis()}] $message")
        println("LOG: $message")
    }
    
    override fun run() {
        logs.forEach { println(it) }
    }
}

Logger.log("Application started")
Logger.log("User logged in")
Logger.run()
```

### Object Expression (Anonymous Object)

```kotlin
// Anonymous object - เหมือน anonymous class ใน Java
val clickHandler = object {
    fun onClick() = println("Clicked!")
    fun onLongClick() = println("Long clicked!")
}

clickHandler.onClick()
clickHandler.onLongClick()

// Anonymous object ที่ implement interface
interface EventListener {
    fun onEvent(event: String)
}

fun setListener(listener: EventListener) {
    listener.onEvent("test_event")
}

setListener(object : EventListener {
    override fun onEvent(event: String) {
        println("Event received: $event")
    }
})
```

---

## 🏗️ 6. Data Classes

```kotlin
// data class - สร้าง equals, hashCode, toString, copy ให้อัตโนมัติ
data class Product(
    val id: Int,
    val name: String,
    val price: Double,
    val category: String = "General"
)

val p1 = Product(1, "Kotlin Book", 599.0)
val p2 = Product(1, "Kotlin Book", 599.0)
val p3 = Product(2, "Spring Course", 1200.0, "Education")

// toString อัตโนมัติ
println(p1)  // Product(id=1, name=Kotlin Book, price=599.0, category=General)

// equals อัตโนมัติ
println(p1 == p2)  // true (content equal)
println(p1 === p2) // false (different objects)

// copy กับการเปลี่ยนบาง field
val discounted = p1.copy(price = 499.0)
println(discounted)  // Product(id=1, name=Kotlin Book, price=499.0, category=General)

// Destructuring
val (id, name, price) = p1
println("$id: $name = $price")

// ใน collections
val products = listOf(p1, p3)
val sorted = products.sortedBy { it.price }
println(sorted)
```

---

## 🔑 7. ตัวอย่างโปรแกรมจริง

### Student Management System

```kotlin
data class Course(val code: String, val name: String, val credits: Int)

class Student(
    val id: String,
    val name: String,
    val faculty: String
) {
    private val enrolledCourses = mutableMapOf<Course, Double?>()  // Course → grade
    
    val courseCount: Int get() = enrolledCourses.size
    
    fun enroll(course: Course) {
        if (course !in enrolledCourses) {
            enrolledCourses[course] = null
            println("$name enrolled in ${course.name}")
        } else {
            println("Already enrolled in ${course.name}")
        }
    }
    
    fun recordGrade(course: Course, grade: Double) {
        require(course in enrolledCourses) { "Not enrolled in ${course.name}" }
        require(grade in 0.0..4.0) { "Grade must be 0.0-4.0" }
        enrolledCourses[course] = grade
    }
    
    fun calculateGPA(): Double {
        val graded = enrolledCourses.filter { it.value != null }
        if (graded.isEmpty()) return 0.0
        
        val totalPoints = graded.entries.sumOf { (course, grade) -> 
            course.credits * (grade ?: 0.0) 
        }
        val totalCredits = graded.keys.sumOf { it.credits }
        
        return totalPoints / totalCredits
    }
    
    fun transcript(): String {
        val sb = StringBuilder()
        sb.appendLine("=".repeat(50))
        sb.appendLine("Student: $name ($id)")
        sb.appendLine("Faculty: $faculty")
        sb.appendLine("-".repeat(50))
        sb.appendLine("%-10s %-20s %7s %6s".format("Code", "Course", "Credits", "Grade"))
        sb.appendLine("-".repeat(50))
        
        for ((course, grade) in enrolledCourses) {
            val gradeStr = grade?.let { "%.1f".format(it) } ?: "N/A"
            sb.appendLine("%-10s %-20s %7d %6s".format(
                course.code, course.name, course.credits, gradeStr
            ))
        }
        
        sb.appendLine("-".repeat(50))
        sb.appendLine("GPA: ${"%.2f".format(calculateGPA())}")
        sb.appendLine("=".repeat(50))
        return sb.toString()
    }
}

fun main() {
    val kotlin101 = Course("CS101", "Kotlin Basics", 3)
    val spring201 = Course("CS201", "Spring Boot", 3)
    val db301 = Course("CS301", "Database", 3)
    
    val alice = Student("STU001", "Alice Smith", "Computer Science")
    alice.enroll(kotlin101)
    alice.enroll(spring201)
    alice.enroll(db301)
    
    alice.recordGrade(kotlin101, 4.0)
    alice.recordGrade(spring201, 3.5)
    alice.recordGrade(db301, 3.0)
    
    println(alice.transcript())
}
```

**ผลลัพธ์:**
```
==================================================
Student: Alice Smith (STU001)
Faculty: Computer Science
--------------------------------------------------
Code       Course               Credits  Grade
--------------------------------------------------
CS101      Kotlin Basics              3    4.0
CS201      Spring Boot                3    3.5
CS301      Database                   3    3.0
--------------------------------------------------
GPA: 3.50
==================================================
```

---

## 🏋️ 8. แบบฝึกหัด

### ข้อ 1: Library System
```kotlin
data class Book(
    val isbn: String,
    val title: String,
    val author: String,
    val year: Int
)

class Library(val name: String) {
    private val books = mutableListOf<Book>()
    private val borrowedBooks = mutableMapOf<String, String>()  // isbn → member
    
    fun addBook(book: Book) = books.add(book)
    
    fun borrow(isbn: String, member: String): Boolean {
        val book = books.find { it.isbn == isbn } ?: return false
        if (isbn in borrowedBooks) return false
        borrowedBooks[isbn] = member
        return true
    }
    
    fun returnBook(isbn: String): Boolean {
        return borrowedBooks.remove(isbn) != null
    }
    
    fun searchByTitle(query: String): List<Book> {
        return books.filter { it.title.contains(query, ignoreCase = true) }
    }
    
    fun availableBooks(): List<Book> {
        return books.filter { it.isbn !in borrowedBooks }
    }
    
    override fun toString(): String = "$name: ${books.size} books"
}

fun main() {
    val library = Library("Kotlin Library")
    
    library.addBook(Book("978-1", "Kotlin in Action", "Dmitry Jemerov", 2017))
    library.addBook(Book("978-2", "Spring Boot in Action", "Craig Walls", 2022))
    library.addBook(Book("978-3", "Clean Code", "Robert Martin", 2008))
    
    println(library.borrow("978-1", "Alice"))  // true
    println(library.borrow("978-1", "Bob"))    // false (already borrowed)
    
    println("Available: ${library.availableBooks().map { it.title }}")
    
    library.returnBook("978-1")
    println("Available after return: ${library.availableBooks().map { it.title }}")
}
```

---

## 📝 สรุป Part 06

| แนวคิด | รายละเอียด |
|--------|-----------|
| `class` | Primary + secondary constructors |
| `init` | Initialization block |
| `val`/`var` | Immutable/mutable properties |
| `get()`/`set()` | Custom accessors |
| `companion object` | Static-like members |
| `object` | Singleton declaration |
| `data class` | Auto equals/toString/copy |
| `private`/`public` | Visibility control |
| `lateinit` | Late initialization |

---

## ➡️ ถัดไป: Part 07 - Inheritance และ Interfaces

---
*Part 06/100+ | Kotlin & Spring Boot Complete Course*
