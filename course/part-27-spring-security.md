# Part 27: Spring Security
## Authentication & Authorization ใน Spring Boot + Kotlin

---

## 🎯 เป้าหมายของ Part นี้

- เข้าใจ Spring Security architecture
- ตั้งค่า Security configuration
- In-memory authentication
- Database authentication
- Authorization: roles และ permissions
- ตัวอย่าง: Basic Auth + Role-based Access

---

## 📦 1. Dependencies

```kotlin
// build.gradle.kts
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-security")
    testImplementation("org.springframework.security:spring-security-test")
}
```

---

## 🏗️ 2. Security Architecture

```
Request Flow:
HTTP Request
    ↓
FilterChain (DelegatingFilterProxy)
    ↓
UsernamePasswordAuthenticationFilter
    ↓
AuthenticationManager
    ↓
AuthenticationProvider
    ↓
UserDetailsService
    ↓
UserDetails (ตรวจสอบ credentials)
    ↓
SecurityContext (เก็บ Authentication)
    ↓
Controller
```

---

## ⚙️ 3. Security Configuration

```kotlin
// config/SecurityConfig.kt
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.security.authentication.AuthenticationManager
import org.springframework.security.config.annotation.authentication.configuration.AuthenticationConfiguration
import org.springframework.security.config.annotation.method.configuration.EnableMethodSecurity
import org.springframework.security.config.annotation.web.builders.HttpSecurity
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity
import org.springframework.security.config.http.SessionCreationPolicy
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder
import org.springframework.security.crypto.password.PasswordEncoder
import org.springframework.security.web.SecurityFilterChain

@Configuration
@EnableWebSecurity
@EnableMethodSecurity  // เปิดใช้ @PreAuthorize, @PostAuthorize
class SecurityConfig {
    
    @Bean
    fun securityFilterChain(http: HttpSecurity): SecurityFilterChain {
        http
            .csrf { it.disable() }  // Disable สำหรับ REST API
            .sessionManagement { session ->
                session.sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            }
            .authorizeHttpRequests { auth ->
                auth
                    // Public endpoints
                    .requestMatchers("/api/auth/**").permitAll()
                    .requestMatchers("/actuator/health").permitAll()
                    .requestMatchers("/api/public/**").permitAll()
                    
                    // Admin only
                    .requestMatchers("/api/admin/**").hasRole("ADMIN")
                    
                    // Authenticated users
                    .anyRequest().authenticated()
            }
            .httpBasic { }  // HTTP Basic Auth
        
        return http.build()
    }
    
    @Bean
    fun passwordEncoder(): PasswordEncoder = BCryptPasswordEncoder()
    
    @Bean
    fun authenticationManager(config: AuthenticationConfiguration): AuthenticationManager =
        config.authenticationManager
}
```

---

## 👤 4. UserDetails กับ Database

```kotlin
// domain/User.kt - ต้อง implement UserDetails
import org.springframework.security.core.GrantedAuthority
import org.springframework.security.core.authority.SimpleGrantedAuthority
import org.springframework.security.core.userdetails.UserDetails

@Entity
@Table(name = "users")
class User(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long = 0,
    
    @Column(unique = true, nullable = false)
    private val username: String,
    
    @Column(nullable = false)
    private var password: String,
    
    @Enumerated(EnumType.STRING)
    val role: Role = Role.USER,
    
    var enabled: Boolean = true
) : UserDetails {
    
    enum class Role { USER, ADMIN, MODERATOR }
    
    override fun getUsername() = username
    override fun getPassword() = password
    override fun isEnabled() = enabled
    override fun isAccountNonExpired() = true
    override fun isAccountNonLocked() = true
    override fun isCredentialsNonExpired() = true
    
    override fun getAuthorities(): Collection<GrantedAuthority> =
        listOf(SimpleGrantedAuthority("ROLE_${role.name}"))
    
    fun updatePassword(newPassword: String) { password = newPassword }
}

// service/UserDetailsServiceImpl.kt
@Service
class UserDetailsServiceImpl(private val userRepo: UserRepository) : UserDetailsService {
    
    override fun loadUserByUsername(username: String): UserDetails {
        return userRepo.findByUsername(username)
            ?: throw UsernameNotFoundException("User not found: $username")
    }
}
```

---

## 🔐 5. In-Memory Users (Development)

```kotlin
@Configuration
class DevSecurityConfig {
    
    @Bean
    @Profile("dev")
    fun inMemoryUsers(encoder: PasswordEncoder): UserDetailsService {
        val userDetails = User
            .withUsername("user")
            .password(encoder.encode("password"))
            .roles("USER")
            .build()
        
        val adminDetails = User
            .withUsername("admin")
            .password(encoder.encode("admin123"))
            .roles("ADMIN", "USER")
            .build()
        
        return InMemoryUserDetailsManager(userDetails, adminDetails)
    }
}
```

---

## 🛡️ 6. Method Security

