# Part 34: Reactive Programming ด้วย Spring WebFlux
## Non-blocking, Reactive Web Applications

---

## 🎯 เป้าหมายของ Part นี้

- เข้าใจ Reactive Programming และ Backpressure
- สร้าง Reactive API ด้วย Spring WebFlux
- ใช้ Mono<T> และ Flux<T>
- Reactive Repositories ด้วย R2DBC
- Server-Sent Events (SSE) สำหรับ real-time
- ตัวอย่าง: Real-time notifications

---

## ⚡ 1. Reactive Programming คืออะไร

**Reactive Programming** คือ programming paradigm ที่มุ่งเน้น asynchronous data streams และ propagation of change

```
Traditional (Blocking):          Reactive (Non-blocking):
Thread 1: [request→DB→wait→response]    Thread 1: [req] [resp] [req] [resp]
Thread 2: [request→DB→wait→response]    (One thread handles many requests!)
Thread 3: idle...
```

### Reactive Streams Spec
- **Publisher**: ผลิต data
- **Subscriber**: รับ data
- **Subscription**: ควบคุม flow
- **Processor**: ทั้ง Publisher และ Subscriber

### Project Reactor (Spring WebFlux ใช้)
- **Mono<T>**: 0-1 element
- **Flux<T>**: 0-N elements

---

## 📦 2. Dependencies

```kotlin
// build.gradle.kts
dependencies {
    // WebFlux แทน Spring MVC
    implementation("org.springframework.boot:spring-boot-starter-webflux")

    // R2DBC สำหรับ reactive database
    implementation("org.springframework.boot:spring-boot-starter-data-r2dbc")
    implementation("io.r2dbc:r2dbc-postgresql")
    implementation("org.postgresql:postgresql")

    // Kotlin Coroutines
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-reactor:1.7.3")
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:1.7.3")

    // Testing
    testImplementation("io.projectreactor:reactor-test")
    testImplementation("org.testcontainers:r2dbc:1.19.0")
}
```

```yaml
# application.yml
spring:
  r2dbc:
    url: r2dbc:postgresql://localhost:5432/reactivedb
    username: postgres
    password: password
    pool:
      max-size: 20
      initial-size: 5

  sql:
    init:
      mode: always
      schema-locations: classpath:schema.sql
```

```sql
-- src/main/resources/schema.sql
CREATE TABLE IF NOT EXISTS products (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    price DECIMAL(10,2) NOT NULL,
    description TEXT,
    category_id BIGINT,
    stock_count INT DEFAULT 0,
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE IF NOT EXISTS notifications (
    id SERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL,
    message TEXT NOT NULL,
    type VARCHAR(50),
    is_read BOOLEAN DEFAULT false,
    created_at TIMESTAMP DEFAULT NOW()
);
```

---

## 📊 3. Mono และ Flux ขั้นพื้นฐาน

```kotlin
// src/main/kotlin/com/example/demo/ReactiveBasics.kt
package com.example.demo

import reactor.core.publisher.Flux
import reactor.core.publisher.Mono
import java.time.Duration

fun main() {
    // ===== MONO =====
    
    // สร้าง Mono จาก value
    val mono1: Mono<String> = Mono.just("Hello")
    
    // Mono ว่าง
    val emptyMono: Mono<String> = Mono.empty()
    
    // Mono จาก Callable
    val mono2: Mono<String> = Mono.fromCallable { "computed value" }
    
    // subscribe และดูผลลัพธ์
    mono1
        .map { it.uppercase() }
        .subscribe(
            { value -> println("Value: $value") },
            { error -> println("Error: ${error.message}") },
            { println("Complete!") }
        )

    // ===== FLUX =====
    
    // สร้าง Flux จาก list
    val flux1: Flux<Int> = Flux.just(1, 2, 3, 4, 5)
    
    // สร้าง Flux จาก range
    val flux2: Flux<Int> = Flux.range(1, 10)
    
    // Flux พร้อม delay
    val flux3: Flux<Long> = Flux.interval(Duration.ofSeconds(1))
    
    // Operators
    flux1
        .filter { it % 2 == 0 }           // เอาแต่เลขคู่
        .map { it * 10 }                    // คูณด้วย 10
        .take(3)                            // เอาแค่ 3 ตัว
        .subscribe { println("Value: $it") }
    
    // ===== Error Handling =====
    
    Flux.just(1, 2, 0, 3)
        .map { 10 / it }  // จะ error ที่ 0
        .onErrorReturn(-1)  // fallback value
        .subscribe { println(it) }

    Mono.error<String>(RuntimeException("Something went wrong"))
        .onErrorResume { error ->
            println("Handling error: ${error.message}")
            Mono.just("fallback")  // fallback
        }
        .subscribe { println("Result: $it") }
}
```

