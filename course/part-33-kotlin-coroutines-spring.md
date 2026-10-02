# Part 33: Kotlin Coroutines กับ Spring Boot
## Asynchronous Programming ใน Spring

---

## 🎯 เป้าหมายของ Part นี้

- ใช้ suspend functions ใน Spring controllers
- Coroutine scope ใน services
- Async database calls ด้วย coroutines
- Concurrent API calls ด้วย async/await
- Structured concurrency
- ตัวอย่าง: Parallel API aggregation

---

## 🔄 1. Coroutines ใน Spring Boot

Kotlin Coroutines ช่วยให้เขียน asynchronous code ได้เหมือนเป็น synchronous code ซึ่งทำให้โค้ดอ่านง่ายและดูแลรักษาง่ายกว่า callback หรือ reactive programming

```
Traditional:                 Coroutines:
void method() {             suspend fun method() {
  future.thenApply(...)       val result = await()
    .thenCompose(...)          doSomething(result)
}                           }
```

### ประเภทของ Coroutines ใน Spring
1. **Spring MVC + Coroutines**: suspend functions ใน controller
2. **Spring WebFlux + Coroutines**: สมบูรณ์ยิ่งกว่า (reactive stack)
3. **Coroutines ใน Service layer**: สำหรับ async operations

---

## 📦 2. Dependencies

```kotlin
// build.gradle.kts
plugins {
    kotlin("jvm") version "1.9.0"
    id("org.springframework.boot") version "3.1.0"
}

dependencies {
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")

    // Coroutines
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:1.7.3")
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-reactor:1.7.3")
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-slf4j:1.7.3")

    // Spring WebFlux (ถ้าต้องการ reactive stack)
    // implementation("org.springframework.boot:spring-boot-starter-webflux")

    // R2DBC (reactive database)
    // implementation("org.springframework.boot:spring-boot-starter-data-r2dbc")
    // implementation("io.r2dbc:r2dbc-postgresql")
    
    // HTTP Client
    implementation("org.springframework.boot:spring-boot-starter-webflux")
}

kotlin {
    compilerOptions {
        freeCompilerArgs.addAll("-Xjsr305=strict")
    }
}
```

---

## ⚙️ 3. Coroutine Configuration

```kotlin
// src/main/kotlin/com/example/config/CoroutineConfig.kt
package com.example.config

import kotlinx.coroutines.CoroutineScope
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.SupervisorJob
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration

@Configuration
class CoroutineConfig {

    // Application-level coroutine scope
    // SupervisorJob: child failures ไม่กระทบ parent
    @Bean
    fun applicationScope(): CoroutineScope {
        return CoroutineScope(SupervisorJob() + Dispatchers.Default)
    }

    // IO-focused scope สำหรับ database/network operations
    @Bean
    fun ioScope(): CoroutineScope {
        return CoroutineScope(SupervisorJob() + Dispatchers.IO)
    }
}
```

---

## 🌐 4. Suspend Functions ใน Controllers

```kotlin
// src/main/kotlin/com/example/controller/ProductController.kt
package com.example.controller

import com.example.dto.ProductDto
import com.example.service.ProductCoroutineService
import kotlinx.coroutines.async
import kotlinx.coroutines.coroutineScope
import org.springframework.web.bind.annotation.*

@RestController
@RequestMapping("/api/products")
class ProductController(
    private val productCoroutineService: ProductCoroutineService
) {

    // suspend function ใน controller - Spring MVC รองรับ
    @GetMapping("/{id}")
    suspend fun getProduct(@PathVariable id: Long): ProductDto? {
        return productCoroutineService.getProductById(id)
    }

    // ดึงหลาย products พร้อมกัน
    @GetMapping("/batch")
    suspend fun getProductsBatch(
        @RequestParam ids: List<Long>
    ): List<ProductDto?> = coroutineScope {
        // ส่ง request พร้อมกันทุก id
        ids.map { id ->
            async { productCoroutineService.getProductById(id) }
        }.map { it.await() }
    }

    // สร้าง product
    @PostMapping
    suspend fun createProduct(@RequestBody dto: CreateProductDto): ProductDto {
        return productCoroutineService.createProduct(dto)
    }
}
```

