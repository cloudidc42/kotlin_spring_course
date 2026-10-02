# Part 41: API Versioning
## การจัดการเวอร์ชัน API อย่างมืออาชีพ

---

## 🎯 เป้าหมายของ Part นี้

- เข้าใจว่าทำไมต้อง Version API
- URL Versioning (/api/v1/, /api/v2/)
- Header Versioning
- Custom RequestMappingHandlerMapping
- Version-specific DTOs
- Backward compatibility strategies
- ตัวอย่างจริง: Evolving User API

---

## 📚 1. ทำไมต้อง API Versioning?

การพัฒนา API มักต้องการเปลี่ยนแปลง structure ของ response หรือ behavior ของ endpoint แต่ถ้า client อื่นยังใช้ API เดิมอยู่ การเปลี่ยนแปลงอาจทำให้ระบบเดิมพัง API Versioning จึงเป็นวิธีที่ทำให้เราสามารถพัฒนา API ต่อไปได้โดยไม่กระทบ client เดิม

### ปัญหาที่เกิดขึ้นบ่อย:

```
Week 1: เปิดตัว API v1
  GET /api/users/{id} → { "name": "John", "email": "john@email.com" }

Week 4: ต้องการแยก name เป็น firstName และ lastName
  GET /api/users/{id} → { "firstName": "John", "lastName": "Doe", "email": "john@email.com" }

ปัญหา: Mobile app ที่ใช้ "name" field จะพัง!
```

### Versioning Strategies:

| Strategy | ตัวอย่าง | ข้อดี | ข้อเสีย |
|----------|---------|-------|---------|
| URL Path | `/api/v1/users` | ชัดเจน, cacheable | URL ยาว |
| Query Param | `/api/users?version=1` | ง่ายต่อการทดสอบ | ไม่สวย, ลืมได้ |
| Header | `API-Version: 1` | URL สะอาด | ซับซ้อนกว่า |
| Content-Type | `Accept: application/vnd.api.v1+json` | RESTful ที่สุด | ซับซ้อนมาก |

---

## 🔧 2. Project Setup

```kotlin
// build.gradle.kts
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
    implementation("org.springframework.boot:spring-boot-starter-validation")
    implementation("com.fasterxml.jackson.module:jackson-module-kotlin")
    implementation("org.jetbrains.kotlin:kotlin-reflect")
    runtimeOnly("com.h2database:h2")
    testImplementation("org.springframework.boot:spring-boot-starter-test")
}
```

```yaml
# application.yml
spring:
  application:
    name: api-versioning-demo
  datasource:
    url: jdbc:h2:mem:testdb
    driver-class-name: org.h2.Driver
  jpa:
    hibernate:
      ddl-auto: create-drop
    show-sql: true
  h2:
    console:
      enabled: true
```

---

## 🛣️ 3. URL Path Versioning

วิธีที่ง่ายและพบเห็นบ่อยที่สุด คือการใส่ version ไว้ใน URL path

### Entity

```kotlin
// entity/User.kt
package com.example.versioning.entity

import jakarta.persistence.*

@Entity
@Table(name = "users")
data class User(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long = 0,
    
    @Column(nullable = false)
    val firstName: String = "",
    
    @Column(nullable = false)
    val lastName: String = "",
    
    @Column(unique = true, nullable = false)
    val email: String = "",
    
    @Column
    val phoneNumber: String? = null,
    
    @Column
    val address: String? = null,
    
    @Column(nullable = false)
    val active: Boolean = true
)
```

### V1 DTO (format เดิม - name เป็น String เดียว)

```kotlin
// dto/v1/UserDtoV1.kt
package com.example.versioning.dto.v1

data class UserResponseV1(
    val id: Long,
    val name: String,        // รวม firstName + lastName
    val email: String
)

data class CreateUserRequestV1(
    val name: String,
    val email: String
)
```

### V2 DTO (format ใหม่ - แยก firstName/lastName)

```kotlin
// dto/v2/UserDtoV2.kt
package com.example.versioning.dto.v2

data class UserResponseV2(
    val id: Long,
    val firstName: String,   // แยกเป็น field ต่างหาก
    val lastName: String,
    val email: String,
    val phoneNumber: String?,
    val fullName: String = "$firstName $lastName"  // computed field
)

data class CreateUserRequestV2(
    val firstName: String,
    val lastName: String,
    val email: String,
    val phoneNumber: String? = null
)
```

