# Part 38: Event-Driven Architecture
## Spring Application Events และ Domain Events

---

## 🎯 เป้าหมายของ Part นี้

- Spring Application Events
- @EventListener
- @TransactionalEventListener
- Async events
- Domain Events pattern
- ตัวอย่าง: User registration events

---

## 🎪 1. Event-Driven Architecture คืออะไร

**Event-Driven Architecture (EDA)** คือ architecture pattern ที่ components สื่อสารกันผ่าน events แทนที่จะ call กันโดยตรง

```
Traditional:                    Event-Driven:
UserService → EmailService      UserService → [UserRegistered Event]
UserService → NotifService                         ↓
UserService → AuditService              EmailService (listens)
                                        NotifService (listens)
                                        AuditService (listens)
```

### ประโยชน์
- **Loose coupling**: components ไม่รู้จักกัน
- **Open/Closed Principle**: เพิ่ม behavior โดยไม่แก้ existing code
- **Testability**: test แต่ละ listener แยกกัน
- **Scalability**: listeners สามารถ scale อิสระ

---

## 📦 2. Dependencies

```kotlin
// build.gradle.kts
// Spring Events อยู่ใน spring-context แล้ว ไม่ต้อง add dependency เพิ่ม
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
}
```

---

## 🎯 3. Spring Application Events

### 3.1 สร้าง Event

```kotlin
// src/main/kotlin/com/example/event/UserEvents.kt
package com.example.event

import com.example.entity.User
import org.springframework.context.ApplicationEvent
import java.time.LocalDateTime

// Event Class 1: UserRegistered
// ApplicationEvent เป็น abstract class ที่ Spring ต้องการ
class UserRegisteredEvent(
    source: Any,            // source object ที่ publish event
    val user: User,
    val ipAddress: String? = null,
    val timestamp: LocalDateTime = LocalDateTime.now()
) : ApplicationEvent(source)

// Event Class 2: UserLoggedIn
class UserLoggedInEvent(
    source: Any,
    val userId: Long,
    val email: String,
    val ipAddress: String,
    val userAgent: String,
    val timestamp: LocalDateTime = LocalDateTime.now()
) : ApplicationEvent(source)

// Event Class 3: UserPasswordChanged
class UserPasswordChangedEvent(
    source: Any,
    val userId: Long,
    val email: String,
    val timestamp: LocalDateTime = LocalDateTime.now()
) : ApplicationEvent(source)

// Event Class 4: UserDeactivated
class UserDeactivatedEvent(
    source: Any,
    val userId: Long,
    val email: String,
    val reason: String? = null
) : ApplicationEvent(source)
```

### 3.2 Publish Events

```kotlin
// src/main/kotlin/com/example/service/UserService.kt
package com.example.service

import com.example.dto.RegisterDto
import com.example.entity.User
import com.example.event.*
import com.example.repository.UserRepository
import org.springframework.context.ApplicationEventPublisher
import org.springframework.security.crypto.password.PasswordEncoder
import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional

@Service
@Transactional
class UserService(
    private val userRepository: UserRepository,
    private val passwordEncoder: PasswordEncoder,
    private val eventPublisher: ApplicationEventPublisher
) {

    fun register(dto: RegisterDto, ipAddress: String? = null): User {
        // Validate
        if (userRepository.existsByEmail(dto.email)) {
            throw IllegalArgumentException("Email already exists: ${dto.email}")
        }

        // สร้าง user
        val user = User(
            email = dto.email,
            username = dto.username,
            passwordHash = passwordEncoder.encode(dto.password),
            isActive = true
        )
        val savedUser = userRepository.save(user)

        // Publish event! ไม่ต้องรู้ว่าใคร handle
        eventPublisher.publishEvent(
            UserRegisteredEvent(
                source = this,
                user = savedUser,
                ipAddress = ipAddress
            )
        )

        return savedUser
    }

    fun login(email: String, password: String, ipAddress: String, userAgent: String): User {
        val user = userRepository.findByEmail(email)
            ?: throw UnauthorizedException("Invalid credentials")

        if (!passwordEncoder.matches(password, user.passwordHash)) {
            throw UnauthorizedException("Invalid credentials")
        }

        // Publish login event
        eventPublisher.publishEvent(
            UserLoggedInEvent(
                source = this,
                userId = user.id,
                email = user.email,
                ipAddress = ipAddress,
                userAgent = userAgent
            )
        )

        return user
    }

    fun changePassword(userId: Long, newPassword: String): User {
        val user = userRepository.findById(userId)
            .orElseThrow { NotFoundException("User not found") }

        user.passwordHash = passwordEncoder.encode(newPassword)
        val saved = userRepository.save(user)

        eventPublisher.publishEvent(
            UserPasswordChangedEvent(
                source = this,
                userId = userId,
                email = user.email
            )
        )

        return saved
    }

    fun deactivateUser(userId: Long, reason: String? = null) {
        val user = userRepository.findById(userId)
            .orElseThrow { NotFoundException("User not found") }

        user.isActive = false
        userRepository.save(user)

        eventPublisher.publishEvent(
            UserDeactivatedEvent(
                source = this,
                userId = userId,
                email = user.email,
                reason = reason
            )
        )
    }
}
```

