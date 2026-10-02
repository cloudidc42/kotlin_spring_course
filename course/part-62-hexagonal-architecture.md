# Part 62: Hexagonal Architecture - Payment Service

## บทนำ

**Hexagonal Architecture** (หรือ Ports and Adapters Architecture) คือ pattern ที่ Alistair Cockburn เสนอในปี 2005 หลักการคือแยก application core ออกจาก external world ผ่าน **Ports** (interfaces) และ **Adapters** (implementations)

## เปรียบเทียบกับ Clean Architecture

| ลักษณะ | Clean Architecture | Hexagonal Architecture |
|--------|-------------------|----------------------|
| Creator | Robert C. Martin | Alistair Cockburn |
| Layers | 4 concentric rings | Core + Ports + Adapters |
| Focus | Dependency direction | Inside-out isolation |
| Naming | Use Cases, Gateways | Ports, Adapters |
| Similarity | สูงมาก | เหมือนกันในหลักการ |

## Hexagonal Architecture Diagram

```
                    ┌───────────────────────────────────┐
                    │         External World             │
                    │                                    │
   REST Client      │   ┌─────────────────────────┐     │   Database
   ──────────▶      │   │     Primary Adapters     │     │
   gRPC Client      │   │  (Driving / Input Side)  │     │   Message Queue
   ──────────▶      │   └──────────┬──────────────┘     │
   CLI             │              │                      │   Payment Gateway
   ──────────▶      │   ┌──────────▼──────────────┐     │
                    │   │       Input Ports        │     │   Email Service
                    │   │    (Use Case Interfaces) │     │
                    │   └──────────┬──────────────┘     │
                    │              │                      │
                    │   ┌──────────▼──────────────┐     │
                    │   │     Application Core     │     │
                    │   │   (Business Logic)       │     │
                    │   └──────────┬──────────────┘     │
                    │              │                      │
                    │   ┌──────────▼──────────────┐     │
                    │   │      Output Ports        │     │
                    │   │  (Repository Interfaces) │     │
                    │   └──────────┬──────────────┘     │
                    │              │                      │
                    │   ┌──────────▼──────────────┐     │
                    │   │    Secondary Adapters    │     │
                    │   │  (Driven / Output Side)  │     │
                    │   └─────────────────────────┘     │
                    └───────────────────────────────────┘
```

## Payment Service โครงสร้าง

```
payment-service/
├── core/
│   ├── domain/
│   │   ├── Payment.kt
│   │   ├── PaymentMethod.kt
│   │   └── exception/
│   ├── port/
│   │   ├── input/           ← Primary Ports (Use Cases)
│   │   │   ├── ProcessPaymentUseCase.kt
│   │   │   ├── RefundPaymentUseCase.kt
│   │   │   └── GetPaymentStatusUseCase.kt
│   │   └── output/          ← Secondary Ports (Dependencies)
│   │       ├── PaymentRepository.kt
│   │       ├── PaymentGatewayPort.kt
│   │       └── NotificationPort.kt
│   └── service/             ← Core Business Logic
│       └── PaymentService.kt
├── adapter/
│   ├── input/               ← Primary Adapters
│   │   ├── rest/
│   │   │   └── PaymentController.kt
│   │   └── messaging/
│   │       └── PaymentEventListener.kt
│   └── output/              ← Secondary Adapters
│       ├── persistence/
│       │   └── PaymentJpaAdapter.kt
│       ├── gateway/
│       │   ├── StripeGatewayAdapter.kt
│       │   └── OmiseGatewayAdapter.kt
│       └── notification/
│           └── EmailNotificationAdapter.kt
└── config/
    └── PaymentConfig.kt
```

## Domain Models

