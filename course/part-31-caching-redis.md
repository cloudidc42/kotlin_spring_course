# Part 31: Caching ด้วย Redis
## Spring Cache Abstraction และ Redis Integration

---

## 🎯 เป้าหมายของ Part นี้

- เข้าใจ Spring Cache Abstraction
- ใช้ Redis เป็น Cache Provider
- ตั้งค่า TTL และ Cache Keys
- ใช้ @Cacheable, @CacheEvict, @CachePut
- Conditional caching
- ตัวอย่าง: Product catalog caching

---

## 🗄️ 1. ทำความเข้าใจ Caching

**Caching** คือการเก็บข้อมูลที่ใช้บ่อยไว้ใน memory เพื่อลดการเข้าถึงฐานข้อมูลหรือ service ภายนอก

### ประโยชน์ของ Caching
- ลด latency ในการตอบสนอง
- ลดภาระ database
- รองรับ traffic สูงได้ดีขึ้น
- ประหยัด cost (ลดจำนวน API calls)

### เมื่อไหร่ควรใช้ Cache
- ข้อมูลที่อ่านบ่อย แต่เปลี่ยนแปลงน้อย
- การคำนวณที่ใช้เวลานาน
- ผลลัพธ์จาก external API
- Session data

---

## 🐳 2. Redis Setup ด้วย Docker

```yaml
# docker-compose.yml
version: '3.8'
services:
  redis:
    image: redis:7-alpine
    container_name: redis-cache
    ports:
      - "6379:6379"
    command: redis-server --maxmemory 256mb --maxmemory-policy allkeys-lru
    volumes:
      - redis-data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis-commander:
    image: rediscommander/redis-commander:latest
    environment:
      - REDIS_HOSTS=local:redis:6379
    ports:
      - "8081:8081"
    depends_on:
      - redis

volumes:
  redis-data:
```

```bash
# Start Redis
docker-compose up -d redis redis-commander

# Test connection
redis-cli ping
# Output: PONG

# Redis Commander UI: http://localhost:8081
```

---

## 📦 3. Dependencies

```kotlin
// build.gradle.kts
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-data-redis")
    implementation("org.springframework.boot:spring-boot-starter-cache")
    implementation("com.fasterxml.jackson.module:jackson-module-kotlin")
    
    // Optional: Jedis as Redis client
    implementation("redis.clients:jedis")
    
    // For serialization
    implementation("com.fasterxml.jackson.datatype:jackson-datatype-jsr310")
}
```

---

## ⚙️ 4. Redis Configuration

