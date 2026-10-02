# Part 61: Clean Architecture - User Management System

## บทนำ

**Clean Architecture** คือสถาปัตยกรรมที่ Robert C. Martin (Uncle Bob) เสนอขึ้นในปี 2012 หลักการสำคัญคือ **Dependency Rule**: ทุก dependency ต้องชี้เข้าหา center (business logic) เท่านั้น ไม่มี outer layer ใดรู้จัก inner layer

## The Clean Architecture Diagram

```
┌─────────────────────────────────────────────────────┐
│              Frameworks & Drivers                    │
│   (Spring Boot, JPA, REST, Database, Web)           │
│  ┌─────────────────────────────────────────────┐    │
│  │           Interface Adapters                 │    │
│  │  (Controllers, Gateways, Presenters)        │    │
│  │  ┌─────────────────────────────────────┐    │    │
│  │  │        Application Business Rules    │    │    │
│  │  │  (Use Cases / Interactors)          │    │    │
│  │  │  ┌───────────────────────────────┐  │    │    │
│  │  │  │   Enterprise Business Rules   │  │    │    │
│  │  │  │      (Entities)               │  │    │    │
│  │  │  └───────────────────────────────┘  │    │    │
│  │  └─────────────────────────────────────┘    │    │
│  └─────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────┘

Dependency Direction: → (outer depends on inner, NEVER inner on outer)
```

## Dependency Rule

**Inner layers ต้องไม่รู้จัก outer layers:**
- Entities ไม่รู้จัก Use Cases
- Use Cases ไม่รู้จัก Controllers หรือ Database
- Controllers ไม่รู้จัก Spring Boot annotations (ideally)

## โครงสร้างโปรเจกต์

```
user-management/
├── domain/
│   ├── entity/
│   │   └── User.kt
│   ├── valueobject/
│   │   ├── Email.kt
│   │   ├── Password.kt
│   │   └── UserId.kt
│   └── exception/
│       └── DomainExceptions.kt
├── application/
│   ├── usecase/
│   │   ├── CreateUserUseCase.kt
│   │   ├── GetUserUseCase.kt
│   │   ├── UpdateUserUseCase.kt
│   │   └── DeleteUserUseCase.kt
│   ├── port/
│   │   ├── input/
│   │   │   └── UserInputPort.kt
│   │   └── output/
│   │       ├── UserOutputPort.kt
│   │       └── PasswordEncoderPort.kt
│   └── dto/
│       └── UserDtos.kt
├── adapter/
│   ├── input/
│   │   ├── web/
│   │   │   └── UserController.kt
│   │   └── messaging/
│   │       └── UserEventListener.kt
│   └── output/
│       ├── persistence/
│       │   ├── UserJpaAdapter.kt
│       │   └── UserJpaRepository.kt
│       └── security/
│           └── BCryptPasswordEncoderAdapter.kt
└── config/
    └── BeanConfig.kt
```

## Layer 1: Entities (Enterprise Business Rules)

