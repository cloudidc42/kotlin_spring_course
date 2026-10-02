# Part 48: OAuth2 และ SSO
## Social Login ด้วย Google, GitHub และ Spring Authorization Server

---

## 🎯 เป้าหมายของ Part นี้

- เข้าใจ OAuth2 Flow
- OAuth2 Client Setup ด้วย Spring Security
- Google Login
- GitHub Login
- Spring Authorization Server (Basic)
- Token introspection
- ตัวอย่างจริง: Social login ใน application

---

## 📚 1. OAuth2 คืออะไร?

OAuth2 เป็น authorization framework ที่ช่วยให้ application สามารถขอ access ไปยัง resource ของ user บน service อื่นได้ โดยไม่ต้องรู้ password

```
OAuth2 Authorization Code Flow:

User → App: "Login with Google"
App → Google: "ขอ auth code"
Google → User: "App ขอ permission เหล่านี้ อนุญาตไหม?"
User → Google: "อนุญาต"
Google → App: "นี่คือ auth code"
App → Google: "ขอ access token ด้วย auth code"
Google → App: "นี่คือ access token"
App → Google API: "ขอข้อมูล user ด้วย token"
Google API → App: "นี่คือข้อมูล user"
```

### Roles ใน OAuth2:

| Role | คือ | ตัวอย่าง |
|------|-----|---------|
| Resource Owner | User | คุณ |
| Client | Application | App ของเรา |
| Authorization Server | ออก token | Google |
| Resource Server | API | Google Profile API |

---

## 🔧 2. Dependencies Setup

```kotlin
// build.gradle.kts
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-security")
    implementation("org.springframework.boot:spring-boot-starter-oauth2-client")
    
    // สำหรับ Resource Server (ตรวจสอบ token)
    implementation("org.springframework.boot:spring-boot-starter-oauth2-resource-server")
    
    // Spring Authorization Server
    implementation("org.springframework.security:spring-security-oauth2-authorization-server")
    
    // JWT
    implementation("io.jsonwebtoken:jjwt-api:0.12.3")
    runtimeOnly("io.jsonwebtoken:jjwt-impl:0.12.3")
    runtimeOnly("io.jsonwebtoken:jjwt-jackson:0.12.3")
    
    implementation("com.fasterxml.jackson.module:jackson-module-kotlin")
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
    runtimeOnly("com.h2database:h2")
    
    testImplementation("org.springframework.boot:spring-boot-starter-test")
    testImplementation("org.springframework.security:spring-security-test")
}
```

```yaml
# application.yml
spring:
  security:
    oauth2:
      client:
        registration:
          google:
            client-id: ${GOOGLE_CLIENT_ID}
            client-secret: ${GOOGLE_CLIENT_SECRET}
            scope:
              - openid
              - profile
              - email
            redirect-uri: "{baseUrl}/login/oauth2/code/{registrationId}"
          
          github:
            client-id: ${GITHUB_CLIENT_ID}
            client-secret: ${GITHUB_CLIENT_SECRET}
            scope:
              - user:email
              - read:user
        
        provider:
          google:
            authorization-uri: https://accounts.google.com/o/oauth2/v2/auth
            token-uri: https://oauth2.googleapis.com/token
            user-info-uri: https://www.googleapis.com/oauth2/v3/userinfo
            user-name-attribute: sub
```

---

## ⚙️ 3. OAuth2 Security Configuration

