# Part 29: Testing ใน Spring Boot
## Unit, Integration, และ E2E Tests

---

## 🎯 เป้าหมายของ Part นี้

- Unit testing ด้วย JUnit 5 + MockK
- Integration testing ด้วย @SpringBootTest
- Controller testing ด้วย @WebMvcTest + MockMvc
- Repository testing ด้วย @DataJpaTest
- Testcontainers สำหรับ integration tests
- ตัวอย่าง: Testing ครบทุก layer

---

## 📦 1. Dependencies

```kotlin
// build.gradle.kts
dependencies {
    testImplementation("org.springframework.boot:spring-boot-starter-test")
    testImplementation("io.mockk:mockk:1.13.8")
    testImplementation("com.ninja-squad:springmockk:4.0.2")
    testImplementation("org.testcontainers:junit-jupiter:1.19.3")
    testImplementation("org.testcontainers:postgresql:1.19.3")
}
```

---

## 🧪 2. Unit Tests

### Service Layer Testing

```kotlin
// src/test/kotlin/com/example/service/UserServiceTest.kt

import io.mockk.*
import io.mockk.impl.annotations.InjectMockKs
import io.mockk.impl.annotations.MockK
import org.junit.jupiter.api.*
import org.junit.jupiter.api.extension.ExtendWith
import io.mockk.junit5.MockKExtension
import org.assertj.core.api.Assertions.*

@ExtendWith(MockKExtension::class)
class UserServiceTest {
    
    @MockK
    lateinit var userRepository: UserRepository
    
    @MockK
    lateinit var passwordEncoder: PasswordEncoder
    
    @InjectMockKs
    lateinit var userService: UserService
    
    @Test
    fun `createUser should save and return user`() {
        // Arrange
        val name = "Alice"
        val email = "alice@example.com"
        val savedUser = User(id = 1L, name = name, email = email)
        
        every { userRepository.existsByEmail(email) } returns false
        every { passwordEncoder.encode(any()) } returns "hashed"
        every { userRepository.save(any()) } returns savedUser
        
        // Act
        val result = userService.createUser(name, email, "password")
        
        // Assert
        assertThat(result.name).isEqualTo(name)
        assertThat(result.email).isEqualTo(email)
        
        verify(exactly = 1) { userRepository.existsByEmail(email) }
        verify(exactly = 1) { userRepository.save(any()) }
    }
    
    @Test
    fun `createUser should throw when email exists`() {
        every { userRepository.existsByEmail("dup@example.com") } returns true
        
        assertThrows<ConflictException> {
            userService.createUser("Bob", "dup@example.com", "password")
        }
        
        verify(exactly = 0) { userRepository.save(any()) }  // should NOT save
    }
    
    @Test
    fun `getUser should throw NotFoundException when not found`() {
        every { userRepository.findById(999L) } returns Optional.empty()
        
        assertThrows<NotFoundException> {
            userService.getUser(999L)
        }
    }
    
    @Test
    fun `deleteUser should call deleteById`() {
        every { userRepository.existsById(1L) } returns true
        every { userRepository.deleteById(1L) } just Runs
        
        userService.deleteUser(1L)
        
        verify(exactly = 1) { userRepository.deleteById(1L) }
    }
}
```

### Kotlin DSL Tests

```kotlin
// ใช้ kotest-style with JUnit 5
@Test
fun `should calculate correct total`() {
    val order = Order(
        items = listOf(
            OrderItem("Apple", 2, 10.0),
            OrderItem("Banana", 3, 5.0)
        )
    )
    
    assertThat(order.total()).isEqualTo(35.0)
}

// Parameterized Tests
@ParameterizedTest
@CsvSource(
    "Alice, alice@example.com, true",
    "  , invalid, false",
    "Bob, not-email, false"
)
fun `validateUser should return expected result`(
    name: String, email: String, expected: Boolean
) {
    val result = userValidator.isValid(name.trim(), email)
    assertThat(result).isEqualTo(expected)
}
```

---

## 🎮 3. Controller Tests (@WebMvcTest)

```kotlin
// src/test/kotlin/com/example/controller/UserControllerTest.kt

@WebMvcTest(UserController::class)
class UserControllerTest {
    
    @Autowired private lateinit var mockMvc: MockMvc
    @Autowired private lateinit var objectMapper: ObjectMapper
    @MockkBean private lateinit var userService: UserService  // springmockk
    
    private val sampleUser = UserResponse(
        id = 1L, name = "Alice", email = "alice@example.com",
        role = "USER", isActive = true, createdAt = "2026-01-01"
    )
    
    @Nested
    @DisplayName("GET /api/users")
    inner class GetUsers {
        
        @Test
        fun `should return 200 with list of users`() {
            every { userService.getAllUsers() } returns listOf(sampleUser)
            
            mockMvc.get("/api/users")
                .andExpect {
                    status { isOk() }
                    content { contentType(MediaType.APPLICATION_JSON) }
                    jsonPath("$") { isArray() }
                    jsonPath("$[0].name") { value("Alice") }
                    jsonPath("$[0].email") { value("alice@example.com") }
                }
        }
        
        @Test
        fun `should return empty list when no users`() {
            every { userService.getAllUsers() } returns emptyList()
            
            mockMvc.get("/api/users")
                .andExpect {
                    status { isOk() }
                    jsonPath("$") { isEmpty() }
                }
        }
    }
    
    @Nested
    @DisplayName("POST /api/users")
    inner class CreateUser {
        
        @Test
        fun `should return 201 when valid request`() {
            val request = CreateUserRequest("Alice", "alice@example.com", "Password123")
            every { userService.createUser(request) } returns sampleUser
            
            mockMvc.post("/api/users") {
                contentType = MediaType.APPLICATION_JSON
                content = objectMapper.writeValueAsString(request)
            }.andExpect {
                status { isCreated() }
                jsonPath("$.name") { value("Alice") }
            }
        }
        
        @Test
        fun `should return 400 when email invalid`() {
            val request = mapOf("name" to "Alice", "email" to "bad-email", "password" to "Pass123")
            
            mockMvc.post("/api/users") {
                contentType = MediaType.APPLICATION_JSON
                content = objectMapper.writeValueAsString(request)
            }.andExpect {
                status { isBadRequest() }
                jsonPath("$.errors.email") { exists() }
            }
        }
        
        @Test
        fun `should return 409 when email duplicate`() {
            every { userService.createUser(any()) } throws ConflictException(
                "Email already exists", "USER_EMAIL_DUPLICATE"
            )
            
            mockMvc.post("/api/users") {
                contentType = MediaType.APPLICATION_JSON
                content = """{"name":"Alice","email":"a@b.com","password":"Pass123"}"""
            }.andExpect {
                status { isConflict() }
                jsonPath("$.errorCode") { value("USER_EMAIL_DUPLICATE") }
            }
        }
    }
}
```

