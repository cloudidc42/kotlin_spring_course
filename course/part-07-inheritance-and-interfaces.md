# Part 07: Inheritance และ Interfaces
## การสืบทอดคลาสและ Interfaces ใน Kotlin

---

## 🎯 เป้าหมายของ Part นี้

- เข้าใจ `open class` และการ inheritance
- ใช้ `override` เพื่อ override methods และ properties
- สร้างและใช้งาน `abstract class`
- เข้าใจความแตกต่างระหว่าง interface กับ abstract class
- ใช้ multiple interfaces
- เข้าใจ interface delegation
- ใช้ `sealed class` สำหรับ restricted hierarchies
- สร้าง Shape hierarchy และ Animal hierarchy จริง

---

## 🔓 1. open class และ Inheritance

ใน Kotlin โดย default **ทุก class เป็น `final`** (ไม่สามารถ inherit ได้) ต่างจาก Java ที่ class สามารถ inherit ได้โดย default เราต้องใช้ keyword `open` เพื่อบอกว่า class นี้ "เปิดให้ inherit ได้"

### ทำไม Kotlin ถึง default เป็น final?

เพราะ inheritance ที่ไม่ได้ออกแบบมาดีๆ เป็นสาเหตุของ bugs มากมาย Kotlin บังคับให้เราคิดก่อนว่า "class นี้ควรถูก inherit ไหม?" ถ้าใช่ ค่อยเพิ่ม `open`

```kotlin
// ❌ Class ปกติ - ไม่สามารถ inherit ได้
class Animal {
    fun breathe() = println("Breathing...")
}

// class Dog : Animal()  // ERROR: This type is final, cannot be inherited from

// ✅ open class - สามารถ inherit ได้
open class Animal {
    val name: String = "Animal"
    
    fun breathe() = println("$name is breathing...")
    
    // open function = สามารถ override ได้
    open fun makeSound() = println("...")
    
    // ไม่ใส่ open = final method ใน open class
    fun eat() = println("$name is eating...")
}

class Dog : Animal() {
    override fun makeSound() = println("Woof!")
}

class Cat : Animal() {
    override fun makeSound() = println("Meow!")
}

fun main() {
    val dog = Dog()
    dog.breathe()      // Animal is breathing...
    dog.makeSound()    // Woof!
    dog.eat()          // Animal is eating...
    
    val cat = Cat()
    cat.makeSound()    // Meow!
}
```

### Constructor ใน Inheritance

```kotlin
open class Vehicle(
    val brand: String,
    val model: String,
    val year: Int
) {
    open val description: String
        get() = "$year $brand $model"
    
    open fun start() = println("$description starting...")
    open fun stop() = println("$description stopping...")
}

// ส่งต่อ arguments ไปยัง parent constructor
class Car(
    brand: String,
    model: String,
    year: Int,
    val numDoors: Int = 4
) : Vehicle(brand, model, year) {
    
    override val description: String
        get() = "${super.description} ($numDoors doors)"
    
    override fun start() {
        println("Turning key...")
        super.start()  // เรียก parent method
    }
}

class ElectricCar(
    brand: String,
    model: String,
    year: Int,
    val batteryCapacity: Int  // kWh
) : Vehicle(brand, model, year) {
    
    var batteryLevel: Int = 100  // percent
    
    override val description: String
        get() = "${super.description} (Electric, ${batteryCapacity}kWh)"
    
    override fun start() = println("$description silently starting... ⚡")
    
    fun charge() {
        batteryLevel = 100
        println("$description fully charged!")
    }
}

fun main() {
    val car = Car("Toyota", "Camry", 2023)
    println(car.description)    // 2023 Toyota Camry (4 doors)
    car.start()
    // Turning key...
    // 2023 Toyota Camry (4 doors) starting...
    
    val tesla = ElectricCar("Tesla", "Model 3", 2024, 75)
    println(tesla.description)  // 2024 Tesla Model 3 (Electric, 75kWh)
    tesla.start()               // 2024 Tesla Model 3 (Electric, 75kWh) silently starting... ⚡
}
```

---

## 🔄 2. Method Overriding ด้วย override

### กฎของการ Override

1. Parent method ต้องเป็น `open` (หรืออยู่ใน abstract class/interface)
2. ใช้ `override` keyword ใน child class
3. จะ override property ก็ได้เช่นกัน
4. ใช้ `super` เพื่อเรียก parent implementation
5. ถ้าต้องการป้องกัน override ต่อไปอีก ใช้ `final override`

```kotlin
open class Shape {
    open val name: String = "Shape"
    open val area: Double get() = 0.0
    open val perimeter: Double get() = 0.0
    
    open fun describe() {
        println("$name: area = ${"%.2f".format(area)}, perimeter = ${"%.2f".format(perimeter)}")
    }
}

open class Rectangle(val width: Double, val height: Double) : Shape() {
    override val name: String = "Rectangle"
    override val area: Double get() = width * height
    override val perimeter: Double get() = 2 * (width + height)
}

class Square(side: Double) : Rectangle(side, side) {
    override val name: String = "Square"
    // area และ perimeter ใช้จาก Rectangle ได้เลย เพราะ logic เหมือนกัน
    
    // final override - ป้องกันไม่ให้ class ลูกๆ override ต่อ
    final override fun describe() {
        println("$name (${width}x${height}): area = ${"%.2f".format(area)}")
    }
}

class Circle(val radius: Double) : Shape() {
    override val name: String = "Circle"
    override val area: Double get() = Math.PI * radius * radius
    override val perimeter: Double get() = 2 * Math.PI * radius
}

fun main() {
    val shapes: List<Shape> = listOf(
        Rectangle(5.0, 3.0),
        Square(4.0),
        Circle(7.0)
    )
    
    for (shape in shapes) {
        shape.describe()
    }
    // Rectangle: area = 15.00, perimeter = 16.00
    // Square (4.0x4.0): area = 16.00
    // Circle: area = 153.94, perimeter = 43.98
}
```

