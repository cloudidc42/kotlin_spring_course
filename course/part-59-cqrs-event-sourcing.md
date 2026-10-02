# Part 59: CQRS และ Event Sourcing - Order Management

## บทนำ

**CQRS (Command Query Responsibility Segregation)** คือ pattern ที่แยก "การเขียนข้อมูล" (Command) ออกจาก "การอ่านข้อมูล" (Query) เป็นคนละ model และ **Event Sourcing** คือ pattern ที่บันทึก state ของ application ในรูปแบบของ sequence of events แทนที่จะบันทึก current state

## ปัญหาที่ CQRS แก้ไข

### Traditional Architecture

```
Database:
   Write         Read
     ↕              ↕
[   Single Model & Database   ]
```

ปัญหา:
- Read model ต้องการ denormalized data แต่ Write model ต้องการ normalized data
- การ optimize read กระทบ write performance
- Complex queries ทำให้ write ช้าลง

### CQRS Architecture

```
Commands → Command Handler → Write Database
                                    ↓
                              Domain Events → Event Bus
                                                    ↓
                                              Event Handler → Read Database (Optimized)
                                              
Queries → Query Handler → Read Database (Fast)
```

## Event Sourcing Concept

```
Traditional: State = Current Snapshot
Event Sourcing: State = Replay of Events

Order Events:
1. OrderCreated    { orderId, userId, items }
2. OrderConfirmed  { orderId, confirmedAt }
3. OrderShipped    { orderId, trackingNumber }
4. OrderDelivered  { orderId, deliveredAt }

Current State = Apply(OrderCreated + OrderConfirmed + OrderShipped + OrderDelivered)
```

## การตั้งค่าโปรเจกต์

```kotlin
// build.gradle.kts
dependencies {
    // Axon Framework
    implementation("org.axonframework:axon-spring-boot-starter:4.9.1")
    implementation("org.axonframework.extensions.kotlin:axon-kotlin-extension:4.9.0")
    
    // JPA
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
    
    // Web
    implementation("org.springframework.boot:spring-boot-starter-web")
    
    // Kotlin
    implementation("org.jetbrains.kotlin:kotlin-reflect")
    implementation("com.fasterxml.jackson.module:jackson-module-kotlin")
}
```

## Domain Events

```kotlin
// OrderEvents.kt
package com.order.cqrs.event

import java.math.BigDecimal
import java.time.LocalDateTime

// Base Event class
sealed class OrderEvent {
    abstract val orderId: String
    abstract val occurredAt: LocalDateTime
}

data class OrderCreatedEvent(
    override val orderId: String,
    val userId: String,
    val items: List<OrderItemData>,
    val shippingAddress: String,
    val totalAmount: BigDecimal,
    override val occurredAt: LocalDateTime = LocalDateTime.now()
) : OrderEvent()

data class OrderConfirmedEvent(
    override val orderId: String,
    val confirmedBy: String,
    override val occurredAt: LocalDateTime = LocalDateTime.now()
) : OrderEvent()

data class OrderShippedEvent(
    override val orderId: String,
    val trackingNumber: String,
    val carrier: String,
    override val occurredAt: LocalDateTime = LocalDateTime.now()
) : OrderEvent()

data class OrderDeliveredEvent(
    override val orderId: String,
    val deliveredAt: LocalDateTime,
    override val occurredAt: LocalDateTime = LocalDateTime.now()
) : OrderEvent()

data class OrderCancelledEvent(
    override val orderId: String,
    val reason: String,
    val cancelledBy: String,
    override val occurredAt: LocalDateTime = LocalDateTime.now()
) : OrderEvent()

data class OrderItemAddedEvent(
    override val orderId: String,
    val item: OrderItemData,
    override val occurredAt: LocalDateTime = LocalDateTime.now()
) : OrderEvent()

data class OrderItemData(
    val productId: String,
    val productName: String,
    val quantity: Int,
    val unitPrice: BigDecimal
)
```

## Commands

```kotlin
// OrderCommands.kt
package com.order.cqrs.command

import jakarta.validation.constraints.NotBlank
import jakarta.validation.constraints.NotEmpty
import java.math.BigDecimal

// Commands are requests to change state
data class CreateOrderCommand(
    val orderId: String = java.util.UUID.randomUUID().toString(),
    @field:NotBlank val userId: String,
    @field:NotEmpty val items: List<OrderItemCommand>,
    @field:NotBlank val shippingAddress: String
)

data class ConfirmOrderCommand(
    val orderId: String,
    val confirmedBy: String
)

data class ShipOrderCommand(
    val orderId: String,
    val trackingNumber: String,
    val carrier: String
)

data class DeliverOrderCommand(
    val orderId: String
)

data class CancelOrderCommand(
    val orderId: String,
    val reason: String,
    val cancelledBy: String
)

data class OrderItemCommand(
    val productId: String,
    val quantity: Int,
    val unitPrice: BigDecimal
)
```

