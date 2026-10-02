# Part 25: Validation
## Bean Validation กับ Spring Boot + Kotlin

---

## 🎯 เป้าหมายของ Part นี้

- ใช้ Bean Validation (@Valid, @NotBlank, @Size ฯลฯ)
- สร้าง Custom Validators
- จัดการ Validation Errors
- Validation ใน Service Layer
- ตัวอย่าง: Registration Form Validation

---

## 📦 1. Dependencies

```kotlin
// build.gradle.kts
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-validation")
    // ครอบคลุม hibernate-validator ซึ่ง implement Bean Validation 3.0
}
```

---

## 📋 2. Standard Validation Annotations

```kotlin
import jakarta.validation.constraints.*
import java.time.LocalDate

data class CreateUserRequest(
    
    @field:NotBlank(message = "Name is required")
    @field:Size(min = 2, max = 100, message = "Name must be 2-100 chars")
    val name: String,
    
    @field:NotBlank(message = "Email is required")
    @field:Email(message = "Invalid email format")
    val email: String,
    
    @field:NotBlank(message = "Password is required")
    @field:Size(min = 8, message = "Password must be at least 8 characters")
    @field:Pattern(
        regexp = "^(?=.*[A-Z])(?=.*[0-9]).+\$",
        message = "Password must contain uppercase and number"
    )
    val password: String,
    
    @field:Min(value = 13, message = "Must be at least 13 years old")
    @field:Max(value = 120, message = "Invalid age")
    val age: Int,
    
    @field:NotNull(message = "Birth date is required")
    @field:Past(message = "Birth date must be in the past")
    val birthDate: LocalDate?,
    
    @field:Size(max = 500, message = "Bio too long")
    val bio: String? = null,
    
    @field:NotEmpty(message = "At least one interest required")
    val interests: List<String> = emptyList()
)
```

### Annotation ที่มีให้ใช้

```
// String
@NotNull       - ห้าม null
@NotEmpty      - ห้าม null และ empty
@NotBlank      - ห้าม null, empty, และ whitespace-only
@Size          - min/max length
@Email         - valid email
@Pattern       - regex pattern
@URL           - valid URL

// Number
@Min           - minimum value
@Max           - maximum value
@Positive      - > 0
@PositiveOrZero - >= 0
@Negative      - < 0
@NegativeOrZero - <= 0
@Digits        - integer/fraction digits

// Date
@Past          - must be in the past
@PastOrPresent - past or now
@Future        - must be in the future
@FutureOrPresent - future or now

// Collection
@NotEmpty      - not null and not empty
@Size          - min/max size
```

---

## 🎮 3. ใช้ @Valid ใน Controller

```kotlin
@RestController
@RequestMapping("/api/users")
class UserController(private val userService: UserService) {
    
    @PostMapping
    fun createUser(
        @Valid @RequestBody request: CreateUserRequest,  // @Valid triggers validation
        bindingResult: BindingResult? = null             // Optional: manual check
    ): ResponseEntity<UserResponse> {
        val user = userService.createUser(request)
        return ResponseEntity.status(HttpStatus.CREATED).body(user)
    }
    
    @PutMapping("/{id}")
    fun updateUser(
        @PathVariable id: Long,
        @Valid @RequestBody request: UpdateUserRequest
    ): ResponseEntity<UserResponse> {
        return ResponseEntity.ok(userService.updateUser(id, request))
    }
}
```

---

## 🚨 4. Handling Validation Errors

```kotlin
// exception/ValidationExceptionHandler.kt
import org.springframework.http.HttpStatus
import org.springframework.http.ResponseEntity
import org.springframework.validation.FieldError
import org.springframework.web.bind.MethodArgumentNotValidException
import org.springframework.web.bind.annotation.ExceptionHandler
import org.springframework.web.bind.annotation.RestControllerAdvice
import java.time.LocalDateTime

data class ValidationErrorResponse(
    val timestamp: LocalDateTime = LocalDateTime.now(),
    val status: Int = 400,
    val error: String = "Validation Failed",
    val message: String = "Input validation failed",
    val errors: Map<String, String>
)

@RestControllerAdvice
class GlobalExceptionHandler {
    
    // Handles @Valid failures on @RequestBody
    @ExceptionHandler(MethodArgumentNotValidException::class)
    fun handleValidation(ex: MethodArgumentNotValidException): ResponseEntity<ValidationErrorResponse> {
        val errors = ex.bindingResult.fieldErrors
            .associate { e: FieldError -> e.field to (e.defaultMessage ?: "Invalid") }
        
        return ResponseEntity.badRequest().body(
            ValidationErrorResponse(errors = errors)
        )
    }
    
    // Handles @Validated failures on @RequestParam, @PathVariable
    @ExceptionHandler(jakarta.validation.ConstraintViolationException::class)
    fun handleConstraintViolation(
        ex: jakarta.validation.ConstraintViolationException
    ): ResponseEntity<ValidationErrorResponse> {
        val errors = ex.constraintViolations
            .associate { v -> v.propertyPath.toString() to v.message }
        
        return ResponseEntity.badRequest().body(
            ValidationErrorResponse(errors = errors)
        )
    }
}
```

**ตัวอย่าง response เมื่อ validation ล้มเหลว:**
```json
{
    "timestamp": "2026-10-02T10:00:00",
    "status": 400,
    "error": "Validation Failed",
    "message": "Input validation failed",
    "errors": {
        "email": "Invalid email format",
        "password": "Password must contain uppercase and number",
        "age": "Must be at least 13 years old"
    }
}
```