```kotlin
// src/main/kotlin/com/example/config/RedisConfig.kt
package com.example.config

import com.fasterxml.jackson.annotation.JsonTypeInfo
import com.fasterxml.jackson.databind.ObjectMapper
import com.fasterxml.jackson.databind.jsontype.impl.LaissezFaireSubTypeValidator
import com.fasterxml.jackson.module.kotlin.kotlinModule
import org.springframework.cache.CacheManager
import org.springframework.cache.annotation.EnableCaching
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.data.redis.cache.RedisCacheConfiguration
import org.springframework.data.redis.cache.RedisCacheManager
import org.springframework.data.redis.connection.RedisConnectionFactory
import org.springframework.data.redis.core.RedisTemplate
import org.springframework.data.redis.serializer.GenericJackson2JsonRedisSerializer
import org.springframework.data.redis.serializer.RedisSerializationContext
import org.springframework.data.redis.serializer.StringRedisSerializer
import java.time.Duration

@Configuration
@EnableCaching  // เปิดใช้งาน Spring Cache
class RedisConfig {

    // สร้าง ObjectMapper สำหรับ serialize/deserialize
    @Bean
    fun redisObjectMapper(): ObjectMapper {
        return ObjectMapper().apply {
            registerModule(kotlinModule())
            // เก็บ type information เพื่อ deserialize กลับมาถูกต้อง
            activateDefaultTyping(
                LaissezFaireSubTypeValidator.instance,
                ObjectMapper.DefaultTyping.NON_FINAL,
                JsonTypeInfo.As.PROPERTY
            )
        }
    }

    // Redis Serializer สำหรับ JSON
    @Bean
    fun redisSerializer(redisObjectMapper: ObjectMapper): GenericJackson2JsonRedisSerializer {
        return GenericJackson2JsonRedisSerializer(redisObjectMapper)
    }

    // RedisTemplate สำหรับการใช้งาน Redis โดยตรง
    @Bean
    fun redisTemplate(
        connectionFactory: RedisConnectionFactory,
        redisSerializer: GenericJackson2JsonRedisSerializer
    ): RedisTemplate<String, Any> {
        return RedisTemplate<String, Any>().apply {
            setConnectionFactory(connectionFactory)
            keySerializer = StringRedisSerializer()
            valueSerializer = redisSerializer
            hashKeySerializer = StringRedisSerializer()
            hashValueSerializer = redisSerializer
        }
    }

    // Default Cache Configuration
    private fun defaultCacheConfig(
        redisSerializer: GenericJackson2JsonRedisSerializer,
        ttl: Duration = Duration.ofMinutes(10)
    ): RedisCacheConfiguration {
        return RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(ttl)
            .disableCachingNullValues()  // ไม่ cache null values
            .serializeKeysWith(
                RedisSerializationContext.SerializationPair.fromSerializer(StringRedisSerializer())
            )
            .serializeValuesWith(
                RedisSerializationContext.SerializationPair.fromSerializer(redisSerializer)
            )
    }

    // CacheManager พร้อม TTL settings แต่ละ cache
    @Bean
    fun cacheManager(
        connectionFactory: RedisConnectionFactory,
        redisSerializer: GenericJackson2JsonRedisSerializer
    ): CacheManager {
        // กำหนด TTL แยกตาม cache name
        val cacheConfigurations = mapOf(
            "products" to defaultCacheConfig(redisSerializer, Duration.ofMinutes(30)),
            "categories" to defaultCacheConfig(redisSerializer, Duration.ofHours(1)),
            "users" to defaultCacheConfig(redisSerializer, Duration.ofMinutes(15)),
            "productList" to defaultCacheConfig(redisSerializer, Duration.ofMinutes(5)),
        )

        return RedisCacheManager.builder(connectionFactory)
            .cacheDefaults(defaultCacheConfig(redisSerializer))
            .withInitialCacheConfigurations(cacheConfigurations)
            .build()
    }
}
```

```yaml
# application.yml
spring:
  data:
    redis:
      host: localhost
      port: 6379
      timeout: 2000ms
      connect-timeout: 2000ms
      lettuce:
        pool:
          max-active: 8
          max-idle: 8
          min-idle: 0
          max-wait: -1ms

  cache:
    type: redis
    redis:
      time-to-live: 600000  # 10 minutes default (ms)
      cache-null-values: false
      use-key-prefix: true
      key-prefix: "myapp:"

logging:
  level:
    org.springframework.cache: DEBUG  # เปิด log สำหรับ debug
```

---

## 🏷️ 5. Spring Cache Annotations

### 5.1 @Cacheable - อ่านจาก Cache

