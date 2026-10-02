# Part 65: Kotlin Multiplatform (KMP) - Shared Business Logic

## บทนำ

**Kotlin Multiplatform (KMP)** คือ technology ที่ช่วยให้เราเขียน code หนึ่งชุดแล้วรันได้บนหลาย platforms เช่น JVM, Android, iOS, JavaScript, Native โดยใช้ **expect/actual** mechanism สำหรับ platform-specific implementations

## ทำไมต้อง KMP?

### ปัญหาของ Multiplatform Development

```
ปกติ: เขียน validation logic 3 ครั้ง
- Backend (Kotlin/Java): EmailValidator.kt
- Android: EmailValidator.kt (ซ้ำ!)  
- iOS: EmailValidator.swift (ซ้ำ!)
- Web: emailValidator.ts (ซ้ำ!)

KMP: เขียนครั้งเดียว รันได้ทุก platform
- Shared: EmailValidator.kt → ใช้ได้ทุกที่
```

## โครงสร้าง KMP Project

```
shared-lib/
├── src/
│   ├── commonMain/
│   │   └── kotlin/
│   │       └── com/shared/
│   │           ├── validation/
│   │           │   ├── EmailValidator.kt    ← Shared code
│   │           │   ├── PasswordValidator.kt ← Shared code
│   │           │   └── PhoneValidator.kt    ← Shared code
│   │           ├── model/
│   │           │   └── User.kt              ← Shared model
│   │           └── utils/
│   │               └── DateUtils.kt         ← Shared utils
│   ├── jvmMain/
│   │   └── kotlin/
│   │       └── com/shared/platform/
│   │           └── JvmPlatform.kt           ← JVM actual
│   ├── androidMain/
│   │   └── kotlin/
│   │       └── com/shared/platform/
│   │           └── AndroidPlatform.kt       ← Android actual
│   ├── iosMain/
│   │   └── kotlin/
│   │       └── com/shared/platform/
│   │           └── IosPlatform.kt           ← iOS actual
│   └── jsMain/
│       └── kotlin/
│           └── com/shared/platform/
│               └── JsPlatform.kt            ← JS actual
└── build.gradle.kts
```

## การตั้งค่า Build

```kotlin
// shared-lib/build.gradle.kts
plugins {
    kotlin("multiplatform") version "1.9.20"
    kotlin("plugin.serialization") version "1.9.20"
    id("com.android.library")  // ถ้าต้องการ Android support
}

kotlin {
    // JVM target (รวม Spring Boot)
    jvm {
        compilations.all {
            kotlinOptions.jvmTarget = "17"
        }
    }
    
    // Android target
    androidTarget {
        compilations.all {
            kotlinOptions.jvmTarget = "17"
        }
    }
    
    // iOS targets
    listOf(
        iosX64(),
        iosArm64(),
        iosSimulatorArm64()
    ).forEach { iosTarget ->
        iosTarget.binaries.framework {
            baseName = "SharedLib"
        }
    }
    
    // JavaScript target
    js(IR) {
        browser()
        nodejs()
    }
    
    sourceSets {
        val commonMain by getting {
            dependencies {
                implementation("org.jetbrains.kotlinx:kotlinx-serialization-json:1.6.0")
                implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:1.7.3")
                implementation("org.jetbrains.kotlinx:kotlinx-datetime:0.4.1")
            }
        }
        
        val commonTest by getting {
            dependencies {
                implementation(kotlin("test"))
            }
        }
        
        val jvmMain by getting {
            dependencies {
                // JVM-specific dependencies
            }
        }
        
        val androidMain by getting {
            dependencies {
                implementation("androidx.core:core-ktx:1.12.0")
            }
        }
        
        val iosMain by creating {
            dependsOn(commonMain)
        }
    }
}
```

## expect/actual Mechanism

### expect declaration (common code)

```kotlin
// commonMain/kotlin/com/shared/platform/Platform.kt
package com.shared.platform

// expect ประกาศว่า "มีฟังก์ชันนี้" แต่แต่ละ platform จะ implement เอง
expect class Platform() {
    val name: String
    val version: String
    val isDebug: Boolean
}

expect fun currentTimeMillis(): Long

expect fun generateUUID(): String

expect fun encryptData(data: String, key: String): String

expect fun decryptData(encryptedData: String, key: String): String
```

### actual implementations

