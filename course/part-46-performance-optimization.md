# Part 46: Performance Optimization
## การปรับแต่ง Performance ของ Spring Boot Application

---

## 🎯 เป้าหมายของ Part นี้

- เข้าใจและแก้ N+1 Query Problem
- @EntityGraph สำหรับ eager loading
- Projection interfaces สำหรับ lightweight queries
- Database indexing strategies
- Query optimization techniques
- Connection pooling ด้วย HikariCP
- Caching strategies
- ตัวอย่างจริง: Optimize slow API endpoint

---

## 📚 1. N+1 Query Problem

ปัญหาที่พบบ่อยที่สุดใน JPA applications:

```kotlin
// ปัญหา: โหลด 100 orders = 1 + 100 queries!
val orders = orderRepository.findAll()  // Query 1: SELECT * FROM orders

orders.forEach { order ->
    // Query 2-101: SELECT * FROM users WHERE id = ?
    println(order.user.name)
    
    // Query 102-201: SELECT * FROM order_items WHERE order_id = ?
    order.items.forEach { item ->
        println(item.productName)
    }
}
// รวม: 1 + 100 + 100 = 201 queries!
```

### ตรวจจับ N+1 ด้วย Logging

```yaml
# application.yml
spring:
  jpa:
    show-sql: true
    properties:
      hibernate:
        generate_statistics: true
        format_sql: true

logging:
  level:
    org.hibernate.stat: DEBUG
    org.hibernate.SQL: DEBUG
    org.hibernate.type.descriptor.sql: TRACE
```

---

## 🔧 2. Dependencies

```kotlin
// build.gradle.kts
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-cache")
    
    // P6Spy สำหรับ SQL monitoring
    implementation("p6spy:p6spy:3.9.1")
    
    // Querydsl สำหรับ type-safe queries
    implementation("com.querydsl:querydsl-jpa:5.0.0:jakarta")
    kapt("com.querydsl:querydsl-apt:5.0.0:jakarta")
    kapt("jakarta.persistence:jakarta.persistence-api")
    
    runtimeOnly("com.mysql:mysql-connector-j")
    testImplementation("org.springframework.boot:spring-boot-starter-test")
}
```

---

## 📦 3. Entities

```kotlin
// entity/Order.kt
package com.example.perf.entity

import jakarta.persistence.*

@Entity
@Table(name = "orders")
data class Order(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long = 0,

    @ManyToOne(fetch = FetchType.LAZY)  // LAZY เป็น default ที่ดี
    @JoinColumn(name = "user_id")
    val user: User? = null,

    @OneToMany(
        mappedBy = "order",
        fetch = FetchType.LAZY,         // LAZY เสมอสำหรับ collections
        cascade = [CascadeType.ALL],
        orphanRemoval = true
    )
    val items: MutableList<OrderItem> = mutableListOf(),

    @Column(name = "total_amount")
    val totalAmount: Double = 0.0,

    @Enumerated(EnumType.STRING)
    val status: OrderStatus = OrderStatus.PENDING,

    @Column(name = "order_number", unique = true)
    val orderNumber: String = ""
)

@Entity
@Table(name = "users")
data class User(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long = 0,

    val firstName: String = "",
    val lastName: String = "",
    val email: String = "",

    @OneToMany(mappedBy = "user", fetch = FetchType.LAZY)
    val orders: MutableList<Order> = mutableListOf()
)

@Entity
@Table(name = "order_items")
data class OrderItem(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long = 0,

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "order_id")
    val order: Order? = null,

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "product_id")
    val product: Product? = null,

    val productName: String = "",  // snapshot
    val productPrice: Double = 0.0,
    val quantity: Int = 0,
    val subtotal: Double = 0.0
)

@Entity
@Table(name = "products")
data class Product(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long = 0,

    val name: String = "",
    val price: Double = 0.0,
    val stockQuantity: Int = 0,

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "category_id")
    val category: Category? = null
)

@Entity
@Table(name = "categories")
data class Category(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long = 0,
    val name: String = ""
)

enum class OrderStatus { PENDING, CONFIRMED, SHIPPED, DELIVERED, CANCELLED }
```

---

## 🚀 4. แก้ N+1 ด้วย @EntityGraph

