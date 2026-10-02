# Part 42: Rate Limiting
## การควบคุม Traffic ด้วย Bucket4j และ Redis

---

## 🎯 เป้าหมายของ Part นี้

- เข้าใจแนวคิด Rate Limiting
- ติดตั้งและใช้งาน Bucket4j
- Rate limiting by IP address
- Rate limiting by authenticated user
- Rate limit response headers (X-RateLimit-*)
- Redis-based distributed rate limiting
- ตัวอย่างจริง: API Rate Limiter

---

## 📚 1. Rate Limiting คืออะไร?

Rate Limiting เป็นการจำกัดจำนวน request ที่ client สามารถส่งมาได้ในช่วงเวลาหนึ่ง เพื่อป้องกัน:
- **DDoS attacks** - การโจมตีด้วย traffic จำนวนมาก
- **API abuse** - การใช้ API เกินสิทธิ์ที่กำหนด
- **Cost control** - จำกัดค่าใช้จ่าย (เช่น จำกัด AI API calls)
- **Fair usage** - ให้ทุก user ได้ใช้งานอย่างยุติธรรม

### Token Bucket Algorithm

```
Bucket ขนาด 100 tokens
เติม 10 tokens ต่อวินาที
แต่ละ request ใช้ 1 token

Request 1: bucket = 99 ✅
Request 2: bucket = 98 ✅
...
Request 100: bucket = 0 ✅
Request 101: bucket = 0 ❌ → 429 Too Many Requests
หลังจาก 1 วินาที: bucket = 10 → รับ request ได้อีก
```

---

## 🔧 2. Dependencies Setup

```kotlin
// build.gradle.kts
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-data-redis")
    implementation("org.springframework.boot:spring-boot-starter-security")
    
    // Bucket4j สำหรับ Rate Limiting
    implementation("com.bucket4j:bucket4j-core:8.10.1")
    implementation("com.bucket4j:bucket4j-redis:8.10.1")
    
    // Jackson สำหรับ Kotlin
    implementation("com.fasterxml.jackson.module:jackson-module-kotlin")
    implementation("org.jetbrains.kotlin:kotlin-reflect")
    
    testImplementation("org.springframework.boot:spring-boot-starter-test")
    testImplementation("org.springframework.security:spring-security-test")
    testImplementation("org.testcontainers:redis:1.19.3")
}
```

```yaml
# application.yml
spring:
  application:
    name: rate-limiter-demo
  data:
    redis:
      host: ${REDIS_HOST:localhost}
      port: ${REDIS_PORT:6379}
      password: ${REDIS_PASSWORD:}

rate-limit:
  default:
    capacity: 100
    refill-tokens: 10
    refill-duration-seconds: 1
  premium:
    capacity: 1000
    refill-tokens: 100
    refill-duration-seconds: 1
  public:
    capacity: 20
    refill-tokens: 2
    refill-duration-seconds: 1
```

---

## ⚙️ 3. Rate Limit Properties

```kotlin
// config/RateLimitProperties.kt
package com.example.ratelimit.config

import org.springframework.boot.context.properties.ConfigurationProperties
import org.springframework.stereotype.Component

@Component
@ConfigurationProperties(prefix = "rate-limit")
data class RateLimitProperties(
    val default: TierConfig = TierConfig(),
    val premium: TierConfig = TierConfig(1000, 100, 1),
    val public: TierConfig = TierConfig(20, 2, 1)
) {
    data class TierConfig(
        val capacity: Long = 100,
        val refillTokens: Long = 10,
        val refillDurationSeconds: Long = 1
    )
}
```

---

## 🪣 4. Bucket4j Rate Limiter Service

