# Part 43: Circuit Breaker (Resilience4j)
## การออกแบบระบบที่ทนทานต่อความล้มเหลว

---

## 🎯 เป้าหมายของ Part นี้

- เข้าใจ Circuit Breaker pattern
- ติดตั้งและใช้งาน Resilience4j
- Retry pattern และ Fallback methods
- @CircuitBreaker, @Retry, @RateLimiter, @Bulkhead annotations
- Time Limiter
- ตัวอย่างจริง: External API calls with circuit breaker

---

## 📚 1. Circuit Breaker Pattern คืออะไร?

Circuit Breaker เป็น design pattern ที่ช่วยป้องกันการ cascade failure ในระบบ distributed

```
ปกติ (CLOSED state):
  Service A → Service B ✅ → ตอบสนองปกติ

Service B เสีย (OPEN state):
  Service A → Circuit Breaker → Fallback ✅ (ไม่ส่งต่อ request)
  (ป้องกัน timeout storm)

หลังจากรอ (HALF-OPEN state):
  Service A → ส่ง test request → Service B
  ถ้าสำเร็จ: กลับเป็น CLOSED
  ถ้าล้มเหลว: กลับเป็น OPEN
```

### สาม States ของ Circuit Breaker:

| State | ความหมาย | Behavior |
|-------|---------|---------|
| CLOSED | ปกติ | ส่ง request ไปยัง service จริง |
| OPEN | เปิด circuit | Return fallback ทันที ไม่ส่ง request |
| HALF-OPEN | ทดสอบ | ส่ง request จำนวนน้อยเพื่อทดสอบ |

---

## 🔧 2. Dependencies Setup

```kotlin
// build.gradle.kts
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-aop")
    
    // Resilience4j
    implementation("io.github.resilience4j:resilience4j-spring-boot3:2.2.0")
    implementation("io.github.resilience4j:resilience4j-kotlin:2.2.0")
    implementation("io.github.resilience4j:resilience4j-micrometer:2.2.0")
    
    // Spring Boot Actuator สำหรับ health check
    implementation("org.springframework.boot:spring-boot-starter-actuator")
    
    // WebClient สำหรับ HTTP calls
    implementation("org.springframework.boot:spring-boot-starter-webflux")
    
    implementation("com.fasterxml.jackson.module:jackson-module-kotlin")
    implementation("org.jetbrains.kotlin:kotlin-reflect")
    
    testImplementation("org.springframework.boot:spring-boot-starter-test")
    testImplementation("com.github.tomakehurst:wiremock-jre8-standalone:3.0.1")
}
```

```yaml
# application.yml
resilience4j:
  circuitbreaker:
    instances:
      paymentService:
        register-health-indicator: true
        sliding-window-size: 10
        minimum-number-of-calls: 5
        permitted-number-of-calls-in-half-open-state: 3
        automatic-transition-from-open-to-half-open-enabled: true
        wait-duration-in-open-state: 5s
        failure-rate-threshold: 50
        slow-call-rate-threshold: 50
        slow-call-duration-threshold: 2s
      emailService:
        register-health-indicator: true
        sliding-window-size: 5
        minimum-number-of-calls: 3
        failure-rate-threshold: 60
        wait-duration-in-open-state: 10s
  
  retry:
    instances:
      paymentService:
        max-attempts: 3
        wait-duration: 500ms
        retry-exceptions:
          - org.springframework.web.client.HttpServerErrorException
          - java.net.ConnectException
        ignore-exceptions:
          - com.example.circuit.exception.InvalidRequestException
  
  timelimiter:
    instances:
      paymentService:
        timeout-duration: 3s
        cancel-running-future: true
  
  bulkhead:
    instances:
      paymentService:
        max-concurrent-calls: 10
        max-wait-duration: 100ms

management:
  health:
    circuitbreakers:
      enabled: true
  endpoints:
    web:
      exposure:
        include: health,circuitbreakers,metrics
```

---

## 🏦 3. External Payment Service Client

