# Part 44: API Documentation (OpenAPI/Swagger)
## การสร้าง API Documentation อัตโนมัติด้วย springdoc-openapi

---

## 🎯 เป้าหมายของ Part นี้

- ติดตั้งและ configure springdoc-openapi
- @Operation, @ApiResponse, @Parameter annotations
- Schema customization ด้วย @Schema
- Security schemes ใน Swagger UI
- Grouping endpoints
- Custom OpenAPI configuration
- ตัวอย่างจริง: Fully documented REST API

---

## 📚 1. ทำไมต้องมี API Documentation?

API Documentation ที่ดีช่วยให้:
- Frontend developers รู้ว่า API ทำงานอย่างไร
- ทดสอบ API ได้โดยตรงจาก Swagger UI
- Generate client SDK ได้อัตโนมัติ
- ลดเวลาในการ onboard developer ใหม่
- เป็น source of truth สำหรับ API contract

### OpenAPI Specification:

```yaml
# ตัวอย่าง OpenAPI spec ที่จะถูก generate อัตโนมัติ
openapi: 3.0.3
info:
  title: User Management API
  version: 2.0.0
paths:
  /api/v2/users:
    get:
      summary: Get all users
      responses:
        '200':
          description: List of users
```

---

## 🔧 2. Dependencies Setup

```kotlin
// build.gradle.kts
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
    implementation("org.springframework.boot:spring-boot-starter-security")
    implementation("org.springframework.boot:spring-boot-starter-validation")
    
    // springdoc-openapi
    implementation("org.springdoc:springdoc-openapi-starter-webmvc-ui:2.3.0")
    
    implementation("com.fasterxml.jackson.module:jackson-module-kotlin")
    implementation("org.jetbrains.kotlin:kotlin-reflect")
    
    runtimeOnly("com.h2database:h2")
    testImplementation("org.springframework.boot:spring-boot-starter-test")
}
```

```yaml
# application.yml
springdoc:
  api-docs:
    path: /api-docs
    enabled: true
  swagger-ui:
    path: /swagger-ui.html
    enabled: true
    operations-sorter: alpha
    tags-sorter: alpha
    display-request-duration: true
    filter: true
    try-it-out-enabled: true
  packages-to-scan: com.example.apidoc.controller
  paths-to-match: /api/**
  show-actuator: false

spring:
  application:
    name: api-documentation-demo
```

---

## ⚙️ 3. OpenAPI Configuration Bean

```kotlin
// config/OpenApiConfig.kt
package com.example.apidoc.config

import io.swagger.v3.oas.models.Components
import io.swagger.v3.oas.models.ExternalDocumentation
import io.swagger.v3.oas.models.OpenAPI
import io.swagger.v3.oas.models.info.Contact
import io.swagger.v3.oas.models.info.Info
import io.swagger.v3.oas.models.info.License
import io.swagger.v3.oas.models.security.SecurityRequirement
import io.swagger.v3.oas.models.security.SecurityScheme
import io.swagger.v3.oas.models.servers.Server
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration

@Configuration
class OpenApiConfig {

    @Bean
    fun customOpenAPI(): OpenAPI {
        return OpenAPI()
            .info(
                Info()
                    .title("User Management API")
                    .description("""
                        ## Overview
                        REST API สำหรับจัดการข้อมูล User
                        
                        ## Authentication
                        API นี้ใช้ JWT Bearer token สำหรับ authentication
                        1. เรียก POST /auth/login เพื่อรับ token
                        2. ใส่ token ใน Authorization header: `Bearer {token}`
                        
                        ## Rate Limiting
                        - Anonymous: 20 requests/minute
                        - Authenticated: 100 requests/minute
                        - Premium: 1000 requests/minute
                    """.trimIndent())
                    .version("2.0.0")
                    .contact(
                        Contact()
                            .name("API Support Team")
                            .email("api@example.com")
                            .url("https://support.example.com")
                    )
                    .license(
                        License()
                            .name("Apache 2.0")
                            .url("https://www.apache.org/licenses/LICENSE-2.0")
                    )
            )
            .servers(
                listOf(
                    Server().url("https://api.example.com").description("Production"),
                    Server().url("https://staging-api.example.com").description("Staging"),
                    Server().url("http://localhost:8080").description("Local Development")
                )
            )
            .externalDocs(
                ExternalDocumentation()
                    .description("Full API Documentation")
                    .url("https://docs.example.com")
            )
            .components(
                Components()
                    .addSecuritySchemes(
                        "bearerAuth",
                        SecurityScheme()
                            .type(SecurityScheme.Type.HTTP)
                            .scheme("bearer")
                            .bearerFormat("JWT")
                            .description("JWT token obtained from /auth/login")
                    )
                    .addSecuritySchemes(
                        "apiKey",
                        SecurityScheme()
                            .type(SecurityScheme.Type.APIKEY)
                            .`in`(SecurityScheme.In.HEADER)
                            .name("X-API-Key")
                            .description("API Key for service-to-service calls")
                    )
            )
            .addSecurityItem(SecurityRequirement().addList("bearerAuth"))
    }
}
```

