# Part 69: Advanced Testing

## Advanced Testing — Property-based, Mutation, Performance และ BDD Testing

---

## 🎯 เป้าหมายของ Part นี้

- Property-based testing ด้วย Kotest
- Mutation testing ด้วย PITest
- Performance testing ด้วย JMH
- BDD testing ด้วย Cucumber
- Test fixtures และ factories
- สร้าง comprehensive test suite

---

## 📖 1. Testing Pyramid

```
          /\
         /e2e\
        /------\
       /  integ  \
      /------------\
     /     unit     \
    /-----------------\
```

- **Unit tests** — เร็ว, มาก (70%)
- **Integration tests** — ปานกลาง (20%)
- **E2E tests** — ช้า, น้อย (10%)

---

## ⚗️ 2. Property-Based Testing ด้วย Kotest

Property-based testing ทดสอบ properties ที่ควรเป็นจริงสำหรับข้อมูลทุกชุด ไม่ใช่แค่ตัวอย่างที่กำหนด

### Dependencies

```kotlin
// build.gradle.kts
testImplementation("io.kotest:kotest-runner-junit5:5.8.0")
testImplementation("io.kotest:kotest-assertions-core:5.8.0")
testImplementation("io.kotest:kotest-property:5.8.0")
testImplementation("io.kotest:kotest-extensions-spring:1.1.3")
```

### ตัวอย่าง Property-Based Test

```kotlin
import io.kotest.core.spec.style.StringSpec
import io.kotest.property.*
import io.kotest.property.arbitrary.*
import io.kotest.matchers.*

class StringUtilsPropertyTest : StringSpec({
    
    "reverse twice should equal original" {
        forAll(Arb.string()) { str ->
            str.reversed().reversed() == str
        }
    }

    "concatenation length should equal sum of lengths" {
        forAll(Arb.string(), Arb.string()) { a, b ->
            (a + b).length == a.length + b.length
        }
    }

    "sorted list should have same elements as original" {
        forAll(Arb.list(Arb.int(), 0..100)) { list ->
            list.sorted().toSet() == list.toSet()
        }
    }

    "sorted list should be in ascending order" {
        forAll(Arb.list(Arb.int(), 0..100)) { list ->
            val sorted = list.sorted()
            sorted.zipWithNext().all { (a, b) -> a <= b }
        }
    }
})
```

### Custom Arbitraries (Generators)

```kotlin
import io.kotest.property.arbitrary.*

data class Money(val amount: Long, val currency: String) {
    operator fun plus(other: Money): Money {
        require(currency == other.currency) { "Cannot add different currencies" }
        return Money(amount + other.amount, currency)
    }
}

val arbMoney = arbitrary {
    Money(
        amount = Arb.long(0L..1_000_000L).bind(),
        currency = Arb.element("THB", "USD", "EUR", "JPY").bind()
    )
}

val arbPositiveMoney = arbitrary {
    Money(
        amount = Arb.long(1L..1_000_000L).bind(),
        currency = "THB"
    )
}

class MoneyPropertyTest : StringSpec({
    "addition is commutative" {
        forAll(arbPositiveMoney, arbPositiveMoney) { a, b ->
            a + b == b + a
        }
    }

    "addition is associative" {
        forAll(arbPositiveMoney, arbPositiveMoney, arbPositiveMoney) { a, b, c ->
            (a + b) + c == a + (b + c)
        }
    }

    "zero is identity element" {
        forAll(arbPositiveMoney) { money ->
            money + Money(0L, "THB") == money
        }
    }
})
```

---

## 🧬 3. Mutation Testing ด้วย PITest

Mutation testing ทดสอบคุณภาพของ test suite โดยการแก้ไข (mutate) source code และดูว่า tests จับได้หรือไม่

### ตั้งค่า PITest

```kotlin
// build.gradle.kts
plugins {
    id("info.solidsoft.pitest") version "1.15.0"
}

pitest {
    junit5PluginVersion.set("1.2.1")
    targetClasses.set(listOf("com.example.*"))
    targetTests.set(listOf("com.example.*Test", "com.example.*Spec"))
    mutators.set(listOf("DEFAULTS"))
    outputFormats.set(listOf("XML", "HTML"))
    failWhenNoMutations.set(false)
    threads.set(4)
}
```

### โค้ดที่จะ Mutate

```kotlin
class PriceCalculator {
    fun calculateDiscount(price: Double, discountPercent: Double): Double {
        require(price >= 0) { "Price must be non-negative" }
        require(discountPercent in 0.0..100.0) { "Discount must be 0-100%" }
        return price * (1 - discountPercent / 100)
    }

    fun applyTax(price: Double, taxRate: Double): Double {
        return price * (1 + taxRate / 100)
    }

    fun calculateTotal(items: List<Pair<Double, Int>>): Double {
        return items.sumOf { (price, qty) -> price * qty }
    }
}
```