```kotlin
// jvmMain/kotlin/com/shared/platform/JvmPlatform.kt
package com.shared.platform

import java.util.UUID
import java.util.Base64
import javax.crypto.Cipher
import javax.crypto.spec.SecretKeySpec

actual class Platform actual constructor() {
    actual val name: String = "JVM"
    actual val version: String = System.getProperty("java.version") ?: "unknown"
    actual val isDebug: Boolean = System.getProperty("debug") == "true"
}

actual fun currentTimeMillis(): Long = System.currentTimeMillis()

actual fun generateUUID(): String = UUID.randomUUID().toString()

actual fun encryptData(data: String, key: String): String {
    val keySpec = SecretKeySpec(key.toByteArray().copyOf(16), "AES")
    val cipher = Cipher.getInstance("AES")
    cipher.init(Cipher.ENCRYPT_MODE, keySpec)
    val encrypted = cipher.doFinal(data.toByteArray())
    return Base64.getEncoder().encodeToString(encrypted)
}

actual fun decryptData(encryptedData: String, key: String): String {
    val keySpec = SecretKeySpec(key.toByteArray().copyOf(16), "AES")
    val cipher = Cipher.getInstance("AES")
    cipher.init(Cipher.DECRYPT_MODE, keySpec)
    val decoded = Base64.getDecoder().decode(encryptedData)
    return String(cipher.doFinal(decoded))
}
```

```kotlin
// androidMain/kotlin/com/shared/platform/AndroidPlatform.kt
package com.shared.platform

import android.os.Build
import java.util.UUID

actual class Platform actual constructor() {
    actual val name: String = "Android"
    actual val version: String = Build.VERSION.RELEASE
    actual val isDebug: Boolean = BuildConfig.DEBUG
}

actual fun currentTimeMillis(): Long = System.currentTimeMillis()

actual fun generateUUID(): String = UUID.randomUUID().toString()

actual fun encryptData(data: String, key: String): String {
    // Android-specific encryption using AndroidKeyStore
    val keySpec = android.security.keystore.KeyGenParameterSpec.Builder(
        "shared_key",
        android.security.keystore.KeyProperties.PURPOSE_ENCRYPT or
        android.security.keystore.KeyProperties.PURPOSE_DECRYPT
    )
    .setBlockModes(android.security.keystore.KeyProperties.BLOCK_MODE_CBC)
    .build()
    // ... encryption logic
    return ""
}

actual fun decryptData(encryptedData: String, key: String): String {
    // Android-specific decryption
    return ""
}
```

```kotlin
// iosMain/kotlin/com/shared/platform/IosPlatform.kt
package com.shared.platform

import platform.Foundation.NSProcessInfo
import platform.Foundation.NSUUID

actual class Platform actual constructor() {
    actual val name: String = "iOS"
    actual val version: String = NSProcessInfo.processInfo.operatingSystemVersionString
    actual val isDebug: Boolean = false // iOS specific
}

actual fun currentTimeMillis(): Long = 
    (platform.Foundation.NSDate().timeIntervalSince1970 * 1000).toLong()

actual fun generateUUID(): String = NSUUID().UUIDString()

actual fun encryptData(data: String, key: String): String {
    // iOS-specific encryption using CommonCrypto or CryptoKit
    return ""
}

actual fun decryptData(encryptedData: String, key: String): String {
    return ""
}
```

```kotlin
// jsMain/kotlin/com/shared/platform/JsPlatform.kt
package com.shared.platform

actual class Platform actual constructor() {
    actual val name: String = "JavaScript"
    actual val version: String = js("navigator.userAgent") as String
    actual val isDebug: Boolean = js("process?.env?.NODE_ENV === 'development'") as Boolean
}

actual fun currentTimeMillis(): Long = js("Date.now()") as Long

actual fun generateUUID(): String = js("crypto.randomUUID()") as String

actual fun encryptData(data: String, key: String): String {
    // Web Crypto API
    return js("btoa(data)") as String  // simplified
}

actual fun decryptData(encryptedData: String, key: String): String {
    return js("atob(encryptedData)") as String  // simplified
}
```

## Shared Business Logic

### Validation (ใช้ร่วมกันทุก platform)

