# Part 84: Security Penetration Testing
## OWASP Testing และ Security Code Review

---

## 🎯 เป้าหมายของ Part นี้

- เข้าใจ OWASP Top 10
- ค้นหา vulnerabilities ทั่วไปในโค้ด
- ทำ Security Code Review
- Dependency Vulnerability Scanning
- สร้าง Security Audit Checklist

---

## 📖 1. OWASP Top 10 (2021)

OWASP (Open Web Application Security Project) เป็นองค์กรที่ publish list ของ vulnerabilities ที่พบบ่อยที่สุด

```
OWASP Top 10 - 2021:
A01: Broken Access Control       ← อันตรายที่สุด
A02: Cryptographic Failures
A03: Injection
A04: Insecure Design
A05: Security Misconfiguration
A06: Vulnerable & Outdated Components
A07: Identification & Authentication Failures
A08: Software & Data Integrity Failures
A09: Security Logging & Monitoring Failures
A10: Server-Side Request Forgery (SSRF)
```

---

## 🔓 2. A01: Broken Access Control

### ตัวอย่างโค้ดที่มี Vulnerability

```kotlin
// ❌ VULNERABLE: ไม่ check authorization
@GetMapping("/users/{id}/profile")
fun getUserProfile(@PathVariable id: Long): UserProfile {
    // ใคร request มาก็ดูได้ทุก user!
    return userService.getProfile(id)
}

// ❌ VULNERABLE: ใช้ predictable IDs
@GetMapping("/orders/{id}")
fun getOrder(@PathVariable id: Long): Order {
    // User A สามารถดู Order ของ User B ได้โดย guess ID
    return orderService.getOrder(id)
}
```

### โค้ดที่ปลอดภัย

```kotlin
// ✅ SECURE: Check authorization
@GetMapping("/users/{id}/profile")
@PreAuthorize("hasRole('ADMIN') or #id == authentication.principal.id")
fun getUserProfile(
    @PathVariable id: Long,
    @AuthenticationPrincipal currentUser: UserDetails
): UserProfile {
    // ตรวจสอบว่า request user ตรงกับ profile ที่ขอ
    val profile = userService.getProfile(id)
    if (!currentUser.hasRole("ADMIN") && profile.userId != currentUser.id) {
        throw AccessDeniedException("Cannot access other user's profile")
    }
    return profile
}

// ✅ SECURE: Filter by current user
@GetMapping("/orders/{id}")
fun getOrder(
    @PathVariable id: Long,
    @AuthenticationPrincipal currentUser: UserDetails
): Order {
    val order = orderService.getOrder(id)

    // ตรวจสอบว่า order เป็นของ user คนนี้
    if (order.userId != currentUser.id && !currentUser.hasRole("ADMIN")) {
        throw AccessDeniedException("Order $id does not belong to current user")
    }
    return order
}
```

---

## 💉 3. A03: SQL Injection

### Vulnerable Code

```kotlin
// ❌ VULNERABLE: SQL Injection
@Repository
class UserRepositoryImpl(
    private val jdbcTemplate: JdbcTemplate
) {
    fun findByUsername(username: String): User? {
        // อันตราย! String concatenation
        val sql = "SELECT * FROM users WHERE username = '$username'"
        return jdbcTemplate.queryForObject(sql, userRowMapper)
    }

    // Input: ' OR '1'='1
    // SQL becomes: SELECT * FROM users WHERE username = '' OR '1'='1'
    // ผลลัพธ์: คืนทุก user!
}
```

### Secure Code

```kotlin
// ✅ SECURE: Parameterized queries
@Repository
class UserRepositoryImpl(
    private val jdbcTemplate: JdbcTemplate
) {
    fun findByUsername(username: String): User? {
        // ปลอดภัย: ใช้ placeholder ?
        val sql = "SELECT * FROM users WHERE username = ?"
        return jdbcTemplate.queryForObject(sql, userRowMapper, username)
    }
}

// ✅ SECURE: ใช้ Spring Data JPA (ปลอดภัยโดย default)
@Repository
interface UserRepository : JpaRepository<User, Long> {
    fun findByUsername(username: String): User?

    // JPQL ก็ปลอดภัย
    @Query("SELECT u FROM User u WHERE u.email = :email")
    fun findByEmail(@Param("email") email: String): User?
}
```

