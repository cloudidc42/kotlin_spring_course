# Part 37: Scheduled Tasks
## Cron Jobs และ Background Tasks ใน Spring Boot

---

## 🎯 เป้าหมายของ Part นี้

- @Scheduled annotation (fixedRate, fixedDelay, cron)
- Async scheduled tasks
- SchedulingConfigurer
- Distributed lock สำหรับ scheduled tasks
- Dynamic scheduling
- ตัวอย่าง: Daily report, cleanup jobs

---

## ⏰ 1. Spring Scheduling Overview

Spring Scheduling ช่วยให้ run tasks ในเวลาที่กำหนดโดยอัตโนมัติ ไม่ต้องพึ่ง external scheduler

### ประเภทของ Scheduling
- **Fixed Rate**: รันทุกๆ X milliseconds (นับจากต้น)
- **Fixed Delay**: รันทุกๆ X milliseconds (นับจากสิ้นสุด)
- **Cron Expression**: รันตามตาราง cron

---

## ⚙️ 2. Enable Scheduling

```kotlin
// src/main/kotlin/com/example/Application.kt
package com.example

import org.springframework.boot.autoconfigure.SpringBootApplication
import org.springframework.boot.runApplication
import org.springframework.scheduling.annotation.EnableScheduling

@SpringBootApplication
@EnableScheduling  // เปิดใช้งาน scheduling
class Application

fun main(args: Array<String>) {
    runApplication<Application>(*args)
}
```

```kotlin
// src/main/kotlin/com/example/config/SchedulingConfig.kt
package com.example.config

import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.scheduling.annotation.EnableScheduling
import org.springframework.scheduling.concurrent.ThreadPoolTaskScheduler

@Configuration
@EnableScheduling
class SchedulingConfig {

    // Thread pool สำหรับ scheduled tasks
    @Bean
    fun taskScheduler(): ThreadPoolTaskScheduler {
        return ThreadPoolTaskScheduler().apply {
            poolSize = 5
            setThreadNamePrefix("scheduled-task-")
            setErrorHandler { t ->
                println("Scheduled task error: ${t.message}")
                // ส่ง alert หรือ log ที่นี่
            }
            initialize()
        }
    }
}
```

---

## 📝 3. @Scheduled Annotations

```kotlin
// src/main/kotlin/com/example/scheduler/BasicScheduler.kt
package com.example.scheduler

import org.springframework.scheduling.annotation.Scheduled
import org.springframework.stereotype.Component
import java.time.LocalDateTime
import java.time.format.DateTimeFormatter

@Component
class BasicScheduler {

    // ===== Fixed Rate =====

    // รันทุก 5 วินาที (นับจากเวลาเริ่มต้นของ task ก่อน)
    @Scheduled(fixedRate = 5000)
    fun fixedRateTask() {
        println("[fixedRate] Running at: ${now()}")
        Thread.sleep(1000)  // จำลอง task ที่ใช้เวลา 1 วินาที
        // ถ้า task ใช้เวลานานกว่า rate → tasks จะซ้อนกัน!
    }

    // รันทุก 5 วินาที แต่รอ 10 วินาทีก่อนครั้งแรก
    @Scheduled(fixedRate = 5000, initialDelay = 10000)
    fun fixedRateWithDelay() {
        println("[fixedRateWithDelay] Running at: ${now()}")
    }

    // ===== Fixed Delay =====

    // รัน และรอ 5 วินาทีหลังจาก task สิ้นสุด ก่อนรอบต่อไป
    @Scheduled(fixedDelay = 5000)
    fun fixedDelayTask() {
        println("[fixedDelay] Start: ${now()}")
        Thread.sleep(2000)  // task ใช้เวลา 2 วินาที
        println("[fixedDelay] End: ${now()}")
        // รอบต่อไปจะรัน 5 วินาทีหลังจาก "End"
    }

    // ===== Cron =====

    // รันทุกนาที
    @Scheduled(cron = "0 * * * * *")
    fun everyMinute() {
        println("[cron] Every minute: ${now()}")
    }

    // รันทุก 30 นาที
    @Scheduled(cron = "0 */30 * * * *")
    fun everyThirtyMinutes() {
        println("[cron] Every 30 minutes: ${now()}")
    }

    // รันทุกวันเวลาเที่ยงคืน
    @Scheduled(cron = "0 0 0 * * *")
    fun everyDayAtMidnight() {
        println("[cron] Midnight: ${now()}")
    }

    // รันทุกวันจันทร์-ศุกร์ เวลา 8:00
    @Scheduled(cron = "0 0 8 * * MON-FRI")
    fun weekdaysAt8am() {
        println("[cron] Weekday 8am: ${now()}")
    }

    // รัน timezone-specific
    @Scheduled(cron = "0 0 9 * * *", zone = "Asia/Bangkok")
    fun bangkokTime9am() {
        println("[cron] Bangkok 9am: ${now()}")
    }

    // Cron ด้วย SpEL จาก properties
    @Scheduled(cron = "\${app.scheduler.daily-report.cron:0 0 6 * * *}")
    fun configuredCron() {
        println("[cron] Configured cron: ${now()}")
    }

    private fun now(): String {
        return LocalDateTime.now().format(DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss"))
    }
}
```