```kotlin
// service/RateLimiterService.kt
package com.example.ratelimit.service

import com.example.ratelimit.config.RateLimitProperties
import io.github.bucket4j.Bandwidth
import io.github.bucket4j.Bucket
import io.github.bucket4j.Refill
import org.springframework.stereotype.Service
import java.time.Duration
import java.util.concurrent.ConcurrentHashMap

@Service
class RateLimiterService(private val properties: RateLimitProperties) {

    // In-memory cache สำหรับ local rate limiting (ไม่ใช่ distributed)
    private val buckets = ConcurrentHashMap<String, Bucket>()

    /**
     * ดึง bucket สำหรับ key นั้น ถ้ายังไม่มีจะสร้างใหม่
     * @param key - อาจเป็น IP หรือ userId
     * @param tier - ระดับ rate limit (default, premium, public)
     */
    fun resolveBucket(key: String, tier: String = "default"): Bucket {
        return buckets.computeIfAbsent(key) { createBucket(tier) }
    }

    private fun createBucket(tier: String): Bucket {
        val config = when (tier) {
            "premium" -> properties.premium
            "public" -> properties.public
            else -> properties.default
        }

        val bandwidth = Bandwidth.classic(
            config.capacity,
            Refill.greedy(
                config.refillTokens,
                Duration.ofSeconds(config.refillDurationSeconds)
            )
        )

        return Bucket.builder()
            .addLimit(bandwidth)
            .build()
    }

    /**
     * ตรวจสอบว่า request นี้ผ่าน rate limit ได้หรือไม่
     */
    fun tryConsume(key: String, tokens: Long = 1, tier: String = "default"): ConsumptionResult {
        val bucket = resolveBucket(key, tier)
        val probe = bucket.tryConsumeAndReturnRemaining(tokens)

        return ConsumptionResult(
            allowed = probe.isConsumed,
            remainingTokens = probe.remainingTokens,
            nanosToWaitForRefill = probe.nanosToWaitForRefill,
            bucketCapacity = getBucketCapacity(tier)
        )
    }

    private fun getBucketCapacity(tier: String): Long {
        return when (tier) {
            "premium" -> properties.premium.capacity
            "public" -> properties.public.capacity
            else -> properties.default.capacity
        }
    }
}

data class ConsumptionResult(
    val allowed: Boolean,
    val remainingTokens: Long,
    val nanosToWaitForRefill: Long,
    val bucketCapacity: Long
) {
    val retryAfterSeconds: Long get() = nanosToWaitForRefill / 1_000_000_000
}
```

---

## 🔒 5. Rate Limiting Filter

```kotlin
// filter/RateLimitFilter.kt
package com.example.ratelimit.filter

import com.example.ratelimit.service.RateLimiterService
import com.fasterxml.jackson.databind.ObjectMapper
import jakarta.servlet.FilterChain
import jakarta.servlet.http.HttpServletRequest
import jakarta.servlet.http.HttpServletResponse
import org.springframework.http.HttpStatus
import org.springframework.http.MediaType
import org.springframework.security.core.context.SecurityContextHolder
import org.springframework.web.filter.OncePerRequestFilter

class RateLimitFilter(
    private val rateLimiterService: RateLimiterService,
    private val objectMapper: ObjectMapper
) : OncePerRequestFilter() {

    override fun doFilterInternal(
        request: HttpServletRequest,
        response: HttpServletResponse,
        filterChain: FilterChain
    ) {
        val key = resolveKey(request)
        val tier = resolveTier(request)
        val result = rateLimiterService.tryConsume(key, tier = tier)

        // เพิ่ม rate limit headers ทุก response
        response.setHeader("X-RateLimit-Limit", result.bucketCapacity.toString())
        response.setHeader("X-RateLimit-Remaining", result.remainingTokens.toString())

        if (!result.allowed) {
            // แจ้ง retry after
            response.setHeader("X-RateLimit-Retry-After-Seconds", result.retryAfterSeconds.toString())
            response.setHeader("Retry-After", result.retryAfterSeconds.toString())

            response.status = HttpStatus.TOO_MANY_REQUESTS.value()
            response.contentType = MediaType.APPLICATION_JSON_VALUE

            val errorResponse = mapOf(
                "error" to "Too Many Requests",
                "message" to "Rate limit exceeded. Please try again in ${result.retryAfterSeconds} seconds.",
                "retryAfter" to result.retryAfterSeconds
            )

            response.writer.write(objectMapper.writeValueAsString(errorResponse))
            return
        }

        filterChain.doFilter(request, response)
    }

    /**
     * กำหนด key สำหรับ rate limiting
     * - ถ้า authenticated ใช้ userId
     * - ถ้าไม่ authenticated ใช้ IP address
     */
    private fun resolveKey(request: HttpServletRequest): String {
        val authentication = SecurityContextHolder.getContext().authentication
        return if (authentication != null && authentication.isAuthenticated && 
                   authentication.name != "anonymousUser") {
            "user:${authentication.name}"
        } else {
            "ip:${getClientIp(request)}"
        }
    }

    /**
     * กำหนด tier ของ user (ควรดึงจาก database หรือ JWT claims จริงๆ)
     */
    private fun resolveTier(request: HttpServletRequest): String {
        val authentication = SecurityContextHolder.getContext().authentication
        return when {
            authentication?.authorities?.any { it.authority == "ROLE_PREMIUM" } == true -> "premium"
            authentication?.isAuthenticated == true && authentication.name != "anonymousUser" -> "default"
            else -> "public"
        }
    }

    private fun getClientIp(request: HttpServletRequest): String {
        // ตรวจสอบ headers ที่ reverse proxy ส่งมา
        val headers = listOf(
            "X-Forwarded-For",
            "Proxy-Client-IP",
            "WL-Proxy-Client-IP",
            "HTTP_X_FORWARDED_FOR",
            "HTTP_X_FORWARDED",
            "HTTP_FORWARDED_FOR",
            "HTTP_FORWARDED"
        )

        for (header in headers) {
            val ip = request.getHeader(header)
            if (!ip.isNullOrBlank() && ip != "unknown") {
                return ip.split(",").first().trim()
            }
        }

        return request.remoteAddr ?: "unknown"
    }
}
```