---

## 🔐 4. A02: Cryptographic Failures

### Vulnerable Code

```kotlin
// ❌ VULNERABLE: Weak encryption
object WeakCrypto {
    fun hashPassword(password: String): String {
        // MD5 ถูก crack ได้ง่ายมาก!
        val md = java.security.MessageDigest.getInstance("MD5")
        return md.digest(password.toByteArray()).joinToString("") {
            "%02x".format(it)
        }
    }

    // ❌ Storing plain passwords
    fun storeUser(username: String, password: String) {
        database.save(username, password) // เก็บ plain text!
    }
}
```

### Secure Code

```kotlin
// ✅ SECURE: BCrypt password hashing
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder

@Service
class UserSecurityService {
    private val passwordEncoder = BCryptPasswordEncoder(12) // cost factor 12

    fun hashPassword(password: String): String {
        return passwordEncoder.encode(password)
    }

    fun verifyPassword(rawPassword: String, hashedPassword: String): Boolean {
        return passwordEncoder.matches(rawPassword, hashedPassword)
    }
}

// ✅ SECURE: AES encryption for sensitive data
import javax.crypto.Cipher
import javax.crypto.SecretKeyFactory
import javax.crypto.spec.IvParameterSpec
import javax.crypto.spec.PBEKeySpec
import javax.crypto.spec.SecretKeySpec
import java.security.SecureRandom
import java.util.Base64

@Service
class EncryptionService(
    @Value("\${encryption.secret}") private val secret: String
) {
    private val SALT = "RandomSaltValue123".toByteArray()
    private val ITERATIONS = 65536
    private val KEY_LENGTH = 256
    private val ALGORITHM = "AES/CBC/PKCS5Padding"

    fun encrypt(data: String): String {
        val factory = SecretKeyFactory.getInstance("PBKDF2WithHmacSHA256")
        val spec = PBEKeySpec(secret.toCharArray(), SALT, ITERATIONS, KEY_LENGTH)
        val keyBytes = factory.generateSecret(spec).encoded
        val keySpec = SecretKeySpec(keyBytes, "AES")

        val iv = ByteArray(16)
        SecureRandom().nextBytes(iv)
        val ivSpec = IvParameterSpec(iv)

        val cipher = Cipher.getInstance(ALGORITHM)
        cipher.init(Cipher.ENCRYPT_MODE, keySpec, ivSpec)
        val encrypted = cipher.doFinal(data.toByteArray())

        val combined = iv + encrypted
        return Base64.getEncoder().encodeToString(combined)
    }
}
```

---

## 🚫 5. A07: Broken Authentication

### Vulnerable Code

```kotlin
// ❌ VULNERABLE: Weak JWT configuration
@Bean
fun jwtDecoder(): JwtDecoder {
    // ❌ Hard-coded secret
    return NimbusJwtDecoder.withSecretKey(
        SecretKeySpec("weak-secret-123".toByteArray(), "HmacSHA256")
    ).build()
}

// ❌ VULNERABLE: No rate limiting on login
@PostMapping("/login")
fun login(@RequestBody credentials: LoginRequest): TokenResponse {
    // Brute force attack possible!
    return authService.authenticate(credentials)
}
```

### Secure Code