---

## 🗄️ 4. R2DBC Reactive Repository

```kotlin
// Entity
// src/main/kotlin/com/example/entity/Product.kt
package com.example.entity

import org.springframework.data.annotation.Id
import org.springframework.data.relational.core.mapping.Column
import org.springframework.data.relational.core.mapping.Table
import java.math.BigDecimal
import java.time.LocalDateTime

@Table("products")
data class Product(
    @Id
    val id: Long? = null,

    val name: String,

    val price: BigDecimal,

    val description: String? = null,

    @Column("category_id")
    val categoryId: Long? = null,

    @Column("stock_count")
    val stockCount: Int = 0,

    @Column("is_active")
    val isActive: Boolean = true,

    @Column("created_at")
    val createdAt: LocalDateTime? = null
)
```

```kotlin
// Repository
// src/main/kotlin/com/example/repository/ReactiveProductRepository.kt
package com.example.repository

import com.example.entity.Product
import org.springframework.data.r2dbc.repository.Query
import org.springframework.data.repository.reactive.ReactiveCrudRepository
import reactor.core.publisher.Flux
import reactor.core.publisher.Mono

interface ReactiveProductRepository : ReactiveCrudRepository<Product, Long> {

    // Flux: หลาย results
    fun findByCategoryId(categoryId: Long): Flux<Product>

    // Mono: single result
    fun findByName(name: String): Mono<Product>

    // Custom query
    @Query("SELECT * FROM products WHERE price BETWEEN :min AND :max AND is_active = true")
    fun findByPriceRange(min: Double, max: Double): Flux<Product>

    // Count
    fun countByCategoryId(categoryId: Long): Mono<Long>

    // Delete and return count
    fun deleteByIsActiveFalse(): Mono<Long>

    // Exists
    fun existsByName(name: String): Mono<Boolean>
}
```

---

## 🌐 5. Reactive Controller

```kotlin
// src/main/kotlin/com/example/controller/ReactiveProductController.kt
package com.example.controller

import com.example.dto.*
import com.example.service.ReactiveProductService
import org.springframework.http.HttpStatus
import org.springframework.http.MediaType
import org.springframework.http.ResponseEntity
import org.springframework.web.bind.annotation.*
import reactor.core.publisher.Flux
import reactor.core.publisher.Mono

@RestController
@RequestMapping("/api/reactive/products")
class ReactiveProductController(
    private val productService: ReactiveProductService
) {

    // Return Mono - Spring WebFlux จัดการ non-blocking
    @GetMapping("/{id}")
    fun getProduct(@PathVariable id: Long): Mono<ResponseEntity<ProductDto>> {
        return productService.findById(id)
            .map { ResponseEntity.ok(it) }
            .defaultIfEmpty(ResponseEntity.notFound().build())
    }

    // Return Flux
    @GetMapping
    fun getAllProducts(): Flux<ProductDto> {
        return productService.findAll()
    }

    // Streaming response (Server-Sent Events)
    @GetMapping(
        path = ["/stream"],
        produces = [MediaType.TEXT_EVENT_STREAM_VALUE]
    )
    fun streamProducts(): Flux<ProductDto> {
        return productService.findAll()
            .delayElements(Duration.ofMillis(100))  // simulate streaming
    }

    // Mono สำหรับ create
    @PostMapping
    fun createProduct(@RequestBody dto: Mono<CreateProductDto>): Mono<ResponseEntity<ProductDto>> {
        return dto
            .flatMap { productService.create(it) }
            .map { ResponseEntity.status(HttpStatus.CREATED).body(it) }
    }

    // ดึงหลาย products พร้อมกัน
    @GetMapping("/batch")
    fun getProductsBatch(@RequestParam ids: List<Long>): Flux<ProductDto> {
        return Flux.fromIterable(ids)
            .flatMap { id -> productService.findById(id) }
    }

    // Merge results จากหลาย categories
    @GetMapping("/multi-category")
    fun getByMultipleCategories(@RequestParam categoryIds: List<Long>): Flux<ProductDto> {
        return Flux.fromIterable(categoryIds)
            .flatMap { categoryId -> productService.findByCategory(categoryId) }
    }
}
```

---

## 🔧 6. Reactive Service

