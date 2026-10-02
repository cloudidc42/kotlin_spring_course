# Part 28: JWT Authentication
## JSON Web Token กับ Spring Boot + Kotlin

---

## 🎯 เป้าหมายของ Part นี้

- เข้าใจ JWT (JSON Web Token)
- สร้าง JWT Authentication ครบวงจร
- Access Token + Refresh Token
- Stateless Authentication
- ตัวอย่าง: Login / Register / Protected API

---

## 📦 1. Dependencies

```kotlin
// build.gradle.kts
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-security")
    implementation("io.jsonwebtoken:jjwt-api:0.12.3")
    runtimeOnly("io.jsonwebtoken:jjwt-impl:0.12.3")
    runtimeOnly("io.jsonwebtoken:jjwt-jackson:0.12.3")
}
```

---

## 🔑 2. JWT Utility

```kotlin
// security/JwtService.kt
import io.jsonwebtoken.Claims
import io.jsonwebtoken.Jwts
import io.jsonwebtoken.security.Keys
import org.springframework.beans.factory.annotation.Value
import org.springframework.security.core.userdetails.UserDetails
import org.springframework.stereotype.Service
import java.util.Date
import javax.crypto.SecretKey

@Service
class JwtService(
    @Value("\${app.security.jwt-secret}") private val secret: String,
    @Value("\${app.security.jwt-expiration-ms:3600000}") private val expirationMs: Long,
    @Value("\${app.security.refresh-expiration-ms:604800000}") private val refreshExpirationMs: Long
) {
    private val signingKey: SecretKey by lazy {
        Keys.hmacShaKeyFor(secret.toByteArray())
    }
    
    fun generateAccessToken(userDetails: UserDetails): String =
        buildToken(userDetails, expirationMs, mapOf("type" to "access"))
    
    fun generateRefreshToken(userDetails: UserDetails): String =
        buildToken(userDetails, refreshExpirationMs, mapOf("type" to "refresh"))
    
    private fun buildToken(
        userDetails: UserDetails,
        expiration: Long,
        extraClaims: Map<String, Any> = emptyMap()
    ): String {
        return Jwts.builder()
            .subject(userDetails.username)
            .claims(extraClaims)
            .issuedAt(Date())
            .expiration(Date(System.currentTimeMillis() + expiration))
            .signWith(signingKey)
            .compact()
    }
    
    fun extractUsername(token: String): String? =
        extractClaim(token, Claims::getSubject)
    
    fun isTokenValid(token: String, userDetails: UserDetails): Boolean {
        val username = extractUsername(token) ?: return false
        return username == userDetails.username && !isTokenExpired(token)
    }
    
    fun isTokenExpired(token: String): Boolean =
        extractClaim(token, Claims::getExpiration)?.before(Date()) ?: true
    
    private fun <T> extractClaim(token: String, claimsResolver: (Claims) -> T): T? {
        return runCatching {
            val claims = Jwts.parser()
                .verifyWith(signingKey)
                .build()
                .parseSignedClaims(token)
                .payload
            claimsResolver(claims)
        }.getOrNull()
    }
}
```

---

## 🔒 3. JWT Filter

```kotlin
// security/JwtAuthFilter.kt
import jakarta.servlet.FilterChain
import jakarta.servlet.http.HttpServletRequest
import jakarta.servlet.http.HttpServletResponse
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken
import org.springframework.security.core.context.SecurityContextHolder
import org.springframework.security.core.userdetails.UserDetailsService
import org.springframework.security.web.authentication.WebAuthenticationDetailsSource
import org.springframework.stereotype.Component
import org.springframework.web.filter.OncePerRequestFilter

@Component
class JwtAuthFilter(
    private val jwtService: JwtService,
    private val userDetailsService: UserDetailsService
) : OncePerRequestFilter() {
    
    override fun doFilterInternal(
        request: HttpServletRequest,
        response: HttpServletResponse,
        filterChain: FilterChain
    ) {
        val authHeader = request.getHeader("Authorization")
        
        // ข้าม filter ถ้าไม่มี Bearer token
        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            filterChain.doFilter(request, response)
            return
        }
        
        val token = authHeader.substring(7)  // ตัด "Bearer "
        val username = jwtService.extractUsername(token)
        
        // Authenticate ถ้ายังไม่ได้ auth
        if (username != null && SecurityContextHolder.getContext().authentication == null) {
            val userDetails = userDetailsService.loadUserByUsername(username)
            
            if (jwtService.isTokenValid(token, userDetails)) {
                val authToken = UsernamePasswordAuthenticationToken(
                    userDetails,
                    null,
                    userDetails.authorities
                )
                authToken.details = WebAuthenticationDetailsSource().buildDetails(request)
                SecurityContextHolder.getContext().authentication = authToken
            }
        }
        
        filterChain.doFilter(request, response)
    }
}
```

---

## ⚙️ 4. Security Config กับ JWT

