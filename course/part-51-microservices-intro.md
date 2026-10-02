# Part 51: แนะนำ Microservices
## Microservices Architecture กับ Spring Boot + Kotlin

---

## 🎯 เป้าหมายของ Part นี้

- เข้าใจ Microservices vs Monolith
- รู้จัก patterns หลักของ Microservices
- Spring Cloud ecosystem
- Service decomposition strategies
- ตัวอย่าง: แปลง Monolith เป็น Microservices

---

## 🏗️ 1. Monolith vs Microservices

```
Monolith:
├── UI Layer
├── Business Logic (ทั้งหมดอยู่ด้วยกัน)
├── Data Access Layer
└── Database (single)

ข้อดี:
✅ Simple development และ testing
✅ Easy deployment (1 JAR)
✅ No network latency between components
✅ ACID transactions ง่าย

ข้อเสีย:
❌ Scale ทั้ง app ต้องทำพร้อมกัน
❌ Technology lock-in
❌ ทีมใหญ่ conflict กันเยอะ
❌ ถ้า crash ทั้งระบบล่ม

Microservices:
User Service  →  Product Service  →  Order Service
    ↓                  ↓                   ↓
 User DB          Product DB           Order DB

ข้อดี:
✅ Scale แต่ละ service แยกกัน
✅ เลือก tech stack ได้ตาม service
✅ ทีมเล็กๆ ทำงานแยกกันได้
✅ Fault isolation

ข้อเสีย:
❌ Distributed systems complexity
❌ Network latency
❌ Data consistency ยากขึ้น
❌ Operational overhead
```

---

## 🌐 2. Spring Cloud Ecosystem

```
Spring Cloud Components:

Service Discovery:
  - Spring Cloud Netflix Eureka
  - Consul

API Gateway:
  - Spring Cloud Gateway
  - Nginx

Load Balancing:
  - Spring Cloud LoadBalancer (แทน Ribbon)

Circuit Breaker:
  - Resilience4j

Config Server:
  - Spring Cloud Config

Distributed Tracing:
  - Spring Cloud Sleuth + Zipkin
  - Micrometer Tracing
```

---

## 📦 3. build.gradle.kts สำหรับ Microservices

```kotlin
// shared dependency version
extra["springCloudVersion"] = "2023.0.0"

dependencyManagement {
    imports {
        mavenBom("org.springframework.cloud:spring-cloud-dependencies:${property("springCloudVersion")}")
    }
}

dependencies {
    // Eureka Client
    implementation("org.springframework.cloud:spring-cloud-starter-netflix-eureka-client")
    
    // Config Client
    implementation("org.springframework.cloud:spring-cloud-starter-config")
    
    // LoadBalancer
    implementation("org.springframework.cloud:spring-cloud-starter-loadbalancer")
    
    // Circuit Breaker
    implementation("io.github.resilience4j:resilience4j-spring-boot3:2.1.0")
    
    // OpenFeign (HTTP client)
    implementation("org.springframework.cloud:spring-cloud-starter-openfeign")
}
```

---

## 🔍 4. Service Discovery (Eureka)

### Eureka Server

```kotlin
// eureka-server/build.gradle.kts
dependencies {
    implementation("org.springframework.cloud:spring-cloud-starter-netflix-eureka-server")
}

// EurekaServerApplication.kt
@SpringBootApplication
@EnableEurekaServer  // เปิด Eureka Server
class EurekaServerApplication

fun main(args: Array<String>) {
    runApplication<EurekaServerApplication>(*args)
}
```

```yaml
# eureka-server/application.yml
server:
  port: 8761

eureka:
  instance:
    hostname: localhost
  client:
    register-with-eureka: false  # ไม่ลง register ตัวเอง
    fetch-registry: false
  server:
    wait-time-in-ms-when-sync-empty: 0
```

### Eureka Client

```kotlin
// user-service/UserServiceApplication.kt
@SpringBootApplication
@EnableDiscoveryClient  // ลง register กับ Eureka
class UserServiceApplication

fun main(args: Array<String>) {
    runApplication<UserServiceApplication>(*args)
}
```

```yaml
# user-service/application.yml
spring:
  application:
    name: user-service  # ชื่อใน Eureka registry

eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
  instance:
    prefer-ip-address: true
```

---

## 🌉 5. API Gateway

