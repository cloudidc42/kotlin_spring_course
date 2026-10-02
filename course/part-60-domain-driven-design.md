# Part 60: Domain-Driven Design (DDD) - E-Commerce Model

## บทนำ

**Domain-Driven Design (DDD)** คือ approach ในการออกแบบ software ที่เน้นการสร้าง model ให้ตรงกับ business domain โดย Eric Evans เป็นผู้เขียน Blue Book (Domain-Driven Design: Tackling Complexity in the Heart of Software)

## หลักการสำคัญของ DDD

### Ubiquitous Language

ทีม developer และ business stakeholders ต้องใช้ภาษาเดียวกัน เมื่อ business พูดว่า "Order" code ก็ต้องเรียกว่า `Order` ไม่ใช่ `PurchaseRequest` หรือ `TransactionRecord`

### Bounded Context

แต่ละ context มีนิยามของ terms ต่างกัน:

```
Product Context:
  - Product = catalog item ที่มี description, price, image

Inventory Context:
  - Product = item ที่มี stock level, location, barcode

Order Context:
  - Product = line item ในใบสั่งซื้อ ที่มี price ณ เวลานั้น
```

## E-Commerce DDD Model

### Context Map

```
┌─────────────────┐    ┌─────────────────┐
│  Catalog Context│    │Inventory Context│
│  (Products)     │    │  (Stock)        │
└────────┬────────┘    └────────┬────────┘
         │                      │
         └──────────┬───────────┘
                    │
            ┌───────▼────────┐
            │  Order Context │
            │   (Orders)     │
            └───────┬────────┘
                    │
         ┌──────────┴──────────┐
         │                     │
┌────────▼────────┐   ┌────────▼────────┐
│Payment Context  │   │Shipping Context │
│  (Payments)     │   │  (Delivery)     │
└─────────────────┘   └─────────────────┘
```

## Building Blocks ของ DDD

### 1. Value Objects

Value Objects ไม่มี identity ถูก defined โดยค่าของมัน และ immutable

```kotlin
// Money.kt
package com.ecommerce.ddd.catalog.valueobject

import java.math.BigDecimal
import java.util.Currency

data class Money(
    val amount: BigDecimal,
    val currency: Currency
) {
    init {
        require(amount >= BigDecimal.ZERO) { "Amount cannot be negative" }
    }

    operator fun plus(other: Money): Money {
        require(currency == other.currency) { "Cannot add different currencies" }
        return Money(amount + other.amount, currency)
    }

    operator fun minus(other: Money): Money {
        require(currency == other.currency) { "Cannot subtract different currencies" }
        val result = amount - other.amount
        require(result >= BigDecimal.ZERO) { "Result cannot be negative" }
        return Money(result, currency)
    }

    operator fun times(multiplier: BigDecimal): Money {
        return Money(amount * multiplier, currency)
    }

    operator fun times(multiplier: Int): Money = times(multiplier.toBigDecimal())

    fun isGreaterThan(other: Money): Boolean = amount > other.amount

    companion object {
        fun of(amount: BigDecimal, currencyCode: String = "THB"): Money {
            return Money(amount, Currency.getInstance(currencyCode))
        }

        fun zero(currencyCode: String = "THB"): Money {
            return Money(BigDecimal.ZERO, Currency.getInstance(currencyCode))
        }
    }

    override fun toString(): String = "${currency.symbol}${amount}"
}
```

```kotlin
// Address.kt
package com.ecommerce.ddd.shared.valueobject

data class Address(
    val street: String,
    val city: String,
    val province: String,
    val postalCode: String,
    val country: String = "Thailand"
) {
    init {
        require(street.isNotBlank()) { "Street cannot be blank" }
        require(city.isNotBlank()) { "City cannot be blank" }
        require(postalCode.matches(Regex("\\d{5}"))) { "Invalid postal code format" }
    }

    fun toFormattedString(): String {
        return "$street, $city, $province $postalCode, $country"
    }
}
```

```kotlin
// ProductId.kt - Strongly typed ID
package com.ecommerce.ddd.catalog.valueobject

import java.util.UUID

@JvmInline
value class ProductId(val value: String) {
    init {
        require(value.isNotBlank()) { "ProductId cannot be blank" }
    }

    companion object {
        fun generate(): ProductId = ProductId(UUID.randomUUID().toString())
        fun of(value: String): ProductId = ProductId(value)
    }
}

@JvmInline
value class OrderId(val value: String) {
    companion object {
        fun generate(): OrderId = OrderId(UUID.randomUUID().toString())
    }
}

@JvmInline
value class CustomerId(val value: String) {
    companion object {
        fun generate(): CustomerId = CustomerId(UUID.randomUUID().toString())
    }
}
```

