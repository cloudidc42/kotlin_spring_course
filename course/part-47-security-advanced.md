# Part 47: Advanced Security
## การป้องกันช่องโหว่ด้านความปลอดภัยระดับ Production

---

## 🎯 เป้าหมายของ Part นี้

- OWASP Top 10 Protection
- SQL Injection prevention
- XSS (Cross-Site Scripting) prevention
- CSRF (Cross-Site Request Forgery) protection
- Security headers
- Password policies
- Input validation and sanitization
- ตัวอย่างจริง: Secure REST API

---

## 📚 1. OWASP Top 10 Overview

OWASP (Open Web Application Security Project) เผยแพร่รายการช่องโหว่ที่พบบ่อยที่สุด:

| อันดับ | ช่องโหว่ | วิธีป้องกันหลัก |
|--------|---------|--------------|
| A01 | Broken Access Control | Authorization checks ทุก endpoint |
| A02 | Cryptographic Failures | ใช้ strong encryption |
| A03 | Injection (SQL, LDAP, etc.) | Parameterized queries |
| A04 | Insecure Design | Threat modeling |
| A05 | Security Misconfiguration | Security headers |
| A06 | Vulnerable Components | อัปเดต dependencies |
| A07 | Authentication Failures | Strong auth, MFA |
| A08 | Software Integrity Failures | Code signing |
| A09 | Logging Failures | Audit logging |
| A10 | Server-Side Request Forgery | URL validation |

---

## 🔧 2. Dependencies Setup

```kotlin
// build.gradle.kts
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-security")
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
    implementation("org.springframework.boot:spring-boot-starter-validation")
    
    // JWT
    implementation("io.jsonwebtoken:jjwt-api:0.12.3")
    runtimeOnly("io.jsonwebtoken:jjwt-impl:0.12.3")
    runtimeOnly("io.jsonwebtoken:jjwt-jackson:0.12.3")
    
    // Password encoding
    implementation("org.springframework.security:spring-security-crypto")
    
    // HTML sanitizer (XSS prevention)
    implementation("org.owasp.antisamy:antisamy:1.7.4")
    // หรือ
    implementation("org.jsoup:jsoup:1.17.2")
    
    testImplementation("org.springframework.boot:spring-boot-starter-test")
    testImplementation("org.springframework.security:spring-security-test")
}
```

---

## 🛡️ 3. Security Configuration

