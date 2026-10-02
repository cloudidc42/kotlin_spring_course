# Part 55: Microservices Mini Project - ระบบ E-Commerce สมบูรณ์แบบ

## บทนำ

ในบทนี้เราจะสร้าง **Microservices System** ที่สมบูรณ์แบบสำหรับระบบ E-Commerce โดยประกอบด้วย 3 services หลัก ได้แก่ User Service, Product Service และ Order Service พร้อมด้วย Eureka Service Discovery, API Gateway, Kafka Messaging และ Docker Compose ครบถ้วน

## โครงสร้างโปรเจกต์

```
ecommerce-microservices/
├── eureka-server/
├── api-gateway/
├── user-service/
├── product-service/
├── order-service/
├── docker-compose.yml
└── README.md
```

## 1. Eureka Server - Service Discovery

### การตั้งค่า Eureka Server

```kotlin
// eureka-server/build.gradle.kts
dependencies {
    implementation("org.springframework.cloud:spring-cloud-starter-netflix-eureka-server")
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-security")
}
```

```kotlin
// EurekaServerApplication.kt
package com.ecommerce.eureka

import org.springframework.boot.autoconfigure.SpringBootApplication
import org.springframework.boot.runApplication
import org.springframework.cloud.netflix.eureka.server.EnableEurekaServer

@SpringBootApplication
@EnableEurekaServer
class EurekaServerApplication

fun main(args: Array<String>) {
    runApplication<EurekaServerApplication>(*args)
}
```

```yaml
# eureka-server/src/main/resources/application.yml
server:
  port: 8761

spring:
  application:
    name: eureka-server
  security:
    user:
      name: eureka
      password: eureka-password

eureka:
  instance:
    hostname: localhost
  client:
    register-with-eureka: false
    fetch-registry: false
    service-url:
      defaultZone: http://eureka:eureka-password@localhost:8761/eureka/
  server:
    wait-time-in-ms-when-sync-empty: 0
    enable-self-preservation: false
```

## 2. API Gateway

### การตั้งค่า Spring Cloud Gateway

```kotlin
// api-gateway/build.gradle.kts
dependencies {
    implementation("org.springframework.cloud:spring-cloud-starter-gateway")
    implementation("org.springframework.cloud:spring-cloud-starter-netflix-eureka-client")
    implementation("org.springframework.boot:spring-boot-starter-security")
    implementation("io.jsonwebtoken:jjwt-api:0.11.5")
    runtimeOnly("io.jsonwebtoken:jjwt-impl:0.11.5")
    runtimeOnly("io.jsonwebtoken:jjwt-jackson:0.11.5")
}
```

```kotlin
// ApiGatewayApplication.kt
package com.ecommerce.gateway

import org.springframework.boot.autoconfigure.SpringBootApplication
import org.springframework.boot.runApplication
import org.springframework.cloud.client.discovery.EnableDiscoveryClient

@SpringBootApplication
@EnableDiscoveryClient
class ApiGatewayApplication

fun main(args: Array<String>) {
    runApplication<ApiGatewayApplication>(*args)
}
```

```yaml
# api-gateway/src/main/resources/application.yml
server:
  port: 8080

spring:
  application:
    name: api-gateway
  cloud:
    gateway:
      routes:
        - id: user-service
          uri: lb://user-service
          predicates:
            - Path=/api/users/**
          filters:
            - AuthenticationFilter
            - name: CircuitBreaker
              args:
                name: userServiceCircuitBreaker
                fallbackUri: forward:/fallback/users

        - id: product-service
          uri: lb://product-service
          predicates:
            - Path=/api/products/**
          filters:
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 10
                redis-rate-limiter.burstCapacity: 20

        - id: order-service
          uri: lb://order-service
          predicates:
            - Path=/api/orders/**
          filters:
            - AuthenticationFilter

eureka:
  client:
    service-url:
      defaultZone: http://eureka:eureka-password@localhost:8761/eureka/
```

### JWT Authentication Filter