```kotlin
// src/main/kotlin/com/example/service/ProductService.kt
package com.example.service

import com.example.dto.ProductDto
import com.example.repository.ProductRepository
import org.springframework.cache.annotation.Cacheable
import org.springframework.stereotype.Service

@Service
class ProductService(private val productRepository: ProductRepository) {

    // Cache ผลลัพธ์โดยใช้ id เป็น key
    // ครั้งแรก: query DB, ครั้งต่อไป: อ่านจาก cache
    @Cacheable(
        cacheNames = ["products"],
        key = "#id"
    )
    fun getProductById(id: Long): ProductDto? {
        println("Fetching from database: $id")  // จะปรากฎเฉพาะครั้งแรก
        return productRepository.findById(id)
            .map { it.toDto() }
            .orElse(null)
    }

    // Cache พร้อม condition - cache เฉพาะ price > 0
    @Cacheable(
        cacheNames = ["products"],
        key = "#id",
        condition = "#id > 0"  // condition ตรวจก่อน execute
    )
    fun getProductByIdConditional(id: Long): ProductDto? {
        return productRepository.findById(id)
            .map { it.toDto() }
            .orElse(null)
    }

    // Cache พร้อม unless - ไม่ cache ถ้าผลลัพธ์เป็น null
    @Cacheable(
        cacheNames = ["products"],
        key = "#id",
        unless = "#result == null"  // unless ตรวจหลัง execute
    )
    fun getProductByIdUnlessNull(id: Long): ProductDto? {
        return productRepository.findById(id)
            .map { it.toDto() }
            .orElse(null)
    }

    // Cache list ของ products
    @Cacheable(
        cacheNames = ["productList"],
        key = "'all'"  // key คงที่สำหรับ list ทั้งหมด
    )
    fun getAllProducts(): List<ProductDto> {
        println("Fetching all products from database")
        return productRepository.findAll().map { it.toDto() }
    }

    // Cache พร้อม SpEL expression ซับซ้อน
    @Cacheable(
        cacheNames = ["productList"],
        key = "'category:' + #categoryId + ':page:' + #page + ':size:' + #size"
    )
    fun getProductsByCategory(
        categoryId: Long,
        page: Int = 0,
        size: Int = 10
    ): List<ProductDto> {
        println("Fetching products by category: $categoryId, page: $page")
        return productRepository.findByCategoryId(categoryId, page, size)
            .map { it.toDto() }
    }
}
```

### 5.2 @CacheEvict - ลบ Cache

```kotlin
// เพิ่มใน ProductService
import org.springframework.cache.annotation.CacheEvict

// ลบ cache เมื่อ update product
@CacheEvict(
    cacheNames = ["products"],
    key = "#id"
)
fun updateProduct(id: Long, dto: UpdateProductDto): ProductDto {
    val product = productRepository.findById(id)
        .orElseThrow { NotFoundException("Product $id not found") }
    
    product.apply {
        name = dto.name ?: name
        price = dto.price ?: price
        description = dto.description ?: description
    }
    
    return productRepository.save(product).toDto()
}

// ลบ cache เมื่อ delete product
@CacheEvict(
    cacheNames = ["products"],
    key = "#id"
)
fun deleteProduct(id: Long) {
    productRepository.deleteById(id)
}

// ลบ cache ทั้งหมดใน "products" cache
@CacheEvict(
    cacheNames = ["products"],
    allEntries = true  // ลบทุก entry
)
fun clearProductCache() {
    println("Cleared all product cache")
}

// ลบหลาย cache พร้อมกัน
@Caching(
    evict = [
        CacheEvict(cacheNames = ["products"], key = "#id"),
        CacheEvict(cacheNames = ["productList"], allEntries = true)
    ]
)
fun deleteProductAndClearList(id: Long) {
    productRepository.deleteById(id)
}
```

### 5.3 @CachePut - อัพเดท Cache

```kotlin
import org.springframework.cache.annotation.CachePut

// @CachePut: update cache เสมอ (ไม่ skip method execution)
// ใช้เมื่อต้องการ update cache หลัง write
@CachePut(
    cacheNames = ["products"],
    key = "#result.id"  // ใช้ id จากผลลัพธ์
)
fun createProduct(dto: CreateProductDto): ProductDto {
    val product = Product(
        name = dto.name,
        price = dto.price,
        description = dto.description,
        categoryId = dto.categoryId
    )
    return productRepository.save(product).toDto()
}

// @CachePut กับ update - cache ด้วย id ที่ระบุ
@CachePut(
    cacheNames = ["products"],
    key = "#id"
)
fun updateProductAndCache(id: Long, dto: UpdateProductDto): ProductDto {
    val product = productRepository.findById(id)
        .orElseThrow { NotFoundException("Product $id not found") }
    
    product.apply {
        name = dto.name ?: name
        price = dto.price ?: price
    }
    
    return productRepository.save(product).toDto()
}
```

### 5.4 @Caching - รวม Annotations