---

## 📝 4. Documented DTOs ด้วย @Schema

```kotlin
// dto/UserDtos.kt
package com.example.apidoc.dto

import io.swagger.v3.oas.annotations.media.Schema
import jakarta.validation.constraints.*

@Schema(description = "Request body สำหรับสร้าง user ใหม่")
data class CreateUserRequest(

    @Schema(
        description = "ชื่อ",
        example = "John",
        minLength = 2,
        maxLength = 50,
        required = true
    )
    @field:NotBlank(message = "First name is required")
    @field:Size(min = 2, max = 50)
    val firstName: String,

    @Schema(
        description = "นามสกุล",
        example = "Doe",
        minLength = 2,
        maxLength = 50,
        required = true
    )
    @field:NotBlank(message = "Last name is required")
    @field:Size(min = 2, max = 50)
    val lastName: String,

    @Schema(
        description = "Email address (ต้องไม่ซ้ำในระบบ)",
        example = "john.doe@example.com",
        format = "email",
        required = true
    )
    @field:NotBlank
    @field:Email(message = "Invalid email format")
    val email: String,

    @Schema(
        description = "เบอร์โทรศัพท์ (optional)",
        example = "+66891234567",
        pattern = "^\\+?[1-9]\\d{1,14}$",
        required = false
    )
    val phoneNumber: String? = null,

    @Schema(
        description = "วันเกิด (ISO 8601 format)",
        example = "1990-01-15",
        format = "date",
        required = false
    )
    val birthDate: String? = null
)

@Schema(description = "Response body สำหรับข้อมูล User")
data class UserResponse(

    @Schema(description = "User ID", example = "42", accessMode = Schema.AccessMode.READ_ONLY)
    val id: Long,

    @Schema(description = "ชื่อ", example = "John")
    val firstName: String,

    @Schema(description = "นามสกุล", example = "Doe")
    val lastName: String,

    @Schema(description = "ชื่อเต็ม (computed)", example = "John Doe",
            accessMode = Schema.AccessMode.READ_ONLY)
    val fullName: String = "$firstName $lastName",

    @Schema(description = "Email", example = "john.doe@example.com")
    val email: String,

    @Schema(description = "เบอร์โทร", example = "+66891234567", nullable = true)
    val phoneNumber: String?,

    @Schema(description = "สถานะ active", example = "true")
    val active: Boolean = true,

    @Schema(description = "วันที่สร้าง (ISO 8601)", example = "2024-01-15T10:30:00Z",
            accessMode = Schema.AccessMode.READ_ONLY)
    val createdAt: String? = null
)

@Schema(description = "Response สำหรับ paginated data")
data class PagedResponse<T>(

    @Schema(description = "รายการข้อมูล")
    val content: List<T>,

    @Schema(description = "หน้าปัจจุบัน (เริ่มจาก 0)", example = "0")
    val page: Int,

    @Schema(description = "จำนวนต่อหน้า", example = "20")
    val size: Int,

    @Schema(description = "จำนวนทั้งหมด", example = "150")
    val totalElements: Long,

    @Schema(description = "จำนวนหน้าทั้งหมด", example = "8")
    val totalPages: Int,

    @Schema(description = "เป็นหน้าแรกหรือไม่", example = "true")
    val first: Boolean,

    @Schema(description = "เป็นหน้าสุดท้ายหรือไม่", example = "false")
    val last: Boolean
)

@Schema(description = "Standard error response")
data class ErrorResponse(

    @Schema(description = "HTTP status code", example = "400")
    val status: Int,

    @Schema(description = "Error type", example = "Bad Request")
    val error: String,

    @Schema(description = "รายละเอียด error", example = "Validation failed")
    val message: String,

    @Schema(description = "Request path", example = "/api/v2/users")
    val path: String,

    @Schema(description = "Timestamp", example = "2024-01-15T10:30:00Z")
    val timestamp: String,

    @Schema(description = "Field-specific errors (สำหรับ validation errors)")
    val fieldErrors: Map<String, String>? = null
)
```