### 2. Entities

Entities มี identity และ lifecycle

```kotlin
// Product.kt (Catalog Context)
package com.ecommerce.ddd.catalog.entity

import com.ecommerce.ddd.catalog.valueobject.*
import com.ecommerce.ddd.catalog.event.ProductPriceChangedEvent

class Product private constructor(
    val id: ProductId,
    private var name: String,
    private var description: String,
    private var price: Money,
    private var category: Category,
    private var images: MutableList<ProductImage>,
    private var active: Boolean,
    private val domainEvents: MutableList<Any> = mutableListOf()
) {

    // Factory method - enforce invariants at creation
    companion object {
        fun create(
            name: String,
            description: String,
            price: Money,
            category: Category
        ): Product {
            require(name.isNotBlank()) { "Product name cannot be blank" }
            require(price.amount > BigDecimal.ZERO) { "Product price must be positive" }
            
            return Product(
                id = ProductId.generate(),
                name = name,
                description = description,
                price = price,
                category = category,
                images = mutableListOf(),
                active = true
            )
        }
    }

    fun changeName(newName: String) {
        require(newName.isNotBlank()) { "Name cannot be blank" }
        this.name = newName
    }

    fun changePrice(newPrice: Money) {
        require(newPrice.amount > BigDecimal.ZERO) { "Price must be positive" }
        
        val oldPrice = this.price
        this.price = newPrice
        
        // Domain event
        domainEvents.add(ProductPriceChangedEvent(
            productId = id,
            oldPrice = oldPrice,
            newPrice = newPrice
        ))
    }

    fun addImage(image: ProductImage) {
        require(images.size < 10) { "Maximum 10 images per product" }
        images.add(image)
    }

    fun activate() { this.active = true }
    fun deactivate() { this.active = false }

    // Getters
    fun getName(): String = name
    fun getDescription(): String = description
    fun getPrice(): Money = price
    fun getCategory(): Category = category
    fun getImages(): List<ProductImage> = images.toList()
    fun isActive(): Boolean = active
    fun getDomainEvents(): List<Any> = domainEvents.toList()
    fun clearDomainEvents() = domainEvents.clear()
}
```

### 3. Aggregates

Aggregate คือ cluster ของ entities และ value objects ที่มี root entity หนึ่งตัว

```kotlin
// Order.kt - Aggregate Root
package com.ecommerce.ddd.order.aggregate

import com.ecommerce.ddd.order.entity.OrderItem
import com.ecommerce.ddd.order.valueobject.*
import com.ecommerce.ddd.order.event.*
import com.ecommerce.ddd.shared.valueobject.*

class Order private constructor(
    val id: OrderId,
    val customerId: CustomerId,
    private var status: OrderStatus,
    private val items: MutableList<OrderItem>,
    private val shippingAddress: Address,
    private val domainEvents: MutableList<Any> = mutableListOf()
) {

    companion object {
        fun create(
            customerId: CustomerId,
            shippingAddress: Address
        ): Order {
            val order = Order(
                id = OrderId.generate(),
                customerId = customerId,
                status = OrderStatus.DRAFT,
                items = mutableListOf(),
                shippingAddress = shippingAddress
            )
            
            order.domainEvents.add(OrderCreatedEvent(order.id, customerId))
            return order
        }
    }

    fun addItem(
        productId: ProductId,
        productName: String,
        quantity: Quantity,
        unitPrice: Money
    ): OrderItem {
        require(status == OrderStatus.DRAFT) { "Can only add items to draft orders" }
        require(quantity.value > 0) { "Quantity must be positive" }
        
        // Check if product already in order
        val existingItem = items.find { it.productId == productId }
        
        return if (existingItem != null) {
            val updatedItem = existingItem.increaseQuantity(quantity)
            val index = items.indexOf(existingItem)
            items[index] = updatedItem
            updatedItem
        } else {
            val newItem = OrderItem.create(
                orderId = id,
                productId = productId,
                productName = productName,
                quantity = quantity,
                unitPrice = unitPrice
            )
            items.add(newItem)
            domainEvents.add(ItemAddedToOrderEvent(id, newItem))
            newItem
        }
    }

    fun removeItem(productId: ProductId) {
        require(status == OrderStatus.DRAFT) { "Can only remove items from draft orders" }
        
        val item = items.find { it.productId == productId }
            ?: throw IllegalArgumentException("Item not found in order")
        
        items.remove(item)
        domainEvents.add(ItemRemovedFromOrderEvent(id, productId))
    }

    fun place() {
        require(status == OrderStatus.DRAFT) { "Can only place draft orders" }
        require(items.isNotEmpty()) { "Cannot place empty order" }
        
        status = OrderStatus.PLACED
        domainEvents.add(OrderPlacedEvent(id, customerId, calculateTotal(), items.toList()))
    }

    fun confirm() {
        require(status == OrderStatus.PLACED) { "Can only confirm placed orders" }
        status = OrderStatus.CONFIRMED
        domainEvents.add(OrderConfirmedEvent(id))
    }

    fun cancel(reason: String) {
        require(status in listOf(OrderStatus.PLACED, OrderStatus.DRAFT)) {
            "Cannot cancel order in status: $status"
        }
        status = OrderStatus.CANCELLED
        domainEvents.add(OrderCancelledEvent(id, reason))
    }

    fun calculateTotal(): Money {
        return items.fold(Money.zero()) { acc, item ->
            acc + item.calculateSubtotal()
        }
    }

    fun getItems(): List<OrderItem> = items.toList()
    fun getStatus(): OrderStatus = status
    fun getShippingAddress(): Address = shippingAddress
    fun getDomainEvents(): List<Any> = domainEvents.toList()
    fun clearDomainEvents() = domainEvents.clear()
}

enum class OrderStatus {
    DRAFT, PLACED, CONFIRMED, PROCESSING, SHIPPED, DELIVERED, CANCELLED
}
```