```kotlin
// config/OAuth2SecurityConfig.kt
package com.example.oauth2.config

import com.example.oauth2.handler.OAuth2AuthenticationSuccessHandler
import com.example.oauth2.handler.OAuth2AuthenticationFailureHandler
import com.example.oauth2.service.CustomOAuth2UserService
import com.example.oauth2.service.CustomOidcUserService
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.security.config.annotation.web.builders.HttpSecurity
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity
import org.springframework.security.config.http.SessionCreationPolicy
import org.springframework.security.web.SecurityFilterChain

@Configuration
@EnableWebSecurity
class OAuth2SecurityConfig(
    private val customOAuth2UserService: CustomOAuth2UserService,
    private val customOidcUserService: CustomOidcUserService,
    private val successHandler: OAuth2AuthenticationSuccessHandler,
    private val failureHandler: OAuth2AuthenticationFailureHandler
) {

    @Bean
    fun securityFilterChain(http: HttpSecurity): SecurityFilterChain {
        http
            .csrf { it.disable() }
            .cors { }

            .authorizeHttpRequests { auth ->
                auth
                    .requestMatchers("/", "/auth/**", "/oauth2/**").permitAll()
                    .requestMatchers("/public/**").permitAll()
                    .anyRequest().authenticated()
            }

            // OAuth2 Login configuration
            .oauth2Login { oauth2 ->
                oauth2
                    .authorizationEndpoint { endpoint ->
                        endpoint.baseUri("/oauth2/authorize")
                    }
                    .redirectionEndpoint { endpoint ->
                        endpoint.baseUri("/login/oauth2/code/*")
                    }
                    .userInfoEndpoint { userInfo ->
                        userInfo
                            .userService(customOAuth2UserService)   // สำหรับ GitHub (non-OIDC)
                            .oidcUserService(customOidcUserService) // สำหรับ Google (OIDC)
                    }
                    .successHandler(successHandler)
                    .failureHandler(failureHandler)
            }

        return http.build()
    }
}
```

---

## 👤 4. Custom OAuth2 User Service

```kotlin
// entity/User.kt
package com.example.oauth2.entity

import jakarta.persistence.*

@Entity
@Table(name = "users")
data class User(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long = 0,

    @Column(unique = true)
    val email: String = "",

    val name: String = "",
    val firstName: String = "",
    val lastName: String = "",
    val imageUrl: String? = null,

    @Column(name = "provider")
    @Enumerated(EnumType.STRING)
    val provider: AuthProvider = AuthProvider.LOCAL,

    @Column(name = "provider_id")
    val providerId: String? = null,

    val emailVerified: Boolean = false,
    
    @Column(name = "auth_token")
    var authToken: String? = null
)

enum class AuthProvider {
    LOCAL, GOOGLE, GITHUB, FACEBOOK
}
```

```kotlin
// service/CustomOAuth2UserService.kt
package com.example.oauth2.service

import com.example.oauth2.entity.AuthProvider
import com.example.oauth2.entity.User
import com.example.oauth2.repository.UserRepository
import org.springframework.security.authentication.InternalAuthenticationServiceException
import org.springframework.security.core.AuthenticationException
import org.springframework.security.oauth2.client.userinfo.DefaultOAuth2UserService
import org.springframework.security.oauth2.client.userinfo.OAuth2UserRequest
import org.springframework.security.oauth2.core.user.OAuth2User
import org.springframework.stereotype.Service

@Service
class CustomOAuth2UserService(
    private val userRepository: UserRepository
) : DefaultOAuth2UserService() {

    override fun loadUser(userRequest: OAuth2UserRequest): OAuth2User {
        val oAuth2User = super.loadUser(userRequest)

        return try {
            processOAuth2User(userRequest, oAuth2User)
        } catch (ex: AuthenticationException) {
            throw ex
        } catch (ex: Exception) {
            throw InternalAuthenticationServiceException(ex.message, ex.cause)
        }
    }

    private fun processOAuth2User(
        userRequest: OAuth2UserRequest,
        oAuth2User: OAuth2User
    ): OAuth2User {
        val registrationId = userRequest.clientRegistration.registrationId
        val userInfo = OAuth2UserInfoFactory.getOAuth2UserInfo(registrationId, oAuth2User.attributes)

        val email = userInfo.email
            ?: throw IllegalArgumentException("Email not found from OAuth2 provider")

        val user = userRepository.findByEmail(email)?.let { existingUser ->
            updateExistingUser(existingUser, userInfo)
        } ?: registerNewUser(userRequest, userInfo)

        return CustomUserPrincipal.create(user, oAuth2User.attributes)
    }

    private fun registerNewUser(
        userRequest: OAuth2UserRequest,
        userInfo: OAuth2UserInfo
    ): User {
        val provider = AuthProvider.valueOf(
            userRequest.clientRegistration.registrationId.uppercase()
        )

        val user = User(
            email = userInfo.email ?: "",
            name = userInfo.name ?: "",
            firstName = userInfo.firstName ?: "",
            lastName = userInfo.lastName ?: "",
            imageUrl = userInfo.imageUrl,
            provider = provider,
            providerId = userInfo.id,
            emailVerified = true
        )

        return userRepository.save(user)
    }

    private fun updateExistingUser(existingUser: User, userInfo: OAuth2UserInfo): User {
        return userRepository.save(
            existingUser.copy(
                name = userInfo.name ?: existingUser.name,
                imageUrl = userInfo.imageUrl ?: existingUser.imageUrl
            )
        )
    }
}
```