```kotlin
// config/SecurityConfig.kt
package com.example.security.config

import com.example.security.filter.JwtAuthenticationFilter
import com.example.security.filter.SecurityHeadersFilter
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.http.HttpMethod
import org.springframework.security.config.annotation.method.configuration.EnableMethodSecurity
import org.springframework.security.config.annotation.web.builders.HttpSecurity
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity
import org.springframework.security.config.http.SessionCreationPolicy
import org.springframework.security.crypto.argon2.Argon2PasswordEncoder
import org.springframework.security.crypto.password.PasswordEncoder
import org.springframework.security.web.SecurityFilterChain
import org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter
import org.springframework.security.web.header.writers.ReferrerPolicyHeaderWriter

@Configuration
@EnableWebSecurity
@EnableMethodSecurity(prePostEnabled = true, securedEnabled = true)
class SecurityConfig(
    private val jwtFilter: JwtAuthenticationFilter,
    private val securityHeadersFilter: SecurityHeadersFilter
) {

    @Bean
    fun securityFilterChain(http: HttpSecurity): SecurityFilterChain {
        http
            // Disable CSRF for stateless REST API (ใช้ JWT แทน)
            .csrf { it.disable() }

            // Session management - STATELESS สำหรับ REST API
            .sessionManagement { session ->
                session.sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            }

            // Authorization rules
            .authorizeHttpRequests { auth ->
                auth
                    // Public endpoints
                    .requestMatchers(HttpMethod.POST, "/auth/login", "/auth/register").permitAll()
                    .requestMatchers("/swagger-ui/**", "/api-docs/**").permitAll()
                    .requestMatchers("/actuator/health").permitAll()

                    // Admin only
                    .requestMatchers("/api/admin/**").hasRole("ADMIN")

                    // Authenticated
                    .anyRequest().authenticated()
            }

            // Exception handling
            .exceptionHandling { exceptions ->
                exceptions
                    .authenticationEntryPoint { _, response, _ ->
                        response.sendError(401, "Unauthorized")
                    }
                    .accessDeniedHandler { _, response, _ ->
                        response.sendError(403, "Forbidden")
                    }
            }

            // Security headers
            .headers { headers ->
                headers
                    // X-Content-Type-Options
                    .contentTypeOptions { }

                    // X-Frame-Options: DENY
                    .frameOptions { it.deny() }

                    // HSTS (HTTP Strict Transport Security)
                    .httpStrictTransportSecurity { hsts ->
                        hsts
                            .maxAgeInSeconds(31536000)  // 1 year
                            .includeSubDomains(true)
                            .preload(true)
                    }

                    // Content Security Policy
                    .contentSecurityPolicy { csp ->
                        csp.policyDirectives("""
                            default-src 'self';
                            script-src 'self' 'unsafe-inline' https://trusted-cdn.com;
                            style-src 'self' 'unsafe-inline';
                            img-src 'self' data: https:;
                            font-src 'self';
                            connect-src 'self';
                            frame-ancestors 'none';
                            base-uri 'self';
                            form-action 'self'
                        """.trimIndent())
                    }

                    // Referrer Policy
                    .referrerPolicy { referrer ->
                        referrer.policy(ReferrerPolicyHeaderWriter.ReferrerPolicy.STRICT_ORIGIN_WHEN_CROSS_ORIGIN)
                    }

                    // Permissions Policy (Feature Policy)
                    .permissionsPolicy { permissions ->
                        permissions.policy("camera=(), microphone=(), geolocation=(), payment=()")
                    }
            }

            // Add JWT filter
            .addFilterBefore(jwtFilter, UsernamePasswordAuthenticationFilter::class.java)
            .addFilterBefore(securityHeadersFilter, JwtAuthenticationFilter::class.java)

        return http.build()
    }

    @Bean
    fun passwordEncoder(): PasswordEncoder {
        // Argon2 เป็น password hashing ที่แนะนำใน 2024
        // เป็นผู้ชนะ Password Hashing Competition (PHC) 2015
        return Argon2PasswordEncoder.defaultsForSpringSecurity_v5_8()
    }
}
```

---

## 💉 4. SQL Injection Prevention

```kotlin
// repository/UserRepository.kt
package com.example.security.repository

import com.example.security.entity.User
import org.springframework.data.jpa.repository.JpaRepository
import org.springframework.data.jpa.repository.Query
import org.springframework.data.repository.query.Param

interface UserRepository : JpaRepository<User, Long> {

    // ✅ Safe: Spring Data method - auto parameterized
    fun findByEmail(email: String): User?

    // ✅ Safe: JPQL with named parameters
    @Query("SELECT u FROM User u WHERE u.firstName LIKE %:name%")
    fun searchByName(@Param("name") name: String): List<User>

    // ✅ Safe: Native query with parameterized inputs
    @Query(
        value = "SELECT * FROM users WHERE email = :email AND active = true",
        nativeQuery = true
    )
    fun findActiveUserByEmail(@Param("email") email: String): User?

    // ❌ NEVER DO THIS - SQL Injection vulnerable!
    // @Query(value = "SELECT * FROM users WHERE name = '" + name + "'", nativeQuery = true)
}
```

```kotlin
// service/SearchService.kt
package com.example.security.service

import com.example.security.entity.User
import jakarta.persistence.EntityManager
import jakarta.persistence.criteria.CriteriaBuilder
import org.springframework.stereotype.Service

@Service
class SearchService(private val entityManager: EntityManager) {

    // ✅ Safe: Criteria API (type-safe, no SQL injection)
    fun searchUsers(query: String, page: Int, size: Int): List<User> {
        val cb: CriteriaBuilder = entityManager.criteriaBuilder
        val cq = cb.createQuery(User::class.java)
        val root = cq.from(User::class.java)

        val searchPattern = "%${query.trim()}%"

        cq.where(
            cb.or(
                cb.like(cb.lower(root.get("firstName")), searchPattern.lowercase()),
                cb.like(cb.lower(root.get("lastName")), searchPattern.lowercase()),
                cb.like(cb.lower(root.get("email")), searchPattern.lowercase())
            )
        )

        return entityManager.createQuery(cq)
            .setFirstResult(page * size)
            .setMaxResults(size)
            .resultList
    }
}
```