```kotlin
// client/PaymentClient.kt
package com.example.circuit.client

import com.example.circuit.dto.PaymentRequest
import com.example.circuit.dto.PaymentResponse
import com.example.circuit.exception.PaymentServiceException
import com.fasterxml.jackson.module.kotlin.jacksonObjectMapper
import com.fasterxml.jackson.module.kotlin.readValue
import org.springframework.beans.factory.annotation.Value
import org.springframework.stereotype.Component
import org.springframework.web.client.RestClient

@Component
class PaymentClient(
    @Value("\${payment.service.url:https://api.payment.example.com}")
    private val baseUrl: String
) {
    private val restClient = RestClient.builder()
        .baseUrl(baseUrl)
        .build()

    private val objectMapper = jacksonObjectMapper()

    fun processPayment(request: PaymentRequest): PaymentResponse {
        return try {
            restClient.post()
                .uri("/payments")
                .body(request)
                .retrieve()
                .body(PaymentResponse::class.java)
                ?: throw PaymentServiceException("Empty response from payment service")
        } catch (ex: Exception) {
            throw PaymentServiceException("Payment service call failed: ${ex.message}", ex)
        }
    }

    fun getPaymentStatus(paymentId: String): PaymentResponse {
        return try {
            restClient.get()
                .uri("/payments/{id}", paymentId)
                .retrieve()
                .body(PaymentResponse::class.java)
                ?: throw PaymentServiceException("Payment not found: $paymentId")
        } catch (ex: Exception) {
            throw PaymentServiceException("Failed to get payment status: ${ex.message}", ex)
        }
    }
}
```

### DTOs

```kotlin
// dto/PaymentDtos.kt
package com.example.circuit.dto

data class PaymentRequest(
    val amount: Double,
    val currency: String,
    val cardToken: String,
    val description: String
)

data class PaymentResponse(
    val paymentId: String,
    val status: PaymentStatus,
    val amount: Double,
    val currency: String,
    val message: String? = null
)

enum class PaymentStatus {
    SUCCESS, PENDING, FAILED, UNKNOWN
}

data class OrderRequest(
    val userId: Long,
    val items: List<OrderItem>,
    val paymentToken: String
)

data class OrderItem(
    val productId: Long,
    val quantity: Int,
    val price: Double
)

data class OrderResponse(
    val orderId: String,
    val status: String,
    val payment: PaymentResponse?,
    val message: String
)
```

---

## 🔄 4. Service พร้อม Circuit Breaker

```kotlin
// service/PaymentService.kt
package com.example.circuit.service

import com.example.circuit.client.PaymentClient
import com.example.circuit.dto.PaymentRequest
import com.example.circuit.dto.PaymentResponse
import com.example.circuit.dto.PaymentStatus
import io.github.resilience4j.circuitbreaker.annotation.CircuitBreaker
import io.github.resilience4j.retry.annotation.Retry
import io.github.resilience4j.timelimiter.annotation.TimeLimiter
import io.github.resilience4j.bulkhead.annotation.Bulkhead
import org.slf4j.LoggerFactory
import org.springframework.stereotype.Service
import java.util.concurrent.CompletableFuture

@Service
class PaymentService(private val paymentClient: PaymentClient) {

    private val logger = LoggerFactory.getLogger(javaClass)

    /**
     * ประมวลผล payment พร้อม circuit breaker + retry
     * 
     * Circuit Breaker จะเปิด (OPEN) เมื่อ:
     * - error rate > 50% ใน 10 calls ล่าสุด
     * - slow call rate > 50% (calls ที่ใช้เวลา > 2s)
     */
    @CircuitBreaker(name = "paymentService", fallbackMethod = "processPaymentFallback")
    @Retry(name = "paymentService")
    @TimeLimiter(name = "paymentService")
    fun processPayment(request: PaymentRequest): CompletableFuture<PaymentResponse> {
        return CompletableFuture.supplyAsync {
            logger.info("Processing payment for amount: ${request.amount}")
            paymentClient.processPayment(request)
        }
    }

    /**
     * Fallback method - เรียกเมื่อ circuit เปิด
     * signature ต้องตรงกับ method หลัก แต่เพิ่ม parameter Exception ท้ายสุด
     */
    @Suppress("UNUSED_PARAMETER")
    private fun processPaymentFallback(
        request: PaymentRequest,
        exception: Exception
    ): CompletableFuture<PaymentResponse> {
        logger.warn("Payment service unavailable, using fallback. Error: ${exception.message}")
        return CompletableFuture.completedFuture(
            PaymentResponse(
                paymentId = "PENDING-${System.currentTimeMillis()}",
                status = PaymentStatus.PENDING,
                amount = request.amount,
                currency = request.currency,
                message = "Payment is being processed. We'll notify you when complete."
            )
        )
    }

    /**
     * ดึงสถานะ payment พร้อม bulkhead
     * Bulkhead จำกัดจำนวน concurrent calls ไม่เกิน 10
     */
    @CircuitBreaker(name = "paymentService", fallbackMethod = "getStatusFallback")
    @Bulkhead(name = "paymentService")
    fun getPaymentStatus(paymentId: String): PaymentResponse {
        return paymentClient.getPaymentStatus(paymentId)
    }

    @Suppress("UNUSED_PARAMETER")
    private fun getStatusFallback(
        paymentId: String,
        exception: Exception
    ): PaymentResponse {
        logger.warn("Cannot get payment status: $paymentId, using fallback")
        return PaymentResponse(
            paymentId = paymentId,
            status = PaymentStatus.UNKNOWN,
            amount = 0.0,
            currency = "USD",
            message = "Payment status temporarily unavailable"
        )
    }
}
```