```kotlin
// core/domain/Payment.kt
package com.payment.hex.core.domain

import java.math.BigDecimal
import java.time.LocalDateTime
import java.util.UUID

data class Payment(
    val id: PaymentId,
    val orderId: OrderId,
    val customerId: CustomerId,
    val amount: Amount,
    val method: PaymentMethod,
    val status: PaymentStatus,
    val gatewayTransactionId: String? = null,
    val failureReason: String? = null,
    val createdAt: LocalDateTime,
    val updatedAt: LocalDateTime
) {
    fun isSuccessful(): Boolean = status == PaymentStatus.COMPLETED
    fun isFailed(): Boolean = status == PaymentStatus.FAILED
    fun canBeRefunded(): Boolean = status == PaymentStatus.COMPLETED

    fun complete(gatewayTransactionId: String): Payment = copy(
        status = PaymentStatus.COMPLETED,
        gatewayTransactionId = gatewayTransactionId,
        updatedAt = LocalDateTime.now()
    )

    fun fail(reason: String): Payment = copy(
        status = PaymentStatus.FAILED,
        failureReason = reason,
        updatedAt = LocalDateTime.now()
    )

    fun refund(): Payment {
        require(canBeRefunded()) { "Payment cannot be refunded in status: $status" }
        return copy(
            status = PaymentStatus.REFUNDED,
            updatedAt = LocalDateTime.now()
        )
    }

    companion object {
        fun create(
            orderId: OrderId,
            customerId: CustomerId,
            amount: Amount,
            method: PaymentMethod
        ): Payment {
            val now = LocalDateTime.now()
            return Payment(
                id = PaymentId(UUID.randomUUID().toString()),
                orderId = orderId,
                customerId = customerId,
                amount = amount,
                method = method,
                status = PaymentStatus.PENDING,
                createdAt = now,
                updatedAt = now
            )
        }
    }
}

enum class PaymentStatus {
    PENDING, PROCESSING, COMPLETED, FAILED, REFUNDED, CANCELLED
}

@JvmInline value class PaymentId(val value: String)
@JvmInline value class OrderId(val value: String)
@JvmInline value class CustomerId(val value: String)

data class Amount(val value: BigDecimal, val currency: String = "THB") {
    init {
        require(value > BigDecimal.ZERO) { "Amount must be positive" }
    }
}

sealed class PaymentMethod {
    data class CreditCard(
        val cardNumber: String,
        val expiryMonth: Int,
        val expiryYear: Int,
        val cvv: String
    ) : PaymentMethod()
    
    data class BankTransfer(
        val bankCode: String,
        val accountNumber: String
    ) : PaymentMethod()
    
    data object PromptPay : PaymentMethod()
    
    data class QRCode(val provider: String) : PaymentMethod()
}
```

## Input Ports (Primary Ports)

```kotlin
// core/port/input/ProcessPaymentUseCase.kt
package com.payment.hex.core.port.input

import com.payment.hex.core.domain.*

interface ProcessPaymentUseCase {
    fun processPayment(command: ProcessPaymentCommand): PaymentResult
}

data class ProcessPaymentCommand(
    val orderId: String,
    val customerId: String,
    val amount: java.math.BigDecimal,
    val currency: String = "THB",
    val paymentMethod: PaymentMethodCommand
)

sealed class PaymentMethodCommand {
    data class CreditCard(
        val cardNumber: String,
        val expiryMonth: Int,
        val expiryYear: Int,
        val cvv: String
    ) : PaymentMethodCommand()
    
    data class BankTransfer(val bankCode: String, val accountNumber: String) : PaymentMethodCommand()
    data object PromptPay : PaymentMethodCommand()
}

data class PaymentResult(
    val paymentId: String,
    val status: String,
    val gatewayTransactionId: String? = null,
    val message: String? = null
)
```

```kotlin
// core/port/input/RefundPaymentUseCase.kt
package com.payment.hex.core.port.input

import com.payment.hex.core.domain.PaymentResult

interface RefundPaymentUseCase {
    fun refundPayment(command: RefundPaymentCommand): PaymentResult
}

data class RefundPaymentCommand(
    val paymentId: String,
    val reason: String,
    val requestedBy: String
)
```

```kotlin
// core/port/input/GetPaymentStatusUseCase.kt
package com.payment.hex.core.port.input

import com.payment.hex.core.domain.Payment

interface GetPaymentStatusUseCase {
    fun getPayment(paymentId: String): Payment?
    fun getPaymentsByOrder(orderId: String): List<Payment>
    fun getPaymentsByCustomer(customerId: String): List<Payment>
}
```

## Output Ports (Secondary Ports)

```kotlin
// core/port/output/PaymentRepository.kt
package com.payment.hex.core.port.output

import com.payment.hex.core.domain.*

interface PaymentRepository {
    fun save(payment: Payment): Payment
    fun findById(id: PaymentId): Payment?
    fun findByOrderId(orderId: OrderId): List<Payment>
    fun findByCustomerId(customerId: CustomerId): List<Payment>
    fun findByStatus(status: PaymentStatus): List<Payment>
}
```

