# Part 40: Mini Project 2 - E-Commerce API
## Full E-Commerce API ที่รวมทุกสิ่งที่เรียนมา

---

## 🎯 เป้าหมายของ Part นี้

- สร้าง complete E-Commerce API ด้วย Spring Boot + Kotlin
- JWT Authentication
- Product, Category, Order, Cart management
- Redis Caching
- RabbitMQ Messaging
- Spring Events
- Email Notifications
- Scheduled Tasks
- Audit Logging

---

## 🏗️ 1. Project Structure

```
ecommerce-api/
├── src/main/kotlin/com/example/ecommerce/
│   ├── EcommerceApplication.kt
│   ├── config/
│   │   ├── SecurityConfig.kt
│   │   ├── JwtConfig.kt
│   │   ├── RedisConfig.kt
│   │   ├── RabbitMQConfig.kt
│   │   └── AsyncConfig.kt
│   ├── entity/
│   │   ├── User.kt
│   │   ├── Category.kt
│   │   ├── Product.kt
│   │   ├── Cart.kt
│   │   ├── CartItem.kt
│   │   ├── Order.kt
│   │   ├── OrderItem.kt
│   │   └── AuditableEntity.kt
│   ├── repository/
│   │   ├── UserRepository.kt
│   │   ├── CategoryRepository.kt
│   │   ├── ProductRepository.kt
│   │   ├── CartRepository.kt
│   │   ├── OrderRepository.kt
│   │   └── AuditLogRepository.kt
│   ├── service/
│   │   ├── AuthService.kt
│   │   ├── UserService.kt
│   │   ├── CategoryService.kt
│   │   ├── ProductService.kt
│   │   ├── CartService.kt
│   │   ├── OrderService.kt
│   │   ├── EmailService.kt
│   │   └── AuditService.kt
│   ├── controller/
│   │   ├── AuthController.kt
│   │   ├── CategoryController.kt
│   │   ├── ProductController.kt
│   │   ├── CartController.kt
│   │   └── OrderController.kt
│   ├── dto/
│   │   ├── AuthDtos.kt
│   │   ├── ProductDtos.kt
│   │   ├── CartDtos.kt
│   │   └── OrderDtos.kt
│   ├── event/
│   │   ├── OrderEvents.kt
│   │   └── UserEvents.kt
│   ├── listener/
│   │   ├── OrderEventListener.kt
│   │   └── UserEventListener.kt
│   ├── messaging/
│   │   ├── OrderMessageProducer.kt
│   │   └── OrderMessageConsumer.kt
│   ├── scheduler/
│   │   └── EcommerceScheduler.kt
│   └── security/
│       ├── JwtFilter.kt
│       └── UserDetailsServiceImpl.kt
└── src/main/resources/
    ├── application.yml
    └── db/migration/
        └── V1__init.sql
```

---

## 📦 2. Build Configuration

```kotlin
// build.gradle.kts
import org.jetbrains.kotlin.gradle.tasks.KotlinCompile

plugins {
    id("org.springframework.boot") version "3.1.5"
    id("io.spring.dependency-management") version "1.1.3"
    kotlin("jvm") version "1.9.10"
    kotlin("plugin.spring") version "1.9.10"
    kotlin("plugin.jpa") version "1.9.10"
}

group = "com.example"
version = "0.0.1-SNAPSHOT"

java {
    sourceCompatibility = JavaVersion.VERSION_17
}

repositories {
    mavenCentral()
}

dependencies {
    // Spring Boot
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
    implementation("org.springframework.boot:spring-boot-starter-security")
    implementation("org.springframework.boot:spring-boot-starter-validation")
    implementation("org.springframework.boot:spring-boot-starter-cache")

    // Database
    implementation("org.springframework.boot:spring-boot-starter-data-redis")
    runtimeOnly("org.postgresql:postgresql")
    implementation("org.flywaydb:flyway-core")

    // Messaging
    implementation("org.springframework.boot:spring-boot-starter-amqp")

    // Email
    implementation("org.springframework.boot:spring-boot-starter-mail")
    implementation("org.springframework.boot:spring-boot-starter-thymeleaf")

    // Kotlin
    implementation("com.fasterxml.jackson.module:jackson-module-kotlin")
    implementation("org.jetbrains.kotlin:kotlin-reflect")
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:1.7.3")

    // JWT
    implementation("io.jsonwebtoken:jjwt-api:0.12.3")
    runtimeOnly("io.jsonwebtoken:jjwt-impl:0.12.3")
    runtimeOnly("io.jsonwebtoken:jjwt-jackson:0.12.3")

    // Swagger/OpenAPI
    implementation("org.springdoc:springdoc-openapi-starter-webmvc-ui:2.2.0")

    // Testing
    testImplementation("org.springframework.boot:spring-boot-starter-test")
    testImplementation("org.springframework.security:spring-security-test")
    testImplementation("org.testcontainers:postgresql:1.19.0")
    testImplementation("io.mockk:mockk:1.13.8")
}

tasks.withType<KotlinCompile> {
    kotlinOptions {
        freeCompilerArgs += "-Xjsr305=strict"
        jvmTarget = "17"
    }
}
```