---

## 🔧 5. OAuth2 User Info Abstraction

```kotlin
// service/OAuth2UserInfo.kt
package com.example.oauth2.service

abstract class OAuth2UserInfo(val attributes: Map<String, Any>) {
    abstract val id: String?
    abstract val name: String?
    abstract val firstName: String?
    abstract val lastName: String?
    abstract val email: String?
    abstract val imageUrl: String?
}

class GoogleOAuth2UserInfo(attributes: Map<String, Any>) : OAuth2UserInfo(attributes) {
    override val id: String? get() = attributes["sub"]?.toString()
    override val name: String? get() = attributes["name"]?.toString()
    override val firstName: String? get() = attributes["given_name"]?.toString()
    override val lastName: String? get() = attributes["family_name"]?.toString()
    override val email: String? get() = attributes["email"]?.toString()
    override val imageUrl: String? get() = attributes["picture"]?.toString()
}

class GithubOAuth2UserInfo(attributes: Map<String, Any>) : OAuth2UserInfo(attributes) {
    override val id: String? get() = attributes["id"]?.toString()
    override val name: String? get() = attributes["name"]?.toString()
    override val firstName: String? get() = name?.split(" ")?.firstOrNull()
    override val lastName: String? get() = name?.split(" ")?.let { if (it.size > 1) it.last() else "" }
    override val email: String? get() = attributes["email"]?.toString()
    override val imageUrl: String? get() = attributes["avatar_url"]?.toString()
}

object OAuth2UserInfoFactory {
    fun getOAuth2UserInfo(registrationId: String, attributes: Map<String, Any>): OAuth2UserInfo {
        return when (registrationId.lowercase()) {
            "google" -> GoogleOAuth2UserInfo(attributes)
            "github" -> GithubOAuth2UserInfo(attributes)
            else -> throw IllegalArgumentException("Unsupported OAuth2 provider: $registrationId")
        }
    }
}
```

---

## ✅ 6. Success Handler - ออก JWT หลัง OAuth2 Login

```kotlin
// handler/OAuth2AuthenticationSuccessHandler.kt
package com.example.oauth2.handler

import com.example.oauth2.service.CustomUserPrincipal
import com.example.oauth2.service.JwtTokenService
import jakarta.servlet.http.HttpServletRequest
import jakarta.servlet.http.HttpServletResponse
import org.springframework.beans.factory.annotation.Value
import org.springframework.security.core.Authentication
import org.springframework.security.web.authentication.SimpleUrlAuthenticationSuccessHandler
import org.springframework.stereotype.Component
import org.springframework.web.util.UriComponentsBuilder

@Component
class OAuth2AuthenticationSuccessHandler(
    private val jwtTokenService: JwtTokenService
) : SimpleUrlAuthenticationSuccessHandler() {

    @Value("\${app.frontend-url:http://localhost:3000}")
    private lateinit var frontendUrl: String

    override fun onAuthenticationSuccess(
        request: HttpServletRequest,
        response: HttpServletResponse,
        authentication: Authentication
    ) {
        val targetUrl = determineTargetUrl(request, response, authentication)

        if (response.isCommitted) {
            logger.debug("Response has already been committed. Unable to redirect to $targetUrl")
            return
        }

        clearAuthenticationAttributes(request)
        redirectStrategy.sendRedirect(request, response, targetUrl)
    }

    override fun determineTargetUrl(
        request: HttpServletRequest,
        response: HttpServletResponse,
        authentication: Authentication
    ): String {
        val userPrincipal = authentication.principal as CustomUserPrincipal
        val token = jwtTokenService.generateToken(userPrincipal.user)

        return UriComponentsBuilder
            .fromUriString("$frontendUrl/oauth2/callback")
            .queryParam("token", token)
            .build()
            .toUriString()
    }
}
```