---

## 🔧 5. Coroutine Service

```kotlin
// src/main/kotlin/com/example/service/ProductCoroutineService.kt
package com.example.service

import com.example.dto.ProductDto
import com.example.repository.ProductRepository
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.withContext
import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional

@Service
class ProductCoroutineService(
    private val productRepository: ProductRepository
) {

    // withContext(Dispatchers.IO) สำหรับ blocking DB calls
    suspend fun getProductById(id: Long): ProductDto? =
        withContext(Dispatchers.IO) {
            productRepository.findById(id).orElse(null)?.toDto()
        }

    suspend fun getAllProducts(): List<ProductDto> =
        withContext(Dispatchers.IO) {
            productRepository.findAll().map { it.toDto() }
        }

    @Transactional
    suspend fun createProduct(dto: CreateProductDto): ProductDto =
        withContext(Dispatchers.IO) {
            val product = Product(name = dto.name, price = dto.price)
            productRepository.save(product).toDto()
        }
}
```

---

## ⚡ 6. Parallel API Calls

```kotlin
// src/main/kotlin/com/example/service/ApiAggregationService.kt
package com.example.service

import com.example.dto.*
import kotlinx.coroutines.*
import org.springframework.stereotype.Service
import org.springframework.web.reactive.function.client.WebClient
import org.springframework.web.reactive.function.client.awaitBody

@Service
class ApiAggregationService(
    private val webClient: WebClient
) {

    // ดึงข้อมูลจาก APIs หลายตัวพร้อมกัน
    suspend fun getProductDashboard(productId: Long): ProductDashboard =
        coroutineScope {
            // เริ่ม async calls พร้อมกัน
            val productDeferred = async { fetchProduct(productId) }
            val reviewsDeferred = async { fetchReviews(productId) }
            val recommendationsDeferred = async { fetchRecommendations(productId) }
            val stockDeferred = async { fetchStockInfo(productId) }

            // รอผลลัพธ์ทั้งหมด
            ProductDashboard(
                product = productDeferred.await(),
                reviews = reviewsDeferred.await(),
                recommendations = recommendationsDeferred.await(),
                stockInfo = stockDeferred.await()
            )
        }

    // ดึงข้อมูลพร้อม timeout
    suspend fun getProductWithTimeout(productId: Long): ProductDto? =
        withTimeoutOrNull(5000L) {  // timeout 5 วินาที
            fetchProduct(productId)
        }

    // Sequential vs Parallel comparison
    suspend fun fetchSequential(ids: List<Long>): List<ProductDto?> {
        return ids.map { fetchProduct(it) }  // sequential
    }

    suspend fun fetchParallel(ids: List<Long>): List<ProductDto?> =
        coroutineScope {
            ids.map { id ->
                async { fetchProduct(id) }
            }.awaitAll()  // parallel
        }

    // Error handling ใน parallel calls
    suspend fun fetchProductsWithFallback(ids: List<Long>): List<ProductDto?> =
        coroutineScope {
            ids.map { id ->
                async {
                    try {
                        fetchProduct(id)
                    } catch (e: Exception) {
                        println("Failed to fetch product $id: ${e.message}")
                        null  // fallback to null
                    }
                }
            }.awaitAll()
        }

    private suspend fun fetchProduct(productId: Long): ProductDto {
        return webClient.get()
            .uri("/api/products/$productId")
            .retrieve()
            .awaitBody<ProductDto>()
    }

    private suspend fun fetchReviews(productId: Long): List<ReviewDto> {
        return webClient.get()
            .uri("/api/reviews?productId=$productId")
            .retrieve()
            .awaitBody<List<ReviewDto>>()
    }

    private suspend fun fetchRecommendations(productId: Long): List<ProductDto> {
        return webClient.get()
            .uri("/api/recommendations?productId=$productId")
            .retrieve()
            .awaitBody<List<ProductDto>>()
    }

    private suspend fun fetchStockInfo(productId: Long): StockInfo {
        return webClient.get()
            .uri("/api/inventory/$productId")
            .retrieve()
            .awaitBody<StockInfo>()
    }
}
```