---

## ⚙️ 3. Application Configuration

```yaml
# application.yml
server:
  port: 8080

spring:
  application:
    name: ecommerce-api

  datasource:
    url: jdbc:postgresql://localhost:5432/ecommerce
    username: postgres
    password: password
    driver-class-name: org.postgresql.Driver

  jpa:
    hibernate:
      ddl-auto: validate
    properties:
      hibernate:
        dialect: org.hibernate.dialect.PostgreSQLDialect
        format_sql: true
    show-sql: false
    open-in-view: false

  flyway:
    enabled: true
    locations: classpath:db/migration

  data:
    redis:
      host: localhost
      port: 6379
      timeout: 2000ms

  rabbitmq:
    host: localhost
    port: 5672
    username: admin
    password: password

  mail:
    host: smtp.gmail.com
    port: 587
    username: ${MAIL_USERNAME:}
    password: ${MAIL_PASSWORD:}
    properties:
      mail.smtp.auth: true
      mail.smtp.starttls.enable: true

  cache:
    type: redis

app:
  jwt:
    secret: ${JWT_SECRET:mySecretKeyForDevelopmentPurposesOnly12345}
    expiration: 86400000  # 24 hours
  mail:
    from: noreply@ecommerce.com
    from-name: E-Commerce
    base-url: ${APP_BASE_URL:http://localhost:3000}
```

---

## 🗃️ 4. Database Migration

```sql
-- src/main/resources/db/migration/V1__init.sql

-- Users
CREATE TABLE users (
    id          BIGSERIAL PRIMARY KEY,
    email       VARCHAR(255) UNIQUE NOT NULL,
    username    VARCHAR(100) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    full_name   VARCHAR(255),
    phone       VARCHAR(20),
    role        VARCHAR(50) DEFAULT 'CUSTOMER',
    is_active   BOOLEAN DEFAULT TRUE,
    created_at  TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMP,
    created_by  VARCHAR(100),
    updated_by  VARCHAR(100)
);

-- Categories
CREATE TABLE categories (
    id          BIGSERIAL PRIMARY KEY,
    name        VARCHAR(255) UNIQUE NOT NULL,
    description TEXT,
    slug        VARCHAR(255) UNIQUE NOT NULL,
    parent_id   BIGINT REFERENCES categories(id),
    is_active   BOOLEAN DEFAULT TRUE,
    sort_order  INT DEFAULT 0,
    created_at  TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMP,
    created_by  VARCHAR(100),
    updated_by  VARCHAR(100)
);

-- Products
CREATE TABLE products (
    id          BIGSERIAL PRIMARY KEY,
    sku         VARCHAR(100) UNIQUE NOT NULL,
    name        VARCHAR(255) NOT NULL,
    description TEXT,
    price       DECIMAL(12, 2) NOT NULL,
    sale_price  DECIMAL(12, 2),
    cost_price  DECIMAL(12, 2),
    category_id BIGINT REFERENCES categories(id),
    stock_count INT DEFAULT 0,
    image_url   VARCHAR(500),
    is_active   BOOLEAN DEFAULT TRUE,
    created_at  TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMP,
    created_by  VARCHAR(100),
    updated_by  VARCHAR(100)
);

-- Carts
CREATE TABLE carts (
    id          BIGSERIAL PRIMARY KEY,
    user_id     BIGINT UNIQUE REFERENCES users(id),
    created_at  TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMP
);

CREATE TABLE cart_items (
    id          BIGSERIAL PRIMARY KEY,
    cart_id     BIGINT NOT NULL REFERENCES carts(id) ON DELETE CASCADE,
    product_id  BIGINT NOT NULL REFERENCES products(id),
    quantity    INT NOT NULL DEFAULT 1,
    price       DECIMAL(12, 2) NOT NULL,
    created_at  TIMESTAMP NOT NULL DEFAULT NOW(),
    UNIQUE(cart_id, product_id)
);

-- Orders
CREATE TABLE orders (
    id              BIGSERIAL PRIMARY KEY,
    order_number    VARCHAR(50) UNIQUE NOT NULL,
    user_id         BIGINT NOT NULL REFERENCES users(id),
    status          VARCHAR(50) NOT NULL DEFAULT 'PENDING',
    total_amount    DECIMAL(12, 2) NOT NULL,
    shipping_address TEXT,
    payment_method  VARCHAR(50),
    payment_status  VARCHAR(50) DEFAULT 'PENDING',
    notes           TEXT,
    created_at      TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMP,
    created_by      VARCHAR(100),
    updated_by      VARCHAR(100)
);

CREATE TABLE order_items (
    id          BIGSERIAL PRIMARY KEY,
    order_id    BIGINT NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
    product_id  BIGINT NOT NULL REFERENCES products(id),
    quantity    INT NOT NULL,
    unit_price  DECIMAL(12, 2) NOT NULL,
    total_price DECIMAL(12, 2) NOT NULL
);

-- Audit Logs
CREATE TABLE audit_logs (
    id              BIGSERIAL PRIMARY KEY,
    entity_type     VARCHAR(100) NOT NULL,
    entity_id       BIGINT NOT NULL,
    action          VARCHAR(50) NOT NULL,
    performed_by    VARCHAR(100),
    performed_at    TIMESTAMP NOT NULL DEFAULT NOW(),
    ip_address      VARCHAR(50),
    old_values      TEXT,
    new_values      TEXT,
    changed_fields  TEXT,
    details         TEXT
);

CREATE INDEX idx_audit_entity ON audit_logs(entity_type, entity_id);
CREATE INDEX idx_audit_user ON audit_logs(performed_by);
CREATE INDEX idx_audit_time ON audit_logs(performed_at);

-- ShedLock
CREATE TABLE shedlock (
    name       VARCHAR(64)  NOT NULL,
    lock_until TIMESTAMP    NOT NULL,
    locked_at  TIMESTAMP    NOT NULL,
    locked_by  VARCHAR(255) NOT NULL,
    PRIMARY KEY (name)
);
```