```kotlin
// AuthenticationFilter.kt
package com.ecommerce.gateway.filter

import io.jsonwebtoken.Jwts
import org.springframework.beans.factory.annotation.Value
import org.springframework.cloud.gateway.filter.GatewayFilter
import org.springframework.cloud.gateway.filter.factory.AbstractGatewayFilterFactory
import org.springframework.http.HttpHeaders
import org.springframework.http.HttpStatus
import org.springframework.stereotype.Component
import org.springframework.web.server.ServerWebExchange
import reactor.core.publisher.Mono

@Component
class AuthenticationFilter(
    private val routeValidator: RouteValidator
) : AbstractGatewayFilterFactory<AuthenticationFilter.Config>(Config::class.java) {

    @Value("\${jwt.secret}")
    private lateinit var secret: String

    override fun apply(config: Config): GatewayFilter {
        return GatewayFilter { exchange, chain ->
            if (routeValidator.isSecured(exchange.request)) {
                val authHeader = exchange.request.headers.getFirst(HttpHeaders.AUTHORIZATION)
                
                if (authHeader == null || !authHeader.startsWith("Bearer ")) {
                    return@GatewayFilter onError(exchange, HttpStatus.UNAUTHORIZED)
                }

                val token = authHeader.substring(7)
                
                try {
                    val claims = Jwts.parserBuilder()
                        .setSigningKey(secret.toByteArray())
                        .build()
                        .parseClaimsJws(token)
                        .body
                    
                    val modifiedRequest = exchange.request.mutate()
                        .header("X-User-Id", claims.subject)
                        .header("X-User-Role", claims["role"].toString())
                        .build()
                    
                    chain.filter(exchange.mutate().request(modifiedRequest).build())
                } catch (e: Exception) {
                    onError(exchange, HttpStatus.UNAUTHORIZED)
                }
            } else {
                chain.filter(exchange)
            }
        }
    }

    private fun onError(exchange: ServerWebExchange, status: HttpStatus): Mono<Void> {
        exchange.response.statusCode = status
        return exchange.response.setComplete()
    }

    class Config
}
```

```kotlin
// RouteValidator.kt
package com.ecommerce.gateway.filter

import org.springframework.http.server.reactive.ServerHttpRequest
import org.springframework.stereotype.Component

@Component
class RouteValidator {
    companion object {
        val openApiEndpoints = listOf(
            "/api/users/login",
            "/api/users/register",
            "/api/products",
            "/eureka"
        )
    }

    fun isSecured(request: ServerHttpRequest): Boolean {
        return openApiEndpoints.none { 
            request.uri.path.contains(it)
        }
    }
}
```

## 3. User Service

### Domain Model

```kotlin
// User.kt
package com.ecommerce.user.domain

import jakarta.persistence.*
import org.springframework.data.annotation.CreatedDate
import org.springframework.data.annotation.LastModifiedDate
import java.time.LocalDateTime

@Entity
@Table(name = "users")
data class User(
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long = 0,
    
    @Column(unique = true, nullable = false)
    val email: String,
    
    @Column(nullable = false)
    val password: String,
    
    @Column(nullable = false)
    val firstName: String,
    
    @Column(nullable = false)
    val lastName: String,
    
    @Enumerated(EnumType.STRING)
    val role: Role = Role.CUSTOMER,
    
    @Column(nullable = false)
    val active: Boolean = true,
    
    @CreatedDate
    val createdAt: LocalDateTime = LocalDateTime.now(),
    
    @LastModifiedDate
    val updatedAt: LocalDateTime = LocalDateTime.now()
)

enum class Role {
    CUSTOMER, ADMIN, SELLER
}
```

### Repository และ Service

```kotlin
// UserRepository.kt
package com.ecommerce.user.repository

import com.ecommerce.user.domain.User
import org.springframework.data.jpa.repository.JpaRepository
import org.springframework.stereotype.Repository
import java.util.Optional

@Repository
interface UserRepository : JpaRepository<User, Long> {
    fun findByEmail(email: String): Optional<User>
    fun existsByEmail(email: String): Boolean
}
```