---

## 🎮 5. Fully Documented Controller

```kotlin
// controller/UserController.kt
package com.example.apidoc.controller

import com.example.apidoc.dto.*
import com.example.apidoc.service.UserService
import io.swagger.v3.oas.annotations.Operation
import io.swagger.v3.oas.annotations.Parameter
import io.swagger.v3.oas.annotations.enums.ParameterIn
import io.swagger.v3.oas.annotations.media.Content
import io.swagger.v3.oas.annotations.media.Schema
import io.swagger.v3.oas.annotations.responses.ApiResponse
import io.swagger.v3.oas.annotations.responses.ApiResponses
import io.swagger.v3.oas.annotations.security.SecurityRequirement
import io.swagger.v3.oas.annotations.tags.Tag
import jakarta.validation.Valid
import org.springframework.http.HttpStatus
import org.springframework.http.ResponseEntity
import org.springframework.web.bind.annotation.*

@RestController
@RequestMapping("/api/v2/users")
@Tag(
    name = "User Management",
    description = "Endpoints สำหรับจัดการข้อมูล User: สร้าง, อ่าน, อัปเดต, ลบ"
)
class UserController(private val userService: UserService) {

    @Operation(
        summary = "ดึงรายการ Users ทั้งหมด",
        description = """
            ดึงรายการ user ทั้งหมดแบบ paginated
            
            **Permissions required:** `user:read`
            
            **Caching:** Response ถูก cache 60 วินาที
        """,
        security = [SecurityRequirement(name = "bearerAuth")]
    )
    @ApiResponses(
        ApiResponse(
            responseCode = "200",
            description = "รายการ user",
            content = [Content(
                mediaType = "application/json",
                schema = Schema(implementation = PagedResponse::class)
            )]
        ),
        ApiResponse(
            responseCode = "401",
            description = "ไม่ได้ authenticate",
            content = [Content(
                mediaType = "application/json",
                schema = Schema(implementation = ErrorResponse::class)
            )]
        ),
        ApiResponse(
            responseCode = "403",
            description = "ไม่มีสิทธิ์",
            content = [Content(schema = Schema(implementation = ErrorResponse::class))]
        )
    )
    @GetMapping
    fun getAllUsers(
        @Parameter(description = "หน้าที่ต้องการ (เริ่มจาก 0)", example = "0", `in` = ParameterIn.QUERY)
        @RequestParam(defaultValue = "0") page: Int,

        @Parameter(description = "จำนวนต่อหน้า", example = "20", `in` = ParameterIn.QUERY)
        @RequestParam(defaultValue = "20") size: Int,

        @Parameter(description = "เรียงตาม field", example = "lastName", `in` = ParameterIn.QUERY)
        @RequestParam(defaultValue = "id") sortBy: String,

        @Parameter(description = "ทิศทางการเรียง (ASC หรือ DESC)", example = "ASC", `in` = ParameterIn.QUERY)
        @RequestParam(defaultValue = "ASC") sortDir: String,

        @Parameter(description = "Filter เฉพาะ active users", `in` = ParameterIn.QUERY)
        @RequestParam(required = false) active: Boolean?
    ): ResponseEntity<PagedResponse<UserResponse>> {
        return ResponseEntity.ok(userService.getAllUsers(page, size, sortBy, sortDir, active))
    }

    @Operation(
        summary = "ดึงข้อมูล User ตาม ID",
        description = "ดึงข้อมูล user คนเดียวตาม ID",
        security = [SecurityRequirement(name = "bearerAuth")]
    )
    @ApiResponses(
        ApiResponse(responseCode = "200", description = "ข้อมูล user"),
        ApiResponse(responseCode = "404", description = "ไม่พบ user",
            content = [Content(schema = Schema(implementation = ErrorResponse::class))])
    )
    @GetMapping("/{id}")
    fun getUserById(
        @Parameter(description = "User ID", example = "42", required = true)
        @PathVariable id: Long
    ): ResponseEntity<UserResponse> {
        return ResponseEntity.ok(userService.getUserById(id))
    }

    @Operation(
        summary = "สร้าง User ใหม่",
        description = """
            สร้าง user ใหม่ในระบบ
            
            **Validation rules:**
            - firstName และ lastName ต้องมีความยาว 2-50 ตัวอักษร
            - email ต้องเป็น format ที่ถูกต้องและไม่ซ้ำในระบบ
            - phoneNumber เป็น optional
        """,
        security = [SecurityRequirement(name = "bearerAuth")]
    )
    @ApiResponses(
        ApiResponse(
            responseCode = "201",
            description = "สร้าง user สำเร็จ",
            content = [Content(schema = Schema(implementation = UserResponse::class))]
        ),
        ApiResponse(
            responseCode = "400",
            description = "Request body ไม่ถูกต้อง",
            content = [Content(schema = Schema(implementation = ErrorResponse::class))]
        ),
        ApiResponse(
            responseCode = "409",
            description = "Email นี้มีอยู่แล้ว",
            content = [Content(schema = Schema(implementation = ErrorResponse::class))]
        )
    )
    @PostMapping
    fun createUser(
        @io.swagger.v3.oas.annotations.parameters.RequestBody(
            description = "ข้อมูล user ที่ต้องการสร้าง",
            required = true,
            content = [Content(schema = Schema(implementation = CreateUserRequest::class))]
        )
        @Valid @RequestBody request: CreateUserRequest
    ): ResponseEntity<UserResponse> {
        return ResponseEntity.status(HttpStatus.CREATED)
            .body(userService.createUser(request))
    }

    @Operation(
        summary = "อัปเดตข้อมูล User",
        description = "อัปเดตข้อมูล user ที่มีอยู่แล้ว (partial update)",
        security = [SecurityRequirement(name = "bearerAuth")]
    )
    @PatchMapping("/{id}")
    fun updateUser(
        @Parameter(description = "User ID", required = true)
        @PathVariable id: Long,
        @Valid @RequestBody request: Map<String, Any>
    ): ResponseEntity<UserResponse> {
        return ResponseEntity.ok(userService.updateUser(id, request))
    }

    @Operation(
        summary = "ลบ User",
        description = "Soft delete user (ตั้งค่า active = false ไม่ลบจริง)",
        security = [SecurityRequirement(name = "bearerAuth")]
    )
    @ApiResponses(
        ApiResponse(responseCode = "204", description = "ลบสำเร็จ"),
        ApiResponse(responseCode = "404", description = "ไม่พบ user")
    )
    @DeleteMapping("/{id}")
    fun deleteUser(
        @Parameter(description = "User ID", required = true)
        @PathVariable id: Long
    ): ResponseEntity<Void> {
        userService.deleteUser(id)
        return ResponseEntity.noContent().build()
    }

    @Operation(
        summary = "ค้นหา User",
        description = "ค้นหา user ด้วย keyword (ค้นใน firstName, lastName, email)",
        security = [SecurityRequirement(name = "bearerAuth")]
    )
    @GetMapping("/search")
    fun searchUsers(
        @Parameter(description = "Keyword ที่ต้องการค้นหา", example = "john", required = true)
        @RequestParam query: String,

        @Parameter(description = "หน้า (เริ่มจาก 0)")
        @RequestParam(defaultValue = "0") page: Int,

        @Parameter(description = "จำนวนต่อหน้า")
        @RequestParam(defaultValue = "20") size: Int
    ): ResponseEntity<PagedResponse<UserResponse>> {
        return ResponseEntity.ok(userService.searchUsers(query, page, size))
    }
}
```