---

## 🔐 5. Security Configuration

```kotlin
// src/main/kotlin/com/example/ecommerce/config/SecurityConfig.kt
package com.example.ecommerce.config

import com.example.ecommerce.security.JwtAuthFilter
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.security.authentication.AuthenticationManager
import org.springframework.security.config.annotation.authentication.configuration.AuthenticationConfiguration
import org.springframework.security.config.annotation.method.configuration.EnableMethodSecurity
import org.springframework.security.config.annotation.web.builders.HttpSecurity
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity
import org.springframework.security.config.http.SessionCreationPolicy
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder
import org.springframework.security.crypto.password.PasswordEncoder
import org.springframework.security.web.SecurityFilterChain
import org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter

@Configuration
@EnableWebSecurity
@EnableMethodSecurity
class SecurityConfig(private val jwtAuthFilter: JwtAuthFilter) {

    @Bean
    fun securityFilterChain(http: HttpSecurity): SecurityFilterChain {
        return http
            .csrf { it.disable() }
            .sessionManagement { it.sessionCreationPolicy(SessionCreationPolicy.STATELESS) }
            .authorizeHttpRequests { auth ->
                auth
                    .requestMatchers("/api/auth/**").permitAll()
                    .requestMatchers("/api/products/**").permitAll()
                    .requestMatchers("/api/categories/**").permitAll()
                    .requestMatchers("/swagger-ui/**", "/api-docs/**").permitAll()
                    .requestMatchers("/api/admin/**").hasRole("ADMIN")
                    .anyRequest().authenticated()
            }
            .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter::class.java)
            .build()
    }

    @Bean
    fun passwordEncoder(): PasswordEncoder = BCryptPasswordEncoder()

    @Bean
    fun authenticationManager(config: AuthenticationConfiguration): AuthenticationManager =
        config.authenticationManager
}
```

---

## 👤 6. Authentication