```kotlin
// UserService.kt
package com.ecommerce.user.service

import com.ecommerce.user.domain.User
import com.ecommerce.user.dto.*
import com.ecommerce.user.event.UserCreatedEvent
import com.ecommerce.user.exception.UserAlreadyExistsException
import com.ecommerce.user.exception.UserNotFoundException
import com.ecommerce.user.repository.UserRepository
import org.springframework.kafka.core.KafkaTemplate
import org.springframework.security.crypto.password.PasswordEncoder
import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional

@Service
class UserService(
    private val userRepository: UserRepository,
    private val passwordEncoder: PasswordEncoder,
    private val kafkaTemplate: KafkaTemplate<String, UserCreatedEvent>
) {

    @Transactional
    fun registerUser(request: RegisterRequest): UserResponse {
        if (userRepository.existsByEmail(request.email)) {
            throw UserAlreadyExistsException("Email ${request.email} already exists")
        }

        val user = User(
            email = request.email,
            password = passwordEncoder.encode(request.password),
            firstName = request.firstName,
            lastName = request.lastName
        )

        val savedUser = userRepository.save(user)
        
        // ส่ง event ไปยัง Kafka เมื่อสร้าง user สำเร็จ
        val event = UserCreatedEvent(
            userId = savedUser.id,
            email = savedUser.email,
            firstName = savedUser.firstName,
            lastName = savedUser.lastName
        )
        kafkaTemplate.send("user-created", savedUser.id.toString(), event)

        return savedUser.toResponse()
    }

    fun getUserById(id: Long): UserResponse {
        return userRepository.findById(id)
            .orElseThrow { UserNotFoundException("User with id $id not found") }
            .toResponse()
    }

    fun getUserByEmail(email: String): UserResponse {
        return userRepository.findByEmail(email)
            .orElseThrow { UserNotFoundException("User with email $email not found") }
            .toResponse()
    }

    @Transactional
    fun updateUser(id: Long, request: UpdateUserRequest): UserResponse {
        val user = userRepository.findById(id)
            .orElseThrow { UserNotFoundException("User with id $id not found") }

        val updatedUser = user.copy(
            firstName = request.firstName ?: user.firstName,
            lastName = request.lastName ?: user.lastName
        )

        return userRepository.save(updatedUser).toResponse()
    }

    fun getAllUsers(): List<UserResponse> {
        return userRepository.findAll().map { it.toResponse() }
    }
}

fun User.toResponse() = UserResponse(
    id = id,
    email = email,
    firstName = firstName,
    lastName = lastName,
    role = role.name,
    active = active,
    createdAt = createdAt
)
```

### DTOs

```kotlin
// UserDtos.kt
package com.ecommerce.user.dto

import jakarta.validation.constraints.Email
import jakarta.validation.constraints.NotBlank
import jakarta.validation.constraints.Size
import java.time.LocalDateTime

data class RegisterRequest(
    @field:Email(message = "Invalid email format")
    @field:NotBlank(message = "Email is required")
    val email: String,
    
    @field:NotBlank(message = "Password is required")
    @field:Size(min = 8, message = "Password must be at least 8 characters")
    val password: String,
    
    @field:NotBlank(message = "First name is required")
    val firstName: String,
    
    @field:NotBlank(message = "Last name is required")
    val lastName: String
)

data class LoginRequest(
    @field:Email
    val email: String,
    val password: String
)

data class LoginResponse(
    val token: String,
    val userId: Long,
    val email: String,
    val role: String
)

data class UpdateUserRequest(
    val firstName: String? = null,
    val lastName: String? = null
)

data class UserResponse(
    val id: Long,
    val email: String,
    val firstName: String,
    val lastName: String,
    val role: String,
    val active: Boolean,
    val createdAt: LocalDateTime
)
```

### Kafka Event

```kotlin
// UserCreatedEvent.kt
package com.ecommerce.user.event

data class UserCreatedEvent(
    val userId: Long,
    val email: String,
    val firstName: String,
    val lastName: String
)
```

### Controller