### Repository

```kotlin
// repository/UserRepository.kt
package com.example.versioning.repository

import com.example.versioning.entity.User
import org.springframework.data.jpa.repository.JpaRepository
import org.springframework.stereotype.Repository

@Repository
interface UserRepository : JpaRepository<User, Long> {
    fun findByEmail(email: String): User?
    fun findByActive(active: Boolean): List<User>
}
```

### Service

```kotlin
// service/UserService.kt
package com.example.versioning.service

import com.example.versioning.dto.v1.CreateUserRequestV1
import com.example.versioning.dto.v1.UserResponseV1
import com.example.versioning.dto.v2.CreateUserRequestV2
import com.example.versioning.dto.v2.UserResponseV2
import com.example.versioning.entity.User
import com.example.versioning.repository.UserRepository
import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional

@Service
class UserService(private val userRepository: UserRepository) {

    // V1 Methods
    fun getUserV1(id: Long): UserResponseV1 {
        val user = userRepository.findById(id)
            .orElseThrow { NoSuchElementException("User not found: $id") }
        return user.toResponseV1()
    }

    fun getAllUsersV1(): List<UserResponseV1> {
        return userRepository.findAll().map { it.toResponseV1() }
    }

    @Transactional
    fun createUserV1(request: CreateUserRequestV1): UserResponseV1 {
        // แยก name เป็น firstName, lastName
        val nameParts = request.name.split(" ", limit = 2)
        val user = User(
            firstName = nameParts.getOrElse(0) { request.name },
            lastName = nameParts.getOrElse(1) { "" },
            email = request.email
        )
        return userRepository.save(user).toResponseV1()
    }

    // V2 Methods
    fun getUserV2(id: Long): UserResponseV2 {
        val user = userRepository.findById(id)
            .orElseThrow { NoSuchElementException("User not found: $id") }
        return user.toResponseV2()
    }

    fun getAllUsersV2(): List<UserResponseV2> {
        return userRepository.findAll().map { it.toResponseV2() }
    }

    @Transactional
    fun createUserV2(request: CreateUserRequestV2): UserResponseV2 {
        val user = User(
            firstName = request.firstName,
            lastName = request.lastName,
            email = request.email,
            phoneNumber = request.phoneNumber
        )
        return userRepository.save(user).toResponseV2()
    }

    // Mapper functions
    private fun User.toResponseV1() = UserResponseV1(
        id = id,
        name = "$firstName $lastName".trim(),
        email = email
    )

    private fun User.toResponseV2() = UserResponseV2(
        id = id,
        firstName = firstName,
        lastName = lastName,
        email = email,
        phoneNumber = phoneNumber
    )
}
```

### V1 Controller

```kotlin
// controller/v1/UserControllerV1.kt
package com.example.versioning.controller.v1

import com.example.versioning.dto.v1.CreateUserRequestV1
import com.example.versioning.dto.v1.UserResponseV1
import com.example.versioning.service.UserService
import org.springframework.http.HttpStatus
import org.springframework.http.ResponseEntity
import org.springframework.web.bind.annotation.*

@RestController
@RequestMapping("/api/v1/users")
class UserControllerV1(private val userService: UserService) {

    @GetMapping
    fun getAllUsers(): ResponseEntity<List<UserResponseV1>> {
        return ResponseEntity.ok(userService.getAllUsersV1())
    }

    @GetMapping("/{id}")
    fun getUserById(@PathVariable id: Long): ResponseEntity<UserResponseV1> {
        return ResponseEntity.ok(userService.getUserV1(id))
    }

    @PostMapping
    fun createUser(@RequestBody request: CreateUserRequestV1): ResponseEntity<UserResponseV1> {
        return ResponseEntity.status(HttpStatus.CREATED)
            .body(userService.createUserV1(request))
    }
}
```

### V2 Controller