```kotlin
// domain/entity/User.kt
package com.user.clean.domain.entity

import com.user.clean.domain.exception.InvalidEmailException
import com.user.clean.domain.exception.WeakPasswordException
import com.user.clean.domain.valueobject.*
import java.time.LocalDateTime

class User private constructor(
    val id: UserId,
    private var email: Email,
    private var hashedPassword: HashedPassword,
    private var firstName: String,
    private var lastName: String,
    private var role: UserRole,
    private var active: Boolean,
    val createdAt: LocalDateTime,
    private var updatedAt: LocalDateTime
) {

    companion object {
        fun create(
            email: Email,
            hashedPassword: HashedPassword,
            firstName: String,
            lastName: String,
            role: UserRole = UserRole.USER
        ): User {
            require(firstName.isNotBlank()) { "First name cannot be blank" }
            require(lastName.isNotBlank()) { "Last name cannot be blank" }
            
            val now = LocalDateTime.now()
            return User(
                id = UserId.generate(),
                email = email,
                hashedPassword = hashedPassword,
                firstName = firstName,
                lastName = lastName,
                role = role,
                active = true,
                createdAt = now,
                updatedAt = now
            )
        }

        fun reconstitute(
            id: UserId,
            email: Email,
            hashedPassword: HashedPassword,
            firstName: String,
            lastName: String,
            role: UserRole,
            active: Boolean,
            createdAt: LocalDateTime,
            updatedAt: LocalDateTime
        ): User {
            return User(id, email, hashedPassword, firstName, lastName, role, active, createdAt, updatedAt)
        }
    }

    fun changeEmail(newEmail: Email): User {
        this.email = newEmail
        this.updatedAt = LocalDateTime.now()
        return this
    }

    fun changeName(firstName: String, lastName: String): User {
        require(firstName.isNotBlank()) { "First name cannot be blank" }
        require(lastName.isNotBlank()) { "Last name cannot be blank" }
        
        this.firstName = firstName
        this.lastName = lastName
        this.updatedAt = LocalDateTime.now()
        return this
    }

    fun changePassword(newHashedPassword: HashedPassword): User {
        this.hashedPassword = newHashedPassword
        this.updatedAt = LocalDateTime.now()
        return this
    }

    fun activate(): User {
        this.active = true
        this.updatedAt = LocalDateTime.now()
        return this
    }

    fun deactivate(): User {
        this.active = false
        this.updatedAt = LocalDateTime.now()
        return this
    }

    fun promoteToAdmin(): User {
        this.role = UserRole.ADMIN
        this.updatedAt = LocalDateTime.now()
        return this
    }

    // Queries
    fun getEmail(): Email = email
    fun getHashedPassword(): HashedPassword = hashedPassword
    fun getFirstName(): String = firstName
    fun getLastName(): String = lastName
    fun getFullName(): String = "$firstName $lastName"
    fun getRole(): UserRole = role
    fun isActive(): Boolean = active
    fun getUpdatedAt(): LocalDateTime = updatedAt
    fun isAdmin(): Boolean = role == UserRole.ADMIN
}

enum class UserRole {
    USER, ADMIN, MODERATOR
}
```

### Value Objects

```kotlin
// domain/valueobject/Email.kt
package com.user.clean.domain.valueobject

import com.user.clean.domain.exception.InvalidEmailException

@JvmInline
value class Email(val value: String) {
    init {
        val emailRegex = Regex("^[A-Za-z0-9+_.-]+@[A-Za-z0-9.-]+\\.[A-Za-z]{2,}$")
        if (!emailRegex.matches(value)) {
            throw InvalidEmailException("Invalid email format: $value")
        }
    }

    companion object {
        fun of(value: String): Email = Email(value.lowercase().trim())
    }
}
```

```kotlin
// domain/valueobject/Password.kt
package com.user.clean.domain.valueobject

import com.user.clean.domain.exception.WeakPasswordException

@JvmInline
value class RawPassword(val value: String) {
    init {
        if (value.length < 8) {
            throw WeakPasswordException("Password must be at least 8 characters")
        }
        if (!value.any { it.isDigit() }) {
            throw WeakPasswordException("Password must contain at least one digit")
        }
        if (!value.any { it.isLetter() }) {
            throw WeakPasswordException("Password must contain at least one letter")
        }
    }
}

@JvmInline
value class HashedPassword(val value: String) {
    // Hashed password - no validation needed
}

@JvmInline
value class UserId(val value: String) {
    companion object {
        fun generate() = UserId(java.util.UUID.randomUUID().toString())
        fun of(value: String) = UserId(value)
    }
}
```

## Layer 2: Use Cases (Application Business Rules)

### Ports (Interfaces)

```kotlin
// application/port/output/UserOutputPort.kt
package com.user.clean.application.port.output

import com.user.clean.domain.entity.User
import com.user.clean.domain.valueobject.*

// Output Port - what application needs from infrastructure
interface UserOutputPort {
    fun save(user: User): User
    fun findById(id: UserId): User?
    fun findByEmail(email: Email): User?
    fun existsByEmail(email: Email): Boolean
    fun findAll(page: Int, size: Int): List<User>
    fun count(): Long
    fun delete(id: UserId)
}
```

```kotlin
// application/port/output/PasswordEncoderPort.kt
package com.user.clean.application.port.output

import com.user.clean.domain.valueobject.*

interface PasswordEncoderPort {
    fun encode(rawPassword: RawPassword): HashedPassword
    fun matches(rawPassword: RawPassword, hashedPassword: HashedPassword): Boolean
}
```

