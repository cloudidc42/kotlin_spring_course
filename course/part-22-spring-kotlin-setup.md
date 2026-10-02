# Part 22: Spring Boot + Kotlin Setup
## Kotlin-specific Spring Configuration

---

## 🎯 เป้าหมายของ Part นี้

- ตั้งค่า Spring Boot สำหรับ Kotlin
- Jackson กับ Kotlin data classes
- Null Safety ใน Spring
- Spring Boot DSL (Kotlin)
- DevTools และ Hot Reload
- Profile configuration

---

## 🔧 1. Kotlin Plugin สำหรับ Spring

```kotlin
// build.gradle.kts - plugins ที่จำเป็น
plugins {
    id("org.springframework.boot") version "3.2.3"
    id("io.spring.dependency-management") version "1.1.4"
    kotlin("jvm") version "1.9.22"
    kotlin("plugin.spring") version "1.9.22"   // ทำให้ Spring classes เป็น open
    kotlin("plugin.jpa") version "1.9.22"       // ทำให้ JPA entities ทำงานได้
    kotlin("plugin.allopen") version "1.9.22"   // allopen support
}

// kotlin("plugin.spring") ทำให้ class ที่มี annotations เหล่านี้ เป็น open:
// @Component, @Service, @Repository, @Controller, @RestController
// @Configuration, @Bean

// ถ้าไม่มี plugin.spring จะ error เพราะ Kotlin classes เป็น final by default
// Spring ต้องสร้าง subclass (proxy) ไม่ได้!
```

---

## 📦 2. Jackson กับ Kotlin

### ปัญหากับ Data Classes

```kotlin
// ปัญหา: Jackson ต้องการ no-arg constructor
// Data class ไม่มี no-arg constructor

data class User(val name: String, val age: Int)

// ❌ โดยไม่มี jackson-module-kotlin จะ error:
// com.fasterxml.jackson.databind.exc.MismatchedInputException

// ✅ แก้โดยเพิ่ม jackson-module-kotlin
implementation("com.fasterxml.jackson.module:jackson-module-kotlin")
```

### Jackson Configuration

```kotlin
// JacksonConfig.kt
package com.example.demo.config

import com.fasterxml.jackson.databind.ObjectMapper
import com.fasterxml.jackson.databind.SerializationFeature
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule
import com.fasterxml.jackson.module.kotlin.registerKotlinModule
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.http.converter.json.Jackson2ObjectMapperBuilder

@Configuration
class JacksonConfig {
    
    @Bean
    fun objectMapper(builder: Jackson2ObjectMapperBuilder): ObjectMapper {
        return builder
            .modules(JavaTimeModule())                    // สำหรับ LocalDateTime
            .featuresToDisable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS)
            .build<ObjectMapper>()
            .apply {
                registerKotlinModule()
            }
    }
}
```

### application.yml (แนะนำแทน .properties)

```yaml
# application.yml
spring:
  application:
    name: my-app
  
  jackson:
    serialization:
      write-dates-as-timestamps: false
      indent-output: false
    deserialization:
      fail-on-unknown-properties: false
    default-property-inclusion: non_null   # ไม่ส่ง null fields
    time-zone: Asia/Bangkok
  
  datasource:
    url: jdbc:h2:mem:testdb
    driver-class-name: org.h2.Driver
    username: sa
    password: ""
  
  h2:
    console:
      enabled: true
      path: /h2-console
  
  jpa:
    show-sql: true
    hibernate:
      ddl-auto: create-drop
    properties:
      hibernate:
        format_sql: true
        dialect: org.hibernate.dialect.H2Dialect

server:
  port: 8080

logging:
  level:
    com.example: DEBUG
    org.springframework: INFO
```

---

## 🌿 3. Spring Boot กับ Kotlin Null Safety