```kotlin
// controller/v2/UserControllerV2.kt
package com.example.versioning.controller.v2

import com.example.versioning.dto.v2.CreateUserRequestV2
import com.example.versioning.dto.v2.UserResponseV2
import com.example.versioning.service.UserService
import org.springframework.http.HttpStatus
import org.springframework.http.ResponseEntity
import org.springframework.web.bind.annotation.*

@RestController
@RequestMapping("/api/v2/users")
class UserControllerV2(private val userService: UserService) {

    @GetMapping
    fun getAllUsers(): ResponseEntity<List<UserResponseV2>> {
        return ResponseEntity.ok(userService.getAllUsersV2())
    }

    @GetMapping("/{id}")
    fun getUserById(@PathVariable id: Long): ResponseEntity<UserResponseV2> {
        return ResponseEntity.ok(userService.getUserV2(id))
    }

    @PostMapping
    fun createUser(@RequestBody request: CreateUserRequestV2): ResponseEntity<UserResponseV2> {
        return ResponseEntity.status(HttpStatus.CREATED)
            .body(userService.createUserV2(request))
    }
}
```

---

## 📨 4. Header-based Versioning

บางองค์กรชอบใช้ HTTP header แทนการใส่ version ใน URL

```kotlin
// config/ApiVersionRequestMappingHandlerMapping.kt
package com.example.versioning.config

import org.springframework.web.servlet.mvc.method.RequestMappingInfo
import org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping
import java.lang.reflect.Method

@Target(AnnotationTarget.CLASS, AnnotationTarget.FUNCTION)
@Retention(AnnotationRetention.RUNTIME)
annotation class ApiVersion(val value: String)
```

```kotlin
// controller/UserControllerHeader.kt
package com.example.versioning.controller

import com.example.versioning.config.ApiVersion
import com.example.versioning.dto.v1.UserResponseV1
import com.example.versioning.dto.v2.UserResponseV2
import com.example.versioning.service.UserService
import org.springframework.http.ResponseEntity
import org.springframework.web.bind.annotation.*

@RestController
@RequestMapping("/api/users")
class UserControllerHeader(private val userService: UserService) {

    // รับ header: X-API-Version: 1
    @GetMapping(headers = ["X-API-Version=1"])
    fun getAllUsersV1(): ResponseEntity<List<UserResponseV1>> {
        return ResponseEntity.ok(userService.getAllUsersV1())
    }

    // รับ header: X-API-Version: 2
    @GetMapping(headers = ["X-API-Version=2"])
    fun getAllUsersV2(): ResponseEntity<List<UserResponseV2>> {
        return ResponseEntity.ok(userService.getAllUsersV2())
    }

    // Default (ไม่มี header) → ใช้ V1 เป็น fallback
    @GetMapping
    fun getAllUsersDefault(): ResponseEntity<List<UserResponseV1>> {
        return ResponseEntity.ok(userService.getAllUsersV1())
    }
}
```

---

## 🔀 5. Custom RequestMappingHandlerMapping

สำหรับระบบที่ต้องการ versioning แบบยืดหยุ่นมากขึ้น สามารถสร้าง custom handler mapping ได้

```kotlin
// config/VersionedRequestMappingHandlerMapping.kt
package com.example.versioning.config

import org.springframework.web.bind.annotation.RequestMapping
import org.springframework.web.servlet.mvc.method.RequestMappingInfo
import org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping
import java.lang.reflect.Method

class VersionedRequestMappingHandlerMapping : RequestMappingHandlerMapping() {

    override fun getMappingForMethod(method: Method, handlerType: Class<*>): RequestMappingInfo? {
        val info = super.getMappingForMethod(method, handlerType) ?: return null

        val methodVersion = method.getAnnotation(ApiVersion::class.java)
        val classVersion = handlerType.getAnnotation(ApiVersion::class.java)

        val version = methodVersion ?: classVersion

        return if (version != null) {
            val versionInfo = RequestMappingInfo
                .paths("/api/v${version.value}")
                .build()
            versionInfo.combine(info)
        } else {
            info
        }
    }
}
```

```kotlin
// config/WebMvcConfig.kt
package com.example.versioning.config

import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.web.servlet.config.annotation.WebMvcConfigurationSupport

@Configuration
class WebMvcConfig : WebMvcConfigurationSupport() {

    @Bean
    override fun requestMappingHandlerMapping(
        contentNegotiationManager: org.springframework.web.accept.ContentNegotiationManager,
        conversionService: org.springframework.core.convert.ConversionService,
        resourceUrlProvider: org.springframework.web.servlet.resource.ResourceUrlProvider
    ) = VersionedRequestMappingHandlerMapping()
}
```