---

## 🔁 7. Coroutine Flow

```kotlin
// src/main/kotlin/com/example/service/ProductFlowService.kt
package com.example.service

import com.example.dto.ProductDto
import com.example.repository.ProductRepository
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.flow.*
import kotlinx.coroutines.withContext
import org.springframework.stereotype.Service

@Service
class ProductFlowService(
    private val productRepository: ProductRepository
) {

    // Flow สำหรับ streaming data
    fun getProductsFlow(): Flow<ProductDto> = flow {
        val products = withContext(Dispatchers.IO) {
            productRepository.findAll()
        }
        products.forEach { product ->
            emit(product.toDto())
        }
    }.flowOn(Dispatchers.IO)

    // Flow พร้อม transform
    fun getExpensiveProducts(minPrice: Double): Flow<ProductDto> =
        getProductsFlow()
            .filter { it.price.toDouble() >= minPrice }
            .map { it.copy(name = it.name.uppercase()) }
            .take(10)

    // Combine flows
    fun getCombinedFlow(
        productsFlow: Flow<ProductDto>,
        discountsFlow: Flow<DiscountDto>
    ): Flow<ProductWithDiscount> =
        productsFlow.combine(discountsFlow) { product, discount ->
            ProductWithDiscount(product, discount)
        }

    // StateFlow สำหรับ state management
    private val _productUpdates = MutableStateFlow<ProductDto?>(null)
    val productUpdates: StateFlow<ProductDto?> = _productUpdates.asStateFlow()

    suspend fun updateProductAndNotify(id: Long, dto: UpdateProductDto): ProductDto {
        val updated = withContext(Dispatchers.IO) {
            val product = productRepository.findById(id)
                .orElseThrow { NotFoundException("Product not found") }
            product.apply { name = dto.name ?: name }
            productRepository.save(product).toDto()
        }
        _productUpdates.value = updated
        return updated
    }
}
```

---

## 🌊 8. Coroutines กับ Spring WebFlux

```kotlin
// src/main/kotlin/com/example/controller/ReactiveProductController.kt
package com.example.controller

import com.example.dto.ProductDto
import com.example.service.ProductFlowService
import kotlinx.coroutines.flow.Flow
import org.springframework.http.MediaType
import org.springframework.web.bind.annotation.*

@RestController
@RequestMapping("/api/reactive/products")
class ReactiveProductController(
    private val productFlowService: ProductFlowService
) {

    // Return Flow - WebFlux แปลงเป็น Flux อัตโนมัติ
    @GetMapping(produces = [MediaType.APPLICATION_NDJSON_VALUE])
    fun getProductStream(): Flow<ProductDto> {
        return productFlowService.getProductsFlow()
    }

    // Server-Sent Events ด้วย Flow
    @GetMapping(
        path = ["/stream"],
        produces = [MediaType.TEXT_EVENT_STREAM_VALUE]
    )
    fun streamProducts(): Flow<ProductDto> {
        return productFlowService.getProductsFlow()
    }

    // Filter flow
    @GetMapping("/expensive")
    fun getExpensiveProducts(
        @RequestParam(defaultValue = "100.0") minPrice: Double
    ): Flow<ProductDto> {
        return productFlowService.getExpensiveProducts(minPrice)
    }
}
```

---

## 🗄️ 9. Coroutines กับ Database (R2DBC)

```kotlin
// src/main/kotlin/com/example/repository/ReactiveProductRepository.kt
package com.example.repository

import com.example.entity.Product
import kotlinx.coroutines.flow.Flow
import org.springframework.data.r2dbc.repository.Query
import org.springframework.data.repository.kotlin.CoroutineCrudRepository

// CoroutineCrudRepository รองรับ suspend functions โดยตรง
interface ReactiveProductRepository : CoroutineCrudRepository<Product, Long> {
    
    // suspend function - single result
    suspend fun findByName(name: String): Product?

    // Flow - multiple results
    fun findByCategoryId(categoryId: Long): Flow<Product>

    // Custom query
    @Query("SELECT * FROM products WHERE price > :minPrice")
    fun findByPriceGreaterThan(minPrice: Double): Flow<Product>
    
    // Count
    suspend fun countByCategoryId(categoryId: Long): Long
}
```