### Overriding Properties

```kotlin
open class Base {
    open val x: Int = 10        // val property
    open var count: Int = 0     // var property
}

class Derived : Base() {
    // val สามารถ override เป็น var ได้ (เพิ่ม setter)
    override var x: Int = 20
    
    // var ไม่สามารถ override เป็น val ได้ (ลบ setter ไม่ได้)
    override var count: Int = 100
    
    // override ด้วย custom getter
    val doubled: Int
        get() = x * 2
}

fun main() {
    val d = Derived()
    println(d.x)        // 20
    d.x = 30
    println(d.x)        // 30
    println(d.doubled)  // 60
}
```

---

## 🎭 3. Abstract Classes

`abstract class` คือ class ที่ **ไม่สามารถสร้าง instance ได้โดยตรง** และมี abstract members ที่ subclass ต้องไป implement

### เมื่อไหรควรใช้ Abstract Class?

- เมื่อต้องการ define template/blueprint ที่มีทั้ง implemented code และ abstract methods
- เมื่อ subclasses มี state (properties) ร่วมกัน
- เมื่อต้องการ constructor

```kotlin
abstract class Animal(val name: String, val age: Int) {
    // Abstract property - subclass ต้อง implement
    abstract val sound: String
    abstract val legs: Int
    
    // Abstract method - subclass ต้อง implement
    abstract fun move(): String
    
    // Concrete method - มี implementation แล้ว
    fun breathe() = println("$name is breathing")
    
    fun describe() {
        println("""
            Animal: $name
            Age: $age years
            Sound: $sound
            Legs: $legs
            Movement: ${move()}
        """.trimIndent())
    }
    
    // Template method pattern - กำหนด algorithm structure
    fun dailyRoutine() {
        println("$name's daily routine:")
        println("  1. ${wakeUp()}")
        println("  2. ${eat()}")
        println("  3. ${move()}")
        println("  4. Sleep")
    }
    
    protected open fun wakeUp() = "Wakes up"
    protected abstract fun eat(): String
}

class Dog(name: String, age: Int, val breed: String) : Animal(name, age) {
    override val sound: String = "Woof"
    override val legs: Int = 4
    
    override fun move() = "Runs on 4 legs"
    override fun eat() = "Eats kibble from bowl"
    override fun wakeUp() = "Wakes up and wags tail"
    
    fun fetch() = println("$name fetches the ball!")
}

class Bird(name: String, age: Int, val canFly: Boolean) : Animal(name, age) {
    override val sound: String = "Tweet"
    override val legs: Int = 2
    
    override fun move() = if (canFly) "Flies through the air" else "Walks on 2 legs"
    override fun eat() = "Pecks at seeds"
}

class Fish(name: String, age: Int) : Animal(name, age) {
    override val sound: String = "(silent)"
    override val legs: Int = 0
    
    override fun move() = "Swims through water"
    override fun eat() = "Eats fish food"
}

fun main() {
    // val a = Animal("Test", 1)  // ERROR: Cannot create instance of abstract class
    
    val dog = Dog("Buddy", 3, "Labrador")
    dog.describe()
    println()
    dog.dailyRoutine()
    println()
    dog.fetch()
    
    println("---")
    
    val bird = Bird("Tweety", 1, canFly = true)
    bird.describe()
}
```

### Abstract Class กับ Template Method Pattern

```kotlin
abstract class DataProcessor<T> {
    // Template method
    fun process(rawData: List<String>): List<T> {
        val validated = validate(rawData)
        val parsed = parse(validated)
        val filtered = filter(parsed)
        return transform(filtered)
    }
    
    protected open fun validate(data: List<String>): List<String> {
        return data.filter { it.isNotBlank() }
    }
    
    protected abstract fun parse(data: List<String>): List<T>
    
    protected open fun filter(data: List<T>): List<T> = data
    
    protected open fun transform(data: List<T>): List<T> = data
}

data class Product(val name: String, val price: Double, val category: String)

class ProductProcessor : DataProcessor<Product>() {
    override fun parse(data: List<String>): List<Product> {
        return data.mapNotNull { line ->
            val parts = line.split(",")
            if (parts.size == 3) {
                try {
                    Product(parts[0].trim(), parts[1].trim().toDouble(), parts[2].trim())
                } catch (e: NumberFormatException) {
                    null
                }
            } else null
        }
    }
    
    override fun filter(data: List<Product>): List<Product> {
        return data.filter { it.price > 0 }
    }
    
    override fun transform(data: List<Product>): List<Product> {
        return data.sortedBy { it.price }
    }
}

fun main() {
    val rawData = listOf(
        "Apple, 1.50, Fruit",
        "  ",  // blank - will be filtered
        "Banana, 0.75, Fruit",
        "Laptop, 999.99, Electronics",
        "Bad data",  // invalid - will be null
        "Phone, 599.00, Electronics"
    )
    
    val processor = ProductProcessor()
    val products = processor.process(rawData)
    
    products.forEach { println("${it.name}: $${it.price} [${it.category}]") }
    // Banana: $0.75 [Fruit]
    // Apple: $1.5 [Fruit]
    // Phone: $599.0 [Electronics]
    // Laptop: $999.99 [Electronics]
}
```