## Command Side - Aggregate

```kotlin
// OrderAggregate.kt (Axon Framework)
package com.order.cqrs.aggregate

import com.order.cqrs.command.*
import com.order.cqrs.event.*
import org.axonframework.commandhandling.CommandHandler
import org.axonframework.eventsourcing.EventSourcingHandler
import org.axonframework.modelling.command.AggregateIdentifier
import org.axonframework.modelling.command.AggregateLifecycle.apply
import org.axonframework.spring.stereotype.Aggregate
import java.math.BigDecimal
import java.time.LocalDateTime

@Aggregate
class OrderAggregate {

    @AggregateIdentifier
    private lateinit var orderId: String
    private lateinit var userId: String
    private lateinit var status: OrderStatus
    private var items: MutableList<OrderItemData> = mutableListOf()
    private var totalAmount: BigDecimal = BigDecimal.ZERO

    // No-arg constructor required by Axon
    constructor()

    @CommandHandler
    constructor(command: CreateOrderCommand) {
        // Validation
        require(command.items.isNotEmpty()) { "Order must have at least one item" }
        
        val total = command.items.sumOf { 
            it.unitPrice * it.quantity.toBigDecimal() 
        }

        // Apply event (don't set state directly!)
        apply(OrderCreatedEvent(
            orderId = command.orderId,
            userId = command.userId,
            items = command.items.map { 
                OrderItemData(it.productId, "", it.quantity, it.unitPrice) 
            },
            shippingAddress = command.shippingAddress,
            totalAmount = total
        ))
    }

    @CommandHandler
    fun handle(command: ConfirmOrderCommand) {
        check(status == OrderStatus.PENDING) { 
            "Cannot confirm order in status: $status" 
        }
        apply(OrderConfirmedEvent(
            orderId = orderId,
            confirmedBy = command.confirmedBy
        ))
    }

    @CommandHandler
    fun handle(command: ShipOrderCommand) {
        check(status == OrderStatus.CONFIRMED) { 
            "Cannot ship order in status: $status" 
        }
        apply(OrderShippedEvent(
            orderId = orderId,
            trackingNumber = command.trackingNumber,
            carrier = command.carrier
        ))
    }

    @CommandHandler
    fun handle(command: DeliverOrderCommand) {
        check(status == OrderStatus.SHIPPED) { 
            "Cannot deliver order in status: $status" 
        }
        apply(OrderDeliveredEvent(
            orderId = orderId,
            deliveredAt = LocalDateTime.now()
        ))
    }

    @CommandHandler
    fun handle(command: CancelOrderCommand) {
        check(status in listOf(OrderStatus.PENDING, OrderStatus.CONFIRMED)) {
            "Cannot cancel order in status: $status"
        }
        apply(OrderCancelledEvent(
            orderId = orderId,
            reason = command.reason,
            cancelledBy = command.cancelledBy
        ))
    }

    // ===== Event Sourcing Handlers =====
    // These rebuild state from events

    @EventSourcingHandler
    fun on(event: OrderCreatedEvent) {
        this.orderId = event.orderId
        this.userId = event.userId
        this.items = event.items.toMutableList()
        this.totalAmount = event.totalAmount
        this.status = OrderStatus.PENDING
    }

    @EventSourcingHandler
    fun on(event: OrderConfirmedEvent) {
        this.status = OrderStatus.CONFIRMED
    }

    @EventSourcingHandler
    fun on(event: OrderShippedEvent) {
        this.status = OrderStatus.SHIPPED
    }

    @EventSourcingHandler
    fun on(event: OrderDeliveredEvent) {
        this.status = OrderStatus.DELIVERED
    }

    @EventSourcingHandler
    fun on(event: OrderCancelledEvent) {
        this.status = OrderStatus.CANCELLED
    }
}

enum class OrderStatus {
    PENDING, CONFIRMED, SHIPPED, DELIVERED, CANCELLED
}
```

## Query Side - Projections