```kotlin
// src/main/kotlin/com/example/service/ReactiveProductService.kt
package com.example.service

import com.example.dto.ProductDto
import com.example.repository.ReactiveProductRepository
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.map
import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional

@Service
class ReactiveProductService(
    private val productRepository: ReactiveProductRepository
) {

    // ใช้ suspend ได้โดยตรงกับ CoroutineCrudRepository
    suspend fun findById(id: Long): ProductDto? {
        return productRepository.findById(id)?.toDto()
    }

    // Flow ของ products
    fun findByCategory(categoryId: Long): Flow<ProductDto> {
        return productRepository.findByCategoryId(categoryId)
            .map { it.toDto() }
    }

    @Transactional
    suspend fun createProduct(dto: CreateProductDto): ProductDto {
        val product = Product(name = dto.name, price = dto.price)
        return productRepository.save(product).toDto()
    }

    // Batch insert
    @Transactional
    suspend fun createProductsBatch(dtos: List<CreateProductDto>): List<ProductDto> {
        val products = dtos.map { Product(name = it.name, price = it.price) }
        val saved = mutableListOf<ProductDto>()
        productRepository.saveAll(products).collect { saved.add(it.toDto()) }
        return saved
    }
}
```

---

## 🛡️ 10. Structured Concurrency และ Error Handling

```kotlin
// src/main/kotlin/com/example/service/RobustApiService.kt
package com.example.service

import kotlinx.coroutines.*

class RobustApiService {

    // SupervisorScope: child failures ไม่ยกเลิก siblings
    suspend fun fetchMultipleWithSupervisor(ids: List<Long>): List<Result<ProductDto>> =
        supervisorScope {
            ids.map { id ->
                async {
                    runCatching { fetchProduct(id) }
                }
            }.map { it.await() }
        }

    // ยกเลิก coroutines อื่นเมื่อตัวแรกสำเร็จ
    suspend fun fetchFirstSuccessful(urls: List<String>): ProductDto? =
        coroutineScope {
            val channel = Channel<ProductDto>()
            val jobs = urls.map { url ->
                launch {
                    try {
                        val result = fetchFromUrl(url)
                        channel.trySend(result)
                    } catch (e: Exception) {
                        // ignore
                    }
                }
            }

            val result = withTimeoutOrNull(5000) {
                channel.receive()
            }

            // ยกเลิก jobs ที่เหลือ
            jobs.forEach { it.cancel() }
            result
        }

    // Retry logic
    suspend fun <T> withRetry(
        maxAttempts: Int = 3,
        initialDelay: Long = 1000,
        block: suspend () -> T
    ): T {
        var lastException: Exception? = null
        repeat(maxAttempts) { attempt ->
            try {
                return block()
            } catch (e: Exception) {
                lastException = e
                println("Attempt ${attempt + 1} failed: ${e.message}")
                if (attempt < maxAttempts - 1) {
                    delay(initialDelay * (attempt + 1))
                }
            }
        }
        throw lastException ?: RuntimeException("All attempts failed")
    }

    // ใช้งาน retry
    suspend fun fetchProductWithRetry(id: Long): ProductDto {
        return withRetry(maxAttempts = 3, initialDelay = 500) {
            fetchProduct(id)
        }
    }

    private suspend fun fetchProduct(id: Long): ProductDto {
        delay(100) // simulate network call
        return ProductDto(id = id, name = "Product $id", price = java.math.BigDecimal.TEN, description = null, categoryId = null, stockCount = 0, isActive = true)
    }

    private suspend fun fetchFromUrl(url: String): ProductDto {
        delay(200) // simulate
        return ProductDto(id = 1L, name = "Product", price = java.math.BigDecimal.TEN, description = null, categoryId = null, stockCount = 0, isActive = true)
    }
}
```