---

## 🔌 4. Interfaces

Interface คือ contract ที่กำหนดว่า class ต้อง implement อะไรบ้าง ต่างจาก abstract class ตรงที่ interface **ไม่มี state (backing field)** และ class สามารถ implement **หลาย interfaces** พร้อมกันได้

### Interface พื้นฐาน

```kotlin
interface Printable {
    fun print()
    fun printFormatted(format: String) = print()  // default implementation
}

interface Saveable {
    val filename: String  // abstract property (ไม่มี backing field)
    
    fun save(): Boolean
    fun load(): Boolean
}

interface Describable {
    fun describe(): String
    
    // Default implementation
    fun printDescription() {
        println(describe())
    }
}

// Implement หลาย interfaces
class Document(
    val title: String,
    val content: String
) : Printable, Saveable, Describable {
    
    override val filename: String = "${title.lowercase().replace(" ", "_")}.txt"
    
    override fun print() {
        println("=== $title ===")
        println(content)
    }
    
    override fun save(): Boolean {
        println("Saving to $filename...")
        return true  // simulate success
    }
    
    override fun load(): Boolean {
        println("Loading from $filename...")
        return true
    }
    
    override fun describe(): String = "Document: '$title' (${content.length} chars)"
}

fun main() {
    val doc = Document("My Report", "This is the content of the report.")
    
    doc.print()
    doc.printFormatted("PDF")
    doc.save()
    doc.printDescription()
}
```

### Interface กับ Properties

```kotlin
interface Shape {
    val name: String
    val area: Double
    val perimeter: Double
    
    // Interface สามารถมี computed property ที่ไม่ต้องการ backing field
    val isLargerThanUnit: Boolean
        get() = area > 1.0
    
    fun describe() = "$name: area=${"%.2f".format(area)}, perimeter=${"%.2f".format(perimeter)}"
}

interface Colorable {
    var color: String
    
    fun paint(newColor: String) {
        color = newColor
        println("Painted $newColor")
    }
}

class ColoredCircle(val radius: Double, override var color: String = "white") : Shape, Colorable {
    override val name: String = "Circle"
    override val area: Double get() = Math.PI * radius * radius
    override val perimeter: Double get() = 2 * Math.PI * radius
}

fun main() {
    val circle = ColoredCircle(5.0, "blue")
    println(circle.describe())
    println("Large? ${circle.isLargerThanUnit}")  // true
    circle.paint("red")
    println("Color: ${circle.color}")
}
```

---

## 🆚 5. Interface vs Abstract Class

| ลักษณะ | Interface | Abstract Class |
|--------|-----------|----------------|
| Multiple inheritance | ✅ ได้หลายอัน | ❌ ได้แค่อันเดียว |
| Constructor | ❌ ไม่มี | ✅ มีได้ |
| State (backing fields) | ❌ ไม่มี | ✅ มีได้ |
| Default implementation | ✅ ได้ (ตั้งแต่ Kotlin 1.0) | ✅ ได้ |
| Abstract members | ✅ ได้ | ✅ ได้ |
| Access modifiers | public เท่านั้น | ทุก modifier ได้ |
| เหมาะกับ | Capabilities/Behaviors | "Is-a" relationship |

### ตัวอย่างเปรียบเทียบ

```kotlin
// ใช้ Abstract Class เมื่อต้องการ "is-a" และมี shared state
abstract class Employee(
    val id: String,
    val name: String,
    val baseSalary: Double
) {
    abstract fun calculateBonus(): Double
    
    fun totalCompensation() = baseSalary + calculateBonus()
    
    fun introduce() = "I'm $name, employee #$id"
}

// ใช้ Interface เมื่อต้องการ "can-do" capabilities
interface Reportable {
    fun generateReport(): String
}

interface Approvable {
    fun approve(item: String): Boolean
    fun reject(item: String, reason: String)
}

interface Trainable {
    val trainingLevel: Int
    fun train()
}

// Manager เป็น Employee (abstract class) และมี capabilities หลายอย่าง
class Manager(
    id: String,
    name: String,
    baseSalary: Double,
    val department: String
) : Employee(id, name, baseSalary), Reportable, Approvable {
    
    private val approvedItems = mutableListOf<String>()
    private val rejectedItems = mutableMapOf<String, String>()
    
    override fun calculateBonus() = baseSalary * 0.20  // 20% bonus
    
    override fun generateReport(): String {
        return """
            Department Report: $department
            Manager: $name
            Approved: ${approvedItems.size} items
            Rejected: ${rejectedItems.size} items
        """.trimIndent()
    }
    
    override fun approve(item: String): Boolean {
        approvedItems.add(item)
        println("$name approved: $item")
        return true
    }
    
    override fun reject(item: String, reason: String) {
        rejectedItems[item] = reason
        println("$name rejected: $item (reason: $reason)")
    }
}

class Developer(
    id: String,
    name: String,
    baseSalary: Double,
    val programmingLanguage: String,
    override val trainingLevel: Int = 1
) : Employee(id, name, baseSalary), Trainable, Reportable {
    
    override fun calculateBonus() = baseSalary * 0.15  // 15% bonus
    
    override fun train() {
        println("$name is learning advanced $programmingLanguage...")
    }
    
    override fun generateReport(): String {
        return "Developer $name: $programmingLanguage (Level $trainingLevel)"
    }
}

fun main() {
    val manager = Manager("M001", "Alice", 8000.0, "Engineering")
    val dev = Developer("D001", "Bob", 6000.0, "Kotlin")
    
    println(manager.introduce())
    println("Total: ${manager.totalCompensation()}")
    manager.approve("New Feature Proposal")
    manager.reject("Old Project", "Not aligned with goals")
    println(manager.generateReport())
    
    println()
    
    println(dev.introduce())
    println("Total: ${dev.totalCompensation()}")
    dev.train()
    println(dev.generateReport())
}
```

