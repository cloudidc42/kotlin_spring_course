# Part 52: Inter-Service Communication Patterns
## Synchronous และ Asynchronous Communication

---

## 🎯 เป้าหมายของ Part นี้

- Synchronous vs Asynchronous communication
- REST + Feign สำหรับ sync calls
- Kafka สำหรับ async messaging
- Saga pattern
- Outbox pattern
- ตัวอย่าง: Order processing workflow

---

## 🔄 1. Synchronous Communication

### REST with OpenFeign

```kotlin
// shared/dto/UserResponse.kt
data class UserResponse(
    val id: Long,
    val name: String,
    val email: String,
    val isActive: Boolean
)

// order-service/client/UserClient.kt
@FeignClient(
    name = "user-service",
    fallbackFactory = UserClientFallbackFactory::class
)
interface UserClient {
    
    @GetMapping("/api/users/{id}")
    fun getUser(@PathVariable id: Long): UserResponse
    
    @PostMapping("/api/users/{id}/validate")
    fun validateUser(@PathVariable id: Long): Boolean
}

// Fallback factory (Circuit breaker fallback)
@Component
class UserClientFallbackFactory : FallbackFactory<UserClient> {
    
    private val log = LoggerFactory.getLogger(UserClientFallbackFactory::class.java)
    
    override fun create(cause: Throwable): UserClient {
        return object : UserClient {
            override fun getUser(id: Long): UserResponse {
                log.error("UserClient fallback triggered: ${cause.message}")
                throw ServiceUnavailableException("User service unavailable")
            }
            
            override fun validateUser(id: Long): Boolean {
                log.error("UserClient validateUser fallback")
                return false  // Default: assume invalid
            }
        }
    }
}

// Enable Feign in application
@SpringBootApplication
@EnableFeignClients  // ต้องเพิ่ม
class OrderServiceApplication
```

### WebClient (Reactive)

```kotlin
// order-service/client/ProductWebClient.kt
@Component
class ProductWebClient(private val webClientBuilder: WebClient.Builder) {
    
    private val webClient: WebClient by lazy {
        webClientBuilder
            .baseUrl("http://product-service")  // Eureka will resolve
            .defaultHeader(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE)
            .build()
    }
    
    suspend fun getProduct(id: Long): ProductResponse? {
        return webClient.get()
            .uri("/api/products/{id}", id)
            .retrieve()
            .onStatus(HttpStatusCode::is4xxClientError) { response ->
                response.bodyToMono(String::class.java).map {
                    NotFoundException("Product not found", "Product", id)
                }
            }
            .bodyToMono(ProductResponse::class.java)
            .awaitSingleOrNull()
    }
    
    // Parallel calls
    suspend fun getProducts(ids: List<Long>): List<ProductResponse> {
        return ids
            .map { id -> async { getProduct(id) } }
            .awaitAll()
            .filterNotNull()
    }
}
```

---

## 📨 2. Asynchronous Communication (Kafka)

```kotlin
// build.gradle.kts
dependencies {
    implementation("org.springframework.kafka:spring-kafka")
}
```

```yaml
# application.yml
spring:
  kafka:
    bootstrap-servers: localhost:9092
    consumer:
      group-id: order-service
      auto-offset-reset: earliest
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
      properties:
        spring.json.trusted.packages: "*"
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
```

### Producer (Publishing Events)

```kotlin
// events/OrderEvents.kt
data class OrderCreatedEvent(
    val orderId: Long,
    val userId: Long,
    val totalAmount: Double,
    val items: List<OrderItemDto>,
    val createdAt: LocalDateTime = LocalDateTime.now()
)

data class OrderCancelledEvent(
    val orderId: Long,
    val reason: String,
    val cancelledAt: LocalDateTime = LocalDateTime.now()
)

// service/OrderEventPublisher.kt
@Service
class OrderEventPublisher(private val kafkaTemplate: KafkaTemplate<String, Any>) {
    
    fun publishOrderCreated(event: OrderCreatedEvent) {
        kafkaTemplate.send("order.created", event.orderId.toString(), event)
            .whenComplete { result, ex ->
                if (ex != null) {
                    log.error("Failed to publish OrderCreatedEvent: ${ex.message}")
                } else {
                    log.info("Published OrderCreatedEvent: orderId=${event.orderId}")
                }
            }
    }
    
    fun publishOrderCancelled(event: OrderCancelledEvent) {
        kafkaTemplate.send("order.cancelled", event.orderId.toString(), event)
    }
}
```

### Consumer (Handling Events)

```kotlin
// notification-service/consumer/OrderEventConsumer.kt
@Component
class OrderEventConsumer(private val notificationService: NotificationService) {
    
    private val log = LoggerFactory.getLogger(OrderEventConsumer::class.java)
    
    @KafkaListener(topics = ["order.created"], groupId = "notification-service")
    fun handleOrderCreated(
        event: OrderCreatedEvent,
        @Header(KafkaHeaders.RECEIVED_TOPIC) topic: String,
        @Header(KafkaHeaders.OFFSET) offset: Long
    ) {
        log.info("Received order.created event: orderId=${event.orderId}, offset=$offset")
        
        try {
            notificationService.sendOrderConfirmation(event)
        } catch (e: Exception) {
            log.error("Failed to process OrderCreatedEvent: ${e.message}")
            throw e  // Kafka will retry
        }
    }
    
    @KafkaListener(topics = ["order.cancelled"], groupId = "notification-service")
    fun handleOrderCancelled(event: OrderCancelledEvent) {
        log.info("Order cancelled: ${event.orderId}")
        notificationService.sendCancellationNotice(event)
    }
}

// inventory-service/consumer/OrderEventConsumer.kt
@Component
class InventoryEventConsumer(private val inventoryService: InventoryService) {
    
    @KafkaListener(topics = ["order.created"], groupId = "inventory-service")
    fun reserveInventory(event: OrderCreatedEvent) {
        event.items.forEach { item ->
            inventoryService.reserve(item.productId, item.quantity)
        }
    }
    
    @KafkaListener(topics = ["order.cancelled"], groupId = "inventory-service")
    fun releaseInventory(event: OrderCancelledEvent) {
        inventoryService.releaseReservation(event.orderId)
    }
}
```

