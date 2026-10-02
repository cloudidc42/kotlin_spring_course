# Part 32: Async Messaging ด้วย RabbitMQ
## Message Queue สำหรับ Asynchronous Communication

---

## 🎯 เป้าหมายของ Part นี้

- เข้าใจ Message Queue และ RabbitMQ
- Setup RabbitMQ ด้วย Docker
- Producer ส่ง message ด้วย RabbitTemplate
- Consumer รับ message ด้วย @RabbitListener
- Message serialization ด้วย JSON
- Dead Letter Queue (DLQ) สำหรับ error handling
- ตัวอย่าง: Order processing queue

---

## 📨 1. ทำความเข้าใจ Message Queue

**Message Queue** คือระบบที่ช่วยให้ applications สื่อสารกันแบบ asynchronous

```
Producer → [Queue] → Consumer
   App A      MQ       App B
```

### ประโยชน์
- **Decoupling**: Producer และ Consumer ทำงานอิสระจากกัน
- **Reliability**: Message ไม่หายแม้ Consumer ล่ม
- **Scalability**: เพิ่ม Consumer ได้ตามต้องการ
- **Load leveling**: ดูดซับ traffic spike ได้

### RabbitMQ Concepts
- **Exchange**: รับ message จาก Producer แล้วส่งต่อไป Queue
- **Queue**: เก็บ message รอ Consumer
- **Binding**: เชื่อม Exchange กับ Queue
- **Routing Key**: ใช้ Route message ไปยัง Queue ที่ถูกต้อง

---

## 🐳 2. RabbitMQ Setup ด้วย Docker

```yaml
# docker-compose.yml
version: '3.8'
services:
  rabbitmq:
    image: rabbitmq:3.12-management-alpine
    container_name: rabbitmq
    ports:
      - "5672:5672"    # AMQP port
      - "15672:15672"  # Management UI
    environment:
      RABBITMQ_DEFAULT_USER: admin
      RABBITMQ_DEFAULT_PASS: password
      RABBITMQ_DEFAULT_VHOST: /
    volumes:
      - rabbitmq-data:/var/lib/rabbitmq
    healthcheck:
      test: ["CMD", "rabbitmq-diagnostics", "-q", "ping"]
      interval: 30s
      timeout: 10s
      retries: 5

volumes:
  rabbitmq-data:
```

```bash
# Start RabbitMQ
docker-compose up -d rabbitmq

# Management UI: http://localhost:15672
# Username: admin, Password: password
```

---

## 📦 3. Dependencies

```kotlin
// build.gradle.kts
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-amqp")
    implementation("com.fasterxml.jackson.module:jackson-module-kotlin")
    implementation("com.fasterxml.jackson.datatype:jackson-datatype-jsr310")
    
    testImplementation("org.springframework.amqp:spring-rabbit-test")
}
```

```yaml
# application.yml
spring:
  rabbitmq:
    host: localhost
    port: 5672
    username: admin
    password: password
    virtual-host: /
    connection-timeout: 10000
    listener:
      simple:
        acknowledge-mode: manual  # Manual acknowledgment
        prefetch: 1               # รับทีละ 1 message
        retry:
          enabled: true
          max-attempts: 3
          initial-interval: 1000
          multiplier: 2.0
          max-interval: 10000

# Custom settings
app:
  rabbitmq:
    exchange:
      orders: orders.exchange
      dlx: orders.dlx
    queue:
      order-created: orders.created
      order-payment: orders.payment
      order-notification: orders.notification
      dlq: orders.dlq
    routing-key:
      order-created: order.created
      order-payment: order.payment
      order-notification: order.notification
```

---

## ⚙️ 4. RabbitMQ Configuration