```kotlin
// UserController.kt
package com.ecommerce.user.controller

import com.ecommerce.user.dto.*
import com.ecommerce.user.service.AuthService
import com.ecommerce.user.service.UserService
import jakarta.validation.Valid
import org.springframework.http.HttpStatus
import org.springframework.http.ResponseEntity
import org.springframework.web.bind.annotation.*

@RestController
@RequestMapping("/api/users")
class UserController(
    private val userService: UserService,
    private val authService: AuthService
) {

    @PostMapping("/register")
    fun register(@Valid @RequestBody request: RegisterRequest): ResponseEntity<UserResponse> {
        val user = userService.registerUser(request)
        return ResponseEntity.status(HttpStatus.CREATED).body(user)
    }

    @PostMapping("/login")
    fun login(@Valid @RequestBody request: LoginRequest): ResponseEntity<LoginResponse> {
        val response = authService.login(request)
        return ResponseEntity.ok(response)
    }

    @GetMapping("/{id}")
    fun getUserById(@PathVariable id: Long): ResponseEntity<UserResponse> {
        return ResponseEntity.ok(userService.getUserById(id))
    }

    @PutMapping("/{id}")
    fun updateUser(
        @PathVariable id: Long,
        @Valid @RequestBody request: UpdateUserRequest
    ): ResponseEntity<UserResponse> {
        return ResponseEntity.ok(userService.updateUser(id, request))
    }

    @GetMapping
    fun getAllUsers(): ResponseEntity<List<UserResponse>> {
        return ResponseEntity.ok(userService.getAllUsers())
    }
}
```

## 4. Product Service

### Domain Model

```kotlin
// Product.kt
package com.ecommerce.product.domain

import jakarta.persistence.*
import java.math.BigDecimal
import java.time.LocalDateTime

@Entity
@Table(name = "products")
data class Product(
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long = 0,
    
    @Column(nullable = false)
    val name: String,
    
    @Column(length = 1000)
    val description: String = "",
    
    @Column(nullable = false, precision = 10, scale = 2)
    val price: BigDecimal,
    
    @Column(nullable = false)
    val stock: Int = 0,
    
    @Column(nullable = false)
    val category: String,
    
    val imageUrl: String? = null,
    
    val active: Boolean = true,
    
    val createdAt: LocalDateTime = LocalDateTime.now(),
    
    val updatedAt: LocalDateTime = LocalDateTime.now()
)
```

### Product Service

```kotlin
// ProductService.kt
package com.ecommerce.product.service

import com.ecommerce.product.domain.Product
import com.ecommerce.product.dto.*
import com.ecommerce.product.event.StockUpdatedEvent
import com.ecommerce.product.exception.InsufficientStockException
import com.ecommerce.product.exception.ProductNotFoundException
import com.ecommerce.product.repository.ProductRepository
import org.springframework.kafka.annotation.KafkaListener
import org.springframework.kafka.core.KafkaTemplate
import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional

@Service
class ProductService(
    private val productRepository: ProductRepository,
    private val kafkaTemplate: KafkaTemplate<String, StockUpdatedEvent>
) {

    fun getAllProducts(category: String? = null): List<ProductResponse> {
        val products = if (category != null) {
            productRepository.findByCategory(category)
        } else {
            productRepository.findAllByActiveTrue()
        }
        return products.map { it.toResponse() }
    }

    fun getProductById(id: Long): ProductResponse {
        return productRepository.findById(id)
            .orElseThrow { ProductNotFoundException("Product $id not found") }
            .toResponse()
    }

    @Transactional
    fun createProduct(request: CreateProductRequest): ProductResponse {
        val product = Product(
            name = request.name,
            description = request.description,
            price = request.price,
            stock = request.stock,
            category = request.category,
            imageUrl = request.imageUrl
        )
        return productRepository.save(product).toResponse()
    }

    @Transactional
    fun updateStock(productId: Long, quantity: Int) {
        val product = productRepository.findById(productId)
            .orElseThrow { ProductNotFoundException("Product $productId not found") }

        if (product.stock < quantity) {
            throw InsufficientStockException(
                "Insufficient stock for product $productId. Available: ${product.stock}, Requested: $quantity"
            )
        }

        val updatedProduct = product.copy(stock = product.stock - quantity)
        productRepository.save(updatedProduct)

        val event = StockUpdatedEvent(
            productId = productId,
            previousStock = product.stock,
            newStock = updatedProduct.stock,
            quantity = quantity
        )
        kafkaTemplate.send("stock-updated", productId.toString(), event)
    }

    @KafkaListener(topics = ["order-created"], groupId = "product-service")
    fun handleOrderCreated(event: OrderCreatedEvent) {
        // ลด stock เมื่อมี order ใหม่
        event.items.forEach { item ->
            updateStock(item.productId, item.quantity)
        }
    }
}
```