```kotlin
import org.springframework.cache.annotation.Caching

// รวม Cacheable, CachePut, CacheEvict ในครั้งเดียว
@Caching(
    cacheable = [
        Cacheable(cacheNames = ["products"], key = "#id")
    ],
    evict = [
        CacheEvict(cacheNames = ["productList"], allEntries = true)
    ]
)
fun getOrLoadProduct(id: Long): ProductDto? {
    return productRepository.findById(id).map { it.toDto() }.orElse(null)
}
```

---

## 🔑 6. Custom Cache Key Generator

```kotlin
// src/main/kotlin/com/example/config/CustomKeyGenerator.kt
package com.example.config

import org.springframework.cache.interceptor.KeyGenerator
import org.springframework.stereotype.Component
import java.lang.reflect.Method

@Component("customKeyGenerator")
class CustomKeyGenerator : KeyGenerator {
    override fun generate(target: Any, method: Method, vararg params: Any?): Any {
        // สร้าง key จาก class name + method name + parameters
        val key = buildString {
            append(target.javaClass.simpleName)
            append(".")
            append(method.name)
            append("(")
            append(params.joinToString(",") { it?.toString() ?: "null" })
            append(")")
        }
        return key
    }
}
```

```kotlin
// ใช้งาน Custom Key Generator
@Cacheable(
    cacheNames = ["products"],
    keyGenerator = "customKeyGenerator"
)
fun getProductWithCustomKey(id: Long, version: String): ProductDto? {
    return productRepository.findByIdAndVersion(id, version)?.toDto()
}
```

---

## 📊 7. Cache Statistics และ Monitoring

```kotlin
// src/main/kotlin/com/example/service/CacheMonitorService.kt
package com.example.service

import org.springframework.cache.CacheManager
import org.springframework.data.redis.core.RedisTemplate
import org.springframework.stereotype.Service

@Service
class CacheMonitorService(
    private val cacheManager: CacheManager,
    private val redisTemplate: RedisTemplate<String, Any>
) {

    // ดูข้อมูล cache ทั้งหมด
    fun getCacheNames(): Collection<String> {
        return cacheManager.cacheNames
    }

    // นับ key ใน cache
    fun countCacheKeys(cacheName: String): Long {
        val pattern = "myapp:${cacheName}::*"
        val keys = redisTemplate.keys(pattern)
        return keys?.size?.toLong() ?: 0L
    }

    // ลบ cache ทั้งหมดสำหรับ cache name ที่ระบุ
    fun clearCache(cacheName: String) {
        cacheManager.getCache(cacheName)?.clear()
    }

    // ลบ cache ทั้งหมด
    fun clearAllCaches() {
        cacheManager.cacheNames.forEach { cacheName ->
            cacheManager.getCache(cacheName)?.clear()
        }
    }

    // ดู TTL ของ key
    fun getTtl(cacheName: String, key: String): Long {
        val fullKey = "myapp:${cacheName}::${key}"
        return redisTemplate.getExpire(fullKey) ?: -1
    }
}
```

```kotlin
// src/main/kotlin/com/example/controller/CacheController.kt
package com.example.controller

import com.example.service.CacheMonitorService
import org.springframework.http.ResponseEntity
import org.springframework.web.bind.annotation.*

@RestController
@RequestMapping("/api/cache")
class CacheController(private val cacheMonitorService: CacheMonitorService) {

    @GetMapping("/names")
    fun getCacheNames(): ResponseEntity<Collection<String>> {
        return ResponseEntity.ok(cacheMonitorService.getCacheNames())
    }

    @GetMapping("/{cacheName}/count")
    fun countKeys(@PathVariable cacheName: String): ResponseEntity<Map<String, Long>> {
        val count = cacheMonitorService.countCacheKeys(cacheName)
        return ResponseEntity.ok(mapOf("cacheName" to cacheName, "count" to count))
    }

    @DeleteMapping("/{cacheName}")
    fun clearCache(@PathVariable cacheName: String): ResponseEntity<Map<String, String>> {
        cacheMonitorService.clearCache(cacheName)
        return ResponseEntity.ok(mapOf("message" to "Cache '$cacheName' cleared"))
    }

    @DeleteMapping
    fun clearAllCaches(): ResponseEntity<Map<String, String>> {
        cacheMonitorService.clearAllCaches()
        return ResponseEntity.ok(mapOf("message" to "All caches cleared"))
    }
}
```