```kotlin
// repository/OrderRepository.kt
package com.example.perf.repository

import com.example.perf.entity.Order
import org.springframework.data.jpa.repository.EntityGraph
import org.springframework.data.jpa.repository.JpaRepository
import org.springframework.data.jpa.repository.Query

interface OrderRepository : JpaRepository<Order, Long> {

    // ❌ Bad: N+1 problem
    fun findAll(): List<Order>

    // ✅ Good: @EntityGraph - Load user + items ใน query เดียว
    @EntityGraph(attributePaths = ["user", "items", "items.product"])
    fun findAllWithDetails(): List<Order>

    // ✅ Good: JPQL JOIN FETCH
    @Query("""
        SELECT DISTINCT o FROM Order o
        LEFT JOIN FETCH o.user
        LEFT JOIN FETCH o.items i
        LEFT JOIN FETCH i.product
        WHERE o.status = :status
    """)
    fun findByStatusWithDetails(status: String): List<Order>

    // Named EntityGraph (defined on entity)
    @EntityGraph(value = "Order.withUser")
    fun findByOrderNumber(orderNumber: String): Order?

    // Batch loading สำหรับ collection (แนะนำเมื่อ collection ใหญ่)
    @Query("SELECT o FROM Order o WHERE o.user.id = :userId")
    fun findByUserId(userId: Long): List<Order>
}
```

```kotlin
// entity/Order.kt (เพิ่ม Named EntityGraph)
@Entity
@Table(name = "orders")
@NamedEntityGraphs(
    NamedEntityGraph(
        name = "Order.withUser",
        attributeNodes = [NamedAttributeNode("user")]
    ),
    NamedEntityGraph(
        name = "Order.withUserAndItems",
        attributeNodes = [
            NamedAttributeNode("user"),
            NamedAttributeNode(value = "items", subgraph = "items-subgraph")
        ],
        subgraphs = [
            NamedSubgraph(
                name = "items-subgraph",
                attributeNodes = [NamedAttributeNode("product")]
            )
        ]
    )
)
data class Order(/* ... */)
```

---

## 📊 5. Projection Interfaces

เมื่อต้องการ query เฉพาะบาง fields (ไม่ต้อง load ทั้ง entity)

```kotlin
// projection/OrderProjections.kt
package com.example.perf.projection

import java.time.LocalDateTime

// Interface projection - ดีมาก performance
interface OrderSummary {
    val id: Long
    val orderNumber: String
    val totalAmount: Double
    val status: String
    val customerName: String  // derived from user.firstName + lastName
        get() = ""  // JPQL expression จัดการให้
}

// DTO projection - type-safe และเร็วที่สุด
data class OrderDto(
    val id: Long,
    val orderNumber: String,
    val totalAmount: Double,
    val status: String,
    val customerFirstName: String,
    val customerEmail: String,
    val itemCount: Long
)

// Nested projection
interface OrderWithItemsView {
    val id: Long
    val orderNumber: String
    val totalAmount: Double
    val user: UserView

    interface UserView {
        val id: Long
        val firstName: String
        val lastName: String
    }
}
```

```kotlin
// repository/OrderRepository.kt (เพิ่ม projection queries)
interface OrderRepository : JpaRepository<Order, Long> {

    // Interface projection
    @Query("SELECT o.id as id, o.orderNumber as orderNumber, o.totalAmount as totalAmount, " +
           "o.status as status FROM Order o")
    fun findAllSummaries(): List<OrderSummary>

    // DTO projection (JPQL constructor expression)
    @Query("""
        SELECT new com.example.perf.projection.OrderDto(
            o.id, o.orderNumber, o.totalAmount, o.status,
            u.firstName, u.email,
            COUNT(i)
        )
        FROM Order o
        JOIN o.user u
        LEFT JOIN o.items i
        GROUP BY o.id, o.orderNumber, o.totalAmount, o.status, u.firstName, u.email
    """)
    fun findAllOrderDtos(): List<OrderDto>

    // Nested projection
    fun findAllBy(): List<OrderWithItemsView>
}
```

---

## 🔎 6. Query Optimization