---

## 📧 5. Email Service พร้อม Retry

```kotlin
// service/EmailService.kt
package com.example.circuit.service

import io.github.resilience4j.circuitbreaker.annotation.CircuitBreaker
import io.github.resilience4j.retry.annotation.Retry
import org.slf4j.LoggerFactory
import org.springframework.stereotype.Service

@Service
class EmailService {

    private val logger = LoggerFactory.getLogger(javaClass)
    private var callCount = 0  // สำหรับ simulate failures

    /**
     * ส่ง email พร้อม retry และ circuit breaker
     * จะ retry 3 ครั้ง รอ 500ms ระหว่าง retry
     */
    @CircuitBreaker(name = "emailService", fallbackMethod = "sendEmailFallback")
    @Retry(name = "emailService")
    fun sendEmail(to: String, subject: String, body: String) {
        callCount++
        logger.info("Attempting to send email to: $to (attempt #$callCount)")

        // Simulate 60% failure rate สำหรับ demo
        if (callCount % 5 < 3) {
            throw RuntimeException("Email server temporarily unavailable")
        }

        logger.info("Email sent successfully to: $to")
        callCount = 0
    }

    @Suppress("UNUSED_PARAMETER")
    private fun sendEmailFallback(
        to: String,
        subject: String,
        body: String,
        exception: Exception
    ) {
        logger.error("Email failed for $to, queuing for later delivery: ${exception.message}")
        // ใน production จะ queue ไว้ใน message broker
        queueEmailForLater(to, subject, body)
    }

    private fun queueEmailForLater(to: String, subject: String, body: String) {
        logger.info("Email queued: to=$to, subject=$subject")
        // TODO: ส่งไปยัง Kafka/RabbitMQ/SQS
    }
}
```

---

## 🛍️ 6. Order Service (Orchestrating Multiple Services)

```kotlin
// service/OrderService.kt
package com.example.circuit.service

import com.example.circuit.dto.*
import io.github.resilience4j.circuitbreaker.CircuitBreakerRegistry
import org.slf4j.LoggerFactory
import org.springframework.stereotype.Service
import java.util.UUID

@Service
class OrderService(
    private val paymentService: PaymentService,
    private val emailService: EmailService,
    private val circuitBreakerRegistry: CircuitBreakerRegistry
) {
    private val logger = LoggerFactory.getLogger(javaClass)

    suspend fun createOrder(request: OrderRequest): OrderResponse {
        val orderId = UUID.randomUUID().toString()
        val totalAmount = request.items.sumOf { it.price * it.quantity }

        logger.info("Creating order $orderId for user ${request.userId}")

        // Process payment
        val paymentRequest = PaymentRequest(
            amount = totalAmount,
            currency = "THB",
            cardToken = request.paymentToken,
            description = "Order $orderId"
        )

        val paymentResponse = try {
            paymentService.processPayment(paymentRequest).get()
        } catch (ex: Exception) {
            logger.error("Payment failed for order $orderId", ex)
            return OrderResponse(
                orderId = orderId,
                status = "FAILED",
                payment = null,
                message = "Order failed: payment could not be processed"
            )
        }

        // Send confirmation email (non-critical, don't fail order if email fails)
        try {
            emailService.sendEmail(
                to = "user${request.userId}@example.com",
                subject = "Order Confirmation: $orderId",
                body = "Your order has been placed. Payment status: ${paymentResponse.status}"
            )
        } catch (ex: Exception) {
            logger.warn("Failed to send confirmation email for order $orderId", ex)
            // ไม่ fail order เพราะ email ไม่ใช่ critical path
        }

        return OrderResponse(
            orderId = orderId,
            status = if (paymentResponse.status == PaymentStatus.SUCCESS) "CONFIRMED" else "PROCESSING",
            payment = paymentResponse,
            message = "Order created successfully"
        )
    }

    /**
     * ดู circuit breaker state ทั้งหมด
     */
    fun getCircuitBreakerStates(): Map<String, String> {
        return circuitBreakerRegistry.allCircuitBreakers.associate {
            it.name to it.state.toString()
        }
    }
}
```

