# Part 85: API Design Best Practices
## ออกแบบ RESTful API ที่สมบูรณ์แบบ

---

## 🎯 เป้าหมายของ Part นี้

- เข้าใจหลักการ RESTful API Design
- Naming conventions ที่ถูกต้อง
- Versioning strategies
- Error response standards
- API documentation

---

## 📖 1. REST Principles

REST (Representational State Transfer) มี 6 constraints:

```
1. Client-Server separation
2. Stateless
3. Cacheable
4. Uniform Interface
5. Layered System
6. Code on Demand (optional)
```

### Resource-Oriented Design

```
ผิด (Action-based):
GET /getUser?id=123
POST /createOrder
PUT /updateProduct?id=456

ถูก (Resource-based):
GET /users/123
POST /orders
PUT /products/456
```

---

## 📝 2. Naming Conventions

### URL Naming Rules

```
✅ ถูกต้อง:
GET    /users              # list users
GET    /users/123          # get user
POST   /users              # create user
PUT    /users/123          # full update
PATCH  /users/123          # partial update
DELETE /users/123          # delete user

✅ Nested resources:
GET    /users/123/orders          # orders ของ user 123
GET    /users/123/orders/456      # order 456 ของ user 123
POST   /users/123/orders          # create order สำหรับ user 123

✅ Use nouns, not verbs:
/users     (ไม่ใช่ /getUsers)
/orders    (ไม่ใช่ /createOrder)
/products  (ไม่ใช่ /deleteProduct)

✅ Plural nouns:
/users     (ไม่ใช่ /user)
/products  (ไม่ใช่ /product)
/orders    (ไม่ใช่ /order)

✅ Lowercase, hyphen-separated:
/user-profiles    (ไม่ใช่ /userProfiles หรือ /UserProfiles)
/order-items      (ไม่ใช่ /orderItems)
```

### HTTP Methods

```kotlin
// ✅ ใช้ HTTP methods ตาม semantics
@RestController
@RequestMapping("/api/v1/products")
class ProductController(private val productService: ProductService) {

    @GetMapping           // Safe, Idempotent - list
    fun list(
        @RequestParam(defaultValue = "0") page: Int,
        @RequestParam(defaultValue = "20") size: Int,
        @RequestParam(required = false) search: String?
    ): ResponseEntity<PageResponse<ProductDto>> {
        return ResponseEntity.ok(productService.list(page, size, search))
    }

    @GetMapping("/{id}")  // Safe, Idempotent - get one
    fun getById(@PathVariable id: Long): ResponseEntity<ProductDto> {
        return ResponseEntity.ok(productService.getById(id))
    }

    @PostMapping          // Not safe, Not idempotent - create
    fun create(@Valid @RequestBody request: CreateProductRequest): ResponseEntity<ProductDto> {
        val product = productService.create(request)
        val location = URI.create("/api/v1/products/${product.id}")
        return ResponseEntity.created(location).body(product)
    }

    @PutMapping("/{id}")  // Not safe, Idempotent - full replace
    fun update(
        @PathVariable id: Long,
        @Valid @RequestBody request: UpdateProductRequest
    ): ResponseEntity<ProductDto> {
        return ResponseEntity.ok(productService.update(id, request))
    }

    @PatchMapping("/{id}") // Not safe, Not idempotent - partial update
    fun partialUpdate(
        @PathVariable id: Long,
        @RequestBody patch: Map<String, Any>
    ): ResponseEntity<ProductDto> {
        return ResponseEntity.ok(productService.partialUpdate(id, patch))
    }

    @DeleteMapping("/{id}") // Not safe, Idempotent - delete
    fun delete(@PathVariable id: Long): ResponseEntity<Void> {
        productService.delete(id)
        return ResponseEntity.noContent().build()
    }
}
```

---

## 🔢 3. HTTP Status Codes

```kotlin
// ✅ ใช้ Status Codes ที่ถูกต้อง

// 2xx Success
// 200 OK           - GET, PUT, PATCH success
// 201 Created      - POST success (เพิ่ม Location header)
// 204 No Content   - DELETE success
// 206 Partial Content - pagination

// 4xx Client Errors
// 400 Bad Request          - validation error
// 401 Unauthorized         - not authenticated
// 403 Forbidden            - authenticated but no permission
// 404 Not Found            - resource not found
// 405 Method Not Allowed   - wrong HTTP method
// 409 Conflict             - duplicate resource
// 422 Unprocessable Entity - business logic error
// 429 Too Many Requests    - rate limit exceeded

// 5xx Server Errors
// 500 Internal Server Error - unexpected server error
// 502 Bad Gateway           - upstream service error
// 503 Service Unavailable   - server overloaded/maintenance
// 504 Gateway Timeout       - upstream timeout

@RestController
@RequestMapping("/api/v1/users")
class UserController(private val userService: UserService) {

    @PostMapping
    fun create(@Valid @RequestBody request: CreateUserRequest): ResponseEntity<UserDto> {
        return try {
            val user = userService.create(request)
            val uri = URI.create("/api/v1/users/${user.id}")
            ResponseEntity.created(uri).body(user)    // 201
        } catch (e: DuplicateEmailException) {
            ResponseEntity.status(HttpStatus.CONFLICT).build()  // 409
        }
    }

    @GetMapping("/{id}")
    fun getById(@PathVariable id: Long): ResponseEntity<UserDto> {
        return try {
            ResponseEntity.ok(userService.getById(id))  // 200
        } catch (e: UserNotFoundException) {
            ResponseEntity.notFound().build()  // 404
        }
    }
}
```