---

## 🔄 6. Backward Compatibility Strategies

### Strategy 1: Deprecation headers

```kotlin
// interceptor/DeprecationInterceptor.kt
package com.example.versioning.interceptor

import jakarta.servlet.http.HttpServletRequest
import jakarta.servlet.http.HttpServletResponse
import org.springframework.web.servlet.HandlerInterceptor

class DeprecationInterceptor : HandlerInterceptor {

    companion object {
        private val DEPRECATED_PATHS = setOf("/api/v1/")
        private val SUNSET_DATE = "2025-12-31"
    }

    override fun postHandle(
        request: HttpServletRequest,
        response: HttpServletResponse,
        handler: Any,
        modelAndView: org.springframework.web.servlet.ModelAndView?
    ) {
        val path = request.requestURI
        if (DEPRECATED_PATHS.any { path.startsWith(it) }) {
            response.setHeader("Deprecation", "true")
            response.setHeader("Sunset", SUNSET_DATE)
            response.setHeader(
                "Link",
                "</api/v2${path.substringAfter("/api/v1")}>; rel=\"successor-version\""
            )
        }
    }
}
```

```kotlin
// config/WebConfig.kt (register interceptor)
package com.example.versioning.config

import com.example.versioning.interceptor.DeprecationInterceptor
import org.springframework.context.annotation.Configuration
import org.springframework.web.servlet.config.annotation.InterceptorRegistry
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer

@Configuration
class WebConfig : WebMvcConfigurer {

    override fun addInterceptors(registry: InterceptorRegistry) {
        registry.addInterceptor(DeprecationInterceptor())
    }
}
```

### Strategy 2: Version migration helper

```kotlin
// service/VersionMigrationService.kt
package com.example.versioning.service

import com.example.versioning.dto.v1.UserResponseV1
import com.example.versioning.dto.v2.UserResponseV2
import org.springframework.stereotype.Service

@Service
class VersionMigrationService {

    // แปลง V2 response เป็น V1 (backward compat)
    fun downgradeToV1(userV2: UserResponseV2): UserResponseV1 {
        return UserResponseV1(
            id = userV2.id,
            name = "${userV2.firstName} ${userV2.lastName}".trim(),
            email = userV2.email
        )
    }

    // แปลง V1 response เป็น V2 (upgrade)
    fun upgradeToV2(userV1: UserResponseV1): UserResponseV2 {
        val nameParts = userV1.name.split(" ", limit = 2)
        return UserResponseV2(
            id = userV1.id,
            firstName = nameParts.getOrElse(0) { userV1.name },
            lastName = nameParts.getOrElse(1) { "" },
            email = userV1.email,
            phoneNumber = null
        )
    }
}
```

---

## 📊 7. Version Routing with Accept Header (Content Negotiation)

```kotlin
// controller/UserControllerContentNegotiation.kt
package com.example.versioning.controller

import com.example.versioning.dto.v1.UserResponseV1
import com.example.versioning.dto.v2.UserResponseV2
import com.example.versioning.service.UserService
import org.springframework.http.ResponseEntity
import org.springframework.web.bind.annotation.*

@RestController
@RequestMapping("/api/users")
class UserControllerContentNegotiation(private val userService: UserService) {

    // curl -H "Accept: application/vnd.company.api.v1+json"
    @GetMapping(
        produces = ["application/vnd.company.api.v1+json"]
    )
    fun getAllUsersV1(): ResponseEntity<List<UserResponseV1>> {
        return ResponseEntity.ok(userService.getAllUsersV1())
    }

    // curl -H "Accept: application/vnd.company.api.v2+json"
    @GetMapping(
        produces = ["application/vnd.company.api.v2+json"]
    )
    fun getAllUsersV2(): ResponseEntity<List<UserResponseV2>> {
        return ResponseEntity.ok(userService.getAllUsersV2())
    }
}
```

---

## 🧪 8. Testing API Versions