### Cron Expression Format
```
Cron: "second minute hour day-of-month month day-of-week"

Examples:
"0 0 * * * *"         - ทุกชั่วโมง
"*/10 * * * * *"      - ทุก 10 วินาที
"0 0 8 * * *"         - ทุกวัน 8:00
"0 0 8 * * MON"       - ทุกวันจันทร์ 8:00
"0 0 0 1 * *"         - วันที่ 1 ของทุกเดือน 00:00
"0 0 0 * * MON-FRI"   - วันจันทร์-ศุกร์ 00:00
"0 0 0 L * *"         - วันสุดท้ายของเดือน 00:00
```

---

## ⚡ 4. Async Scheduled Tasks

```kotlin
// src/main/kotlin/com/example/scheduler/AsyncScheduler.kt
package com.example.scheduler

import org.springframework.scheduling.annotation.Async
import org.springframework.scheduling.annotation.Scheduled
import org.springframework.stereotype.Component

@Component
class AsyncScheduler {

    // @Async + @Scheduled = รันใน thread แยก ไม่บล็อค
    @Async
    @Scheduled(fixedRate = 5000)
    fun asyncTask() {
        println("[async] Running in thread: ${Thread.currentThread().name}")
        Thread.sleep(10000)  // 10 วินาที - ไม่บล็อค scheduler thread!
        println("[async] Completed!")
    }

    // หลาย tasks รันพร้อมกัน
    @Async("emailTaskExecutor")
    @Scheduled(cron = "0 0 * * * *")
    fun hourlyEmailTask() {
        println("Sending hourly emails...")
        // ใช้ custom executor pool
    }
}
```

---

## 🔒 5. Distributed Lock สำหรับ Scheduled Tasks

ปัญหาเมื่อ deploy หลาย instances: Task จะรันพร้อมกันทุก instance!

```kotlin
// build.gradle.kts - เพิ่ม dependency
// implementation("net.javacrumbs.shedlock:shedlock-spring:5.10.0")
// implementation("net.javacrumbs.shedlock:shedlock-provider-jdbc-template:5.10.0")
```

```sql
-- Create ShedLock table
CREATE TABLE shedlock (
    name       VARCHAR(64)  NOT NULL,
    lock_until TIMESTAMP    NOT NULL,
    locked_at  TIMESTAMP    NOT NULL,
    locked_by  VARCHAR(255) NOT NULL,
    PRIMARY KEY (name)
);
```

```kotlin
// src/main/kotlin/com/example/config/ShedLockConfig.kt
package com.example.config

import net.javacrumbs.shedlock.core.LockProvider
import net.javacrumbs.shedlock.provider.jdbctemplate.JdbcTemplateLockProvider
import net.javacrumbs.shedlock.spring.annotation.EnableSchedulerLock
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.jdbc.core.JdbcTemplate

@Configuration
@EnableSchedulerLock(defaultLockAtMostFor = "10m")
class ShedLockConfig {

    @Bean
    fun lockProvider(jdbcTemplate: JdbcTemplate): LockProvider {
        return JdbcTemplateLockProvider(
            JdbcTemplateLockProvider.Configuration.builder()
                .withJdbcTemplate(jdbcTemplate)
                .usingDbTime()  // ใช้ DB time (ไม่ใช่ app server time)
                .build()
        )
    }
}
```