---

## 🛍️ 8. ตัวอย่าง: Product Catalog Caching (Complete)

```kotlin
// Entity
// src/main/kotlin/com/example/entity/Product.kt
package com.example.entity

import jakarta.persistence.*
import java.math.BigDecimal

@Entity
@Table(name = "products")
data class Product(
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long = 0,

    @Column(nullable = false)
    var name: String,

    @Column(nullable = false)
    var price: BigDecimal,

    @Column(length = 1000)
    var description: String? = null,

    @Column(name = "category_id")
    var categoryId: Long? = null,

    @Column(name = "stock_count")
    var stockCount: Int = 0,

    @Column(name = "is_active")
    var isActive: Boolean = true
) {
    fun toDto() = ProductDto(
        id = id,
        name = name,
        price = price,
        description = description,
        categoryId = categoryId,
        stockCount = stockCount,
        isActive = isActive
    )
}
```

```kotlin
// DTO
// src/main/kotlin/com/example/dto/ProductDtos.kt
package com.example.dto

import java.io.Serializable
import java.math.BigDecimal

// Serializable สำคัญมากสำหรับ Redis caching!
data class ProductDto(
    val id: Long,
    val name: String,
    val price: BigDecimal,
    val description: String?,
    val categoryId: Long?,
    val stockCount: Int,
    val isActive: Boolean
) : Serializable

data class CreateProductDto(
    val name: String,
    val price: BigDecimal,
    val description: String? = null,
    val categoryId: Long? = null,
    val stockCount: Int = 0
)

data class UpdateProductDto(
    val name: String? = null,
    val price: BigDecimal? = null,
    val description: String? = null,
    val stockCount: Int? = null
)
```

```kotlin
// Repository
// src/main/kotlin/com/example/repository/ProductRepository.kt
package com.example.repository

import com.example.entity.Product
import org.springframework.data.domain.Page
import org.springframework.data.domain.Pageable
import org.springframework.data.jpa.repository.JpaRepository
import org.springframework.data.jpa.repository.Query

interface ProductRepository : JpaRepository<Product, Long> {
    fun findByCategoryId(categoryId: Long, page: Int, size: Int): List<Product>
    fun findByIsActiveTrue(pageable: Pageable): Page<Product>
    
    @Query("SELECT p FROM Product p WHERE p.categoryId = :categoryId AND p.isActive = true")
    fun findActiveByCategoryId(categoryId: Long, pageable: Pageable): Page<Product>
}
```