```kotlin
// src/main/kotlin/com/example/config/RabbitMQConfig.kt
package com.example.config

import com.fasterxml.jackson.databind.ObjectMapper
import com.fasterxml.jackson.module.kotlin.kotlinModule
import org.springframework.amqp.core.*
import org.springframework.amqp.rabbit.config.SimpleRabbitListenerContainerFactory
import org.springframework.amqp.rabbit.connection.ConnectionFactory
import org.springframework.amqp.rabbit.core.RabbitTemplate
import org.springframework.amqp.support.converter.Jackson2JsonMessageConverter
import org.springframework.amqp.support.converter.MessageConverter
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration

@Configuration
class RabbitMQConfig {

    // ===== Exchanges =====

    // Direct Exchange: route by exact routing key
    @Bean
    fun ordersExchange(): DirectExchange {
        return DirectExchange("orders.exchange", true, false)
    }

    // Dead Letter Exchange
    @Bean
    fun deadLetterExchange(): DirectExchange {
        return DirectExchange("orders.dlx", true, false)
    }

    // Topic Exchange: route by pattern (*.order, order.#)
    @Bean
    fun notificationExchange(): TopicExchange {
        return TopicExchange("notifications.exchange", true, false)
    }

    // ===== Queues =====

    // Order Created Queue พร้อม DLQ configuration
    @Bean
    fun orderCreatedQueue(): Queue {
        return QueueBuilder.durable("orders.created")
            .withArgument("x-dead-letter-exchange", "orders.dlx")
            .withArgument("x-dead-letter-routing-key", "orders.dead")
            .withArgument("x-message-ttl", 60000)  // 1 minute TTL
            .build()
    }

    // Order Payment Queue
    @Bean
    fun orderPaymentQueue(): Queue {
        return QueueBuilder.durable("orders.payment")
            .withArgument("x-dead-letter-exchange", "orders.dlx")
            .withArgument("x-dead-letter-routing-key", "orders.dead")
            .build()
    }

    // Order Notification Queue
    @Bean
    fun orderNotificationQueue(): Queue {
        return QueueBuilder.durable("orders.notification").build()
    }

    // Dead Letter Queue
    @Bean
    fun deadLetterQueue(): Queue {
        return QueueBuilder.durable("orders.dlq").build()
    }

    // ===== Bindings =====

    @Bean
    fun orderCreatedBinding(): Binding {
        return BindingBuilder.bind(orderCreatedQueue())
            .to(ordersExchange())
            .with("order.created")
    }

    @Bean
    fun orderPaymentBinding(): Binding {
        return BindingBuilder.bind(orderPaymentQueue())
            .to(ordersExchange())
            .with("order.payment")
    }

    @Bean
    fun orderNotificationBinding(): Binding {
        return BindingBuilder.bind(orderNotificationQueue())
            .to(notificationExchange())
            .with("order.#")  // matches order.created, order.updated, etc.
    }

    @Bean
    fun deadLetterBinding(): Binding {
        return BindingBuilder.bind(deadLetterQueue())
            .to(deadLetterExchange())
            .with("orders.dead")
    }

    // ===== Message Converter =====

    @Bean
    fun messageConverter(): MessageConverter {
        val objectMapper = ObjectMapper().apply {
            registerModule(kotlinModule())
            findAndRegisterModules()
        }
        return Jackson2JsonMessageConverter(objectMapper)
    }

    // ===== RabbitTemplate =====

    @Bean
    fun rabbitTemplate(
        connectionFactory: ConnectionFactory,
        messageConverter: MessageConverter
    ): RabbitTemplate {
        return RabbitTemplate(connectionFactory).apply {
            this.messageConverter = messageConverter
            // Confirm mode สำหรับ publisher confirms
            setConfirmCallback { correlationData, ack, cause ->
                if (!ack) {
                    println("Message not confirmed: $cause")
                }
            }
        }
    }

    // ===== Listener Container Factory =====

    @Bean
    fun rabbitListenerContainerFactory(
        connectionFactory: ConnectionFactory,
        messageConverter: MessageConverter
    ): SimpleRabbitListenerContainerFactory {
        return SimpleRabbitListenerContainerFactory().apply {
            setConnectionFactory(connectionFactory)
            setMessageConverter(messageConverter)
            setAcknowledgeMode(AcknowledgeMode.MANUAL)
            setPrefetchCount(1)
        }
    }
}
```

---

## 📤 5. Message Producer