```kotlin
// OrderProjection.kt
package com.order.cqrs.projection

import com.order.cqrs.event.*
import com.order.cqrs.query.*
import com.order.cqrs.readmodel.*
import org.axonframework.eventhandling.EventHandler
import org.axonframework.queryhandling.QueryHandler
import org.springframework.stereotype.Component

@Component
class OrderProjection(
    private val orderReadModelRepository: OrderReadModelRepository
) {

    // ===== Event Handlers - Update Read Model =====

    @EventHandler
    fun on(event: OrderCreatedEvent) {
        val readModel = OrderReadModel(
            orderId = event.orderId,
            userId = event.userId,
            status = "PENDING",
            totalAmount = event.totalAmount,
            shippingAddress = event.shippingAddress,
            items = event.items.map { it.toReadModel() },
            createdAt = event.occurredAt
        )
        orderReadModelRepository.save(readModel)
    }

    @EventHandler
    fun on(event: OrderConfirmedEvent) {
        val order = orderReadModelRepository.findById(event.orderId)
            .orElseThrow { IllegalArgumentException("Order not found: ${event.orderId}") }
        
        orderReadModelRepository.save(order.copy(
            status = "CONFIRMED",
            confirmedAt = event.occurredAt
        ))
    }

    @EventHandler
    fun on(event: OrderShippedEvent) {
        val order = orderReadModelRepository.findById(event.orderId)
            .orElseThrow { IllegalArgumentException("Order not found") }
        
        orderReadModelRepository.save(order.copy(
            status = "SHIPPED",
            trackingNumber = event.trackingNumber,
            carrier = event.carrier,
            shippedAt = event.occurredAt
        ))
    }

    @EventHandler
    fun on(event: OrderDeliveredEvent) {
        val order = orderReadModelRepository.findById(event.orderId)
            .orElseThrow { IllegalArgumentException("Order not found") }
        
        orderReadModelRepository.save(order.copy(
            status = "DELIVERED",
            deliveredAt = event.deliveredAt
        ))
    }

    @EventHandler
    fun on(event: OrderCancelledEvent) {
        val order = orderReadModelRepository.findById(event.orderId)
            .orElseThrow { IllegalArgumentException("Order not found") }
        
        orderReadModelRepository.save(order.copy(
            status = "CANCELLED",
            cancellationReason = event.reason,
            cancelledAt = event.occurredAt
        ))
    }

    // ===== Query Handlers - Answer Queries =====

    @QueryHandler
    fun handle(query: FindOrderByIdQuery): OrderReadModel? {
        return orderReadModelRepository.findById(query.orderId).orElse(null)
    }

    @QueryHandler
    fun handle(query: FindOrdersByUserIdQuery): List<OrderReadModel> {
        return orderReadModelRepository.findByUserId(query.userId)
    }

    @QueryHandler
    fun handle(query: FindOrdersByStatusQuery): List<OrderReadModel> {
        return orderReadModelRepository.findByStatus(query.status)
    }

    @QueryHandler
    fun handle(query: OrderStatisticsQuery): OrderStatistics {
        return OrderStatistics(
            totalOrders = orderReadModelRepository.count(),
            pendingOrders = orderReadModelRepository.countByStatus("PENDING"),
            totalRevenue = orderReadModelRepository.sumTotalAmount()
        )
    }
}
```

## Read Models

```kotlin
// OrderReadModel.kt
package com.order.cqrs.readmodel

import jakarta.persistence.*
import org.hibernate.annotations.Type
import java.math.BigDecimal
import java.time.LocalDateTime

@Entity
@Table(name = "order_read_models")
data class OrderReadModel(
    @Id
    val orderId: String,
    
    val userId: String,
    val status: String,
    val totalAmount: BigDecimal,
    val shippingAddress: String,
    
    @ElementCollection
    @CollectionTable(name = "order_item_read_models")
    val items: List<OrderItemReadModel> = emptyList(),
    
    val trackingNumber: String? = null,
    val carrier: String? = null,
    val cancellationReason: String? = null,
    
    val createdAt: LocalDateTime,
    val confirmedAt: LocalDateTime? = null,
    val shippedAt: LocalDateTime? = null,
    val deliveredAt: LocalDateTime? = null,
    val cancelledAt: LocalDateTime? = null
)

@Embeddable
data class OrderItemReadModel(
    val productId: String,
    val productName: String,
    val quantity: Int,
    val unitPrice: BigDecimal,
    val totalPrice: BigDecimal = unitPrice * quantity.toBigDecimal()
)

data class OrderStatistics(
    val totalOrders: Long,
    val pendingOrders: Long,
    val totalRevenue: BigDecimal
)
```

