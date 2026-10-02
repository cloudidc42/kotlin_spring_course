# Part 81: GraalVM Native Image
## สร้าง Native Binary ด้วย Spring Boot และ GraalVM

---

## 🎯 เป้าหมายของ Part นี้

- เข้าใจว่า GraalVM Native Image คืออะไร
- ตั้งค่า Spring Boot กับ GraalVM
- ทำ AOT (Ahead-of-Time) Processing
- เปรียบเทียบ startup time ระหว่าง JVM กับ Native
- Deploy Native Image ใน container

---

## 📖 1. GraalVM Native Image คืออะไร?

GraalVM Native Image เป็นเทคโนโลยีที่แปลง Java/Kotlin bytecode เป็น **native binary** โดยตรง แทนที่จะรันบน JVM

### ความแตกต่างระหว่าง JVM กับ Native Image

```
JVM Application:
  Source Code → Bytecode (.class) → JVM Runtime → Execute
  Startup: 2-10 วินาที | Memory: 200-500 MB

Native Image:
  Source Code → Bytecode → Native Binary → Execute
  Startup: 0.05-0.5 วินาที | Memory: 20-50 MB
```

### ข้อดีของ Native Image

| ด้าน | JVM | Native Image |
|------|-----|--------------|
| Startup Time | 2-10s | 50-500ms |
| Memory Usage | สูง | ต่ำมาก |
| Peak Performance | ดีมาก (JIT) | ดี (AOT) |
| Build Time | เร็ว | ช้า (5-15 นาที) |
| Docker Image Size | 200MB+ | 20-50MB |

---

## 📦 2. การติดตั้ง GraalVM

### ติดตั้งด้วย SDKMAN

```bash
# ติดตั้ง SDKMAN
curl -s "https://get.sdkman.io" | bash
source "$HOME/.sdkman/bin/sdkman-init.sh"

# ติดตั้ง GraalVM
sdk install java 21.0.1-graal

# ตรวจสอบ
java -version
# openjdk version "21.0.1" 2023-10-17
# OpenJDK Runtime Environment GraalVM CE 21.0.1+12.1 (build 21.0.1+12-jvmci-23.1-b19)

# ติดตั้ง native-image tool
gu install native-image
native-image --version
```

### ติดตั้งบน macOS ด้วย Homebrew

```bash
brew install --cask graalvm/tap/graalvm-jdk21
export JAVA_HOME=/Library/Java/JavaVirtualMachines/graalvm-21.jdk/Contents/Home
export PATH=$JAVA_HOME/bin:$PATH
gu install native-image
```

---

## 🏗️ 3. สร้าง Spring Boot Native Project

### build.gradle.kts

```kotlin
plugins {
    kotlin("jvm") version "1.9.21"
    kotlin("plugin.spring") version "1.9.21"
    id("org.springframework.boot") version "3.2.1"
    id("io.spring.dependency-management") version "1.1.4"
    // Plugin สำคัญสำหรับ Native Image
    id("org.graalvm.buildtools.native") version "0.9.28"
}

group = "com.example"
version = "0.0.1-SNAPSHOT"

java {
    sourceCompatibility = JavaVersion.VERSION_21
}

repositories {
    mavenCentral()
}

dependencies {
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-actuator")
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
    implementation("com.fasterxml.jackson.module:jackson-module-kotlin")
    implementation("org.jetbrains.kotlin:kotlin-reflect")
    runtimeOnly("com.h2database:h2")
    testImplementation("org.springframework.boot:spring-boot-starter-test")
}

// Native Image configuration
graalvmNative {
    binaries {
        named("main") {
            imageName.set("my-native-app")
            mainClass.set("com.example.NativeAppKt")
            buildArgs.add("--verbose")
            buildArgs.add("-O2") // Optimization level
        }
    }
    toolchainDetection.set(false)
}

kotlin {
    compilerOptions {
        freeCompilerArgs.addAll("-Xjsr305=strict")
    }
}
```