---

## 🌐 6. Redis-based Distributed Rate Limiting

เมื่อมีหลาย instance ของ application ต้องใช้ Redis เพื่อ share state

```kotlin
// service/RedisRateLimiterService.kt
package com.example.ratelimit.service

import com.example.ratelimit.config.RateLimitProperties
import io.github.bucket4j.BucketConfiguration
import io.github.bucket4j.distributed.proxy.ProxyManager
import io.github.bucket4j.redis.lettuce.cas.LettuceBasedProxyManager
import io.lettuce.core.RedisClient
import io.lettuce.core.api.StatefulRedisConnection
import io.lettuce.core.codec.ByteArrayCodec
import io.lettuce.core.codec.RedisCodec
import io.lettuce.core.codec.StringCodec
import org.springframework.context.annotation.Profile
import org.springframework.stereotype.Service
import java.time.Duration

@Service
@Profile("redis")
class RedisRateLimiterService(
    private val redisClient: RedisClient,
    private val properties: RateLimitProperties
) {
    private val connection: StatefulRedisConnection<String, ByteArray> by lazy {
        redisClient.connect(RedisCodec.of(StringCodec.UTF8, ByteArrayCodec.INSTANCE))
    }

    private val proxyManager: ProxyManager<String> by lazy {
        LettuceBasedProxyManager.builderFor(connection)
            .withExpirationAfterWrite(Duration.ofHours(1))
            .build()
    }

    fun tryConsume(key: String, tokens: Long = 1, tier: String = "default"): ConsumptionResult {
        val configuration = buildConfiguration(tier)
        val bucket = proxyManager.builder()
            .build(key, configuration)

        val probe = bucket.tryConsumeAndReturnRemaining(tokens)

        return ConsumptionResult(
            allowed = probe.isConsumed,
            remainingTokens = probe.remainingTokens,
            nanosToWaitForRefill = probe.nanosToWaitForRefill,
            bucketCapacity = getBucketCapacity(tier)
        )
    }

    private fun buildConfiguration(tier: String): BucketConfiguration {
        val config = when (tier) {
            "premium" -> properties.premium
            "public" -> properties.public
            else -> properties.default
        }

        return BucketConfiguration.builder()
            .addLimit { limit ->
                limit.capacity(config.capacity)
                    .refillGreedy(
                        config.refillTokens,
                        Duration.ofSeconds(config.refillDurationSeconds)
                    )
            }
            .build()
    }

    private fun getBucketCapacity(tier: String): Long {
        return when (tier) {
            "premium" -> properties.premium.capacity
            "public" -> properties.public.capacity
            else -> properties.default.capacity
        }
    }
}
```

```kotlin
// config/RedisConfig.kt
package com.example.ratelimit.config

import io.lettuce.core.RedisClient
import io.lettuce.core.RedisURI
import org.springframework.beans.factory.annotation.Value
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.context.annotation.Profile

@Configuration
@Profile("redis")
class RedisConfig {

    @Value("\${spring.data.redis.host:localhost}")
    private lateinit var host: String

    @Value("\${spring.data.redis.port:6379}")
    private var port: Int = 6379

    @Value("\${spring.data.redis.password:}")
    private lateinit var password: String

    @Bean
    fun redisClient(): RedisClient {
        val uri = RedisURI.builder()
            .withHost(host)
            .withPort(port)
            .apply {
                if (password.isNotBlank()) {
                    withPassword(password.toCharArray())
                }
            }
            .build()

        return RedisClient.create(uri)
    }
}
```

---

## 📝 7. Rate Limiting Annotation

สร้าง annotation เพื่อ configure rate limit ในระดับ method