```kotlin
// handler/OAuth2AuthenticationFailureHandler.kt
package com.example.oauth2.handler

import jakarta.servlet.http.HttpServletRequest
import jakarta.servlet.http.HttpServletResponse
import org.springframework.beans.factory.annotation.Value
import org.springframework.security.core.AuthenticationException
import org.springframework.security.web.authentication.SimpleUrlAuthenticationFailureHandler
import org.springframework.stereotype.Component
import org.springframework.web.util.UriComponentsBuilder
import java.net.URLEncoder
import java.nio.charset.StandardCharsets

@Component
class OAuth2AuthenticationFailureHandler : SimpleUrlAuthenticationFailureHandler() {

    @Value("\${app.frontend-url:http://localhost:3000}")
    private lateinit var frontendUrl: String

    override fun onAuthenticationFailure(
        request: HttpServletRequest,
        response: HttpServletResponse,
        exception: AuthenticationException
    ) {
        val targetUrl = UriComponentsBuilder
            .fromUriString("$frontendUrl/login")
            .queryParam("error", URLEncoder.encode(exception.localizedMessage, StandardCharsets.UTF_8))
            .build()
            .toUriString()

        redirectStrategy.sendRedirect(request, response, targetUrl)
    }
}
```

---

## 🏛️ 7. Spring Authorization Server (Basic)

สำหรับการสร้าง OAuth2/OIDC server เอง

