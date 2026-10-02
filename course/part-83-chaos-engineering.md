# Part 83: Chaos Engineering
## ทดสอบความทนทานของระบบด้วย Chaos Monkey

---

## 🎯 เป้าหมายของ Part นี้

- เข้าใจหลักการ Chaos Engineering
- ใช้ Chaos Monkey สำหรับ Spring Boot
- Latency และ Error Injection
- ทดสอบ Circuit Breaker
- สร้าง Resilience Testing Suite

---

## 📖 1. Chaos Engineering คืออะไร?

Chaos Engineering คือการ **จงใจทำให้ระบบล้มเหลว** เพื่อค้นหาจุดอ่อนก่อนที่จะเกิดปัญหาจริงใน production

> "If it hurts, do it more often" - Netflix Engineering

### หลักการ Chaos Engineering

```
Principles of Chaos:
1. Build hypothesis around steady-state behavior
2. Vary real-world events (failures, spikes, etc.)
3. Run experiments in production
4. Automate experiments to run continuously
5. Minimize blast radius
```

### Chaos Monkey History

```
2010: Netflix สร้าง Chaos Monkey เพื่อทดสอบ AWS
2011: เปิดเป็น open source
2012: "Simian Army" - ครอบครัวของ chaos tools
2019: Chaos Monkey for Spring Boot
```

---

## 📦 2. ติดตั้ง Chaos Monkey for Spring Boot

### build.gradle.kts

```kotlin
plugins {
    kotlin("jvm") version "1.9.21"
    kotlin("plugin.spring") version "1.9.21"
    id("org.springframework.boot") version "3.2.1"
    id("io.spring.dependency-management") version "1.1.4"
}

dependencies {
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-actuator")
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
    implementation("com.fasterxml.jackson.module:jackson-module-kotlin")
    implementation("org.jetbrains.kotlin:kotlin-reflect")

    // Chaos Monkey
    implementation("de.codecentric:chaos-monkey-spring-boot:3.1.0")

    // Resilience4j สำหรับ Circuit Breaker
    implementation("io.github.resilience4j:resilience4j-spring-boot3:2.1.0")
    implementation("org.springframework.boot:spring-boot-starter-aop")

    runtimeOnly("com.h2database:h2")
    testImplementation("org.springframework.boot:spring-boot-starter-test")
}
```

---

## ⚙️ 3. Configuration

### application.yml

```yaml
# src/main/resources/application.yml
spring:
  application:
    name: chaos-demo

# Chaos Monkey Configuration
chaos:
  monkey:
    enabled: true
    assaults:
      level: 3                    # 1-10 (สูงขึ้น = โดนบ่อยขึ้น)
      latency-active: false       # เปิด latency injection
      latency-range-start: 1000   # 1000ms minimum
      latency-range-end: 3000     # 3000ms maximum
      exceptions-active: false    # เปิด exception injection
      kill-application-active: false  # เปิด app kill (ระวัง!)
      memory-active: false        # เปิด memory attack
    watcher:
      controller: true
      restController: true
      service: true
      repository: false
      component: false

management:
  endpoints:
    web:
      exposure:
        include: "*"
  endpoint:
    chaosmonkey:
      enabled: true

# Resilience4j
resilience4j:
  circuitbreaker:
    instances:
      orderService:
        registerHealthIndicator: true
        slidingWindowSize: 10
        permittedNumberOfCallsInHalfOpenState: 3
        slidingWindowType: COUNT_BASED
        minimumNumberOfCalls: 5
        waitDurationInOpenState: 5s
        failureRateThreshold: 50
        eventConsumerBufferSize: 10
  timelimiter:
    instances:
      orderService:
        timeoutDuration: 3s
```

---

## 🏗️ 4. Service ที่จะทดสอบ

```kotlin
// src/main/kotlin/com/example/entity/Order.kt
package com.example.entity

import jakarta.persistence.*

@Entity
@Table(name = "orders")
data class Order(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long = 0,
    val productId: String,
    val quantity: Int,
    val status: String = "PENDING",
    val userId: String
)
```

```kotlin
// src/main/kotlin/com/example/service/OrderService.kt
package com.example.service

import com.example.entity.Order
import com.example.repository.OrderRepository
import io.github.resilience4j.circuitbreaker.annotation.CircuitBreaker
import io.github.resilience4j.timelimiter.annotation.TimeLimiter
import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional
import java.util.concurrent.CompletableFuture

@Service
@Transactional
class OrderService(
    private val orderRepository: OrderRepository,
    private val inventoryService: InventoryService,
    private val notificationService: NotificationService
) {

    @CircuitBreaker(name = "orderService", fallbackMethod = "createOrderFallback")
    fun createOrder(order: Order): Order {
        // Check inventory (อาจ fail ถ้า Chaos Monkey โจมตี)
        val available = inventoryService.checkAvailability(order.productId, order.quantity)

        if (!available) {
            throw IllegalStateException("Product ${order.productId} not available")
        }

        val savedOrder = orderRepository.save(order.copy(status = "CONFIRMED"))

        // Send notification (อาจ fail)
        notificationService.sendOrderConfirmation(savedOrder)

        return savedOrder
    }

    // Fallback เมื่อ Circuit Breaker เปิด
    @Suppress("unused")
    private fun createOrderFallback(order: Order, ex: Exception): Order {
        println("Circuit breaker opened! Falling back for order: $order")
        println("Reason: ${ex.message}")

        // บันทึก order ใน pending state
        return orderRepository.save(order.copy(status = "PENDING_RETRY"))
    }

    fun getAllOrders(): List<Order> = orderRepository.findAll()

    fun getOrderById(id: Long): Order =
        orderRepository.findById(id)
            .orElseThrow { NoSuchElementException("Order $id not found") }
}
```

