# Part 26: Exception Handling ขั้นสูง
## Advanced Error Handling ใน Spring Boot + Kotlin

---

## 🎯 เป้าหมายของ Part นี้

- ออกแบบ Exception hierarchy
- RFC 7807 Problem Details
- Error codes ที่ชัดเจน
- Logging ที่ถูกต้อง
- ตัวอย่าง: Production-ready error handling

---

## 🏗️ 1. Exception Hierarchy

```kotlin
// exceptions/AppExceptions.kt

// Base exception
abstract class AppException(
    message: String,
    val errorCode: String,
    val httpStatus: Int,
    cause: Throwable? = null
) : RuntimeException(message, cause)

// 4xx Client Errors
class NotFoundException(
    message: String,
    val resourceType: String,
    val resourceId: Any,
    errorCode: String = "RESOURCE_NOT_FOUND"
) : AppException(message, errorCode, 404)

class BadRequestException(
    message: String,
    errorCode: String = "BAD_REQUEST"
) : AppException(message, errorCode, 400)

class ConflictException(
    message: String,
    errorCode: String = "CONFLICT"
) : AppException(message, errorCode, 409)

class ForbiddenException(
    message: String,
    errorCode: String = "FORBIDDEN"
) : AppException(message, errorCode, 403)

class UnauthorizedException(
    message: String,
    errorCode: String = "UNAUTHORIZED"
) : AppException(message, errorCode, 401)

// 5xx Server Errors
class InternalException(
    message: String,
    errorCode: String = "INTERNAL_ERROR",
    cause: Throwable? = null
) : AppException(message, errorCode, 500, cause)

class ServiceUnavailableException(
    message: String,
    val retryAfterSeconds: Int? = null
) : AppException(message, "SERVICE_UNAVAILABLE", 503)
```

---

## 📋 2. RFC 7807 Problem Details

RFC 7807 เป็นมาตรฐาน HTTP Error response format

```kotlin
// dto/ProblemDetails.kt
import com.fasterxml.jackson.annotation.JsonInclude
import java.net.URI

@JsonInclude(JsonInclude.Include.NON_NULL)
data class ProblemDetails(
    val type: String = "about:blank",       // URI ของ error type
    val title: String,                       // Human-readable title
    val status: Int,                         // HTTP status code
    val detail: String? = null,             // ลายละเอียดเพิ่มเติม
    val instance: String? = null,           // URI ของ request นี้
    val timestamp: String = java.time.LocalDateTime.now().toString(),
    
    // Extensions (ใส่ข้อมูลเพิ่มเติมได้)
    val errorCode: String? = null,
    val errors: Map<String, String>? = null,  // Validation errors
    val traceId: String? = null
)
```

---

## 🚨 3. Global Exception Handler