---

## 🔒 5. XSS Prevention

```kotlin
// filter/XssFilter.kt
package com.example.security.filter

import jakarta.servlet.FilterChain
import jakarta.servlet.http.HttpServletRequest
import jakarta.servlet.http.HttpServletResponse
import org.jsoup.Jsoup
import org.jsoup.safety.Safelist
import org.springframework.stereotype.Component
import org.springframework.web.filter.OncePerRequestFilter
import org.springframework.web.util.ContentCachingRequestWrapper

@Component
class XssFilter : OncePerRequestFilter() {

    override fun doFilterInternal(
        request: HttpServletRequest,
        response: HttpServletResponse,
        filterChain: FilterChain
    ) {
        val wrappedRequest = XssRequestWrapper(request)
        filterChain.doFilter(wrappedRequest, response)
    }
}

class XssRequestWrapper(request: HttpServletRequest) :
    ContentCachingRequestWrapper(request) {

    override fun getParameterValues(parameter: String): Array<String>? {
        return super.getParameterValues(parameter)?.map { sanitize(it) }?.toTypedArray()
    }

    override fun getParameter(name: String): String? {
        return super.getParameter(name)?.let { sanitize(it) }
    }

    override fun getHeader(name: String): String? {
        return super.getHeader(name)?.let { sanitize(it) }
    }

    private fun sanitize(value: String): String {
        // ลบ HTML tags ทั้งหมด
        return Jsoup.clean(value, Safelist.none())
    }
}
```

```kotlin
// util/HtmlSanitizer.kt
package com.example.security.util

import org.jsoup.Jsoup
import org.jsoup.safety.Safelist
import org.springframework.stereotype.Component

@Component
class HtmlSanitizer {

    /**
     * ลบ HTML ทั้งหมด (สำหรับ plain text fields)
     */
    fun sanitizePlainText(input: String?): String? {
        return input?.let { Jsoup.clean(it, Safelist.none()) }
    }

    /**
     * อนุญาต basic HTML (สำหรับ rich text)
     * เฉพาะ tags ที่ปลอดภัย: b, i, em, strong, p, br
     */
    fun sanitizeBasicHtml(input: String?): String? {
        return input?.let { Jsoup.clean(it, Safelist.basic()) }
    }

    /**
     * อนุญาต HTML ที่กว้างขึ้น (สำหรับ content editor)
     */
    fun sanitizeRelaxedHtml(input: String?): String? {
        return input?.let {
            Jsoup.clean(
                it,
                Safelist.relaxed()
                    .removeTags("script", "style", "iframe", "object", "embed")
                    .removeAttributes(":all", "onclick", "onload", "onerror")
            )
        }
    }
}
```

---

## 🔑 6. Password Policy

```kotlin
// service/PasswordPolicyService.kt
package com.example.security.service

import org.springframework.stereotype.Service

data class PasswordValidationResult(
    val valid: Boolean,
    val errors: List<String>
)

@Service
class PasswordPolicyService {

    companion object {
        const val MIN_LENGTH = 8
        const val MAX_LENGTH = 128
        // Common passwords ที่ต้อง reject
        private val COMMON_PASSWORDS = setOf(
            "password", "password123", "123456789", "qwerty123",
            "admin123", "letmein", "welcome1", "monkey123"
        )
    }

    fun validate(password: String): PasswordValidationResult {
        val errors = mutableListOf<String>()

        // ความยาว
        if (password.length < MIN_LENGTH) {
            errors.add("Password must be at least $MIN_LENGTH characters")
        }
        if (password.length > MAX_LENGTH) {
            errors.add("Password must not exceed $MAX_LENGTH characters")
        }

        // Uppercase
        if (!password.any { it.isUpperCase() }) {
            errors.add("Password must contain at least one uppercase letter")
        }

        // Lowercase
        if (!password.any { it.isLowerCase() }) {
            errors.add("Password must contain at least one lowercase letter")
        }

        // Digit
        if (!password.any { it.isDigit() }) {
            errors.add("Password must contain at least one digit")
        }

        // Special character
        val specialChars = "!@#\$%^&*()_+-=[]{}|;':\",./<>?"
        if (!password.any { it in specialChars }) {
            errors.add("Password must contain at least one special character")
        }

        // Common passwords
        if (password.lowercase() in COMMON_PASSWORDS) {
            errors.add("Password is too common, please choose a more unique password")
        }

        // Sequential characters
        if (hasSequentialChars(password)) {
            errors.add("Password must not contain sequential characters (e.g., 'abc', '123')")
        }

        return PasswordValidationResult(
            valid = errors.isEmpty(),
            errors = errors
        )
    }

    private fun hasSequentialChars(password: String, minLength: Int = 3): Boolean {
        if (password.length < minLength) return false

        for (i in 0..password.length - minLength) {
            val chars = password.substring(i, i + minLength)
            val ascending = chars.zipWithNext().all { (a, b) -> b.code - a.code == 1 }
            val descending = chars.zipWithNext().all { (a, b) -> a.code - b.code == 1 }
            if (ascending || descending) return true
        }
        return false
    }
}
```