```kotlin
// repository/ProductRepository.kt
package com.example.perf.repository

import com.example.perf.entity.Product
import org.springframework.data.jpa.repository.JpaRepository
import org.springframework.data.jpa.repository.Modifying
import org.springframework.data.jpa.repository.Query
import org.springframework.transaction.annotation.Transactional

interface ProductRepository : JpaRepository<Product, Long> {

    // ✅ Specific fields only (avoid SELECT *)
    @Query("SELECT p.id, p.name, p.price, p.stockQuantity FROM Product p WHERE p.category.id = :categoryId")
    fun findBasicByCategoryId(categoryId: Long): List<Array<Any>>

    // ✅ Pagination + Sorting
    fun findByCategoryId(
        categoryId: Long,
        pageable: org.springframework.data.domain.Pageable
    ): org.springframework.data.domain.Page<Product>

    // ✅ Count query (ไม่ต้อง load data)
    fun countByCategoryId(categoryId: Long): Long

    // ✅ Exists query (เร็วกว่า findById != null)
    fun existsByEmail(email: String): Boolean

    // ✅ Bulk update (1 query แทน N queries)
    @Modifying
    @Transactional
    @Query("UPDATE Product p SET p.stockQuantity = p.stockQuantity - :quantity WHERE p.id = :id AND p.stockQuantity >= :quantity")
    fun decrementStock(id: Long, quantity: Int): Int

    // ✅ Native query สำหรับ complex analytics
    @Query(
        value = """
            SELECT 
                p.id,
                p.name,
                COUNT(oi.id) as total_orders,
                SUM(oi.quantity) as total_quantity_sold,
                SUM(oi.subtotal) as total_revenue
            FROM products p
            LEFT JOIN order_items oi ON p.id = oi.product_id
            LEFT JOIN orders o ON oi.order_id = o.id AND o.status = 'DELIVERED'
            WHERE p.deleted_at IS NULL
            GROUP BY p.id, p.name
            ORDER BY total_revenue DESC
            LIMIT :limit
        """,
        nativeQuery = true
    )
    fun findTopSellingProducts(limit: Int): List<Map<String, Any>>
}
```

---

## 💾 7. HikariCP Connection Pool

```yaml
# application.yml - HikariCP Configuration
spring:
  datasource:
    hikari:
      # Core pool settings
      maximum-pool-size: 20          # สูงสุด 20 connections
      minimum-idle: 5                # รักษา idle connections ไว้ 5
      idle-timeout: 600000           # 10 นาที ก่อน idle connection ถูกปิด
      max-lifetime: 1800000          # 30 นาที max lifetime ของ connection
      connection-timeout: 30000      # 30 วินาที timeout ในการขอ connection
      
      # Health check
      keepalive-time: 60000          # Send keepalive query ทุก 1 นาที
      connection-test-query: SELECT 1
      
      # Pool name สำหรับ monitoring
      pool-name: HikariPool-Main
      
      # Leak detection
      leak-detection-threshold: 2000  # แจ้งเตือนถ้า connection ถือไว้ > 2 วินาที
```

```kotlin
// config/DataSourceConfig.kt
package com.example.perf.config

import com.zaxxer.hikari.HikariConfig
import com.zaxxer.hikari.HikariDataSource
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import javax.sql.DataSource

@Configuration
class DataSourceConfig {

    @Bean
    fun dataSource(): DataSource {
        val config = HikariConfig().apply {
            jdbcUrl = "jdbc:mysql://localhost:3306/myapp"
            username = "root"
            password = "password"
            driverClassName = "com.mysql.cj.jdbc.Driver"

            // Connection pool
            maximumPoolSize = 20
            minimumIdle = 5
            idleTimeout = 600_000L
            maxLifetime = 1_800_000L
            connectionTimeout = 30_000L

            // Performance
            addDataSourceProperty("cachePrepStmts", "true")
            addDataSourceProperty("prepStmtCacheSize", "250")
            addDataSourceProperty("prepStmtCacheSqlLimit", "2048")
            addDataSourceProperty("useServerPrepStmts", "true")
            addDataSourceProperty("useLocalSessionState", "true")
            addDataSourceProperty("rewriteBatchedStatements", "true")
            addDataSourceProperty("cacheResultSetMetadata", "true")
            addDataSourceProperty("cacheServerConfiguration", "true")
            addDataSourceProperty("elideSetAutoCommits", "true")
            addDataSourceProperty("maintainTimeStats", "false")
        }

        return HikariDataSource(config)
    }
}
```

---

## 🗄️ 8. Database Indexing

```sql
-- สร้าง indexes สำหรับ common query patterns

-- Compound index สำหรับ ORDER BY + WHERE
CREATE INDEX idx_orders_user_created 
    ON orders (user_id, created_at DESC);

-- Covering index (include columns ที่ query ต้องการ)
CREATE INDEX idx_products_category_price 
    ON products (category_id, price, name, id);

-- Partial index สำหรับ MySQL (ใช้ WHERE ใน index expression)
-- MySQL ไม่รองรับ partial index แต่ใช้ prefix index ได้
CREATE INDEX idx_users_email_prefix 
    ON users (email(100));

-- Full-text index สำหรับ search
ALTER TABLE products 
    ADD FULLTEXT INDEX ft_products_name (name, description);

-- ตรวจสอบ index usage
EXPLAIN SELECT * FROM orders 
WHERE user_id = 1 AND created_at > '2024-01-01' 
ORDER BY created_at DESC;
```

