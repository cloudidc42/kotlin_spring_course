# Part 39: Audit Logging
## Spring Data JPA Auditing และ Custom Audit Trail

---

## 🎯 เป้าหมายของ Part นี้

- Spring Data JPA Auditing
- @CreatedBy, @LastModifiedBy
- @CreatedDate, @LastModifiedDate
- AuditorAware สำหรับ tracking user
- Custom audit trail
- ตัวอย่าง: Entity audit history

---

## 📋 1. Audit Logging คืออะไร

**Audit Logging** คือการบันทึกว่าใคร ทำอะไร เมื่อไหร่ กับ data ใด

### ประโยชน์
- Compliance (GDPR, HIPAA, SOX)
- Security monitoring
- Debugging และ troubleshooting
- Business intelligence

### ข้อมูลที่ควร audit
- ใคร (WHO): user ที่ทำ action
- อะไร (WHAT): action ที่ทำ (CREATE/UPDATE/DELETE)
- เมื่อไหร่ (WHEN): timestamp
- Data เก่าและใหม่ (BEFORE/AFTER)

---

## ⚙️ 2. Spring Data JPA Auditing

### 2.1 Enable Auditing

```kotlin
// src/main/kotlin/com/example/config/JpaAuditingConfig.kt
package com.example.config

import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.data.domain.AuditorAware
import org.springframework.data.jpa.repository.config.EnableJpaAuditing
import org.springframework.security.core.context.SecurityContextHolder
import java.util.Optional

@Configuration
@EnableJpaAuditing(auditorAwareRef = "auditorProvider")
class JpaAuditingConfig {

    // AuditorAware: บอก Spring ว่า current user คือใคร
    @Bean
    fun auditorProvider(): AuditorAware<String> {
        return AuditorAware {
            val authentication = SecurityContextHolder.getContext().authentication
            if (authentication != null && authentication.isAuthenticated
                && authentication.name != "anonymousUser") {
                Optional.of(authentication.name)
            } else {
                Optional.of("system")  // default เมื่อไม่มี user
            }
        }
    }
}
```

### 2.2 Auditable Base Entity

```kotlin
// src/main/kotlin/com/example/entity/AuditableEntity.kt
package com.example.entity

import jakarta.persistence.Column
import jakarta.persistence.EntityListeners
import jakarta.persistence.MappedSuperclass
import org.springframework.data.annotation.CreatedBy
import org.springframework.data.annotation.CreatedDate
import org.springframework.data.annotation.LastModifiedBy
import org.springframework.data.annotation.LastModifiedDate
import org.springframework.data.jpa.domain.support.AuditingEntityListener
import java.time.LocalDateTime

@MappedSuperclass  // fields จะถูก inherit โดย subclasses
@EntityListeners(AuditingEntityListener::class)  // เปิดใช้งาน JPA auditing
abstract class AuditableEntity {

    @CreatedDate
    @Column(name = "created_at", nullable = false, updatable = false)
    var createdAt: LocalDateTime? = null

    @LastModifiedDate
    @Column(name = "updated_at")
    var updatedAt: LocalDateTime? = null

    @CreatedBy
    @Column(name = "created_by", updatable = false, length = 100)
    var createdBy: String? = null

    @LastModifiedBy
    @Column(name = "updated_by", length = 100)
    var updatedBy: String? = null
}
```

### 2.3 Entity ที่ใช้ Auditing

```kotlin
// src/main/kotlin/com/example/entity/Product.kt
package com.example.entity

import jakarta.persistence.*
import java.math.BigDecimal

@Entity
@Table(name = "products")
data class Product(
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long = 0,

    @Column(nullable = false)
    var name: String,

    @Column(nullable = false)
    var price: BigDecimal,

    var description: String? = null,

    @Column(name = "category_id")
    var categoryId: Long? = null,

    @Column(name = "is_active")
    var isActive: Boolean = true
) : AuditableEntity()  // inherit audit fields
```

