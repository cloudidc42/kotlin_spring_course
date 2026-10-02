# Part 36: Email Sending
## Spring Mail กับ Thymeleaf Templates

---

## 🎯 เป้าหมายของ Part นี้

- Setup Spring Mail กับ JavaMailSender
- สร้าง HTML email templates ด้วย Thymeleaf
- Async email sending
- Email with attachments
- ตัวอย่าง: Welcome email และ Password reset

---

## 📧 1. Email ใน Spring Boot

Spring Boot ใช้ **JavaMailSender** เป็น interface หลักสำหรับส่ง email โดยรองรับทั้ง plain text และ HTML

### Email Architecture
```
Service → JavaMailSender → SMTP Server → Recipient
           (Gmail/SES/etc)
```

---

## 📦 2. Dependencies

```kotlin
// build.gradle.kts
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-mail")
    implementation("org.springframework.boot:spring-boot-starter-thymeleaf")
    
    // Async support
    implementation("org.springframework.boot:spring-boot-starter-web")
}
```

---

## ⚙️ 3. Email Configuration

```yaml
# application.yml
spring:
  mail:
    host: smtp.gmail.com
    port: 587
    username: ${MAIL_USERNAME:your-email@gmail.com}
    password: ${MAIL_PASSWORD:your-app-password}
    properties:
      mail:
        smtp:
          auth: true
          starttls:
            enable: true
            required: true
        connection-timeout: 5000
        timeout: 5000
        write-timeout: 5000

app:
  mail:
    from: ${MAIL_FROM:noreply@example.com}
    from-name: "My App"
    base-url: ${APP_BASE_URL:http://localhost:3000}

# Development: ใช้ Mailtrap หรือ MailHog
# spring.mail.host: sandbox.smtp.mailtrap.io
# spring.mail.port: 2525
```

```kotlin
// src/main/kotlin/com/example/config/MailProperties.kt
package com.example.config

import org.springframework.boot.context.properties.ConfigurationProperties
import org.springframework.stereotype.Component

@Component
@ConfigurationProperties(prefix = "app.mail")
class MailProperties {
    var from: String = "noreply@example.com"
    var fromName: String = "My App"
    var baseUrl: String = "http://localhost:3000"
}
```

---

## 🖼️ 4. Email Templates (Thymeleaf)

```html
<!-- src/main/resources/templates/email/welcome.html -->
<!DOCTYPE html>
<html lang="th" xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>ยินดีต้อนรับ!</title>
    <style>
        body {
            font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif;
            background-color: #f4f4f4;
            margin: 0;
            padding: 0;
        }
        .container {
            max-width: 600px;
            margin: 30px auto;
            background-color: #ffffff;
            border-radius: 8px;
            overflow: hidden;
            box-shadow: 0 2px 10px rgba(0,0,0,0.1);
        }
        .header {
            background-color: #4F46E5;
            color: white;
            padding: 30px;
            text-align: center;
        }
        .header h1 {
            margin: 0;
            font-size: 28px;
        }
        .body {
            padding: 30px;
            color: #333333;
        }
        .button {
            display: inline-block;
            background-color: #4F46E5;
            color: white;
            text-decoration: none;
            padding: 14px 28px;
            border-radius: 6px;
            font-weight: bold;
            margin: 20px 0;
        }
        .footer {
            background-color: #f8f8f8;
            padding: 20px;
            text-align: center;
            font-size: 12px;
            color: #999999;
            border-top: 1px solid #eeeeee;
        }
    </style>
</head>
<body>
    <div class="container">
        <div class="header">
            <h1>🎉 ยินดีต้อนรับ!</h1>
        </div>
        <div class="body">
            <p>สวัสดีคุณ <strong th:text="${userName}">ชื่อผู้ใช้</strong>,</p>
            <p>ขอบคุณที่สมัครสมาชิกกับ <span th:text="${appName}">My App</span>!</p>
            <p>บัญชีของคุณได้รับการสร้างเรียบร้อยแล้ว กรุณายืนยัน email เพื่อเริ่มใช้งาน</p>

            <div style="text-align: center;">
                <a th:href="${verificationUrl}" class="button">
                    ยืนยัน Email ของฉัน
                </a>
            </div>

            <p>หรือ copy URL นี้ไปวางในเบราว์เซอร์:</p>
            <p style="background-color: #f4f4f4; padding: 10px; border-radius: 4px; word-break: break-all;">
                <a th:href="${verificationUrl}" th:text="${verificationUrl}">verification-url</a>
            </p>

            <p style="color: #999; font-size: 14px;">
                Link นี้จะหมดอายุใน <span th:text="${expiryHours}">24</span> ชั่วโมง
            </p>

            <p>ถ้าคุณไม่ได้สมัครสมาชิก กรุณาเพิกเฉยต่ออีเมลนี้</p>
            <p>ขอบคุณ,<br><span th:text="${appName}">My App</span> Team</p>
        </div>
        <div class="footer">
            <p>© <span th:text="${year}">2025</span> <span th:text="${appName}">My App</span>. All rights reserved.</p>
            <p>คุณได้รับอีเมลนี้เพราะคุณสมัครสมาชิกกับเรา</p>
        </div>
    </div>
</body>
</html>
```