```kotlin
// src/main/kotlin/com/example/service/InventoryService.kt
package com.example.service

import org.springframework.stereotype.Service

@Service
class InventoryService {
    // Chaos Monkey จะโจมตี service นี้
    fun checkAvailability(productId: String, quantity: Int): Boolean {
        println("Checking inventory for $productId x$quantity")
        // Simulate real inventory check
        Thread.sleep(100) // small delay
        return true
    }
}
```

```kotlin
// src/main/kotlin/com/example/service/NotificationService.kt
package com.example.service

import org.springframework.stereotype.Service

@Service
class NotificationService {
    fun sendOrderConfirmation(order: Any) {
        println("Sending confirmation for order")
        // Simulate notification
    }
}
```

---

## 💥 5. ทดสอบด้วย Chaos Monkey API

### เปิด/ปิด Chaos Monkey

```bash
# เปิด Chaos Monkey
curl -X POST http://localhost:8080/actuator/chaosmonkey/enable

# ปิด Chaos Monkey
curl -X POST http://localhost:8080/actuator/chaosmonkey/disable

# ดู status
curl http://localhost:8080/actuator/chaosmonkey
```

### ตั้งค่า Latency Injection

```bash
# ฉีด latency 2-5 วินาที
curl -X POST http://localhost:8080/actuator/chaosmonkey/assaults \
  -H "Content-Type: application/json" \
  -d '{
    "level": 5,
    "latencyActive": true,
    "latencyRangeStart": 2000,
    "latencyRangeEnd": 5000,
    "exceptionsActive": false
  }'
```

### ตั้งค่า Exception Injection

```bash
# ฉีด RuntimeException
curl -X POST http://localhost:8080/actuator/chaosmonkey/assaults \
  -H "Content-Type: application/json" \
  -d '{
    "level": 3,
    "latencyActive": false,
    "exceptionsActive": true,
    "exception": {
      "type": "java.lang.RuntimeException",
      "arguments": [{
        "className": "java.lang.String",
        "value": "Chaos Monkey: intentional failure!"
      }]
    }
  }'
```

### ตั้งค่า Watcher

```bash
# ดู watcher configuration
curl http://localhost:8080/actuator/chaosmonkey/watchers

# เปลี่ยน watcher
curl -X POST http://localhost:8080/actuator/chaosmonkey/watchers \
  -H "Content-Type: application/json" \
  -d '{
    "controller": true,
    "restController": true,
    "service": true,
    "repository": true,
    "component": false
  }'
```

---

## 🧪 6. Resilience Testing Script

```kotlin
// src/test/kotlin/com/example/chaos/ResilienceTest.kt
package com.example.chaos

import com.example.entity.Order
import org.junit.jupiter.api.Test
import org.springframework.beans.factory.annotation.Autowired
import org.springframework.boot.test.context.SpringBootTest
import org.springframework.boot.test.web.client.TestRestTemplate
import org.springframework.boot.test.web.server.LocalServerPort
import org.springframework.http.HttpEntity
import org.springframework.http.HttpHeaders
import org.springframework.http.MediaType
import java.util.concurrent.CountDownLatch
import java.util.concurrent.Executors
import java.util.concurrent.atomic.AtomicInteger

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class ResilienceTest {

    @LocalServerPort
    private var port: Int = 0

    @Autowired
    private lateinit var restTemplate: TestRestTemplate

    @Test
    fun `should handle concurrent requests gracefully`() {
        val numRequests = 100
        val successCount = AtomicInteger(0)
        val failureCount = AtomicInteger(0)
        val latch = CountDownLatch(numRequests)
        val executor = Executors.newFixedThreadPool(20)

        repeat(numRequests) { i ->
            executor.submit {
                try {
                    val headers = HttpHeaders()
                    headers.contentType = MediaType.APPLICATION_JSON
                    val body = """{"productId":"PROD001","quantity":1,"userId":"user$i"}"""
                    val entity = HttpEntity(body, headers)

                    val response = restTemplate.postForEntity(
                        "http://localhost:$port/api/orders",
                        entity,
                        String::class.java
                    )

                    if (response.statusCode.is2xxSuccessful) {
                        successCount.incrementAndGet()
                    } else {
                        failureCount.incrementAndGet()
                    }
                } catch (e: Exception) {
                    failureCount.incrementAndGet()
                } finally {
                    latch.countDown()
                }
            }
        }

        latch.await()
        executor.shutdown()

        println("Total requests: $numRequests")
        println("Success: ${successCount.get()}")
        println("Failure: ${failureCount.get()}")
        println("Success rate: ${successCount.get() * 100.0 / numRequests}%")

        // ต้องมี success rate อย่างน้อย 80% แม้ในสภาพ chaos
        assert(successCount.get() >= numRequests * 0.8) {
            "Success rate too low: ${successCount.get()}/$numRequests"
        }
    }
}
```

