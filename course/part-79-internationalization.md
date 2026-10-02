# Part 79: Internationalization (i18n)

## Internationalization — รองรับหลายภาษาใน Spring Boot API

---

## 🎯 เป้าหมายของ Part นี้

- MessageSource สำหรับ i18n
- LocaleContextHolder
- Locale resolution strategies
- Date/Number/Currency formatting
- สร้าง Multi-language API

---

## 📖 1. i18n คืออะไร?

**Internationalization (i18n)** = กระบวนการออกแบบ software ให้สามารถปรับใช้กับ locale ต่างๆ ได้

**Localization (L10n)** = การปรับ software สำหรับ locale เฉพาะ (แปลภาษา, format วันที่ ฯลฯ)

### Locale ประกอบด้วย
- **Language**: `th`, `en`, `ja`, `zh`
- **Country/Region**: `TH`, `US`, `JP`, `CN`
- **Variant**: `th_TH`, `en_US`, `ja_JP`

---

## ⚙️ 2. Setup MessageSource

### application.yml

```yaml
spring:
  messages:
    basename: i18n/messages
    encoding: UTF-8
    fallback-to-system-locale: false
    use-code-as-default-message: false
    cache-duration: 3600s
```

### Messages Files

```
src/main/resources/i18n/
├── messages.properties          (default/English)
├── messages_th.properties       (Thai)
├── messages_ja.properties       (Japanese)
└── messages_zh.properties       (Chinese)
```

### messages.properties (Default)

```properties
# Common
common.success=Operation completed successfully
common.error=An error occurred
common.not.found={0} with id {1} not found
common.required={0} is required
common.invalid={0} is invalid

# User
user.created=User created successfully
user.updated=User profile updated
user.deleted=User account deleted
user.not.found=User not found
user.email.exists=Email address already registered
user.password.weak=Password must be at least 8 characters

# Product
product.created=Product {0} added to catalog
product.out.of.stock=Product {0} is out of stock
product.price.updated=Price updated from {0} to {1} {2}

# Order
order.placed=Order #{0} placed successfully. Total: {1}
order.cancelled=Order #{0} has been cancelled
order.shipped=Your order #{0} has been shipped

# Validation
validation.email=Please provide a valid email address
validation.phone=Please provide a valid phone number
validation.date.format=Date must be in format {0}
```

### messages_th.properties (Thai)

```properties
# Common
common.success=ดำเนินการสำเร็จ
common.error=เกิดข้อผิดพลาด
common.not.found=ไม่พบ{0} รหัส {1}
common.required={0} ต้องไม่ว่างเปล่า
common.invalid={0} ไม่ถูกต้อง

# User
user.created=สร้างบัญชีผู้ใช้สำเร็จ
user.updated=อัพเดตข้อมูลผู้ใช้แล้ว
user.deleted=ลบบัญชีผู้ใช้แล้ว
user.not.found=ไม่พบผู้ใช้งาน
user.email.exists=อีเมลนี้ถูกใช้งานแล้ว
user.password.weak=รหัสผ่านต้องมีความยาวอย่างน้อย 8 ตัวอักษร

# Product
product.created=เพิ่มสินค้า {0} ในแค็ตตาล็อกแล้ว
product.out.of.stock=สินค้า {0} หมดสต็อก
product.price.updated=อัพเดตราคาจาก {0} เป็น {1} {2}

# Order
order.placed=สั่งซื้อสำเร็จ หมายเลขคำสั่งซื้อ #{0} ยอดรวม: {1}
order.cancelled=ยกเลิกคำสั่งซื้อ #{0} แล้ว
order.shipped=จัดส่งคำสั่งซื้อ #{0} แล้ว

# Validation
validation.email=กรุณากรอกอีเมลที่ถูกต้อง
validation.phone=กรุณากรอกหมายเลขโทรศัพท์ที่ถูกต้อง
validation.date.format=รูปแบบวันที่ต้องเป็น {0}
```

---

## 🌐 3. Locale Resolution Strategies

### Multi-Strategy Resolver