```html
<!-- src/main/resources/templates/email/password-reset.html -->
<!DOCTYPE html>
<html lang="th" xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>Reset รหัสผ่าน</title>
    <style>
        body { font-family: Arial, sans-serif; background-color: #f4f4f4; margin: 0; padding: 0; }
        .container { max-width: 600px; margin: 30px auto; background: white; border-radius: 8px; overflow: hidden; }
        .header { background-color: #DC2626; color: white; padding: 25px; text-align: center; }
        .body { padding: 30px; color: #333; }
        .reset-code {
            font-size: 36px;
            font-weight: bold;
            letter-spacing: 8px;
            color: #DC2626;
            text-align: center;
            padding: 20px;
            background-color: #FEF2F2;
            border-radius: 8px;
            margin: 20px 0;
        }
        .button { display: inline-block; background-color: #DC2626; color: white; text-decoration: none; padding: 12px 24px; border-radius: 6px; }
        .footer { background-color: #f8f8f8; padding: 15px; text-align: center; font-size: 12px; color: #999; }
        .warning { background-color: #FFFBEB; border-left: 4px solid #F59E0B; padding: 15px; border-radius: 4px; }
    </style>
</head>
<body>
    <div class="container">
        <div class="header">
            <h1>🔐 Reset รหัสผ่าน</h1>
        </div>
        <div class="body">
            <p>สวัสดีคุณ <strong th:text="${userName}">ชื่อผู้ใช้</strong>,</p>
            <p>เราได้รับคำขอให้ reset รหัสผ่านสำหรับบัญชีของคุณ</p>

            <!-- Option 1: OTP Code -->
            <div th:if="${resetCode != null}">
                <p>กรอกรหัสนี้เพื่อ reset รหัสผ่าน:</p>
                <div class="reset-code" th:text="${resetCode}">123456</div>
            </div>

            <!-- Option 2: Reset Link -->
            <div th:if="${resetUrl != null}">
                <p>หรือกดปุ่มด้านล่างเพื่อสร้างรหัสผ่านใหม่:</p>
                <div style="text-align: center;">
                    <a th:href="${resetUrl}" class="button">Reset รหัสผ่าน</a>
                </div>
            </div>

            <div class="warning">
                <p><strong>⚠️ สำคัญ:</strong></p>
                <ul>
                    <li>Link/รหัสนี้จะหมดอายุใน <span th:text="${expiryMinutes}">30</span> นาที</li>
                    <li>ใช้ได้เพียงครั้งเดียวเท่านั้น</li>
                    <li>ถ้าคุณไม่ได้ขอ reset รหัสผ่าน กรุณาเพิกเฉยต่ออีเมลนี้</li>
                </ul>
            </div>

            <p>ถ้าต้องการความช่วยเหลือ กรุณาติดต่อ support@example.com</p>
        </div>
        <div class="footer">
            <p>อีเมลนี้ส่งอัตโนมัติ กรุณาอย่าตอบกลับ</p>
        </div>
    </div>
</body>
</html>
```