```kotlin
// src/main/kotlin/com/example/messaging/OrderMessageProducer.kt
package com.example.messaging

import com.example.dto.OrderMessage
import org.springframework.amqp.rabbit.core.RabbitTemplate
import org.springframework.stereotype.Component
import java.util.UUID

@Component
class OrderMessageProducer(private val rabbitTemplate: RabbitTemplate) {

    companion object {
        const val ORDERS_EXCHANGE = "orders.exchange"
        const val NOTIFICATIONS_EXCHANGE = "notifications.exchange"

        const val ROUTING_ORDER_CREATED = "order.created"
        const val ROUTING_ORDER_PAYMENT = "order.payment"
        const val ROUTING_ORDER_NOTIFICATION = "order.notification"
    }

    // ส่ง message เมื่อ order ถูกสร้าง
    fun sendOrderCreated(orderMessage: OrderMessage) {
        println("[Producer] Sending order created: ${orderMessage.orderId}")
        rabbitTemplate.convertAndSend(
            ORDERS_EXCHANGE,
            ROUTING_ORDER_CREATED,
            orderMessage
        )
    }

    // ส่ง message สำหรับ payment processing
    fun sendOrderForPayment(orderMessage: OrderMessage) {
        println("[Producer] Sending order for payment: ${orderMessage.orderId}")
        rabbitTemplate.convertAndSend(
            ORDERS_EXCHANGE,
            ROUTING_ORDER_PAYMENT,
            orderMessage
        )
    }

    // ส่ง notification message
    fun sendOrderNotification(orderMessage: OrderMessage, routingKey: String) {
        println("[Producer] Sending notification for order: ${orderMessage.orderId}")
        rabbitTemplate.convertAndSend(
            NOTIFICATIONS_EXCHANGE,
            routingKey,  // e.g., "order.created", "order.shipped"
            orderMessage
        )
    }

    // ส่ง message พร้อม delay (ต้องการ RabbitMQ Delayed Message Plugin)
    fun sendDelayedMessage(orderMessage: OrderMessage, delayMs: Long) {
        rabbitTemplate.convertAndSend(
            ORDERS_EXCHANGE,
            ROUTING_ORDER_CREATED,
            orderMessage
        ) { message ->
            message.messageProperties.headers["x-delay"] = delayMs
            message
        }
    }
}
```

---

## 📥 6. Message Consumer

```kotlin
// src/main/kotlin/com/example/messaging/OrderMessageConsumer.kt
package com.example.messaging

import com.example.dto.OrderMessage
import com.example.service.OrderProcessingService
import com.rabbitmq.client.Channel
import org.springframework.amqp.rabbit.annotation.RabbitListener
import org.springframework.amqp.support.AmqpHeaders
import org.springframework.messaging.handler.annotation.Header
import org.springframework.stereotype.Component

@Component
class OrderMessageConsumer(
    private val orderProcessingService: OrderProcessingService
) {

    // Consumer สำหรับ order created
    @RabbitListener(queues = ["orders.created"])
    fun handleOrderCreated(
        orderMessage: OrderMessage,
        channel: Channel,
        @Header(AmqpHeaders.DELIVERY_TAG) deliveryTag: Long
    ) {
        println("[Consumer] Processing order created: ${orderMessage.orderId}")
        try {
            orderProcessingService.processNewOrder(orderMessage)
            // ยืนยัน message สำเร็จ
            channel.basicAck(deliveryTag, false)
            println("[Consumer] Order created processed: ${orderMessage.orderId}")
        } catch (e: Exception) {
            println("[Consumer] Error processing order ${orderMessage.orderId}: ${e.message}")
            // Reject และส่งไป DLQ (requeue = false)
            channel.basicNack(deliveryTag, false, false)
        }
    }

    // Consumer สำหรับ payment processing
    @RabbitListener(queues = ["orders.payment"])
    fun handleOrderPayment(
        orderMessage: OrderMessage,
        channel: Channel,
        @Header(AmqpHeaders.DELIVERY_TAG) deliveryTag: Long,
        @Header(AmqpHeaders.REDELIVERED) redelivered: Boolean
    ) {
        println("[Consumer] Processing payment for order: ${orderMessage.orderId}")
        try {
            if (redelivered) {
                println("[Consumer] This is a redelivered message!")
            }
            orderProcessingService.processPayment(orderMessage)
            channel.basicAck(deliveryTag, false)
        } catch (e: Exception) {
            println("[Consumer] Payment error: ${e.message}")
            // Requeue ถ้าเป็น temporary error
            val shouldRequeue = e is TemporaryException
            channel.basicNack(deliveryTag, false, shouldRequeue)
        }
    }

    // Consumer สำหรับ notifications (Topic Exchange)
    @RabbitListener(queues = ["orders.notification"])
    fun handleOrderNotification(orderMessage: OrderMessage) {
        println("[Consumer] Sending notification for order: ${orderMessage.orderId}")
        // ไม่ต้องการ manual ack สำหรับ notifications
        orderProcessingService.sendNotification(orderMessage)
    }

    // Dead Letter Queue consumer
    @RabbitListener(queues = ["orders.dlq"])
    fun handleDeadLetterMessages(
        orderMessage: OrderMessage,
        channel: Channel,
        @Header(AmqpHeaders.DELIVERY_TAG) deliveryTag: Long
    ) {
        println("[DLQ Consumer] Processing dead letter: ${orderMessage.orderId}")
        try {
            // บันทึก failed message ลง database
            orderProcessingService.handleFailedOrder(orderMessage)
            channel.basicAck(deliveryTag, false)
        } catch (e: Exception) {
            // ถ้า DLQ processing ล้มเหลว ให้ reject โดยไม่ requeue
            channel.basicNack(deliveryTag, false, false)
        }
    }
}

class TemporaryException(message: String) : RuntimeException(message)
```