```kotlin
// Spring เข้าใจ Kotlin nullable types
@RestController
class UserController {
    
    // @RequestParam กับ nullable
    @GetMapping("/users")
    fun getUsers(
        @RequestParam name: String?,     // Optional parameter
        @RequestParam(required = true) page: Int
    ): List<String> {
        return if (name != null) {
            listOf("Found user: $name on page $page")
        } else {
            listOf("All users on page $page")
        }
    }
    
    // @PathVariable ที่ non-null (Spring จะ throw ถ้าไม่มี)
    @GetMapping("/users/{id}")
    fun getUser(@PathVariable id: Long): Map<String, Any> {
        return mapOf("id" to id, "name" to "User $id")
    }
    
    // @RequestBody กับ data class
    @PostMapping("/users")
    fun createUser(@RequestBody user: CreateUserRequest): UserResponse {
        return UserResponse(
            id = 1L,
            name = user.name,
            email = user.email
        )
    }
}

data class CreateUserRequest(
    val name: String,
    val email: String,
    val age: Int? = null  // Optional
)

data class UserResponse(
    val id: Long,
    val name: String,
    val email: String
)
```

---

## 🌸 4. Spring Boot DSL (Kotlin)

```kotlin
// แบบ Java-style
@SpringBootApplication
class Application

fun main(args: Array<String>) {
    runApplication<Application>(*args)
}

// แบบ Kotlin DSL (Spring Boot 2.x+)
fun main(args: Array<String>) {
    runApplication<Application>(*args) {
        setBannerMode(Banner.Mode.OFF)
        setAdditionalProfiles("dev")
    }
}

// Beans DSL (Spring Boot 2.x+)
@Configuration
class AppConfig {
    
    @Bean
    fun myService(repo: UserRepository) = UserService(repo)
}

// หรือแบบ Kotlin functional
val beans = beans {
    bean<UserRepository>()
    bean { UserService(ref()) }
    bean { UserController(ref()) }
}

// ใช้ใน Application
@SpringBootApplication
class Application

fun main(args: Array<String>) {
    runApplication<Application>(*args) {
        addInitializers(beans)
    }
}
```

---

## 📋 5. Configuration Properties

```kotlin
// AppProperties.kt
package com.example.demo.config

import org.springframework.boot.context.properties.ConfigurationProperties
import org.springframework.boot.context.properties.bind.ConstructorBinding

@ConfigurationProperties(prefix = "app")
data class AppProperties(
    val name: String = "My App",
    val version: String = "1.0.0",
    val features: Features = Features(),
    val security: Security = Security()
) {
    data class Features(
        val enableRegistration: Boolean = true,
        val enableSocialLogin: Boolean = false,
        val maxUploadSizeMb: Int = 10
    )
    
    data class Security(
        val jwtSecret: String = "secret",
        val jwtExpirationMs: Long = 86400000L,
        val allowedOrigins: List<String> = listOf("http://localhost:3000")
    )
}

// เปิดใช้
@SpringBootApplication
@EnableConfigurationProperties(AppProperties::class)  // ต้องเพิ่มบรรทัดนี้
class DemoApplication
```

```yaml
# application.yml
app:
  name: "Task Manager"
  version: "2.0.0"
  features:
    enable-registration: true
    enable-social-login: true
    max-upload-size-mb: 20
  security:
    jwt-secret: "my-very-secret-key-change-in-production"
    jwt-expiration-ms: 3600000
    allowed-origins:
      - "http://localhost:3000"
      - "https://myapp.example.com"
```

```kotlin
// ใช้งาน
@Service
class UserService(private val appProperties: AppProperties) {
    
    fun canRegister(): Boolean = appProperties.features.enableRegistration
    
    fun getJwtConfig() = appProperties.security
}
```

---

## 🔄 6. Profiles

```yaml
# application.yml (shared)
spring:
  application:
    name: my-app

app:
  version: "1.0.0"
```

```yaml
# application-dev.yml
spring:
  datasource:
    url: jdbc:h2:mem:devdb
  jpa:
    show-sql: true
    hibernate.ddl-auto: create-drop
  h2:
    console.enabled: true

logging:
  level:
    com.example: DEBUG

server:
  port: 8080
```

```yaml
# application-prod.yml
spring:
  datasource:
    url: ${DATABASE_URL}
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
  jpa:
    show-sql: false
    hibernate.ddl-auto: validate

logging:
  level:
    root: WARN
    com.example: INFO

server:
  port: ${PORT:8080}
```

