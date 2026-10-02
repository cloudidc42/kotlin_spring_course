# Part 86: Code Quality
## SOLID, Refactoring, และ Static Analysis

---

## 🎯 เป้าหมายของ Part นี้

- เข้าใจ SOLID Principles
- ระบุ Code Smells และแก้ไข
- ทำ Refactoring อย่างปลอดภัย
- ใช้ Detekt และ SonarQube
- Code Review best practices

---

## 📖 1. SOLID Principles

### S: Single Responsibility Principle (SRP)

```kotlin
// ❌ ละเมิด SRP: Class ทำงานหลายอย่าง
class UserManager {
    fun createUser(user: User) { /* database logic */ }
    fun sendWelcomeEmail(user: User) { /* email logic */ }
    fun generateUserReport(users: List<User>): String { /* report logic */ }
    fun validateUserData(user: User): Boolean { /* validation logic */ }
}

// ✅ ถูกต้อง: แยก responsibilities
class UserRepository {
    fun save(user: User): User { /* database logic only */ }
}

class EmailService {
    fun sendWelcomeEmail(user: User) { /* email logic only */ }
}

class UserReportService {
    fun generateReport(users: List<User>): String { /* report logic only */ }
}

class UserValidator {
    fun validate(user: User): ValidationResult { /* validation logic only */ }
}
```

### O: Open/Closed Principle (OCP)

```kotlin
// ❌ ละเมิด OCP: ต้อง modify code เมื่อเพิ่ม payment type
class PaymentProcessor {
    fun process(type: String, amount: Double) {
        when (type) {
            "CREDIT_CARD" -> processCreditCard(amount)
            "PAYPAL" -> processPaypal(amount)
            // ถ้าจะเพิ่ม crypto ต้อง modify class นี้
            else -> throw IllegalArgumentException("Unknown type")
        }
    }
}

// ✅ ถูกต้อง: Open for extension, closed for modification
interface PaymentGateway {
    fun process(amount: Double): PaymentResult
    fun supports(type: PaymentType): Boolean
}

class CreditCardGateway : PaymentGateway {
    override fun process(amount: Double) = PaymentResult(success = true)
    override fun supports(type: PaymentType) = type == PaymentType.CREDIT_CARD
}

class PaypalGateway : PaymentGateway {
    override fun process(amount: Double) = PaymentResult(success = true)
    override fun supports(type: PaymentType) = type == PaymentType.PAYPAL
}

// เพิ่ม CryptoGateway โดยไม่ต้อง modify PaymentProcessor
class CryptoGateway : PaymentGateway {
    override fun process(amount: Double) = PaymentResult(success = true)
    override fun supports(type: PaymentType) = type == PaymentType.CRYPTO
}

class PaymentProcessor(private val gateways: List<PaymentGateway>) {
    fun process(type: PaymentType, amount: Double): PaymentResult {
        val gateway = gateways.find { it.supports(type) }
            ?: throw IllegalArgumentException("No gateway for $type")
        return gateway.process(amount)
    }
}
```

### L: Liskov Substitution Principle (LSP)

```kotlin
// ❌ ละเมิด LSP: Subclass เปลี่ยน behavior
open class Rectangle(open var width: Int, open var height: Int) {
    fun area() = width * height
}

class Square(side: Int) : Rectangle(side, side) {
    override var width: Int
        get() = super.width
        set(value) { super.width = value; super.height = value }  // เปลี่ยน both!
    // ทำให้ code ที่ expect Rectangle จะ fail
}

// ✅ ถูกต้อง: ใช้ interface แทน
interface Shape {
    fun area(): Int
}

class Rectangle(val width: Int, val height: Int) : Shape {
    override fun area() = width * height
}

class Square(val side: Int) : Shape {
    override fun area() = side * side
}
```

### I: Interface Segregation Principle (ISP)

```kotlin
// ❌ ละเมิด ISP: Interface ใหญ่เกินไป
interface UserService {
    fun createUser(user: User): User
    fun deleteUser(id: Long)
    fun updateUser(id: Long, user: User): User
    fun generateReport(): ByteArray
    fun sendEmail(userId: Long, message: String)
    fun exportToExcel(): ByteArray
}

// ✅ ถูกต้อง: แยก interfaces
interface UserCrudService {
    fun createUser(user: User): User
    fun deleteUser(id: Long)
    fun updateUser(id: Long, user: User): User
}

interface UserReportService {
    fun generateReport(): ByteArray
    fun exportToExcel(): ByteArray
}

interface UserNotificationService {
    fun sendEmail(userId: Long, message: String)
}
```