```kotlin
// commonMain/kotlin/com/shared/validation/Validators.kt
package com.shared.validation

import kotlinx.serialization.Serializable

@Serializable
sealed class ValidationResult {
    @Serializable
    object Valid : ValidationResult()
    
    @Serializable
    data class Invalid(val errors: List<ValidationError>) : ValidationResult()

    fun isValid(): Boolean = this is Valid
    fun getErrors(): List<ValidationError> = if (this is Invalid) errors else emptyList()
}

@Serializable
data class ValidationError(
    val field: String,
    val message: String,
    val code: String
)

object EmailValidator {
    private val emailRegex = Regex(
        "^[A-Za-z0-9+_.-]+@[A-Za-z0-9.-]+\\.[A-Za-z]{2,}$"
    )
    
    fun validate(email: String): ValidationResult {
        val errors = mutableListOf<ValidationError>()
        
        if (email.isBlank()) {
            errors.add(ValidationError("email", "Email is required", "REQUIRED"))
        } else if (!emailRegex.matches(email)) {
            errors.add(ValidationError("email", "Invalid email format", "INVALID_FORMAT"))
        } else if (email.length > 254) {
            errors.add(ValidationError("email", "Email too long", "TOO_LONG"))
        }
        
        return if (errors.isEmpty()) ValidationResult.Valid else ValidationResult.Invalid(errors)
    }
}

object PasswordValidator {
    fun validate(password: String): ValidationResult {
        val errors = mutableListOf<ValidationError>()
        
        if (password.length < 8) {
            errors.add(ValidationError("password", "Password must be at least 8 characters", "TOO_SHORT"))
        }
        if (!password.any { it.isUpperCase() }) {
            errors.add(ValidationError("password", "Password must contain uppercase letter", "NO_UPPERCASE"))
        }
        if (!password.any { it.isLowerCase() }) {
            errors.add(ValidationError("password", "Password must contain lowercase letter", "NO_LOWERCASE"))
        }
        if (!password.any { it.isDigit() }) {
            errors.add(ValidationError("password", "Password must contain a digit", "NO_DIGIT"))
        }
        if (!password.any { !it.isLetterOrDigit() }) {
            errors.add(ValidationError("password", "Password must contain special character", "NO_SPECIAL_CHAR"))
        }
        
        return if (errors.isEmpty()) ValidationResult.Valid else ValidationResult.Invalid(errors)
    }
    
    fun calculateStrength(password: String): PasswordStrength {
        var score = 0
        if (password.length >= 8) score++
        if (password.length >= 12) score++
        if (password.any { it.isUpperCase() }) score++
        if (password.any { it.isLowerCase() }) score++
        if (password.any { it.isDigit() }) score++
        if (password.any { !it.isLetterOrDigit() }) score++
        
        return when {
            score <= 2 -> PasswordStrength.WEAK
            score <= 4 -> PasswordStrength.MEDIUM
            score == 5 -> PasswordStrength.STRONG
            else -> PasswordStrength.VERY_STRONG
        }
    }
}

enum class PasswordStrength { WEAK, MEDIUM, STRONG, VERY_STRONG }

object PhoneValidator {
    // Thailand phone number
    private val thaiPhoneRegex = Regex("^(0[689]\\d{8}|\\+66[689]\\d{8})$")
    
    fun validate(phone: String, countryCode: String = "TH"): ValidationResult {
        val errors = mutableListOf<ValidationError>()
        val cleanPhone = phone.replace("-", "").replace(" ", "")
        
        when (countryCode) {
            "TH" -> {
                if (!thaiPhoneRegex.matches(cleanPhone)) {
                    errors.add(ValidationError("phone", "Invalid Thai phone number", "INVALID_FORMAT"))
                }
            }
            else -> {
                if (cleanPhone.length < 7 || cleanPhone.length > 15) {
                    errors.add(ValidationError("phone", "Invalid phone number length", "INVALID_LENGTH"))
                }
            }
        }
        
        return if (errors.isEmpty()) ValidationResult.Valid else ValidationResult.Invalid(errors)
    }
}
```

### Shared Models

```kotlin
// commonMain/kotlin/com/shared/model/UserModels.kt
package com.shared.model

import kotlinx.serialization.Serializable
import kotlinx.datetime.LocalDateTime
import kotlinx.datetime.Clock
import kotlinx.datetime.TimeZone
import kotlinx.datetime.toLocalDateTime

@Serializable
data class UserRegistration(
    val email: String,
    val password: String,
    val firstName: String,
    val lastName: String,
    val phoneNumber: String? = null
) {
    fun validate(): ValidationResult {
        val errors = mutableListOf<ValidationError>()
        
        val emailResult = EmailValidator.validate(email)
        if (!emailResult.isValid()) errors.addAll(emailResult.getErrors())
        
        val passwordResult = PasswordValidator.validate(password)
        if (!passwordResult.isValid()) errors.addAll(passwordResult.getErrors())
        
        if (firstName.isBlank()) {
            errors.add(ValidationError("firstName", "First name is required", "REQUIRED"))
        }
        if (lastName.isBlank()) {
            errors.add(ValidationError("lastName", "Last name is required", "REQUIRED"))
        }
        
        phoneNumber?.let { phone ->
            val phoneResult = PhoneValidator.validate(phone)
            if (!phoneResult.isValid()) errors.addAll(phoneResult.getErrors())
        }
        
        return if (errors.isEmpty()) ValidationResult.Valid else ValidationResult.Invalid(errors)
    }
}

@Serializable
data class UserProfile(
    val id: String,
    val email: String,
    val firstName: String,
    val lastName: String,
    val fullName: String = "$firstName $lastName",
    val role: UserRole,
    val active: Boolean,
    val createdAt: String,
    val updatedAt: String
)

@Serializable
enum class UserRole {
    USER, ADMIN, MODERATOR
}
```