```kotlin
// src/main/kotlin/com/example/ecommerce/service/AuthService.kt
package com.example.ecommerce.service

import com.example.ecommerce.dto.*
import com.example.ecommerce.entity.User
import com.example.ecommerce.event.UserRegisteredEvent
import com.example.ecommerce.repository.UserRepository
import com.example.ecommerce.security.JwtService
import org.springframework.context.ApplicationEventPublisher
import org.springframework.security.authentication.AuthenticationManager
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken
import org.springframework.security.crypto.password.PasswordEncoder
import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional

@Service
class AuthService(
    private val userRepository: UserRepository,
    private val passwordEncoder: PasswordEncoder,
    private val jwtService: JwtService,
    private val authManager: AuthenticationManager,
    private val eventPublisher: ApplicationEventPublisher
) {

    @Transactional
    fun register(dto: RegisterDto): AuthResponse {
        if (userRepository.existsByEmail(dto.email)) {
            throw IllegalArgumentException("Email already in use")
        }

        val user = User(
            email = dto.email,
            username = dto.username,
            passwordHash = passwordEncoder.encode(dto.password),
            fullName = dto.fullName
        )
        val saved = userRepository.save(user)

        eventPublisher.publishEvent(UserRegisteredEvent(this, saved))

        val token = jwtService.generateToken(saved.email)
        return AuthResponse(
            token = token,
            userId = saved.id,
            email = saved.email,
            username = saved.username,
            role = saved.role.name
        )
    }

    fun login(dto: LoginDto): AuthResponse {
        authManager.authenticate(
            UsernamePasswordAuthenticationToken(dto.email, dto.password)
        )

        val user = userRepository.findByEmail(dto.email)
            ?: throw IllegalArgumentException("User not found")

        val token = jwtService.generateToken(user.email)
        return AuthResponse(
            token = token,
            userId = user.id,
            email = user.email,
            username = user.username,
            role = user.role.name
        )
    }

    fun refreshToken(token: String): AuthResponse {
        val email = jwtService.extractEmail(token)
        val user = userRepository.findByEmail(email)
            ?: throw IllegalArgumentException("User not found")

        val newToken = jwtService.generateToken(user.email)
        return AuthResponse(
            token = newToken,
            userId = user.id,
            email = user.email,
            username = user.username,
            role = user.role.name
        )
    }
}
```

---

## 🛍️ 7. Product Service (with Caching)

```kotlin
// src/main/kotlin/com/example/ecommerce/service/ProductService.kt
package com.example.ecommerce.service

import com.example.ecommerce.dto.*
import com.example.ecommerce.entity.Product
import com.example.ecommerce.repository.ProductRepository
import com.example.ecommerce.service.AuditService
import com.example.ecommerce.entity.AuditAction
import org.springframework.cache.annotation.*
import org.springframework.data.domain.Page
import org.springframework.data.domain.PageRequest
import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional

@Service
@Transactional(readOnly = true)
class ProductService(
    private val productRepository: ProductRepository,
    private val auditService: AuditService
) {

    @Cacheable(cacheNames = ["products"], key = "#id", unless = "#result == null")
    fun getById(id: Long): ProductDto? {
        return productRepository.findById(id).orElse(null)?.toDto()
    }

    @Cacheable(
        cacheNames = ["productList"],
        key = "'active:p' + #page + ':s' + #size"
    )
    fun getActiveProducts(page: Int = 0, size: Int = 20): Page<ProductDto> {
        val pageable = PageRequest.of(page, size)
        return productRepository.findByIsActiveTrue(pageable).map { it.toDto() }
    }

    @Cacheable(
        cacheNames = ["productList"],
        key = "'cat:' + #categoryId + ':p' + #page"
    )
    fun getByCategory(categoryId: Long, page: Int = 0): Page<ProductDto> {
        val pageable = PageRequest.of(page, 20)
        return productRepository.findByCategoryIdAndIsActiveTrue(categoryId, pageable)
            .map { it.toDto() }
    }

    fun searchProducts(query: String, page: Int = 0): Page<ProductDto> {
        val pageable = PageRequest.of(page, 20)
        return productRepository.searchByNameOrDescription(query, pageable).map { it.toDto() }
    }

    @Transactional
    @Caching(
        put = [CachePut(cacheNames = ["products"], key = "#result.id")],
        evict = [CacheEvict(cacheNames = ["productList"], allEntries = true)]
    )
    fun createProduct(dto: CreateProductDto): ProductDto {
        val product = Product(
            sku = dto.sku,
            name = dto.name,
            description = dto.description,
            price = dto.price,
            salePrice = dto.salePrice,
            categoryId = dto.categoryId,
            stockCount = dto.stockCount
        )
        val saved = productRepository.save(product)
        auditService.log("PRODUCT", saved.id, AuditAction.CREATE, newValues = saved.toDto())
        return saved.toDto()
    }

    @Transactional
    @Caching(
        put = [CachePut(cacheNames = ["products"], key = "#id")],
        evict = [CacheEvict(cacheNames = ["productList"], allEntries = true)]
    )
    fun updateProduct(id: Long, dto: UpdateProductDto): ProductDto {
        val product = productRepository.findById(id)
            .orElseThrow { NoSuchElementException("Product $id not found") }
        val oldDto = product.toDto()

        product.apply {
            dto.name?.let { name = it }
            dto.description?.let { description = it }
            dto.price?.let { price = it }
            dto.salePrice?.let { salePrice = it }
            dto.stockCount?.let { stockCount = it }
        }

        val saved = productRepository.save(product)
        auditService.log("PRODUCT", id, AuditAction.UPDATE, oldValues = oldDto, newValues = saved.toDto())
        return saved.toDto()
    }

    @Transactional
    @Caching(
        evict = [
            CacheEvict(cacheNames = ["products"], key = "#id"),
            CacheEvict(cacheNames = ["productList"], allEntries = true)
        ]
    )
    fun deleteProduct(id: Long) {
        val product = productRepository.findById(id)
            .orElseThrow { NoSuchElementException("Product $id not found") }
        productRepository.delete(product)
        auditService.log("PRODUCT", id, AuditAction.DELETE)
    }
}
```