### D: Dependency Inversion Principle (DIP)

```kotlin
// ❌ ละเมิด DIP: High-level module depends on low-level
class OrderService {
    private val mysqlOrderRepository = MySqlOrderRepository() // concrete!
    private val smtpEmailService = SmtpEmailService()        // concrete!
}

// ✅ ถูกต้อง: Depend on abstractions
interface OrderRepository {
    fun save(order: Order): Order
    fun findById(id: Long): Order?
}

interface EmailService {
    fun send(to: String, subject: String, body: String)
}

class OrderService(
    private val orderRepository: OrderRepository,  // abstraction
    private val emailService: EmailService          // abstraction
) {
    // ไม่รู้ว่า implementation คืออะไร (MySQL? MongoDB? SES? Sendgrid?)
}
```

---

## 🐛 2. Code Smells

### Long Method

```kotlin
// ❌ Code Smell: Method ยาวเกินไป
fun processOrder(order: Order): OrderResult {
    // Validate order
    if (order.items.isEmpty()) throw ValidationException("No items")
    if (order.userId <= 0) throw ValidationException("Invalid user")
    order.items.forEach { item ->
        if (item.quantity <= 0) throw ValidationException("Invalid quantity")
        if (item.productId <= 0) throw ValidationException("Invalid product")
    }

    // Check inventory
    val inventoryResults = order.items.map { item ->
        val product = productRepository.findById(item.productId)
            ?: throw NotFoundException("Product ${item.productId} not found")
        if (product.stock < item.quantity) {
            throw InsufficientStockException("Not enough stock for ${product.name}")
        }
        product
    }

    // Calculate total
    var total = 0.0
    order.items.forEachIndexed { i, item ->
        total += inventoryResults[i].price * item.quantity
    }

    // Apply discount
    val user = userRepository.findById(order.userId)!!
    val discount = when {
        user.isPremium -> 0.1
        total > 1000 -> 0.05
        else -> 0.0
    }
    val finalTotal = total * (1 - discount)

    // Save and notify
    val savedOrder = orderRepository.save(order.copy(total = finalTotal, status = "CONFIRMED"))
    emailService.sendConfirmation(user.email, savedOrder)
    return OrderResult(savedOrder, finalTotal)
}

// ✅ Refactored: แยกเป็น private methods ที่ชัดเจน
fun processOrder(order: Order): OrderResult {
    validateOrder(order)
    val products = checkInventory(order)
    val total = calculateTotal(order, products)
    val finalTotal = applyDiscount(total, order.userId)
    return saveAndNotify(order, finalTotal)
}

private fun validateOrder(order: Order) {
    require(order.items.isNotEmpty()) { "No items" }
    require(order.userId > 0) { "Invalid user" }
    order.items.forEach { item ->
        require(item.quantity > 0) { "Invalid quantity" }
        require(item.productId > 0) { "Invalid product" }
    }
}
```

### Magic Numbers

```kotlin
// ❌ Magic Numbers
fun calculateDiscount(total: Double, userType: Int): Double {
    return when (userType) {
        1 -> total * 0.1    // 1 คืออะไร? 0.1 คือเท่าไหร่?
        2 -> total * 0.15
        else -> total * 0.05
    }
}

// ✅ Named Constants
enum class UserType(val discountRate: Double) {
    REGULAR(0.05),
    SILVER(0.10),
    GOLD(0.15)
}

fun calculateDiscount(total: Double, userType: UserType): Double {
    return total * userType.discountRate
}
```

---

## 🔧 3. Detekt Configuration

```kotlin
// build.gradle.kts
plugins {
    id("io.gitlab.arturbosch.detekt") version "1.23.4"
}

detekt {
    config.setFrom(file("detekt.yml"))
    buildUponDefaultConfig = true
    allRules = false

    reports {
        html.required.set(true)
        xml.required.set(true)
        txt.required.set(false)
    }
}

tasks.withType<io.gitlab.arturbosch.detekt.Detekt>().configureEach {
    jvmTarget = "21"
}
```