```kotlin
// เลือก profile
// Environment variable: SPRING_PROFILES_ACTIVE=prod
// Command line: --spring.profiles.active=prod
// JVM: -Dspring.profiles.active=prod

// Profile-specific beans
@Configuration
class DataSourceConfig {
    
    @Bean
    @Profile("dev")
    fun h2DataSource(): DataSource {
        println("Using H2 (dev)")
        return EmbeddedDatabaseBuilder()
            .setType(EmbeddedDatabaseType.H2)
            .build()
    }
    
    @Bean
    @Profile("prod")
    fun postgresDataSource(): DataSource {
        println("Using PostgreSQL (prod)")
        // configure real datasource
        return HikariDataSource(HikariConfig().apply {
            jdbcUrl = System.getenv("DATABASE_URL")
        })
    }
}
```

---

## 🌡️ 7. Spring Boot Actuator

```yaml
# application.yml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,env
      base-path: /actuator
  endpoint:
    health:
      show-details: always
  info:
    env:
      enabled: true

info:
  app:
    name: "@project.name@"
    version: "@project.version@"
```

```bash
# Health check
curl http://localhost:8080/actuator/health
# {"status":"UP","components":{"db":{"status":"UP"},"diskSpace":{"status":"UP"}}}

# App info
curl http://localhost:8080/actuator/info
# {"app":{"name":"demo","version":"0.0.1-SNAPSHOT"}}

# Metrics
curl http://localhost:8080/actuator/metrics
curl http://localhost:8080/actuator/metrics/jvm.memory.used
```

---

## 🧪 8. Testing Setup

```kotlin
// TestConfig.kt
package com.example.demo

import org.springframework.boot.test.context.TestConfiguration
import org.springframework.context.annotation.Bean

@TestConfiguration
class TestConfig {
    // Test-specific beans
}

// Integration Test
@SpringBootTest
@ActiveProfiles("test")
class IntegrationTest {
    
    @Autowired
    private lateinit var userService: UserService
    
    @Test
    fun `should create user successfully`() {
        val user = userService.createUser("Alice", "alice@example.com")
        assertThat(user.id).isNotNull()
        assertThat(user.name).isEqualTo("Alice")
    }
}

// Web Layer Test (ไม่ต้อง start server จริง)
@WebMvcTest(UserController::class)
class UserControllerTest {
    
    @Autowired
    private lateinit var mockMvc: MockMvc
    
    @MockBean
    private lateinit var userService: UserService
    
    @Test
    fun `GET users should return 200`() {
        mockMvc.get("/api/users") {
            contentType = MediaType.APPLICATION_JSON
        }.andExpect {
            status { isOk() }
            content { contentType(MediaType.APPLICATION_JSON) }
        }
    }
}
```

---

## 🏋️ 9. แบบฝึกหัด

### ข้อ 1: สร้าง Custom Health Indicator
```kotlin
@Component
class DatabaseHealthIndicator(private val dataSource: DataSource) : HealthIndicator {
    
    override fun health(): Health {
        return try {
            dataSource.connection.use { conn ->
                val meta = conn.metaData
                Health.up()
                    .withDetail("database", meta.databaseProductName)
                    .withDetail("version", meta.databaseProductVersion)
                    .build()
            }
        } catch (e: Exception) {
            Health.down()
                .withException(e)
                .build()
        }
    }
}
```

### ข้อ 2: Environment-aware configuration
```kotlin
@Configuration
class EnvironmentConfig(
    private val environment: Environment,
    private val appProperties: AppProperties
) {
    
    @PostConstruct
    fun logStartup() {
        println("=".repeat(50))
        println("App: ${appProperties.name} v${appProperties.version}")
        println("Profiles: ${environment.activeProfiles.joinToString()}")
        println("Port: ${environment.getProperty("server.port")}")
        println("=".repeat(50))
    }
}
```

---

## 📝 สรุป Part 22

| ส่วน | รายละเอียด |
|-----|-----------|
| Kotlin plugin.spring | ทำ Spring classes เป็น open |
| Jackson module | Kotlin data class serialization |
| Null safety | `@RequestParam String?` |
| ConfigurationProperties | Type-safe config binding |
| Profiles | dev/test/prod configurations |
| Actuator | Health, metrics, info endpoints |

---

## ➡️ ถัดไป: Part 23 - REST API พื้นฐาน

---
*Part 22/100+ | Kotlin & Spring Boot Complete Course*