---

## 👂 4. Event Listeners

### 4.1 @EventListener

```kotlin
// src/main/kotlin/com/example/listener/UserEmailListener.kt
package com.example.listener

import com.example.event.*
import com.example.service.EmailService
import org.springframework.context.event.EventListener
import org.springframework.stereotype.Component

@Component
class UserEmailListener(private val emailService: EmailService) {

    // รับ UserRegisteredEvent
    @EventListener
    fun onUserRegistered(event: UserRegisteredEvent) {
        println("[EmailListener] User registered: ${event.user.email}")
        emailService.sendWelcomeEmail(
            to = event.user.email,
            userName = event.user.username,
            verificationToken = "token-${event.user.id}"  // generate real token
        )
    }

    // รับ UserPasswordChangedEvent
    @EventListener
    fun onPasswordChanged(event: UserPasswordChangedEvent) {
        emailService.sendSimpleEmail(
            to = event.email,
            subject = "รหัสผ่านถูกเปลี่ยน",
            body = "รหัสผ่านของคุณถูกเปลี่ยนเมื่อ ${event.timestamp}\nถ้าไม่ใช่คุณ กรุณาติดต่อ support ทันที"
        )
    }

    // รับ UserDeactivatedEvent
    @EventListener
    fun onUserDeactivated(event: UserDeactivatedEvent) {
        emailService.sendSimpleEmail(
            to = event.email,
            subject = "บัญชีของคุณถูกระงับ",
            body = "บัญชีของคุณถูกระงับ\nเหตุผล: ${event.reason ?: "ไม่ระบุ"}"
        )
    }
}
```

```kotlin
// src/main/kotlin/com/example/listener/UserAuditListener.kt
package com.example.listener

import com.example.entity.AuditLog
import com.example.event.*
import com.example.repository.AuditLogRepository
import org.springframework.context.event.EventListener
import org.springframework.stereotype.Component

@Component
class UserAuditListener(private val auditLogRepository: AuditLogRepository) {

    @EventListener
    fun onUserRegistered(event: UserRegisteredEvent) {
        auditLogRepository.save(
            AuditLog(
                entityType = "USER",
                entityId = event.user.id,
                action = "REGISTERED",
                details = "IP: ${event.ipAddress}",
                timestamp = event.timestamp
            )
        )
    }

    @EventListener
    fun onUserLoggedIn(event: UserLoggedInEvent) {
        auditLogRepository.save(
            AuditLog(
                entityType = "USER",
                entityId = event.userId,
                action = "LOGIN",
                details = "IP: ${event.ipAddress}, Agent: ${event.userAgent}",
                timestamp = event.timestamp
            )
        )
    }

    @EventListener
    fun onPasswordChanged(event: UserPasswordChangedEvent) {
        auditLogRepository.save(
            AuditLog(
                entityType = "USER",
                entityId = event.userId,
                action = "PASSWORD_CHANGED",
                timestamp = event.timestamp
            )
        )
    }

    @EventListener
    fun onUserDeactivated(event: UserDeactivatedEvent) {
        auditLogRepository.save(
            AuditLog(
                entityType = "USER",
                entityId = event.userId,
                action = "DEACTIVATED",
                details = "Reason: ${event.reason}"
            )
        )
    }
}
```