```html
<!-- src/main/resources/templates/email/order-confirmation.html -->
<!DOCTYPE html>
<html lang="th" xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>ยืนยันการสั่งซื้อ</title>
    <style>
        body { font-family: Arial, sans-serif; background-color: #f4f4f4; margin: 0; padding: 0; }
        .container { max-width: 600px; margin: 30px auto; background: white; border-radius: 8px; overflow: hidden; }
        .header { background-color: #059669; color: white; padding: 25px; text-align: center; }
        .body { padding: 25px; color: #333; }
        table { width: 100%; border-collapse: collapse; margin: 15px 0; }
        th { background-color: #f3f4f6; padding: 10px; text-align: left; font-weight: bold; }
        td { padding: 10px; border-bottom: 1px solid #e5e7eb; }
        .total-row { font-weight: bold; background-color: #ecfdf5; }
        .footer { background-color: #f8f8f8; padding: 15px; text-align: center; font-size: 12px; color: #999; }
    </style>
</head>
<body>
    <div class="container">
        <div class="header">
            <h1>✅ ยืนยันการสั่งซื้อ</h1>
            <p>หมายเลขคำสั่งซื้อ: <strong th:text="${orderId}">ORD-001</strong></p>
        </div>
        <div class="body">
            <p>สวัสดีคุณ <strong th:text="${customerName}">ลูกค้า</strong>,</p>
            <p>ขอบคุณสำหรับการสั่งซื้อ! เราได้รับคำสั่งซื้อของคุณแล้ว</p>

            <h3>รายการสินค้า</h3>
            <table>
                <thead>
                    <tr>
                        <th>สินค้า</th>
                        <th style="text-align: center;">จำนวน</th>
                        <th style="text-align: right;">ราคา</th>
                    </tr>
                </thead>
                <tbody>
                    <tr th:each="item : ${orderItems}">
                        <td th:text="${item.productName}">Product Name</td>
                        <td style="text-align: center;" th:text="${item.quantity}">1</td>
                        <td style="text-align: right;" th:text="'฿' + ${#numbers.formatDecimal(item.totalPrice, 1, 'COMMA', 2, 'POINT')}">฿0.00</td>
                    </tr>
                    <tr class="total-row">
                        <td colspan="2">ยอดรวม</td>
                        <td style="text-align: right;" th:text="'฿' + ${#numbers.formatDecimal(totalAmount, 1, 'COMMA', 2, 'POINT')}">฿0.00</td>
                    </tr>
                </tbody>
            </table>

            <p>สถานะ: <strong th:text="${status}">Processing</strong></p>
            <p>วันที่สั่งซื้อ: <span th:text="${#temporals.format(orderDate, 'dd/MM/yyyy HH:mm')}">01/01/2025</span></p>
        </div>
        <div class="footer">
            <p>© 2025 My Shop. ขอบคุณที่ใช้บริการ</p>
        </div>
    </div>
</body>
</html>
```

---

## 📨 5. Email Service