```kotlin
// core/port/output/PaymentGatewayPort.kt
package com.payment.hex.core.port.output

import com.payment.hex.core.domain.*

interface PaymentGatewayPort {
    fun charge(request: ChargeRequest): GatewayResponse
    fun refund(transactionId: String, amount: Amount): GatewayResponse
    fun getTransactionStatus(transactionId: String): GatewayTransactionStatus
}

data class ChargeRequest(
    val amount: Amount,
    val paymentMethod: PaymentMethod,
    val metadata: Map<String, String> = emptyMap()
)

data class GatewayResponse(
    val success: Boolean,
    val transactionId: String? = null,
    val errorCode: String? = null,
    val errorMessage: String? = null
)

enum class GatewayTransactionStatus {
    PENDING, SUCCESS, FAILED, REFUNDED
}
```

```kotlin
// core/port/output/NotificationPort.kt
package com.payment.hex.core.port.output

interface NotificationPort {
    fun notifyPaymentSuccess(customerId: String, amount: java.math.BigDecimal, currency: String)
    fun notifyPaymentFailure(customerId: String, reason: String)
    fun notifyRefundProcessed(customerId: String, amount: java.math.BigDecimal)
}
```

## Application Core (Business Logic)

```kotlin
// core/service/PaymentService.kt
package com.payment.hex.core.service

import com.payment.hex.core.domain.*
import com.payment.hex.core.port.input.*
import com.payment.hex.core.port.output.*
import org.slf4j.LoggerFactory

class PaymentService(
    private val paymentRepository: PaymentRepository,
    private val paymentGateway: PaymentGatewayPort,
    private val notification: NotificationPort
) : ProcessPaymentUseCase, RefundPaymentUseCase, GetPaymentStatusUseCase {

    private val logger = LoggerFactory.getLogger(PaymentService::class.java)

    // ===== ProcessPaymentUseCase =====

    override fun processPayment(command: ProcessPaymentCommand): PaymentResult {
        logger.info("Processing payment for order ${command.orderId}")

        val paymentMethod = mapPaymentMethod(command.paymentMethod)
        val amount = Amount(command.amount, command.currency)

        val payment = Payment.create(
            orderId = OrderId(command.orderId),
            customerId = CustomerId(command.customerId),
            amount = amount,
            method = paymentMethod
        )

        val savedPayment = paymentRepository.save(payment)

        return try {
            val gatewayResponse = paymentGateway.charge(
                ChargeRequest(amount, paymentMethod, mapOf(
                    "orderId" to command.orderId,
                    "customerId" to command.customerId
                ))
            )

            if (gatewayResponse.success && gatewayResponse.transactionId != null) {
                val completedPayment = savedPayment.complete(gatewayResponse.transactionId)
                val finalPayment = paymentRepository.save(completedPayment)

                notification.notifyPaymentSuccess(
                    command.customerId,
                    command.amount,
                    command.currency
                )

                PaymentResult(
                    paymentId = finalPayment.id.value,
                    status = finalPayment.status.name,
                    gatewayTransactionId = gatewayResponse.transactionId,
                    message = "Payment processed successfully"
                )
            } else {
                val failedPayment = savedPayment.fail(
                    gatewayResponse.errorMessage ?: "Payment gateway error"
                )
                paymentRepository.save(failedPayment)

                notification.notifyPaymentFailure(
                    command.customerId,
                    gatewayResponse.errorMessage ?: "Unknown error"
                )

                PaymentResult(
                    paymentId = failedPayment.id.value,
                    status = failedPayment.status.name,
                    message = gatewayResponse.errorMessage
                )
            }
        } catch (e: Exception) {
            logger.error("Payment gateway error", e)
            val failedPayment = savedPayment.fail("Internal payment error")
            paymentRepository.save(failedPayment)
            throw PaymentProcessingException("Failed to process payment", e)
        }
    }

    // ===== RefundPaymentUseCase =====

    override fun refundPayment(command: RefundPaymentCommand): PaymentResult {
        val payment = paymentRepository.findById(PaymentId(command.paymentId))
            ?: throw PaymentNotFoundException("Payment ${command.paymentId} not found")

        if (!payment.canBeRefunded()) {
            throw PaymentRefundException("Payment cannot be refunded in status: ${payment.status}")
        }

        val gatewayResponse = payment.gatewayTransactionId?.let { transactionId ->
            paymentGateway.refund(transactionId, payment.amount)
        } ?: GatewayResponse(success = false, errorMessage = "No transaction ID")

        return if (gatewayResponse.success) {
            val refundedPayment = payment.refund()
            paymentRepository.save(refundedPayment)
            
            notification.notifyRefundProcessed(
                payment.customerId.value,
                payment.amount.value
            )

            PaymentResult(
                paymentId = payment.id.value,
                status = refundedPayment.status.name,
                message = "Refund processed successfully"
            )
        } else {
            throw PaymentRefundException("Refund failed: ${gatewayResponse.errorMessage}")
        }
    }

    // ===== GetPaymentStatusUseCase =====

    override fun getPayment(paymentId: String): Payment? {
        return paymentRepository.findById(PaymentId(paymentId))
    }

    override fun getPaymentsByOrder(orderId: String): List<Payment> {
        return paymentRepository.findByOrderId(OrderId(orderId))
    }

    override fun getPaymentsByCustomer(customerId: String): List<Payment> {
        return paymentRepository.findByCustomerId(CustomerId(customerId))
    }

    // ===== Helper =====

    private fun mapPaymentMethod(command: PaymentMethodCommand): PaymentMethod {
        return when (command) {
            is PaymentMethodCommand.CreditCard -> PaymentMethod.CreditCard(
                command.cardNumber, command.expiryMonth, command.expiryYear, command.cvv
            )
            is PaymentMethodCommand.BankTransfer -> PaymentMethod.BankTransfer(
                command.bankCode, command.accountNumber
            )
            PaymentMethodCommand.PromptPay -> PaymentMethod.PromptPay
        }
    }
}
```