```kotlin
// test/UserControllerTest.kt
package com.example.versioning

import org.junit.jupiter.api.Test
import org.springframework.beans.factory.annotation.Autowired
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc
import org.springframework.boot.test.context.SpringBootTest
import org.springframework.http.MediaType
import org.springframework.test.web.servlet.MockMvc
import org.springframework.test.web.servlet.get
import org.springframework.test.web.servlet.post

@SpringBootTest
@AutoConfigureMockMvc
class UserControllerTest {

    @Autowired
    private lateinit var mockMvc: MockMvc

    @Test
    fun `V1 API returns combined name field`() {
        // สร้าง user ก่อน
        mockMvc.post("/api/v1/users") {
            contentType = MediaType.APPLICATION_JSON
            content = """{"name": "John Doe", "email": "john@test.com"}"""
        }.andExpect {
            status { isCreated() }
        }

        // ดึงข้อมูลผ่าน V1
        mockMvc.get("/api/v1/users/1")
            .andExpect {
                status { isOk() }
                jsonPath("$.name") { value("John Doe") }
                jsonPath("$.firstName") { doesNotExist() }  // V1 ไม่มี field นี้
            }
    }

    @Test
    fun `V2 API returns separate firstName and lastName`() {
        mockMvc.post("/api/v2/users") {
            contentType = MediaType.APPLICATION_JSON
            content = """{"firstName": "Jane", "lastName": "Smith", "email": "jane@test.com"}"""
        }.andExpect {
            status { isCreated() }
        }

        mockMvc.get("/api/v2/users/1")
            .andExpect {
                status { isOk() }
                jsonPath("$.firstName") { value("Jane") }
                jsonPath("$.lastName") { value("Smith") }
                jsonPath("$.fullName") { value("Jane Smith") }
                jsonPath("$.name") { doesNotExist() }  // V2 ไม่มี field นี้
            }
    }

    @Test
    fun `V1 endpoint returns deprecation headers`() {
        mockMvc.get("/api/v1/users")
            .andExpect {
                status { isOk() }
                header { exists("Deprecation") }
                header { exists("Sunset") }
            }
    }
}
```

---

## 🏗️ 9. Data Migration Example

เมื่อมีการเปลี่ยน version ของ API มักต้องมี data migration ด้วย

```kotlin
// migration/UserDataMigration.kt
package com.example.versioning.migration

import com.example.versioning.repository.UserRepository
import org.springframework.boot.CommandLineRunner
import org.springframework.context.annotation.Profile
import org.springframework.stereotype.Component

@Component
@Profile("migration")
class UserDataMigration(private val userRepository: UserRepository) : CommandLineRunner {

    override fun run(vararg args: String?) {
        println("Starting user data migration...")

        val users = userRepository.findAll()
        var migrated = 0

        users.forEach { user ->
            // ตรวจสอบว่า user มี firstName ที่ถูกต้องหรือไม่
            if (user.firstName.isBlank()) {
                // มี bug เก่าที่ไม่ได้แยก name
                println("Skipping user ${user.id} - missing firstName")
                return@forEach
            }
            migrated++
        }

        println("Migration complete: $migrated users processed")
    }
}
```

---

## 📋 สรุปเนื้อหา

| หัวข้อ | รายละเอียด |
|--------|------------|
| URL Versioning | ใส่ version ใน path เช่น `/api/v1/`, `/api/v2/` |
| Header Versioning | ใช้ header `X-API-Version: 1` |
| Content Negotiation | ใช้ Accept header แบบ `application/vnd.api.v1+json` |
| Deprecation Headers | แจ้ง client ว่า endpoint กำลังจะถูกลบ |
| Migration Helpers | แปลง response ระหว่าง version |
| Backward Compat | V2 service ยังสามารถ serve V1 format ได้ |

### Best Practices:

- เริ่มต้นด้วย URL versioning เพราะง่ายที่สุด
- ประกาศ deprecation ล่วงหน้า 6+ เดือน
- เก็บ V1 ไว้อย่างน้อย 12 เดือนหลัง deprecate
- ใช้ semantic versioning (major เปลี่ยน = breaking change)
- เขียน changelog ทุกครั้งที่เปลี่ยน version

---

*Part 41/100+ | Kotlin & Spring Boot Complete Course*