```kotlin
// exception/GlobalExceptionHandler.kt
import org.slf4j.LoggerFactory
import org.springframework.http.HttpHeaders
import org.springframework.http.HttpStatus
import org.springframework.http.ResponseEntity
import org.springframework.web.bind.MethodArgumentNotValidException
import org.springframework.web.bind.annotation.ExceptionHandler
import org.springframework.web.bind.annotation.RestControllerAdvice
import org.springframework.web.context.request.WebRequest
import java.util.UUID

@RestControllerAdvice
class GlobalExceptionHandler {
    
    private val log = LoggerFactory.getLogger(GlobalExceptionHandler::class.java)
    
    // =============== App Exceptions ===============
    
    @ExceptionHandler(NotFoundException::class)
    fun handleNotFound(ex: NotFoundException, request: WebRequest): ResponseEntity<ProblemDetails> {
        log.warn("Resource not found: {} {} - {}", ex.resourceType, ex.resourceId, ex.message)
        
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(
            ProblemDetails(
                type = "/errors/not-found",
                title = "Resource Not Found",
                status = 404,
                detail = ex.message,
                instance = request.getDescription(false),
                errorCode = ex.errorCode
            )
        )
    }
    
    @ExceptionHandler(BadRequestException::class)
    fun handleBadRequest(ex: BadRequestException, request: WebRequest): ResponseEntity<ProblemDetails> {
        log.warn("Bad request: {}", ex.message)
        
        return ResponseEntity.status(HttpStatus.BAD_REQUEST).body(
            ProblemDetails(
                type = "/errors/bad-request",
                title = "Bad Request",
                status = 400,
                detail = ex.message,
                instance = request.getDescription(false),
                errorCode = ex.errorCode
            )
        )
    }
    
    @ExceptionHandler(ConflictException::class)
    fun handleConflict(ex: ConflictException, request: WebRequest): ResponseEntity<ProblemDetails> {
        log.warn("Conflict: {}", ex.message)
        
        return ResponseEntity.status(HttpStatus.CONFLICT).body(
            ProblemDetails(
                type = "/errors/conflict",
                title = "Conflict",
                status = 409,
                detail = ex.message,
                errorCode = ex.errorCode
            )
        )
    }
    
    // =============== Validation ===============
    
    @ExceptionHandler(MethodArgumentNotValidException::class)
    fun handleValidation(ex: MethodArgumentNotValidException): ResponseEntity<ProblemDetails> {
        val errors = ex.bindingResult.fieldErrors
            .associate { it.field to (it.defaultMessage ?: "Invalid value") }
        
        return ResponseEntity.badRequest().body(
            ProblemDetails(
                type = "/errors/validation-failed",
                title = "Validation Failed",
                status = 400,
                detail = "One or more fields failed validation",
                errorCode = "VALIDATION_ERROR",
                errors = errors
            )
        )
    }
    
    // =============== Generic ===============
    
    @ExceptionHandler(Exception::class)
    fun handleGeneral(ex: Exception, request: WebRequest): ResponseEntity<ProblemDetails> {
        val traceId = UUID.randomUUID().toString()
        log.error("Unexpected error [traceId={}]: {}", traceId, ex.message, ex)
        
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body(
            ProblemDetails(
                type = "/errors/internal-error",
                title = "Internal Server Error",
                status = 500,
                detail = "An unexpected error occurred. Please contact support.",
                errorCode = "INTERNAL_ERROR",
                traceId = traceId
            )
        )
    }
}
```

---

## 📊 4. Error Codes ที่ชัดเจน

```kotlin
// constants/ErrorCodes.kt
object ErrorCodes {
    // User domain
    const val USER_NOT_FOUND = "USER_NOT_FOUND"
    const val USER_EMAIL_DUPLICATE = "USER_EMAIL_DUPLICATE"
    const val USER_INACTIVE = "USER_INACTIVE"
    const val USER_PASSWORD_WRONG = "USER_PASSWORD_WRONG"
    
    // Product domain
    const val PRODUCT_NOT_FOUND = "PRODUCT_NOT_FOUND"
    const val PRODUCT_OUT_OF_STOCK = "PRODUCT_OUT_OF_STOCK"
    const val PRODUCT_SKU_DUPLICATE = "PRODUCT_SKU_DUPLICATE"
    
    // Order domain
    const val ORDER_NOT_FOUND = "ORDER_NOT_FOUND"
    const val ORDER_ALREADY_CANCELLED = "ORDER_ALREADY_CANCELLED"
    const val ORDER_CANNOT_MODIFY = "ORDER_CANNOT_MODIFY"
    
    // Auth domain
    const val AUTH_TOKEN_EXPIRED = "AUTH_TOKEN_EXPIRED"
    const val AUTH_TOKEN_INVALID = "AUTH_TOKEN_INVALID"
    const val AUTH_INSUFFICIENT_PERMISSION = "AUTH_INSUFFICIENT_PERMISSION"
    
    // Validation
    const val VALIDATION_ERROR = "VALIDATION_ERROR"
    const val BAD_REQUEST = "BAD_REQUEST"
}

// ใช้งาน
@Service
class UserService(private val userRepo: UserRepository) {
    
    fun getUser(id: Long): User {
        return userRepo.findById(id).orElseThrow {
            NotFoundException(
                message = "User with id=$id not found",
                resourceType = "User",
                resourceId = id,
                errorCode = ErrorCodes.USER_NOT_FOUND
            )
        }
    }
    
    fun createUser(email: String, name: String): User {
        if (userRepo.existsByEmail(email)) {
            throw ConflictException(
                message = "Email '$email' is already registered",
                errorCode = ErrorCodes.USER_EMAIL_DUPLICATE
            )
        }
        return userRepo.save(User(name = name, email = email))
    }
}
```

