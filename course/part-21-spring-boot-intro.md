# Part 21: แนะนำ Spring Boot
## Introduction to Spring Boot

---

## 🎯 เป้าหมายของ Part นี้

- เข้าใจ Spring Framework และ Spring Boot
- รู้จัก IoC Container และ Dependency Injection
- สร้าง Spring Boot project แรก
- เข้าใจโครงสร้าง project
- รัน application และทดสอบ

---

## 🌱 1. Spring Framework คืออะไร?

Spring Framework เป็น **Java/Kotlin application framework** ที่ใหญ่ที่สุดในโลก ถูกสร้างในปี 2003 โดย Rod Johnson เพื่อแก้ปัญหาความซับซ้อนของ J2EE (Java Enterprise Edition)

### ทำไมต้องใช้ Spring?

```
ปัญหาที่ Spring แก้:
1. Object creation และ wiring (Dependency Injection)
2. Database access (Spring Data)
3. Security (Spring Security)
4. Web MVC (Spring MVC / WebFlux)
5. Transaction management
6. Testing
7. Integration กับ services ต่างๆ
```

### Spring Boot vs Spring Framework

```
Spring Framework:
- Full-featured framework
- ต้องตั้งค่าเยอะมาก (XML config หรือ Java Config)
- Flexible แต่ verbose

Spring Boot:
- Built on top of Spring Framework
- Convention over configuration
- Auto-configuration
- Embedded server (Tomcat, Netty)
- Production-ready ทันที (Actuator)
- Starter dependencies
```

---

## ⚙️ 2. Core Concepts

### IoC (Inversion of Control)

```kotlin
// ❌ แบบปกติ - Object สร้าง dependencies เอง (Tight coupling)
class OrderService {
    private val emailService = EmailService()  // สร้างเอง
    private val paymentService = PaymentService()  // สร้างเอง
    
    fun processOrder(order: Order) {
        paymentService.charge(order)
        emailService.sendConfirmation(order)
    }
}

// ✅ IoC - Dependencies ถูก inject จากภายนอก (Loose coupling)
class OrderService(
    private val emailService: EmailService,    // inject จากภายนอก
    private val paymentService: PaymentService  // inject จากภายนอก
) {
    fun processOrder(order: Order) {
        paymentService.charge(order)
        emailService.sendConfirmation(order)
    }
}
```

### Dependency Injection (DI)

```kotlin
// Interface-based DI
interface NotificationService {
    fun send(message: String)
}

class EmailNotificationService : NotificationService {
    override fun send(message: String) = println("Email: $message")
}

class SmsNotificationService : NotificationService {
    override fun send(message: String) = println("SMS: $message")
}

// ง่ายต่อการเปลี่ยน implementation
class UserService(private val notificationService: NotificationService) {
    fun register(username: String) {
        // save user...
        notificationService.send("Welcome, $username!")
    }
}

// Spring จัดการ DI ให้อัตโนมัติ
// @Service
// class UserService(@Autowired val notificationService: NotificationService)
```

### Spring Beans

```
Bean คือ Object ที่ Spring Container จัดการ:
- Spring สร้าง object ให้
- Spring จัดการ lifecycle
- Spring inject dependencies ให้
- Default scope คือ Singleton

Annotations:
@Component    - Generic bean
@Service      - Business logic layer  
@Repository   - Data access layer
@Controller   - Web controller
@Bean         - Manual bean declaration
```

---

## 🚀 3. สร้าง Spring Boot Project

### วิธีที่ 1: Spring Initializr (แนะนำ!)

ไปที่ https://start.spring.io/ แล้วตั้งค่า:

```
Project:  Gradle - Kotlin
Language: Kotlin
Spring Boot: 3.2.x (latest)
Group: com.example
Artifact: demo
Name: demo
Description: Demo project for Spring Boot
Package name: com.example.demo
Packaging: Jar
Java: 17

Dependencies:
✅ Spring Web
✅ Spring Data JPA
✅ H2 Database (embedded, for development)
✅ Spring Boot DevTools
```

คลิก **Generate** แล้ว unzip

### วิธีที่ 2: IntelliJ IDEA

1. File → New Project
2. เลือก **Spring Boot**
3. ตั้งค่าตามด้านบน
4. เลือก dependencies

### โครงสร้าง Project

```
demo/
├── build.gradle.kts
├── settings.gradle.kts
├── src/
│   ├── main/
│   │   ├── kotlin/
│   │   │   └── com/example/demo/
│   │   │       └── DemoApplication.kt    # Main class
│   │   └── resources/
│   │       ├── application.properties    # Config
│   │       ├── static/                   # Static files (CSS, JS)
│   │       └── templates/               # Templates (Thymeleaf etc.)
│   └── test/
│       └── kotlin/
│           └── com/example/demo/
│               └── DemoApplicationTests.kt
└── gradle/
```