```kotlin
// gateway/GatewayApplication.kt
@SpringBootApplication
class GatewayApplication

fun main(args: Array<String>) {
    runApplication<GatewayApplication>(*args)
}
```

```yaml
# gateway/application.yml
spring:
  application:
    name: api-gateway
  cloud:
    gateway:
      routes:
        - id: user-service
          uri: lb://user-service  # lb:// = load balanced ผ่าน Eureka
          predicates:
            - Path=/api/users/**
          filters:
            - StripPrefix=0
        
        - id: product-service
          uri: lb://product-service
          predicates:
            - Path=/api/products/**
        
        - id: order-service
          uri: lb://order-service
          predicates:
            - Path=/api/orders/**
      
      # Global CORS
      globalcors:
        cors-configurations:
          '[/**]':
            allowedOrigins: "*"
            allowedMethods: "*"
            allowedHeaders: "*"

server:
  port: 8080
```

---

## 📞 6. Service-to-Service Communication (OpenFeign)

```kotlin
// order-service: เรียก user-service ผ่าน Feign

@FeignClient(name = "user-service")  // ชื่อ service ใน Eureka
interface UserServiceClient {
    
    @GetMapping("/api/users/{id}")
    fun getUserById(@PathVariable id: Long): UserResponse?
    
    @GetMapping("/api/users")
    fun getAllUsers(): List<UserResponse>
}

// ใช้งาน
@Service
class OrderService(
    private val orderRepo: OrderRepository,
    private val userClient: UserServiceClient,  // inject ได้เลย
    private val productClient: ProductServiceClient
) {
    
    fun createOrder(userId: Long, items: List<OrderItemRequest>): OrderResponse {
        // ตรวจสอบ user
        val user = userClient.getUserById(userId)
            ?: throw NotFoundException("User not found", "User", userId)
        
        // ตรวจสอบ products
        val products = items.map { item ->
            productClient.getProductById(item.productId)
                ?: throw NotFoundException("Product not found", "Product", item.productId)
        }
        
        // สร้าง order
        val order = Order(
            userId = userId,
            status = OrderStatus.PENDING
        )
        
        return orderRepo.save(order).toResponse()
    }
}
```

---

## 🗂️ 7. Project Structure ของ Microservices

```
microservices-project/
├── eureka-server/                 # Service Discovery
│   ├── src/
│   └── build.gradle.kts
├── api-gateway/                   # Entry point
│   ├── src/
│   └── build.gradle.kts
├── user-service/                  # User management
│   ├── src/
│   └── build.gradle.kts
├── product-service/               # Product catalog
│   ├── src/
│   └── build.gradle.kts
├── order-service/                 # Order processing
│   ├── src/
│   └── build.gradle.kts
├── shared-lib/                    # Shared DTOs, Utils
│   ├── src/
│   └── build.gradle.kts
├── docker-compose.yml             # Start all services
└── settings.gradle.kts            # Multi-project build
```

```kotlin
// settings.gradle.kts
rootProject.name = "microservices-project"
include("eureka-server", "api-gateway", "user-service", "product-service", "order-service", "shared-lib")
```

```yaml
# docker-compose.yml
version: '3.8'
services:
  eureka:
    build: ./eureka-server
    ports: ["8761:8761"]
  
  gateway:
    build: ./api-gateway
    ports: ["8080:8080"]
    depends_on: [eureka]
    environment:
      EUREKA_CLIENT_SERVICE_URL_DEFAULTZONE: http://eureka:8761/eureka/
  
  user-service:
    build: ./user-service
    environment:
      EUREKA_CLIENT_SERVICE_URL_DEFAULTZONE: http://eureka:8761/eureka/
      SPRING_DATASOURCE_URL: jdbc:postgresql://user-db:5432/userdb
    depends_on: [eureka, user-db]
  
  user-db:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: userdb
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
```

---

## 📝 สรุป Part 51

| แนวคิด | รายละเอียด |
|--------|-----------|
| Microservices | แยก service ตาม business domain |
| Eureka | Service discovery registry |
| API Gateway | Single entry point |
| OpenFeign | Declarative HTTP client |
| Spring Cloud | Ecosystem สำหรับ distributed |
| Docker Compose | รัน services ทั้งหมดพร้อมกัน |

---

## ➡️ ถัดไป: Part 52 - Inter-Service Communication Patterns

---
*Part 51/100+ | Kotlin & Spring Boot Complete Course*