```kotlin
// application/port/output/EmailNotificationPort.kt
package com.user.clean.application.port.output

interface EmailNotificationPort {
    fun sendWelcomeEmail(email: String, firstName: String)
    fun sendPasswordResetEmail(email: String, resetToken: String)
}
```

```kotlin
// application/port/input/UserInputPort.kt
package com.user.clean.application.port.input

import com.user.clean.application.dto.*

// Input Port - what the application can do
interface UserInputPort {
    fun createUser(command: CreateUserCommand): UserResult
    fun getUser(query: GetUserQuery): UserResult
    fun updateUser(command: UpdateUserCommand): UserResult
    fun deleteUser(command: DeleteUserCommand)
    fun listUsers(query: ListUsersQuery): UserListResult
    fun changePassword(command: ChangePasswordCommand)
}
```

### Use Cases

```kotlin
// application/usecase/CreateUserUseCase.kt
package com.user.clean.application.usecase

import com.user.clean.application.dto.*
import com.user.clean.application.port.input.UserInputPort
import com.user.clean.application.port.output.*
import com.user.clean.domain.entity.User
import com.user.clean.domain.exception.UserAlreadyExistsException
import com.user.clean.domain.valueobject.*

class CreateUserUseCase(
    private val userRepository: UserOutputPort,
    private val passwordEncoder: PasswordEncoderPort,
    private val emailNotification: EmailNotificationPort
) : UserInputPort {

    override fun createUser(command: CreateUserCommand): UserResult {
        val email = Email.of(command.email)
        
        // Business rule: email must be unique
        if (userRepository.existsByEmail(email)) {
            throw UserAlreadyExistsException("User with email ${command.email} already exists")
        }

        val rawPassword = RawPassword(command.password)
        val hashedPassword = passwordEncoder.encode(rawPassword)

        val user = User.create(
            email = email,
            hashedPassword = hashedPassword,
            firstName = command.firstName,
            lastName = command.lastName
        )

        val savedUser = userRepository.save(user)
        
        // Side effect: send welcome email
        emailNotification.sendWelcomeEmail(
            email = command.email,
            firstName = command.firstName
        )

        return savedUser.toResult()
    }

    override fun getUser(query: GetUserQuery): UserResult {
        val user = userRepository.findById(UserId.of(query.userId))
            ?: throw UserNotFoundException("User ${query.userId} not found")
        return user.toResult()
    }

    override fun updateUser(command: UpdateUserCommand): UserResult {
        val user = userRepository.findById(UserId.of(command.userId))
            ?: throw UserNotFoundException("User ${command.userId} not found")

        command.firstName?.let { firstName ->
            command.lastName?.let { lastName ->
                user.changeName(firstName, lastName)
            }
        }

        command.email?.let { email ->
            val newEmail = Email.of(email)
            if (userRepository.existsByEmail(newEmail)) {
                throw UserAlreadyExistsException("Email $email already in use")
            }
            user.changeEmail(newEmail)
        }

        return userRepository.save(user).toResult()
    }

    override fun deleteUser(command: DeleteUserCommand) {
        val user = userRepository.findById(UserId.of(command.userId))
            ?: throw UserNotFoundException("User ${command.userId} not found")
        
        // Business rule: cannot delete admin users
        if (user.isAdmin()) {
            throw ForbiddenOperationException("Cannot delete admin users")
        }

        userRepository.delete(UserId.of(command.userId))
    }

    override fun listUsers(query: ListUsersQuery): UserListResult {
        val users = userRepository.findAll(query.page, query.size)
        val total = userRepository.count()
        
        return UserListResult(
            users = users.map { it.toResult() },
            total = total,
            page = query.page,
            size = query.size
        )
    }

    override fun changePassword(command: ChangePasswordCommand) {
        val user = userRepository.findById(UserId.of(command.userId))
            ?: throw UserNotFoundException("User ${command.userId} not found")

        // Verify old password
        if (!passwordEncoder.matches(RawPassword(command.oldPassword), user.getHashedPassword())) {
            throw InvalidPasswordException("Current password is incorrect")
        }

        val newHashedPassword = passwordEncoder.encode(RawPassword(command.newPassword))
        user.changePassword(newHashedPassword)
        userRepository.save(user)
    }
}

// Extension function for mapping
fun User.toResult() = UserResult(
    id = id.value,
    email = getEmail().value,
    firstName = getFirstName(),
    lastName = getLastName(),
    fullName = getFullName(),
    role = getRole().name,
    active = isActive(),
    createdAt = createdAt,
    updatedAt = getUpdatedAt()
)
```