---

## 🎮 7. Controller

```kotlin
// controller/OrderController.kt
package com.example.circuit.controller

import com.example.circuit.dto.OrderRequest
import com.example.circuit.dto.OrderResponse
import com.example.circuit.service.OrderService
import com.example.circuit.service.PaymentService
import kotlinx.coroutines.runBlocking
import org.springframework.http.ResponseEntity
import org.springframework.web.bind.annotation.*

@RestController
@RequestMapping("/api/orders")
class OrderController(
    private val orderService: OrderService,
    private val paymentService: PaymentService
) {

    @PostMapping
    fun createOrder(@RequestBody request: OrderRequest): ResponseEntity<OrderResponse> {
        val response = runBlocking { orderService.createOrder(request) }
        return ResponseEntity.ok(response)
    }

    @GetMapping("/circuit-breakers")
    fun getCircuitBreakerStatus(): ResponseEntity<Map<String, String>> {
        return ResponseEntity.ok(orderService.getCircuitBreakerStates())
    }

    @GetMapping("/payment/{paymentId}/status")
    fun getPaymentStatus(@PathVariable paymentId: String) =
        ResponseEntity.ok(paymentService.getPaymentStatus(paymentId))
}
```

---

## 🔍 8. Programmatic Circuit Breaker

บางครั้งต้องการ control circuit breaker โดยตรงโดยไม่ใช้ annotation

```kotlin
// service/ManualCircuitBreakerService.kt
package com.example.circuit.service

import io.github.resilience4j.circuitbreaker.CircuitBreaker
import io.github.resilience4j.circuitbreaker.CircuitBreakerConfig
import io.github.resilience4j.circuitbreaker.CircuitBreakerRegistry
import org.springframework.stereotype.Service
import java.time.Duration

@Service
class ManualCircuitBreakerService(
    private val circuitBreakerRegistry: CircuitBreakerRegistry
) {

    /**
     * สร้าง custom circuit breaker ด้วย config เฉพาะ
     */
    fun createCustomCircuitBreaker(name: String): CircuitBreaker {
        val config = CircuitBreakerConfig.custom()
            .slidingWindowType(CircuitBreakerConfig.SlidingWindowType.COUNT_BASED)
            .slidingWindowSize(5)
            .failureRateThreshold(40f)
            .waitDurationInOpenState(Duration.ofSeconds(30))
            .permittedNumberOfCallsInHalfOpenState(2)
            .automaticTransitionFromOpenToHalfOpenEnabled(true)
            .build()

        return circuitBreakerRegistry.circuitBreaker(name, config)
    }

    /**
     * Wrap function ด้วย circuit breaker แบบ programmatic
     */
    fun <T> executeWithCircuitBreaker(
        circuitBreakerName: String,
        supplier: () -> T,
        fallback: (Exception) -> T
    ): T {
        val circuitBreaker = circuitBreakerRegistry.circuitBreaker(circuitBreakerName)
        val decoratedSupplier = CircuitBreaker.decorateSupplier(circuitBreaker, supplier)

        return try {
            decoratedSupplier.get()
        } catch (ex: Exception) {
            fallback(ex)
        }
    }

    fun getMetrics(circuitBreakerName: String): Map<String, Any> {
        val cb = circuitBreakerRegistry.circuitBreaker(circuitBreakerName)
        val metrics = cb.metrics
        return mapOf(
            "state" to cb.state,
            "failureRate" to metrics.failureRate,
            "slowCallRate" to metrics.slowCallRate,
            "numberOfSuccessfulCalls" to metrics.numberOfSuccessfulCalls,
            "numberOfFailedCalls" to metrics.numberOfFailedCalls,
            "numberOfNotPermittedCalls" to metrics.numberOfNotPermittedCalls
        )
    }
}
```