---

## 🛒 8. Cart Service

```kotlin
// src/main/kotlin/com/example/ecommerce/service/CartService.kt
package com.example.ecommerce.service

import com.example.ecommerce.dto.*
import com.example.ecommerce.entity.*
import com.example.ecommerce.repository.*
import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional

@Service
@Transactional
class CartService(
    private val cartRepository: CartRepository,
    private val cartItemRepository: CartItemRepository,
    private val productRepository: ProductRepository
) {

    @Transactional(readOnly = true)
    fun getCart(userId: Long): CartDto {
        val cart = cartRepository.findByUserId(userId)
            ?: return CartDto(userId = userId, items = emptyList(), totalAmount = java.math.BigDecimal.ZERO)
        return cart.toDto()
    }

    fun addItem(userId: Long, dto: AddCartItemDto): CartDto {
        val product = productRepository.findById(dto.productId)
            .orElseThrow { NoSuchElementException("Product not found") }

        if (!product.isActive) throw IllegalStateException("Product is not available")
        if (product.stockCount < dto.quantity) throw IllegalStateException("Insufficient stock")

        val cart = cartRepository.findByUserId(userId)
            ?: cartRepository.save(Cart(userId = userId))

        val existingItem = cartItemRepository.findByCartIdAndProductId(cart.id, dto.productId)
        if (existingItem != null) {
            existingItem.quantity += dto.quantity
            cartItemRepository.save(existingItem)
        } else {
            cartItemRepository.save(
                CartItem(
                    cartId = cart.id,
                    productId = dto.productId,
                    quantity = dto.quantity,
                    price = product.salePrice ?: product.price
                )
            )
        }

        return getCart(userId)
    }

    fun updateItem(userId: Long, itemId: Long, quantity: Int): CartDto {
        val cart = cartRepository.findByUserId(userId)
            ?: throw NoSuchElementException("Cart not found")

        val item = cartItemRepository.findByIdAndCartId(itemId, cart.id)
            ?: throw NoSuchElementException("Cart item not found")

        if (quantity <= 0) {
            cartItemRepository.delete(item)
        } else {
            val product = productRepository.findById(item.productId).orElseThrow()
            if (product.stockCount < quantity) throw IllegalStateException("Insufficient stock")
            item.quantity = quantity
            cartItemRepository.save(item)
        }

        return getCart(userId)
    }

    fun removeItem(userId: Long, itemId: Long): CartDto {
        val cart = cartRepository.findByUserId(userId)
            ?: throw NoSuchElementException("Cart not found")

        val item = cartItemRepository.findByIdAndCartId(itemId, cart.id)
            ?: throw NoSuchElementException("Item not found")

        cartItemRepository.delete(item)
        return getCart(userId)
    }

    fun clearCart(userId: Long) {
        val cart = cartRepository.findByUserId(userId) ?: return
        cartItemRepository.deleteByCartId(cart.id)
    }
}
```

---

## 📦 9. Order Service (with Events and Messaging)

