# Part 54: Distributed Tracing
## ติดตาม Request ข้าม Services ด้วย Micrometer Tracing

---

## 🎯 เป้าหมายของ Part นี้

- เข้าใจ Distributed Tracing
- ตั้งค่า Micrometer Tracing + Zipkin
- Trace ID และ Span ID
- Custom spans
- Log correlation
- ตัวอย่าง: Trace request across services

---

## 📦 1. Dependencies

```kotlin
// build.gradle.kts
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-actuator")
    implementation("io.micrometer:micrometer-tracing-bridge-brave")  // Brave tracer
    implementation("io.zipkin.reporter2:zipkin-reporter-brave")       // Send to Zipkin
    runtimeOnly("io.zipkin.reporter2:zipkin-sender-okhttp3")
}
```

---

## ⚙️ 2. Configuration

```yaml
# application.yml
management:
  tracing:
    sampling:
      probability: 1.0  # 100% ใน dev, ใช้ 0.1 (10%) ใน prod
  zipkin:
    tracing:
      endpoint: http://localhost:9411/api/v2/spans

logging:
  pattern:
    console: "%d{yyyy-MM-dd HH:mm:ss} [%thread] %-5level [%X{traceId},%X{spanId}] %logger{36} - %msg%n"
```

---

## 🔍 3. Automatic Instrumentation

Spring Boot auto-instruments:
- HTTP requests (incoming + outgoing)
- JDBC queries
- Spring Data JPA
- Kafka messages
- Scheduled tasks

```kotlin
// ทุก request จะมี traceId โดยอัตโนมัติ
@RestController
class OrderController {
    
    @GetMapping("/api/orders/{id}")
    fun getOrder(@PathVariable id: Long): Order {
        // Log จะมี [traceId,spanId] โดยอัตโนมัติ:
        // 2026-10-02 10:00:00 [http-nio-8080-exec-1] INFO [abc123def,xyz789] 
        //     OrderController - Getting order 42
        log.info("Getting order $id")
        return orderService.getOrder(id)
    }
}
```

---

## 🎯 4. Custom Spans

```kotlin
import io.micrometer.tracing.Tracer
import io.micrometer.tracing.annotation.NewSpan
import io.micrometer.tracing.annotation.SpanTag

@Service
class OrderService(
    private val tracer: Tracer,
    private val orderRepo: OrderRepository,
    private val paymentClient: PaymentClient
) {
    
    // สร้าง span ใหม่ด้วย annotation
    @NewSpan("order.processPayment")
    fun processPayment(
        @SpanTag("orderId") orderId: Long,
        @SpanTag("amount") amount: Double
    ): PaymentResult {
        return paymentClient.charge(orderId, amount)
    }
    
    // สร้าง span ด้วย code
    fun createOrder(request: CreateOrderRequest): Order {
        val span = tracer.nextSpan().name("order.create")
        
        return tracer.withSpan(span.start()).use {
            span.tag("userId", request.userId.toString())
            span.tag("itemCount", request.items.size.toString())
            
            try {
                val order = orderRepo.save(Order(/* ... */))
                span.tag("orderId", order.id.toString())
                order
            } catch (e: Exception) {
                span.error(e)
                throw e
            }
        }
    }
}
```

---

## 🐳 5. Zipkin กับ Docker

```yaml
# docker-compose.yml
services:
  zipkin:
    image: openzipkin/zipkin:latest
    ports:
      - "9411:9411"
    environment:
      - STORAGE_TYPE=mem  # หรือ elasticsearch สำหรับ production

# ดู UI ที่: http://localhost:9411
```

---

## 📊 6. Log Correlation

```kotlin
// ดึง trace ID ใน code
@Component
class RequestLogger(private val tracer: Tracer) {
    
    fun logWithTrace(message: String) {
        val traceId = tracer.currentSpan()?.context()?.traceId() ?: "no-trace"
        log.info("[traceId=$traceId] $message")
    }
}

// ใน error response
@RestControllerAdvice
class GlobalExceptionHandler(private val tracer: Tracer) {
    
    @ExceptionHandler(Exception::class)
    fun handleError(ex: Exception): ResponseEntity<ProblemDetails> {
        val traceId = tracer.currentSpan()?.context()?.traceId()
        
        return ResponseEntity.status(500).body(
            ProblemDetails(
                status = 500,
                title = "Internal Server Error",
                detail = "Check logs with traceId: $traceId",
                traceId = traceId
            )
        )
    }
}
```

---

## 📝 สรุป Part 54

| แนวคิด | รายละเอียด |
|--------|-----------|
| Distributed tracing | ติดตาม request ข้ามหลาย services |
| Trace ID | ตัวระบุ unique ต่อ request |
| Span | หน่วยงานย่อยภายใน trace |
| Zipkin | Tracing visualization |
| Log correlation | ค้นหา log ด้วย traceId |
| Auto-instrumentation | HTTP, JDBC, Kafka ถูก trace อัตโนมัติ |

---

## ➡️ ถัดไป: Part 55 - Microservices Mini Project

---
*Part 54/100+ | Kotlin & Spring Boot Complete Course*