```kotlin
// ✅ SECURE: Strong JWT with proper configuration
@Configuration
class SecurityConfig {

    @Value("\${jwt.secret}") // จาก environment variable
    private lateinit var jwtSecret: String

    @Bean
    fun jwtDecoder(): JwtDecoder {
        // ใช้ secret อย่างน้อย 256 bits
        val secretBytes = Base64.getDecoder().decode(jwtSecret)
        require(secretBytes.size >= 32) { "JWT secret must be at least 256 bits" }

        return NimbusJwtDecoder.withSecretKey(
            SecretKeySpec(secretBytes, "HmacSHA256")
        ).build()
    }
}

// ✅ SECURE: Rate limiting on authentication
@RestController
@RequestMapping("/auth")
class AuthController(
    private val authService: AuthService,
    private val rateLimiter: RateLimiter
) {
    @PostMapping("/login")
    fun login(
        @RequestBody credentials: LoginRequest,
        request: HttpServletRequest
    ): ResponseEntity<TokenResponse> {
        val clientIp = request.remoteAddr

        // Rate limit: 5 attempts per minute per IP
        if (!rateLimiter.tryAcquire(clientIp)) {
            throw TooManyRequestsException("Too many login attempts. Try again later.")
        }

        return try {
            ResponseEntity.ok(authService.authenticate(credentials))
        } catch (e: BadCredentialsException) {
            rateLimiter.recordFailure(clientIp)
            throw e
        }
    }
}
```

---

## 🔍 6. A09: Security Logging Failures

### Vulnerable Code

```kotlin
// ❌ VULNERABLE: Logging sensitive data
@Service
class AuthService {
    private val log = LoggerFactory.getLogger(AuthService::class.java)

    fun authenticate(username: String, password: String): User {
        log.info("Login attempt: username=$username, password=$password") // อันตราย!
        // ...
    }
}

// ❌ VULNERABLE: No security event logging
fun deleteUser(id: Long) {
    userRepository.deleteById(id) // ไม่มี audit log!
}
```

### Secure Code

```kotlin
// ✅ SECURE: Proper security logging
@Service
class AuthService {
    private val securityLog = LoggerFactory.getLogger("SECURITY")

    fun authenticate(username: String, password: String): User {
        // อย่า log password!
        securityLog.info("Login attempt: username={}", username)

        return try {
            val user = findUser(username)
            verifyPassword(password, user.passwordHash)
            securityLog.info("Login success: username={}, userId={}", username, user.id)
            user
        } catch (e: Exception) {
            // Log failure แต่ไม่บอกรายละเอียดว่า username ผิดหรือ password ผิด
            securityLog.warn("Login failed: username={}, reason=invalid_credentials", username)
            throw BadCredentialsException("Invalid credentials")
        }
    }
}

// ✅ SECURE: Audit logging
@Component
class AuditLogger {
    private val auditLog = LoggerFactory.getLogger("AUDIT")

    fun logAction(
        userId: Long,
        action: String,
        resource: String,
        resourceId: Any?,
        success: Boolean,
        details: String? = null
    ) {
        auditLog.info(
            "AUDIT: userId={} action={} resource={} resourceId={} success={} details={}",
            userId, action, resource, resourceId, success, details
        )
    }
}

@Service
class UserService(
    private val userRepository: UserRepository,
    private val auditLogger: AuditLogger
) {
    fun deleteUser(id: Long, requestingUserId: Long) {
        val user = userRepository.findById(id)
            .orElseThrow { NoSuchElementException("User $id not found") }

        userRepository.deleteById(id)

        auditLogger.logAction(
            userId = requestingUserId,
            action = "DELETE_USER",
            resource = "User",
            resourceId = id,
            success = true,
            details = "Deleted user: ${user.email}"
        )
    }
}
```

---

## 🔧 7. Dependency Vulnerability Scanning

### OWASP Dependency Check

```kotlin
// build.gradle.kts
plugins {
    id("org.owasp.dependencycheck") version "9.0.7"
}

dependencyCheck {
    failBuildOnCVSS = 7.0f  // Fail if CVSS score >= 7.0
    format = "ALL"           // HTML, XML, JSON, CSV reports
    outputDirectory = "build/reports/dependency-check"

    suppressionFile = "owasp-suppressions.xml"

    nvd {
        apiKey = System.getenv("NVD_API_KEY") ?: ""
    }
}
```

```bash
# Run dependency check
./gradlew dependencyCheckAnalyze

# ดู report
open build/reports/dependency-check/dependency-check-report.html
```