### DTOs

```kotlin
// application/dto/UserDtos.kt
package com.user.clean.application.dto

import java.time.LocalDateTime

// Commands
data class CreateUserCommand(
    val email: String,
    val password: String,
    val firstName: String,
    val lastName: String
)

data class UpdateUserCommand(
    val userId: String,
    val email: String? = null,
    val firstName: String? = null,
    val lastName: String? = null
)

data class DeleteUserCommand(val userId: String)

data class ChangePasswordCommand(
    val userId: String,
    val oldPassword: String,
    val newPassword: String
)

// Queries
data class GetUserQuery(val userId: String)
data class ListUsersQuery(val page: Int = 0, val size: Int = 20)

// Results
data class UserResult(
    val id: String,
    val email: String,
    val firstName: String,
    val lastName: String,
    val fullName: String,
    val role: String,
    val active: Boolean,
    val createdAt: LocalDateTime,
    val updatedAt: LocalDateTime
)

data class UserListResult(
    val users: List<UserResult>,
    val total: Long,
    val page: Int,
    val size: Int
)
```

## Layer 3: Interface Adapters

### Web Controller

```kotlin
// adapter/input/web/UserController.kt
package com.user.clean.adapter.input.web

import com.user.clean.application.dto.*
import com.user.clean.application.port.input.UserInputPort
import jakarta.validation.Valid
import org.springframework.http.HttpStatus
import org.springframework.http.ResponseEntity
import org.springframework.web.bind.annotation.*

@RestController
@RequestMapping("/api/users")
class UserController(
    private val userInputPort: UserInputPort
) {

    @PostMapping
    fun createUser(
        @Valid @RequestBody request: CreateUserRequest
    ): ResponseEntity<UserResponse> {
        val command = CreateUserCommand(
            email = request.email,
            password = request.password,
            firstName = request.firstName,
            lastName = request.lastName
        )
        val result = userInputPort.createUser(command)
        return ResponseEntity.status(HttpStatus.CREATED).body(result.toResponse())
    }

    @GetMapping("/{userId}")
    fun getUser(@PathVariable userId: String): ResponseEntity<UserResponse> {
        val result = userInputPort.getUser(GetUserQuery(userId))
        return ResponseEntity.ok(result.toResponse())
    }

    @PutMapping("/{userId}")
    fun updateUser(
        @PathVariable userId: String,
        @Valid @RequestBody request: UpdateUserRequest
    ): ResponseEntity<UserResponse> {
        val command = UpdateUserCommand(
            userId = userId,
            email = request.email,
            firstName = request.firstName,
            lastName = request.lastName
        )
        val result = userInputPort.updateUser(command)
        return ResponseEntity.ok(result.toResponse())
    }

    @DeleteMapping("/{userId}")
    fun deleteUser(@PathVariable userId: String): ResponseEntity<Void> {
        userInputPort.deleteUser(DeleteUserCommand(userId))
        return ResponseEntity.noContent().build()
    }

    @GetMapping
    fun listUsers(
        @RequestParam(defaultValue = "0") page: Int,
        @RequestParam(defaultValue = "20") size: Int
    ): ResponseEntity<UserListResponse> {
        val result = userInputPort.listUsers(ListUsersQuery(page, size))
        return ResponseEntity.ok(result.toResponse())
    }

    @PatchMapping("/{userId}/password")
    fun changePassword(
        @PathVariable userId: String,
        @Valid @RequestBody request: ChangePasswordRequest
    ): ResponseEntity<Void> {
        userInputPort.changePassword(ChangePasswordCommand(
            userId = userId,
            oldPassword = request.oldPassword,
            newPassword = request.newPassword
        ))
        return ResponseEntity.ok().build()
    }
}

// Web Request/Response DTOs (different from application DTOs!)
data class CreateUserRequest(
    @field:NotBlank val email: String,
    @field:NotBlank val password: String,
    @field:NotBlank val firstName: String,
    @field:NotBlank val lastName: String
)

data class UpdateUserRequest(
    val email: String? = null,
    val firstName: String? = null,
    val lastName: String? = null
)

data class ChangePasswordRequest(
    @field:NotBlank val oldPassword: String,
    @field:NotBlank val newPassword: String
)

data class UserResponse(
    val id: String,
    val email: String,
    val firstName: String,
    val lastName: String,
    val role: String,
    val active: Boolean
)

data class UserListResponse(
    val users: List<UserResponse>,
    val total: Long,
    val page: Int,
    val size: Int
)

fun UserResult.toResponse() = UserResponse(
    id = id,
    email = email,
    firstName = firstName,
    lastName = lastName,
    role = role,
    active = active
)

fun UserListResult.toResponse() = UserListResponse(
    users = users.map { it.toResponse() },
    total = total,
    page = page,
    size = size
)
```