### build.gradle.kts

```kotlin
import org.jetbrains.kotlin.gradle.tasks.KotlinCompile

plugins {
    id("org.springframework.boot") version "3.2.3"
    id("io.spring.dependency-management") version "1.1.4"
    kotlin("jvm") version "1.9.22"
    kotlin("plugin.spring") version "1.9.22"  // สำหรับ open classes
    kotlin("plugin.jpa") version "1.9.22"     // สำหรับ JPA entities
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
    // Spring Boot Starters
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
    implementation("org.springframework.boot:spring-boot-starter-validation")
    implementation("org.springframework.boot:spring-boot-starter-actuator")
    
    // Kotlin
    implementation("com.fasterxml.jackson.module:jackson-module-kotlin")
    implementation("org.jetbrains.kotlin:kotlin-reflect")
    
    // Database
    runtimeOnly("com.h2database:h2")  // In-memory DB for dev
    // runtimeOnly("org.postgresql:postgresql")  // For production
    
    // Dev tools
    developmentOnly("org.springframework.boot:spring-boot-devtools")
    
    // Testing
    testImplementation("org.springframework.boot:spring-boot-starter-test")
}

tasks.withType<KotlinCompile> {
    kotlinOptions {
        freeCompilerArgs += "-Xjsr305=strict"  // Null safety กับ Spring
        jvmTarget = "17"
    }
}

tasks.withType<Test> {
    useJUnitPlatform()
}
```

---

## 📄 4. Main Application Class

```kotlin
// DemoApplication.kt
package com.example.demo

import org.springframework.boot.autoconfigure.SpringBootApplication
import org.springframework.boot.runApplication

@SpringBootApplication  // = @Configuration + @ComponentScan + @EnableAutoConfiguration
class DemoApplication

fun main(args: Array<String>) {
    runApplication<DemoApplication>(*args)
}
```

---

## ⚡ 5. Hello World REST API

```kotlin
// HelloController.kt
package com.example.demo.controller

import org.springframework.web.bind.annotation.*

@RestController  // = @Controller + @ResponseBody
@RequestMapping("/api")
class HelloController {
    
    @GetMapping("/hello")
    fun hello(): String {
        return "Hello, Spring Boot with Kotlin!"
    }
    
    @GetMapping("/greet/{name}")
    fun greet(@PathVariable name: String): Map<String, String> {
        return mapOf(
            "message" to "Hello, $name!",
            "timestamp" to System.currentTimeMillis().toString()
        )
    }
    
    @PostMapping("/echo")
    fun echo(@RequestBody body: Map<String, Any>): Map<String, Any> {
        return mapOf(
            "received" to body,
            "processed" to true
        )
    }
}
```

### application.properties

```properties
# Server
server.port=8080

# App info
spring.application.name=demo

# H2 Database (development)
spring.datasource.url=jdbc:h2:mem:testdb
spring.datasource.driver-class-name=org.h2.Driver
spring.datasource.username=sa
spring.datasource.password=
spring.jpa.database-platform=org.hibernate.dialect.H2Dialect
spring.h2.console.enabled=true
spring.h2.console.path=/h2-console

# JPA
spring.jpa.show-sql=true
spring.jpa.hibernate.ddl-auto=create-drop

# Logging
logging.level.com.example=DEBUG
```

---

## ▶️ 6. รัน Application

```bash
# ด้วย Gradle
./gradlew bootRun

# หรือ build แล้วรัน
./gradlew build
java -jar build/libs/demo-0.0.1-SNAPSHOT.jar
```

**ผลลัพธ์:**
```
  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::                (v3.2.3)

2026-10-02T10:00:00 INFO  Starting DemoApplication
2026-10-02T10:00:00 INFO  Started DemoApplication in 2.345 seconds
```

### ทดสอบ API

```bash
# GET request
curl http://localhost:8080/api/hello
# Hello, Spring Boot with Kotlin!

curl http://localhost:8080/api/greet/Alice
# {"message":"Hello, Alice!","timestamp":"1696230000000"}

# POST request
curl -X POST http://localhost:8080/api/echo \
  -H "Content-Type: application/json" \
  -d '{"name":"Alice","age":25}'
# {"received":{"name":"Alice","age":25},"processed":true}
```

---

## 🔧 7. Spring Annotations ที่ต้องรู้