```kotlin
@Configuration
class LocaleConfig {

    @Bean
    fun localeResolver(): LocaleResolver {
        val resolver = HeaderLocaleResolver()
        resolver.setDefaultLocale(Locale("th", "TH"))
        return resolver
    }
}

@Component
class CompositeLocaleResolver : LocaleResolver {

    override fun resolveLocale(request: HttpServletRequest): Locale {
        // 1. จาก query parameter: ?lang=th
        request.getParameter("lang")?.let { lang ->
            return parseLocale(lang) ?: return@let
        }

        // 2. จาก cookie: locale=th_TH
        request.cookies?.find { it.name == "locale" }?.let { cookie ->
            return parseLocale(cookie.value) ?: return@let
        }

        // 3. จาก header: Accept-Language: th-TH,th;q=0.9
        request.getHeader("Accept-Language")?.let { header ->
            Locale.lookup(Locale.LanguageRange.parse(header), supportedLocales)?.let {
                return it
            }
        }

        // 4. จาก user preferences (ถ้า authenticated)
        // userPreferenceService.getLocale(userId)

        return Locale("th", "TH") // default
    }

    override fun setLocale(
        request: HttpServletRequest,
        response: HttpServletResponse?,
        locale: Locale?
    ) {
        locale?.let {
            response?.addCookie(Cookie("locale", it.toLanguageTag()).apply {
                maxAge = 365 * 24 * 60 * 60
                path = "/"
            })
        }
    }

    private fun parseLocale(lang: String): Locale? {
        return try {
            Locale.forLanguageTag(lang).takeIf { it != Locale.ROOT }
        } catch (e: Exception) {
            null
        }
    }

    private val supportedLocales = listOf(
        Locale("th", "TH"),
        Locale("en", "US"),
        Locale("ja", "JP"),
        Locale.ENGLISH
    )
}
```

---

## 💬 4. MessageService

```kotlin
@Service
class MessageService(
    private val messageSource: MessageSource
) {

    fun getMessage(code: String, vararg args: Any?, locale: Locale? = null): String {
        val resolvedLocale = locale ?: LocaleContextHolder.getLocale()
        return try {
            messageSource.getMessage(code, args.map { it?.toString() }.toTypedArray(), resolvedLocale)
        } catch (ex: NoSuchMessageException) {
            messageSource.getMessage(code, args.map { it?.toString() }.toTypedArray(), Locale.ENGLISH)
        }
    }

    fun getLocaleFromRequest(): Locale = LocaleContextHolder.getLocale()
}
```

---

## 💰 5. Date/Number/Currency Formatting

```kotlin
@Service
class LocalizationService {

    fun formatCurrency(amount: Double, currency: String, locale: Locale): String {
        val formatter = NumberFormat.getCurrencyInstance(locale)
        formatter.currency = java.util.Currency.getInstance(currency)
        return formatter.format(amount)
    }

    fun formatNumber(number: Double, locale: Locale, decimals: Int = 2): String {
        val formatter = NumberFormat.getNumberInstance(locale)
        formatter.minimumFractionDigits = decimals
        formatter.maximumFractionDigits = decimals
        return formatter.format(number)
    }

    fun formatDate(date: java.time.LocalDate, locale: Locale, style: FormatStyle = FormatStyle.MEDIUM): String {
        return java.time.format.DateTimeFormatter
            .ofLocalizedDate(style)
            .withLocale(locale)
            .format(date)
    }

    fun formatDateTime(dateTime: java.time.LocalDateTime, locale: Locale): String {
        return java.time.format.DateTimeFormatter
            .ofLocalizedDateTime(FormatStyle.MEDIUM)
            .withLocale(locale)
            .format(dateTime)
    }

    fun getRelativeTime(timestamp: java.time.Instant, locale: Locale): String {
        val now = java.time.Instant.now()
        val duration = java.time.Duration.between(timestamp, now)

        return when {
            duration.toMinutes() < 1 -> getMessage("time.just.now", locale = locale)
            duration.toMinutes() < 60 -> getMessage("time.minutes.ago", duration.toMinutes(), locale = locale)
            duration.toHours() < 24 -> getMessage("time.hours.ago", duration.toHours(), locale = locale)
            duration.toDays() < 30 -> getMessage("time.days.ago", duration.toDays(), locale = locale)
            else -> formatDate(timestamp.atZone(java.time.ZoneId.systemDefault()).toLocalDate(), locale)
        }
    }

    private fun getMessage(code: String, vararg args: Any?, locale: Locale): String = ""
}
```

---

## 🔧 6. Localized Response Format