### 4.2 Conditional Event Listener

```kotlin
// src/main/kotlin/com/example/listener/VIPListener.kt
package com.example.listener

import com.example.event.UserRegisteredEvent
import org.springframework.context.event.EventListener
import org.springframework.stereotype.Component

@Component
class VIPListener {

    // รับเฉพาะ event ที่ตรงเงื่อนไข (SpEL)
    @EventListener(condition = "#event.user.tier == 'VIP'")
    fun onVIPUserRegistered(event: UserRegisteredEvent) {
        println("[VIP] VIP user registered: ${event.user.email}")
        // ส่ง gift หรือ special welcome package
    }

    // รับหลาย event types พร้อมกัน
    @EventListener(classes = [UserRegisteredEvent::class, UserLoggedInEvent::class])
    fun onUserActivity(event: Any) {
        println("[Activity] User activity: ${event::class.simpleName}")
    }
}
```

---

## 🔄 5. @TransactionalEventListener

```kotlin
// src/main/kotlin/com/example/listener/TransactionalUserListener.kt
package com.example.listener

import com.example.event.UserRegisteredEvent
import com.example.service.EmailService
import org.springframework.stereotype.Component
import org.springframework.transaction.event.TransactionPhase
import org.springframework.transaction.event.TransactionalEventListener

@Component
class TransactionalUserListener(private val emailService: EmailService) {

    // รัน AFTER_COMMIT - ส่ง email เฉพาะเมื่อ transaction สำเร็จ
    // ป้องกันส่ง email เมื่อ transaction rollback
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    fun sendWelcomeEmailAfterCommit(event: UserRegisteredEvent) {
        println("[TransactionalListener] User committed: ${event.user.email}")
        // ปลอดภัยที่จะส่ง email ตอนนี้ เพราะ transaction commit แล้ว
        emailService.sendWelcomeEmail(
            to = event.user.email,
            userName = event.user.username,
            verificationToken = "verification-token"
        )
    }

    // BEFORE_COMMIT - ทำงานก่อน transaction commit
    @TransactionalEventListener(phase = TransactionPhase.BEFORE_COMMIT)
    fun beforeCommit(event: UserRegisteredEvent) {
        println("[TransactionalListener] Before commit: ${event.user.email}")
        // ทำ validation ที่ยังต้องการ transaction
    }

    // AFTER_ROLLBACK - ทำงานเมื่อ transaction rollback
    @TransactionalEventListener(phase = TransactionPhase.AFTER_ROLLBACK)
    fun onRollback(event: UserRegisteredEvent) {
        println("[TransactionalListener] Transaction rolled back for: ${event.user.email}")
        // Log หรือ cleanup
    }

    // AFTER_COMPLETION - ทำงานทุกกรณี (commit หรือ rollback)
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMPLETION)
    fun afterCompletion(event: UserRegisteredEvent) {
        println("[TransactionalListener] Transaction completed (any outcome)")
    }
}
```

---

## ⚡ 6. Async Events

```kotlin
// src/main/kotlin/com/example/listener/AsyncUserListener.kt
package com.example.listener

import com.example.event.UserRegisteredEvent
import org.springframework.context.event.EventListener
import org.springframework.scheduling.annotation.Async
import org.springframework.stereotype.Component

@Component
class AsyncUserListener {

    // @Async: listener รันใน thread แยก ไม่บล็อค publisher
    @Async
    @EventListener
    fun handleAsyncRegistration(event: UserRegisteredEvent) {
        println("[AsyncListener] Processing in thread: ${Thread.currentThread().name}")
        // งานที่ใช้เวลานาน เช่น ส่ง email, resize image
        Thread.sleep(2000)
        println("[AsyncListener] Completed for: ${event.user.email}")
    }

    // Async + Transactional
    @Async
    @org.springframework.transaction.event.TransactionalEventListener(
        phase = org.springframework.transaction.event.TransactionPhase.AFTER_COMMIT
    )
    fun asyncAfterCommit(event: UserRegisteredEvent) {
        // รันใน thread แยก หลัง transaction commit
        println("[AsyncTransactional] Processing after commit")
    }
}
```

---

## 🏗️ 7. Domain Events Pattern