```kotlin
// src/main/kotlin/com/example/ecommerce/service/OrderService.kt
package com.example.ecommerce.service

import com.example.ecommerce.dto.*
import com.example.ecommerce.entity.*
import com.example.ecommerce.event.OrderPlacedEvent
import com.example.ecommerce.messaging.OrderMessageProducer
import com.example.ecommerce.repository.*
import org.springframework.context.ApplicationEventPublisher
import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional
import java.math.BigDecimal
import java.time.LocalDateTime
import java.util.UUID

@Service
@Transactional
class OrderService(
    private val orderRepository: OrderRepository,
    private val orderItemRepository: OrderItemRepository,
    private val cartService: CartService,
    private val productRepository: ProductRepository,
    private val userRepository: UserRepository,
    private val orderMessageProducer: OrderMessageProducer,
    private val eventPublisher: ApplicationEventPublisher,
    private val auditService: AuditService
) {

    fun placeOrder(userId: Long, dto: PlaceOrderDto): OrderDto {
        val user = userRepository.findById(userId)
            .orElseThrow { NoSuchElementException("User not found") }

        val cart = cartService.getCart(userId)
        if (cart.items.isEmpty()) throw IllegalStateException("Cart is empty")

        // Validate stock สำหรับทุก item
        cart.items.forEach { item ->
            val product = productRepository.findById(item.productId).orElseThrow()
            if (product.stockCount < item.quantity) {
                throw IllegalStateException("Insufficient stock for: ${product.name}")
            }
        }

        // คำนวณราคา
        val totalAmount = cart.items.sumOf { it.price * it.quantity.toBigDecimal() }

        // สร้าง Order
        val order = Order(
            orderNumber = "ORD-${UUID.randomUUID().toString().take(8).uppercase()}",
            userId = userId,
            totalAmount = totalAmount,
            shippingAddress = dto.shippingAddress,
            paymentMethod = dto.paymentMethod,
            notes = dto.notes
        )
        val savedOrder = orderRepository.save(order)

        // สร้าง Order Items และลด stock
        val orderItems = cart.items.map { cartItem ->
            val product = productRepository.findById(cartItem.productId).orElseThrow()
            product.stockCount -= cartItem.quantity
            productRepository.save(product)

            orderItemRepository.save(
                OrderItem(
                    orderId = savedOrder.id,
                    productId = cartItem.productId,
                    quantity = cartItem.quantity,
                    unitPrice = cartItem.price,
                    totalPrice = cartItem.price * cartItem.quantity.toBigDecimal()
                )
            )
        }

        // ล้าง cart
        cartService.clearCart(userId)

        // Audit
        auditService.log("ORDER", savedOrder.id, AuditAction.CREATE, newValues = savedOrder.toDto())

        // Publish event (for email, notification)
        eventPublisher.publishEvent(
            OrderPlacedEvent(
                source = this,
                orderId = savedOrder.id,
                userId = userId,
                customerEmail = user.email,
                orderNumber = savedOrder.orderNumber,
                totalAmount = totalAmount,
                items = orderItems
            )
        )

        // ส่ง message ไป queue (for payment processing)
        orderMessageProducer.sendOrderForProcessing(savedOrder.toMessage(user))

        return savedOrder.toDto(orderItems)
    }

    @Transactional(readOnly = true)
    fun getOrderById(orderId: Long, userId: Long): OrderDto {
        val order = orderRepository.findByIdAndUserId(orderId, userId)
            ?: throw NoSuchElementException("Order not found")
        val items = orderItemRepository.findByOrderId(orderId)
        return order.toDto(items)
    }

    @Transactional(readOnly = true)
    fun getUserOrders(userId: Long, page: Int = 0): org.springframework.data.domain.Page<OrderSummaryDto> {
        val pageable = org.springframework.data.domain.PageRequest.of(page, 10)
        return orderRepository.findByUserIdOrderByCreatedAtDesc(userId, pageable)
            .map { it.toSummaryDto() }
    }

    fun updateOrderStatus(orderId: Long, status: OrderStatus): OrderDto {
        val order = orderRepository.findById(orderId)
            .orElseThrow { NoSuchElementException("Order not found") }

        val oldStatus = order.status
        order.status = status
        val saved = orderRepository.save(order)

        auditService.log(
            "ORDER", orderId, AuditAction.UPDATE,
            details = "Status changed: $oldStatus → $status"
        )

        return saved.toDto(orderItemRepository.findByOrderId(orderId))
    }

    fun cancelOrder(orderId: Long, userId: Long, reason: String? = null): OrderDto {
        val order = orderRepository.findByIdAndUserId(orderId, userId)
            ?: throw NoSuchElementException("Order not found")

        if (order.status !in listOf(OrderStatus.PENDING, OrderStatus.CONFIRMED)) {
            throw IllegalStateException("Cannot cancel order in status: ${order.status}")
        }

        // คืน stock
        val items = orderItemRepository.findByOrderId(orderId)
        items.forEach { item ->
            val product = productRepository.findById(item.productId).orElse(null)
            product?.let {
                it.stockCount += item.quantity
                productRepository.save(it)
            }
        }

        order.status = OrderStatus.CANCELLED
        val saved = orderRepository.save(order)

        auditService.log("ORDER", orderId, AuditAction.UPDATE, details = "Cancelled: $reason")

        return saved.toDto(items)
    }
}
```

---

## 🌐 10. Controllers