```kotlin
// src/main/kotlin/com/example/scheduler/DistributedScheduler.kt
package com.example.scheduler

import net.javacrumbs.shedlock.spring.annotation.SchedulerLock
import org.springframework.scheduling.annotation.Scheduled
import org.springframework.stereotype.Component

@Component
class DistributedScheduler {

    // lock เพื่อให้มีแค่ 1 instance ที่รัน task นี้
    @Scheduled(cron = "0 0 2 * * *")  // ทุกวัน 2:00
    @SchedulerLock(
        name = "dailyReport",
        lockAtMostFor = "30m",  // lock สูงสุด 30 นาที (ป้องกัน dead lock)
        lockAtLeastFor = "5m"   // lock อย่างน้อย 5 นาที
    )
    fun dailyReportTask() {
        println("Running daily report (only on 1 instance)...")
        // รัน daily report
    }

    @Scheduled(cron = "0 0 3 * * *")
    @SchedulerLock(name = "cleanupTask", lockAtMostFor = "1h")
    fun cleanupTask() {
        println("Running cleanup task...")
    }
}
```

---

## 📊 6. ตัวอย่าง: Daily Report Task

```kotlin
// src/main/kotlin/com/example/scheduler/DailyReportScheduler.kt
package com.example.scheduler

import com.example.service.ReportService
import com.example.service.EmailService
import net.javacrumbs.shedlock.spring.annotation.SchedulerLock
import org.springframework.scheduling.annotation.Scheduled
import org.springframework.stereotype.Component
import java.time.LocalDate
import java.time.LocalDateTime

@Component
class DailyReportScheduler(
    private val reportService: ReportService,
    private val emailService: EmailService
) {

    // Daily sales report - ทุกวันเวลา 6:00 AM
    @Scheduled(cron = "0 0 6 * * *", zone = "Asia/Bangkok")
    @SchedulerLock(name = "dailySalesReport", lockAtMostFor = "30m")
    fun generateDailySalesReport() {
        val yesterday = LocalDate.now().minusDays(1)
        println("[DailyReport] Generating sales report for $yesterday")

        try {
            val report = reportService.generateSalesReport(yesterday)
            emailService.sendSimpleEmail(
                to = "admin@example.com",
                subject = "Daily Sales Report - $yesterday",
                body = buildReportText(report)
            )
            println("[DailyReport] Report sent successfully")
        } catch (e: Exception) {
            println("[DailyReport] Failed: ${e.message}")
        }
    }

    // Weekly summary - ทุกวันจันทร์เวลา 8:00 AM
    @Scheduled(cron = "0 0 8 * * MON", zone = "Asia/Bangkok")
    @SchedulerLock(name = "weeklySummary", lockAtMostFor = "1h")
    fun generateWeeklySummary() {
        val weekStart = LocalDate.now().minusWeeks(1)
        val weekEnd = LocalDate.now().minusDays(1)
        println("[WeeklyReport] Generating weekly summary from $weekStart to $weekEnd")

        val summary = reportService.generateWeeklyReport(weekStart, weekEnd)
        // ส่งรายงาน...
    }

    private fun buildReportText(report: SalesReport): String {
        return """
            Daily Sales Report - ${report.date}
            =====================================
            Total Orders: ${report.totalOrders}
            Total Revenue: ฿${report.totalRevenue}
            New Customers: ${report.newCustomers}
            Top Products:
            ${report.topProducts.joinToString("\n") { "  - ${it.name}: ${it.quantity} units" }}
        """.trimIndent()
    }
}
```

---

## 🧹 7. ตัวอย่าง: Cleanup Jobs