```kotlin
// config/SecurityConfig.kt
@Configuration
@EnableWebSecurity
@EnableMethodSecurity
class SecurityConfig(
    private val jwtAuthFilter: JwtAuthFilter,
    private val userDetailsService: UserDetailsServiceImpl
) {
    
    @Bean
    fun securityFilterChain(http: HttpSecurity): SecurityFilterChain {
        http
            .csrf { it.disable() }
            .cors { }  // ใช้ CorsConfig bean
            .sessionManagement { it.sessionCreationPolicy(SessionCreationPolicy.STATELESS) }
            .authorizeHttpRequests { auth ->
                auth
                    .requestMatchers("/api/auth/**").permitAll()
                    .requestMatchers("/actuator/health").permitAll()
                    .anyRequest().authenticated()
            }
            .authenticationProvider(authenticationProvider())
            .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter::class.java)
        
        return http.build()
    }
    
    @Bean
    fun authenticationProvider(): AuthenticationProvider {
        val provider = DaoAuthenticationProvider()
        provider.setUserDetailsService(userDetailsService)
        provider.setPasswordEncoder(passwordEncoder())
        return provider
    }
    
    @Bean
    fun authenticationManager(config: AuthenticationConfiguration): AuthenticationManager =
        config.authenticationManager
    
    @Bean
    fun passwordEncoder(): PasswordEncoder = BCryptPasswordEncoder()
}
```

---

## 🎮 5. Auth Controller

```kotlin
// DTOs
data class LoginRequest(val username: String, val password: String)
data class RegisterRequest(val username: String, val password: String, val email: String)
data class AuthResponse(
    val accessToken: String,
    val refreshToken: String,
    val tokenType: String = "Bearer",
    val expiresIn: Long = 3600
)
data class RefreshTokenRequest(val refreshToken: String)

// controller/AuthController.kt
@RestController
@RequestMapping("/api/auth")
class AuthController(
    private val authService: AuthService,
    private val jwtService: JwtService,
    private val authManager: AuthenticationManager,
    private val userDetailsService: UserDetailsService
) {
    
    @PostMapping("/register")
    fun register(@Valid @RequestBody request: RegisterRequest): ResponseEntity<AuthResponse> {
        val user = authService.register(request.username, request.password, request.email)
        val tokens = generateTokens(user)
        return ResponseEntity.status(HttpStatus.CREATED).body(tokens)
    }
    
    @PostMapping("/login")
    fun login(@Valid @RequestBody request: LoginRequest): ResponseEntity<AuthResponse> {
        // Throws if credentials invalid
        authManager.authenticate(
            UsernamePasswordAuthenticationToken(request.username, request.password)
        )
        
        val userDetails = userDetailsService.loadUserByUsername(request.username)
        val tokens = generateTokens(userDetails)
        return ResponseEntity.ok(tokens)
    }
    
    @PostMapping("/refresh")
    fun refreshToken(@RequestBody request: RefreshTokenRequest): ResponseEntity<AuthResponse> {
        val username = jwtService.extractUsername(request.refreshToken)
            ?: return ResponseEntity.status(HttpStatus.UNAUTHORIZED).build()
        
        val userDetails = userDetailsService.loadUserByUsername(username)
        
        if (!jwtService.isTokenValid(request.refreshToken, userDetails)) {
            return ResponseEntity.status(HttpStatus.UNAUTHORIZED).build()
        }
        
        return ResponseEntity.ok(generateTokens(userDetails))
    }
    
    private fun generateTokens(userDetails: UserDetails) = AuthResponse(
        accessToken = jwtService.generateAccessToken(userDetails),
        refreshToken = jwtService.generateRefreshToken(userDetails)
    )
}
```

---

## 📋 6. application.yml

```yaml
app:
  security:
    jwt-secret: "your-256-bit-secret-key-here-must-be-at-least-32-chars"
    jwt-expiration-ms: 3600000       # 1 hour
    refresh-expiration-ms: 604800000  # 7 days
```

---

## 🧪 7. Testing JWT

```bash
# Register
curl -X POST http://localhost:8080/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{"username":"alice","password":"Pass123!","email":"alice@example.com"}'
# Response:
# {"accessToken":"eyJ...","refreshToken":"eyJ...","tokenType":"Bearer","expiresIn":3600}

# Login
curl -X POST http://localhost:8080/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"alice","password":"Pass123!"}'

# ใช้ Token
ACCESS_TOKEN="eyJ..."
curl http://localhost:8080/api/users/me \
  -H "Authorization: Bearer $ACCESS_TOKEN"

# Refresh Token
curl -X POST http://localhost:8080/api/auth/refresh \
  -H "Content-Type: application/json" \
  -d '{"refreshToken":"eyJ..."}'
```

---

## 📝 สรุป Part 28

| แนวคิด | รายละเอียด |
|--------|-----------|
| JWT | stateless token authentication |
| `JwtService` | generate และ validate tokens |
| `JwtAuthFilter` | ตรวจสอบ token ทุก request |
| Access Token | short-lived (1 hr) |
| Refresh Token | long-lived (7 days) |
| `SecurityContextHolder` | เก็บ current user |

---

## ➡️ ถัดไป: Part 29 - Testing in Spring Boot

---
*Part 28/100+ | Kotlin & Spring Boot Complete Course*