---

## 🔐 6. Auth Controller (Documented)

```kotlin
// controller/AuthController.kt
package com.example.apidoc.controller

import io.swagger.v3.oas.annotations.Operation
import io.swagger.v3.oas.annotations.tags.Tag
import io.swagger.v3.oas.annotations.security.SecurityRequirement
import org.springframework.http.ResponseEntity
import org.springframework.web.bind.annotation.*

@RestController
@RequestMapping("/auth")
@Tag(name = "Authentication", description = "Login, logout, token refresh")
class AuthController {

    data class LoginRequest(val username: String, val password: String)
    data class LoginResponse(val accessToken: String, val refreshToken: String, val expiresIn: Long)
    data class RefreshRequest(val refreshToken: String)

    @Operation(
        summary = "Login",
        description = "รับ JWT token สำหรับ authentication",
        security = []  // ไม่ต้องการ auth
    )
    @PostMapping("/login")
    fun login(@RequestBody request: LoginRequest): ResponseEntity<LoginResponse> {
        // Implementation
        return ResponseEntity.ok(
            LoginResponse(
                accessToken = "eyJhbGciOiJIUzI1NiJ9...",
                refreshToken = "eyJhbGciOiJIUzI1NiJ9...",
                expiresIn = 3600
            )
        )
    }

    @Operation(
        summary = "Refresh Token",
        description = "ขอ access token ใหม่ด้วย refresh token",
        security = []
    )
    @PostMapping("/refresh")
    fun refresh(@RequestBody request: RefreshRequest): ResponseEntity<LoginResponse> {
        return ResponseEntity.ok(
            LoginResponse(
                accessToken = "new_access_token",
                refreshToken = request.refreshToken,
                expiresIn = 3600
            )
        )
    }

    @Operation(
        summary = "Logout",
        security = [SecurityRequirement(name = "bearerAuth")]
    )
    @PostMapping("/logout")
    fun logout(): ResponseEntity<Void> {
        return ResponseEntity.noContent().build()
    }
}
```