## 5. Order Service

### Domain Model

```kotlin
// Order.kt
package com.ecommerce.order.domain

import jakarta.persistence.*
import java.math.BigDecimal
import java.time.LocalDateTime

@Entity
@Table(name = "orders")
data class Order(
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long = 0,
    
    @Column(nullable = false)
    val userId: Long,
    
    @Enumerated(EnumType.STRING)
    val status: OrderStatus = OrderStatus.PENDING,
    
    @Column(nullable = false, precision = 10, scale = 2)
    val totalAmount: BigDecimal,
    
    val shippingAddress: String,
    
    @OneToMany(mappedBy = "order", cascade = [CascadeType.ALL], fetch = FetchType.EAGER)
    val items: List<OrderItem> = emptyList(),
    
    val createdAt: LocalDateTime = LocalDateTime.now(),
    val updatedAt: LocalDateTime = LocalDateTime.now()
)

@Entity
@Table(name = "order_items")
data class OrderItem(
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long = 0,
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "order_id")
    val order: Order? = null,
    
    val productId: Long,
    val productName: String,
    val quantity: Int,
    val unitPrice: BigDecimal,
    val totalPrice: BigDecimal = unitPrice * quantity.toBigDecimal()
)

enum class OrderStatus {
    PENDING, CONFIRMED, PROCESSING, SHIPPED, DELIVERED, CANCELLED
}
```

### Order Service

```kotlin
// OrderService.kt
package com.ecommerce.order.service

import com.ecommerce.order.client.ProductClient
import com.ecommerce.order.client.UserClient
import com.ecommerce.order.domain.*
import com.ecommerce.order.dto.*
import com.ecommerce.order.event.OrderCreatedEvent
import com.ecommerce.order.exception.OrderNotFoundException
import com.ecommerce.order.repository.OrderRepository
import org.springframework.kafka.core.KafkaTemplate
import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional

@Service
class OrderService(
    private val orderRepository: OrderRepository,
    private val productClient: ProductClient,
    private val userClient: UserClient,
    private val kafkaTemplate: KafkaTemplate<String, OrderCreatedEvent>
) {

    @Transactional
    fun createOrder(userId: Long, request: CreateOrderRequest): OrderResponse {
        // ตรวจสอบ user มีอยู่จริง
        val user = userClient.getUserById(userId)
        
        // ดึงข้อมูล products และคำนวณราคา
        val orderItems = request.items.map { item ->
            val product = productClient.getProductById(item.productId)
            
            OrderItem(
                productId = product.id,
                productName = product.name,
                quantity = item.quantity,
                unitPrice = product.price
            )
        }

        val totalAmount = orderItems.sumOf { it.totalPrice }

        val order = Order(
            userId = userId,
            totalAmount = totalAmount,
            shippingAddress = request.shippingAddress,
            items = orderItems
        )

        val savedOrder = orderRepository.save(order)

        // ส่ง event ไปยัง Kafka เพื่อให้ Product Service ลด stock
        val event = OrderCreatedEvent(
            orderId = savedOrder.id,
            userId = userId,
            items = orderItems.map { 
                OrderItemEvent(it.productId, it.quantity) 
            }
        )
        kafkaTemplate.send("order-created", savedOrder.id.toString(), event)

        return savedOrder.toResponse()
    }

    fun getOrderById(id: Long): OrderResponse {
        return orderRepository.findById(id)
            .orElseThrow { OrderNotFoundException("Order $id not found") }
            .toResponse()
    }

    fun getOrdersByUserId(userId: Long): List<OrderResponse> {
        return orderRepository.findByUserId(userId).map { it.toResponse() }
    }

    @Transactional
    fun updateOrderStatus(id: Long, status: OrderStatus): OrderResponse {
        val order = orderRepository.findById(id)
            .orElseThrow { OrderNotFoundException("Order $id not found") }

        val updatedOrder = order.copy(status = status)
        return orderRepository.save(updatedOrder).toResponse()
    }
}
```

### Feign Client สำหรับ Inter-Service Communication