```kotlin
// src/main/kotlin/com/example/service/ReactiveProductService.kt
package com.example.service

import com.example.dto.CreateProductDto
import com.example.dto.ProductDto
import com.example.entity.Product
import com.example.repository.ReactiveProductRepository
import org.springframework.stereotype.Service
import org.springframework.transaction.reactive.TransactionalOperator
import reactor.core.publisher.Flux
import reactor.core.publisher.Mono

@Service
class ReactiveProductService(
    private val productRepository: ReactiveProductRepository,
    private val transactionalOperator: TransactionalOperator
) {

    fun findById(id: Long): Mono<ProductDto> {
        return productRepository.findById(id)
            .map { it.toDto() }
    }

    fun findAll(): Flux<ProductDto> {
        return productRepository.findAll()
            .map { it.toDto() }
    }

    fun findByCategory(categoryId: Long): Flux<ProductDto> {
        return productRepository.findByCategoryId(categoryId)
            .map { it.toDto() }
    }

    fun create(dto: CreateProductDto): Mono<ProductDto> {
        val product = Product(
            name = dto.name,
            price = dto.price,
            description = dto.description
        )
        return productRepository.save(product)
            .map { it.toDto() }
    }

    // Transactional reactive
    fun createWithTransaction(dto: CreateProductDto): Mono<ProductDto> {
        return create(dto)
            .`as`(transactionalOperator::transactional)
    }

    // zip: รวมผลลัพธ์ 2 Mono
    fun getProductWithCategory(productId: Long): Mono<ProductWithCategory> {
        return productRepository.findById(productId)
            .zipWith(Mono.just(CategoryDto(id = 1L, name = "Electronics"))) { product, category ->
                ProductWithCategory(product.toDto(), category)
            }
    }

    // flatMap: chain async operations
    fun findAndEnrich(id: Long): Mono<EnrichedProduct> {
        return productRepository.findById(id)
            .flatMap { product ->
                // fetch additional data
                fetchAdditionalData(product.id!!)
                    .map { extra -> EnrichedProduct(product.toDto(), extra) }
            }
    }

    private fun fetchAdditionalData(productId: Long): Mono<Map<String, Any>> {
        // simulate external call
        return Mono.just(mapOf("views" to 100, "rating" to 4.5))
    }
}
```

---

## 📡 7. Server-Sent Events (SSE) สำหรับ Real-time

```kotlin
// Notification Entity
// src/main/kotlin/com/example/entity/Notification.kt
package com.example.entity

import org.springframework.data.annotation.Id
import org.springframework.data.relational.core.mapping.Column
import org.springframework.data.relational.core.mapping.Table
import java.time.LocalDateTime

@Table("notifications")
data class Notification(
    @Id
    val id: Long? = null,

    @Column("user_id")
    val userId: Long,

    val message: String,

    val type: String? = null,

    @Column("is_read")
    val isRead: Boolean = false,

    @Column("created_at")
    val createdAt: LocalDateTime = LocalDateTime.now()
)
```

```kotlin
// Notification Service
// src/main/kotlin/com/example/service/NotificationService.kt
package com.example.service

import com.example.dto.NotificationDto
import com.example.entity.Notification
import com.example.repository.NotificationRepository
import org.springframework.stereotype.Service
import reactor.core.publisher.Flux
import reactor.core.publisher.Sinks
import java.time.Duration

@Service
class NotificationService(
    private val notificationRepository: NotificationRepository
) {

    // Sinks สำหรับ emit notifications
    private val notificationSink: Sinks.Many<NotificationDto> =
        Sinks.many().multicast().onBackpressureBuffer()

    // Stream notifications สำหรับ user
    fun streamNotifications(userId: Long): Flux<NotificationDto> {
        return notificationSink.asFlux()
            .filter { it.userId == userId }
            .mergeWith(
                // ส่ง pending notifications ที่ยังไม่ได้อ่าน
                notificationRepository.findByUserIdAndIsReadFalse(userId)
                    .map { it.toDto() }
            )
    }

    // Broadcast notification
    fun broadcastNotification(dto: NotificationDto) {
        notificationRepository.save(dto.toEntity())
            .subscribe { saved ->
                notificationSink.tryEmitNext(saved.toDto())
            }
    }

    // Stream ทุกๆ X วินาที (heartbeat)
    fun heartbeat(): Flux<String> {
        return Flux.interval(Duration.ofSeconds(30))
            .map { "heartbeat: $it" }
    }

    // Real-time product updates
    private val productUpdateSink: Sinks.Many<ProductUpdateEvent> =
        Sinks.many().replay().limit(100)  // เก็บ 100 events ล่าสุด

    fun streamProductUpdates(): Flux<ProductUpdateEvent> {
        return productUpdateSink.asFlux()
    }

    fun publishProductUpdate(event: ProductUpdateEvent) {
        productUpdateSink.tryEmitNext(event)
    }
}
```