---

## 📝 7. Message DTOs

```kotlin
// src/main/kotlin/com/example/dto/OrderMessage.kt
package com.example.dto

import java.io.Serializable
import java.math.BigDecimal
import java.time.LocalDateTime

data class OrderMessage(
    val orderId: Long,
    val customerId: Long,
    val customerEmail: String,
    val items: List<OrderItemMessage>,
    val totalAmount: BigDecimal,
    val status: OrderStatus,
    val createdAt: LocalDateTime = LocalDateTime.now(),
    val correlationId: String? = null  // สำหรับ tracing
) : Serializable

data class OrderItemMessage(
    val productId: Long,
    val productName: String,
    val quantity: Int,
    val price: BigDecimal
) : Serializable

enum class OrderStatus {
    PENDING,
    PAYMENT_PROCESSING,
    PAYMENT_CONFIRMED,
    PAYMENT_FAILED,
    PROCESSING,
    SHIPPED,
    DELIVERED,
    CANCELLED
}
```

---

## 🛒 8. Order Processing Service

```kotlin
// src/main/kotlin/com/example/service/OrderProcessingService.kt
package com.example.service

import com.example.dto.OrderMessage
import com.example.dto.OrderStatus
import com.example.entity.Order
import com.example.messaging.OrderMessageProducer
import com.example.repository.OrderRepository
import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional

@Service
class OrderProcessingService(
    private val orderRepository: OrderRepository,
    private val orderMessageProducer: OrderMessageProducer
) {

    // ประมวลผล order ใหม่
    @Transactional
    fun processNewOrder(message: OrderMessage) {
        println("[Service] Processing new order: ${message.orderId}")

        // บันทึก order ลง DB
        val order = orderRepository.findById(message.orderId)
            .orElseThrow { RuntimeException("Order not found: ${message.orderId}") }

        order.status = OrderStatus.PAYMENT_PROCESSING
        orderRepository.save(order)

        // ส่ง message สำหรับ payment
        orderMessageProducer.sendOrderForPayment(message)
    }

    // ประมวลผล payment
    @Transactional
    fun processPayment(message: OrderMessage) {
        println("[Service] Processing payment: ${message.orderId}")

        // จำลอง payment processing
        val paymentSuccess = simulatePayment(message)

        val order = orderRepository.findById(message.orderId)
            .orElseThrow { RuntimeException("Order not found") }

        if (paymentSuccess) {
            order.status = OrderStatus.PAYMENT_CONFIRMED
            orderRepository.save(order)

            // ส่ง notification
            val confirmedMessage = message.copy(status = OrderStatus.PAYMENT_CONFIRMED)
            orderMessageProducer.sendOrderNotification(confirmedMessage, "order.payment.confirmed")
        } else {
            order.status = OrderStatus.PAYMENT_FAILED
            orderRepository.save(order)
            throw RuntimeException("Payment failed for order ${message.orderId}")
        }
    }

    // ส่ง notification
    fun sendNotification(message: OrderMessage) {
        println("[Service] Sending notification to: ${message.customerEmail}")
        // ส่ง email notification ที่นี่
    }

    // จัดการ failed orders จาก DLQ
    @Transactional
    fun handleFailedOrder(message: OrderMessage) {
        println("[Service] Handling failed order: ${message.orderId}")
        // บันทึก error ลง database หรือส่ง alert
        val order = orderRepository.findById(message.orderId).orElse(null)
        order?.let {
            it.status = OrderStatus.CANCELLED
            orderRepository.save(it)
        }
    }

    private fun simulatePayment(message: OrderMessage): Boolean {
        // จำลอง: 90% success rate
        return Math.random() > 0.1
    }
}
```