```kotlin
// Complete Service with Caching
// src/main/kotlin/com/example/service/ProductCatalogService.kt
package com.example.service

import com.example.dto.*
import com.example.entity.Product
import com.example.exception.NotFoundException
import com.example.repository.ProductRepository
import org.springframework.cache.annotation.*
import org.springframework.data.domain.PageRequest
import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional
import java.math.BigDecimal

@Service
@Transactional(readOnly = true)
class ProductCatalogService(
    private val productRepository: ProductRepository
) {

    // Cache product by ID - TTL 30 minutes (configured in RedisConfig)
    @Cacheable(cacheNames = ["products"], key = "#id", unless = "#result == null")
    fun getProductById(id: Long): ProductDto? {
        println("[DB] Fetching product: $id")
        return productRepository.findById(id).map { it.toDto() }.orElse(null)
    }

    // Cache all active products - TTL 5 minutes
    @Cacheable(cacheNames = ["productList"], key = "'active:page:' + #page + ':size:' + #size")
    fun getActiveProducts(page: Int = 0, size: Int = 20): List<ProductDto> {
        println("[DB] Fetching active products page $page")
        val pageable = PageRequest.of(page, size)
        return productRepository.findByIsActiveTrue(pageable)
            .content
            .map { it.toDto() }
    }

    // Cache by category
    @Cacheable(
        cacheNames = ["productList"],
        key = "'category:' + #categoryId + ':page:' + #page",
        condition = "#categoryId != null"
    )
    fun getProductsByCategory(categoryId: Long, page: Int = 0): List<ProductDto> {
        println("[DB] Fetching products for category: $categoryId")
        val pageable = PageRequest.of(page, 20)
        return productRepository.findActiveByCategoryId(categoryId, pageable)
            .content
            .map { it.toDto() }
    }

    // Create และ cache ผลลัพธ์
    @Transactional
    @Caching(
        put = [CachePut(cacheNames = ["products"], key = "#result.id")],
        evict = [CacheEvict(cacheNames = ["productList"], allEntries = true)]
    )
    fun createProduct(dto: CreateProductDto): ProductDto {
        val product = Product(
            name = dto.name,
            price = dto.price,
            description = dto.description,
            categoryId = dto.categoryId,
            stockCount = dto.stockCount
        )
        val saved = productRepository.save(product)
        println("[DB] Created product: ${saved.id}")
        return saved.toDto()
    }

    // Update และ invalidate cache
    @Transactional
    @Caching(
        put = [CachePut(cacheNames = ["products"], key = "#id")],
        evict = [CacheEvict(cacheNames = ["productList"], allEntries = true)]
    )
    fun updateProduct(id: Long, dto: UpdateProductDto): ProductDto {
        val product = productRepository.findById(id)
            .orElseThrow { NotFoundException("Product $id not found") }

        product.apply {
            dto.name?.let { name = it }
            dto.price?.let { price = it }
            dto.description?.let { description = it }
            dto.stockCount?.let { stockCount = it }
        }

        val updated = productRepository.save(product)
        println("[DB] Updated product: $id")
        return updated.toDto()
    }

    // Delete และ evict cache
    @Transactional
    @Caching(
        evict = [
            CacheEvict(cacheNames = ["products"], key = "#id"),
            CacheEvict(cacheNames = ["productList"], allEntries = true)
        ]
    )
    fun deleteProduct(id: Long) {
        if (!productRepository.existsById(id)) {
            throw NotFoundException("Product $id not found")
        }
        productRepository.deleteById(id)
        println("[DB] Deleted product: $id")
    }

    // Update stock - cache invalidation
    @Transactional
    @CacheEvict(cacheNames = ["products"], key = "#id")
    fun updateStock(id: Long, quantity: Int): ProductDto {
        val product = productRepository.findById(id)
            .orElseThrow { NotFoundException("Product $id not found") }

        if (product.stockCount + quantity < 0) {
            throw IllegalStateException("Insufficient stock for product $id")
        }

        product.stockCount += quantity
        return productRepository.save(product).toDto()
    }
}
```

```kotlin
// Controller
// src/main/kotlin/com/example/controller/ProductController.kt
package com.example.controller

import com.example.dto.*
import com.example.service.ProductCatalogService
import org.springframework.http.HttpStatus
import org.springframework.http.ResponseEntity
import org.springframework.web.bind.annotation.*

@RestController
@RequestMapping("/api/products")
class ProductController(private val productCatalogService: ProductCatalogService) {

    @GetMapping("/{id}")
    fun getProduct(@PathVariable id: Long): ResponseEntity<ProductDto> {
        val product = productCatalogService.getProductById(id)
            ?: return ResponseEntity.notFound().build()
        return ResponseEntity.ok(product)
    }

    @GetMapping
    fun getActiveProducts(
        @RequestParam(defaultValue = "0") page: Int,
        @RequestParam(defaultValue = "20") size: Int
    ): ResponseEntity<List<ProductDto>> {
        return ResponseEntity.ok(productCatalogService.getActiveProducts(page, size))
    }

    @GetMapping("/category/{categoryId}")
    fun getByCategory(
        @PathVariable categoryId: Long,
        @RequestParam(defaultValue = "0") page: Int
    ): ResponseEntity<List<ProductDto>> {
        return ResponseEntity.ok(
            productCatalogService.getProductsByCategory(categoryId, page)
        )
    }

    @PostMapping
    fun createProduct(@RequestBody dto: CreateProductDto): ResponseEntity<ProductDto> {
        val created = productCatalogService.createProduct(dto)
        return ResponseEntity.status(HttpStatus.CREATED).body(created)
    }

    @PutMapping("/{id}")
    fun updateProduct(
        @PathVariable id: Long,
        @RequestBody dto: UpdateProductDto
    ): ResponseEntity<ProductDto> {
        val updated = productCatalogService.updateProduct(id, dto)
        return ResponseEntity.ok(updated)
    }

    @DeleteMapping("/{id}")
    fun deleteProduct(@PathVariable id: Long): ResponseEntity<Void> {
        productCatalogService.deleteProduct(id)
        return ResponseEntity.noContent().build()
    }
}
```

