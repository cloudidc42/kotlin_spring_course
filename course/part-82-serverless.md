# Part 82: Serverless กับ Spring Boot
## Spring Cloud Function และ AWS Lambda

---

## 🎯 เป้าหมายของ Part นี้

- เข้าใจ Serverless architecture
- ใช้ Spring Cloud Function
- Deploy Spring Boot บน AWS Lambda
- แก้ปัญหา Cold Start
- Function Composition

---

## 📖 1. Serverless คืออะไร?

Serverless ไม่ได้แปลว่า "ไม่มี server" แต่หมายความว่า **เราไม่ต้องดูแล server เอง** โดยผู้ให้บริการ cloud จัดการ infrastructure ทั้งหมดให้

### Serverless vs Traditional

```
Traditional:
  [Server] → ทำงานตลอดเวลา → จ่ายเงินตลอด 24/7
  CPU 5% แต่ต้องจ่าย 100%

Serverless:
  [Function] → ทำงานเฉพาะเมื่อมี request → จ่ายเฉพาะเวลาใช้งาน
  จ่ายตาม invocations และ duration
```

### AWS Lambda Pricing (ตัวอย่าง)

```
Free Tier: 1 million requests/เดือน
After free tier:
  $0.20 per 1 million requests
  $0.0000166667 per GB-second
  
ตัวอย่าง: 10M requests, avg 500ms, 512MB RAM
= $2.00 (requests) + $41.67 (compute) = ~$43.67/เดือน
```

---

## 📦 2. Spring Cloud Function

Spring Cloud Function เป็น library ที่ทำให้เราเขียน business logic เป็น **pure functions** แล้ว deploy ได้บน platform ต่างๆ

### build.gradle.kts

```kotlin
plugins {
    kotlin("jvm") version "1.9.21"
    kotlin("plugin.spring") version "1.9.21"
    id("org.springframework.boot") version "3.2.1"
    id("io.spring.dependency-management") version "1.1.4"
}

dependencies {
    implementation("org.springframework.boot:spring-boot-starter")
    implementation("org.springframework.cloud:spring-cloud-function-kotlin")
    implementation("org.springframework.cloud:spring-cloud-function-web")

    // สำหรับ AWS Lambda
    implementation("org.springframework.cloud:spring-cloud-function-adapter-aws")
    implementation("com.amazonaws:aws-lambda-java-core:1.2.3")
    implementation("com.amazonaws:aws-lambda-java-events:3.11.3")

    implementation("com.fasterxml.jackson.module:jackson-module-kotlin")
    implementation("org.jetbrains.kotlin:kotlin-reflect")
}

dependencyManagement {
    imports {
        mavenBom("org.springframework.cloud:spring-cloud-dependencies:2023.0.0")
    }
}
```

---

## 🔧 3. เขียน Spring Cloud Functions

### Data Classes

```kotlin
// src/main/kotlin/com/example/model/Models.kt
package com.example.model

data class OrderRequest(
    val productId: String,
    val quantity: Int,
    val userId: String
)

data class OrderResponse(
    val orderId: String,
    val status: String,
    val totalPrice: Double,
    val estimatedDelivery: String
)

data class EmailRequest(
    val to: String,
    val subject: String,
    val body: String
)

data class ProcessingResult(
    val success: Boolean,
    val message: String,
    val processedAt: Long = System.currentTimeMillis()
)
```

### Functions เป็น Beans

```kotlin
// src/main/kotlin/com/example/functions/OrderFunctions.kt
package com.example.functions

import com.example.model.*
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import java.util.UUID
import java.util.function.Function
import java.util.function.Supplier
import java.util.function.Consumer

@Configuration
class OrderFunctions {

    // Function: input → output (synchronous transform)
    @Bean
    fun processOrder(): Function<OrderRequest, OrderResponse> = Function { request ->
        println("Processing order for user: ${request.userId}")

        // Business logic
        val price = calculatePrice(request.productId, request.quantity)

        OrderResponse(
            orderId = UUID.randomUUID().toString(),
            status = "CONFIRMED",
            totalPrice = price,
            estimatedDelivery = "3-5 business days"
        )
    }

    // Consumer: input only (side effects)
    @Bean
    fun sendEmail(): Consumer<EmailRequest> = Consumer { request ->
        println("Sending email to: ${request.to}")
        println("Subject: ${request.subject}")
        // ส่ง email จริงๆ ที่นี่
    }

    // Supplier: no input, produces output
    @Bean
    fun healthCheck(): Supplier<Map<String, String>> = Supplier {
        mapOf(
            "status" to "UP",
            "version" to "1.0.0",
            "timestamp" to System.currentTimeMillis().toString()
        )
    }

    private fun calculatePrice(productId: String, quantity: Int): Double {
        // Mock pricing logic
        val unitPrice = mapOf(
            "PROD001" to 99.99,
            "PROD002" to 199.99,
            "PROD003" to 49.99
        )
        return (unitPrice[productId] ?: 0.0) * quantity
    }
}
```