```sql
-- DDL สำหรับ auditable table
CREATE TABLE products (
    id          BIGSERIAL PRIMARY KEY,
    name        VARCHAR(255) NOT NULL,
    price       DECIMAL(10, 2) NOT NULL,
    description TEXT,
    category_id BIGINT,
    is_active   BOOLEAN DEFAULT TRUE,
    
    -- Audit fields
    created_at  TIMESTAMP NOT NULL,
    updated_at  TIMESTAMP,
    created_by  VARCHAR(100),
    updated_by  VARCHAR(100)
);
```

---

## 📚 3. Custom Audit Trail (Change History)

```kotlin
// src/main/kotlin/com/example/entity/AuditLog.kt
package com.example.entity

import jakarta.persistence.*
import java.time.LocalDateTime

@Entity
@Table(name = "audit_logs", indexes = [
    Index(name = "idx_audit_entity", columnList = "entity_type, entity_id"),
    Index(name = "idx_audit_user", columnList = "performed_by"),
    Index(name = "idx_audit_timestamp", columnList = "performed_at")
])
data class AuditLog(
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long = 0,

    @Column(name = "entity_type", nullable = false, length = 100)
    val entityType: String,

    @Column(name = "entity_id", nullable = false)
    val entityId: Long,

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    val action: AuditAction,

    @Column(name = "performed_by", length = 100)
    val performedBy: String? = null,

    @Column(name = "performed_at", nullable = false)
    val performedAt: LocalDateTime = LocalDateTime.now(),

    @Column(name = "ip_address", length = 50)
    val ipAddress: String? = null,

    @Column(name = "old_values", columnDefinition = "TEXT")
    val oldValues: String? = null,  // JSON

    @Column(name = "new_values", columnDefinition = "TEXT")
    val newValues: String? = null,  // JSON

    @Column(name = "changed_fields", columnDefinition = "TEXT")
    val changedFields: String? = null,  // comma-separated

    @Column(columnDefinition = "TEXT")
    val details: String? = null
)

enum class AuditAction {
    CREATE, UPDATE, DELETE, VIEW, LOGIN, LOGOUT, EXPORT
}
```

---

## 🔍 4. JPA Entity Listener

```kotlin
// src/main/kotlin/com/example/audit/AuditEntityListener.kt
package com.example.audit

import com.example.entity.AuditLog
import com.example.entity.AuditAction
import com.fasterxml.jackson.databind.ObjectMapper
import jakarta.persistence.*
import org.springframework.beans.factory.annotation.Autowired
import org.springframework.security.core.context.SecurityContextHolder
import org.springframework.stereotype.Component
import java.time.LocalDateTime

@Component
class AuditEntityListener {

    @Autowired
    private lateinit var auditLogRepository: com.example.repository.AuditLogRepository

    @Autowired
    private lateinit var objectMapper: ObjectMapper

    // ก่อน persist - เก็บ state เดิม
    private val originalStateMap = ThreadLocal<Map<String, Any?>>()

    @PrePersist
    fun prePersist(entity: Any) {
        // เตรียม state ก่อน create
    }

    @PostPersist
    fun postPersist(entity: Any) {
        saveAuditLog(entity, AuditAction.CREATE, null)
    }

    @PreUpdate
    fun preUpdate(entity: Any) {
        // เก็บ old values ก่อน update
        // (ต้องใช้ EntityManager เพื่อ load original)
    }

    @PostUpdate
    fun postUpdate(entity: Any) {
        saveAuditLog(entity, AuditAction.UPDATE, null)
    }

    @PreRemove
    fun preRemove(entity: Any) {
        // เก็บ values ก่อนลบ
    }

    @PostRemove
    fun postRemove(entity: Any) {
        saveAuditLog(entity, AuditAction.DELETE, null)
    }

    private fun saveAuditLog(entity: Any, action: AuditAction, oldValues: Map<String, Any?>?) {
        val currentUser = getCurrentUser()
        val entityInfo = extractEntityInfo(entity)

        val auditLog = AuditLog(
            entityType = entityInfo.first,
            entityId = entityInfo.second,
            action = action,
            performedBy = currentUser,
            performedAt = LocalDateTime.now(),
            newValues = if (action != AuditAction.DELETE) {
                objectMapper.writeValueAsString(entity)
            } else null,
            oldValues = if (oldValues != null) {
                objectMapper.writeValueAsString(oldValues)
            } else null
        )

        auditLogRepository.save(auditLog)
    }

    private fun getCurrentUser(): String? {
        return try {
            SecurityContextHolder.getContext().authentication?.name
        } catch (e: Exception) {
            "system"
        }
    }

    private fun extractEntityInfo(entity: Any): Pair<String, Long> {
        val entityClass = entity::class.java
        val entityName = entityClass.simpleName ?: "Unknown"

        // ดึง ID ด้วย reflection
        val idField = entityClass.declaredFields.firstOrNull { field ->
            field.isAnnotationPresent(Id::class.java)
        }
        idField?.isAccessible = true
        val id = (idField?.get(entity) as? Long) ?: 0L

        return Pair(entityName, id)
    }
}
```