```kotlin
// config/AuthorizationServerConfig.kt
package com.example.oauth2.config

import com.nimbusds.jose.jwk.JWKSet
import com.nimbusds.jose.jwk.RSAKey
import com.nimbusds.jose.jwk.source.ImmutableJWKSet
import com.nimbusds.jose.jwk.source.JWKSource
import com.nimbusds.jose.proc.SecurityContext
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.core.annotation.Order
import org.springframework.http.MediaType
import org.springframework.security.config.Customizer
import org.springframework.security.config.annotation.web.builders.HttpSecurity
import org.springframework.security.oauth2.core.AuthorizationGrantType
import org.springframework.security.oauth2.core.ClientAuthenticationMethod
import org.springframework.security.oauth2.core.oidc.OidcScopes
import org.springframework.security.oauth2.jwt.JwtDecoder
import org.springframework.security.oauth2.server.authorization.client.InMemoryRegisteredClientRepository
import org.springframework.security.oauth2.server.authorization.client.RegisteredClient
import org.springframework.security.oauth2.server.authorization.client.RegisteredClientRepository
import org.springframework.security.oauth2.server.authorization.config.annotation.web.configuration.OAuth2AuthorizationServerConfiguration
import org.springframework.security.oauth2.server.authorization.config.annotation.web.configurers.OAuth2AuthorizationServerConfigurer
import org.springframework.security.oauth2.server.authorization.settings.AuthorizationServerSettings
import org.springframework.security.oauth2.server.authorization.settings.ClientSettings
import org.springframework.security.oauth2.server.authorization.settings.TokenSettings
import org.springframework.security.web.SecurityFilterChain
import org.springframework.security.web.authentication.LoginUrlAuthenticationEntryPoint
import org.springframework.security.web.util.matcher.MediaTypeRequestMatcher
import java.security.KeyPairGenerator
import java.security.interfaces.RSAPrivateKey
import java.security.interfaces.RSAPublicKey
import java.time.Duration
import java.util.UUID

@Configuration
class AuthorizationServerConfig {

    @Bean
    @Order(1)
    fun authorizationServerSecurityFilterChain(http: HttpSecurity): SecurityFilterChain {
        OAuth2AuthorizationServerConfiguration.applyDefaultSecurity(http)

        http.getConfigurer(OAuth2AuthorizationServerConfigurer::class.java)
            .oidc(Customizer.withDefaults())  // Enable OpenID Connect

        http
            .exceptionHandling { exceptions ->
                exceptions.defaultAuthenticationEntryPointFor(
                    LoginUrlAuthenticationEntryPoint("/login"),
                    MediaTypeRequestMatcher(MediaType.TEXT_HTML)
                )
            }
            .oauth2ResourceServer { it.jwt(Customizer.withDefaults()) }

        return http.build()
    }

    @Bean
    fun registeredClientRepository(): RegisteredClientRepository {
        val webClient = RegisteredClient
            .withId(UUID.randomUUID().toString())
            .clientId("web-client")
            .clientSecret("{bcrypt}encoded_secret")
            .clientAuthenticationMethod(ClientAuthenticationMethod.CLIENT_SECRET_BASIC)
            .authorizationGrantType(AuthorizationGrantType.AUTHORIZATION_CODE)
            .authorizationGrantType(AuthorizationGrantType.REFRESH_TOKEN)
            .redirectUri("http://localhost:3000/callback")
            .scope(OidcScopes.OPENID)
            .scope(OidcScopes.PROFILE)
            .scope(OidcScopes.EMAIL)
            .scope("api:read")
            .scope("api:write")
            .clientSettings(
                ClientSettings.builder()
                    .requireAuthorizationConsent(true)
                    .build()
            )
            .tokenSettings(
                TokenSettings.builder()
                    .accessTokenTimeToLive(Duration.ofHours(1))
                    .refreshTokenTimeToLive(Duration.ofDays(30))
                    .reuseRefreshTokens(false)
                    .build()
            )
            .build()

        val mobileClient = RegisteredClient
            .withId(UUID.randomUUID().toString())
            .clientId("mobile-client")
            .clientAuthenticationMethod(ClientAuthenticationMethod.NONE)  // PKCE
            .authorizationGrantType(AuthorizationGrantType.AUTHORIZATION_CODE)
            .authorizationGrantType(AuthorizationGrantType.REFRESH_TOKEN)
            .redirectUri("com.example.app://callback")
            .scope(OidcScopes.OPENID)
            .scope(OidcScopes.PROFILE)
            .clientSettings(
                ClientSettings.builder()
                    .requireProofKey(true)  // PKCE required
                    .build()
            )
            .build()

        return InMemoryRegisteredClientRepository(webClient, mobileClient)
    }

    @Bean
    fun jwkSource(): JWKSource<SecurityContext> {
        val keyPairGenerator = KeyPairGenerator.getInstance("RSA")
        keyPairGenerator.initialize(2048)
        val keyPair = keyPairGenerator.generateKeyPair()

        val rsaKey = RSAKey.Builder(keyPair.public as RSAPublicKey)
            .privateKey(keyPair.private as RSAPrivateKey)
            .keyID(UUID.randomUUID().toString())
            .build()

        return ImmutableJWKSet(JWKSet(rsaKey))
    }

    @Bean
    fun jwtDecoder(jwkSource: JWKSource<SecurityContext>): JwtDecoder {
        return OAuth2AuthorizationServerConfiguration.jwtDecoder(jwkSource)
    }

    @Bean
    fun authorizationServerSettings(): AuthorizationServerSettings {
        return AuthorizationServerSettings.builder()
            .issuer("http://localhost:8080")
            .build()
    }
}
```

---

## 🎮 8. Controller