---

## 🛠️ 5. Custom Validators

### Step 1: สร้าง Annotation

```kotlin
// validation/UniqueEmail.kt
import jakarta.validation.Constraint
import jakarta.validation.Payload
import kotlin.reflect.KClass

@Target(AnnotationTarget.FIELD, AnnotationTarget.VALUE_PARAMETER)
@Retention(AnnotationRetention.RUNTIME)
@Constraint(validatedBy = [UniqueEmailValidator::class])
annotation class UniqueEmail(
    val message: String = "Email already exists",
    val groups: Array<KClass<*>> = [],
    val payload: Array<KClass<out Payload>> = []
)
```

### Step 2: สร้าง Validator

```kotlin
// validation/UniqueEmailValidator.kt
import jakarta.validation.ConstraintValidator
import jakarta.validation.ConstraintValidatorContext

class UniqueEmailValidator(
    private val userRepository: UserRepository
) : ConstraintValidator<UniqueEmail, String> {
    
    override fun isValid(
        value: String?,
        context: ConstraintValidatorContext
    ): Boolean {
        if (value == null) return true  // let @NotNull handle null
        return !userRepository.existsByEmail(value)
    }
}
```

### Step 3: ใช้งาน

```kotlin
data class CreateUserRequest(
    @field:Email
    @field:UniqueEmail  // ← Custom validator!
    val email: String
)
```

---

## 🔗 6. Cross-Field Validation

```kotlin
// Validate fields ที่ depend กัน
@Target(AnnotationTarget.CLASS)
@Retention(AnnotationRetention.RUNTIME)
@Constraint(validatedBy = [PasswordMatchValidator::class])
annotation class PasswordMatch(
    val message: String = "Passwords do not match",
    val groups: Array<KClass<*>> = [],
    val payload: Array<KClass<out Payload>> = []
)

class PasswordMatchValidator : ConstraintValidator<PasswordMatch, Any> {
    override fun isValid(value: Any?, context: ConstraintValidatorContext): Boolean {
        if (value == null) return true
        
        val password = value::class.java.getDeclaredField("password")
            .apply { isAccessible = true }.get(value) as? String
        val confirmPassword = value::class.java.getDeclaredField("confirmPassword")
            .apply { isAccessible = true }.get(value) as? String
        
        return password == confirmPassword
    }
}

@PasswordMatch  // Class-level annotation
data class ChangePasswordRequest(
    @field:NotBlank val currentPassword: String,
    @field:Size(min = 8) val password: String,
    @field:NotBlank val confirmPassword: String
)
```

---

## 🏢 7. Validation ใน Service Layer

```kotlin
// ใช้ Spring @Validated สำหรับ service-level validation
@Service
@Validated  // ต้องมี annotation นี้
class ProductService(private val productRepo: ProductRepository) {
    
    fun createProduct(
        @NotBlank(message = "Name required") name: String,
        @Positive(message = "Price must be positive") price: Double,
        @Min(0) quantity: Int
    ): Product {
        return productRepo.save(Product(name = name, price = price, quantity = quantity))
    }
    
    // Validate objects
    fun updateProduct(@Valid request: UpdateProductRequest): Product {
        // ...
    }
}
```

---

## 🧪 8. Testing Validation

```kotlin
@WebMvcTest(UserController::class)
class UserControllerValidationTest {
    
    @Autowired private lateinit var mockMvc: MockMvc
    @Autowired private lateinit var objectMapper: ObjectMapper
    @MockBean private lateinit var userService: UserService
    
    @Test
    fun `should return 400 when email invalid`() {
        val request = mapOf(
            "name" to "Alice",
            "email" to "not-an-email",
            "password" to "Password123",
            "age" to 25
        )
        
        mockMvc.post("/api/users") {
            contentType = MediaType.APPLICATION_JSON
            content = objectMapper.writeValueAsString(request)
        }.andExpect {
            status { isBadRequest() }
            jsonPath("$.errors.email") { exists() }
        }
    }
    
    @Test
    fun `should return 400 when name blank`() {
        val request = mapOf("name" to "  ", "email" to "a@b.com",
                           "password" to "Password123", "age" to 25)
        
        mockMvc.post("/api/users") {
            contentType = MediaType.APPLICATION_JSON
            content = objectMapper.writeValueAsString(request)
        }.andExpect {
            status { isBadRequest() }
            jsonPath("$.errors.name") { value("Name is required") }
        }
    }
    
    @Test
    fun `should return 201 when request is valid`() {
        val request = mapOf("name" to "Alice", "email" to "alice@example.com",
                           "password" to "Password123", "age" to 25)
        val response = UserResponse(1L, "Alice", "alice@example.com", "USER", true, "")
        
        every { userService.createUser(any()) } returns response
        
        mockMvc.post("/api/users") {
            contentType = MediaType.APPLICATION_JSON
            content = objectMapper.writeValueAsString(request)
        }.andExpect {
            status { isCreated() }
        }
    }
}
```

---

## 📝 สรุป Part 25

| แนวคิด | รายละเอียด |
|--------|-----------|
| `@Valid` | เปิดใช้ Bean Validation |
| `@NotBlank`, `@Email` | Built-in constraints |
| Custom validator | `@Constraint` + `ConstraintValidator` |
| Cross-field | Class-level annotation |
| Error handling | `MethodArgumentNotValidException` |
| `@Validated` | Service-level validation |

---

## ➡️ ถัดไป: Part 26 - Exception Handling ขั้นสูง

---
*Part 25/100+ | Kotlin & Spring Boot Complete Course*