---

## 📊 5. Manual Audit Service

```kotlin
// src/main/kotlin/com/example/service/AuditService.kt
package com.example.service

import com.example.entity.AuditAction
import com.example.entity.AuditLog
import com.example.repository.AuditLogRepository
import com.fasterxml.jackson.databind.ObjectMapper
import com.fasterxml.jackson.module.kotlin.readValue
import org.springframework.data.domain.Page
import org.springframework.data.domain.Pageable
import org.springframework.security.core.context.SecurityContextHolder
import org.springframework.stereotype.Service
import org.springframework.web.context.request.RequestContextHolder
import org.springframework.web.context.request.ServletRequestAttributes
import java.time.LocalDateTime

@Service
class AuditService(
    private val auditLogRepository: AuditLogRepository,
    private val objectMapper: ObjectMapper
) {

    // บันทึก audit log แบบ manual
    fun log(
        entityType: String,
        entityId: Long,
        action: AuditAction,
        oldValues: Any? = null,
        newValues: Any? = null,
        details: String? = null
    ) {
        val currentUser = getCurrentUser()
        val ipAddress = getClientIpAddress()

        val changedFields = if (oldValues != null && newValues != null) {
            findChangedFields(oldValues, newValues)
        } else null

        val auditLog = AuditLog(
            entityType = entityType,
            entityId = entityId,
            action = action,
            performedBy = currentUser,
            performedAt = LocalDateTime.now(),
            ipAddress = ipAddress,
            oldValues = oldValues?.let { objectMapper.writeValueAsString(it) },
            newValues = newValues?.let { objectMapper.writeValueAsString(it) },
            changedFields = changedFields,
            details = details
        )

        auditLogRepository.save(auditLog)
    }

    // ดึง audit history ของ entity
    fun getAuditHistory(
        entityType: String,
        entityId: Long,
        pageable: Pageable = Pageable.unpaged()
    ): Page<AuditLog> {
        return auditLogRepository.findByEntityTypeAndEntityIdOrderByPerformedAtDesc(
            entityType, entityId, pageable
        )
    }

    // ดึง audit logs ของ user
    fun getUserAuditLogs(
        username: String,
        from: LocalDateTime? = null,
        to: LocalDateTime? = null,
        pageable: Pageable = Pageable.unpaged()
    ): Page<AuditLog> {
        return if (from != null && to != null) {
            auditLogRepository.findByPerformedByAndPerformedAtBetween(
                username, from, to, pageable
            )
        } else {
            auditLogRepository.findByPerformedBy(username, pageable)
        }
    }

    // ค้นหา audit logs
    fun searchAuditLogs(
        entityType: String? = null,
        action: AuditAction? = null,
        performedBy: String? = null,
        from: LocalDateTime? = null,
        to: LocalDateTime? = null,
        pageable: Pageable
    ): Page<AuditLog> {
        return auditLogRepository.search(entityType, action, performedBy, from, to, pageable)
    }

    private fun findChangedFields(oldValues: Any, newValues: Any): String {
        val oldMap: Map<String, Any?> = objectMapper.readValue(
            objectMapper.writeValueAsString(oldValues)
        )
        val newMap: Map<String, Any?> = objectMapper.readValue(
            objectMapper.writeValueAsString(newValues)
        )

        return oldMap.keys
            .filter { key -> oldMap[key] != newMap[key] }
            .joinToString(",")
    }

    private fun getCurrentUser(): String {
        return try {
            SecurityContextHolder.getContext().authentication?.name ?: "anonymous"
        } catch (e: Exception) {
            "system"
        }
    }

    private fun getClientIpAddress(): String? {
        return try {
            val request = (RequestContextHolder.getRequestAttributes()
                as? ServletRequestAttributes)?.request

            request?.let {
                it.getHeader("X-Forwarded-For")?.split(",")?.firstOrNull()?.trim()
                    ?: it.getHeader("X-Real-IP")
                    ?: it.remoteAddr
            }
        } catch (e: Exception) {
            null
        }
    }
}
```