---

## 📂 7. API Grouping

```kotlin
// config/OpenApiGroupConfig.kt
package com.example.apidoc.config

import org.springdoc.core.models.GroupedOpenApi
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration

@Configuration
class OpenApiGroupConfig {

    @Bean
    fun publicApi(): GroupedOpenApi {
        return GroupedOpenApi.builder()
            .group("public")
            .displayName("Public APIs (No Auth Required)")
            .pathsToMatch("/auth/**", "/api/public/**")
            .build()
    }

    @Bean
    fun userManagementApi(): GroupedOpenApi {
        return GroupedOpenApi.builder()
            .group("user-management")
            .displayName("User Management APIs")
            .pathsToMatch("/api/v2/users/**")
            .packagesToScan("com.example.apidoc.controller")
            .build()
    }

    @Bean
    fun adminApi(): GroupedOpenApi {
        return GroupedOpenApi.builder()
            .group("admin")
            .displayName("Admin APIs")
            .pathsToMatch("/api/admin/**")
            .build()
    }

    @Bean
    fun allApis(): GroupedOpenApi {
        return GroupedOpenApi.builder()
            .group("all")
            .displayName("All APIs")
            .pathsToMatch("/api/**", "/auth/**")
            .build()
    }
}
```

---

## 🎨 8. Custom Swagger UI Configuration