```kotlin
// src/main/kotlin/com/example/ecommerce/controller/AuthController.kt
package com.example.ecommerce.controller

import com.example.ecommerce.dto.*
import com.example.ecommerce.service.AuthService
import jakarta.validation.Valid
import org.springframework.http.ResponseEntity
import org.springframework.web.bind.annotation.*

@RestController
@RequestMapping("/api/auth")
class AuthController(private val authService: AuthService) {

    @PostMapping("/register")
    fun register(@Valid @RequestBody dto: RegisterDto): ResponseEntity<AuthResponse> {
        return ResponseEntity.ok(authService.register(dto))
    }

    @PostMapping("/login")
    fun login(@Valid @RequestBody dto: LoginDto): ResponseEntity<AuthResponse> {
        return ResponseEntity.ok(authService.login(dto))
    }

    @PostMapping("/refresh")
    fun refresh(@RequestHeader("Authorization") token: String): ResponseEntity<AuthResponse> {
        val actualToken = token.removePrefix("Bearer ")
        return ResponseEntity.ok(authService.refreshToken(actualToken))
    }
}
```

```kotlin
// src/main/kotlin/com/example/ecommerce/controller/OrderController.kt
package com.example.ecommerce.controller

import com.example.ecommerce.dto.*
import com.example.ecommerce.service.OrderService
import jakarta.validation.Valid
import org.springframework.data.domain.Page
import org.springframework.http.HttpStatus
import org.springframework.http.ResponseEntity
import org.springframework.security.core.annotation.AuthenticationPrincipal
import org.springframework.web.bind.annotation.*

@RestController
@RequestMapping("/api/orders")
class OrderController(private val orderService: OrderService) {

    @PostMapping
    fun placeOrder(
        @AuthenticationPrincipal userId: Long,
        @Valid @RequestBody dto: PlaceOrderDto
    ): ResponseEntity<OrderDto> {
        val order = orderService.placeOrder(userId, dto)
        return ResponseEntity.status(HttpStatus.CREATED).body(order)
    }

    @GetMapping("/{orderId}")
    fun getOrder(
        @PathVariable orderId: Long,
        @AuthenticationPrincipal userId: Long
    ): ResponseEntity<OrderDto> {
        return ResponseEntity.ok(orderService.getOrderById(orderId, userId))
    }

    @GetMapping
    fun getMyOrders(
        @AuthenticationPrincipal userId: Long,
        @RequestParam(defaultValue = "0") page: Int
    ): ResponseEntity<Page<OrderSummaryDto>> {
        return ResponseEntity.ok(orderService.getUserOrders(userId, page))
    }

    @PostMapping("/{orderId}/cancel")
    fun cancelOrder(
        @PathVariable orderId: Long,
        @AuthenticationPrincipal userId: Long,
        @RequestParam(required = false) reason: String?
    ): ResponseEntity<OrderDto> {
        return ResponseEntity.ok(orderService.cancelOrder(orderId, userId, reason))
    }
}
```

---

## 📅 11. Ecommerce Scheduler

```kotlin
// src/main/kotlin/com/example/ecommerce/scheduler/EcommerceScheduler.kt
package com.example.ecommerce.scheduler

import com.example.ecommerce.repository.OrderRepository
import com.example.ecommerce.entity.OrderStatus
import net.javacrumbs.shedlock.spring.annotation.SchedulerLock
import org.springframework.scheduling.annotation.Scheduled
import org.springframework.stereotype.Component
import org.springframework.transaction.annotation.Transactional
import java.time.LocalDateTime

@Component
class EcommerceScheduler(
    private val orderRepository: OrderRepository
) {

    // ยกเลิก orders ที่ค้างอยู่เกิน 24 ชั่วโมง
    @Scheduled(cron = "0 0 2 * * *")
    @SchedulerLock(name = "cancelStaleOrders", lockAtMostFor = "30m")
    @Transactional
    fun cancelStaleOrders() {
        val cutoff = LocalDateTime.now().minusHours(24)
        val staleOrders = orderRepository.findByStatusAndCreatedAtBefore(
            OrderStatus.PENDING, cutoff
        )
        staleOrders.forEach { order ->
            order.status = OrderStatus.CANCELLED
            orderRepository.save(order)
        }
        println("[Scheduler] Cancelled ${staleOrders.size} stale orders")
    }

    // Daily sales summary
    @Scheduled(cron = "0 0 6 * * *")
    @SchedulerLock(name = "dailySalesSummary", lockAtMostFor = "15m")
    fun generateDailySalesSummary() {
        val yesterday = LocalDateTime.now().minusDays(1)
        val orders = orderRepository.findCompletedOrdersForDay(yesterday)
        val revenue = orders.sumOf { it.totalAmount }
        println("[Scheduler] Yesterday: ${orders.size} orders, revenue: ฿$revenue")
    }
}
```

---

## 🧪 12. Testing