### Kotlin-specific Functions (ใช้ suspend)

```kotlin
// src/main/kotlin/com/example/functions/AsyncFunctions.kt
package com.example.functions

import com.example.model.OrderRequest
import com.example.model.OrderResponse
import kotlinx.coroutines.reactor.mono
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import reactor.core.publisher.Flux
import reactor.core.publisher.Mono
import java.util.UUID
import java.util.function.Function

@Configuration
class AsyncFunctions {

    // Reactive Function (Mono → Mono)
    @Bean
    fun processOrderAsync(): Function<Mono<OrderRequest>, Mono<OrderResponse>> =
        Function { requestMono ->
            requestMono.map { request ->
                OrderResponse(
                    orderId = UUID.randomUUID().toString(),
                    status = "PROCESSING",
                    totalPrice = 100.0,
                    estimatedDelivery = "2-3 days"
                )
            }
        }

    // Stream Function (Flux → Flux)
    @Bean
    fun processOrderBatch(): Function<Flux<OrderRequest>, Flux<OrderResponse>> =
        Function { requestFlux ->
            requestFlux.map { request ->
                OrderResponse(
                    orderId = UUID.randomUUID().toString(),
                    status = "BATCH_PROCESSED",
                    totalPrice = 50.0,
                    estimatedDelivery = "5-7 days"
                )
            }
        }
}
```

---

## 🔗 4. Function Composition

Spring Cloud Function รองรับการ compose functions เข้าด้วยกัน

```kotlin
// src/main/kotlin/com/example/functions/ComposedFunctions.kt
package com.example.functions

import com.example.model.*
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import java.util.function.Function

@Configuration
class ComposedFunctions {

    @Bean
    fun validateOrder(): Function<OrderRequest, OrderRequest> = Function { request ->
        require(request.quantity > 0) { "Quantity must be positive" }
        require(request.userId.isNotBlank()) { "User ID required" }
        request
    }

    @Bean
    fun enrichOrder(): Function<OrderRequest, OrderRequest> = Function { request ->
        // เพิ่ม metadata
        println("Enriching order with additional data")
        request
    }

    @Bean
    fun createOrder(): Function<OrderRequest, OrderResponse> = Function { request ->
        OrderResponse(
            orderId = "ORD-${System.currentTimeMillis()}",
            status = "CREATED",
            totalPrice = request.quantity * 99.99,
            estimatedDelivery = "3-5 days"
        )
    }
}
```

### เรียกใช้ Composed Function

```bash
# Compose functions ด้วย pipe operator ใน URL
curl -X POST http://localhost:8080/validateOrder,enrichOrder,createOrder \
  -H "Content-Type: application/json" \
  -d '{"productId":"PROD001","quantity":2,"userId":"user123"}'
```

---

## ☁️ 5. Deploy บน AWS Lambda

### Lambda Handler

```kotlin
// src/main/kotlin/com/example/lambda/LambdaHandler.kt
package com.example.lambda

import org.springframework.cloud.function.adapter.aws.FunctionInvoker

// Handler class สำหรับ AWS Lambda
class LambdaHandler : FunctionInvoker()
```

### application.properties สำหรับ Lambda

```properties
# src/main/resources/application.properties

# ระบุ function ที่จะใช้ใน Lambda
spring.cloud.function.definition=processOrder

# Optimize สำหรับ Lambda
spring.main.web-application-type=none
spring.main.lazy-initialization=true

# Logging
logging.level.org.springframework=WARN
logging.level.com.example=INFO
```

### Build สำหรับ AWS Lambda

```kotlin
// build.gradle.kts - เพิ่ม task สำหรับ Lambda package
tasks.register<Zip>("buildLambdaZip") {
    from(tasks.compileKotlin)
    from(tasks.processResources)
    into("lib") {
        from(configurations.runtimeClasspath)
    }
    archiveFileName.set("lambda-function.zip")
    destinationDirectory.set(file("$buildDir/distributions"))
}
```

```bash
# Build ZIP สำหรับ upload ไป Lambda
./gradlew buildLambdaZip

# Upload ด้วย AWS CLI
aws lambda create-function \
  --function-name spring-order-processor \
  --zip-file fileb://build/distributions/lambda-function.zip \
  --handler com.example.lambda.LambdaHandler::handleRequest \
  --runtime java21 \
  --memory-size 512 \
  --timeout 30 \
  --role arn:aws:iam::YOUR-ACCOUNT:role/lambda-role

# Test Lambda
aws lambda invoke \
  --function-name spring-order-processor \
  --payload '{"productId":"PROD001","quantity":2,"userId":"user123"}' \
  output.json
```

---

## ❄️ 6. Cold Start Optimization

Cold Start คือการที่ Lambda ต้องเริ่มต้น JVM ใหม่ เมื่อไม่มี warm instance พร้อม