```kotlin
// annotation/RateLimit.kt
package com.example.ratelimit.annotation

@Target(AnnotationTarget.FUNCTION)
@Retention(AnnotationRetention.RUNTIME)
annotation class RateLimit(
    val capacity: Long = 100,
    val refillTokens: Long = 10,
    val refillDurationSeconds: Long = 1,
    val keyStrategy: KeyStrategy = KeyStrategy.IP
)

enum class KeyStrategy {
    IP,         // Rate limit by IP
    USER,       // Rate limit by authenticated user
    GLOBAL,     // เดียวกันทุก request (endpoint limit)
    API_KEY     // Rate limit by API key header
}
```

```kotlin
// aspect/RateLimitAspect.kt
package com.example.ratelimit.aspect

import com.example.ratelimit.annotation.KeyStrategy
import com.example.ratelimit.annotation.RateLimit
import com.example.ratelimit.service.RateLimiterService
import io.github.bucket4j.Bandwidth
import io.github.bucket4j.Bucket
import io.github.bucket4j.Refill
import jakarta.servlet.http.HttpServletRequest
import jakarta.servlet.http.HttpServletResponse
import org.aspectj.lang.ProceedingJoinPoint
import org.aspectj.lang.annotation.Around
import org.aspectj.lang.annotation.Aspect
import org.springframework.http.HttpStatus
import org.springframework.security.core.context.SecurityContextHolder
import org.springframework.stereotype.Component
import org.springframework.web.context.request.RequestContextHolder
import org.springframework.web.context.request.ServletRequestAttributes
import java.time.Duration
import java.util.concurrent.ConcurrentHashMap

@Aspect
@Component
class RateLimitAspect {

    private val buckets = ConcurrentHashMap<String, Bucket>()

    @Around("@annotation(rateLimit)")
    fun applyRateLimit(joinPoint: ProceedingJoinPoint, rateLimit: RateLimit): Any? {
        val attributes = RequestContextHolder.currentRequestAttributes() as ServletRequestAttributes
        val request = attributes.request
        val response = attributes.response ?: return joinPoint.proceed()

        val key = resolveKey(request, rateLimit.keyStrategy, joinPoint.signature.name)
        val bucket = buckets.computeIfAbsent(key) {
            createBucket(rateLimit)
        }

        val probe = bucket.tryConsumeAndReturnRemaining(1)
        response.setHeader("X-RateLimit-Remaining", probe.remainingTokens.toString())

        if (!probe.isConsumed) {
            response.status = HttpStatus.TOO_MANY_REQUESTS.value()
            response.writer.write("""{"error": "Rate limit exceeded for this endpoint"}""")
            return null
        }

        return joinPoint.proceed()
    }

    private fun createBucket(rateLimit: RateLimit): Bucket {
        val bandwidth = Bandwidth.classic(
            rateLimit.capacity,
            Refill.greedy(rateLimit.refillTokens, Duration.ofSeconds(rateLimit.refillDurationSeconds))
        )
        return Bucket.builder().addLimit(bandwidth).build()
    }

    private fun resolveKey(
        request: HttpServletRequest,
        strategy: KeyStrategy,
        methodName: String
    ): String = when (strategy) {
        KeyStrategy.IP -> "ip:${request.remoteAddr}:$methodName"
        KeyStrategy.USER -> {
            val user = SecurityContextHolder.getContext().authentication?.name ?: "anonymous"
            "user:$user:$methodName"
        }
        KeyStrategy.GLOBAL -> "global:$methodName"
        KeyStrategy.API_KEY -> {
            val apiKey = request.getHeader("X-API-Key") ?: "no-key"
            "apikey:$apiKey:$methodName"
        }
    }
}
```

---

## 🎮 8. Controller พร้อม Rate Limiting