---

## 🛒 9. Order Service (Main)

```kotlin
// src/main/kotlin/com/example/service/OrderService.kt
package com.example.service

import com.example.dto.*
import com.example.entity.Order
import com.example.messaging.OrderMessageProducer
import com.example.repository.OrderRepository
import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional
import java.math.BigDecimal
import java.time.LocalDateTime

@Service
@Transactional
class OrderService(
    private val orderRepository: OrderRepository,
    private val orderMessageProducer: OrderMessageProducer
) {

    // สร้าง order และส่ง message
    fun createOrder(dto: CreateOrderDto): OrderResponseDto {
        // สร้าง order ใน DB
        val order = Order(
            customerId = dto.customerId,
            customerEmail = dto.customerEmail,
            items = dto.items,
            totalAmount = dto.items.sumOf { it.price * it.quantity.toBigDecimal() },
            status = OrderStatus.PENDING,
            createdAt = LocalDateTime.now()
        )
        val savedOrder = orderRepository.save(order)

        // สร้าง message และส่ง
        val message = OrderMessage(
            orderId = savedOrder.id,
            customerId = savedOrder.customerId,
            customerEmail = savedOrder.customerEmail,
            items = dto.items.map { item ->
                OrderItemMessage(
                    productId = item.productId,
                    productName = item.productName,
                    quantity = item.quantity,
                    price = item.price
                )
            },
            totalAmount = savedOrder.totalAmount,
            status = OrderStatus.PENDING
        )

        // ส่ง message ไป queue (async)
        orderMessageProducer.sendOrderCreated(message)
        orderMessageProducer.sendOrderNotification(message, "order.created")

        println("[Service] Order created and message sent: ${savedOrder.id}")

        return OrderResponseDto(
            orderId = savedOrder.id,
            status = savedOrder.status,
            message = "Order created successfully and is being processed"
        )
    }

    fun getOrderStatus(orderId: Long): OrderStatus {
        return orderRepository.findById(orderId)
            .map { it.status }
            .orElseThrow { NotFoundException("Order $orderId not found") }
    }
}
```

---

## 🔁 10. Retry และ Error Handling

```kotlin
// src/main/kotlin/com/example/config/RabbitRetryConfig.kt
package com.example.config

import org.springframework.amqp.rabbit.config.RetryInterceptorBuilder
import org.springframework.amqp.rabbit.config.SimpleRabbitListenerContainerFactory
import org.springframework.amqp.rabbit.connection.ConnectionFactory
import org.springframework.amqp.rabbit.retry.RejectAndDontRequeueRecoverer
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.retry.interceptor.RetryOperationsInterceptor

@Configuration
class RabbitRetryConfig {

    @Bean
    fun retryInterceptor(): RetryOperationsInterceptor {
        return RetryInterceptorBuilder.stateless()
            .maxAttempts(3)
            .backOffOptions(1000, 2.0, 10000)  // initial, multiplier, max
            .recoverer(RejectAndDontRequeueRecoverer())  // ส่งไป DLQ หลัง retry หมด
            .build()
    }

    @Bean
    fun retryContainerFactory(
        connectionFactory: ConnectionFactory,
        retryInterceptor: RetryOperationsInterceptor
    ): SimpleRabbitListenerContainerFactory {
        return SimpleRabbitListenerContainerFactory().apply {
            setConnectionFactory(connectionFactory)
            setAdviceChain(retryInterceptor)
        }
    }
}
```

---

## 📊 11. Monitoring และ Management