## Layer 4: Frameworks & Drivers

### JPA Adapter

```kotlin
// adapter/output/persistence/UserJpaAdapter.kt
package com.user.clean.adapter.output.persistence

import com.user.clean.application.port.output.UserOutputPort
import com.user.clean.domain.entity.*
import com.user.clean.domain.valueobject.*
import jakarta.persistence.*
import org.springframework.data.jpa.repository.JpaRepository
import org.springframework.stereotype.Repository

@Repository
class UserJpaAdapter(
    private val jpaRepository: UserJpaRepository
) : UserOutputPort {

    override fun save(user: User): User {
        val entity = user.toEntity()
        val saved = jpaRepository.save(entity)
        return saved.toDomain()
    }

    override fun findById(id: UserId): User? {
        return jpaRepository.findById(id.value).map { it.toDomain() }.orElse(null)
    }

    override fun findByEmail(email: Email): User? {
        return jpaRepository.findByEmail(email.value)?.toDomain()
    }

    override fun existsByEmail(email: Email): Boolean {
        return jpaRepository.existsByEmail(email.value)
    }

    override fun findAll(page: Int, size: Int): List<User> {
        val pageable = org.springframework.data.domain.PageRequest.of(page, size)
        return jpaRepository.findAll(pageable).content.map { it.toDomain() }
    }

    override fun count(): Long = jpaRepository.count()

    override fun delete(id: UserId) {
        jpaRepository.deleteById(id.value)
    }
}

// JPA Entity (Infrastructure concern)
@Entity
@Table(name = "users")
data class UserJpaEntity(
    @Id
    val id: String,
    @Column(unique = true, nullable = false)
    val email: String,
    @Column(nullable = false)
    val hashedPassword: String,
    @Column(nullable = false)
    val firstName: String,
    @Column(nullable = false)
    val lastName: String,
    @Enumerated(EnumType.STRING)
    val role: String,
    val active: Boolean,
    val createdAt: java.time.LocalDateTime,
    val updatedAt: java.time.LocalDateTime
)

// JPA Repository
interface UserJpaRepository : JpaRepository<UserJpaEntity, String> {
    fun findByEmail(email: String): UserJpaEntity?
    fun existsByEmail(email: String): Boolean
}

// Mapping functions
fun User.toEntity() = UserJpaEntity(
    id = id.value,
    email = getEmail().value,
    hashedPassword = getHashedPassword().value,
    firstName = getFirstName(),
    lastName = getLastName(),
    role = getRole().name,
    active = isActive(),
    createdAt = createdAt,
    updatedAt = getUpdatedAt()
)

fun UserJpaEntity.toDomain() = User.reconstitute(
    id = UserId.of(id),
    email = Email.of(email),
    hashedPassword = HashedPassword(hashedPassword),
    firstName = firstName,
    lastName = lastName,
    role = UserRole.valueOf(role),
    active = active,
    createdAt = createdAt,
    updatedAt = updatedAt
)
```

### Password Encoder Adapter