---

## 🗄️ 6. Audit Repository

```kotlin
// src/main/kotlin/com/example/repository/AuditLogRepository.kt
package com.example.repository

import com.example.entity.AuditAction
import com.example.entity.AuditLog
import org.springframework.data.domain.Page
import org.springframework.data.domain.Pageable
import org.springframework.data.jpa.repository.JpaRepository
import org.springframework.data.jpa.repository.Query
import org.springframework.data.repository.query.Param
import java.time.LocalDateTime

interface AuditLogRepository : JpaRepository<AuditLog, Long> {

    fun findByEntityTypeAndEntityIdOrderByPerformedAtDesc(
        entityType: String,
        entityId: Long,
        pageable: Pageable
    ): Page<AuditLog>

    fun findByPerformedBy(performedBy: String, pageable: Pageable): Page<AuditLog>

    fun findByPerformedByAndPerformedAtBetween(
        performedBy: String,
        from: LocalDateTime,
        to: LocalDateTime,
        pageable: Pageable
    ): Page<AuditLog>

    fun findByEntityTypeAndAction(
        entityType: String,
        action: AuditAction,
        pageable: Pageable
    ): Page<AuditLog>

    // Complex search query
    @Query("""
        SELECT a FROM AuditLog a 
        WHERE (:entityType IS NULL OR a.entityType = :entityType)
          AND (:action IS NULL OR a.action = :action)
          AND (:performedBy IS NULL OR a.performedBy = :performedBy)
          AND (:from IS NULL OR a.performedAt >= :from)
          AND (:to IS NULL OR a.performedAt <= :to)
        ORDER BY a.performedAt DESC
    """)
    fun search(
        @Param("entityType") entityType: String?,
        @Param("action") action: AuditAction?,
        @Param("performedBy") performedBy: String?,
        @Param("from") from: LocalDateTime?,
        @Param("to") to: LocalDateTime?,
        pageable: Pageable
    ): Page<AuditLog>

    // Archive old logs
    @Query("SELECT COUNT(a) FROM AuditLog a WHERE a.performedAt < :before")
    fun countOlderThan(@Param("before") before: LocalDateTime): Long

    @Query("DELETE FROM AuditLog a WHERE a.performedAt < :before")
    fun deleteOlderThan(@Param("before") before: LocalDateTime): Int
}
```

---

## 🔧 7. Audit Aspect (AOP)