```kotlin
// ProductClient.kt
package com.ecommerce.order.client

import com.ecommerce.order.dto.ProductDto
import org.springframework.cloud.openfeign.FeignClient
import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.PathVariable

@FeignClient(name = "product-service", fallback = ProductClientFallback::class)
interface ProductClient {
    
    @GetMapping("/api/products/{id}")
    fun getProductById(@PathVariable id: Long): ProductDto
}

@Component
class ProductClientFallback : ProductClient {
    override fun getProductById(id: Long): ProductDto {
        throw ProductServiceUnavailableException("Product service is unavailable")
    }
}
```

```kotlin
// UserClient.kt
package com.ecommerce.order.client

import com.ecommerce.order.dto.UserDto
import org.springframework.cloud.openfeign.FeignClient
import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.PathVariable

@FeignClient(name = "user-service", fallback = UserClientFallback::class)
interface UserClient {
    
    @GetMapping("/api/users/{id}")
    fun getUserById(@PathVariable id: Long): UserDto
}
```

## 6. Kafka Configuration

### Kafka Producer Configuration

```kotlin
// KafkaProducerConfig.kt
package com.ecommerce.user.config

import com.ecommerce.user.event.UserCreatedEvent
import org.apache.kafka.clients.producer.ProducerConfig
import org.apache.kafka.common.serialization.StringSerializer
import org.springframework.boot.autoconfigure.kafka.KafkaProperties
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.kafka.core.DefaultKafkaProducerFactory
import org.springframework.kafka.core.KafkaTemplate
import org.springframework.kafka.core.ProducerFactory
import org.springframework.kafka.support.serializer.JsonSerializer

@Configuration
class KafkaProducerConfig(private val kafkaProperties: KafkaProperties) {

    @Bean
    fun producerFactory(): ProducerFactory<String, UserCreatedEvent> {
        val configProps = mapOf(
            ProducerConfig.BOOTSTRAP_SERVERS_CONFIG to kafkaProperties.bootstrapServers.joinToString(","),
            ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG to StringSerializer::class.java,
            ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG to JsonSerializer::class.java,
            ProducerConfig.ACKS_CONFIG to "all",
            ProducerConfig.RETRIES_CONFIG to 3,
            ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG to true
        )
        return DefaultKafkaProducerFactory(configProps)
    }

    @Bean
    fun kafkaTemplate(): KafkaTemplate<String, UserCreatedEvent> {
        return KafkaTemplate(producerFactory())
    }
}
```

### Kafka Consumer Configuration

```kotlin
// KafkaConsumerConfig.kt
package com.ecommerce.order.config

import com.ecommerce.order.event.OrderCreatedEvent
import org.apache.kafka.clients.consumer.ConsumerConfig
import org.apache.kafka.common.serialization.StringDeserializer
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.kafka.annotation.EnableKafka
import org.springframework.kafka.config.ConcurrentKafkaListenerContainerFactory
import org.springframework.kafka.core.DefaultKafkaConsumerFactory
import org.springframework.kafka.support.serializer.JsonDeserializer

@Configuration
@EnableKafka
class KafkaConsumerConfig {

    @Bean
    fun consumerFactory(): DefaultKafkaConsumerFactory<String, OrderCreatedEvent> {
        val deserializer = JsonDeserializer(OrderCreatedEvent::class.java)
        deserializer.setRemoveTypeHeaders(false)
        deserializer.addTrustedPackages("*")
        deserializer.setUseTypeMapperForKey(true)

        val configProps = mapOf(
            ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG to "localhost:9092",
            ConsumerConfig.GROUP_ID_CONFIG to "order-service",
            ConsumerConfig.AUTO_OFFSET_RESET_CONFIG to "earliest",
            ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG to false,
            ConsumerConfig.MAX_POLL_RECORDS_CONFIG to 100
        )

        return DefaultKafkaConsumerFactory(configProps, StringDeserializer(), deserializer)
    }

    @Bean
    fun kafkaListenerContainerFactory(): ConcurrentKafkaListenerContainerFactory<String, OrderCreatedEvent> {
        val factory = ConcurrentKafkaListenerContainerFactory<String, OrderCreatedEvent>()
        factory.consumerFactory = consumerFactory()
        factory.containerProperties.ackMode = ContainerProperties.AckMode.MANUAL_IMMEDIATE
        factory.setConcurrency(3)
        return factory
    }
}
```

