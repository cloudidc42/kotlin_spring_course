# Part 23: REST API พื้นฐาน
## Building REST APIs with Spring Boot & Kotlin

---

## 🎯 เป้าหมายของ Part นี้

- เข้าใจ REST principles
- สร้าง CRUD API ครบฟีเจอร์
- ใช้ HTTP status codes ถูกต้อง
- ใช้ ResponseEntity
- Request/Response DTOs
- ตัวอย่าง: User Management API

---

## 🌐 1. REST Principles

```
REST = REpresentational State Transfer

หลักการ:
1. Client-Server: แยก UI กับ backend
2. Stateless: ทุก request มีข้อมูลครบในตัว
3. Cacheable: response บอกว่า cache ได้หรือไม่
4. Uniform Interface: ใช้ HTTP methods และ URLs มาตรฐาน
5. Layered System: ใช้ proxies/load balancers ได้

HTTP Methods:
GET    - อ่านข้อมูล (idempotent)
POST   - สร้างข้อมูลใหม่
PUT    - แก้ทั้ง resource
PATCH  - แก้บางส่วน
DELETE - ลบข้อมูล

HTTP Status Codes:
2xx - Success
  200 OK            - ทั่วไป
  201 Created       - สร้างสำเร็จ
  204 No Content    - สำเร็จ ไม่มี body

4xx - Client Error
  400 Bad Request   - ข้อมูลไม่ถูกต้อง
  401 Unauthorized  - ไม่ได้ login
  403 Forbidden     - ไม่มีสิทธิ์
  404 Not Found     - ไม่พบ resource
  409 Conflict      - ข้อมูลซ้ำ
  422 Unprocessable - Validation error

5xx - Server Error
  500 Internal Server Error
  503 Service Unavailable
```

---

## 📦 2. Data Models และ DTOs

```kotlin
// domain/User.kt
package com.example.demo.domain

import jakarta.persistence.*
import java.time.LocalDateTime

@Entity
@Table(name = "users")
data class User(
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long = 0,
    
    @Column(nullable = false, length = 100)
    val name: String,
    
    @Column(unique = true, nullable = false)
    val email: String,
    
    @Column(nullable = false)
    var password: String,
    
    @Enumerated(EnumType.STRING)
    var role: Role = Role.USER,
    
    var isActive: Boolean = true,
    
    val createdAt: LocalDateTime = LocalDateTime.now(),
    var updatedAt: LocalDateTime = LocalDateTime.now()
) {
    enum class Role { USER, ADMIN, MODERATOR }
}

// dto/UserDto.kt - ไม่ส่ง password ออกไป!
data class UserResponse(
    val id: Long,
    val name: String,
    val email: String,
    val role: String,
    val isActive: Boolean,
    val createdAt: String
)

data class CreateUserRequest(
    val name: String,
    val email: String,
    val password: String
)

data class UpdateUserRequest(
    val name: String?,
    val email: String?
)

// Extensions สำหรับ convert
fun User.toResponse() = UserResponse(
    id = id,
    name = name,
    email = email,
    role = role.name,
    isActive = isActive,
    createdAt = createdAt.toString()
)
```

---

## 🗄️ 3. Repository

```kotlin
// repository/UserRepository.kt
package com.example.demo.repository

import com.example.demo.domain.User
import org.springframework.data.jpa.repository.JpaRepository
import org.springframework.data.jpa.repository.Query
import org.springframework.stereotype.Repository

@Repository
interface UserRepository : JpaRepository<User, Long> {
    
    // Spring Data ทำ SQL ให้อัตโนมัติจากชื่อ method!
    fun findByEmail(email: String): User?
    fun findByNameContainingIgnoreCase(name: String): List<User>
    fun existsByEmail(email: String): Boolean
    fun findByIsActiveTrue(): List<User>
    fun findByRole(role: User.Role): List<User>
    
    // Custom JPQL query
    @Query("SELECT u FROM User u WHERE u.isActive = true ORDER BY u.createdAt DESC")
    fun findAllActiveOrderByCreatedAt(): List<User>
    
    @Query("SELECT COUNT(u) FROM User u WHERE u.role = :role")
    fun countByRole(role: User.Role): Long
}
```

---

## ⚙️ 4. Service Layer