---

## 🧪 9. Testing Cache

```kotlin
// src/test/kotlin/com/example/service/ProductCachingTest.kt
package com.example.service

import com.example.dto.CreateProductDto
import org.junit.jupiter.api.Test
import org.springframework.beans.factory.annotation.Autowired
import org.springframework.boot.test.context.SpringBootTest
import org.springframework.cache.CacheManager
import org.springframework.test.context.DynamicPropertyRegistry
import org.springframework.test.context.DynamicPropertySource
import org.testcontainers.containers.GenericContainer
import org.testcontainers.junit.jupiter.Container
import org.testcontainers.junit.jupiter.Testcontainers
import java.math.BigDecimal
import kotlin.test.assertNotNull
import kotlin.test.assertNull

@SpringBootTest
@Testcontainers
class ProductCachingTest {

    companion object {
        @Container
        val redis = GenericContainer<Nothing>("redis:7-alpine").apply {
            withExposedPorts(6379)
        }

        @DynamicPropertySource
        @JvmStatic
        fun redisProperties(registry: DynamicPropertyRegistry) {
            registry.add("spring.data.redis.host") { redis.host }
            registry.add("spring.data.redis.port") { redis.getMappedPort(6379) }
        }
    }

    @Autowired
    lateinit var productCatalogService: ProductCatalogService

    @Autowired
    lateinit var cacheManager: CacheManager

    @Test
    fun `should cache product after first fetch`() {
        // ล้าง cache ก่อน
        cacheManager.getCache("products")?.clear()

        // First call - DB query
        val product = productCatalogService.getProductById(1L)

        // Cache ควรมีข้อมูลแล้ว
        val cached = cacheManager.getCache("products")?.get(1L)
        assertNotNull(cached)
    }

    @Test
    fun `should evict cache on delete`() {
        // ดึงข้อมูลก่อน (เพื่อ cache)
        productCatalogService.getProductById(1L)

        // Delete
        productCatalogService.deleteProduct(1L)

        // Cache ควรถูกลบแล้ว
        val cached = cacheManager.getCache("products")?.get(1L)
        assertNull(cached?.get())
    }

    @Test
    fun `should update cache on product update`() {
        val dto = CreateProductDto(
            name = "Test Product",
            price = BigDecimal("99.99")
        )
        val created = productCatalogService.createProduct(dto)

        // ตรวจสอบว่า cache มีข้อมูลหลัง create
        val cached = cacheManager.getCache("products")?.get(created.id)
        assertNotNull(cached)
    }
}
```

---

## 📊 10. Performance Comparison

```kotlin
// src/main/kotlin/com/example/service/BenchmarkService.kt
package com.example.service

import org.springframework.stereotype.Service
import kotlin.system.measureTimeMillis

@Service
class BenchmarkService(private val productCatalogService: ProductCatalogService) {

    fun comparePerformance(productId: Long): Map<String, Long> {
        // First call (cache miss)
        val firstCallTime = measureTimeMillis {
            productCatalogService.getProductById(productId)
        }

        // Second call (cache hit)
        val secondCallTime = measureTimeMillis {
            productCatalogService.getProductById(productId)
        }

        return mapOf(
            "firstCall_ms" to firstCallTime,    // ~50-200ms (DB query)
            "secondCall_ms" to secondCallTime,   // ~1-5ms (Redis hit)
            "improvement_factor" to (firstCallTime / maxOf(secondCallTime, 1))
        )
    }
}
```