### Snyk Integration

```bash
# ติดตั้ง Snyk CLI
npm install -g snyk

# authenticate
snyk auth

# scan dependencies
snyk test --all-sub-projects

# monitor continuously
snyk monitor

# fix vulnerabilities
snyk fix
```

---

## 📋 8. Security Audit Checklist

```markdown
# Security Audit Checklist

## Authentication & Authorization
- [ ] ใช้ BCrypt หรือ Argon2 สำหรับ password hashing
- [ ] JWT expiration ไม่เกิน 1 hour
- [ ] Refresh token rotation
- [ ] Rate limiting บน login endpoint
- [ ] Account lockout หลัง failed attempts
- [ ] @PreAuthorize บน sensitive endpoints
- [ ] Row-level security (users เห็นแค่ data ของตัวเอง)

## Input Validation
- [ ] Validate ทุก input ด้วย @Valid
- [ ] ไม่ใช้ string concatenation ใน SQL
- [ ] Sanitize HTML input (ป้องกัน XSS)
- [ ] File upload validation (type, size)
- [ ] Path traversal protection

## Cryptography
- [ ] ไม่ใช้ MD5, SHA1 สำหรับ password
- [ ] SSL/TLS 1.2+ เท่านั้น
- [ ] Secrets อยู่ใน environment variables ไม่ใน code
- [ ] Sensitive data encrypted at rest

## HTTP Security Headers
- [ ] Content-Security-Policy
- [ ] X-Frame-Options: DENY
- [ ] X-Content-Type-Options: nosniff
- [ ] Strict-Transport-Security
- [ ] Referrer-Policy

## Logging & Monitoring
- [ ] Log security events (login, logout, failed auth)
- [ ] ไม่ log passwords หรือ tokens
- [ ] Centralized log management
- [ ] Alerting บน suspicious activities

## Dependencies
- [ ] OWASP Dependency Check ใน CI/CD
- [ ] Regular dependency updates
- [ ] No known CVEs
```

### Spring Security Headers Configuration

```kotlin
// src/main/kotlin/com/example/config/SecurityHeadersConfig.kt
package com.example.config

import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.security.config.annotation.web.builders.HttpSecurity
import org.springframework.security.web.SecurityFilterChain

@Configuration
class SecurityHeadersConfig {

    @Bean
    fun securityFilterChain(http: HttpSecurity): SecurityFilterChain {
        http
            .headers { headers ->
                headers
                    .frameOptions { it.deny() }
                    .xssProtection { it.headerValue(XXssProtectionHeaderWriter.HeaderValue.ENABLED_MODE_BLOCK) }
                    .contentTypeOptions { }
                    .httpStrictTransportSecurity { hsts ->
                        hsts.includeSubDomains(true)
                            .maxAgeInSeconds(31536000)
                    }
                    .contentSecurityPolicy { csp ->
                        csp.policyDirectives(
                            "default-src 'self'; " +
                            "script-src 'self'; " +
                            "style-src 'self' 'unsafe-inline'; " +
                            "img-src 'self' data:; " +
                            "frame-ancestors 'none'"
                        )
                    }
                    .referrerPolicy { it.policy(ReferrerPolicyHeaderWriter.ReferrerPolicy.STRICT_ORIGIN_WHEN_CROSS_ORIGIN) }
            }

        return http.build()
    }
}
```

---

## 📋 สรุป

| Vulnerability | ความอันตราย | วิธีป้องกัน |
|---------------|------------|------------|
| Broken Access Control | สูงมาก | @PreAuthorize, row-level check |
| SQL Injection | สูงมาก | Parameterized queries, JPA |
| Weak Cryptography | สูง | BCrypt, AES-256 |
| Broken Authentication | สูง | Rate limiting, strong JWT |
| Security Logging | ปานกลาง | Audit logs, SIEM |
| Outdated Dependencies | ปานกลาง | OWASP Dependency Check |

---

*Part 84/100+ | Kotlin & Spring Boot Complete Course*