## Queries

```kotlin
// OrderQueries.kt
package com.order.cqrs.query

data class FindOrderByIdQuery(val orderId: String)
data class FindOrdersByUserIdQuery(val userId: String)
data class FindOrdersByStatusQuery(val status: String)
data class OrderStatisticsQuery(val fromDate: java.time.LocalDate? = null)
```

## API Controller

```kotlin
// OrderController.kt
package com.order.cqrs.controller

import com.order.cqrs.command.*
import com.order.cqrs.query.*
import com.order.cqrs.readmodel.*
import org.axonframework.commandhandling.gateway.CommandGateway
import org.axonframework.messaging.responsetypes.ResponseTypes
import org.axonframework.queryhandling.QueryGateway
import org.springframework.http.ResponseEntity
import org.springframework.web.bind.annotation.*
import java.util.concurrent.CompletableFuture

@RestController
@RequestMapping("/api/orders")
class OrderController(
    private val commandGateway: CommandGateway,
    private val queryGateway: QueryGateway
) {

    // ===== Commands =====

    @PostMapping
    fun createOrder(
        @RequestBody request: CreateOrderRequest
    ): CompletableFuture<ResponseEntity<Map<String, String>>> {
        val command = CreateOrderCommand(
            userId = request.userId,
            items = request.items.map { 
                OrderItemCommand(it.productId, it.quantity, it.unitPrice) 
            },
            shippingAddress = request.shippingAddress
        )

        return commandGateway
            .send<String>(command)
            .thenApply { orderId ->
                ResponseEntity.ok(mapOf("orderId" to orderId))
            }
    }

    @PutMapping("/{orderId}/confirm")
    fun confirmOrder(
        @PathVariable orderId: String,
        @RequestParam confirmedBy: String
    ): CompletableFuture<ResponseEntity<Void>> {
        return commandGateway
            .send<Void>(ConfirmOrderCommand(orderId, confirmedBy))
            .thenApply { ResponseEntity.ok<Void>(null) }
    }

    @PutMapping("/{orderId}/ship")
    fun shipOrder(
        @PathVariable orderId: String,
        @RequestBody request: ShipOrderRequest
    ): CompletableFuture<ResponseEntity<Void>> {
        return commandGateway
            .send<Void>(ShipOrderCommand(orderId, request.trackingNumber, request.carrier))
            .thenApply { ResponseEntity.ok<Void>(null) }
    }

    @PutMapping("/{orderId}/cancel")
    fun cancelOrder(
        @PathVariable orderId: String,
        @RequestBody request: CancelOrderRequest
    ): CompletableFuture<ResponseEntity<Void>> {
        return commandGateway
            .send<Void>(CancelOrderCommand(orderId, request.reason, request.cancelledBy))
            .thenApply { ResponseEntity.ok<Void>(null) }
    }

    // ===== Queries =====

    @GetMapping("/{orderId}")
    fun getOrder(@PathVariable orderId: String): CompletableFuture<OrderReadModel?> {
        return queryGateway.query(
            FindOrderByIdQuery(orderId),
            ResponseTypes.instanceOf(OrderReadModel::class.java)
        )
    }

    @GetMapping("/user/{userId}")
    fun getOrdersByUser(@PathVariable userId: String): CompletableFuture<List<OrderReadModel>> {
        return queryGateway.query(
            FindOrdersByUserIdQuery(userId),
            ResponseTypes.multipleInstancesOf(OrderReadModel::class.java)
        )
    }

    @GetMapping("/statistics")
    fun getStatistics(): CompletableFuture<OrderStatistics> {
        return queryGateway.query(
            OrderStatisticsQuery(),
            ResponseTypes.instanceOf(OrderStatistics::class.java)
        )
    }
}
```

## Event Store

```kotlin
// EventStoreInspector.kt - ดู event history ของ aggregate
package com.order.cqrs.service

import org.axonframework.eventsourcing.eventstore.EventStore
import org.springframework.stereotype.Service

@Service
class EventStoreInspector(
    private val eventStore: EventStore
) {

    fun getOrderHistory(orderId: String): List<Any> {
        return eventStore.readEvents(orderId)
            .asStream()
            .map { it.payload }
            .toList()
    }

    fun replayOrderState(orderId: String): String {
        val events = getOrderHistory(orderId)
        val statusTransitions = events.mapIndexed { index, event ->
            val statusText = when (event) {
                is OrderCreatedEvent -> "CREATED"
                is OrderConfirmedEvent -> "CONFIRMED"
                is OrderShippedEvent -> "SHIPPED"
                is OrderDeliveredEvent -> "DELIVERED"
                is OrderCancelledEvent -> "CANCELLED"
                else -> "UNKNOWN"
            }
            "Step ${index + 1}: $statusText"
        }
        return statusTransitions.joinToString(" → ")
    }
}
```