```kotlin
// controller/ApiController.kt
package com.example.ratelimit.controller

import com.example.ratelimit.annotation.KeyStrategy
import com.example.ratelimit.annotation.RateLimit
import org.springframework.http.ResponseEntity
import org.springframework.web.bind.annotation.*

@RestController
@RequestMapping("/api")
class ApiController {

    // จำกัด 5 requests ต่อวินาที per endpoint (global)
    @GetMapping("/public/data")
    @RateLimit(capacity = 5, refillTokens = 5, refillDurationSeconds = 1, keyStrategy = KeyStrategy.GLOBAL)
    fun getPublicData(): ResponseEntity<Map<String, Any>> {
        return ResponseEntity.ok(mapOf(
            "data" to listOf("item1", "item2", "item3"),
            "timestamp" to System.currentTimeMillis()
        ))
    }

    // จำกัด 100 requests ต่อนาที per user
    @GetMapping("/user/profile")
    @RateLimit(capacity = 100, refillTokens = 10, refillDurationSeconds = 60, keyStrategy = KeyStrategy.USER)
    fun getUserProfile(): ResponseEntity<Map<String, String>> {
        val username = org.springframework.security.core.context.SecurityContextHolder
            .getContext().authentication?.name ?: "anonymous"
        return ResponseEntity.ok(mapOf("username" to username))
    }

    // Heavy operation - จำกัด 10 requests ต่อ 10 วินาที per IP
    @PostMapping("/heavy-operation")
    @RateLimit(capacity = 10, refillTokens = 1, refillDurationSeconds = 10, keyStrategy = KeyStrategy.IP)
    fun performHeavyOperation(@RequestBody request: Map<String, Any>): ResponseEntity<Map<String, Any>> {
        // simulate heavy work
        Thread.sleep(100)
        return ResponseEntity.ok(mapOf("result" to "completed", "input" to request))
    }

    // Export operation - จำกัด 3 ต่อชั่วโมง per user
    @GetMapping("/export")
    @RateLimit(capacity = 3, refillTokens = 1, refillDurationSeconds = 1200, keyStrategy = KeyStrategy.USER)
    fun exportData(): ResponseEntity<Map<String, Any>> {
        return ResponseEntity.ok(mapOf(
            "export" to "data",
            "format" to "csv",
            "rows" to 1000
        ))
    }
}
```

---

## 🧪 9. Testing Rate Limiting

```kotlin
// test/RateLimitTest.kt
package com.example.ratelimit

import org.junit.jupiter.api.Test
import org.springframework.beans.factory.annotation.Autowired
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc
import org.springframework.boot.test.context.SpringBootTest
import org.springframework.test.web.servlet.MockMvc
import org.springframework.test.web.servlet.get
import kotlin.test.assertEquals

@SpringBootTest
@AutoConfigureMockMvc
class RateLimitTest {

    @Autowired
    private lateinit var mockMvc: MockMvc

    @Test
    fun `should allow requests within rate limit`() {
        repeat(5) {
            mockMvc.get("/api/public/data")
                .andExpect {
                    status { isOk() }
                    header { exists("X-RateLimit-Remaining") }
                }
        }
    }

    @Test
    fun `should reject request exceeding rate limit`() {
        // เคลียร์ state ก่อน (ในการทดสอบจริงควรใช้ mock หรือ reset bucket)
        repeat(6) { index ->
            val result = mockMvc.get("/api/public/data").andReturn()
            if (index < 5) {
                assertEquals(200, result.response.status)
            } else {
                assertEquals(429, result.response.status)
                val retryAfter = result.response.getHeader("X-RateLimit-Retry-After-Seconds")
                println("Retry after: $retryAfter seconds")
            }
        }
    }

    @Test
    fun `rate limit headers should be present`() {
        val result = mockMvc.get("/api/public/data")
            .andExpect { status { isOk() } }
            .andReturn()

        assert(result.response.getHeader("X-RateLimit-Remaining") != null)
        assert(result.response.getHeader("X-RateLimit-Limit") != null)
    }
}
```

---

## 🐋 10. Docker Compose with Redis

```yaml
# docker-compose.yml
version: '3.8'
services:
  app:
    build: .
    ports:
      - "8080:8080"
    environment:
      - SPRING_PROFILES_ACTIVE=redis
      - REDIS_HOST=redis
      - REDIS_PORT=6379
    depends_on:
      redis:
        condition: service_healthy

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
    volumes:
      - redis-data:/data
    command: redis-server --appendonly yes

volumes:
  redis-data:
```

---

## 📊 สรุปเนื้อหา

| หัวข้อ | รายละเอียด |
|--------|------------|
| Token Bucket | Algorithm หลักของ Bucket4j |
| In-memory Rate Limit | สำหรับ single instance |
| Redis Rate Limit | สำหรับ distributed system |
| Rate by IP | ป้องกัน anonymous abuse |
| Rate by User | Fair usage per authenticated user |
| Custom Annotation | @RateLimit สำหรับ per-endpoint config |
| Response Headers | X-RateLimit-Limit, X-RateLimit-Remaining, Retry-After |

### HTTP Response Codes:

- `200 OK` - Request ผ่าน rate limit
- `429 Too Many Requests` - Request ถูก rate limited

### Rate Limit Headers:

| Header | ความหมาย |
|--------|---------|
| `X-RateLimit-Limit` | จำนวน requests สูงสุดต่อ window |
| `X-RateLimit-Remaining` | requests ที่เหลือใน window นี้ |
| `X-RateLimit-Retry-After-Seconds` | วินาทีที่ต้องรอ |
| `Retry-After` | เหมือนกัน (standard header) |

---

*Part 42/100+ | Kotlin & Spring Boot Complete Course*