```kotlin
// adapter/output/security/BCryptPasswordEncoderAdapter.kt
package com.user.clean.adapter.output.security

import com.user.clean.application.port.output.PasswordEncoderPort
import com.user.clean.domain.valueobject.*
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder
import org.springframework.stereotype.Component

@Component
class BCryptPasswordEncoderAdapter : PasswordEncoderPort {
    
    private val bcrypt = BCryptPasswordEncoder(12)

    override fun encode(rawPassword: RawPassword): HashedPassword {
        return HashedPassword(bcrypt.encode(rawPassword.value))
    }

    override fun matches(rawPassword: RawPassword, hashedPassword: HashedPassword): Boolean {
        return bcrypt.matches(rawPassword.value, hashedPassword.value)
    }
}
```

### Dependency Injection Configuration

```kotlin
// config/BeanConfig.kt
package com.user.clean.config

import com.user.clean.application.port.output.*
import com.user.clean.application.usecase.CreateUserUseCase
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration

@Configuration
class BeanConfig {

    @Bean
    fun createUserUseCase(
        userRepository: UserOutputPort,
        passwordEncoder: PasswordEncoderPort,
        emailNotification: EmailNotificationPort
    ): CreateUserUseCase {
        return CreateUserUseCase(userRepository, passwordEncoder, emailNotification)
    }
}
```

## Testing

```kotlin
// CreateUserUseCaseTest.kt
package com.user.clean.application.usecase

import com.user.clean.application.dto.CreateUserCommand
import com.user.clean.application.port.output.*
import com.user.clean.domain.exception.UserAlreadyExistsException
import com.user.clean.domain.valueobject.*
import io.mockk.*
import org.junit.jupiter.api.Test
import org.junit.jupiter.api.assertThrows

class CreateUserUseCaseTest {

    private val userRepository = mockk<UserOutputPort>()
    private val passwordEncoder = mockk<PasswordEncoderPort>()
    private val emailNotification = mockk<EmailNotificationPort>(relaxed = true)

    private val useCase = CreateUserUseCase(userRepository, passwordEncoder, emailNotification)

    @Test
    fun `should create user successfully`() {
        val command = CreateUserCommand(
            email = "john@example.com",
            password = "Password123",
            firstName = "John",
            lastName = "Doe"
        )

        every { userRepository.existsByEmail(any()) } returns false
        every { passwordEncoder.encode(any()) } returns HashedPassword("hashed-password")
        every { userRepository.save(any()) } answers { firstArg() }

        val result = useCase.createUser(command)

        verify { userRepository.save(any()) }
        verify { emailNotification.sendWelcomeEmail("john@example.com", "John") }
        assert(result.email == "john@example.com")
    }

    @Test
    fun `should throw exception when email already exists`() {
        every { userRepository.existsByEmail(any()) } returns true

        assertThrows<UserAlreadyExistsException> {
            useCase.createUser(
                CreateUserCommand("existing@example.com", "Password123", "John", "Doe")
            )
        }
    }

    @Test
    fun `should not delete admin user`() {
        // Test business rule: admin cannot be deleted
        val adminUser = createAdminUser()
        every { userRepository.findById(any()) } returns adminUser

        assertThrows<ForbiddenOperationException> {
            useCase.deleteUser(DeleteUserCommand("admin-id"))
        }
    }
}
```

## สรุป Clean Architecture

| Layer | ความรับผิดชอบ | Dependency |
|-------|-------------|-----------|
| Entities | Business rules หลัก | ไม่ขึ้นกับอะไร |
| Use Cases | Application logic | Entities เท่านั้น |
| Interface Adapters | แปลง format | Use Cases |
| Frameworks | Technical details | ทุกอย่าง |

## ข้อดีของ Clean Architecture

1. **Independent of Framework** - ไม่ผูกกับ Spring Boot
2. **Testable** - Business logic ทดสอบได้โดยไม่ต้องการ infrastructure
3. **Independent of Database** - เปลี่ยน DB ได้ง่าย
4. **Independent of UI** - เปลี่ยน REST เป็น gRPC ได้
5. **Independent of External Services** - Mock ได้ง่าย

*Part 61/100+ | Kotlin & Spring Boot Complete Course*