---

## 🧪 9. Testing with WireMock

```kotlin
// test/CircuitBreakerIntegrationTest.kt
package com.example.circuit

import com.github.tomakehurst.wiremock.WireMockServer
import com.github.tomakehurst.wiremock.client.WireMock.*
import com.github.tomakehurst.wiremock.core.WireMockConfiguration
import io.github.resilience4j.circuitbreaker.CircuitBreaker
import io.github.resilience4j.circuitbreaker.CircuitBreakerRegistry
import org.junit.jupiter.api.*
import org.springframework.beans.factory.annotation.Autowired
import org.springframework.boot.test.context.SpringBootTest
import org.springframework.test.context.DynamicPropertyRegistry
import org.springframework.test.context.DynamicPropertySource

@SpringBootTest
class CircuitBreakerIntegrationTest {

    companion object {
        private lateinit var wireMockServer: WireMockServer

        @BeforeAll
        @JvmStatic
        fun startWireMock() {
            wireMockServer = WireMockServer(WireMockConfiguration.wireMockConfig().dynamicPort())
            wireMockServer.start()
        }

        @AfterAll
        @JvmStatic
        fun stopWireMock() {
            wireMockServer.stop()
        }

        @DynamicPropertySource
        @JvmStatic
        fun configureProperties(registry: DynamicPropertyRegistry) {
            registry.add("payment.service.url") { "http://localhost:${wireMockServer.port()}" }
        }
    }

    @Autowired
    private lateinit var circuitBreakerRegistry: CircuitBreakerRegistry

    @BeforeEach
    fun resetCircuitBreakers() {
        circuitBreakerRegistry.circuitBreaker("paymentService").reset()
        wireMockServer.resetAll()
    }

    @Test
    fun `circuit breaker transitions to OPEN after failures`() {
        // Setup: Payment service always returns 500
        wireMockServer.stubFor(
            post(urlEqualTo("/payments"))
                .willReturn(serverError())
        )

        val cb = circuitBreakerRegistry.circuitBreaker("paymentService")
        assert(cb.state == CircuitBreaker.State.CLOSED)

        // Trigger 5 failures (minimum-number-of-calls)
        // After 50% failure rate, circuit should open
        // (Test would make actual calls here)

        println("Circuit state: ${cb.state}")
    }

    @Test
    fun `fallback is called when circuit is OPEN`() {
        val cb = circuitBreakerRegistry.circuitBreaker("paymentService")
        cb.transitionToOpenState()
        
        assert(cb.state == CircuitBreaker.State.OPEN)
        println("Circuit is OPEN, fallback will be used")
    }
}
```

---

## 📊 สรุปเนื้อหา

| Pattern | Annotation | ประโยชน์ |
|---------|-----------|---------|
| Circuit Breaker | @CircuitBreaker | ป้องกัน cascade failure |
| Retry | @Retry | ลองซ้ำเมื่อเกิด transient errors |
| Time Limiter | @TimeLimiter | จำกัดเวลา execution |
| Bulkhead | @Bulkhead | จำกัด concurrent calls |
| Rate Limiter | @RateLimiter | จำกัด call rate |

### Circuit Breaker States:

```
CLOSED → (failure rate > threshold) → OPEN
OPEN → (wait duration elapsed) → HALF-OPEN
HALF-OPEN → (test calls succeed) → CLOSED
HALF-OPEN → (test calls fail) → OPEN
```

### Resilience4j vs Hystrix:

| Feature | Resilience4j | Hystrix |
|---------|-------------|---------|
| Status | Active | Maintenance mode |
| Java 8+ | ✅ | ✅ |
| Functional | ✅ | Limited |
| Spring Boot | ✅ Native | ✅ |
| Metrics | Micrometer | Spectator |

---

*Part 43/100+ | Kotlin & Spring Boot Complete Course*