---

## 🗄️ 4. Repository Tests (@DataJpaTest)

```kotlin
// src/test/kotlin/com/example/repository/UserRepositoryTest.kt

@DataJpaTest  // ใช้ H2 in-memory, rollback หลังแต่ละ test
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
class UserRepositoryTest {
    
    @Autowired
    private lateinit var userRepository: UserRepository
    
    @Autowired
    private lateinit var entityManager: TestEntityManager
    
    @BeforeEach
    fun setup() {
        entityManager.persist(User(name = "Alice", email = "alice@example.com"))
        entityManager.persist(User(name = "Bob", email = "bob@example.com"))
        entityManager.persist(User(name = "Charlie", email = "charlie@example.com"))
        entityManager.flush()
    }
    
    @Test
    fun `findByEmail should return user when exists`() {
        val user = userRepository.findByEmail("alice@example.com")
        assertThat(user).isNotNull
        assertThat(user!!.name).isEqualTo("Alice")
    }
    
    @Test
    fun `findByEmail should return null when not exists`() {
        val user = userRepository.findByEmail("notfound@example.com")
        assertThat(user).isNull()
    }
    
    @Test
    fun `existsByEmail should return true when email registered`() {
        assertThat(userRepository.existsByEmail("alice@example.com")).isTrue
        assertThat(userRepository.existsByEmail("unknown@example.com")).isFalse
    }
    
    @Test
    fun `findByNameContaining should return matching users`() {
        val users = userRepository.findByNameContainingIgnoreCase("ali")
        assertThat(users).hasSize(1)
        assertThat(users[0].email).isEqualTo("alice@example.com")
    }
}
```

---

## 🐳 5. Testcontainers (Real DB)

```kotlin
// src/test/kotlin/com/example/TestContainersConfig.kt

@SpringBootTest
@Testcontainers
@ActiveProfiles("test")
abstract class BaseIntegrationTest {
    
    companion object {
        @Container
        @JvmStatic
        val postgres = PostgreSQLContainer<Nothing>("postgres:15-alpine").apply {
            withDatabaseName("testdb")
            withUsername("test")
            withPassword("test")
        }
        
        @DynamicPropertySource
        @JvmStatic
        fun configureProperties(registry: DynamicPropertyRegistry) {
            registry.add("spring.datasource.url", postgres::getJdbcUrl)
            registry.add("spring.datasource.username", postgres::getUsername)
            registry.add("spring.datasource.password", postgres::getPassword)
        }
    }
}

// Integration test ใช้ real PostgreSQL
class UserIntegrationTest : BaseIntegrationTest() {
    
    @Autowired private lateinit var userService: UserService
    @Autowired private lateinit var userRepository: UserRepository
    
    @Test
    @Transactional
    fun `full user lifecycle`() {
        // Create
        val user = userService.createUser("Alice", "alice@example.com", "Pass123")
        assertThat(user.id).isGreaterThan(0)
        
        // Read
        val found = userService.getUser(user.id)
        assertThat(found.email).isEqualTo("alice@example.com")
        
        // Update
        val updated = userService.updateUser(user.id, UpdateUserRequest(name = "Alice Smith"))
        assertThat(updated.name).isEqualTo("Alice Smith")
        
        // Delete
        userService.deleteUser(user.id)
        assertThat(userRepository.existsById(user.id)).isFalse
    }
}
```

---

## 📊 6. Test Coverage

```kotlin
// build.gradle.kts - Jacoco coverage
plugins {
    jacoco
}

tasks.test {
    finalizedBy(tasks.jacocoTestReport)
}

tasks.jacocoTestReport {
    reports {
        xml.required = true
        html.required = true
    }
}

tasks.jacocoTestCoverageVerification {
    violationRules {
        rule {
            limit {
                minimum = "0.80".toBigDecimal()  // 80% coverage required
            }
        }
    }
}

// รัน tests
// ./gradlew test
// ./gradlew test jacocoTestReport  (สร้าง HTML report)
// ./gradlew check  (รัน tests + coverage verification)
```

---

## 📝 สรุป Part 29

| แนวคิด | รายละเอียด |
|--------|-----------|
| `@ExtendWith(MockKExtension::class)` | Unit tests + MockK |
| `@WebMvcTest` | Controller layer test |
| `@DataJpaTest` | Repository layer test |
| `@SpringBootTest` | Full integration test |
| `@Testcontainers` | Real database containers |
| MockMvc | HTTP request simulation |
| Jacoco | Code coverage reports |

---

## ➡️ ถัดไป: Part 30 - Docker และ Deployment

---
*Part 29/100+ | Kotlin & Spring Boot Complete Course*