```kotlin
// service/UserService.kt
package com.example.demo.service

import com.example.demo.domain.User
import com.example.demo.dto.*
import com.example.demo.repository.UserRepository
import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional
import java.time.LocalDateTime

@Service
@Transactional(readOnly = true)
class UserService(private val userRepository: UserRepository) {
    
    fun getAllUsers(): List<UserResponse> =
        userRepository.findAll().map { it.toResponse() }
    
    fun getUserById(id: Long): UserResponse =
        userRepository.findById(id)
            .orElseThrow { NoSuchElementException("User not found: $id") }
            .toResponse()
    
    fun getUserByEmail(email: String): UserResponse =
        userRepository.findByEmail(email)?.toResponse()
            ?: throw NoSuchElementException("User not found: $email")
    
    fun searchUsers(name: String): List<UserResponse> =
        userRepository.findByNameContainingIgnoreCase(name).map { it.toResponse() }
    
    @Transactional
    fun createUser(request: CreateUserRequest): UserResponse {
        if (userRepository.existsByEmail(request.email)) {
            throw IllegalArgumentException("Email already exists: ${request.email}")
        }
        
        val user = User(
            name = request.name.trim(),
            email = request.email.trim().lowercase(),
            password = hashPassword(request.password)  // ควร hash จริงๆ
        )
        
        return userRepository.save(user).toResponse()
    }
    
    @Transactional
    fun updateUser(id: Long, request: UpdateUserRequest): UserResponse {
        val user = userRepository.findById(id)
            .orElseThrow { NoSuchElementException("User not found: $id") }
        
        // Email uniqueness check
        request.email?.let { newEmail ->
            if (newEmail != user.email && userRepository.existsByEmail(newEmail)) {
                throw IllegalArgumentException("Email already exists: $newEmail")
            }
        }
        
        val updated = user.copy(
            name = request.name ?: user.name,
            email = request.email ?: user.email,
            updatedAt = LocalDateTime.now()
        )
        
        return userRepository.save(updated).toResponse()
    }
    
    @Transactional
    fun deleteUser(id: Long) {
        if (!userRepository.existsById(id)) {
            throw NoSuchElementException("User not found: $id")
        }
        userRepository.deleteById(id)
    }
    
    @Transactional
    fun toggleActive(id: Long): UserResponse {
        val user = userRepository.findById(id)
            .orElseThrow { NoSuchElementException("User not found: $id") }
        
        return userRepository.save(
            user.copy(isActive = !user.isActive, updatedAt = LocalDateTime.now())
        ).toResponse()
    }
    
    private fun hashPassword(password: String): String {
        // ในโปรดักชันใช้ BCrypt!
        return "hashed_$password"
    }
}
```

---

## 🎮 5. REST Controller ครบฟีเจอร์

```kotlin
// controller/UserController.kt
package com.example.demo.controller

import com.example.demo.dto.*
import com.example.demo.service.UserService
import org.springframework.http.HttpStatus
import org.springframework.http.ResponseEntity
import org.springframework.web.bind.annotation.*

@RestController
@RequestMapping("/api/v1/users")
class UserController(private val userService: UserService) {
    
    // GET /api/v1/users
    @GetMapping
    fun getAllUsers(
        @RequestParam(required = false) search: String?
    ): ResponseEntity<List<UserResponse>> {
        val users = if (search != null) {
            userService.searchUsers(search)
        } else {
            userService.getAllUsers()
        }
        return ResponseEntity.ok(users)
    }
    
    // GET /api/v1/users/{id}
    @GetMapping("/{id}")
    fun getUserById(@PathVariable id: Long): ResponseEntity<UserResponse> {
        return ResponseEntity.ok(userService.getUserById(id))
    }
    
    // POST /api/v1/users
    @PostMapping
    fun createUser(@RequestBody request: CreateUserRequest): ResponseEntity<UserResponse> {
        val user = userService.createUser(request)
        return ResponseEntity
            .status(HttpStatus.CREATED)
            .body(user)
    }
    
    // PUT /api/v1/users/{id}
    @PutMapping("/{id}")
    fun updateUser(
        @PathVariable id: Long,
        @RequestBody request: UpdateUserRequest
    ): ResponseEntity<UserResponse> {
        return ResponseEntity.ok(userService.updateUser(id, request))
    }
    
    // DELETE /api/v1/users/{id}
    @DeleteMapping("/{id}")
    fun deleteUser(@PathVariable id: Long): ResponseEntity<Void> {
        userService.deleteUser(id)
        return ResponseEntity.noContent().build()
    }
    
    // PATCH /api/v1/users/{id}/toggle-active
    @PatchMapping("/{id}/toggle-active")
    fun toggleActive(@PathVariable id: Long): ResponseEntity<UserResponse> {
        return ResponseEntity.ok(userService.toggleActive(id))
    }
}
```

---

## 🚨 6. Global Exception Handler