```kotlin
// OrderItem.kt - Entity within Order aggregate
package com.ecommerce.ddd.order.entity

import com.ecommerce.ddd.order.valueobject.*
import com.ecommerce.ddd.shared.valueobject.*

class OrderItem private constructor(
    val id: OrderItemId,
    val orderId: OrderId,
    val productId: ProductId,
    val productName: String,
    private var quantity: Quantity,
    val unitPrice: Money
) {

    companion object {
        fun create(
            orderId: OrderId,
            productId: ProductId,
            productName: String,
            quantity: Quantity,
            unitPrice: Money
        ): OrderItem {
            return OrderItem(
                id = OrderItemId.generate(),
                orderId = orderId,
                productId = productId,
                productName = productName,
                quantity = quantity,
                unitPrice = unitPrice
            )
        }
    }

    fun increaseQuantity(additionalQuantity: Quantity): OrderItem {
        return OrderItem(
            id = id,
            orderId = orderId,
            productId = productId,
            productName = productName,
            quantity = quantity + additionalQuantity,
            unitPrice = unitPrice
        )
    }

    fun calculateSubtotal(): Money {
        return unitPrice * quantity.value
    }

    fun getQuantity(): Quantity = quantity
}
```

### 4. Domain Services

Domain Services ใช้สำหรับ business logic ที่ไม่เหมาะอยู่ใน Entity ใดๆ

```kotlin
// PricingService.kt - Domain Service
package com.ecommerce.ddd.order.service

import com.ecommerce.ddd.order.aggregate.Order
import com.ecommerce.ddd.catalog.repository.ProductRepository
import com.ecommerce.ddd.promotion.repository.PromotionRepository
import com.ecommerce.ddd.shared.valueobject.Money

class OrderPricingService(
    private val productRepository: ProductRepository,
    private val promotionRepository: PromotionRepository
) {

    fun calculateFinalPrice(order: Order): Money {
        val baseTotal = order.calculateTotal()
        val promotions = promotionRepository.findApplicablePromotions(order)
        
        return promotions.fold(baseTotal) { total, promotion ->
            promotion.apply(total)
        }
    }

    fun calculateShippingCost(order: Order): Money {
        val total = order.calculateTotal()
        
        // Free shipping for orders over 1000 THB
        return if (total.isGreaterThan(Money.of(BigDecimal("1000")))) {
            Money.zero()
        } else {
            Money.of(BigDecimal("50"))
        }
    }
}
```