```kotlin
// src/main/kotlin/com/example/aspect/AuditAspect.kt
package com.example.aspect

import com.example.entity.AuditAction
import com.example.service.AuditService
import org.aspectj.lang.ProceedingJoinPoint
import org.aspectj.lang.annotation.Around
import org.aspectj.lang.annotation.Aspect
import org.springframework.stereotype.Component

// Custom annotation สำหรับ audit
@Target(AnnotationTarget.FUNCTION)
@Retention(AnnotationRetention.RUNTIME)
annotation class Audited(
    val entityType: String,
    val action: AuditAction
)

@Aspect
@Component
class AuditAspect(private val auditService: AuditService) {

    @Around("@annotation(audited)")
    fun auditMethod(joinPoint: ProceedingJoinPoint, audited: Audited): Any? {
        val args = joinPoint.args
        val entityId = args.firstOrNull { it is Long } as? Long ?: 0L

        val oldValues = if (audited.action == AuditAction.UPDATE) {
            // ดึง old values ก่อน update
            null // TODO: load from DB
        } else null

        return try {
            val result = joinPoint.proceed()

            auditService.log(
                entityType = audited.entityType,
                entityId = entityId,
                action = audited.action,
                oldValues = oldValues,
                newValues = if (result != null) result else null,
                details = "Method: ${joinPoint.signature.name}"
            )

            result
        } catch (e: Exception) {
            auditService.log(
                entityType = audited.entityType,
                entityId = entityId,
                action = audited.action,
                details = "FAILED: ${e.message}"
            )
            throw e
        }
    }
}
```

```kotlin
// ใช้งาน @Audited annotation
@Service
class ProductService(
    private val productRepository: ProductRepository,
    private val auditService: AuditService
) {

    @Audited(entityType = "PRODUCT", action = AuditAction.CREATE)
    fun createProduct(dto: CreateProductDto): ProductDto {
        val product = Product(name = dto.name, price = dto.price)
        return productRepository.save(product).toDto()
    }

    // Manual audit เมื่อต้องการ old/new values
    fun updateProduct(id: Long, dto: UpdateProductDto): ProductDto {
        val oldProduct = productRepository.findById(id)
            .orElseThrow { NotFoundException("Product not found") }
        val oldValues = oldProduct.toDto()

        oldProduct.apply {
            dto.name?.let { name = it }
            dto.price?.let { price = it }
        }
        val updated = productRepository.save(oldProduct).toDto()

        auditService.log(
            entityType = "PRODUCT",
            entityId = id,
            action = AuditAction.UPDATE,
            oldValues = oldValues,
            newValues = updated
        )

        return updated
    }
}
```

---

## 🌐 8. Audit Controller

```kotlin
// src/main/kotlin/com/example/controller/AuditController.kt
package com.example.controller

import com.example.entity.AuditAction
import com.example.entity.AuditLog
import com.example.service.AuditService
import org.springframework.data.domain.Page
import org.springframework.data.domain.PageRequest
import org.springframework.data.domain.Sort
import org.springframework.format.annotation.DateTimeFormat
import org.springframework.http.ResponseEntity
import org.springframework.security.access.prepost.PreAuthorize
import org.springframework.web.bind.annotation.*
import java.time.LocalDateTime

@RestController
@RequestMapping("/api/admin/audit")
@PreAuthorize("hasRole('ADMIN')")  // เฉพาะ admin เท่านั้น
class AuditController(private val auditService: AuditService) {

    // ดู audit history ของ entity
    @GetMapping("/{entityType}/{entityId}/history")
    fun getEntityHistory(
        @PathVariable entityType: String,
        @PathVariable entityId: Long,
        @RequestParam(defaultValue = "0") page: Int,
        @RequestParam(defaultValue = "20") size: Int
    ): ResponseEntity<Page<AuditLog>> {
        val pageable = PageRequest.of(page, size)
        return ResponseEntity.ok(auditService.getAuditHistory(entityType, entityId, pageable))
    }

    // ดู audit logs ของ user
    @GetMapping("/users/{username}")
    fun getUserAuditLogs(
        @PathVariable username: String,
        @RequestParam(required = false) @DateTimeFormat(iso = DateTimeFormat.ISO.DATE_TIME) from: LocalDateTime?,
        @RequestParam(required = false) @DateTimeFormat(iso = DateTimeFormat.ISO.DATE_TIME) to: LocalDateTime?,
        @RequestParam(defaultValue = "0") page: Int,
        @RequestParam(defaultValue = "20") size: Int
    ): ResponseEntity<Page<AuditLog>> {
        val pageable = PageRequest.of(page, size, Sort.by("performedAt").descending())
        return ResponseEntity.ok(auditService.getUserAuditLogs(username, from, to, pageable))
    }

    // Search audit logs
    @GetMapping("/search")
    fun searchAuditLogs(
        @RequestParam(required = false) entityType: String?,
        @RequestParam(required = false) action: AuditAction?,
        @RequestParam(required = false) performedBy: String?,
        @RequestParam(required = false) @DateTimeFormat(iso = DateTimeFormat.ISO.DATE_TIME) from: LocalDateTime?,
        @RequestParam(required = false) @DateTimeFormat(iso = DateTimeFormat.ISO.DATE_TIME) to: LocalDateTime?,
        @RequestParam(defaultValue = "0") page: Int,
        @RequestParam(defaultValue = "20") size: Int
    ): ResponseEntity<Page<AuditLog>> {
        val pageable = PageRequest.of(page, size, Sort.by("performedAt").descending())
        return ResponseEntity.ok(
            auditService.searchAuditLogs(entityType, action, performedBy, from, to, pageable)
        )
    }
}
```