```kotlin
import org.springframework.security.access.prepost.PreAuthorize
import org.springframework.security.access.prepost.PostAuthorize
import org.springframework.security.core.annotation.AuthenticationPrincipal

@RestController
@RequestMapping("/api/users")
class UserController(private val userService: UserService) {
    
    // ต้อง login ก่อน
    @GetMapping
    @PreAuthorize("isAuthenticated()")
    fun getUsers(): List<UserResponse> = userService.getAllUsers()
    
    // เฉพาะ ADMIN เท่านั้น
    @DeleteMapping("/{id}")
    @PreAuthorize("hasRole('ADMIN')")
    fun deleteUser(@PathVariable id: Long): ResponseEntity<Void> {
        userService.deleteUser(id)
        return ResponseEntity.noContent().build()
    }
    
    // User ดูได้แค่ของตัวเอง หรือ ADMIN ดูได้ทุกคน
    @GetMapping("/{id}")
    @PreAuthorize("hasRole('ADMIN') or #id == authentication.principal.id")
    fun getUser(
        @PathVariable id: Long,
        @AuthenticationPrincipal currentUser: User
    ): UserResponse {
        return userService.getUser(id)
    }
    
    // ดึงข้อมูล current user
    @GetMapping("/me")
    fun getMe(@AuthenticationPrincipal user: User): UserResponse {
        return user.toResponse()
    }
}
```

---

## 🔒 7. Password Encoding

```kotlin
@Service
class AuthService(
    private val userRepo: UserRepository,
    private val passwordEncoder: PasswordEncoder
) {
    
    fun register(username: String, password: String): User {
        if (userRepo.existsByUsername(username)) {
            throw ConflictException("Username already taken")
        }
        
        val user = User(
            username = username,
            password = passwordEncoder.encode(password)  // BCrypt hash
        )
        return userRepo.save(user)
    }
    
    fun changePassword(userId: Long, oldPass: String, newPass: String): User {
        val user = userRepo.findById(userId).orElseThrow {
            NotFoundException("User not found", "User", userId)
        }
        
        // ตรวจสอบ password เก่า
        if (!passwordEncoder.matches(oldPass, user.password)) {
            throw BadRequestException("Wrong current password")
        }
        
        user.updatePassword(passwordEncoder.encode(newPass))
        return userRepo.save(user)
    }
}
```

---

## 📊 8. Security Testing

```kotlin
@WebMvcTest(UserController::class)
@Import(SecurityConfig::class)
class UserControllerSecurityTest {
    
    @Autowired private lateinit var mockMvc: MockMvc
    @MockBean private lateinit var userService: UserService
    @MockBean private lateinit var userDetailsService: UserDetailsServiceImpl
    
    @Test
    fun `GET users without auth should return 401`() {
        mockMvc.get("/api/users")
            .andExpect { status { isUnauthorized() } }
    }
    
    @Test
    @WithMockUser(roles = ["USER"])
    fun `GET users with auth should return 200`() {
        every { userService.getAllUsers() } returns emptyList()
        
        mockMvc.get("/api/users")
            .andExpect { status { isOk() } }
    }
    
    @Test
    @WithMockUser(roles = ["USER"])
    fun `DELETE user as USER should return 403`() {
        mockMvc.delete("/api/users/1")
            .andExpect { status { isForbidden() } }
    }
    
    @Test
    @WithMockUser(roles = ["ADMIN"])
    fun `DELETE user as ADMIN should return 204`() {
        every { userService.deleteUser(1L) } just Runs
        
        mockMvc.delete("/api/users/1")
            .andExpect { status { isNoContent() } }
    }
    
    @Test
    fun `GET with basic auth should return 200`() {
        val details = User
            .withUsername("alice")
            .password("{noop}password")
            .roles("USER")
            .build()
        
        every { userDetailsService.loadUserByUsername("alice") } returns details
        every { userService.getAllUsers() } returns emptyList()
        
        mockMvc.get("/api/users") {
            with(httpBasic("alice", "password"))
        }.andExpect {
            status { isOk() }
        }
    }
}
```

---

## 🌐 9. CORS Configuration

```kotlin
@Configuration
class WebConfig : WebMvcConfigurer {
    
    override fun addCorsMappings(registry: CorsRegistry) {
        registry.addMapping("/api/**")
            .allowedOrigins(
                "http://localhost:3000",
                "https://myapp.example.com"
            )
            .allowedMethods("GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS")
            .allowedHeaders("*")
            .allowCredentials(true)
            .maxAge(3600)
    }
}
```

---

## 📝 สรุป Part 27

| แนวคิด | รายละเอียด |
|--------|-----------|
| `@EnableWebSecurity` | เปิด Spring Security |
| `SecurityFilterChain` | ตั้งค่า security rules |
| `UserDetails` | User object สำหรับ Security |
| `UserDetailsService` | โหลด user จาก DB |
| `BCryptPasswordEncoder` | Hash passwords |
| `@PreAuthorize` | Method-level security |
| `@AuthenticationPrincipal` | รับ current user |
| `@WithMockUser` | Testing security |

---

## ➡️ ถัดไป: Part 28 - JWT Authentication

---
*Part 27/100+ | Kotlin & Spring Boot Complete Course*