```kotlin
// exception/GlobalExceptionHandler.kt
package com.example.demo.exception

import org.springframework.http.HttpStatus
import org.springframework.http.ResponseEntity
import org.springframework.web.bind.annotation.ExceptionHandler
import org.springframework.web.bind.annotation.RestControllerAdvice
import java.time.LocalDateTime

data class ErrorResponse(
    val timestamp: String = LocalDateTime.now().toString(),
    val status: Int,
    val error: String,
    val message: String,
    val path: String? = null
)

@RestControllerAdvice
class GlobalExceptionHandler {
    
    @ExceptionHandler(NoSuchElementException::class)
    fun handleNotFound(ex: NoSuchElementException): ResponseEntity<ErrorResponse> {
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(
            ErrorResponse(
                status = 404,
                error = "Not Found",
                message = ex.message ?: "Resource not found"
            )
        )
    }
    
    @ExceptionHandler(IllegalArgumentException::class)
    fun handleBadRequest(ex: IllegalArgumentException): ResponseEntity<ErrorResponse> {
        return ResponseEntity.status(HttpStatus.BAD_REQUEST).body(
            ErrorResponse(
                status = 400,
                error = "Bad Request",
                message = ex.message ?: "Invalid input"
            )
        )
    }
    
    @ExceptionHandler(Exception::class)
    fun handleGeneral(ex: Exception): ResponseEntity<ErrorResponse> {
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body(
            ErrorResponse(
                status = 500,
                error = "Internal Server Error",
                message = "An unexpected error occurred"
            )
        )
    }
}
```

---

## 📊 7. ทดสอบ API

```bash
# สร้าง user
curl -X POST http://localhost:8080/api/v1/users \
  -H "Content-Type: application/json" \
  -d '{"name":"Alice Smith","email":"alice@example.com","password":"pass123"}'
# Response: {"id":1,"name":"Alice Smith","email":"alice@example.com",...}

# ดู users ทั้งหมด
curl http://localhost:8080/api/v1/users
# [{"id":1,...},{"id":2,...}]

# ดู user ที่ id=1
curl http://localhost:8080/api/v1/users/1
# {"id":1,"name":"Alice Smith",...}

# ค้นหา
curl "http://localhost:8080/api/v1/users?search=alice"
# [{"id":1,"name":"Alice Smith",...}]

# อัปเดต
curl -X PUT http://localhost:8080/api/v1/users/1 \
  -H "Content-Type: application/json" \
  -d '{"name":"Alice Johnson"}'
# {"id":1,"name":"Alice Johnson",...}

# ลบ
curl -X DELETE http://localhost:8080/api/v1/users/1
# 204 No Content

# Not found
curl http://localhost:8080/api/v1/users/999
# {"status":404,"error":"Not Found","message":"User not found: 999"}
```

---

## 🧪 8. Unit Tests สำหรับ Controller

```kotlin
@WebMvcTest(UserController::class)
class UserControllerTest {
    
    @Autowired
    private lateinit var mockMvc: MockMvc
    
    @MockBean
    private lateinit var userService: UserService
    
    @Autowired
    private lateinit var objectMapper: ObjectMapper
    
    private val sampleUser = UserResponse(
        id = 1L,
        name = "Alice",
        email = "alice@example.com",
        role = "USER",
        isActive = true,
        createdAt = LocalDateTime.now().toString()
    )
    
    @Test
    fun `GET all users should return 200 with list`() {
        every { userService.getAllUsers() } returns listOf(sampleUser)
        
        mockMvc.get("/api/v1/users")
            .andExpect {
                status { isOk() }
                jsonPath("$[0].name") { value("Alice") }
                jsonPath("$[0].email") { value("alice@example.com") }
            }
    }
    
    @Test
    fun `GET user by id should return 200`() {
        every { userService.getUserById(1L) } returns sampleUser
        
        mockMvc.get("/api/v1/users/1")
            .andExpect {
                status { isOk() }
                jsonPath("$.id") { value(1) }
            }
    }
    
    @Test
    fun `GET non-existent user should return 404`() {
        every { userService.getUserById(999L) } throws NoSuchElementException("User not found")
        
        mockMvc.get("/api/v1/users/999")
            .andExpect {
                status { isNotFound() }
            }
    }
    
    @Test
    fun `POST create user should return 201`() {
        val request = CreateUserRequest("Bob", "bob@example.com", "pass123")
        val created = sampleUser.copy(id = 2L, name = "Bob", email = "bob@example.com")
        
        every { userService.createUser(request) } returns created
        
        mockMvc.post("/api/v1/users") {
            contentType = MediaType.APPLICATION_JSON
            content = objectMapper.writeValueAsString(request)
        }.andExpect {
            status { isCreated() }
            jsonPath("$.name") { value("Bob") }
        }
    }
}
```

---

## 📝 สรุป Part 23

| แนวคิด | รายละเอียด |
|--------|-----------|
| REST | Stateless, uniform interface |
| `@RestController` | REST API controller |
| HTTP Methods | GET/POST/PUT/PATCH/DELETE |
| `ResponseEntity` | Control HTTP response |
| DTOs | Request/Response objects |
| Exception Handler | `@RestControllerAdvice` |
| Status codes | 200/201/204/400/404/500 |

---

## ➡️ ถัดไป: Part 24 - Spring Data JPA

---
*Part 23/100+ | Kotlin & Spring Boot Complete Course*