```kotlin
// src/main/kotlin/com/example/entity/BaseEntity.kt
package com.example.entity

import org.springframework.data.domain.AbstractAggregateRoot

// AbstractAggregateRoot ช่วย collect domain events
abstract class BaseAggregateRoot<T : BaseAggregateRoot<T>> : AbstractAggregateRoot<T>() {

    // Register domain event
    protected fun publishDomainEvent(event: Any) {
        registerEvent(event)
    }
}
```

```kotlin
// src/main/kotlin/com/example/entity/Order.kt - Entity ที่มี Domain Events
package com.example.entity

import com.example.event.*
import jakarta.persistence.*
import java.math.BigDecimal
import java.time.LocalDateTime

@Entity
@Table(name = "orders")
class Order(
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long = 0,

    val customerId: Long,
    val customerEmail: String,

    @Enumerated(EnumType.STRING)
    var status: OrderStatus = OrderStatus.PENDING,

    val totalAmount: BigDecimal,
    val createdAt: LocalDateTime = LocalDateTime.now()
) : BaseAggregateRoot<Order>() {

    // Business method ที่ publish domain event
    fun confirm() {
        check(status == OrderStatus.PENDING) { "Can only confirm PENDING orders" }
        status = OrderStatus.CONFIRMED
        publishDomainEvent(OrderConfirmedEvent(this, id, customerEmail, totalAmount))
    }

    fun ship(trackingNumber: String) {
        check(status == OrderStatus.CONFIRMED) { "Can only ship CONFIRMED orders" }
        status = OrderStatus.SHIPPED
        publishDomainEvent(OrderShippedEvent(this, id, customerEmail, trackingNumber))
    }

    fun deliver() {
        check(status == OrderStatus.SHIPPED) { "Can only deliver SHIPPED orders" }
        status = OrderStatus.DELIVERED
        publishDomainEvent(OrderDeliveredEvent(this, id, customerEmail))
    }

    fun cancel(reason: String) {
        check(status !in listOf(OrderStatus.DELIVERED, OrderStatus.CANCELLED)) {
            "Cannot cancel order in status: $status"
        }
        status = OrderStatus.CANCELLED
        publishDomainEvent(OrderCancelledEvent(this, id, customerEmail, reason))
    }
}

enum class OrderStatus {
    PENDING, CONFIRMED, SHIPPED, DELIVERED, CANCELLED
}
```

```kotlin
// src/main/kotlin/com/example/event/OrderEvents.kt
package com.example.event

import java.math.BigDecimal
import java.time.LocalDateTime

data class OrderConfirmedEvent(
    val source: Any,
    val orderId: Long,
    val customerEmail: String,
    val totalAmount: BigDecimal,
    val timestamp: LocalDateTime = LocalDateTime.now()
)

data class OrderShippedEvent(
    val source: Any,
    val orderId: Long,
    val customerEmail: String,
    val trackingNumber: String
)

data class OrderDeliveredEvent(
    val source: Any,
    val orderId: Long,
    val customerEmail: String
)

data class OrderCancelledEvent(
    val source: Any,
    val orderId: Long,
    val customerEmail: String,
    val reason: String
)
```

```kotlin
// src/main/kotlin/com/example/service/OrderDomainService.kt
package com.example.service

import com.example.entity.Order
import com.example.repository.OrderRepository
import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional

@Service
@Transactional
class OrderDomainService(private val orderRepository: OrderRepository) {

    // Spring Data JPA จะ publish domain events อัตโนมัติเมื่อ save
    fun confirmOrder(orderId: Long): Order {
        val order = orderRepository.findById(orderId)
            .orElseThrow { NotFoundException("Order not found") }
        
        order.confirm()  // registers domain event
        return orderRepository.save(order)  // Spring publishes events here!
    }

    fun shipOrder(orderId: Long, trackingNumber: String): Order {
        val order = orderRepository.findById(orderId)
            .orElseThrow { NotFoundException("Order not found") }
        
        order.ship(trackingNumber)
        return orderRepository.save(order)
    }
}
```