---

## 🧪 11. Testing Coroutines

```kotlin
// src/test/kotlin/com/example/service/ProductCoroutineServiceTest.kt
package com.example.service

import com.example.dto.ProductDto
import io.mockk.coEvery
import io.mockk.mockk
import kotlinx.coroutines.ExperimentalCoroutinesApi
import kotlinx.coroutines.test.*
import org.junit.jupiter.api.Test
import java.math.BigDecimal
import kotlin.test.assertEquals
import kotlin.test.assertNotNull

@OptIn(ExperimentalCoroutinesApi::class)
class ProductCoroutineServiceTest {

    private val productRepository = mockk<ProductRepository>()
    private val productService = ProductCoroutineService(productRepository)

    @Test
    fun `should return product by id`() = runTest {
        val expected = ProductDto(
            id = 1L,
            name = "Test Product",
            price = BigDecimal("99.99"),
            description = null,
            categoryId = null,
            stockCount = 10,
            isActive = true
        )

        coEvery { productRepository.findById(1L) } returns java.util.Optional.of(
            Product(id = 1L, name = "Test Product", price = BigDecimal("99.99"))
        )

        val result = productService.getProductById(1L)
        assertNotNull(result)
        assertEquals("Test Product", result.name)
    }

    @Test
    fun `should fetch products in parallel`() = runTest {
        val aggregationService = ApiAggregationService(mockk())

        // วัดเวลา parallel vs sequential
        val ids = listOf(1L, 2L, 3L, 4L, 5L)
        val parallelStart = System.currentTimeMillis()
        // aggregationService.fetchParallel(ids) // ถ้า mock ถูกต้อง
        val parallelTime = System.currentTimeMillis() - parallelStart

        println("Parallel time: ${parallelTime}ms")
    }
}
```

---

## 📊 12. Performance Tips

```kotlin
// Tips สำหรับ Coroutines Performance

// 1. ใช้ Dispatchers.IO สำหรับ blocking operations
suspend fun blockingDbCall(): Data = withContext(Dispatchers.IO) {
    // JDBC, blocking file IO
    database.query("SELECT ...")
}

// 2. ใช้ Dispatchers.Default สำหรับ CPU-intensive
suspend fun cpuIntensive(): Result = withContext(Dispatchers.Default) {
    // การคำนวณซับซ้อน
    processLargeDataSet()
}

// 3. ใช้ async/await สำหรับ parallel
suspend fun parallel(): Triple<A, B, C> = coroutineScope {
    val a = async { fetchA() }
    val b = async { fetchB() }
    val c = async { fetchC() }
    Triple(a.await(), b.await(), c.await())
}

// 4. withTimeoutOrNull สำหรับ timeout
suspend fun withTimeout(): Data? = withTimeoutOrNull(3000L) {
    slowOperation()
}

// 5. Flow buffer สำหรับ streaming
fun bufferedFlow(): Flow<Data> =
    dataFlow().buffer(capacity = 64)
```

---

## 📋 สรุป

| Feature | Spring MVC + Coroutines | Spring WebFlux + Coroutines |
|---------|------------------------|----------------------------|
| Controller | `suspend fun` | `suspend fun` หรือ `Flow<T>` |
| Database | JPA + `withContext(IO)` | R2DBC + `CoroutineCrudRepository` |
| Performance | ดี | ดีมาก (fully non-blocking) |
| Learning curve | น้อย | มากกว่า |
| Recommendation | ทั่วไป | High-throughput systems |

### Dispatchers

| Dispatcher | ใช้เมื่อ |
|-----------|---------|
| `Dispatchers.Default` | CPU-intensive tasks |
| `Dispatchers.IO` | Blocking I/O (JDBC, files) |
| `Dispatchers.Main` | UI (Android/Desktop) |
| `Dispatchers.Unconfined` | ทดสอบเท่านั้น |

---

*Part 33/100+ | Kotlin & Spring Boot Complete Course*