---

## ❌ 4. Error Response Standards

### Error Response Format

```kotlin
// src/main/kotlin/com/example/dto/ErrorResponse.kt
package com.example.dto

import java.time.Instant

data class ErrorResponse(
    val status: Int,
    val error: String,
    val message: String,
    val path: String,
    val timestamp: Instant = Instant.now(),
    val errors: List<FieldError>? = null,
    val traceId: String? = null
)

data class FieldError(
    val field: String,
    val message: String,
    val rejectedValue: Any? = null
)
```

### Global Exception Handler

```kotlin
// src/main/kotlin/com/example/exception/GlobalExceptionHandler.kt
package com.example.exception

import com.example.dto.ErrorResponse
import com.example.dto.FieldError
import jakarta.servlet.http.HttpServletRequest
import org.springframework.http.HttpStatus
import org.springframework.http.ResponseEntity
import org.springframework.validation.BindingResult
import org.springframework.web.bind.MethodArgumentNotValidException
import org.springframework.web.bind.annotation.ExceptionHandler
import org.springframework.web.bind.annotation.RestControllerAdvice
import java.time.Instant

@RestControllerAdvice
class GlobalExceptionHandler {

    @ExceptionHandler(MethodArgumentNotValidException::class)
    fun handleValidation(
        ex: MethodArgumentNotValidException,
        request: HttpServletRequest
    ): ResponseEntity<ErrorResponse> {
        val fieldErrors = ex.bindingResult.fieldErrors.map { error ->
            FieldError(
                field = error.field,
                message = error.defaultMessage ?: "Invalid value",
                rejectedValue = error.rejectedValue
            )
        }

        val response = ErrorResponse(
            status = 400,
            error = "Validation Failed",
            message = "Request contains invalid fields",
            path = request.requestURI,
            errors = fieldErrors
        )
        return ResponseEntity.badRequest().body(response)
    }

    @ExceptionHandler(ResourceNotFoundException::class)
    fun handleNotFound(
        ex: ResourceNotFoundException,
        request: HttpServletRequest
    ): ResponseEntity<ErrorResponse> {
        val response = ErrorResponse(
            status = 404,
            error = "Not Found",
            message = ex.message ?: "Resource not found",
            path = request.requestURI
        )
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(response)
    }

    @ExceptionHandler(DuplicateResourceException::class)
    fun handleConflict(
        ex: DuplicateResourceException,
        request: HttpServletRequest
    ): ResponseEntity<ErrorResponse> {
        val response = ErrorResponse(
            status = 409,
            error = "Conflict",
            message = ex.message ?: "Resource already exists",
            path = request.requestURI
        )
        return ResponseEntity.status(HttpStatus.CONFLICT).body(response)
    }

    @ExceptionHandler(Exception::class)
    fun handleGeneral(
        ex: Exception,
        request: HttpServletRequest
    ): ResponseEntity<ErrorResponse> {
        // Log ข้อมูลจริง แต่ไม่ส่ง internal error details ให้ client
        val response = ErrorResponse(
            status = 500,
            error = "Internal Server Error",
            message = "An unexpected error occurred",
            path = request.requestURI
        )
        return ResponseEntity.internalServerError().body(response)
    }
}
```

### ตัวอย่าง Error Response

```json
// 400 Validation Error
{
  "status": 400,
  "error": "Validation Failed",
  "message": "Request contains invalid fields",
  "path": "/api/v1/users",
  "timestamp": "2024-01-15T10:30:00Z",
  "errors": [
    {
      "field": "email",
      "message": "must be a valid email address",
      "rejectedValue": "not-an-email"
    },
    {
      "field": "password",
      "message": "must be at least 8 characters",
      "rejectedValue": "123"
    }
  ]
}

// 404 Not Found
{
  "status": 404,
  "error": "Not Found",
  "message": "User with id 999 not found",
  "path": "/api/v1/users/999",
  "timestamp": "2024-01-15T10:30:00Z"
}
```

---

## 📊 5. Pagination Standard