## 7. Docker Compose - ครบทุก Service

```yaml
# docker-compose.yml
version: '3.8'

services:
  # Infrastructure Services
  zookeeper:
    image: confluentinc/cp-zookeeper:7.4.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000
    ports:
      - "2181:2181"

  kafka:
    image: confluentinc/cp-kafka:7.4.0
    depends_on:
      - zookeeper
    ports:
      - "9092:9092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:29092,PLAINTEXT_HOST://localhost:9092
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_AUTO_CREATE_TOPICS_ENABLE: "true"

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    command: redis-server --appendonly yes
    volumes:
      - redis-data:/data

  # Databases
  user-db:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: userdb
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: password
    ports:
      - "5432:5432"
    volumes:
      - user-db-data:/var/lib/postgresql/data

  product-db:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: productdb
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: password
    ports:
      - "5433:5432"
    volumes:
      - product-db-data:/var/lib/postgresql/data

  order-db:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: orderdb
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: password
    ports:
      - "5434:5432"
    volumes:
      - order-db-data:/var/lib/postgresql/data

  # Application Services
  eureka-server:
    build: ./eureka-server
    ports:
      - "8761:8761"
    environment:
      SPRING_PROFILES_ACTIVE: docker
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8761/actuator/health"]
      interval: 30s
      timeout: 10s
      retries: 5

  api-gateway:
    build: ./api-gateway
    ports:
      - "8080:8080"
    depends_on:
      eureka-server:
        condition: service_healthy
    environment:
      SPRING_PROFILES_ACTIVE: docker
      EUREKA_CLIENT_SERVICEURL_DEFAULTZONE: http://eureka:eureka-password@eureka-server:8761/eureka/
      JWT_SECRET: your-super-secret-key-here

  user-service:
    build: ./user-service
    ports:
      - "8081:8081"
    depends_on:
      - user-db
      - kafka
      - eureka-server
    environment:
      SPRING_PROFILES_ACTIVE: docker
      SPRING_DATASOURCE_URL: jdbc:postgresql://user-db:5432/userdb
      SPRING_KAFKA_BOOTSTRAP_SERVERS: kafka:29092
      EUREKA_CLIENT_SERVICEURL_DEFAULTZONE: http://eureka:eureka-password@eureka-server:8761/eureka/

  product-service:
    build: ./product-service
    ports:
      - "8082:8082"
    depends_on:
      - product-db
      - kafka
      - eureka-server
    environment:
      SPRING_PROFILES_ACTIVE: docker
      SPRING_DATASOURCE_URL: jdbc:postgresql://product-db:5432/productdb
      SPRING_KAFKA_BOOTSTRAP_SERVERS: kafka:29092
      EUREKA_CLIENT_SERVICEURL_DEFAULTZONE: http://eureka:eureka-password@eureka-server:8761/eureka/

  order-service:
    build: ./order-service
    ports:
      - "8083:8083"
    depends_on:
      - order-db
      - kafka
      - eureka-server
    environment:
      SPRING_PROFILES_ACTIVE: docker
      SPRING_DATASOURCE_URL: jdbc:postgresql://order-db:5432/orderdb
      SPRING_KAFKA_BOOTSTRAP_SERVERS: kafka:29092
      EUREKA_CLIENT_SERVICEURL_DEFAULTZONE: http://eureka:eureka-password@eureka-server:8761/eureka/

volumes:
  redis-data:
  user-db-data:
  product-db-data:
  order-db-data:
```

## 8. Health Check และ Monitoring

```kotlin
// HealthCheckConfig.kt
package com.ecommerce.user.config

import org.springframework.boot.actuate.health.Health
import org.springframework.boot.actuate.health.HealthIndicator
import org.springframework.kafka.core.KafkaTemplate
import org.springframework.stereotype.Component

@Component
class KafkaHealthIndicator(
    private val kafkaTemplate: KafkaTemplate<String, Any>
) : HealthIndicator {

    override fun health(): Health {
        return try {
            kafkaTemplate.defaultTopic
            Health.up()
                .withDetail("kafka", "Available")
                .build()
        } catch (e: Exception) {
            Health.down()
                .withDetail("kafka", "Unavailable: ${e.message}")
                .build()
        }
    }
}
```