```kotlin
// src/main/kotlin/com/example/service/EmailService.kt
package com.example.service

import com.example.config.MailProperties
import com.example.dto.OrderDto
import jakarta.mail.internet.MimeMessage
import org.springframework.mail.MailException
import org.springframework.mail.SimpleMailMessage
import org.springframework.mail.javamail.JavaMailSender
import org.springframework.mail.javamail.MimeMessageHelper
import org.springframework.scheduling.annotation.Async
import org.springframework.stereotype.Service
import org.thymeleaf.context.Context
import org.thymeleaf.spring6.SpringTemplateEngine
import java.io.File
import java.time.LocalDate
import java.util.concurrent.CompletableFuture

@Service
class EmailService(
    private val mailSender: JavaMailSender,
    private val templateEngine: SpringTemplateEngine,
    private val mailProperties: MailProperties
) {

    // Simple text email
    fun sendSimpleEmail(to: String, subject: String, body: String) {
        val message = SimpleMailMessage().apply {
            from = "${mailProperties.fromName} <${mailProperties.from}>"
            setTo(to)
            this.subject = subject
            text = body
        }
        mailSender.send(message)
    }

    // HTML email ด้วย Thymeleaf template
    fun sendHtmlEmail(
        to: String,
        subject: String,
        templateName: String,
        variables: Map<String, Any>
    ) {
        val context = Context().apply {
            setVariables(variables)
        }

        val htmlContent = templateEngine.process("email/$templateName", context)

        val message: MimeMessage = mailSender.createMimeMessage()
        val helper = MimeMessageHelper(message, true, "UTF-8").apply {
            setFrom("${mailProperties.fromName} <${mailProperties.from}>")
            setTo(to)
            setSubject(subject)
            setText(htmlContent, true)  // true = HTML
        }

        mailSender.send(message)
    }

    // Async email ส่งในพื้นหลัง
    @Async
    fun sendHtmlEmailAsync(
        to: String,
        subject: String,
        templateName: String,
        variables: Map<String, Any>
    ): CompletableFuture<Boolean> {
        return try {
            sendHtmlEmail(to, subject, templateName, variables)
            CompletableFuture.completedFuture(true)
        } catch (e: MailException) {
            println("Failed to send email to $to: ${e.message}")
            CompletableFuture.completedFuture(false)
        }
    }

    // Email พร้อม attachment
    fun sendEmailWithAttachment(
        to: String,
        subject: String,
        body: String,
        attachmentName: String,
        attachmentFile: File
    ) {
        val message: MimeMessage = mailSender.createMimeMessage()
        val helper = MimeMessageHelper(message, true, "UTF-8").apply {
            setFrom("${mailProperties.fromName} <${mailProperties.from}>")
            setTo(to)
            setSubject(subject)
            setText(body, false)
            addAttachment(attachmentName, attachmentFile)
        }
        mailSender.send(message)
    }

    // Email พร้อม inline image
    fun sendEmailWithInlineImage(
        to: String,
        subject: String,
        htmlContent: String,
        imageId: String,
        imageFile: File
    ) {
        val message: MimeMessage = mailSender.createMimeMessage()
        val helper = MimeMessageHelper(message, true, "UTF-8").apply {
            setFrom("${mailProperties.fromName} <${mailProperties.from}>")
            setTo(to)
            setSubject(subject)
            setText(htmlContent, true)
            addInline(imageId, imageFile)  // cid:imageId ใน HTML
        }
        mailSender.send(message)
    }

    // ===== Specific Email Methods =====

    @Async
    fun sendWelcomeEmail(
        to: String,
        userName: String,
        verificationToken: String
    ): CompletableFuture<Boolean> {
        val verificationUrl = "${mailProperties.baseUrl}/verify-email?token=$verificationToken"

        val variables = mapOf(
            "userName" to userName,
            "appName" to mailProperties.fromName,
            "verificationUrl" to verificationUrl,
            "expiryHours" to 24,
            "year" to LocalDate.now().year
        )

        return sendHtmlEmailAsync(
            to = to,
            subject = "ยินดีต้อนรับสู่ ${mailProperties.fromName}!",
            templateName = "welcome",
            variables = variables
        )
    }

    @Async
    fun sendPasswordResetEmail(
        to: String,
        userName: String,
        resetToken: String
    ): CompletableFuture<Boolean> {
        val resetUrl = "${mailProperties.baseUrl}/reset-password?token=$resetToken"

        val variables = mapOf(
            "userName" to userName,
            "resetUrl" to resetUrl,
            "resetCode" to null,  // หรือส่ง OTP code แทน
            "expiryMinutes" to 30
        )

        return sendHtmlEmailAsync(
            to = to,
            subject = "Reset รหัสผ่าน - ${mailProperties.fromName}",
            templateName = "password-reset",
            variables = variables
        )
    }

    @Async
    fun sendOrderConfirmationEmail(
        to: String,
        order: OrderDto
    ): CompletableFuture<Boolean> {
        val variables = mapOf(
            "orderId" to "ORD-${order.id}",
            "customerName" to order.customerName,
            "orderItems" to order.items,
            "totalAmount" to order.totalAmount,
            "status" to order.status,
            "orderDate" to order.createdAt
        )

        return sendHtmlEmailAsync(
            to = to,
            subject = "ยืนยันการสั่งซื้อ #ORD-${order.id}",
            templateName = "order-confirmation",
            variables = variables
        )
    }
}
```

---

## ⚙️ 6. Async Configuration

```kotlin
// src/main/kotlin/com/example/config/AsyncConfig.kt
package com.example.config

import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.scheduling.annotation.EnableAsync
import org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor
import java.util.concurrent.Executor

@Configuration
@EnableAsync
class AsyncConfig {

    // Thread pool สำหรับ async email
    @Bean("emailTaskExecutor")
    fun emailTaskExecutor(): Executor {
        return ThreadPoolTaskExecutor().apply {
            corePoolSize = 2
            maxPoolSize = 5
            queueCapacity = 100
            setThreadNamePrefix("email-")
            initialize()
        }
    }
}
```

---

## 📬 7. Email Queue Service (ด้วย Database)

```kotlin
// src/main/kotlin/com/example/entity/EmailQueue.kt
package com.example.entity

import jakarta.persistence.*
import java.time.LocalDateTime

@Entity
@Table(name = "email_queue")
data class EmailQueue(
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long = 0,

    val recipientEmail: String,
    val subject: String,
    val templateName: String,

    @Column(columnDefinition = "TEXT")
    val variablesJson: String,

    @Enumerated(EnumType.STRING)
    var status: EmailStatus = EmailStatus.PENDING,

    var attempts: Int = 0,
    var maxAttempts: Int = 3,

    var scheduledAt: LocalDateTime = LocalDateTime.now(),
    var sentAt: LocalDateTime? = null,
    var errorMessage: String? = null,

    val createdAt: LocalDateTime = LocalDateTime.now()
)

enum class EmailStatus { PENDING, SENT, FAILED, CANCELLED }
```