## Snapshot สำหรับ Performance Optimization

```kotlin
// OrderSnapshotTrigger.kt
package com.order.cqrs.snapshot

import org.axonframework.eventsourcing.AbstractSnapshotTriggerDefinition
import org.axonframework.eventsourcing.EventCountSnapshotTriggerDefinition
import org.axonframework.eventsourcing.Snapshotter
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration

@Configuration
class SnapshotConfig {

    @Bean
    fun snapshotTriggerDefinition(snapshotter: Snapshotter): EventCountSnapshotTriggerDefinition {
        // สร้าง snapshot ทุกๆ 50 events
        return EventCountSnapshotTriggerDefinition(snapshotter, 50)
    }
}
```

## Testing

```kotlin
// OrderAggregateTest.kt
package com.order.cqrs.aggregate

import com.order.cqrs.command.*
import com.order.cqrs.event.*
import org.axonframework.test.aggregate.AggregateTestFixture
import org.junit.jupiter.api.BeforeEach
import org.junit.jupiter.api.Test
import java.math.BigDecimal

class OrderAggregateTest {

    private lateinit var fixture: AggregateTestFixture<OrderAggregate>

    @BeforeEach
    fun setUp() {
        fixture = AggregateTestFixture(OrderAggregate::class.java)
    }

    @Test
    fun `should create order successfully`() {
        fixture.givenNoPriorActivity()
            .`when`(CreateOrderCommand(
                userId = "user-1",
                items = listOf(OrderItemCommand("product-1", 2, BigDecimal("100.00"))),
                shippingAddress = "123 Test Street"
            ))
            .expectEventsMatching { events ->
                events.size == 1 && events[0].payload is OrderCreatedEvent
            }
    }

    @Test
    fun `should confirm pending order`() {
        val orderId = "order-1"
        
        fixture.given(OrderCreatedEvent(
            orderId = orderId,
            userId = "user-1",
            items = listOf(OrderItemData("p1", "Product 1", 1, BigDecimal("100"))),
            shippingAddress = "123 Street",
            totalAmount = BigDecimal("100")
        ))
        .`when`(ConfirmOrderCommand(orderId, "admin"))
        .expectEvents(OrderConfirmedEvent(orderId, "admin"))
    }

    @Test
    fun `should not cancel delivered order`() {
        val orderId = "order-1"
        
        fixture.given(
            OrderCreatedEvent(orderId, "user-1", emptyList(), "", BigDecimal.ZERO),
            OrderConfirmedEvent(orderId, "admin"),
            OrderShippedEvent(orderId, "TRACK123", "FedEx"),
            OrderDeliveredEvent(orderId, java.time.LocalDateTime.now())
        )
        .`when`(CancelOrderCommand(orderId, "Change of mind", "user-1"))
        .expectException(IllegalStateException::class.java)
    }
}
```

## สรุป CQRS และ Event Sourcing

| แนวคิด | คำอธิบาย | ประโยชน์ |
|--------|---------|---------|
| CQRS | แยก Read/Write model | Scale แต่ละส่วนได้อิสระ |
| Event Sourcing | บันทึกเป็น events | ดู history ได้, reconstruct state |
| Aggregate | Business object ที่ handle commands | Encapsulate business logic |
| Projection | สร้าง read model จาก events | Query performance |
| Event Store | ที่เก็บ events ทั้งหมด | Source of truth |
| Snapshot | Checkpoint สำหรับ replay ไว | ลด replay time |

## เมื่อไหร่ควรใช้ CQRS + Event Sourcing

**ควรใช้เมื่อ:**
- Business logic ซับซ้อน
- ต้องการ audit trail ครบถ้วน
- Read/Write workload ต่างกันมาก
- ต้องการ time-travel debugging
- Domain เป็น event-heavy (banking, e-commerce)

**ไม่ควรใช้เมื่อ:**
- Application เล็กๆ ง่ายๆ
- Team ไม่คุ้นเคยกับ pattern
- Business logic ไม่ซับซ้อน
- ไม่ต้องการ audit trail

*Part 59/100+ | Kotlin & Spring Boot Complete Course*