---

## 📝 5. ตัวอย่าง Response ต่างๆ

```json
// 404 Not Found
{
    "type": "/errors/not-found",
    "title": "Resource Not Found",
    "status": 404,
    "detail": "User with id=123 not found",
    "instance": "uri=/api/users/123",
    "errorCode": "USER_NOT_FOUND",
    "timestamp": "2026-10-02T10:00:00"
}

// 409 Conflict
{
    "type": "/errors/conflict",
    "title": "Conflict",
    "status": 409,
    "detail": "Email 'alice@example.com' is already registered",
    "errorCode": "USER_EMAIL_DUPLICATE",
    "timestamp": "2026-10-02T10:00:00"
}

// 400 Validation Failed
{
    "type": "/errors/validation-failed",
    "title": "Validation Failed",
    "status": 400,
    "detail": "One or more fields failed validation",
    "errorCode": "VALIDATION_ERROR",
    "errors": {
        "email": "Invalid email format",
        "age": "Must be at least 13 years old"
    },
    "timestamp": "2026-10-02T10:00:00"
}

// 500 Internal Error
{
    "type": "/errors/internal-error",
    "title": "Internal Server Error",
    "status": 500,
    "detail": "An unexpected error occurred. Please contact support.",
    "errorCode": "INTERNAL_ERROR",
    "traceId": "550e8400-e29b-41d4-a716-446655440000",
    "timestamp": "2026-10-02T10:00:00"
}
```

---

## 🧪 6. Testing Exception Handling

```kotlin
@WebMvcTest(UserController::class)
class ExceptionHandlingTest {
    
    @Autowired private lateinit var mockMvc: MockMvc
    @MockBean private lateinit var userService: UserService
    
    @Test
    fun `should return 404 with problem details for missing user`() {
        every { userService.getUser(99L) } throws NotFoundException(
            message = "User with id=99 not found",
            resourceType = "User",
            resourceId = 99L,
            errorCode = "USER_NOT_FOUND"
        )
        
        mockMvc.get("/api/users/99")
            .andExpect {
                status { isNotFound() }
                jsonPath("$.status") { value(404) }
                jsonPath("$.errorCode") { value("USER_NOT_FOUND") }
                jsonPath("$.detail") { value("User with id=99 not found") }
            }
    }
    
    @Test
    fun `should return 409 for duplicate email`() {
        every { userService.createUser(any()) } throws ConflictException(
            message = "Email already exists",
            errorCode = "USER_EMAIL_DUPLICATE"
        )
        
        mockMvc.post("/api/users") {
            contentType = MediaType.APPLICATION_JSON
            content = """{"name":"Alice","email":"dup@example.com","password":"Pass123","age":25}"""
        }.andExpect {
            status { isConflict() }
            jsonPath("$.errorCode") { value("USER_EMAIL_DUPLICATE") }
        }
    }
}
```

---

## 📝 สรุป Part 26

| แนวคิด | รายละเอียด |
|--------|-----------|
| Exception hierarchy | `AppException` → domain-specific |
| RFC 7807 | `ProblemDetails` response format |
| Error codes | Constants สำหรับ client |
| `@RestControllerAdvice` | Central error handling |
| Logging | log warning/error ตาม severity |
| traceId | ติดตาม error ใน production |

---

## ➡️ ถัดไป: Part 27 - Spring Security

---
*Part 26/100+ | Kotlin & Spring Boot Complete Course*