---

## 🔀 6. Multiple Interfaces

```kotlin
interface Flyable {
    val maxAltitude: Int
    fun fly() = println("Flying up to ${maxAltitude}m")
    fun land() = println("Landing...")
}

interface Swimmable {
    val maxDepth: Int
    fun swim() = println("Swimming to ${maxDepth}m depth")
    fun surface() = println("Coming to surface...")
}

interface Runnable {
    val maxSpeed: Int  // km/h
    fun run() = println("Running at ${maxSpeed} km/h")
}

// Duck ทำได้ทุกอย่าง!
class Duck(
    val name: String,
    override val maxAltitude: Int = 100,
    override val maxDepth: Int = 2,
    override val maxSpeed: Int = 5
) : Flyable, Swimmable, Runnable {
    
    fun showOff() {
        println("$name can:")
        fly()
        swim()
        run()
    }
}

// ใช้ interface type ในฟังก์ชัน
fun makeItFly(flyable: Flyable) {
    println("Making something fly:")
    flyable.fly()
}

fun makeItSwim(swimmable: Swimmable) {
    println("Making something swim:")
    swimmable.swim()
}

fun main() {
    val duck = Duck("Donald")
    duck.showOff()
    
    println("---")
    makeItFly(duck)
    makeItSwim(duck)
}
```

### Diamond Problem - การ Resolve Conflicts

```kotlin
interface A {
    fun hello() = println("Hello from A")
}

interface B : A {
    override fun hello() = println("Hello from B")
}

interface C : A {
    override fun hello() = println("Hello from C")
}

// D inherit ทั้ง B และ C ซึ่งทั้งคู่ override hello() จาก A
class D : B, C {
    // Kotlin บังคับให้ resolve conflict โดย override และระบุว่าจะเรียกของใคร
    override fun hello() {
        super<B>.hello()  // เรียกของ B
        super<C>.hello()  // เรียกของ C
    }
}

fun main() {
    val d = D()
    d.hello()
    // Hello from B
    // Hello from C
}
```

---

## 🎯 7. Interface Delegation

Interface Delegation คือการให้ class "delegate" การ implement interface ไปให้ object อื่น แทนที่จะ implement เอง ใช้ keyword `by`

```kotlin
interface Logger {
    fun log(message: String)
    fun logError(message: String) = log("ERROR: $message")
    fun logInfo(message: String) = log("INFO: $message")
}

class ConsoleLogger : Logger {
    override fun log(message: String) = println("[Console] $message")
}

class FileLogger(val filename: String) : Logger {
    override fun log(message: String) = println("[File:$filename] $message")
}

// Delegation: UserService จะ delegate Logger methods ไปให้ logger object
class UserService(logger: Logger) : Logger by logger {
    // ไม่ต้อง implement log() เอง - delegate ให้ logger แล้ว
    
    fun createUser(name: String) {
        logInfo("Creating user: $name")
        // ... business logic ...
        logInfo("User $name created successfully")
    }
    
    fun deleteUser(name: String) {
        logInfo("Deleting user: $name")
        // ... business logic ...
        logInfo("User $name deleted")
    }
}

fun main() {
    // ใช้ ConsoleLogger
    val consoleService = UserService(ConsoleLogger())
    consoleService.createUser("Alice")
    
    println()
    
    // เปลี่ยนเป็น FileLogger ง่ายๆ
    val fileService = UserService(FileLogger("users.log"))
    fileService.createUser("Bob")
}
```

### ตัวอย่าง Delegation ที่ซับซ้อนขึ้น