```kotlin
// Generic response wrapper ที่รองรับ i18n
data class ApiResponse<T>(
    val success: Boolean,
    val message: String,
    val data: T? = null,
    val errors: List<String> = emptyList(),
    val locale: String? = null
)

@Component
class LocalizedResponseBuilder(private val messageService: MessageService) {

    fun <T> success(
        messageCode: String,
        vararg messageArgs: Any?,
        data: T? = null,
        locale: Locale? = null
    ): ApiResponse<T> {
        return ApiResponse(
            success = true,
            message = messageService.getMessage(messageCode, *messageArgs, locale = locale),
            data = data,
            locale = (locale ?: LocaleContextHolder.getLocale()).toLanguageTag()
        )
    }

    fun <T> error(
        messageCode: String,
        vararg messageArgs: Any?,
        locale: Locale? = null
    ): ApiResponse<T> {
        return ApiResponse(
            success = false,
            message = messageService.getMessage(messageCode, *messageArgs, locale = locale),
            locale = (locale ?: LocaleContextHolder.getLocale()).toLanguageTag()
        )
    }
}
```

---

## 🌍 7. Multi-language Product API

```kotlin
@RestController
@RequestMapping("/api/v1/products")
class ProductController(
    private val productService: ProductService,
    private val responseBuilder: LocalizedResponseBuilder,
    private val localizationService: LocalizationService
) {

    @GetMapping("/{id}")
    fun getProduct(
        @PathVariable id: Long,
        @RequestHeader(value = "Accept-Language", defaultValue = "th-TH") language: String
    ): ResponseEntity<ApiResponse<LocalizedProductResponse>> {
        val locale = Locale.forLanguageTag(language)
        val product = productService.findById(id)

        val response = LocalizedProductResponse(
            id = product.id,
            name = product.getName(locale),
            description = product.getDescription(locale),
            price = localizationService.formatCurrency(product.price, product.currency, locale),
            priceNumeric = product.price,
            currency = product.currency,
            category = product.getCategory(locale),
            inStock = product.stock > 0,
            stockMessage = if (product.stock > 0)
                "${product.stock} items available"
            else
                "Out of stock"
        )

        return ResponseEntity.ok(responseBuilder.success("product.found", data = response))
    }

    @PostMapping
    fun createProduct(
        @RequestBody @Valid request: CreateProductRequest
    ): ResponseEntity<ApiResponse<ProductResponse>> {
        val product = productService.create(request)
        val locale = LocaleContextHolder.getLocale()

        return ResponseEntity
            .status(HttpStatus.CREATED)
            .body(responseBuilder.success(
                "product.created",
                product.name,
                data = product.toResponse()
            ))
    }
}

data class LocalizedProductResponse(
    val id: Long,
    val name: String,
    val description: String,
    val price: String,          // formatted: "฿25,000.00"
    val priceNumeric: Double,   // raw: 25000.0
    val currency: String,
    val category: String,
    val inStock: Boolean,
    val stockMessage: String
)
```

---

## 📊 8. สรุปตาราง i18n Components

| Component | คำอธิบาย | ตัวอย่าง |
|-----------|---------|---------|
| MessageSource | อ่าน messages จากไฟล์ | `getMessage("user.created")` |
| LocaleContextHolder | Thread-local locale | `LocaleContextHolder.getLocale()` |
| LocaleResolver | ตัดสินว่า locale คืออะไร | Header, Cookie, URL |
| LocaleChangeInterceptor | เปลี่ยน locale จาก request | `?lang=en` |
| MessageInterpolator | interpolate validation messages | Bean Validation |
| DateTimeFormatter | Format วันที่ตาม locale | `ofLocalizedDate()` |
| NumberFormat | Format ตัวเลขตาม locale | `getCurrencyInstance()` |

---

## 💡 Best Practices

1. **UTF-8 encoding** สำหรับ message files ที่มีภาษาไทย
2. **Fallback chain** — th_TH → th → en → default
3. **ICU message format** สำหรับ plural forms และ gender
4. **Externalize all user-visible strings** — ไม่ hardcode
5. **Test all supported locales** ใน CI pipeline

---

## 🧪 8. Testing i18n