```kotlin
// src/main/kotlin/com/example/scheduler/CleanupScheduler.kt
package com.example.scheduler

import com.example.repository.*
import net.javacrumbs.shedlock.spring.annotation.SchedulerLock
import org.springframework.scheduling.annotation.Scheduled
import org.springframework.stereotype.Component
import org.springframework.transaction.annotation.Transactional
import java.time.LocalDateTime

@Component
class CleanupScheduler(
    private val expiredTokenRepository: ExpiredTokenRepository,
    private val tempFileRepository: TempFileRepository,
    private val auditLogRepository: AuditLogRepository,
    private val sessionRepository: SessionRepository
) {

    // ลบ expired tokens ทุก 1 ชั่วโมง
    @Scheduled(fixedRate = 3600000)
    @SchedulerLock(name = "cleanExpiredTokens", lockAtMostFor = "5m")
    @Transactional
    fun cleanExpiredTokens() {
        val before = LocalDateTime.now()
        val deleted = expiredTokenRepository.deleteByExpiresAtBefore(before)
        println("[Cleanup] Deleted $deleted expired tokens")
    }

    // ลบ temp files ทุกคืน 1:00 AM
    @Scheduled(cron = "0 0 1 * * *")
    @SchedulerLock(name = "cleanTempFiles", lockAtMostFor = "30m")
    @Transactional
    fun cleanTempFiles() {
        val cutoff = LocalDateTime.now().minusDays(1)
        val files = tempFileRepository.findByCreatedAtBefore(cutoff)

        files.forEach { file ->
            try {
                java.io.File(file.filePath).delete()
                tempFileRepository.delete(file)
            } catch (e: Exception) {
                println("[Cleanup] Failed to delete temp file ${file.filePath}: ${e.message}")
            }
        }
        println("[Cleanup] Cleaned ${files.size} temp files")
    }

    // Archive old audit logs ทุกอาทิตย์
    @Scheduled(cron = "0 0 3 * * SUN")
    @SchedulerLock(name = "archiveAuditLogs", lockAtMostFor = "2h")
    @Transactional
    fun archiveOldAuditLogs() {
        val cutoff = LocalDateTime.now().minusMonths(3)
        val count = auditLogRepository.archiveOlderThan(cutoff)
        println("[Cleanup] Archived $count audit log entries")
    }

    // ลบ expired sessions ทุก 15 นาที
    @Scheduled(fixedDelay = 900000)
    @Transactional
    fun cleanExpiredSessions() {
        val deleted = sessionRepository.deleteByExpiresAtBefore(LocalDateTime.now())
        if (deleted > 0) {
            println("[Cleanup] Deleted $deleted expired sessions")
        }
    }

    // Health check - ทุก 5 นาที
    @Scheduled(fixedRate = 300000)
    fun healthCheck() {
        // ตรวจสอบ system health
        val freeMemory = Runtime.getRuntime().freeMemory() / 1024 / 1024
        val totalMemory = Runtime.getRuntime().totalMemory() / 1024 / 1024
        if (freeMemory < 100) {
            println("[Health] WARNING: Low memory! Free: ${freeMemory}MB / Total: ${totalMemory}MB")
        }
    }
}
```

---

## 🔄 8. Dynamic Scheduling

```kotlin
// src/main/kotlin/com/example/scheduler/DynamicScheduler.kt
package com.example.scheduler

import org.springframework.scheduling.TaskScheduler
import org.springframework.scheduling.annotation.EnableScheduling
import org.springframework.scheduling.support.CronTrigger
import org.springframework.stereotype.Service
import java.util.concurrent.ScheduledFuture

@Service
class DynamicScheduler(
    private val taskScheduler: TaskScheduler
) {

    private val scheduledTasks = mutableMapOf<String, ScheduledFuture<*>>()

    // เพิ่ม task แบบ dynamic
    fun scheduleTask(taskId: String, cronExpression: String, task: Runnable): Boolean {
        // ยกเลิก task เก่าถ้ามี
        cancelTask(taskId)

        return try {
            val future = taskScheduler.schedule(task, CronTrigger(cronExpression))
            if (future != null) {
                scheduledTasks[taskId] = future
                println("Scheduled task '$taskId' with cron: $cronExpression")
                true
            } else false
        } catch (e: Exception) {
            println("Failed to schedule task '$taskId': ${e.message}")
            false
        }
    }

    // ยกเลิก task
    fun cancelTask(taskId: String): Boolean {
        val future = scheduledTasks.remove(taskId)
        return future?.cancel(false) ?: false
    }

    // ดูรายการ tasks ที่กำลังรัน
    fun getActiveTasks(): List<String> {
        return scheduledTasks.entries
            .filter { !it.value.isCancelled && !it.value.isDone }
            .map { it.key }
    }
}
```