```kotlin
// Bean declarations
@SpringBootApplication  // Main class
@Component              // Generic bean
@Service                // Business logic
@Repository             // Data access
@Controller             // MVC controller
@RestController         // REST API controller
@Configuration          // Config class
@Bean                   // Method produces bean

// Dependency Injection
@Autowired              // Inject dependency (optional ใน Kotlin)
@Qualifier("name")      // เลือก bean ที่ชื่อนี้
@Primary                // Default bean ถ้ามีหลายตัว
@Value("${prop}")       // Inject config value
@ConfigurationProperties// Bind config อัตโนมัติ

// Web
@RequestMapping("/path")
@GetMapping("/path")
@PostMapping("/path")
@PutMapping("/path")
@DeleteMapping("/path")
@PatchMapping("/path")
@PathVariable           // จาก URL path
@RequestParam           // จาก query string
@RequestBody            // จาก request body
@ResponseStatus         // HTTP status code

// Database
@Entity                 // JPA entity
@Table(name="...")      // Database table
@Id                     // Primary key
@Column                 // Database column
@GeneratedValue         // Auto-generate ID
@Transactional          // Database transaction
```

---

## 🧪 8. Testing

```kotlin
// DemoApplicationTests.kt
package com.example.demo

import com.example.demo.controller.HelloController
import org.junit.jupiter.api.Test
import org.springframework.beans.factory.annotation.Autowired
import org.springframework.boot.test.context.SpringBootTest
import org.springframework.boot.test.web.client.TestRestTemplate
import org.springframework.boot.test.web.server.LocalServerPort
import org.assertj.core.api.Assertions.assertThat

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class DemoApplicationTests {
    
    @LocalServerPort
    private val port: Int = 0
    
    @Autowired
    private lateinit var restTemplate: TestRestTemplate
    
    @Autowired
    private lateinit var helloController: HelloController
    
    @Test
    fun contextLoads() {
        // Application context should load
        assertThat(helloController).isNotNull
    }
    
    @Test
    fun `hello endpoint should return greeting`() {
        val response = restTemplate.getForObject(
            "http://localhost:$port/api/hello",
            String::class.java
        )
        assertThat(response).contains("Hello, Spring Boot")
    }
    
    @Test
    fun `greet endpoint should include name`() {
        val response = restTemplate.getForObject(
            "http://localhost:$port/api/greet/Alice",
            Map::class.java
        )
        assertThat(response!!["message"]).isEqualTo("Hello, Alice!")
    }
}
```

---

## 🏋️ 9. แบบฝึกหัด

### ข้อ 1: สร้าง Calculator API
```kotlin
@RestController
@RequestMapping("/api/calc")
class CalculatorController {
    
    @GetMapping("/add")
    fun add(
        @RequestParam a: Double,
        @RequestParam b: Double
    ): Map<String, Any> = mapOf(
        "operation" to "addition",
        "a" to a,
        "b" to b,
        "result" to a + b
    )
    
    @GetMapping("/subtract")
    fun subtract(@RequestParam a: Double, @RequestParam b: Double) =
        mapOf("result" to a - b)
    
    @GetMapping("/multiply")
    fun multiply(@RequestParam a: Double, @RequestParam b: Double) =
        mapOf("result" to a * b)
    
    @GetMapping("/divide")
    fun divide(@RequestParam a: Double, @RequestParam b: Double): Map<String, Any> {
        if (b == 0.0) return mapOf("error" to "Cannot divide by zero")
        return mapOf("result" to a / b)
    }
}
```

ทดสอบ:
```bash
curl "http://localhost:8080/api/calc/add?a=10&b=5"
# {"operation":"addition","a":10.0,"b":5.0,"result":15.0}

curl "http://localhost:8080/api/calc/divide?a=10&b=0"
# {"error":"Cannot divide by zero"}
```

---

## 📝 สรุป Part 21

| แนวคิด | รายละเอียด |
|--------|-----------|
| IoC | Spring จัดการ object lifecycle |
| DI | Spring inject dependencies ให้ |
| Spring Boot | Auto-configuration + embedded server |
| `@SpringBootApplication` | Main annotation |
| `@RestController` | REST API controller |
| `@GetMapping`/`@PostMapping` | HTTP method mapping |
| application.properties | Configuration |

---

## ➡️ ถัดไป: Part 22 - Spring Boot + Kotlin Setup

เรียนรู้การ configure Spring Boot สำหรับ Kotlin โดยเฉพาะ:
- Kotlin-specific Spring features
- Jackson configuration
- Coroutines integration
- Testing setup

---
*Part 21/100+ | Kotlin & Spring Boot Complete Course*