### Shared Business Logic

```kotlin
// commonMain/kotlin/com/shared/business/OrderCalculator.kt
package com.shared.business

import kotlinx.serialization.Serializable
import kotlin.math.roundToInt

@Serializable
data class OrderItem(
    val productId: String,
    val productName: String,
    val quantity: Int,
    val unitPrice: Double,
    val discountPercent: Double = 0.0
) {
    val subtotal: Double get() = unitPrice * quantity
    val discount: Double get() = subtotal * (discountPercent / 100)
    val total: Double get() = subtotal - discount
}

@Serializable
data class ShippingOption(
    val id: String,
    val name: String,
    val cost: Double,
    val estimatedDays: Int
)

@Serializable
data class OrderSummary(
    val items: List<OrderItem>,
    val subtotal: Double,
    val totalDiscount: Double,
    val shippingCost: Double,
    val taxAmount: Double,
    val grandTotal: Double,
    val taxRate: Double,
    val isFreeShipping: Boolean
)

object OrderCalculator {
    private const val VAT_RATE = 7.0  // Thailand VAT 7%
    private const val FREE_SHIPPING_THRESHOLD = 1000.0

    fun calculateOrder(
        items: List<OrderItem>,
        shippingOption: ShippingOption? = null,
        promoCode: String? = null
    ): OrderSummary {
        val subtotal = items.sumOf { it.total }
        val totalDiscount = items.sumOf { it.discount }
        
        // Free shipping check
        val isFreeShipping = subtotal >= FREE_SHIPPING_THRESHOLD
        val shippingCost = when {
            isFreeShipping -> 0.0
            shippingOption != null -> shippingOption.cost
            else -> calculateDefaultShipping(subtotal)
        }
        
        // Apply promo code
        val promoDiscount = promoCode?.let { calculatePromoDiscount(it, subtotal) } ?: 0.0
        
        val taxableAmount = subtotal + shippingCost - promoDiscount
        val taxAmount = taxableAmount * (VAT_RATE / 100)
        val grandTotal = taxableAmount + taxAmount
        
        return OrderSummary(
            items = items,
            subtotal = subtotal.roundToTwoDecimals(),
            totalDiscount = (totalDiscount + promoDiscount).roundToTwoDecimals(),
            shippingCost = shippingCost.roundToTwoDecimals(),
            taxAmount = taxAmount.roundToTwoDecimals(),
            grandTotal = grandTotal.roundToTwoDecimals(),
            taxRate = VAT_RATE,
            isFreeShipping = isFreeShipping
        )
    }

    private fun calculateDefaultShipping(subtotal: Double): Double {
        return when {
            subtotal >= 500 -> 50.0
            else -> 100.0
        }
    }

    private fun calculatePromoDiscount(code: String, subtotal: Double): Double {
        // Shared promo code logic
        return when (code.uppercase()) {
            "SAVE10" -> subtotal * 0.10
            "SAVE20" -> subtotal * 0.20
            "FREESHIP" -> 0.0
            else -> 0.0
        }
    }

    private fun Double.roundToTwoDecimals(): Double {
        return (this * 100).roundToInt() / 100.0
    }
}
```

## การใช้ใน Spring Boot Backend

```kotlin
// Spring Boot - ใช้ shared validation
package com.backend.service

import com.shared.validation.EmailValidator
import com.shared.validation.PasswordValidator
import com.shared.model.UserRegistration
import org.springframework.stereotype.Service

@Service
class UserRegistrationService {

    fun registerUser(request: UserRegistration): UserProfile {
        // ใช้ shared validation logic
        val validationResult = request.validate()
        
        if (!validationResult.isValid()) {
            val errorMessages = validationResult.getErrors()
                .joinToString(", ") { "${it.field}: ${it.message}" }
            throw ValidationException("Validation failed: $errorMessages")
        }

        // ดำเนินการ register
        return createUser(request)
    }

    private fun createUser(request: UserRegistration): UserProfile {
        // ... save to database
        return UserProfile(
            id = generateId(),
            email = request.email,
            firstName = request.firstName,
            lastName = request.lastName,
            role = UserRole.USER,
            active = true,
            createdAt = now(),
            updatedAt = now()
        )
    }
}
```