---

## 🔧 4. AOT (Ahead-of-Time) Processing

Spring Boot 3.x มี AOT Engine ในตัวที่ช่วยให้ Native Image ทำงานได้ถูกต้อง

### Application Entry Point

```kotlin
// src/main/kotlin/com/example/NativeApp.kt
package com.example

import org.springframework.boot.autoconfigure.SpringBootApplication
import org.springframework.boot.runApplication

@SpringBootApplication
class NativeApp

fun main(args: Array<String>) {
    runApplication<NativeApp>(*args)
}
```

### Entity

```kotlin
// src/main/kotlin/com/example/entity/Product.kt
package com.example.entity

import jakarta.persistence.*

@Entity
@Table(name = "products")
data class Product(
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long = 0,

    @Column(nullable = false)
    val name: String,

    @Column(nullable = false)
    val price: Double,

    @Column
    val description: String? = null
)
```

### Repository

```kotlin
// src/main/kotlin/com/example/repository/ProductRepository.kt
package com.example.repository

import com.example.entity.Product
import org.springframework.data.jpa.repository.JpaRepository
import org.springframework.data.jpa.repository.Query
import org.springframework.stereotype.Repository

@Repository
interface ProductRepository : JpaRepository<Product, Long> {
    fun findByNameContaining(name: String): List<Product>

    @Query("SELECT p FROM Product p WHERE p.price < :maxPrice")
    fun findCheaperThan(maxPrice: Double): List<Product>
}
```

### Service

```kotlin
// src/main/kotlin/com/example/service/ProductService.kt
package com.example.service

import com.example.entity.Product
import com.example.repository.ProductRepository
import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional

@Service
@Transactional
class ProductService(
    private val productRepository: ProductRepository
) {
    fun getAllProducts(): List<Product> =
        productRepository.findAll()

    fun getProductById(id: Long): Product =
        productRepository.findById(id)
            .orElseThrow { NoSuchElementException("Product $id not found") }

    fun createProduct(product: Product): Product =
        productRepository.save(product)

    fun updateProduct(id: Long, product: Product): Product {
        val existing = getProductById(id)
        return productRepository.save(
            existing.copy(
                name = product.name,
                price = product.price,
                description = product.description
            )
        )
    }

    fun deleteProduct(id: Long) {
        if (!productRepository.existsById(id)) {
            throw NoSuchElementException("Product $id not found")
        }
        productRepository.deleteById(id)
    }

    fun searchProducts(name: String): List<Product> =
        productRepository.findByNameContaining(name)
}
```

### Controller

```kotlin
// src/main/kotlin/com/example/controller/ProductController.kt
package com.example.controller

import com.example.entity.Product
import com.example.service.ProductService
import org.springframework.http.HttpStatus
import org.springframework.http.ResponseEntity
import org.springframework.web.bind.annotation.*

@RestController
@RequestMapping("/api/products")
class ProductController(
    private val productService: ProductService
) {
    @GetMapping
    fun getAllProducts(): ResponseEntity<List<Product>> =
        ResponseEntity.ok(productService.getAllProducts())

    @GetMapping("/{id}")
    fun getProduct(@PathVariable id: Long): ResponseEntity<Product> =
        ResponseEntity.ok(productService.getProductById(id))

    @PostMapping
    fun createProduct(@RequestBody product: Product): ResponseEntity<Product> =
        ResponseEntity.status(HttpStatus.CREATED)
            .body(productService.createProduct(product))

    @PutMapping("/{id}")
    fun updateProduct(
        @PathVariable id: Long,
        @RequestBody product: Product
    ): ResponseEntity<Product> =
        ResponseEntity.ok(productService.updateProduct(id, product))

    @DeleteMapping("/{id}")
    fun deleteProduct(@PathVariable id: Long): ResponseEntity<Unit> {
        productService.deleteProduct(id)
        return ResponseEntity.noContent().build()
    }

    @GetMapping("/search")
    fun searchProducts(@RequestParam name: String): ResponseEntity<List<Product>> =
        ResponseEntity.ok(productService.searchProducts(name))
}
```