---

## ⚡ 9. Caching Layer

```kotlin
// service/ProductService.kt
package com.example.perf.service

import com.example.perf.entity.Product
import com.example.perf.repository.ProductRepository
import org.springframework.cache.annotation.CacheEvict
import org.springframework.cache.annotation.CachePut
import org.springframework.cache.annotation.Cacheable
import org.springframework.stereotype.Service

@Service
class ProductService(private val productRepository: ProductRepository) {

    @Cacheable(
        value = ["products"],
        key = "#id",
        unless = "#result == null"
    )
    fun getProductById(id: Long): Product? {
        return productRepository.findById(id).orElse(null)
    }

    @Cacheable(
        value = ["products:category"],
        key = "#categoryId + ':' + #page + ':' + #size"
    )
    fun getProductsByCategory(categoryId: Long, page: Int, size: Int): List<Product> {
        val pageable = org.springframework.data.domain.PageRequest.of(page, size)
        return productRepository.findByCategoryId(categoryId, pageable).content
    }

    @CachePut(value = ["products"], key = "#result.id")
    @CacheEvict(value = ["products:category"], allEntries = true)
    fun updateProduct(id: Long, updates: Map<String, Any>): Product {
        val product = productRepository.findById(id)
            .orElseThrow { NoSuchElementException("Product not found") }
        // apply updates...
        return productRepository.save(product)
    }

    @CacheEvict(value = ["products", "products:category"], allEntries = true)
    fun invalidateAllProductCaches() {
        // เรียกเมื่อต้องการ clear cache ทั้งหมด
    }
}
```

---

## 🧪 10. Performance Testing

```kotlin
// test/PerformanceTest.kt
package com.example.perf

import org.junit.jupiter.api.Test
import org.springframework.beans.factory.annotation.Autowired
import org.springframework.boot.test.context.SpringBootTest
import org.springframework.transaction.annotation.Transactional
import kotlin.system.measureTimeMillis

@SpringBootTest
class PerformanceTest {

    @Autowired
    private lateinit var orderRepository: com.example.perf.repository.OrderRepository

    @Test
    @Transactional
    fun `compare query performance`() {
        // ทดสอบ N+1 vs JOIN FETCH
        val timeWithoutFetch = measureTimeMillis {
            val orders = orderRepository.findAll()
            orders.forEach { order ->
                order.user?.firstName  // trigger lazy load
                order.items.size       // trigger lazy load
            }
        }

        val timeWithFetch = measureTimeMillis {
            val orders = orderRepository.findAllWithDetails()
            orders.forEach { order ->
                order.user?.firstName  // already loaded
                order.items.size       // already loaded
            }
        }

        println("Without JOIN FETCH: ${timeWithoutFetch}ms")
        println("With JOIN FETCH: ${timeWithFetch}ms")
        println("Improvement: ${((timeWithoutFetch - timeWithFetch).toDouble() / timeWithoutFetch * 100).toInt()}%")

        assert(timeWithFetch < timeWithoutFetch) {
            "JOIN FETCH ควรเร็วกว่า N+1"
        }
    }
}
```

---

## 📊 สรุปเนื้อหา

| ปัญหา | วิธีแก้ | ผลลัพธ์ |
|-------|---------|---------|
| N+1 Queries | @EntityGraph / JOIN FETCH | ลด queries จาก N+1 เป็น 1-2 |
| SELECT * | Projection interfaces | ลด data transfer |
| Missing indexes | Database indexing | เร็วขึ้น 10-100x |
| Too many connections | HikariCP tuning | Better throughput |
| Repeated queries | Caching (@Cacheable) | ลด DB load |
| Large result sets | Pagination | ป้องกัน OOM |

### Quick Wins:

1. เพิ่ม `spring.jpa.show-sql=true` เพื่อดู SQL ที่ generate
2. ใช้ EXPLAIN ใน MySQL เพื่อดู query plan
3. ตรวจสอบ slow query log ใน production
4. ใช้ `@BatchSize` สำหรับ collection ที่ load หลายครั้ง

---

*Part 46/100+ | Kotlin & Spring Boot Complete Course*