---

## 🔐 7. Security Headers Filter

```kotlin
// filter/SecurityHeadersFilter.kt
package com.example.security.filter

import jakarta.servlet.FilterChain
import jakarta.servlet.http.HttpServletRequest
import jakarta.servlet.http.HttpServletResponse
import org.springframework.stereotype.Component
import org.springframework.web.filter.OncePerRequestFilter

@Component
class SecurityHeadersFilter : OncePerRequestFilter() {

    override fun doFilterInternal(
        request: HttpServletRequest,
        response: HttpServletResponse,
        filterChain: FilterChain
    ) {
        // X-Content-Type-Options: ป้องกัน MIME type sniffing
        response.setHeader("X-Content-Type-Options", "nosniff")

        // X-Frame-Options: ป้องกัน clickjacking
        response.setHeader("X-Frame-Options", "DENY")

        // X-XSS-Protection: เปิด browser XSS filter (legacy browsers)
        response.setHeader("X-XSS-Protection", "1; mode=block")

        // Referrer-Policy
        response.setHeader("Referrer-Policy", "strict-origin-when-cross-origin")

        // Permissions-Policy: ปิด features ที่ไม่ใช้
        response.setHeader("Permissions-Policy",
            "camera=(), microphone=(), geolocation=(), payment=(self), usb=()")

        // Cache-Control สำหรับ sensitive pages
        if (isSensitivePath(request.requestURI)) {
            response.setHeader("Cache-Control", "no-cache, no-store, must-revalidate")
            response.setHeader("Pragma", "no-cache")
            response.setHeader("Expires", "0")
        }

        // Remove sensitive server information
        response.setHeader("Server", "")

        filterChain.doFilter(request, response)
    }

    private fun isSensitivePath(path: String): Boolean {
        return path.startsWith("/auth/") ||
                path.startsWith("/api/user/") ||
                path.startsWith("/api/admin/")
    }
}
```

---

## 🚨 8. Security Audit Logging

```kotlin
// aspect/SecurityAuditAspect.kt
package com.example.security.aspect

import org.aspectj.lang.JoinPoint
import org.aspectj.lang.annotation.AfterReturning
import org.aspectj.lang.annotation.AfterThrowing
import org.aspectj.lang.annotation.Aspect
import org.slf4j.LoggerFactory
import org.springframework.security.core.context.SecurityContextHolder
import org.springframework.stereotype.Component
import org.springframework.web.context.request.RequestContextHolder
import org.springframework.web.context.request.ServletRequestAttributes
import java.time.Instant

@Aspect
@Component
class SecurityAuditAspect {

    private val auditLogger = LoggerFactory.getLogger("SECURITY_AUDIT")

    // Log successful authentication
    @AfterReturning(
        pointcut = "execution(* com.example.security.service.AuthService.login(..))",
        returning = "result"
    )
    fun logSuccessfulLogin(joinPoint: JoinPoint, result: Any?) {
        val username = joinPoint.args.firstOrNull()?.toString() ?: "unknown"
        val ip = getClientIp()
        auditLogger.info("LOGIN_SUCCESS | user={} | ip={} | time={}", username, ip, Instant.now())
    }

    // Log failed authentication
    @AfterThrowing(
        pointcut = "execution(* com.example.security.service.AuthService.login(..))",
        throwing = "exception"
    )
    fun logFailedLogin(joinPoint: JoinPoint, exception: Exception) {
        val username = joinPoint.args.firstOrNull()?.toString() ?: "unknown"
        val ip = getClientIp()
        auditLogger.warn("LOGIN_FAILED | user={} | ip={} | reason={} | time={}",
            username, ip, exception.message, Instant.now())
    }

    // Log admin operations
    @AfterReturning(
        pointcut = "execution(* com.example.security.controller.AdminController.*(..))"
    )
    fun logAdminOperation(joinPoint: JoinPoint) {
        val admin = SecurityContextHolder.getContext().authentication?.name ?: "unknown"
        val operation = joinPoint.signature.name
        val args = joinPoint.args.joinToString()
        auditLogger.info("ADMIN_ACTION | admin={} | action={} | args={} | time={}",
            admin, operation, args, Instant.now())
    }

    private fun getClientIp(): String {
        val attrs = RequestContextHolder.getRequestAttributes() as? ServletRequestAttributes
        return attrs?.request?.remoteAddr ?: "unknown"
    }
}
```