## Primary Adapters (Input)

```kotlin
// adapter/input/rest/PaymentController.kt
package com.payment.hex.adapter.input.rest

import com.payment.hex.core.port.input.*
import org.springframework.http.ResponseEntity
import org.springframework.web.bind.annotation.*

@RestController
@RequestMapping("/api/payments")
class PaymentController(
    private val processPaymentUseCase: ProcessPaymentUseCase,
    private val refundPaymentUseCase: RefundPaymentUseCase,
    private val getPaymentStatusUseCase: GetPaymentStatusUseCase
) {

    @PostMapping
    fun processPayment(@RequestBody request: ProcessPaymentRequest): ResponseEntity<PaymentResponse> {
        val command = ProcessPaymentCommand(
            orderId = request.orderId,
            customerId = request.customerId,
            amount = request.amount,
            currency = request.currency,
            paymentMethod = request.method.toCommand()
        )
        val result = processPaymentUseCase.processPayment(command)
        return ResponseEntity.ok(result.toResponse())
    }

    @PostMapping("/{paymentId}/refund")
    fun refundPayment(
        @PathVariable paymentId: String,
        @RequestBody request: RefundRequest
    ): ResponseEntity<PaymentResponse> {
        val result = refundPaymentUseCase.refundPayment(
            RefundPaymentCommand(paymentId, request.reason, request.requestedBy)
        )
        return ResponseEntity.ok(result.toResponse())
    }

    @GetMapping("/{paymentId}")
    fun getPayment(@PathVariable paymentId: String): ResponseEntity<PaymentDetailResponse> {
        val payment = getPaymentStatusUseCase.getPayment(paymentId)
            ?: return ResponseEntity.notFound().build()
        return ResponseEntity.ok(payment.toDetailResponse())
    }
}

// REST DTOs
data class ProcessPaymentRequest(
    val orderId: String,
    val customerId: String,
    val amount: java.math.BigDecimal,
    val currency: String = "THB",
    val method: PaymentMethodRequest
)

data class PaymentMethodRequest(
    val type: String,
    val cardNumber: String? = null,
    val expiryMonth: Int? = null,
    val expiryYear: Int? = null,
    val cvv: String? = null,
    val bankCode: String? = null,
    val accountNumber: String? = null
) {
    fun toCommand(): PaymentMethodCommand = when (type) {
        "CREDIT_CARD" -> PaymentMethodCommand.CreditCard(
            cardNumber!!, expiryMonth!!, expiryYear!!, cvv!!
        )
        "BANK_TRANSFER" -> PaymentMethodCommand.BankTransfer(bankCode!!, accountNumber!!)
        "PROMPT_PAY" -> PaymentMethodCommand.PromptPay
        else -> throw IllegalArgumentException("Unknown payment method: $type")
    }
}
```

## Secondary Adapters (Output)