```kotlin
// SSE Controller
// src/main/kotlin/com/example/controller/NotificationController.kt
package com.example.controller

import com.example.dto.NotificationDto
import com.example.service.NotificationService
import org.springframework.http.MediaType
import org.springframework.http.codec.ServerSentEvent
import org.springframework.web.bind.annotation.*
import reactor.core.publisher.Flux
import java.time.Duration
import java.time.LocalDateTime

@RestController
@RequestMapping("/api/notifications")
class NotificationController(
    private val notificationService: NotificationService
) {

    // SSE Stream สำหรับ user-specific notifications
    @GetMapping(
        path = ["/stream/{userId}"],
        produces = [MediaType.TEXT_EVENT_STREAM_VALUE]
    )
    fun streamNotifications(@PathVariable userId: Long): Flux<ServerSentEvent<NotificationDto>> {
        return notificationService.streamNotifications(userId)
            .map { notification ->
                ServerSentEvent.builder(notification)
                    .id(notification.id.toString())
                    .event("notification")
                    .comment("User $userId notification")
                    .build()
            }
    }

    // Product update stream (ทุกคน subscribe ได้)
    @GetMapping(
        path = ["/products/stream"],
        produces = [MediaType.TEXT_EVENT_STREAM_VALUE]
    )
    fun streamProductUpdates(): Flux<ServerSentEvent<Any>> {
        val productStream = notificationService.streamProductUpdates()
            .map { update ->
                ServerSentEvent.builder<Any>(update)
                    .event("product-update")
                    .build()
            }

        val heartbeat = notificationService.heartbeat()
            .map { beat ->
                ServerSentEvent.builder<Any>(mapOf("type" to "heartbeat", "time" to LocalDateTime.now()))
                    .event("heartbeat")
                    .build()
            }

        // Merge product updates กับ heartbeat
        return Flux.merge(productStream, heartbeat)
    }

    // ส่ง notification
    @PostMapping("/send")
    fun sendNotification(@RequestBody dto: NotificationDto): Mono<Void> {
        notificationService.broadcastNotification(dto)
        return Mono.empty()
    }
}
```

---

## 🌊 8. Backpressure

```kotlin
// src/main/kotlin/com/example/demo/BackpressureDemo.kt
package com.example.demo

import reactor.core.publisher.Flux
import reactor.core.scheduler.Schedulers
import java.time.Duration

fun backpressureDemo() {
    // Fast producer, slow consumer
    Flux.interval(Duration.ofMillis(1))  // 1000 items/sec
        .onBackpressureBuffer(100)        // buffer 100 items
        .publishOn(Schedulers.boundedElastic())
        .subscribe { item ->
            Thread.sleep(10)  // slow consumer: 100 items/sec
            println("Processing: $item")
        }

    // Drop strategy: ทิ้ง item ที่ overflow
    Flux.interval(Duration.ofMillis(1))
        .onBackpressureDrop { dropped ->
            println("Dropped: $dropped")
        }
        .publishOn(Schedulers.single())
        .subscribe { item -> Thread.sleep(10) }

    // Latest strategy: เก็บแค่ตัวล่าสุด
    Flux.interval(Duration.ofMillis(1))
        .onBackpressureLatest()
        .publishOn(Schedulers.single())
        .subscribe { item -> Thread.sleep(10) }

    // limitRate: ขอ upstream ทีละ N items
    Flux.range(1, 1000)
        .limitRate(10)  // ขอทีละ 10
        .subscribe { println(it) }
}
```

---

## 🧪 9. Testing Reactive Code