```kotlin
// src/main/kotlin/com/example/dto/PageResponse.kt
data class PageResponse<T>(
    val content: List<T>,
    val page: Int,
    val size: Int,
    val totalElements: Long,
    val totalPages: Int,
    val first: Boolean,
    val last: Boolean,
    val links: PageLinks
)

data class PageLinks(
    val self: String,
    val first: String,
    val prev: String?,
    val next: String?,
    val last: String
)

// ✅ ตัวอย่าง Response
// GET /api/v1/products?page=2&size=10
{
    "content": [...],
    "page": 2,
    "size": 10,
    "totalElements": 150,
    "totalPages": 15,
    "first": false,
    "last": false,
    "links": {
        "self": "/api/v1/products?page=2&size=10",
        "first": "/api/v1/products?page=0&size=10",
        "prev": "/api/v1/products?page=1&size=10",
        "next": "/api/v1/products?page=3&size=10",
        "last": "/api/v1/products?page=14&size=10"
    }
}
```

---

## 🔢 6. API Versioning

```kotlin
// Strategy 1: URL Path Versioning (แนะนำ)
@RestController
@RequestMapping("/api/v1/users")
class UserV1Controller { ... }

@RestController
@RequestMapping("/api/v2/users")
class UserV2Controller { ... }

// Strategy 2: Header Versioning
@GetMapping(
    "/users/{id}",
    headers = ["API-Version=1"]
)
fun getUserV1(@PathVariable id: Long): UserV1Dto { ... }

@GetMapping(
    "/users/{id}",
    headers = ["API-Version=2"]
)
fun getUserV2(@PathVariable id: Long): UserV2Dto { ... }

// Strategy 3: Accept Header Versioning
@GetMapping(
    "/users/{id}",
    produces = ["application/vnd.myapp.v1+json"]
)
fun getUserV1(@PathVariable id: Long): UserV1Dto { ... }

@GetMapping(
    "/users/{id}",
    produces = ["application/vnd.myapp.v2+json"]
)
fun getUserV2(@PathVariable id: Long): UserV2Dto { ... }
```

---

## 📚 7. API Documentation กับ OpenAPI/Swagger

```kotlin
// build.gradle.kts
dependencies {
    implementation("org.springdoc:springdoc-openapi-starter-webmvc-ui:2.3.0")
}
```

```kotlin
// src/main/kotlin/com/example/config/OpenApiConfig.kt
package com.example.config

import io.swagger.v3.oas.models.OpenAPI
import io.swagger.v3.oas.models.info.Contact
import io.swagger.v3.oas.models.info.Info
import io.swagger.v3.oas.models.info.License
import io.swagger.v3.oas.models.security.SecurityRequirement
import io.swagger.v3.oas.models.security.SecurityScheme
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration

@Configuration
class OpenApiConfig {

    @Bean
    fun openAPI(): OpenAPI = OpenAPI()
        .info(
            Info()
                .title("Product API")
                .description("Complete API for managing products")
                .version("1.0.0")
                .contact(
                    Contact()
                        .name("API Team")
                        .email("api@example.com")
                )
                .license(
                    License()
                        .name("MIT")
                        .url("https://opensource.org/licenses/MIT")
                )
        )
        .addSecurityItem(SecurityRequirement().addList("Bearer Auth"))
        .components(
            io.swagger.v3.oas.models.Components()
                .addSecuritySchemes(
                    "Bearer Auth",
                    SecurityScheme()
                        .type(SecurityScheme.Type.HTTP)
                        .scheme("bearer")
                        .bearerFormat("JWT")
                )
        )
}
```

```kotlin
// ใช้ annotations บน Controller
@Operation(
    summary = "Get product by ID",
    description = "Returns a single product by its ID"
)
@ApiResponses(
    ApiResponse(responseCode = "200", description = "Product found"),
    ApiResponse(responseCode = "404", description = "Product not found"),
    ApiResponse(responseCode = "401", description = "Unauthorized")
)
@GetMapping("/{id}")
fun getById(
    @Parameter(description = "Product ID", required = true)
    @PathVariable id: Long
): ResponseEntity<ProductDto> {
    return ResponseEntity.ok(productService.getById(id))
}
```

---

## 📋 สรุป

| หัวข้อ | Best Practice |
|--------|--------------|
| URL Design | Nouns, plural, lowercase, hyphen-separated |
| HTTP Methods | GET read, POST create, PUT replace, PATCH update, DELETE remove |
| Status Codes | ใช้ตาม semantics อย่างถูกต้อง |
| Error Format | Consistent JSON error response |
| Versioning | URL path versioning แนะนำที่สุด |
| Documentation | OpenAPI/Swagger สำคัญมาก |
| Pagination | Metadata + links |

---

*Part 85/100+ | Kotlin & Spring Boot Complete Course*