```kotlin
interface Cache<K, V> {
    fun get(key: K): V?
    fun put(key: K, value: V)
    fun remove(key: K)
    fun size(): Int
    fun clear()
}

class SimpleCache<K, V> : Cache<K, V> {
    private val store = mutableMapOf<K, V>()
    
    override fun get(key: K): V? = store[key]
    override fun put(key: K, value: V) { store[key] = value }
    override fun remove(key: K) { store.remove(key) }
    override fun size(): Int = store.size
    override fun clear() = store.clear()
}

// TimedCache เพิ่ม TTL (Time To Live) functionality โดย delegate ส่วนที่เหลือ
class TimedCache<K, V>(
    private val delegate: Cache<K, V>,
    private val ttlMs: Long = 60_000  // 1 minute default
) : Cache<K, V> by delegate {
    private val timestamps = mutableMapOf<K, Long>()
    
    override fun get(key: K): V? {
        val timestamp = timestamps[key] ?: return null
        return if (System.currentTimeMillis() - timestamp > ttlMs) {
            // Expired!
            remove(key)
            null
        } else {
            delegate.get(key)
        }
    }
    
    override fun put(key: K, value: V) {
        timestamps[key] = System.currentTimeMillis()
        delegate.put(key, value)
    }
    
    override fun remove(key: K) {
        timestamps.remove(key)
        delegate.remove(key)
    }
    
    override fun clear() {
        timestamps.clear()
        delegate.clear()
    }
}

// LoggingCache เพิ่ม logging โดย delegate ส่วนที่เหลือ
class LoggingCache<K, V>(
    private val delegate: Cache<K, V>,
    private val name: String = "Cache"
) : Cache<K, V> by delegate {
    
    override fun get(key: K): V? {
        val value = delegate.get(key)
        println("[$name] GET $key -> ${if (value != null) "HIT" else "MISS"}")
        return value
    }
    
    override fun put(key: K, value: V) {
        println("[$name] PUT $key")
        delegate.put(key, value)
    }
}

fun main() {
    // Stack caches: SimpleCache -> TimedCache -> LoggingCache
    val cache: Cache<String, String> = LoggingCache(
        TimedCache(SimpleCache(), ttlMs = 5000),
        name = "UserCache"
    )
    
    cache.put("user:1", "Alice")
    cache.put("user:2", "Bob")
    
    println(cache.get("user:1"))   // [UserCache] GET user:1 -> HIT -> Alice
    println(cache.get("user:3"))   // [UserCache] GET user:3 -> MISS -> null
    println("Cache size: ${cache.size()}")
}
```

---

## 🔒 8. Sealed Classes

`sealed class` คือ class ที่ **จำกัดว่า subclass สามารถเป็นอะไรได้บ้าง** โดย subclass ทั้งหมดต้องอยู่ใน **package เดียวกัน** (Kotlin 1.5+) ทำให้ compiler รู้ว่ามี subclass ทั้งหมดกี่อัน ทำให้ใช้กับ `when` ได้อย่างปลอดภัยโดยไม่ต้องมี `else`

### Sealed Class พื้นฐาน

```kotlin
sealed class Result<out T> {
    data class Success<T>(val data: T) : Result<T>()
    data class Error(val message: String, val code: Int = 0) : Result<Nothing>()
    object Loading : Result<Nothing>()
}

fun fetchUser(id: Int): Result<String> {
    return when (id) {
        1 -> Result.Success("Alice")
        2 -> Result.Success("Bob")
        -1 -> Result.Error("Invalid ID", code = 400)
        else -> Result.Error("User not found", code = 404)
    }
}

fun handleResult(result: Result<String>) {
    when (result) {
        is Result.Success -> println("✅ Success: ${result.data}")
        is Result.Error -> println("❌ Error ${result.code}: ${result.message}")
        is Result.Loading -> println("⏳ Loading...")
        // ไม่ต้อง else! compiler รู้ว่า cover ทุก case แล้ว
    }
}

fun main() {
    handleResult(fetchUser(1))    // ✅ Success: Alice
    handleResult(fetchUser(99))   // ❌ Error 404: User not found
    handleResult(fetchUser(-1))   // ❌ Error 400: Invalid ID
    handleResult(Result.Loading)  // ⏳ Loading...
}
```

### Sealed Interface (Kotlin 1.5+)

```kotlin
sealed interface NetworkEvent {
    data class Connected(val networkType: String) : NetworkEvent
    data class Disconnected(val reason: String) : NetworkEvent
    data class DataReceived(val bytes: Int) : NetworkEvent
    object Reconnecting : NetworkEvent
}

fun processNetworkEvent(event: NetworkEvent) {
    val message = when (event) {
        is NetworkEvent.Connected -> "Connected via ${event.networkType}"
        is NetworkEvent.Disconnected -> "Disconnected: ${event.reason}"
        is NetworkEvent.DataReceived -> "Received ${event.bytes} bytes"
        NetworkEvent.Reconnecting -> "Reconnecting..."
    }
    println(message)
}

fun main() {
    val events = listOf(
        NetworkEvent.Connected("WiFi"),
        NetworkEvent.DataReceived(1024),
        NetworkEvent.Disconnected("Signal lost"),
        NetworkEvent.Reconnecting,
        NetworkEvent.Connected("4G")
    )
    
    events.forEach { processNetworkEvent(it) }
}
```

### Sealed Class กับ State Machine

```kotlin
sealed class TrafficLight {
    abstract val duration: Int  // seconds
    abstract val next: TrafficLight
    
    object Red : TrafficLight() {
        override val duration = 30
        override val next get() = Green
        override fun toString() = "🔴 RED"
    }
    
    object Yellow : TrafficLight() {
        override val duration = 5
        override val next get() = Red
        override fun toString() = "🟡 YELLOW"
    }
    
    object Green : TrafficLight() {
        override val duration = 25
        override val next get() = Yellow
        override fun toString() = "🟢 GREEN"
    }
}

fun main() {
    var light: TrafficLight = TrafficLight.Red
    
    repeat(6) {
        println("$light - wait ${light.duration} seconds")
        light = light.next
    }
    // 🔴 RED - wait 30 seconds
    // 🟢 GREEN - wait 25 seconds
    // 🟡 YELLOW - wait 5 seconds
    // 🔴 RED - wait 30 seconds
    // 🟢 GREEN - wait 25 seconds
    // 🟡 YELLOW - wait 5 seconds
}
```

---