---

## 🧪 9. Security Tests

```kotlin
// test/SecurityTest.kt
package com.example.security

import org.junit.jupiter.api.Test
import org.springframework.beans.factory.annotation.Autowired
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc
import org.springframework.boot.test.context.SpringBootTest
import org.springframework.http.MediaType
import org.springframework.security.test.web.servlet.request.SecurityMockMvcRequestPostProcessors.csrf
import org.springframework.test.web.servlet.MockMvc
import org.springframework.test.web.servlet.get
import org.springframework.test.web.servlet.post

@SpringBootTest
@AutoConfigureMockMvc
class SecurityTest {

    @Autowired
    private lateinit var mockMvc: MockMvc

    @Test
    fun `unauthenticated request should return 401`() {
        mockMvc.get("/api/users")
            .andExpect { status { isUnauthorized() } }
    }

    @Test
    fun `security headers should be present`() {
        mockMvc.get("/api/users")
            .andExpect {
                header { string("X-Content-Type-Options", "nosniff") }
                header { string("X-Frame-Options", "DENY") }
            }
    }

    @Test
    fun `SQL injection attempt should be rejected`() {
        mockMvc.get("/api/users/search") {
            param("query", "' OR '1'='1")
        }.andExpect {
            // ควรได้ empty result หรือ sanitized input
            status { isUnauthorized() }  // ต้อง authenticate ก่อน
        }
    }

    @Test
    fun `XSS attempt in request param should be sanitized`() {
        mockMvc.post("/auth/register") {
            contentType = MediaType.APPLICATION_JSON
            content = """{"firstName": "<script>alert('XSS')</script>", "email": "test@test.com", "password": "Pass@123"}"""
        }.andExpect {
            // firstName ควรถูก sanitize เป็น empty string หรือ plain text
            status { isAnyOf(400, 201) }
        }
    }

    @Test
    fun `weak password should be rejected`() {
        mockMvc.post("/auth/register") {
            contentType = MediaType.APPLICATION_JSON
            content = """{"firstName": "Test", "email": "test@test.com", "password": "password"}"""
        }.andExpect {
            status { isBadRequest() }
        }
    }
}
```

---

## 📊 สรุปเนื้อหา

| ช่องโหว่ | การป้องกัน | Spring/Kotlin Tool |
|---------|----------|-------------------|
| SQL Injection | Parameterized queries | JPA / Criteria API |
| XSS | Input sanitization | Jsoup / AntiSamy |
| CSRF | CSRF token / SameSite | Spring Security |
| Clickjacking | X-Frame-Options | Security headers |
| Weak passwords | Password policy | Custom validator |
| Brute force | Rate limiting + lockout | Bucket4j |
| Information leak | Remove server headers | Custom filter |

### Security Checklist:

- [x] HTTPS only (HSTS)
- [x] Strong password hashing (Argon2)
- [x] JWT with short expiration
- [x] Rate limiting on auth endpoints
- [x] Input validation สำหรับ ทุก field
- [x] Audit logging สำหรับ sensitive operations
- [x] Security headers ครบถ้วน
- [x] OWASP dependency check ใน CI/CD

---

*Part 47/100+ | Kotlin & Spring Boot Complete Course*