```kotlin
// StockReservationService.kt - Domain Service
package com.ecommerce.ddd.inventory.service

import com.ecommerce.ddd.inventory.aggregate.Inventory
import com.ecommerce.ddd.order.aggregate.Order
import com.ecommerce.ddd.shared.valueobject.ProductId

class StockReservationService(
    private val inventoryRepository: InventoryRepository
) {

    fun reserveStock(order: Order): ReservationResult {
        val reservations = mutableListOf<StockReservation>()
        
        for (item in order.getItems()) {
            val inventory = inventoryRepository.findByProductId(item.productId)
                ?: return ReservationResult.failed(
                    "Product ${item.productId} not found in inventory"
                )

            if (!inventory.hasEnoughStock(item.getQuantity())) {
                // Rollback reservations já feitas
                reservations.forEach { it.cancel() }
                return ReservationResult.failed(
                    "Insufficient stock for product ${item.productId}"
                )
            }

            val reservation = inventory.reserve(item.getQuantity(), order.id)
            reservations.add(reservation)
        }

        return ReservationResult.success(reservations)
    }

    fun releaseReservations(orderId: OrderId) {
        inventoryRepository.findReservationsByOrderId(orderId)
            .forEach { it.release() }
    }
}

sealed class ReservationResult {
    data class Success(val reservations: List<StockReservation>) : ReservationResult()
    data class Failure(val reason: String) : ReservationResult()

    companion object {
        fun success(reservations: List<StockReservation>) = Success(reservations)
        fun failed(reason: String) = Failure(reason)
    }
}
```

### 5. Repositories in DDD

```kotlin
// OrderRepository.kt - Repository interface (Domain Layer)
package com.ecommerce.ddd.order.repository

import com.ecommerce.ddd.order.aggregate.Order
import com.ecommerce.ddd.order.valueobject.OrderId
import com.ecommerce.ddd.order.aggregate.OrderStatus
import com.ecommerce.ddd.shared.valueobject.CustomerId

interface OrderRepository {
    fun save(order: Order): Order
    fun findById(orderId: OrderId): Order?
    fun findByCustomerId(customerId: CustomerId): List<Order>
    fun findByStatus(status: OrderStatus): List<Order>
    fun delete(orderId: OrderId)
}
```

```kotlin
// JpaOrderRepository.kt - Implementation (Infrastructure Layer)
package com.ecommerce.ddd.infrastructure.repository

import com.ecommerce.ddd.order.aggregate.Order
import com.ecommerce.ddd.order.repository.OrderRepository
import com.ecommerce.ddd.order.valueobject.OrderId
import com.ecommerce.ddd.infrastructure.jpa.OrderJpaRepository
import com.ecommerce.ddd.infrastructure.mapper.OrderMapper
import org.springframework.stereotype.Repository

@Repository
class JpaOrderRepository(
    private val jpaRepository: OrderJpaRepository,
    private val mapper: OrderMapper
) : OrderRepository {

    override fun save(order: Order): Order {
        val entity = mapper.toEntity(order)
        val saved = jpaRepository.save(entity)
        
        // Dispatch domain events
        order.getDomainEvents().forEach { event ->
            eventPublisher.publish(event)
        }
        order.clearDomainEvents()
        
        return mapper.toDomain(saved)
    }

    override fun findById(orderId: OrderId): Order? {
        return jpaRepository.findById(orderId.value)
            .map { mapper.toDomain(it) }
            .orElse(null)
    }

    override fun findByCustomerId(customerId: CustomerId): List<Order> {
        return jpaRepository.findByCustomerId(customerId.value)
            .map { mapper.toDomain(it) }
    }

    override fun findByStatus(status: OrderStatus): List<Order> {
        return jpaRepository.findByStatus(status.name)
            .map { mapper.toDomain(it) }
    }

    override fun delete(orderId: OrderId) {
        jpaRepository.deleteById(orderId.value)
    }
}
```

### 6. Domain Events