```kotlin
@SpringBootTest
class LocalizationTest {

    @Autowired
    private lateinit var messageService: MessageService

    @Test
    fun `English message is returned for en locale`() {
        val message = messageService.getMessage("user.created", locale = Locale.ENGLISH)
        assertThat(message).isEqualTo("User created successfully")
    }

    @Test
    fun `Thai message is returned for th locale`() {
        val thaiLocale = Locale("th", "TH")
        val message = messageService.getMessage("user.created", locale = thaiLocale)
        assertThat(message).isEqualTo("สร้างบัญชีผู้ใช้สำเร็จ")
    }

    @Test
    fun `message with arguments is formatted correctly`() {
        val message = messageService.getMessage(
            "product.created",
            "Laptop",
            locale = Locale.ENGLISH
        )
        assertThat(message).contains("Laptop")
    }

    @Test
    fun `currency formatting respects locale`() {
        val service = LocalizationService()
        val thaiFormat = service.formatCurrency(25000.0, "THB", Locale("th", "TH"))
        val usFormat = service.formatCurrency(25000.0, "USD", Locale.US)
        
        assertThat(thaiFormat).contains("25")
        assertThat(usFormat).contains("25")
    }
}
```

---

## 🌏 9. Timezone Handling

Timezone มักจะสร้างปัญหาใน multi-locale APIs

```kotlin
@Service
class TimezoneAwareService {

    fun formatEventTime(
        utcTime: java.time.Instant,
        userTimezone: String,
        locale: Locale
    ): String {
        val zoneId = try {
            java.time.ZoneId.of(userTimezone)
        } catch (ex: Exception) {
            java.time.ZoneId.of("UTC")
        }

        val zonedDateTime = utcTime.atZone(zoneId)
        return java.time.format.DateTimeFormatter
            .ofLocalizedDateTime(java.time.format.FormatStyle.MEDIUM)
            .withLocale(locale)
            .withZone(zoneId)
            .format(zonedDateTime)
    }

    fun getUserTimezone(userId: Long): String {
        return userRepository.findById(userId)?.timezone ?: "Asia/Bangkok"
    }
}
```

### Common Thai Timezones

```kotlin
object ThaiTimezones {
    const val BANGKOK = "Asia/Bangkok"          // UTC+7
    const val YANGON = "Asia/Yangon"            // UTC+6:30
    const val SINGAPORE = "Asia/Singapore"      // UTC+8
    const val TOKYO = "Asia/Tokyo"              // UTC+9
    const val KOLKATA = "Asia/Kolkata"          // UTC+5:30
}
```

---

## 📝 10. Validation Messages i18n

```kotlin
// CustomValidators.kt
@Target(AnnotationTarget.FIELD, AnnotationTarget.VALUE_PARAMETER)
@Retention(AnnotationRetention.RUNTIME)
@Constraint(validatedBy = [ThaiPhoneValidator::class])
annotation class ThaiPhone(
    val message: String = "{validation.thai.phone}",
    val groups: Array<KClass<*>> = [],
    val payload: Array<KClass<out Payload>> = []
)

class ThaiPhoneValidator : ConstraintValidator<ThaiPhone, String?> {
    override fun isValid(value: String?, context: ConstraintValidatorContext): Boolean {
        if (value == null) return true
        return value.matches(Regex("^(\\+66|0)[6-9]\\d{8}$"))
    }
}

// Messages สำหรับ validation
// messages.properties
// validation.thai.phone=Please enter a valid Thai phone number (e.g., 0812345678)
// messages_th.properties
// validation.thai.phone=กรุณากรอกหมายเลขโทรศัพท์ไทยที่ถูกต้อง (เช่น 0812345678)

// Request DTO
data class UserRegistrationRequest(
    @field:NotBlank(message = "{common.required}")
    val name: String,

    @field:Email(message = "{validation.email}")
    val email: String,

    @field:ThaiPhone
    val phone: String?,

    @field:Size(min = 8, message = "{user.password.weak}")
    val password: String
)
```

---

## 📊 11. Supported Locales Configuration

```kotlin
@Configuration
class LocaleConfiguration {

    val supportedLocales = listOf(
        Locale("th", "TH"),
        Locale("en", "US"),
        Locale("en", "GB"),
        Locale("ja", "JP"),
        Locale("zh", "CN"),
        Locale("zh", "TW"),
        Locale("ko", "KR"),
        Locale("ms", "MY"),
        Locale("id", "ID"),
        Locale("vi", "VN")
    )

    @Bean
    fun localeChangeInterceptor(): LocaleChangeInterceptor {
        return LocaleChangeInterceptor().apply {
            paramName = "lang"
        }
    }

    @Bean
    fun handlerMapping(localeChangeInterceptor: LocaleChangeInterceptor): HandlerMapping {
        return RequestMappingHandlerMapping().apply {
            interceptors = arrayOf(localeChangeInterceptor)
        }
    }
}
```

---

*Part 79/100+ | Kotlin & Spring Boot Complete Course*