```kotlin
// src/test/kotlin/com/example/service/ReactiveProductServiceTest.kt
package com.example.service

import com.example.dto.CreateProductDto
import com.example.entity.Product
import com.example.repository.ReactiveProductRepository
import io.mockk.every
import io.mockk.mockk
import org.junit.jupiter.api.Test
import reactor.core.publisher.Flux
import reactor.core.publisher.Mono
import reactor.test.StepVerifier
import java.math.BigDecimal

class ReactiveProductServiceTest {

    private val productRepository = mockk<ReactiveProductRepository>()
    private val transactionalOperator = mockk<TransactionalOperator>()
    private val productService = ReactiveProductService(productRepository, transactionalOperator)

    @Test
    fun `should return product by id`() {
        val product = Product(id = 1L, name = "Test", price = BigDecimal("99.99"))
        every { productRepository.findById(1L) } returns Mono.just(product)

        StepVerifier.create(productService.findById(1L))
            .expectNextMatches { it.name == "Test" }
            .verifyComplete()
    }

    @Test
    fun `should return empty when product not found`() {
        every { productRepository.findById(999L) } returns Mono.empty()

        StepVerifier.create(productService.findById(999L))
            .verifyComplete()  // empty Mono
    }

    @Test
    fun `should return all products as flux`() {
        val products = listOf(
            Product(id = 1L, name = "Product 1", price = BigDecimal("10.00")),
            Product(id = 2L, name = "Product 2", price = BigDecimal("20.00")),
            Product(id = 3L, name = "Product 3", price = BigDecimal("30.00"))
        )
        every { productRepository.findAll() } returns Flux.fromIterable(products)

        StepVerifier.create(productService.findAll())
            .expectNextCount(3)
            .verifyComplete()
    }

    @Test
    fun `should handle error gracefully`() {
        every { productRepository.findById(any()) } returns
            Mono.error(RuntimeException("DB Error"))

        StepVerifier.create(productService.findById(1L))
            .expectError(RuntimeException::class.java)
            .verify()
    }

    @Test
    fun `should create product`() {
        val dto = CreateProductDto(name = "New Product", price = BigDecimal("50.00"))
        val savedProduct = Product(id = 1L, name = "New Product", price = BigDecimal("50.00"))

        every { productRepository.save(any()) } returns Mono.just(savedProduct)

        StepVerifier.create(productService.create(dto))
            .expectNextMatches { it.name == "New Product" }
            .verifyComplete()
    }
}
```

---

## 🔌 10. WebClient (Reactive HTTP Client)

```kotlin
// src/main/kotlin/com/example/client/ExternalApiClient.kt
package com.example.client

import com.example.dto.ProductDto
import org.springframework.stereotype.Component
import org.springframework.web.reactive.function.client.WebClient
import org.springframework.web.reactive.function.client.bodyToFlux
import org.springframework.web.reactive.function.client.bodyToMono
import reactor.core.publisher.Flux
import reactor.core.publisher.Mono

@Component
class ExternalApiClient(
    private val webClient: WebClient
) {

    // Mono request
    fun fetchProduct(id: Long): Mono<ProductDto> {
        return webClient.get()
            .uri("/products/$id")
            .retrieve()
            .onStatus({ it.is4xxClientError }) { response ->
                response.bodyToMono<String>()
                    .flatMap { body ->
                        Mono.error(RuntimeException("Client error: $body"))
                    }
            }
            .onStatus({ it.is5xxServerError }) { response ->
                Mono.error(RuntimeException("Server error"))
            }
            .bodyToMono<ProductDto>()
            .retry(3)  // retry 3 times
            .timeout(Duration.ofSeconds(5))
    }

    // Flux request
    fun fetchAllProducts(): Flux<ProductDto> {
        return webClient.get()
            .uri("/products")
            .retrieve()
            .bodyToFlux<ProductDto>()
    }

    // POST request
    fun createProduct(dto: CreateProductDto): Mono<ProductDto> {
        return webClient.post()
            .uri("/products")
            .bodyValue(dto)
            .retrieve()
            .bodyToMono<ProductDto>()
    }
}
```

---

## 📋 สรุป

| Operator | หน้าที่ | ตัวอย่าง |
|----------|---------|---------|
| `map` | แปลงแต่ละ element | `.map { it.toDto() }` |
| `flatMap` | แปลงเป็น Publisher ใหม่ | `.flatMap { fetchDetails(it) }` |
| `filter` | กรอง elements | `.filter { it.isActive }` |
| `take` | เอาแค่ N elements | `.take(10)` |
| `zip` | รวม 2 Publishers | `Mono.zip(a, b)` |
| `merge` | รวม Flux หลายตัว | `Flux.merge(f1, f2)` |
| `onErrorReturn` | fallback value | `.onErrorReturn(default)` |
| `retry` | retry เมื่อ error | `.retry(3)` |
| `timeout` | ตั้ง timeout | `.timeout(Duration.ofSeconds(5))` |

### Mono vs Flux

| | Mono<T> | Flux<T> |
|-|---------|---------|
| Elements | 0 หรือ 1 | 0 ถึง N |
| ใช้เมื่อ | Single result | Multiple results |
| ตัวอย่าง | `findById()` | `findAll()` |

### Spring WebFlux vs Spring MVC

| | WebFlux | MVC |
|-|---------|-----|
| Thread model | Event loop | Thread per request |
| Scalability | สูงมาก | ปานกลาง |
| Backpressure | รองรับ | ไม่รองรับ |
| Learning curve | สูง | ต่ำ |
| ใช้เมื่อ | High concurrency | ทั่วไป |

---

*Part 34/100+ | Kotlin & Spring Boot Complete Course*