---

## 🧪 9. Testing Audit

```kotlin
// src/test/kotlin/com/example/service/AuditServiceTest.kt
package com.example.service

import com.example.entity.AuditAction
import com.example.entity.AuditLog
import com.example.repository.AuditLogRepository
import io.mockk.every
import io.mockk.mockk
import io.mockk.slot
import io.mockk.verify
import org.junit.jupiter.api.Test
import kotlin.test.assertEquals

class AuditServiceTest {

    private val auditLogRepository = mockk<AuditLogRepository>()
    private val objectMapper = com.fasterxml.jackson.databind.ObjectMapper()
        .apply { registerModule(com.fasterxml.jackson.module.kotlin.kotlinModule()) }

    private val auditService = AuditService(auditLogRepository, objectMapper)

    @Test
    fun `should save audit log on create`() {
        val logSlot = slot<AuditLog>()
        every { auditLogRepository.save(capture(logSlot)) } returns mockk()

        auditService.log(
            entityType = "PRODUCT",
            entityId = 1L,
            action = AuditAction.CREATE,
            details = "Test create"
        )

        val savedLog = logSlot.captured
        assertEquals("PRODUCT", savedLog.entityType)
        assertEquals(1L, savedLog.entityId)
        assertEquals(AuditAction.CREATE, savedLog.action)

        verify(exactly = 1) { auditLogRepository.save(any()) }
    }
}
```

---

## 📋 สรุป

### Spring Data JPA Auditing Annotations

| Annotation | ประเภท field | คำอธิบาย |
|-----------|-------------|---------|
| `@CreatedDate` | `LocalDateTime` | วันที่สร้าง |
| `@LastModifiedDate` | `LocalDateTime` | วันที่แก้ไขล่าสุด |
| `@CreatedBy` | `String` | ผู้สร้าง |
| `@LastModifiedBy` | `String` | ผู้แก้ไขล่าสุด |

### AuditAction Types

| Action | เมื่อ |
|--------|------|
| `CREATE` | สร้าง entity ใหม่ |
| `UPDATE` | อัพเดท entity |
| `DELETE` | ลบ entity |
| `VIEW` | ดู sensitive data |
| `LOGIN` | User login |
| `LOGOUT` | User logout |
| `EXPORT` | Export data |

### Audit Log Storage Best Practices

| ข้อควรปฏิบัติ | เหตุผล |
|-------------|--------|
| แยก schema/DB | ป้องกัน audit log ถูก tamper |
| Index บน entity_type, entity_id | เร็วเมื่อ query history |
| Archive เก่ากว่า 1 ปี | ประหยัด storage |
| Immutable (ไม่มี UPDATE/DELETE) | ความน่าเชื่อถือ |

---

*Part 39/100+ | Kotlin & Spring Boot Complete Course*
