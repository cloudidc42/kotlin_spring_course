# Part 93: Real World Project 3 - Task Management SaaS
## สร้าง Multi-tenant Task Management Platform

---

## 🎯 เป้าหมายของ Part นี้

- Multi-tenant architecture
- Organization, Teams, Members
- Task management กับ assignments
- Real-time notifications
- Webhooks integration

---

## 📋 1. Requirements

```
Features:
- Multi-tenant: แต่ละ organization มี data แยกกัน
- Organizations และ members
- Teams และ projects
- Tasks with subtasks, labels, priorities
- Assignments และ due dates
- Activity feed
- Notifications (email, in-app, webhook)
- Kanban board view
- Reporting and analytics
- API webhooks สำหรับ integrations
```

---

## 🏗️ 2. Multi-tenant Architecture

### Tenant Isolation Strategies

```
1. Separate Database per tenant
   ✅ Complete isolation
   ❌ Expensive, hard to manage

2. Separate Schema per tenant
   ✅ Good isolation
   ❌ Complex migrations

3. Shared Database, shared schema (ใช้ approach นี้)
   ✅ Simple, cost-effective
   ❌ Need careful data isolation
   Implementation: tenant_id column ใน every table
```

```kotlin
// config/TenantContext.kt
object TenantContext {
    private val currentTenant = ThreadLocal<Long>()

    fun setCurrentTenant(tenantId: Long) = currentTenant.set(tenantId)
    fun getCurrentTenant(): Long = currentTenant.get()
        ?: throw IllegalStateException("No tenant in context")
    fun clear() = currentTenant.remove()
}

// Filter ที่ extract tenant จาก JWT
@Component
class TenantFilter(private val jwtService: JwtService) : OncePerRequestFilter() {
    override fun doFilterInternal(
        request: HttpServletRequest,
        response: HttpServletResponse,
        filterChain: FilterChain
    ) {
        try {
            val token = request.getBearerToken()
            if (token != null) {
                val claims = jwtService.extractClaims(token)
                val tenantId = claims["tenantId"] as? Long
                if (tenantId != null) {
                    TenantContext.setCurrentTenant(tenantId)
                }
            }
            filterChain.doFilter(request, response)
        } finally {
            TenantContext.clear()
        }
    }
}
```

---

## 🗃️ 3. Entities

```kotlin
// entity/Organization.kt
@Entity
@Table(name = "organizations")
data class Organization(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long = 0,

    @Column(unique = true, nullable = false)
    val slug: String,  // unique identifier (e.g., "acme-corp")

    val name: String,
    val logoUrl: String? = null,

    @Enumerated(EnumType.STRING)
    val plan: PricingPlan = PricingPlan.FREE,

    val maxMembers: Int = 5,
    val maxProjects: Int = 3,

    val createdAt: Instant = Instant.now(),

    @OneToMany(mappedBy = "organization", cascade = [CascadeType.ALL])
    val members: List<OrganizationMember> = emptyList()
)

enum class PricingPlan { FREE, STARTER, PROFESSIONAL, ENTERPRISE }
```

```kotlin
// entity/Project.kt
@Entity
@Table(name = "projects")
data class Project(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long = 0,

    // Multi-tenant: every entity has organizationId
    @Column(nullable = false)
    val organizationId: Long,

    val name: String,
    val description: String? = null,
    val color: String = "#6366f1",

    @Enumerated(EnumType.STRING)
    val status: ProjectStatus = ProjectStatus.ACTIVE,

    val startDate: LocalDate? = null,
    val endDate: LocalDate? = null,

    @ManyToOne(fetch = FetchType.LAZY)
    val owner: User,

    val createdAt: Instant = Instant.now(),

    @OneToMany(mappedBy = "project", cascade = [CascadeType.ALL])
    val tasks: List<Task> = emptyList()
)

enum class ProjectStatus { ACTIVE, ARCHIVED, COMPLETED }
```

```kotlin
// entity/Task.kt
@Entity
@Table(name = "tasks")
data class Task(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long = 0,

    val organizationId: Long,  // for multi-tenant filtering

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "project_id", nullable = false)
    val project: Project,

    @Column(nullable = false)
    val title: String,

    @Column(length = 10000)
    val description: String? = null,

    @Enumerated(EnumType.STRING)
    val status: TaskStatus = TaskStatus.TODO,

    @Enumerated(EnumType.STRING)
    val priority: TaskPriority = TaskPriority.MEDIUM,

    @ManyToOne(fetch = FetchType.LAZY)
    val assignee: User? = null,

    @ManyToOne(fetch = FetchType.LAZY)
    val creator: User,

    val dueDate: LocalDate? = null,
    val estimatedHours: Double? = null,
    val actualHours: Double? = null,

    @ManyToOne(fetch = FetchType.LAZY)
    val parentTask: Task? = null,  // for subtasks

    @OneToMany(mappedBy = "parentTask")
    val subtasks: List<Task> = emptyList(),

    @ElementCollection
    @CollectionTable(name = "task_labels")
    val labels: Set<String> = emptySet(),

    val sortOrder: Int = 0,
    val createdAt: Instant = Instant.now(),
    val updatedAt: Instant = Instant.now()
)

enum class TaskStatus { TODO, IN_PROGRESS, IN_REVIEW, DONE, CANCELLED }
enum class TaskPriority { LOW, MEDIUM, HIGH, URGENT }
```