```kotlin
// controller/UserController.kt
package com.example.oauth2.controller

import com.example.oauth2.entity.User
import com.example.oauth2.service.CustomUserPrincipal
import org.springframework.http.ResponseEntity
import org.springframework.security.core.annotation.AuthenticationPrincipal
import org.springframework.security.oauth2.core.user.OAuth2User
import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.RequestMapping
import org.springframework.web.bind.annotation.RestController

@RestController
@RequestMapping("/api")
class UserController {

    @GetMapping("/user/me")
    fun getCurrentUser(
        @AuthenticationPrincipal principal: CustomUserPrincipal
    ): ResponseEntity<Map<String, Any?>> {
        return ResponseEntity.ok(
            mapOf(
                "id" to principal.user.id,
                "name" to principal.user.name,
                "email" to principal.user.email,
                "imageUrl" to principal.user.imageUrl,
                "provider" to principal.user.provider
            )
        )
    }

    @GetMapping("/user/oauth2-attributes")
    fun getOAuth2Attributes(
        @AuthenticationPrincipal oauth2User: OAuth2User
    ): ResponseEntity<Map<String, Any>> {
        return ResponseEntity.ok(oauth2User.attributes)
    }
}
```

---

## 🔑 9. JWT Token Service

```kotlin
// service/JwtTokenService.kt
package com.example.oauth2.service

import com.example.oauth2.entity.User
import io.jsonwebtoken.Claims
import io.jsonwebtoken.Jwts
import io.jsonwebtoken.security.Keys
import org.springframework.beans.factory.annotation.Value
import org.springframework.stereotype.Service
import java.util.Date
import javax.crypto.SecretKey

@Service
class JwtTokenService {

    @Value("\${jwt.secret:mySecretKeyThatIsAtLeast256BitsLongForHMACSHA256}")
    private lateinit var jwtSecret: String

    @Value("\${jwt.expiration:3600000}")
    private var jwtExpiration: Long = 3600000

    private val key: SecretKey by lazy {
        Keys.hmacShaKeyFor(jwtSecret.toByteArray())
    }

    fun generateToken(user: User): String {
        val now = Date()
        val expiryDate = Date(now.time + jwtExpiration)

        return Jwts.builder()
            .subject(user.id.toString())
            .claim("email", user.email)
            .claim("name", user.name)
            .claim("provider", user.provider.name)
            .issuedAt(now)
            .expiration(expiryDate)
            .signWith(key)
            .compact()
    }

    fun getUserIdFromToken(token: String): Long {
        val claims = getClaims(token)
        return claims.subject.toLong()
    }

    fun validateToken(token: String): Boolean {
        return try {
            getClaims(token)
            true
        } catch (ex: Exception) {
            false
        }
    }

    private fun getClaims(token: String): Claims {
        return Jwts.parser()
            .verifyWith(key)
            .build()
            .parseSignedClaims(token)
            .payload
    }
}
```

---

## 📊 สรุปเนื้อหา

| ขั้นตอน | รายละเอียด |
|--------|-----------|
| สร้าง App ใน Google Console | ตั้งค่า OAuth credentials |
| สร้าง App ใน GitHub | Settings → Developer → OAuth Apps |
| ตั้งค่า application.yml | Client ID, Secret, Scopes |
| Implement CustomOAuth2UserService | Process user info จาก provider |
| Implement Success Handler | ออก JWT หลัง login สำเร็จ |
| Frontend callback | รับ JWT token และ store |

### OAuth2 Endpoints ที่ Spring สร้างให้:

| Endpoint | ประโยชน์ |
|---------|---------|
| `/oauth2/authorize/google` | เริ่ม Google login |
| `/oauth2/authorize/github` | เริ่ม GitHub login |
| `/login/oauth2/code/google` | Callback จาก Google |
| `/login/oauth2/code/github` | Callback จาก GitHub |

### Setting Up Google OAuth2:

1. ไปที่ [Google Cloud Console](https://console.cloud.google.com)
2. สร้าง project ใหม่หรือเลือก project ที่มีอยู่
3. Enabled APIs & Services → Credentials
4. Create Credentials → OAuth 2.0 Client IDs
5. เพิ่ม Authorized redirect URIs: `http://localhost:8080/login/oauth2/code/google`

---

*Part 48/100+ | Kotlin & Spring Boot Complete Course*