```kotlin
// src/test/kotlin/com/example/ecommerce/service/OrderServiceTest.kt
package com.example.ecommerce.service

import com.example.ecommerce.dto.PlaceOrderDto
import io.mockk.*
import org.junit.jupiter.api.Test
import org.springframework.context.ApplicationEventPublisher
import kotlin.test.assertNotNull
import kotlin.test.assertEquals

class OrderServiceTest {

    private val orderRepository = mockk<OrderRepository>()
    private val cartService = mockk<CartService>()
    private val productRepository = mockk<ProductRepository>()
    private val userRepository = mockk<UserRepository>()
    private val orderItemRepository = mockk<OrderItemRepository>()
    private val orderMessageProducer = mockk<OrderMessageProducer>()
    private val eventPublisher = mockk<ApplicationEventPublisher>()
    private val auditService = mockk<AuditService>()

    private val orderService = OrderService(
        orderRepository, orderItemRepository, cartService,
        productRepository, userRepository, orderMessageProducer,
        eventPublisher, auditService
    )

    @Test
    fun `should place order successfully`() {
        val userId = 1L
        val dto = PlaceOrderDto(
            shippingAddress = "123 Main St",
            paymentMethod = "CREDIT_CARD"
        )

        // Setup mocks...
        every { userRepository.findById(userId) } returns java.util.Optional.of(
            User(id = userId, email = "test@example.com", username = "test", passwordHash = "hash")
        )
        every { cartService.getCart(userId) } returns CartDto(
            userId = userId,
            items = listOf(CartItemDto(id = 1L, productId = 1L, quantity = 2, price = java.math.BigDecimal("50.00"))),
            totalAmount = java.math.BigDecimal("100.00")
        )
        every { productRepository.findById(1L) } returns java.util.Optional.of(
            Product(id = 1L, sku = "P001", name = "Product 1", price = java.math.BigDecimal("50.00"), stockCount = 10)
        )
        every { productRepository.save(any()) } returnsArgument 0
        every { orderRepository.save(any()) } returnsArgument 0
        every { orderItemRepository.save(any()) } returnsArgument 0
        justRun { cartService.clearCart(userId) }
        justRun { eventPublisher.publishEvent(any()) }
        justRun { orderMessageProducer.sendOrderForProcessing(any()) }
        justRun { auditService.log(any(), any(), any(), any(), any(), any()) }

        val result = orderService.placeOrder(userId, dto)

        assertNotNull(result)
        verify { eventPublisher.publishEvent(any()) }
        verify { orderMessageProducer.sendOrderForProcessing(any()) }
    }
}
```

---

## 📋 13. API Endpoints Summary

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/api/auth/register` | Public | สมัครสมาชิก |
| POST | `/api/auth/login` | Public | เข้าสู่ระบบ |
| GET | `/api/products` | Public | รายการสินค้า |
| GET | `/api/products/{id}` | Public | รายละเอียดสินค้า |
| POST | `/api/products` | Admin | เพิ่มสินค้า |
| GET | `/api/categories` | Public | รายการหมวดหมู่ |
| GET | `/api/cart` | User | ดูตะกร้า |
| POST | `/api/cart/items` | User | เพิ่มสินค้าในตะกร้า |
| PUT | `/api/cart/items/{id}` | User | แก้ไขจำนวน |
| DELETE | `/api/cart/items/{id}` | User | ลบออกจากตะกร้า |
| POST | `/api/orders` | User | สั่งซื้อ |
| GET | `/api/orders` | User | รายการคำสั่งซื้อ |
| GET | `/api/orders/{id}` | User | รายละเอียดคำสั่งซื้อ |
| POST | `/api/orders/{id}/cancel` | User | ยกเลิกคำสั่งซื้อ |
| GET | `/api/admin/orders` | Admin | จัดการคำสั่งซื้อ |
| GET | `/api/admin/audit` | Admin | ดู audit logs |

---

## 🏆 สรุป Features ของ Mini Project

| Feature | Technology | Part อ้างอิง |
|---------|-----------|------------|
| REST API | Spring Boot Web | Part 23 |
| Database | Spring Data JPA + PostgreSQL | Part 24 |
| Validation | Spring Validation | Part 25 |
| Authentication | JWT + Spring Security | Part 28 |
| Caching | Spring Cache + Redis | Part 31 |
| Async Messaging | RabbitMQ | Part 32 |
| Events | Spring Events | Part 38 |
| Email | Spring Mail + Thymeleaf | Part 36 |
| Scheduled Tasks | @Scheduled + ShedLock | Part 37 |
| Audit Logging | Spring Data Auditing | Part 39 |
| Testing | JUnit 5 + MockK | Part 29 |
| Deployment | Docker | Part 30 |

---

*Part 40/100+ | Kotlin & Spring Boot Complete Course*