```kotlin
// OrderEvents.kt
package com.ecommerce.ddd.order.event

import com.ecommerce.ddd.order.entity.OrderItem
import com.ecommerce.ddd.order.valueobject.OrderId
import com.ecommerce.ddd.shared.valueobject.*
import java.time.LocalDateTime

data class OrderCreatedEvent(
    val orderId: OrderId,
    val customerId: CustomerId,
    val occurredAt: LocalDateTime = LocalDateTime.now()
)

data class OrderPlacedEvent(
    val orderId: OrderId,
    val customerId: CustomerId,
    val totalAmount: Money,
    val items: List<OrderItem>,
    val occurredAt: LocalDateTime = LocalDateTime.now()
)

data class OrderConfirmedEvent(
    val orderId: OrderId,
    val occurredAt: LocalDateTime = LocalDateTime.now()
)

data class OrderCancelledEvent(
    val orderId: OrderId,
    val reason: String,
    val occurredAt: LocalDateTime = LocalDateTime.now()
)

data class ItemAddedToOrderEvent(
    val orderId: OrderId,
    val item: OrderItem,
    val occurredAt: LocalDateTime = LocalDateTime.now()
)

data class ItemRemovedFromOrderEvent(
    val orderId: OrderId,
    val productId: ProductId,
    val occurredAt: LocalDateTime = LocalDateTime.now()
)
```

### Application Service

```kotlin
// OrderApplicationService.kt
package com.ecommerce.ddd.order.application

import com.ecommerce.ddd.order.aggregate.Order
import com.ecommerce.ddd.order.repository.OrderRepository
import com.ecommerce.ddd.order.service.OrderPricingService
import com.ecommerce.ddd.order.service.StockReservationService
import com.ecommerce.ddd.shared.valueobject.*
import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional

@Service
class OrderApplicationService(
    private val orderRepository: OrderRepository,
    private val pricingService: OrderPricingService,
    private val stockReservationService: StockReservationService
) {

    @Transactional
    fun createOrder(command: CreateOrderCommand): OrderId {
        val order = Order.create(
            customerId = CustomerId.of(command.customerId),
            shippingAddress = command.shippingAddress
        )

        command.items.forEach { item ->
            order.addItem(
                productId = ProductId.of(item.productId),
                productName = item.productName,
                quantity = Quantity(item.quantity),
                unitPrice = Money.of(item.unitPrice)
            )
        }

        return orderRepository.save(order).id
    }

    @Transactional
    fun placeOrder(orderId: String): Order {
        val order = orderRepository.findById(OrderId(orderId))
            ?: throw OrderNotFoundException("Order $orderId not found")

        // Reserve stock before placing
        val reservationResult = stockReservationService.reserveStock(order)
        if (reservationResult is ReservationResult.Failure) {
            throw StockReservationException(reservationResult.reason)
        }

        order.place()
        return orderRepository.save(order)
    }

    @Transactional
    fun cancelOrder(orderId: String, reason: String) {
        val order = orderRepository.findById(OrderId(orderId))
            ?: throw OrderNotFoundException("Order $orderId not found")

        order.cancel(reason)
        stockReservationService.releaseReservations(order.id)
        orderRepository.save(order)
    }
}
```

## DDD Layers

```
┌─────────────────────────────────────────┐
│            Interface Layer              │
│     (REST Controllers, GraphQL, CLI)    │
├─────────────────────────────────────────┤
│          Application Layer              │
│    (Application Services, Use Cases)   │
├─────────────────────────────────────────┤
│            Domain Layer                 │
│  (Entities, Value Objects, Aggregates)  │
│  (Domain Services, Domain Events)       │
│  (Repository Interfaces)               │
├─────────────────────────────────────────┤
│         Infrastructure Layer           │
│  (JPA Repositories, External APIs)     │
│  (Message Queues, File Systems)         │
└─────────────────────────────────────────┘
```

## สรุป DDD Building Blocks

| Building Block | ลักษณะ | ตัวอย่าง |
|---------------|--------|---------|
| Value Object | Immutable, defined by value | Money, Address, ProductId |
| Entity | Has identity, mutable | Product, OrderItem |
| Aggregate Root | Controls invariants | Order, Customer |
| Domain Service | Stateless, operates on multiple objects | PricingService |
| Domain Event | Records something that happened | OrderPlacedEvent |
| Repository | Abstract persistence | OrderRepository |
| Factory | Creates complex objects | Order.create() |
| Application Service | Orchestrates use cases | OrderApplicationService |

## ข้อดีของ DDD

1. **Ubiquitous Language** - ทีมสื่อสารได้ดีขึ้น
2. **Bounded Contexts** - แต่ละ context มี model ที่เหมาะสม
3. **Business Logic อยู่ใน Domain** - ไม่กระจัดกระจาย
4. **Testability** - Domain logic ทดสอบง่าย
5. **Flexibility** - เปลี่ยน infrastructure ได้โดยไม่กระทบ domain

*Part 60/100+ | Kotlin & Spring Boot Complete Course*