```kotlin
// Controller สำหรับ manage scheduled tasks
// src/main/kotlin/com/example/controller/SchedulerController.kt
package com.example.controller

import com.example.scheduler.DynamicScheduler
import org.springframework.http.ResponseEntity
import org.springframework.web.bind.annotation.*

@RestController
@RequestMapping("/api/admin/scheduler")
class SchedulerController(private val dynamicScheduler: DynamicScheduler) {

    @PostMapping("/tasks")
    fun createTask(
        @RequestParam taskId: String,
        @RequestParam cronExpression: String,
        @RequestParam taskType: String
    ): ResponseEntity<Map<String, Any>> {
        val task = createTaskByType(taskType)
        val success = dynamicScheduler.scheduleTask(taskId, cronExpression, task)
        return ResponseEntity.ok(mapOf(
            "taskId" to taskId,
            "success" to success,
            "cron" to cronExpression
        ))
    }

    @DeleteMapping("/tasks/{taskId}")
    fun cancelTask(@PathVariable taskId: String): ResponseEntity<Map<String, Any>> {
        val cancelled = dynamicScheduler.cancelTask(taskId)
        return ResponseEntity.ok(mapOf("taskId" to taskId, "cancelled" to cancelled))
    }

    @GetMapping("/tasks")
    fun getActiveTasks(): ResponseEntity<Map<String, Any>> {
        return ResponseEntity.ok(mapOf("activeTasks" to dynamicScheduler.getActiveTasks()))
    }

    private fun createTaskByType(taskType: String): Runnable {
        return Runnable {
            println("Running task type: $taskType at ${java.time.LocalDateTime.now()}")
        }
    }
}
```

---

## 🧪 9. Testing Scheduled Tasks

```kotlin
// src/test/kotlin/com/example/scheduler/CleanupSchedulerTest.kt
package com.example.scheduler

import com.example.repository.ExpiredTokenRepository
import io.mockk.*
import org.junit.jupiter.api.Test
import java.time.LocalDateTime

class CleanupSchedulerTest {

    private val expiredTokenRepository = mockk<ExpiredTokenRepository>()
    private val tempFileRepository = mockk<TempFileRepository>()
    private val auditLogRepository = mockk<AuditLogRepository>()
    private val sessionRepository = mockk<SessionRepository>()

    private val cleanupScheduler = CleanupScheduler(
        expiredTokenRepository,
        tempFileRepository,
        auditLogRepository,
        sessionRepository
    )

    @Test
    fun `should clean expired tokens`() {
        every { expiredTokenRepository.deleteByExpiresAtBefore(any()) } returns 5

        cleanupScheduler.cleanExpiredTokens()

        verify { expiredTokenRepository.deleteByExpiresAtBefore(any()) }
    }
}
```

---

## 📋 สรุป

### @Scheduled Attributes

| Attribute | ประเภท | คำอธิบาย | ตัวอย่าง |
|-----------|--------|---------|---------|
| `fixedRate` | Long (ms) | ทุกๆ X ms (นับจากต้น) | `fixedRate = 5000` |
| `fixedDelay` | Long (ms) | ทุกๆ X ms หลังจบ | `fixedDelay = 5000` |
| `cron` | String | Cron expression | `cron = "0 0 * * * *"` |
| `initialDelay` | Long (ms) | หน่วงก่อนครั้งแรก | `initialDelay = 10000` |
| `zone` | String | Timezone | `zone = "Asia/Bangkok"` |

### Cron Expression

| Field | Values |
|-------|--------|
| Second | 0-59 |
| Minute | 0-59 |
| Hour | 0-23 |
| Day of Month | 1-31 |
| Month | 1-12 หรือ JAN-DEC |
| Day of Week | 0-7 หรือ MON-SUN |

### Special Characters

| Character | ความหมาย |
|-----------|---------|
| `*` | ทุกค่า |
| `?` | ไม่ระบุ (สำหรับ day-of-month/day-of-week) |
| `-` | ช่วง เช่น `MON-FRI` |
| `,` | หลายค่า เช่น `MON,WED,FRI` |
| `/` | ทุกๆ เช่น `*/5` = ทุก 5 |
| `L` | วันสุดท้าย |

---

*Part 37/100+ | Kotlin & Spring Boot Complete Course*