```kotlin
// src/main/kotlin/com/example/service/EmailQueueService.kt
package com.example.service

import com.example.entity.EmailQueue
import com.example.entity.EmailStatus
import com.example.repository.EmailQueueRepository
import com.fasterxml.jackson.databind.ObjectMapper
import com.fasterxml.jackson.module.kotlin.readValue
import org.springframework.scheduling.annotation.Scheduled
import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional
import java.time.LocalDateTime

@Service
class EmailQueueService(
    private val emailQueueRepository: EmailQueueRepository,
    private val emailService: EmailService,
    private val objectMapper: ObjectMapper
) {

    // เพิ่ม email เข้า queue
    fun enqueue(
        to: String,
        subject: String,
        templateName: String,
        variables: Map<String, Any>,
        scheduledAt: LocalDateTime = LocalDateTime.now()
    ) {
        val emailQueue = EmailQueue(
            recipientEmail = to,
            subject = subject,
            templateName = templateName,
            variablesJson = objectMapper.writeValueAsString(variables),
            scheduledAt = scheduledAt
        )
        emailQueueRepository.save(emailQueue)
    }

    // Process queue ทุก 1 นาที
    @Scheduled(fixedDelay = 60000)
    @Transactional
    fun processEmailQueue() {
        val pendingEmails = emailQueueRepository
            .findByStatusAndScheduledAtBefore(EmailStatus.PENDING, LocalDateTime.now())

        pendingEmails.forEach { email ->
            try {
                val variables: Map<String, Any> = objectMapper.readValue(email.variablesJson)
                emailService.sendHtmlEmail(
                    email.recipientEmail,
                    email.subject,
                    email.templateName,
                    variables
                )
                email.status = EmailStatus.SENT
                email.sentAt = LocalDateTime.now()
            } catch (e: Exception) {
                email.attempts++
                if (email.attempts >= email.maxAttempts) {
                    email.status = EmailStatus.FAILED
                    email.errorMessage = e.message
                }
                println("Failed to send email ${email.id}: ${e.message}")
            }
            emailQueueRepository.save(email)
        }
    }
}
```

---

## 🧪 8. Testing Email

```kotlin
// src/test/kotlin/com/example/service/EmailServiceTest.kt
package com.example.service

import com.icegreen.greenmail.junit5.GreenMailExtension
import com.icegreen.greenmail.util.ServerSetupTest
import org.junit.jupiter.api.Test
import org.junit.jupiter.api.extension.RegisterExtension
import org.springframework.beans.factory.annotation.Autowired
import org.springframework.boot.test.context.SpringBootTest
import org.springframework.test.context.TestPropertySource
import kotlin.test.assertEquals
import kotlin.test.assertTrue

@SpringBootTest
@TestPropertySource(properties = [
    "spring.mail.host=localhost",
    "spring.mail.port=3025",
    "spring.mail.username=test",
    "spring.mail.password=test"
])
class EmailServiceTest {

    companion object {
        @JvmField
        @RegisterExtension
        val greenMail = GreenMailExtension(ServerSetupTest.SMTP)
    }

    @Autowired
    lateinit var emailService: EmailService

    @Test
    fun `should send welcome email`() {
        emailService.sendWelcomeEmail(
            to = "user@example.com",
            userName = "Test User",
            verificationToken = "test-token-123"
        ).get()  // รอ async

        val messages = greenMail.receivedMessages
        assertEquals(1, messages.size)
        assertTrue(messages[0].subject.contains("ยินดีต้อนรับ"))
    }

    @Test
    fun `should send password reset email`() {
        emailService.sendPasswordResetEmail(
            to = "user@example.com",
            userName = "Test User",
            resetToken = "reset-token-456"
        ).get()

        val messages = greenMail.receivedMessages
        assertTrue(messages.any { it.subject.contains("Reset รหัสผ่าน") })
    }
}
```

---

## 📋 สรุป

| Email Type | Method | Template |
|-----------|--------|---------|
| Welcome + Verification | `sendWelcomeEmail()` | `welcome.html` |
| Password Reset | `sendPasswordResetEmail()` | `password-reset.html` |
| Order Confirmation | `sendOrderConfirmationEmail()` | `order-confirmation.html` |
| Simple Text | `sendSimpleEmail()` | - |
| With Attachment | `sendEmailWithAttachment()` | - |

### Best Practices

| ข้อควรปฏิบัติ | เหตุผล |
|-------------|--------|
| ส่ง async เสมอ | ไม่บล็อค request |
| ใช้ queue สำหรับ production | retry ได้เมื่อล้มเหลว |
| Test ด้วย GreenMail/Mailtrap | ไม่ต้องใช้ email จริง |
| ใช้ environment variables | ปลอดภัยสำหรับ credentials |
| Rate limiting | ป้องกัน spam |

---

*Part 36/100+ | Kotlin & Spring Boot Complete Course*