---

## 🔧 11. Advanced Redis Operations

```kotlin
// src/main/kotlin/com/example/service/RedisDirectService.kt
package com.example.service

import org.springframework.data.redis.core.RedisTemplate
import org.springframework.stereotype.Service
import java.time.Duration

@Service
class RedisDirectService(
    private val redisTemplate: RedisTemplate<String, Any>
) {

    // Set key-value พร้อม TTL
    fun set(key: String, value: Any, ttl: Duration = Duration.ofMinutes(10)) {
        redisTemplate.opsForValue().set(key, value, ttl)
    }

    // Get value
    fun get(key: String): Any? {
        return redisTemplate.opsForValue().get(key)
    }

    // Delete key
    fun delete(key: String): Boolean {
        return redisTemplate.delete(key)
    }

    // Increment counter
    fun increment(key: String): Long {
        return redisTemplate.opsForValue().increment(key) ?: 0
    }

    // Hash operations
    fun setHash(key: String, field: String, value: Any) {
        redisTemplate.opsForHash<String, Any>().put(key, field, value)
    }

    fun getHash(key: String): Map<String, Any> {
        @Suppress("UNCHECKED_CAST")
        return redisTemplate.opsForHash<String, Any>().entries(key) as Map<String, Any>
    }

    // List operations
    fun pushToList(key: String, value: Any) {
        redisTemplate.opsForList().rightPush(key, value)
    }

    fun getList(key: String): List<Any?> {
        return redisTemplate.opsForList().range(key, 0, -1) ?: emptyList()
    }

    // Set operations (unique values)
    fun addToSet(key: String, vararg values: Any) {
        redisTemplate.opsForSet().add(key, *values)
    }

    // Check if key exists
    fun exists(key: String): Boolean {
        return redisTemplate.hasKey(key)
    }

    // Get TTL
    fun getTtl(key: String): Long {
        return redisTemplate.getExpire(key)
    }
}
```

---

## 📋 สรุปตาราง Cache Annotations

| Annotation | จุดประสงค์ | Execute Method? | ตัวอย่าง |
|------------|-----------|----------------|---------|
| `@Cacheable` | อ่านจาก cache ถ้ามี | เฉพาะ cache miss | `getProduct(id)` |
| `@CachePut` | เขียนลง cache เสมอ | เสมอ | `updateProduct(id, dto)` |
| `@CacheEvict` | ลบออกจาก cache | เสมอ | `deleteProduct(id)` |
| `@Caching` | รวม annotations | ขึ้นอยู่กับ annotations ภายใน | `createProduct(dto)` |

### Cache Attributes

| Attribute | ประเภท | คำอธิบาย |
|-----------|--------|---------|
| `cacheNames` | String[] | ชื่อ cache(s) ที่ใช้ |
| `key` | SpEL | Cache key expression |
| `condition` | SpEL | ตรวจก่อน execute (true = cache) |
| `unless` | SpEL | ตรวจหลัง execute (true = ไม่ cache) |
| `keyGenerator` | String | Custom key generator bean name |
| `cacheManager` | String | CacheManager bean name |

### Redis TTL Best Practices

| ข้อมูล | TTL แนะนำ | เหตุผล |
|--------|-----------|--------|
| Product list | 5 นาที | เปลี่ยนบ่อย |
| Product detail | 30 นาที | เปลี่ยนน้อย |
| Category | 1 ชั่วโมง | เปลี่ยนน้อยมาก |
| User profile | 15 นาที | ความ sensitive สูง |
| Static content | 24 ชั่วโมง | แทบไม่เปลี่ยน |

---

*Part 31/100+ | Kotlin & Spring Boot Complete Course*