---

## 📊 7. Circuit Breaker Testing

```kotlin
// src/test/kotlin/com/example/chaos/CircuitBreakerTest.kt
package com.example.chaos

import io.github.resilience4j.circuitbreaker.CircuitBreaker
import io.github.resilience4j.circuitbreaker.CircuitBreakerRegistry
import org.junit.jupiter.api.Test
import org.junit.jupiter.api.BeforeEach
import org.springframework.beans.factory.annotation.Autowired
import org.springframework.boot.test.context.SpringBootTest
import kotlin.test.assertEquals

@SpringBootTest
class CircuitBreakerTest {

    @Autowired
    private lateinit var circuitBreakerRegistry: CircuitBreakerRegistry

    @Autowired
    private lateinit var orderService: com.example.service.OrderService

    private lateinit var circuitBreaker: CircuitBreaker

    @BeforeEach
    fun setup() {
        circuitBreaker = circuitBreakerRegistry.circuitBreaker("orderService")
        circuitBreaker.reset() // Reset state ก่อนทดสอบ
    }

    @Test
    fun `circuit breaker should open after failures`() {
        println("Initial state: ${circuitBreaker.state}")
        assertEquals(CircuitBreaker.State.CLOSED, circuitBreaker.state)

        // Simulate failures
        repeat(6) { i ->
            try {
                // Force failure
                circuitBreaker.decorateSupplier {
                    throw RuntimeException("Simulated failure $i")
                }.get()
            } catch (e: Exception) {
                println("Failure $i: ${e.message}")
            }
        }

        println("After failures state: ${circuitBreaker.state}")

        // Circuit should be OPEN now
        assertEquals(CircuitBreaker.State.OPEN, circuitBreaker.state)

        // Metrics
        val metrics = circuitBreaker.metrics
        println("Failed calls: ${metrics.numberOfFailedCalls}")
        println("Failure rate: ${metrics.failureRate}%")
    }
}
```

---

## 🔍 8. Monitoring Chaos Testing

```kotlin
// src/main/kotlin/com/example/monitoring/ChaosMetricsListener.kt
package com.example.monitoring

import io.github.resilience4j.circuitbreaker.CircuitBreaker
import io.github.resilience4j.circuitbreaker.CircuitBreakerRegistry
import io.micrometer.core.instrument.MeterRegistry
import org.springframework.boot.actuate.endpoint.annotation.Endpoint
import org.springframework.boot.actuate.endpoint.annotation.ReadOperation
import org.springframework.stereotype.Component

@Component
@Endpoint(id = "resilience")
class ResilienceEndpoint(
    private val circuitBreakerRegistry: CircuitBreakerRegistry
) {
    @ReadOperation
    fun getResilienceMetrics(): Map<String, Any> {
        return circuitBreakerRegistry.allCircuitBreakers
            .associate { cb ->
                cb.name to mapOf(
                    "state" to cb.state.name,
                    "failureRate" to cb.metrics.failureRate,
                    "successRate" to (100f - cb.metrics.failureRate),
                    "numberOfSuccessfulCalls" to cb.metrics.numberOfSuccessfulCalls,
                    "numberOfFailedCalls" to cb.metrics.numberOfFailedCalls,
                    "numberOfNotPermittedCalls" to cb.metrics.numberOfNotPermittedCalls
                )
            }
    }
}
```

```bash
# ดู resilience metrics
curl http://localhost:8080/actuator/resilience

# Output:
# {
#   "orderService": {
#     "state": "CLOSED",
#     "failureRate": 0.0,
#     "successRate": 100.0,
#     "numberOfSuccessfulCalls": 45,
#     "numberOfFailedCalls": 0,
#     "numberOfNotPermittedCalls": 0
#   }
# }
```

---

## 📋 สรุป

| หัวข้อ | รายละเอียด |
|--------|-----------|
| Chaos Engineering | จงใจทำให้ระบบล้มเหลวเพื่อค้นหาจุดอ่อน |
| Chaos Monkey | Library inject failures ใน Spring Boot |
| Latency Injection | เพิ่ม delay เพื่อทดสอบ timeout handling |
| Exception Injection | throw exception เพื่อทดสอบ error handling |
| Circuit Breaker | ป้องกัน cascade failure |
| Fallback | คืนค่า default เมื่อ service ล้มเหลว |

---

*Part 83/100+ | Kotlin & Spring Boot Complete Course*