```kotlin
// src/main/kotlin/com/example/controller/RabbitMQController.kt
package com.example.controller

import org.springframework.amqp.rabbit.core.RabbitAdmin
import org.springframework.http.ResponseEntity
import org.springframework.web.bind.annotation.*

@RestController
@RequestMapping("/api/rabbitmq")
class RabbitMQController(private val rabbitAdmin: RabbitAdmin) {

    @GetMapping("/queue/{queueName}/info")
    fun getQueueInfo(@PathVariable queueName: String): ResponseEntity<Map<String, Any?>> {
        val properties = rabbitAdmin.getQueueProperties(queueName)
        return ResponseEntity.ok(
            mapOf(
                "name" to queueName,
                "messageCount" to properties?.get("QUEUE_MESSAGE_COUNT"),
                "consumerCount" to properties?.get("QUEUE_CONSUMER_COUNT")
            )
        )
    }

    @DeleteMapping("/queue/{queueName}/purge")
    fun purgeQueue(@PathVariable queueName: String): ResponseEntity<Map<String, String>> {
        rabbitAdmin.purgeQueue(queueName, false)
        return ResponseEntity.ok(mapOf("message" to "Queue $queueName purged"))
    }
}
```

---

## 🧪 12. Testing

```kotlin
// src/test/kotlin/com/example/messaging/OrderMessagingTest.kt
package com.example.messaging

import com.example.dto.OrderMessage
import com.example.dto.OrderStatus
import org.junit.jupiter.api.Test
import org.springframework.amqp.rabbit.core.RabbitTemplate
import org.springframework.amqp.rabbit.test.RabbitListenerTestHarness
import org.springframework.beans.factory.annotation.Autowired
import org.springframework.boot.test.context.SpringBootTest
import org.springframework.test.context.DynamicPropertyRegistry
import org.springframework.test.context.DynamicPropertySource
import org.testcontainers.containers.RabbitMQContainer
import org.testcontainers.junit.jupiter.Container
import org.testcontainers.junit.jupiter.Testcontainers
import java.math.BigDecimal
import java.util.concurrent.TimeUnit
import kotlin.test.assertNotNull

@SpringBootTest
@Testcontainers
class OrderMessagingTest {

    companion object {
        @Container
        val rabbitmq = RabbitMQContainer("rabbitmq:3.12-management-alpine")

        @DynamicPropertySource
        @JvmStatic
        fun rabbitProperties(registry: DynamicPropertyRegistry) {
            registry.add("spring.rabbitmq.host") { rabbitmq.host }
            registry.add("spring.rabbitmq.port") { rabbitmq.amqpPort }
            registry.add("spring.rabbitmq.username") { rabbitmq.adminUsername }
            registry.add("spring.rabbitmq.password") { rabbitmq.adminPassword }
        }
    }

    @Autowired
    lateinit var orderMessageProducer: OrderMessageProducer

    @Autowired
    lateinit var rabbitTemplate: RabbitTemplate

    @Test
    fun `should send and receive order message`() {
        val orderMessage = OrderMessage(
            orderId = 1L,
            customerId = 1L,
            customerEmail = "test@example.com",
            items = emptyList(),
            totalAmount = BigDecimal("100.00"),
            status = OrderStatus.PENDING
        )

        orderMessageProducer.sendOrderCreated(orderMessage)

        // ตรวจสอบว่า message ถูกส่ง
        val received = rabbitTemplate.receiveAndConvert(
            "orders.created",
            5000  // timeout 5 seconds
        )
        assertNotNull(received)
    }
}
```

---

## 📋 สรุป

| Component | หน้าที่ |
|-----------|---------|
| Exchange | รับ message และ route ไปยัง Queue |
| Queue | เก็บ message รอ Consumer |
| Producer | ส่ง message ผ่าน RabbitTemplate |
| Consumer | รับ message ด้วย @RabbitListener |
| DLQ | เก็บ message ที่ process ไม่สำเร็จ |

### Exchange Types

| Type | การ Route |
|------|-----------|
| Direct | ตาม Routing Key ตรงๆ |
| Fanout | ส่งไปทุก Queue ที่ bind |
| Topic | ใช้ pattern matching (`*`, `#`) |
| Headers | ใช้ message headers |

### Acknowledgment Modes

| Mode | คำอธิบาย |
|------|---------|
| `AUTO` | ยืนยันอัตโนมัติหลัง method return |
| `MANUAL` | ยืนยันเองด้วย `basicAck/basicNack` |
| `NONE` | ไม่ต้องยืนยัน (at-most-once) |

---

*Part 32/100+ | Kotlin & Spring Boot Complete Course*