---

## 🔒 4. Multi-tenant Data Access

```kotlin
// repository/TaskRepository.kt
@Repository
interface TaskRepository : JpaRepository<Task, Long> {

    // Always filter by organizationId!
    fun findByOrganizationIdAndProjectId(
        organizationId: Long,
        projectId: Long,
        pageable: Pageable
    ): Page<Task>

    fun findByOrganizationIdAndAssigneeId(
        organizationId: Long,
        assigneeId: Long
    ): List<Task>

    @Query("""
        SELECT t FROM Task t 
        WHERE t.organizationId = :orgId
        AND t.dueDate < :date
        AND t.status NOT IN ('DONE', 'CANCELLED')
    """)
    fun findOverdueTasks(
        @Param("orgId") orgId: Long,
        @Param("date") date: LocalDate
    ): List<Task>
}

// service/TaskService.kt
@Service
@Transactional
class TaskService(
    private val taskRepository: TaskRepository,
    private val activityService: ActivityService,
    private val notificationService: NotificationService
) {
    fun getTasks(projectId: Long, filter: TaskFilter): Page<TaskDto> {
        // Always use current tenant's org ID
        val orgId = TenantContext.getCurrentTenant()

        val spec = buildSpecification(orgId, projectId, filter)
        val pageable = PageRequest.of(filter.page, filter.size, Sort.by("sortOrder"))

        return taskRepository.findAll(spec, pageable).map { it.toDto() }
    }

    fun createTask(projectId: Long, request: CreateTaskRequest, userId: Long): TaskDto {
        val orgId = TenantContext.getCurrentTenant()

        // Verify project belongs to org
        val project = projectRepository.findByIdAndOrganizationId(projectId, orgId)
            ?: throw NotFoundException("Project not found")

        val task = taskRepository.save(
            Task(
                organizationId = orgId,
                project = project,
                title = request.title,
                description = request.description,
                priority = request.priority ?: TaskPriority.MEDIUM,
                assignee = request.assigneeId?.let { userRepository.getReferenceById(it) },
                creator = userRepository.getReferenceById(userId),
                dueDate = request.dueDate,
                labels = request.labels ?: emptySet()
            )
        )

        // Log activity
        activityService.log(
            orgId = orgId,
            userId = userId,
            action = ActivityAction.TASK_CREATED,
            entityId = task.id,
            entityType = "Task"
        )

        // Notify assignee
        task.assignee?.let { assignee ->
            if (assignee.id != userId) {
                notificationService.notifyTaskAssigned(assignee, task, userId)
            }
        }

        return task.toDto()
    }

    fun moveTask(taskId: Long, newStatus: TaskStatus, userId: Long): TaskDto {
        val orgId = TenantContext.getCurrentTenant()
        val task = taskRepository.findByIdAndOrganizationId(taskId, orgId)
            ?: throw NotFoundException("Task not found")

        val oldStatus = task.status
        val updated = taskRepository.save(task.copy(status = newStatus))

        activityService.log(
            orgId = orgId,
            userId = userId,
            action = ActivityAction.TASK_STATUS_CHANGED,
            entityId = taskId,
            entityType = "Task",
            details = mapOf("from" to oldStatus, "to" to newStatus)
        )

        return updated.toDto()
    }
}
```

---

## 🔔 5. Notification System