```kotlin
// adapter/output/gateway/StripeGatewayAdapter.kt
package com.payment.hex.adapter.output.gateway

import com.payment.hex.core.domain.*
import com.payment.hex.core.port.output.*
import org.springframework.beans.factory.annotation.Value
import org.springframework.stereotype.Component

@Component
class StripeGatewayAdapter(
    @Value("\${stripe.api.key}") private val apiKey: String
) : PaymentGatewayPort {

    override fun charge(request: ChargeRequest): GatewayResponse {
        return try {
            // ในโปรเจกต์จริงจะใช้ Stripe SDK
            // val stripe = Stripe.charge(...)
            
            when (request.paymentMethod) {
                is PaymentMethod.CreditCard -> processCardPayment(request)
                is PaymentMethod.BankTransfer -> processBankTransfer(request)
                PaymentMethod.PromptPay -> processPromptPay(request)
                else -> GatewayResponse(success = false, errorMessage = "Unsupported payment method")
            }
        } catch (e: Exception) {
            GatewayResponse(success = false, errorMessage = e.message)
        }
    }

    override fun refund(transactionId: String, amount: Amount): GatewayResponse {
        return try {
            // Stripe refund logic
            GatewayResponse(success = true, transactionId = "refund_${transactionId}")
        } catch (e: Exception) {
            GatewayResponse(success = false, errorMessage = e.message)
        }
    }

    override fun getTransactionStatus(transactionId: String): GatewayTransactionStatus {
        // Check Stripe transaction status
        return GatewayTransactionStatus.SUCCESS
    }

    private fun processCardPayment(request: ChargeRequest): GatewayResponse {
        val card = request.paymentMethod as PaymentMethod.CreditCard
        // Stripe credit card charging logic
        return GatewayResponse(
            success = true,
            transactionId = "stripe_${System.currentTimeMillis()}"
        )
    }

    private fun processBankTransfer(request: ChargeRequest): GatewayResponse {
        return GatewayResponse(
            success = true,
            transactionId = "bank_${System.currentTimeMillis()}"
        )
    }

    private fun processPromptPay(request: ChargeRequest): GatewayResponse {
        return GatewayResponse(
            success = true,
            transactionId = "promptpay_${System.currentTimeMillis()}"
        )
    }
}
```

```kotlin
// adapter/output/persistence/PaymentJpaAdapter.kt
package com.payment.hex.adapter.output.persistence

import com.payment.hex.core.domain.*
import com.payment.hex.core.port.output.PaymentRepository
import jakarta.persistence.*
import org.springframework.data.jpa.repository.JpaRepository
import org.springframework.stereotype.Repository
import java.math.BigDecimal
import java.time.LocalDateTime

@Repository
class PaymentJpaAdapter(
    private val jpaRepository: PaymentJpaRepository
) : PaymentRepository {

    override fun save(payment: Payment): Payment {
        return jpaRepository.save(payment.toEntity()).toDomain()
    }

    override fun findById(id: PaymentId): Payment? {
        return jpaRepository.findById(id.value).map { it.toDomain() }.orElse(null)
    }

    override fun findByOrderId(orderId: OrderId): List<Payment> {
        return jpaRepository.findByOrderId(orderId.value).map { it.toDomain() }
    }

    override fun findByCustomerId(customerId: CustomerId): List<Payment> {
        return jpaRepository.findByCustomerId(customerId.value).map { it.toDomain() }
    }

    override fun findByStatus(status: PaymentStatus): List<Payment> {
        return jpaRepository.findByStatus(status.name).map { it.toDomain() }
    }
}

@Entity
@Table(name = "payments")
data class PaymentEntity(
    @Id val id: String,
    val orderId: String,
    val customerId: String,
    val amount: BigDecimal,
    val currency: String,
    val method: String,
    val status: String,
    val gatewayTransactionId: String?,
    val failureReason: String?,
    val createdAt: LocalDateTime,
    val updatedAt: LocalDateTime
)

interface PaymentJpaRepository : JpaRepository<PaymentEntity, String> {
    fun findByOrderId(orderId: String): List<PaymentEntity>
    fun findByCustomerId(customerId: String): List<PaymentEntity>
    fun findByStatus(status: String): List<PaymentEntity>
}

// Mapping functions
fun Payment.toEntity() = PaymentEntity(
    id = id.value,
    orderId = orderId.value,
    customerId = customerId.value,
    amount = amount.value,
    currency = amount.currency,
    method = method::class.simpleName ?: "UNKNOWN",
    status = status.name,
    gatewayTransactionId = gatewayTransactionId,
    failureReason = failureReason,
    createdAt = createdAt,
    updatedAt = updatedAt
)

fun PaymentEntity.toDomain() = Payment(
    id = PaymentId(id),
    orderId = OrderId(orderId),
    customerId = CustomerId(customerId),
    amount = Amount(amount, currency),
    method = PaymentMethod.PromptPay, // simplified
    status = PaymentStatus.valueOf(status),
    gatewayTransactionId = gatewayTransactionId,
    failureReason = failureReason,
    createdAt = createdAt,
    updatedAt = updatedAt
)
```