```
Cold Start Timeline:
  JVM Init (1-2s) → Spring Context (2-3s) → Function Init (100ms) → Execute
  Total: 3-5 seconds! ← แย่มากสำหรับ production
```

### วิธีแก้ Cold Start

#### 1. Lazy Initialization

```properties
# application.properties
spring.main.lazy-initialization=true
```

#### 2. Reduce Dependencies

```kotlin
// ลบ dependencies ที่ไม่จำเป็นออก
// ตัวอย่าง: ถ้าไม่ใช้ database ก็ไม่ต้องใส่ JPA
dependencies {
    // เฉพาะที่ต้องการจริงๆ
    implementation("org.springframework.cloud:spring-cloud-function-kotlin")
    implementation("org.springframework.cloud:spring-cloud-function-adapter-aws")
    // ไม่ใส่ spring-boot-starter-web ถ้าไม่ต้องการ HTTP server
}
```

#### 3. ใช้ Native Image (เร็วที่สุด)

```kotlin
// build.gradle.kts
plugins {
    id("org.graalvm.buildtools.native") version "0.9.28"
}

graalvmNative {
    lambda {
        // Build native image สำหรับ Lambda
        enabled.set(true)
    }
}
```

#### 4. Provisioned Concurrency (AWS ช่วย)

```bash
# ตั้ง Provisioned Concurrency เพื่อให้ Lambda อุ่นตลอดเวลา
aws lambda put-provisioned-concurrency-config \
  --function-name spring-order-processor \
  --qualifier PROD \
  --provisioned-concurrent-executions 5
```

### Cold Start Benchmark

```
Configuration                    | Cold Start Time
---------------------------------|----------------
Standard Spring Boot (JVM)       | 8-12 seconds
Lazy Init + Minimal Dependencies | 3-5 seconds
Spring Cloud Function            | 2-4 seconds
Native Image                     | 0.1-0.5 seconds
```

---

## 🌐 7. API Gateway Integration

```yaml
# serverless.yml (ใช้ Serverless Framework)
service: spring-serverless-api

provider:
  name: aws
  runtime: java21
  region: ap-southeast-1
  memorySize: 512
  timeout: 30

functions:
  processOrder:
    handler: com.example.lambda.LambdaHandler::handleRequest
    events:
      - http:
          path: /orders
          method: post
          cors: true
    environment:
      SPRING_PROFILES_ACTIVE: lambda

  healthCheck:
    handler: com.example.lambda.LambdaHandler::handleRequest
    events:
      - http:
          path: /health
          method: get

plugins:
  - serverless-plugin-warmup

custom:
  warmup:
    default:
      enabled: production
      events:
        - schedule: rate(5 minutes)
```

---

## 🧪 8. Testing Serverless Functions

```kotlin
// src/test/kotlin/com/example/FunctionTest.kt
package com.example

import com.example.functions.OrderFunctions
import com.example.model.OrderRequest
import org.junit.jupiter.api.Test
import org.junit.jupiter.api.Assertions.*

class FunctionTest {

    private val orderFunctions = OrderFunctions()

    @Test
    fun `processOrder should return valid response`() {
        val function = orderFunctions.processOrder()
        val request = OrderRequest(
            productId = "PROD001",
            quantity = 2,
            userId = "user123"
        )

        val response = function.apply(request)

        assertNotNull(response.orderId)
        assertEquals("CONFIRMED", response.status)
        assertEquals(199.98, response.totalPrice, 0.001)
    }

    @Test
    fun `processOrder should handle unknown product`() {
        val function = orderFunctions.processOrder()
        val request = OrderRequest(
            productId = "UNKNOWN",
            quantity = 1,
            userId = "user123"
        )

        val response = function.apply(request)
        assertEquals(0.0, response.totalPrice)
    }
}
```

```kotlin
// Integration test กับ Spring context
@SpringBootTest
class FunctionIntegrationTest {

    @Autowired
    private lateinit var catalog: FunctionCatalog

    @Test
    fun `should find processOrder function`() {
        val function = catalog.lookup<Function<OrderRequest, OrderResponse>>("processOrder")
        assertNotNull(function)
    }
}
```

---

## 📋 สรุป

| หัวข้อ | รายละเอียด |
|--------|-----------|
| Spring Cloud Function | เขียน logic เป็น Function/Consumer/Supplier |
| Function Composition | Pipe functions ด้วย comma separator |
| AWS Lambda | Deploy ด้วย LambdaHandler |
| Cold Start | ปัญหาหลักของ Serverless บน JVM |
| แก้ Cold Start | Lazy init, Native Image, Provisioned Concurrency |
| ค่าใช้จ่าย | จ่ายเฉพาะตอนใช้งาน ประหยัดกว่า traditional |

---

*Part 82/100+ | Kotlin & Spring Boot Complete Course*