## 🌳 9. ตัวอย่างจริง: Shape Hierarchy

```kotlin
import kotlin.math.*

// Interface สำหรับ transformations
interface Transformable {
    fun scale(factor: Double): Shape
    fun translate(dx: Double, dy: Double): Shape
}

// Interface สำหรับ rendering
interface Renderable {
    fun render(): String
}

// Base sealed class
sealed class Shape : Transformable, Renderable {
    abstract val name: String
    abstract val area: Double
    abstract val perimeter: Double
    
    fun info() = buildString {
        appendLine("Shape: $name")
        appendLine("Area: ${"%.4f".format(area)}")
        appendLine("Perimeter: ${"%.4f".format(perimeter)}")
    }
    
    companion object {
        fun totalArea(shapes: List<Shape>) = shapes.sumOf { it.area }
        fun largestShape(shapes: List<Shape>) = shapes.maxByOrNull { it.area }
        
        fun groupByType(shapes: List<Shape>): Map<String, List<Shape>> {
            return shapes.groupBy { it.name }
        }
    }
}

data class Circle(
    val radius: Double,
    val x: Double = 0.0,
    val y: Double = 0.0
) : Shape() {
    override val name = "Circle"
    override val area = PI * radius * radius
    override val perimeter = 2 * PI * radius
    
    override fun scale(factor: Double) = copy(radius = radius * factor)
    override fun translate(dx: Double, dy: Double) = copy(x = x + dx, y = y + dy)
    override fun render() = "○ Circle(r=$radius) at ($x,$y)"
}

data class Rectangle(
    val width: Double,
    val height: Double,
    val x: Double = 0.0,
    val y: Double = 0.0
) : Shape() {
    override val name = "Rectangle"
    override val area = width * height
    override val perimeter = 2 * (width + height)
    
    val isSquare get() = width == height
    
    override fun scale(factor: Double) = copy(width = width * factor, height = height * factor)
    override fun translate(dx: Double, dy: Double) = copy(x = x + dx, y = y + dy)
    override fun render() = "▭ Rectangle(${width}x${height}) at ($x,$y)"
}

data class Triangle(
    val a: Double,  // side lengths
    val b: Double,
    val c: Double
) : Shape() {
    init {
        require(a + b > c && b + c > a && a + c > b) {
            "Invalid triangle: sides $a, $b, $c don't satisfy triangle inequality"
        }
    }
    
    override val name = "Triangle"
    override val perimeter = a + b + c
    override val area: Double get() {
        // Heron's formula
        val s = perimeter / 2
        return sqrt(s * (s - a) * (s - b) * (s - c))
    }
    
    val isEquilateral get() = a == b && b == c
    val isIsosceles get() = a == b || b == c || a == c
    val isRight: Boolean get() {
        val sides = listOf(a, b, c).sorted()
        return abs(sides[0] * sides[0] + sides[1] * sides[1] - sides[2] * sides[2]) < 0.0001
    }
    
    override fun scale(factor: Double) = copy(a = a * factor, b = b * factor, c = c * factor)
    override fun translate(dx: Double, dy: Double) = this  // Triangle ไม่มี position ในตัวอย่างนี้
    override fun render() = "△ Triangle($a, $b, $c)"
}

fun main() {
    val shapes = listOf(
        Circle(5.0),
        Circle(3.0, 10.0, 10.0),
        Rectangle(4.0, 6.0),
        Rectangle(5.0, 5.0),
        Triangle(3.0, 4.0, 5.0),
        Triangle(6.0, 6.0, 6.0)
    )
    
    // Render all shapes
    println("=== All Shapes ===")
    shapes.forEach { println(it.render()) }
    
    println("\n=== Shape Info ===")
    shapes.forEach { 
        print(it.info())
        println()
    }
    
    println("=== Statistics ===")
    println("Total area: ${"%.2f".format(Shape.totalArea(shapes))}")
    println("Largest: ${Shape.largestShape(shapes)?.render()}")
    
    println("\n=== Grouped by Type ===")
    Shape.groupByType(shapes).forEach { (type, list) ->
        println("$type (${list.size}): ${list.map { it.render() }}")
    }
    
    println("\n=== Transformations ===")
    val circle = Circle(5.0, 0.0, 0.0)
    println("Original: ${circle.render()}")
    println("Scaled 2x: ${circle.scale(2.0).render()}")
    println("Translated: ${circle.translate(3.0, 4.0).render()}")
    
    // Special properties
    val rect1 = Rectangle(5.0, 5.0)
    val tri1 = Triangle(3.0, 4.0, 5.0)
    println("\nIs square: ${rect1.isSquare}")
    println("Is right triangle: ${tri1.isRight}")
}
```

---

## 🦁 10. ตัวอย่างจริง: Animal Hierarchy