## Testing Shared Code

```kotlin
// commonTest/kotlin/com/shared/validation/ValidatorTests.kt
package com.shared.validation

import kotlin.test.*

class EmailValidatorTests {

    @Test
    fun `valid email should pass`() {
        val result = EmailValidator.validate("test@example.com")
        assertTrue(result.isValid())
    }

    @Test
    fun `invalid email should fail`() {
        val result = EmailValidator.validate("not-an-email")
        assertFalse(result.isValid())
        assertEquals(1, result.getErrors().size)
        assertEquals("INVALID_FORMAT", result.getErrors().first().code)
    }

    @Test
    fun `empty email should fail with REQUIRED error`() {
        val result = EmailValidator.validate("")
        assertFalse(result.isValid())
        assertTrue(result.getErrors().any { it.code == "REQUIRED" })
    }
}

class PasswordValidatorTests {

    @Test
    fun `strong password should pass`() {
        val result = PasswordValidator.validate("SecureP@ss1")
        assertTrue(result.isValid())
    }

    @Test
    fun `short password should fail`() {
        val result = PasswordValidator.validate("Abc@1")
        assertFalse(result.isValid())
        assertTrue(result.getErrors().any { it.code == "TOO_SHORT" })
    }

    @Test
    fun `password strength calculation`() {
        assertEquals(PasswordStrength.WEAK, PasswordValidator.calculateStrength("abc"))
        assertEquals(PasswordStrength.STRONG, PasswordValidator.calculateStrength("Secure1@"))
    }
}

class OrderCalculatorTests {

    @Test
    fun `should calculate order correctly`() {
        val items = listOf(
            OrderItem("p1", "Product 1", 2, 500.0),
            OrderItem("p2", "Product 2", 1, 300.0)
        )
        
        val summary = OrderCalculator.calculateOrder(items)
        
        assertEquals(1300.0, summary.subtotal)
        assertEquals(0.0, summary.shippingCost) // Free shipping > 1000
        assertTrue(summary.isFreeShipping)
        assertEquals(91.0, summary.taxAmount) // 7% of 1300
    }

    @Test
    fun `should apply promo code`() {
        val items = listOf(OrderItem("p1", "Product 1", 1, 500.0))
        val summary = OrderCalculator.calculateOrder(items, promoCode = "SAVE10")
        
        assertEquals(50.0, summary.totalDiscount) // 10% of 500
    }
}
```

## ข้อดีของ KMP

| ข้อดี | รายละเอียด |
|------|-----------|
| Code Reuse | เขียน business logic ครั้งเดียว |
| Type Safety | Kotlin type system ทุก platform |
| Consistent Logic | Validation เหมือนกันทุกที่ |
| Easy Testing | Test shared code ใน commonTest |
| Gradual Adoption | เริ่มจาก module เล็กๆ ก่อนได้ |
| Native Performance | Compile เป็น native code |

## Platforms ที่ KMP รองรับ

| Platform | Target | ใช้สำหรับ |
|---------|--------|---------|
| JVM | `jvm()` | Spring Boot, Desktop |
| Android | `androidTarget()` | Android apps |
| iOS | `iosX64()`, `iosArm64()` | iPhone, iPad |
| JavaScript | `js(IR)` | Web, Node.js |
| macOS | `macosX64()`, `macosArm64()` | Mac Desktop |
| Linux | `linuxX64()` | Linux Desktop/Server |
| Windows | `mingwX64()` | Windows Desktop |
| WebAssembly | `wasm32()` | Web performance |

## Best Practices

1. **แยก shared code ให้ชัดเจน** - ใส่ใน `commonMain` เท่านั้น
2. **ใช้ expect/actual อย่างระมัดระวัง** - เฉพาะที่จำเป็น
3. **ทดสอบใน `commonTest`** - ครอบคลุม logic หลัก
4. **ระวัง Java-only dependencies** - ใช้ KMP alternatives
5. **ใช้ `kotlinx-serialization`** - แทน Gson (JVM only)
6. **ใช้ `kotlinx-datetime`** - แทน java.time (JVM only)
7. **ใช้ `kotlinx-coroutines`** - Cross-platform async

*Part 65/100+ | Kotlin & Spring Boot Complete Course*