## Testing

### Testing Application Core (ไม่ต้องการ Spring)

```kotlin
// PaymentServiceTest.kt
package com.payment.hex.core.service

import com.payment.hex.core.domain.*
import com.payment.hex.core.port.input.*
import com.payment.hex.core.port.output.*
import io.mockk.*
import org.junit.jupiter.api.Test
import org.junit.jupiter.api.assertThrows
import java.math.BigDecimal

class PaymentServiceTest {

    private val paymentRepository = mockk<PaymentRepository>()
    private val paymentGateway = mockk<PaymentGatewayPort>()
    private val notification = mockk<NotificationPort>(relaxed = true)

    private val paymentService = PaymentService(
        paymentRepository, paymentGateway, notification
    )

    @Test
    fun `should process payment successfully`() {
        val command = ProcessPaymentCommand(
            orderId = "order-1",
            customerId = "customer-1",
            amount = BigDecimal("500.00"),
            paymentMethod = PaymentMethodCommand.PromptPay
        )

        every { paymentRepository.save(any()) } answers { firstArg() }
        every { paymentGateway.charge(any()) } returns GatewayResponse(
            success = true,
            transactionId = "stripe_123"
        )

        val result = paymentService.processPayment(command)

        assert(result.status == "COMPLETED")
        assert(result.gatewayTransactionId == "stripe_123")
        verify { notification.notifyPaymentSuccess(any(), any(), any()) }
    }

    @Test
    fun `should handle gateway failure gracefully`() {
        every { paymentRepository.save(any()) } answers { firstArg() }
        every { paymentGateway.charge(any()) } returns GatewayResponse(
            success = false,
            errorMessage = "Card declined"
        )

        val result = paymentService.processPayment(ProcessPaymentCommand(
            orderId = "order-1",
            customerId = "customer-1",
            amount = BigDecimal("500.00"),
            paymentMethod = PaymentMethodCommand.PromptPay
        ))

        assert(result.status == "FAILED")
        verify { notification.notifyPaymentFailure(any(), any()) }
    }
}
```

## Bean Configuration

```kotlin
// config/PaymentConfig.kt
package com.payment.hex.config

import com.payment.hex.core.port.output.*
import com.payment.hex.core.service.PaymentService
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration

@Configuration
class PaymentConfig {

    @Bean
    fun paymentService(
        paymentRepository: PaymentRepository,
        paymentGateway: PaymentGatewayPort,
        notification: NotificationPort
    ): PaymentService {
        return PaymentService(paymentRepository, paymentGateway, notification)
    }
}
```

## สรุป Hexagonal Architecture

| Component | ความรับผิดชอบ | ตัวอย่าง |
|-----------|-------------|---------|
| Application Core | Business logic ทั้งหมด | PaymentService |
| Primary Port | Use case interfaces | ProcessPaymentUseCase |
| Secondary Port | External dependency interfaces | PaymentRepository, PaymentGatewayPort |
| Primary Adapter | ส่ง input เข้า core | REST Controller, CLI |
| Secondary Adapter | Implement external deps | JPA, Stripe, Email |

## ข้อดีของ Hexagonal Architecture

1. **Isolation** - Core ไม่รู้จัก infrastructure เลย
2. **Testability** - ทดสอบ core ได้ง่ายด้วย mock adapters
3. **Flexibility** - เปลี่ยน adapter ได้โดยไม่กระทบ core
4. **Multiple Interfaces** - ใช้ core เดียวกันได้ทั้ง REST, gRPC, CLI
5. **Clear Boundaries** - รู้ชัดเจนว่าอะไรอยู่ที่ไหน

*Part 62/100+ | Kotlin & Spring Boot Complete Course*