---

## ⚙️ 5. Hints สำหรับ Reflection

Native Image ต้องการ hints สำหรับ reflection เพราะ GraalVM ไม่สามารถวิเคราะห์ reflection ได้ตอน runtime

```kotlin
// src/main/kotlin/com/example/config/NativeHints.kt
package com.example.config

import com.example.entity.Product
import org.springframework.aot.hint.RuntimeHints
import org.springframework.aot.hint.RuntimeHintsRegistrar
import org.springframework.context.annotation.ImportRuntimeHints

@ImportRuntimeHints(NativeHints.ProductHints::class)
class NativeHints {
    class ProductHints : RuntimeHintsRegistrar {
        override fun registerHints(hints: RuntimeHints, classLoader: ClassLoader?) {
            // Register classes for reflection
            hints.reflection()
                .registerType(Product::class.java) { hint ->
                    hint.withMembers()
                }

            // Register resources
            hints.resources()
                .registerPattern("messages/*.properties")
                .registerPattern("db/migration/*.sql")

            // Register serialization
            hints.serialization()
                .registerType(Product::class.java)
        }
    }
}
```

### ใช้ @RegisterReflectionForBinding

```kotlin
// ง่ายกว่า - ใช้ annotation โดยตรง
import org.springframework.aot.hint.annotation.RegisterReflectionForBinding

@RegisterReflectionForBinding(Product::class)
@RestController
@RequestMapping("/api/products")
class ProductController(
    private val productService: ProductService
) {
    // ...
}
```

---

## 🗂️ 6. application.properties สำหรับ Native

```properties
# src/main/resources/application.properties

# Database
spring.datasource.url=jdbc:h2:mem:testdb
spring.datasource.driver-class-name=org.h2.Driver
spring.jpa.database-platform=org.hibernate.dialect.H2Dialect
spring.h2.console.enabled=false

# JPA - ปิด lazy loading ที่อาจปัญหาใน Native
spring.jpa.open-in-view=false

# Actuator
management.endpoints.web.exposure.include=health,info,metrics
management.endpoint.health.show-details=always

# Native image specific
spring.aot.enabled=true
```

---

## 🔨 7. Build และ Run

### Build Native Image

```bash
# Build ด้วย Gradle (ใช้เวลานาน 5-15 นาที)
./gradlew nativeCompile

# หรือ build เป็น Docker image โดยตรง
./gradlew bootBuildImage

# ดู output
ls -la build/native/nativeCompile/
```

### Run Native Image

```bash
# Run native binary โดยตรง
./build/native/nativeCompile/my-native-app

# เปรียบเทียบกับ JVM
time ./build/native/nativeCompile/my-native-app &
time java -jar build/libs/my-app.jar &
```

### Docker Build

```dockerfile
# Dockerfile สำหรับ Native Image
FROM ghcr.io/graalvm/native-image-community:21-ol9 AS builder

WORKDIR /workspace

COPY . .
RUN ./gradlew nativeCompile --no-daemon

# Runtime image - เล็กมาก!
FROM debian:bookworm-slim

WORKDIR /app
COPY --from=builder /workspace/build/native/nativeCompile/my-native-app .

EXPOSE 8080
ENTRYPOINT ["./my-native-app"]
```

```bash
# Build Docker image
docker build -t my-native-app:latest .

# ดูขนาด image
docker images my-native-app
# REPOSITORY        TAG       IMAGE ID       SIZE
# my-native-app     latest    abc123         45.2MB   <- เล็กมาก!
```

---

## 📊 8. Benchmark: JVM vs Native

```bash
#!/bin/bash
# benchmark.sh

echo "=== JVM Startup Time ==="
time java -jar build/libs/app.jar &
APP_PID=$!
sleep 10
kill $APP_PID

echo ""
echo "=== Native Startup Time ==="
time ./build/native/nativeCompile/my-native-app &
NATIVE_PID=$!
sleep 2
kill $NATIVE_PID
```