```kotlin
// src/main/kotlin/com/example/listener/OrderEventListener.kt
package com.example.listener

import com.example.event.*
import com.example.service.EmailService
import org.springframework.context.event.EventListener
import org.springframework.scheduling.annotation.Async
import org.springframework.stereotype.Component

@Component
class OrderEventListener(private val emailService: EmailService) {

    @Async
    @EventListener
    fun onOrderConfirmed(event: OrderConfirmedEvent) {
        println("[OrderListener] Order confirmed: ${event.orderId}")
        emailService.sendSimpleEmail(
            to = event.customerEmail,
            subject = "คำสั่งซื้อ #${event.orderId} ได้รับการยืนยัน",
            body = "ยอดรวม: ฿${event.totalAmount}"
        )
    }

    @Async
    @EventListener
    fun onOrderShipped(event: OrderShippedEvent) {
        println("[OrderListener] Order shipped: ${event.orderId}")
        emailService.sendSimpleEmail(
            to = event.customerEmail,
            subject = "คำสั่งซื้อ #${event.orderId} จัดส่งแล้ว",
            body = "Tracking: ${event.trackingNumber}"
        )
    }

    @Async
    @EventListener
    fun onOrderCancelled(event: OrderCancelledEvent) {
        emailService.sendSimpleEmail(
            to = event.customerEmail,
            subject = "คำสั่งซื้อ #${event.orderId} ถูกยกเลิก",
            body = "เหตุผล: ${event.reason}"
        )
    }
}
```

---

## 🧪 8. Testing Events

```kotlin
// src/test/kotlin/com/example/service/UserServiceEventTest.kt
package com.example.service

import com.example.dto.RegisterDto
import com.example.event.UserRegisteredEvent
import com.example.repository.UserRepository
import io.mockk.*
import org.junit.jupiter.api.Test
import org.springframework.context.ApplicationEventPublisher
import org.springframework.security.crypto.password.PasswordEncoder
import kotlin.test.assertNotNull

class UserServiceEventTest {

    private val userRepository = mockk<UserRepository>()
    private val passwordEncoder = mockk<PasswordEncoder>()
    private val eventPublisher = mockk<ApplicationEventPublisher>()

    private val userService = UserService(userRepository, passwordEncoder, eventPublisher)

    @Test
    fun `should publish UserRegisteredEvent on registration`() {
        val dto = RegisterDto(
            email = "test@example.com",
            username = "testuser",
            password = "password123"
        )
        val user = com.example.entity.User(id = 1L, email = dto.email, username = dto.username, passwordHash = "hashed")

        every { userRepository.existsByEmail(dto.email) } returns false
        every { passwordEncoder.encode(dto.password) } returns "hashed"
        every { userRepository.save(any()) } returns user
        justRun { eventPublisher.publishEvent(any()) }

        userService.register(dto)

        verify {
            eventPublisher.publishEvent(match { event ->
                event is UserRegisteredEvent && event.user.email == dto.email
            })
        }
    }

    @Test
    fun `should not publish event when registration fails`() {
        val dto = RegisterDto("existing@example.com", "user", "password")
        every { userRepository.existsByEmail(dto.email) } returns true

        org.junit.jupiter.api.assertThrows<IllegalArgumentException> {
            userService.register(dto)
        }

        verify(exactly = 0) { eventPublisher.publishEvent(any()) }
    }
}
```

---

## 📋 สรุป

| Annotation | Phase | Use Case |
|-----------|-------|---------|
| `@EventListener` | Synchronous (same TX) | Default |
| `@EventListener` + `@Async` | Async (new thread) | Email, notifications |
| `@TransactionalEventListener(AFTER_COMMIT)` | After TX commit | ส่ง email หลัง save สำเร็จ |
| `@TransactionalEventListener(AFTER_ROLLBACK)` | After TX rollback | Log failures |
| `@TransactionalEventListener(BEFORE_COMMIT)` | Before TX commit | Final validation |

### Event Patterns Comparison

| Pattern | ข้อดี | ข้อเสีย |
|---------|-------|---------|
| Spring Application Events | ง่าย, in-process | ไม่ข้าม service |
| Domain Events (AbstractAggregateRoot) | Clean, DDD-friendly | ต้องการ Spring Data |
| Message Queue (RabbitMQ/Kafka) | ข้าม service ได้, durable | ซับซ้อนกว่า |

---

*Part 38/100+ | Kotlin & Spring Boot Complete Course*