```yaml
# application.yml - Actuator configuration
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  endpoint:
    health:
      show-details: always
  metrics:
    export:
      prometheus:
        enabled: true
```

## 9. Testing

### Integration Tests

```kotlin
// OrderServiceIntegrationTest.kt
package com.ecommerce.order.service

import com.ecommerce.order.dto.CreateOrderRequest
import com.ecommerce.order.dto.OrderItemRequest
import org.junit.jupiter.api.Test
import org.springframework.beans.factory.annotation.Autowired
import org.springframework.boot.test.context.SpringBootTest
import org.springframework.cloud.contract.wiremock.AutoConfigureWireMock
import org.springframework.kafka.test.context.EmbeddedKafka
import org.springframework.test.context.ActiveProfiles
import com.github.tomakehurst.wiremock.client.WireMock.*
import org.assertj.core.api.Assertions.assertThat
import java.math.BigDecimal

@SpringBootTest
@ActiveProfiles("test")
@EmbeddedKafka(partitions = 1, topics = ["order-created", "stock-updated"])
@AutoConfigureWireMock(port = 0)
class OrderServiceIntegrationTest {

    @Autowired
    private lateinit var orderService: OrderService

    @Test
    fun `should create order successfully`() {
        // Setup WireMock stubs
        stubFor(get(urlEqualTo("/api/users/1"))
            .willReturn(aResponse()
                .withHeader("Content-Type", "application/json")
                .withBody("""{"id":1,"email":"test@test.com","firstName":"John","lastName":"Doe"}""")))

        stubFor(get(urlEqualTo("/api/products/1"))
            .willReturn(aResponse()
                .withHeader("Content-Type", "application/json")
                .withBody("""{"id":1,"name":"Test Product","price":100.00,"stock":10}""")))

        val request = CreateOrderRequest(
            items = listOf(OrderItemRequest(productId = 1L, quantity = 2)),
            shippingAddress = "123 Test Street"
        )

        val order = orderService.createOrder(userId = 1L, request = request)

        assertThat(order).isNotNull
        assertThat(order.userId).isEqualTo(1L)
        assertThat(order.totalAmount).isEqualByComparingTo(BigDecimal("200.00"))
    }
}
```

## สรุปสถาปัตยกรรม

| Component | Technology | Port | Description |
|-----------|-----------|------|-------------|
| Eureka Server | Spring Cloud Netflix | 8761 | Service Discovery |
| API Gateway | Spring Cloud Gateway | 8080 | Request routing, JWT auth |
| User Service | Spring Boot + JPA | 8081 | User management |
| Product Service | Spring Boot + JPA | 8082 | Product catalog |
| Order Service | Spring Boot + JPA | 8083 | Order processing |
| Kafka | Apache Kafka | 9092 | Async messaging |
| Redis | Redis 7 | 6379 | Caching, rate limiting |
| User DB | PostgreSQL 15 | 5432 | User data |
| Product DB | PostgreSQL 15 | 5433 | Product data |
| Order DB | PostgreSQL 15 | 5434 | Order data |

## การ Flow ของ Order Creation

```
Client → API Gateway (JWT Auth) → Order Service
                                      ↓
                              Feign Call → User Service
                                      ↓
                              Feign Call → Product Service
                                      ↓
                              Save Order to DB
                                      ↓
                              Kafka: "order-created" event
                                      ↓
                              Product Service ← Kafka Consumer
                              (Update Stock)
```

## Best Practices ที่ใช้ในโปรเจกต์นี้

1. **Circuit Breaker Pattern** - ป้องกันการล้มเหลวแบบ cascade
2. **Event-Driven Architecture** - ใช้ Kafka สำหรับ async communication
3. **Service Discovery** - Eureka ช่วย scale services ได้ง่าย
4. **API Gateway Pattern** - ควบคุม traffic และ security จากจุดเดียว
5. **Database per Service** - แต่ละ service มี database เป็นของตัวเอง
6. **Health Checks** - Monitor สุขภาพของทุก service
7. **Containerization** - Docker ทำให้ deploy และ scale ง่าย

*Part 55/100+ | Kotlin & Spring Boot Complete Course*