```yaml
# detekt.yml
build:
  maxIssues: 0
  excludeCorrectable: false

complexity:
  active: true
  LongMethod:
    active: true
    threshold: 60
  ComplexMethod:
    active: true
    threshold: 15
  LargeClass:
    active: true
    threshold: 600
  TooManyFunctions:
    active: true
    threshold: 15

naming:
  active: true
  FunctionNaming:
    active: true
    functionPattern: '[a-z][a-zA-Z0-9]*'
  VariableNaming:
    active: true
    variablePattern: '[a-z][a-zA-Z0-9]*'

style:
  active: true
  MagicNumber:
    active: true
    ignoreNumbers: ['-1', '0', '1', '2']
  WildcardImport:
    active: true
  UnusedImports:
    active: true

performance:
  active: true
  SpreadOperator:
    active: true
```

```bash
# Run Detekt
./gradlew detekt

# Auto-fix some issues
./gradlew detektMain --auto-correct
```

---

## 📊 4. SonarQube Integration

```yaml
# docker-compose.yml
version: '3'
services:
  sonarqube:
    image: sonarqube:10-community
    ports:
      - "9000:9000"
    environment:
      SONAR_ES_BOOTSTRAP_CHECKS_DISABLE: "true"
    volumes:
      - sonarqube_data:/opt/sonarqube/data

volumes:
  sonarqube_data:
```

```kotlin
// build.gradle.kts
plugins {
    id("org.sonarqube") version "4.4.1.3373"
    id("jacoco")
}

sonar {
    properties {
        property("sonar.projectKey", "my-kotlin-project")
        property("sonar.projectName", "My Kotlin Spring Boot Project")
        property("sonar.host.url", "http://localhost:9000")
        property("sonar.token", System.getenv("SONAR_TOKEN"))
        property("sonar.coverage.jacoco.xmlReportPaths",
            "${project.buildDir}/reports/jacoco/test/jacocoTestReport.xml")
        property("sonar.kotlin.detekt.reportPaths",
            "${project.buildDir}/reports/detekt/detekt.xml")
    }
}

jacoco {
    toolVersion = "0.8.11"
}

tasks.jacocoTestReport {
    dependsOn(tasks.test)
    reports {
        xml.required.set(true)
        html.required.set(true)
    }
}
```

```bash
# Start SonarQube
docker-compose up -d

# Run analysis
./gradlew test jacocoTestReport sonar \
  -Dsonar.token=your-token-here
```

---

## 👀 5. Code Review Best Practices

### Code Review Checklist

```markdown
# Code Review Checklist

## Correctness
- [ ] Logic ถูกต้องตาม requirements
- [ ] Edge cases ถูกจัดการ (null, empty, boundary)
- [ ] Error handling ครบถ้วน
- [ ] No race conditions ใน concurrent code

## Design
- [ ] SOLID principles ถูกใช้
- [ ] ไม่มี unnecessary dependencies
- [ ] Abstraction levels เหมาะสม
- [ ] Code reuse (ไม่ duplicate)

## Tests
- [ ] Unit tests ครบ
- [ ] Test cases cover edge cases
- [ ] Test names สื่อความหมาย
- [ ] ไม่ทดสอบ implementation details

## Performance
- [ ] ไม่มี N+1 queries
- [ ] Appropriate caching
- [ ] ไม่ทำ heavy computation ใน hot path

## Security
- [ ] ไม่มี SQL injection
- [ ] Input validation ครบ
- [ ] ไม่ log sensitive data
- [ ] Authorization check ทุก endpoint
```

### Commit Message Convention

```bash
# Format: <type>(<scope>): <subject>
#
# type: feat, fix, docs, style, refactor, test, chore
# scope: optional, what area of code
# subject: short description

# Examples:
feat(auth): add JWT refresh token rotation
fix(orders): prevent duplicate order creation on retry
refactor(users): extract UserValidator from UserService
test(products): add integration tests for discount calculation
docs(api): update OpenAPI specification for v2

# Breaking changes:
feat!: change order response to include nested product details
```

---

## 📋 สรุป

| Principle | ความหมาย | ประโยชน์ |
|-----------|---------|---------|
| SRP | 1 class = 1 responsibility | ง่ายต่อการ maintain |
| OCP | Open to extend, closed to modify | ลด regression bugs |
| LSP | Subclass ใช้แทน parent ได้ | Code reliable |
| ISP | Interface เล็กๆ specific | ลด coupling |
| DIP | Depend on abstractions | Test ง่าย, swap implementation ได้ |

| Tool | วัตถุประสงค์ |
|------|------------|
| Detekt | Static analysis สำหรับ Kotlin |
| SonarQube | Code quality + security scan |
| JaCoCo | Code coverage |

---

*Part 86/100+ | Kotlin & Spring Boot Complete Course*