---

## 🔄 3. Saga Pattern

Saga = ลำดับ local transactions ที่ publish events

```kotlin
// Choreography-based Saga (event-driven)
// 1. Order Service → creates order → publishes OrderCreated
// 2. Payment Service → receives OrderCreated → charges payment → publishes PaymentProcessed
// 3. Inventory Service → receives PaymentProcessed → reserves stock → publishes InventoryReserved
// 4. Order Service → receives InventoryReserved → marks order CONFIRMED

// order-service/saga/OrderSaga.kt
@Component
class OrderSaga(
    private val orderRepo: OrderRepository,
    private val eventPublisher: OrderEventPublisher
) {
    @KafkaListener(topics = ["payment.processed"])
    fun onPaymentProcessed(event: PaymentProcessedEvent) {
        val order = orderRepo.findById(event.orderId).orElse(null) ?: return
        
        if (event.success) {
            // Payment OK - update order status
            orderRepo.save(order.copy(status = OrderStatus.PAYMENT_CONFIRMED))
            eventPublisher.publishPaymentConfirmed(order.id)
        } else {
            // Payment failed - compensating transaction
            orderRepo.save(order.copy(status = OrderStatus.PAYMENT_FAILED))
            eventPublisher.publishOrderFailed(order.id, "Payment failed")
        }
    }
    
    @KafkaListener(topics = ["inventory.reserved"])
    fun onInventoryReserved(event: InventoryReservedEvent) {
        val order = orderRepo.findById(event.orderId).orElse(null) ?: return
        orderRepo.save(order.copy(status = OrderStatus.CONFIRMED))
    }
    
    @KafkaListener(topics = ["inventory.insufficient"])
    fun onInventoryInsufficient(event: InventoryInsufficientEvent) {
        val order = orderRepo.findById(event.orderId).orElse(null) ?: return
        
        // Compensating: cancel payment
        orderRepo.save(order.copy(status = OrderStatus.CANCELLED))
        eventPublisher.publishCancelPayment(order.id)
    }
}
```

---

## 📬 4. Outbox Pattern

ป้องกัน lost messages เมื่อ DB transaction และ Kafka publish ล้มเหลว

```kotlin
// domain/OutboxMessage.kt
@Entity
@Table(name = "outbox_messages")
data class OutboxMessage(
    @Id @GeneratedValue val id: Long = 0,
    val aggregateType: String,
    val aggregateId: String,
    val eventType: String,
    
    @Column(columnDefinition = "TEXT")
    val payload: String,
    
    var status: Status = Status.PENDING,
    val createdAt: LocalDateTime = LocalDateTime.now(),
    var processedAt: LocalDateTime? = null
) {
    enum class Status { PENDING, PROCESSED, FAILED }
}

// service/OrderService.kt
@Service
@Transactional
class OrderService(
    private val orderRepo: OrderRepository,
    private val outboxRepo: OutboxMessageRepository,
    private val objectMapper: ObjectMapper
) {
    fun createOrder(request: CreateOrderRequest): Order {
        // 1. Save order
        val order = orderRepo.save(Order(/* ... */))
        
        // 2. Save outbox message IN SAME TRANSACTION
        val event = OrderCreatedEvent(orderId = order.id, /* ... */)
        outboxRepo.save(OutboxMessage(
            aggregateType = "Order",
            aggregateId = order.id.toString(),
            eventType = "OrderCreated",
            payload = objectMapper.writeValueAsString(event)
        ))
        
        return order
        // Both commit or both rollback!
    }
}

// Outbox Processor (scheduler)
@Component
class OutboxProcessor(
    private val outboxRepo: OutboxMessageRepository,
    private val kafkaTemplate: KafkaTemplate<String, String>
) {
    @Scheduled(fixedDelay = 1000)  // ทุก 1 วินาที
    @Transactional
    fun processOutbox() {
        val messages = outboxRepo.findByStatusOrderByCreatedAt(
            OutboxMessage.Status.PENDING,
            PageRequest.of(0, 100)
        )
        
        messages.forEach { msg ->
            try {
                kafkaTemplate.send("events.${msg.eventType}", msg.aggregateId, msg.payload)
                msg.status = OutboxMessage.Status.PROCESSED
                msg.processedAt = LocalDateTime.now()
                outboxRepo.save(msg)
            } catch (e: Exception) {
                msg.status = OutboxMessage.Status.FAILED
                outboxRepo.save(msg)
            }
        }
    }
}
```

---

## 📝 สรุป Part 52

| Pattern | Use Case |
|---------|---------|
| REST/Feign | Simple request/response |
| WebClient | Async HTTP calls |
| Kafka | Event streaming, decoupled services |
| Saga | Distributed transactions |
| Outbox | Reliable event publishing |

---

## ➡️ ถัดไป: Part 53 - Config Server และ Centralized Config

---
*Part 52/100+ | Kotlin & Spring Boot Complete Course*