```kotlin
// Interfaces สำหรับ behaviors
interface Feedable {
    val diet: String
    val feedingFrequency: String
    fun feed() = println("Feeding $this ($diet) - $feedingFrequency")
}

interface Trainable {
    val difficultyLevel: Int  // 1-10
    val tricks: MutableList<String>
    
    fun learnTrick(trick: String) {
        tricks.add(trick)
        println("${javaClass.simpleName} learned: $trick")
    }
    
    fun showTricks() {
        if (tricks.isEmpty()) {
            println("${javaClass.simpleName} doesn't know any tricks yet")
        } else {
            println("${javaClass.simpleName}'s tricks: ${tricks.joinToString(", ")}")
        }
    }
}

interface MedicalRecord {
    val vaccinations: MutableList<String>
    val medications: MutableList<String>
    
    fun vaccinate(vaccine: String) {
        vaccinations.add(vaccine)
        println("Vaccinated with: $vaccine")
    }
    
    fun prescribe(medication: String) {
        medications.add(medication)
    }
    
    fun healthStatus(): String
}

// Abstract base class
abstract class Animal(
    val id: String,
    val name: String,
    val species: String,
    val birthYear: Int
) : Feedable, MedicalRecord {
    
    val age: Int get() = 2024 - birthYear
    
    abstract val sound: String
    abstract fun describe(): String
    
    override val vaccinations = mutableListOf<String>()
    override val medications = mutableListOf<String>()
    
    override fun healthStatus(): String {
        return "[$name] Vaccinations: ${vaccinations.size}, Medications: ${medications.size}"
    }
    
    override fun toString() = "$name ($species)"
    
    fun makeSound() = println("$name says: $sound")
}

// Domestic animals
abstract class DomesticAnimal(
    id: String,
    name: String,
    species: String,
    birthYear: Int,
    val ownerName: String
) : Animal(id, name, species, birthYear), Trainable {
    
    override val tricks = mutableListOf<String>()
    
    override fun describe(): String {
        return """
            |Name: $name
            |Species: $species  
            |Age: $age years
            |Owner: $ownerName
            |Diet: $diet
            |Tricks: ${if (tricks.isEmpty()) "none" else tricks.joinToString(", ")}
        """.trimMargin()
    }
}

class Dog(
    id: String,
    name: String,
    birthYear: Int,
    ownerName: String,
    val breed: String,
    val size: String = "medium"  // small, medium, large
) : DomesticAnimal(id, name, "Canis lupus familiaris", birthYear, ownerName) {
    
    override val sound = "Woof!"
    override val diet = "Omnivore (commercial dog food + meat)"
    override val feedingFrequency = "2x per day"
    override val difficultyLevel = when (breed) {
        "Border Collie" -> 3
        "Golden Retriever" -> 2
        else -> 5
    }
    
    fun fetch(item: String) = println("$name fetches the $item!")
    fun guard() = println("$name is guarding the area!")
}

class Cat(
    id: String,
    name: String,
    birthYear: Int,
    ownerName: String,
    val indoorOnly: Boolean = true
) : DomesticAnimal(id, name, "Felis catus", birthYear, ownerName) {
    
    override val sound = "Meow~"
    override val diet = "Carnivore (commercial cat food)"
    override val feedingFrequency = "3-4x per day"
    override val difficultyLevel = 8  // cats are harder to train!
    
    fun purr() = println("$name purrs contentedly... *purrrrr*")
    fun scratch() = println("$name scratches the furniture...")
}

// Wild animals
abstract class WildAnimal(
    id: String,
    name: String,
    species: String,
    birthYear: Int,
    val habitat: String,
    val conservationStatus: String  // LC, NT, VU, EN, CR, EW, EX
) : Animal(id, name, species, birthYear) {
    
    abstract val territory: Double  // km²
    
    override fun describe(): String {
        return """
            |Name: $name
            |Species: $species
            |Age: $age years
            |Habitat: $habitat
            |Conservation: $conservationStatus
            |Territory: $territory km²
        """.trimMargin()
    }
}

class Lion(
    id: String,
    name: String,
    birthYear: Int,
    habitat: String,
    val isPride: Boolean = true
) : WildAnimal(id, name, "Panthera leo", birthYear, habitat, "VU") {
    
    override val sound = "ROAR!"
    override val diet = "Carnivore"
    override val feedingFrequency = "Every 3-4 days"
    override val territory = 100.0  // km²
    
    fun hunt() = println("$name ${if (isPride) "hunts with the pride" else "hunts alone"}!")
}

class Elephant(
    id: String,
    name: String,
    birthYear: Int,
    habitat: String,
    val isAfrican: Boolean = true
) : WildAnimal(
    id, name, 
    if (isAfrican) "Loxodonta africana" else "Elephas maximus",
    birthYear, habitat,
    if (isAfrican) "VU" else "EN"
) {
    override val sound = "Trumpet!"
    override val diet = "Herbivore"
    override val feedingFrequency = "Throughout the day (~16 hours)"
    override val territory = 11400.0  // km² (African elephant)
    
    fun trumpet() = println("$name trumpets loudly!")
    fun usesTrunk(action: String) = println("$name uses trunk to $action")
}

// Zoo management system
class Zoo(val name: String, val location: String) {
    private val animals = mutableListOf<Animal>()
    
    fun addAnimal(animal: Animal) {
        animals.add(animal)
        println("${animal.name} added to $name Zoo")
    }
    
    fun removeAnimal(id: String): Animal? {
        val animal = animals.find { it.id == id }
        animal?.let { animals.remove(it) }
        return animal
    }
    
    fun findById(id: String) = animals.find { it.id == id }
    
    fun listAll() {
        println("\n=== Animals at $name Zoo ===")
        animals.forEach { animal ->
            println("• ${animal.name} (${animal.species}) - Age: ${animal.age}")
        }
    }
    
    fun listByType() {
        println("\n=== Animals by Category ===")
        val domestic = animals.filterIsInstance<DomesticAnimal>()
        val wild = animals.filterIsInstance<WildAnimal>()
        
        println("Domestic (${domestic.size}):")
        domestic.forEach { println("  - ${it.name} [${it.species}]") }
        
        println("Wild (${wild.size}):")
        wild.forEach { println("  - ${it.name} [${it.species}]") }
    }
    
    fun trainableAnimals(): List<Trainable> {
        return animals.filterIsInstance<Trainable>()
    }
    
    fun feedAll() {
        println("\n=== Feeding Time ===")
        animals.forEach { it.feed() }
    }
    
    fun healthReport() {
        println("\n=== Health Report ===")
        animals.forEach { println(it.healthStatus()) }
    }
}

fun main() {
    val zoo = Zoo("Happy Animals", "Bangkok")
    
    val buddy = Dog("D001", "Buddy", 2020, "Alice", "Golden Retriever", "large")
    val whiskers = Cat("C001", "Whiskers", 2021, "Bob")
    val simba = Lion("L001", "Simba", 2018, "African Savanna")
    val dumbo = Elephant("E001", "Dumbo", 2015, "Asian Jungle", isAfrican = false)
    
    zoo.addAnimal(buddy)
    zoo.addAnimal(whiskers)
    zoo.addAnimal(simba)
    zoo.addAnimal(dumbo)
    
    // Train domestic animals
    println("\n=== Training Session ===")
    zoo.trainableAnimals().forEach { trainable ->
        trainable.learnTrick("sit")
        trainable.learnTrick("stay")
    }
    
    // Vaccinations
    println("\n=== Vaccination Day ===")
    buddy.vaccinate("Rabies")
    buddy.vaccinate("DHPP")
    whiskers.vaccinate("FVRCP")
    
    // Special actions
    println("\n=== Daily Activities ===")
    buddy.makeSound()
    buddy.fetch("ball")
    whiskers.makeSound()
    whiskers.purr()
    simba.makeSound()
    simba.hunt()
    dumbo.trumpet()
    dumbo.usesTrunk("pick up food")
    
    // Reports
    zoo.listAll()
    zoo.listByType()
    zoo.healthReport()
    
    println("\n=== Individual Details ===")
    println(buddy.describe())
    println()
    println(simba.describe())
}
```