### ผลลัพธ์ที่คาดหวัง

```
=== JVM Startup Time ===
Started in 3.456 seconds

real    0m3.456s
user    0m8.234s
sys     0m0.456s

=== Native Startup Time ===
Started in 0.089 seconds

real    0m0.089s
user    0m0.067s
sys     0m0.021s
```

---

## 🧪 9. Testing Native Image

```kotlin
// src/test/kotlin/com/example/NativeAppTest.kt
package com.example

import org.junit.jupiter.api.Test
import org.springframework.boot.test.context.SpringBootTest
import org.springframework.boot.test.web.client.TestRestTemplate
import org.springframework.boot.test.web.server.LocalServerPort
import org.springframework.http.HttpStatus
import org.springframework.test.context.TestPropertySource

// Test ที่รันบน native image จริง
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@TestPropertySource(properties = ["spring.datasource.url=jdbc:h2:mem:testdb"])
class NativeAppTest {

    @LocalServerPort
    private var port: Int = 0

    private val restTemplate = TestRestTemplate()

    @Test
    fun `should return all products`() {
        val response = restTemplate.getForEntity(
            "http://localhost:$port/api/products",
            String::class.java
        )
        assert(response.statusCode == HttpStatus.OK)
    }
}
```

```bash
# Run native tests (ช้ากว่า JVM tests มาก)
./gradlew nativeTest
```

---

## ⚠️ 10. ข้อจำกัดและปัญหาที่พบบ่อย

### Reflection Issues

```kotlin
// ปัญหา: ใช้ reflection โดยไม่มี hints
val obj = Class.forName("com.example.MyClass").newInstance()  // จะ error ใน native

// วิธีแก้: เพิ่ม hints
hints.reflection().registerType(MyClass::class.java) { it.withMembers() }
```

### Dynamic Proxy Issues

```kotlin
// Spring AOP ใช้ dynamic proxy - ต้องระบุ
hints.proxies().registerJdkProxy(MyInterface::class.java)
```

### Resource Loading

```kotlin
// ปัญหา: อ่าน classpath resource ใน native
val input = this::class.java.getResourceAsStream("/data.json")  // อาจ null

// วิธีแก้: register resource hint
hints.resources().registerPattern("data.json")
```

### ตัวอย่าง Error และวิธีแก้

```bash
# Error ที่พบบ่อย
com.oracle.svm.core.jdk.UnsupportedFeatureError: Proxy class defined by interfaces
[com.example.MyRepository] not found.

# วิธีแก้: เพิ่มใน build.gradle.kts
graalvmNative {
    binaries {
        named("main") {
            buildArgs.addAll(
                "--initialize-at-build-time=org.slf4j.LoggerFactory",
                "-H:+ReportExceptionStackTraces"
            )
        }
    }
}
```

---

## 📋 สรุป

| หัวข้อ | รายละเอียด |
|--------|-----------|
| GraalVM | JVM ทางเลือกที่ compile เป็น native binary ได้ |
| Native Image | Binary ที่รันโดยตรงไม่ต้องการ JVM |
| AOT Processing | Spring วิเคราะห์ application ตอน build time |
| Startup Time | เร็วกว่า JVM 10-100 เท่า |
| Memory Usage | น้อยกว่า JVM 5-10 เท่า |
| ข้อจำกัด | Reflection, Dynamic Proxy, Classpath scanning |
| Build Time | นาน 5-15 นาที |

### เหมาะกับงานประเภทไหน?

- **Serverless Functions** - Cold start สำคัญมาก
- **CLI Tools** - ต้องการ startup เร็ว
- **Microservices** - Memory footprint ต่ำ
- **Kubernetes** - Scale เร็ว memory น้อย

---

*Part 81/100+ | Kotlin & Spring Boot Complete Course*