```kotlin
// entity/Notification.kt
@Entity
data class Notification(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long = 0,

    val organizationId: Long,
    val recipientId: Long,
    val actorId: Long,

    @Enumerated(EnumType.STRING)
    val type: NotificationType,

    val entityType: String,
    val entityId: Long,

    val message: String,
    val isRead: Boolean = false,
    val createdAt: Instant = Instant.now()
)

enum class NotificationType {
    TASK_ASSIGNED, TASK_COMMENTED, TASK_DUE_SOON, TASK_COMPLETED,
    MENTIONED, PROJECT_INVITED
}

// service/NotificationService.kt
@Service
class NotificationService(
    private val notificationRepository: NotificationRepository,
    private val emailService: EmailService,
    private val webSocketService: WebSocketNotificationService,
    private val webhookService: WebhookService
) {
    fun notifyTaskAssigned(assignee: User, task: Task, assignedByUserId: Long) {
        val notification = notificationRepository.save(
            Notification(
                organizationId = task.organizationId,
                recipientId = assignee.id,
                actorId = assignedByUserId,
                type = NotificationType.TASK_ASSIGNED,
                entityType = "Task",
                entityId = task.id,
                message = "You have been assigned to '${task.title}'"
            )
        )

        // In-app notification via WebSocket
        webSocketService.sendToUser(
            userId = assignee.id,
            payload = NotificationPayload(notification)
        )

        // Email notification (async)
        emailService.sendTaskAssignedEmail(assignee, task)

        // Webhook
        webhookService.dispatch(
            orgId = task.organizationId,
            event = WebhookEvent.TASK_ASSIGNED,
            payload = TaskWebhookPayload(task, assignee)
        )
    }

    fun getUnreadCount(userId: Long, orgId: Long): Int =
        notificationRepository.countByRecipientIdAndOrganizationIdAndIsReadFalse(userId, orgId)

    fun markAllRead(userId: Long, orgId: Long) =
        notificationRepository.markAllReadByRecipientIdAndOrganizationId(userId, orgId)
}
```

---

## 🌐 6. Webhook System

```kotlin
// entity/Webhook.kt
@Entity
data class Webhook(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long = 0,

    val organizationId: Long,
    val url: String,
    val secret: String = UUID.randomUUID().toString(),

    @ElementCollection
    val events: Set<String>,  // ["task.created", "task.updated"]

    val isActive: Boolean = true,
    val createdAt: Instant = Instant.now()
)

// service/WebhookService.kt
@Service
class WebhookService(
    private val webhookRepository: WebhookRepository,
    private val restTemplate: RestTemplate
) {
    @Async
    fun dispatch(orgId: Long, event: WebhookEvent, payload: Any) {
        val webhooks = webhookRepository.findByOrganizationIdAndIsActiveAndEventsContaining(
            orgId, true, event.name.lowercase()
        )

        webhooks.forEach { webhook ->
            try {
                val body = objectMapper.writeValueAsString(
                    WebhookPayload(
                        event = event.name,
                        timestamp = Instant.now(),
                        data = payload
                    )
                )

                // Sign with HMAC
                val signature = hmacSha256(body, webhook.secret)

                val headers = HttpHeaders().apply {
                    contentType = MediaType.APPLICATION_JSON
                    set("X-Webhook-Signature", "sha256=$signature")
                    set("X-Webhook-Event", event.name)
                }

                restTemplate.exchange(
                    webhook.url,
                    HttpMethod.POST,
                    HttpEntity(body, headers),
                    String::class.java
                )

                webhookRepository.updateLastDelivery(webhook.id, Instant.now(), true)
            } catch (e: Exception) {
                webhookRepository.updateLastDelivery(webhook.id, Instant.now(), false)
            }
        }
    }

    private fun hmacSha256(data: String, secret: String): String {
        val mac = Mac.getInstance("HmacSHA256")
        mac.init(SecretKeySpec(secret.toByteArray(), "HmacSHA256"))
        return mac.doFinal(data.toByteArray())
            .joinToString("") { "%02x".format(it) }
    }
}
```

---

## 📊 7. Analytics

```kotlin
// service/AnalyticsService.kt
@Service
@Transactional(readOnly = true)
class AnalyticsService(
    private val taskRepository: TaskRepository
) {
    fun getProjectStats(projectId: Long): ProjectStats {
        val orgId = TenantContext.getCurrentTenant()

        return ProjectStats(
            totalTasks = taskRepository.countByOrganizationIdAndProjectId(orgId, projectId),
            completedTasks = taskRepository.countByOrganizationIdAndProjectIdAndStatus(
                orgId, projectId, TaskStatus.DONE
            ),
            overdueTasks = taskRepository.countByOrganizationIdAndProjectIdAndDueDateBeforeAndStatusNotIn(
                orgId, projectId, LocalDate.now(), listOf(TaskStatus.DONE, TaskStatus.CANCELLED)
            ),
            tasksByStatus = TaskStatus.values().associate { status ->
                status.name to taskRepository.countByOrganizationIdAndProjectIdAndStatus(orgId, projectId, status)
            },
            tasksByPriority = TaskPriority.values().associate { priority ->
                priority.name to taskRepository.countByOrganizationIdAndProjectIdAndPriority(orgId, projectId, priority)
            }
        )
    }
}
```

---

## 📋 สรุป

| Feature | Design |
|---------|--------|
| Multi-tenancy | organizationId column ใน every table |
| Tenant isolation | TenantContext ThreadLocal + repository filter |
| Real-time | WebSocket สำหรับ in-app notifications |
| Webhooks | HMAC-signed POST requests |
| Analytics | Aggregation queries per project |
| Activity Feed | Event log สำหรับ audit trail |

---

*Part 93/100+ | Kotlin & Spring Boot Complete Course*