```kotlin
// config/SwaggerUiConfig.kt
package com.example.apidoc.config

import org.springframework.context.annotation.Configuration
import org.springframework.web.servlet.config.annotation.ResourceHandlerRegistry
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer

@Configuration
class SwaggerUiConfig : WebMvcConfigurer {

    override fun addResourceHandlers(registry: ResourceHandlerRegistry) {
        registry
            .addResourceHandler("/swagger-ui/**")
            .addResourceLocations("classpath:/META-INF/resources/webjars/swagger-ui/")
    }
}
```

```yaml
# application.yml (เพิ่มเติม)
springdoc:
  swagger-ui:
    # Custom UI settings
    doc-expansion: none        # ปิด expand ทุก endpoint โดย default
    deep-linking: true         # รองรับ URL deep linking
    display-operation-id: false
    default-models-expand-depth: 1
    default-model-rendering: model
    show-extensions: true
    show-common-extensions: true
    # ปรับ layout
    layout: BaseLayout
    # เปิด persistent authorization (remember token)
    persist-authorization: true
```

---

## 🧪 9. Testing Documentation

```kotlin
// test/OpenApiTest.kt
package com.example.apidoc

import org.junit.jupiter.api.Test
import org.springframework.beans.factory.annotation.Autowired
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc
import org.springframework.boot.test.context.SpringBootTest
import org.springframework.test.web.servlet.MockMvc
import org.springframework.test.web.servlet.get

@SpringBootTest
@AutoConfigureMockMvc
class OpenApiTest {

    @Autowired
    private lateinit var mockMvc: MockMvc

    @Test
    fun `OpenAPI spec should be accessible`() {
        mockMvc.get("/api-docs")
            .andExpect {
                status { isOk() }
                content { contentType("application/json") }
                jsonPath("$.openapi") { value("3.0.1") }
                jsonPath("$.info.title") { value("User Management API") }
            }
    }

    @Test
    fun `Swagger UI should be accessible`() {
        mockMvc.get("/swagger-ui/index.html")
            .andExpect {
                status { isOk() }
            }
    }

    @Test
    fun `User endpoints should be documented`() {
        mockMvc.get("/api-docs")
            .andExpect {
                jsonPath("$.paths./api/v2/users") { exists() }
                jsonPath("$.paths./api/v2/users.get") { exists() }
                jsonPath("$.paths./api/v2/users.post") { exists() }
                jsonPath("$.paths./api/v2/users/{id}.get") { exists() }
                jsonPath("$.paths./api/v2/users/{id}.delete") { exists() }
            }
    }

    @Test
    fun `Security schemes should be configured`() {
        mockMvc.get("/api-docs")
            .andExpect {
                jsonPath("$.components.securitySchemes.bearerAuth") { exists() }
                jsonPath("$.components.securitySchemes.bearerAuth.type") { value("http") }
                jsonPath("$.components.securitySchemes.bearerAuth.scheme") { value("bearer") }
            }
    }
}
```

---

## 📊 สรุปเนื้อหา

| Annotation | ใช้กับ | ประโยชน์ |
|-----------|--------|---------|
| @Tag | Class | จัดกลุ่ม endpoints |
| @Operation | Method | อธิบาย endpoint |
| @ApiResponse | Method | อธิบาย response codes |
| @Parameter | Parameter | อธิบาย query/path param |
| @RequestBody | Parameter | อธิบาย request body |
| @Schema | DTO Class/Field | อธิบาย data model |

### Swagger UI Endpoints:

| URL | ประโยชน์ |
|-----|---------|
| `/swagger-ui.html` | Swagger UI |
| `/api-docs` | OpenAPI JSON spec |
| `/api-docs.yaml` | OpenAPI YAML spec |

### Best Practices:

- ใส่ description ทุก endpoint
- ระบุ example values ใน @Schema
- Document ทุก error response code
- แบ่งกลุ่ม endpoints ด้วย @Tag
- ตั้งค่า security scheme อย่างชัดเจน
- Test documentation ให้ถูกต้อง

---

*Part 44/100+ | Kotlin & Spring Boot Complete Course*