### Tests ที่ครอบคลุมสำหรับ Mutation

```kotlin
import org.junit.jupiter.api.*
import org.assertj.core.api.Assertions.*

class PriceCalculatorTest {
    private val calculator = PriceCalculator()

    @Test
    fun `10% discount on 1000 gives 900`() {
        assertThat(calculator.calculateDiscount(1000.0, 10.0))
            .isEqualTo(900.0)
    }

    @Test
    fun `0% discount gives original price`() {
        assertThat(calculator.calculateDiscount(500.0, 0.0))
            .isEqualTo(500.0)
    }

    @Test
    fun `100% discount gives 0`() {
        assertThat(calculator.calculateDiscount(500.0, 100.0))
            .isEqualTo(0.0)
    }

    @Test
    fun `negative price throws exception`() {
        assertThrows<IllegalArgumentException> {
            calculator.calculateDiscount(-100.0, 10.0)
        }
    }

    @Test
    fun `discount over 100 throws exception`() {
        assertThrows<IllegalArgumentException> {
            calculator.calculateDiscount(100.0, 101.0)
        }
    }

    @Test
    fun `7% VAT on 100 gives 107`() {
        assertThat(calculator.applyTax(100.0, 7.0))
            .isEqualTo(107.0)
    }

    @Test
    fun `total of multiple items`() {
        val items = listOf(100.0 to 2, 50.0 to 3, 200.0 to 1)
        assertThat(calculator.calculateTotal(items))
            .isEqualTo(550.0) // 200 + 150 + 200
    }

    @Test
    fun `empty list gives 0`() {
        assertThat(calculator.calculateTotal(emptyList()))
            .isEqualTo(0.0)
    }
}
```

---

## 🚀 4. Performance Testing ด้วย JMH

JMH (Java Microbenchmark Harness) สำหรับ performance benchmarking

### Dependencies

```kotlin
// build.gradle.kts
plugins {
    id("me.champeau.jmh") version "0.7.2"
}

dependencies {
    jmhImplementation("org.openjdk.jmh:jmh-core:1.37")
    jmhAnnotationProcessor("org.openjdk.jmh:jmh-generator-annprocess:1.37")
}
```

### Benchmark Class

```kotlin
import org.openjdk.jmh.annotations.*
import org.openjdk.jmh.infra.Blackhole
import java.util.concurrent.TimeUnit

@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.MICROSECONDS)
@State(Scope.Benchmark)
@Fork(2)
@Warmup(iterations = 5, time = 1)
@Measurement(iterations = 10, time = 1)
open class StringProcessingBenchmark {

    @Param("100", "1000", "10000")
    lateinit var inputSize: String

    private lateinit var data: List<String>

    @Setup
    fun setup() {
        val size = inputSize.toInt()
        data = (1..size).map { "item_$it_value_${it * 2}" }
    }

    @Benchmark
    fun filterWithForLoop(bh: Blackhole) {
        val result = mutableListOf<String>()
        for (item in data) {
            if (item.contains("5")) result.add(item)
        }
        bh.consume(result)
    }

    @Benchmark
    fun filterWithStream(bh: Blackhole) {
        val result = data.filter { it.contains("5") }
        bh.consume(result)
    }

    @Benchmark
    fun filterWithSequence(bh: Blackhole) {
        val result = data.asSequence()
            .filter { it.contains("5") }
            .toList()
        bh.consume(result)
    }
}
```

---

## 🥒 5. BDD Testing ด้วย Cucumber

### Dependencies

```kotlin
testImplementation("io.cucumber:cucumber-java:7.15.0")
testImplementation("io.cucumber:cucumber-junit-platform-engine:7.15.0")
testImplementation("io.cucumber:cucumber-spring:7.15.0")
```

### Feature File

```gherkin
# src/test/resources/features/product.feature

Feature: Product Management
  As a product manager
  I want to manage products in the catalog
  So that customers can browse and buy them

  Background:
    Given the product catalog is empty

  Scenario: Add a new product
    Given I have a product with name "Laptop" and price 25000
    When I add the product to the catalog
    Then the catalog should contain 1 product
    And the product "Laptop" should have price 25000

  Scenario: Apply discount
    Given there is a product "Phone" with price 15000 in the catalog
    When I apply 10% discount to "Phone"
    Then the price of "Phone" should be 13500

  Scenario Outline: Bulk pricing
    Given there is a product "<name>" with price <price> in the catalog
    When I buy <quantity> units
    Then the total should be <total>

    Examples:
      | name    | price | quantity | total  |
      | Laptop  | 25000 | 2        | 50000  |
      | Phone   | 15000 | 3        | 45000  |
      | Tablet  | 12000 | 5        | 60000  |
```

### Step Definitions