---

## 📝 สรุป Part 07

| แนวคิด | รายละเอียด |
|--------|-----------|
| `open class` | อนุญาตให้ inherit ได้ (default เป็น final) |
| `override` | Override method/property จาก parent |
| `super` | เรียก parent implementation |
| `final override` | ป้องกัน override ต่อใน subclass |
| `abstract class` | Blueprint ที่ไม่สามารถสร้าง instance ได้ |
| `interface` | Contract ที่กำหนด behaviors |
| `by` | Interface delegation |
| `sealed class` | Restricted class hierarchy |
| Multiple interfaces | Class implement หลาย interfaces ได้ |
| Diamond problem | ใช้ `super<X>` เพื่อ specify |

### เมื่อไหรใช้อะไร?

- **open class**: ต้องการ share code + state และ "is-a" relationship
- **abstract class**: ต้องการ template method หรือ shared state ใน hierarchy
- **interface**: ต้องการกำหนด capabilities ที่ class หลายๆ อันสามารถมีได้
- **sealed class**: ต้องการ represent state machine หรือ discriminated union
- **delegation**: ต้องการ compose behaviors โดยไม่ inherit

---

## 🏋️ แบบฝึกหัด Part 07

### ระดับ 1 (พื้นฐาน)

1. สร้าง `open class Vehicle` ที่มี properties: `brand`, `year`, `speed` และ methods: `accelerate()`, `brake()` จากนั้นสร้าง `Car` และ `Motorcycle` ที่ inherit จาก Vehicle

2. สร้าง `interface Drawable` ที่มี method `draw()` และ `interface Resizable` ที่มี method `resize(factor: Double)` จากนั้นสร้าง class ที่ implement ทั้งสองพร้อมกัน

3. สร้าง `abstract class Food` ที่มี abstract properties `calories` และ `servingSize` พร้อม concrete method `nutritionLabel()`

### ระดับ 2 (กลาง)

4. สร้าง sealed class `PaymentStatus` ที่มี states: `Pending`, `Processing(transactionId)`, `Completed(transactionId, amount)`, `Failed(reason)` และเขียนฟังก์ชัน `describePayment()` ที่ใช้ when expression

5. ใช้ interface delegation สร้าง `LoggingRepository` ที่ wrap `UserRepository` โดยเพิ่ม logging ทุกครั้งที่มีการ CRUD

6. สร้าง `sealed class Expr` สำหรับ expression tree: `Num(value)`, `Add(left, right)`, `Mul(left, right)`, `Neg(expr)` และเขียน `evaluate()` function

### ระดับ 3 (ท้าทาย)

7. สร้าง UI Component hierarchy:
   - `abstract class Component` (id, visible, onClick)
   - `interface Styleable` (color, fontSize, padding)
   - `interface Animatable` (animate(), stopAnimation())
   - `class Button : Component, Styleable, Animatable`
   - `class TextField : Component, Styleable`
   - `class Container : Component` ที่เก็บ List ของ Component

8. ออกแบบ `sealed class GameState` สำหรับเกม และ state machine ที่จัดการ transitions ระหว่าง states

---

## ➡️ ถัดไป: Part 08 - Data Classes, Sealed Classes และ Enum

---
*Part 07/100+ | Kotlin & Spring Boot Complete Course*