```kotlin
import io.cucumber.java.Before
import io.cucumber.java.en.*
import org.assertj.core.api.Assertions.*

data class CucumberProduct(val name: String, var price: Double)

class ProductStepDefinitions {
    private val catalog = mutableListOf<CucumberProduct>()
    private var lastBoughtQuantity = 0
    private var lastProduct: CucumberProduct? = null

    @Before
    fun setup() {
        catalog.clear()
    }

    @Given("the product catalog is empty")
    fun theCatalogIsEmpty() {
        catalog.clear()
    }

    @Given("I have a product with name {string} and price {int}")
    fun iHaveAProduct(name: String, price: Int) {
        lastProduct = CucumberProduct(name, price.toDouble())
    }

    @Given("there is a product {string} with price {int} in the catalog")
    fun thereIsAProduct(name: String, price: Int) {
        catalog.add(CucumberProduct(name, price.toDouble()))
    }

    @When("I add the product to the catalog")
    fun iAddTheProduct() {
        lastProduct?.let { catalog.add(it) }
    }

    @When("I apply {int}% discount to {string}")
    fun iApplyDiscount(percent: Int, name: String) {
        catalog.find { it.name == name }?.let { product ->
            product.price = product.price * (1 - percent.toDouble() / 100)
        }
    }

    @When("I buy {int} units")
    fun iBuy(quantity: Int) {
        lastBoughtQuantity = quantity
    }

    @Then("the catalog should contain {int} product")
    fun theCatalogShouldContain(count: Int) {
        assertThat(catalog.size).isEqualTo(count)
    }

    @Then("the product {string} should have price {int}")
    fun theProductShouldHavePrice(name: String, price: Int) {
        val product = catalog.find { it.name == name }
        assertThat(product).isNotNull
        assertThat(product!!.price).isEqualTo(price.toDouble())
    }

    @Then("the price of {string} should be {int}")
    fun thePriceShouldBe(name: String, price: Int) {
        val product = catalog.find { it.name == name }
        assertThat(product!!.price).isEqualTo(price.toDouble())
    }

    @Then("the total should be {int}")
    fun theTotalShouldBe(total: Int) {
        val product = lastProduct ?: catalog.lastOrNull()
        assertThat(product!!.price * lastBoughtQuantity).isEqualTo(total.toDouble())
    }
}
```

---

## 🏭 6. Test Fixtures และ Factories

```kotlin
// TestFixtures.kt
import java.time.LocalDateTime

object TestFixtures {
    
    fun user(
        id: Long = 1L,
        name: String = "Test User",
        email: String = "test@example.com",
        role: String = "USER"
    ) = User(id = id, name = name, email = email, role = role)

    fun adminUser() = user(role = "ADMIN")

    fun product(
        id: Long = 1L,
        name: String = "Test Product",
        price: Double = 100.0,
        stock: Int = 10
    ) = Product(id = id, name = name, price = price, stock = stock)

    fun order(
        id: Long = 1L,
        user: User = user(),
        items: List<OrderItem> = listOf(orderItem())
    ) = Order(id = id, user = user, items = items, createdAt = LocalDateTime.now())

    fun orderItem(
        product: Product = product(),
        quantity: Int = 1
    ) = OrderItem(product = product, quantity = quantity)
}

// ใช้ใน tests
class OrderServiceTest {
    @Test
    fun `order total is calculated correctly`() {
        val order = TestFixtures.order(
            items = listOf(
                TestFixtures.orderItem(
                    product = TestFixtures.product(price = 100.0),
                    quantity = 3
                )
            )
        )
        assertThat(order.total()).isEqualTo(300.0)
    }
}
```

---

## 📊 7. สรุปตาราง Testing Approaches

| วิธีการ | เครื่องมือ | จุดประสงค์ | เมื่อใช้ |
|--------|-----------|-----------|---------|
| Unit Test | JUnit 5, Kotest | ทดสอบ logic หน่วยย่อย | ทุก function |
| Property-based | Kotest Property | ทดสอบ properties ทั่วไป | Algorithm, math |
| Mutation | PITest | วัดคุณภาพ tests | CI pipeline |
| Performance | JMH | วัด throughput/latency | Critical paths |
| BDD | Cucumber | Behavior specification | Feature teams |
| Integration | Spring Test | ทดสอบ component interaction | Service layer |

---

## 💡 Best Practices

1. **ใช้ property-based testing** สำหรับ pure functions ที่มี mathematical properties
2. **Mutation score > 80%** ถือว่า test suite มีคุณภาพดี
3. **JMH Warmup** สำคัญมาก — JVM ต้องการเวลา JIT compile
4. **BDD features** เขียนโดย business team, step definitions โดย developers
5. **Test factories** ลด duplication และ maintain ง่ายกว่า

---

*Part 69/100+ | Kotlin & Spring Boot Complete Course*
