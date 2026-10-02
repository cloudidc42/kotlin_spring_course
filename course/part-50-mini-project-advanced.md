# Part 50: Mini Project 3 - Production-Ready API
## สร้าง API ระดับ Production ที่ครบถ้วนสมบูรณ์

---

## 🎯 เป้าหมายของ Part นี้

สร้าง **TaskFlow API** - ระบบ Task Management แบบ Production-Ready ที่รวมทุกสิ่งที่เรียนมา:
- JWT Authentication + Refresh Tokens
- Redis Caching
- Rate Limiting (Bucket4j)
- Circuit Breaker (Resilience4j)
- API Documentation (springdoc-openapi)
- Database Migration (Flyway)
- Monitoring (Micrometer + Prometheus)
- Distributed Tracing (Zipkin)
- Full Test Coverage
- Dockerfile + docker-compose

---

## 📐 1. Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                      TASKFLOW API                           │
├─────────────────────────────────────────────────────────────┤
│  Client → Rate Limiter → Auth Filter → Controller          │
│                    ↓                                        │
│              Service Layer                                  │
│         ↓           ↓          ↓                           │
│   Repository    Cache(Redis)  Notification(Email)          │
│         ↓                          ↓                       │
│      Database              Circuit Breaker                  │
│   (PostgreSQL)             (Resilience4j)                  │
├─────────────────────────────────────────────────────────────┤
│  Monitoring: Prometheus → Grafana                           │
│  Tracing: Zipkin                                            │
└─────────────────────────────────────────────────────────────┘
```

---

## 🏗️ 2. Project Structure

```
taskflow-api/
├── src/
│   ├── main/
│   │   ├── kotlin/com/taskflow/
│   │   │   ├── TaskflowApplication.kt
│   │   │   ├── config/
│   │   │   │   ├── SecurityConfig.kt
│   │   │   │   ├── RedisConfig.kt
│   │   │   │   ├── OpenApiConfig.kt
│   │   │   │   └── WebConfig.kt
│   │   │   ├── controller/
│   │   │   │   ├── AuthController.kt
│   │   │   │   ├── TaskController.kt
│   │   │   │   ├── ProjectController.kt
│   │   │   │   └── UserController.kt
│   │   │   ├── service/
│   │   │   │   ├── AuthService.kt
│   │   │   │   ├── TaskService.kt
│   │   │   │   ├── ProjectService.kt
│   │   │   │   ├── NotificationService.kt
│   │   │   │   └── UserService.kt
│   │   │   ├── repository/
│   │   │   │   ├── TaskRepository.kt
│   │   │   │   ├── ProjectRepository.kt
│   │   │   │   └── UserRepository.kt
│   │   │   ├── entity/
│   │   │   │   ├── Task.kt
│   │   │   │   ├── Project.kt
│   │   │   │   └── User.kt
│   │   │   ├── dto/
│   │   │   │   ├── TaskDtos.kt
│   │   │   │   ├── ProjectDtos.kt
│   │   │   │   └── AuthDtos.kt
│   │   │   ├── filter/
│   │   │   │   ├── JwtAuthFilter.kt
│   │   │   │   └── RateLimitFilter.kt
│   │   │   ├── exception/
│   │   │   │   └── GlobalExceptionHandler.kt
│   │   │   └── metrics/
│   │   │       └── TaskMetrics.kt
│   │   └── resources/
│   │       ├── application.yml
│   │       ├── application-prod.yml
│   │       └── db/migration/
│   │           ├── V1__initial_schema.sql
│   │           ├── V2__add_projects.sql
│   │           └── V3__add_tags.sql
│   └── test/
│       └── kotlin/com/taskflow/
│           ├── integration/
│           │   ├── TaskIntegrationTest.kt
│           │   └── AuthIntegrationTest.kt
│           └── unit/
│               ├── TaskServiceTest.kt
│               └── AuthServiceTest.kt
├── Dockerfile
├── docker-compose.yml
└── README.md
```

---

## 🔧 3. build.gradle.kts

```kotlin
import org.jetbrains.kotlin.gradle.tasks.KotlinCompile

plugins {
    id("org.springframework.boot") version "3.3.0"
    id("io.spring.dependency-management") version "1.1.4"
    kotlin("jvm") version "1.9.24"
    kotlin("plugin.spring") version "1.9.24"
    kotlin("plugin.jpa") version "1.9.24"
    jacoco  // สำหรับ code coverage
}

group = "com.taskflow"
version = "1.0.0"

java {
    sourceCompatibility = JavaVersion.VERSION_21
}

repositories {
    mavenCentral()
}

dependencies {
    // Spring Boot Core
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
    implementation("org.springframework.boot:spring-boot-starter-security")
    implementation("org.springframework.boot:spring-boot-starter-validation")
    implementation("org.springframework.boot:spring-boot-starter-data-redis")
    implementation("org.springframework.boot:spring-boot-starter-cache")
    implementation("org.springframework.boot:spring-boot-starter-actuator")

    // Kotlin
    implementation("com.fasterxml.jackson.module:jackson-module-kotlin")
    implementation("org.jetbrains.kotlin:kotlin-reflect")

    // JWT
    implementation("io.jsonwebtoken:jjwt-api:0.12.3")
    runtimeOnly("io.jsonwebtoken:jjwt-impl:0.12.3")
    runtimeOnly("io.jsonwebtoken:jjwt-jackson:0.12.3")

    // Rate Limiting
    implementation("com.bucket4j:bucket4j-core:8.10.1")

    // Resilience4j
    implementation("io.github.resilience4j:resilience4j-spring-boot3:2.2.0")
    implementation("io.github.resilience4j:resilience4j-kotlin:2.2.0")
    implementation("org.springframework.boot:spring-boot-starter-aop")

    // OpenAPI
    implementation("org.springdoc:springdoc-openapi-starter-webmvc-ui:2.3.0")

    // Flyway
    implementation("org.flywaydb:flyway-core")
    implementation("org.flywaydb:flyway-database-postgresql")

    // Monitoring
    implementation("io.micrometer:micrometer-registry-prometheus")
    implementation("io.micrometer:micrometer-tracing-bridge-brave")
    implementation("io.zipkin.reporter2:zipkin-reporter-brave")

    // Database
    runtimeOnly("org.postgresql:postgresql")

    // Test
    testImplementation("org.springframework.boot:spring-boot-starter-test")
    testImplementation("org.springframework.security:spring-security-test")
    testImplementation("org.testcontainers:postgresql:1.19.3")
    testImplementation("org.testcontainers:junit-jupiter:1.19.3")
    testImplementation("io.rest-assured:rest-assured:5.4.0")
    testImplementation("io.rest-assured:kotlin-extensions:5.4.0")
}

tasks.withType<KotlinCompile> {
    kotlinOptions {
        freeCompilerArgs += "-Xjsr305=strict"
        jvmTarget = "21"
    }
}

tasks.withType<Test> {
    useJUnitPlatform()
}

// JaCoCo coverage report
tasks.jacocoTestReport {
    reports {
        xml.required.set(true)
        html.required.set(true)
    }
}

tasks.jacocoTestCoverageVerification {
    violationRules {
        rule {
            limit {
                minimum = "0.80".toBigDecimal()  // 80% coverage minimum
            }
        }
    }
}

tasks.check {
    dependsOn(tasks.jacocoTestCoverageVerification)
}
```

---

## 🗄️ 4. Database Migration

```sql
-- V1__initial_schema.sql
CREATE TABLE users (
    id              BIGSERIAL       PRIMARY KEY,
    email           VARCHAR(255)    NOT NULL UNIQUE,
    password_hash   VARCHAR(255)    NOT NULL,
    first_name      VARCHAR(50)     NOT NULL,
    last_name       VARCHAR(50)     NOT NULL,
    role            VARCHAR(20)     NOT NULL DEFAULT 'USER',
    active          BOOLEAN         NOT NULL DEFAULT TRUE,
    refresh_token   VARCHAR(500),
    created_at      TIMESTAMP       NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMP       NOT NULL DEFAULT NOW()
);

CREATE TABLE projects (
    id          BIGSERIAL       PRIMARY KEY,
    owner_id    BIGINT          NOT NULL REFERENCES users(id),
    name        VARCHAR(100)    NOT NULL,
    description TEXT,
    color       VARCHAR(7)      DEFAULT '#3B82F6',
    status      VARCHAR(20)     NOT NULL DEFAULT 'ACTIVE',
    created_at  TIMESTAMP       NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMP       NOT NULL DEFAULT NOW()
);

CREATE TABLE tasks (
    id              BIGSERIAL       PRIMARY KEY,
    project_id      BIGINT          NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    assignee_id     BIGINT          REFERENCES users(id),
    creator_id      BIGINT          NOT NULL REFERENCES users(id),
    title           VARCHAR(255)    NOT NULL,
    description     TEXT,
    status          VARCHAR(20)     NOT NULL DEFAULT 'TODO',
    priority        VARCHAR(10)     NOT NULL DEFAULT 'MEDIUM',
    due_date        DATE,
    completed_at    TIMESTAMP,
    created_at      TIMESTAMP       NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMP       NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_tasks_project ON tasks(project_id);
CREATE INDEX idx_tasks_assignee ON tasks(assignee_id);
CREATE INDEX idx_tasks_status ON tasks(status);
CREATE INDEX idx_tasks_due_date ON tasks(due_date);
```

```sql
-- V2__add_tags.sql
CREATE TABLE tags (
    id      BIGSERIAL       PRIMARY KEY,
    name    VARCHAR(50)     NOT NULL,
    color   VARCHAR(7)      NOT NULL DEFAULT '#6B7280',
    user_id BIGINT          NOT NULL REFERENCES users(id),
    UNIQUE (name, user_id)
);

CREATE TABLE task_tags (
    task_id BIGINT  NOT NULL REFERENCES tasks(id) ON DELETE CASCADE,
    tag_id  BIGINT  NOT NULL REFERENCES tags(id) ON DELETE CASCADE,
    PRIMARY KEY (task_id, tag_id)
);
```

---

## 📦 5. Entities

```kotlin
// entity/Task.kt
package com.taskflow.entity

import jakarta.persistence.*
import org.hibernate.annotations.CreationTimestamp
import org.hibernate.annotations.UpdateTimestamp
import java.time.LocalDate
import java.time.LocalDateTime

@Entity
@Table(name = "tasks")
data class Task(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long = 0,

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "project_id", nullable = false)
    val project: Project? = null,

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "assignee_id")
    var assignee: User? = null,

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "creator_id", nullable = false)
    val creator: User? = null,

    @Column(nullable = false)
    var title: String = "",

    @Column(columnDefinition = "TEXT")
    var description: String? = null,

    @Enumerated(EnumType.STRING)
    var status: TaskStatus = TaskStatus.TODO,

    @Enumerated(EnumType.STRING)
    var priority: TaskPriority = TaskPriority.MEDIUM,

    var dueDate: LocalDate? = null,
    var completedAt: LocalDateTime? = null,

    @ManyToMany(fetch = FetchType.LAZY)
    @JoinTable(
        name = "task_tags",
        joinColumns = [JoinColumn(name = "task_id")],
        inverseJoinColumns = [JoinColumn(name = "tag_id")]
    )
    val tags: MutableSet<Tag> = mutableSetOf(),

    @CreationTimestamp
    val createdAt: LocalDateTime = LocalDateTime.now(),

    @UpdateTimestamp
    var updatedAt: LocalDateTime = LocalDateTime.now(),

    @Version
    val version: Long = 0
)

enum class TaskStatus { TODO, IN_PROGRESS, REVIEW, DONE, CANCELLED }
enum class TaskPriority { LOW, MEDIUM, HIGH, URGENT }
```

---

## 🎮 6. Task Controller

```kotlin
// controller/TaskController.kt
package com.taskflow.controller

import com.taskflow.dto.*
import com.taskflow.service.TaskService
import io.swagger.v3.oas.annotations.Operation
import io.swagger.v3.oas.annotations.security.SecurityRequirement
import io.swagger.v3.oas.annotations.tags.Tag
import jakarta.validation.Valid
import org.springframework.data.domain.Page
import org.springframework.data.domain.Pageable
import org.springframework.data.web.PageableDefault
import org.springframework.http.HttpStatus
import org.springframework.http.ResponseEntity
import org.springframework.security.core.annotation.AuthenticationPrincipal
import org.springframework.web.bind.annotation.*

@RestController
@RequestMapping("/api/v1/tasks")
@Tag(name = "Tasks", description = "Task management endpoints")
@SecurityRequirement(name = "bearerAuth")
class TaskController(private val taskService: TaskService) {

    @Operation(summary = "Get all tasks for current user")
    @GetMapping
    fun getTasks(
        @AuthenticationPrincipal userId: Long,
        @RequestParam(required = false) status: String?,
        @RequestParam(required = false) priority: String?,
        @RequestParam(required = false) projectId: Long?,
        @PageableDefault(size = 20, sort = ["dueDate"]) pageable: Pageable
    ): ResponseEntity<Page<TaskResponse>> {
        return ResponseEntity.ok(
            taskService.getTasks(userId, status, priority, projectId, pageable)
        )
    }

    @Operation(summary = "Get task by ID")
    @GetMapping("/{id}")
    fun getTask(
        @AuthenticationPrincipal userId: Long,
        @PathVariable id: Long
    ): ResponseEntity<TaskResponse> {
        return ResponseEntity.ok(taskService.getTask(id, userId))
    }

    @Operation(summary = "Create a new task")
    @PostMapping
    fun createTask(
        @AuthenticationPrincipal userId: Long,
        @Valid @RequestBody request: CreateTaskRequest
    ): ResponseEntity<TaskResponse> {
        return ResponseEntity
            .status(HttpStatus.CREATED)
            .body(taskService.createTask(request, userId))
    }

    @Operation(summary = "Update task")
    @PutMapping("/{id}")
    fun updateTask(
        @AuthenticationPrincipal userId: Long,
        @PathVariable id: Long,
        @Valid @RequestBody request: UpdateTaskRequest
    ): ResponseEntity<TaskResponse> {
        return ResponseEntity.ok(taskService.updateTask(id, request, userId))
    }

    @Operation(summary = "Update task status")
    @PatchMapping("/{id}/status")
    fun updateTaskStatus(
        @AuthenticationPrincipal userId: Long,
        @PathVariable id: Long,
        @RequestBody body: Map<String, String>
    ): ResponseEntity<TaskResponse> {
        val status = body["status"] ?: throw IllegalArgumentException("Status is required")
        return ResponseEntity.ok(taskService.updateStatus(id, status, userId))
    }

    @Operation(summary = "Delete task")
    @DeleteMapping("/{id}")
    fun deleteTask(
        @AuthenticationPrincipal userId: Long,
        @PathVariable id: Long
    ): ResponseEntity<Void> {
        taskService.deleteTask(id, userId)
        return ResponseEntity.noContent().build()
    }

    @Operation(summary = "Get overdue tasks")
    @GetMapping("/overdue")
    fun getOverdueTasks(
        @AuthenticationPrincipal userId: Long
    ): ResponseEntity<List<TaskResponse>> {
        return ResponseEntity.ok(taskService.getOverdueTasks(userId))
    }

    @Operation(summary = "Get task statistics")
    @GetMapping("/stats")
    fun getTaskStats(
        @AuthenticationPrincipal userId: Long
    ): ResponseEntity<TaskStats> {
        return ResponseEntity.ok(taskService.getStats(userId))
    }
}
```

---

## ⚙️ 7. Task Service

```kotlin
// service/TaskService.kt
package com.taskflow.service

import com.taskflow.dto.*
import com.taskflow.entity.*
import com.taskflow.exception.AccessDeniedException
import com.taskflow.exception.NotFoundException
import com.taskflow.metrics.TaskMetrics
import com.taskflow.repository.TaskRepository
import io.github.resilience4j.circuitbreaker.annotation.CircuitBreaker
import org.slf4j.LoggerFactory
import org.springframework.cache.annotation.CacheEvict
import org.springframework.cache.annotation.Cacheable
import org.springframework.data.domain.Page
import org.springframework.data.domain.Pageable
import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional
import java.time.LocalDate
import java.time.LocalDateTime

@Service
@Transactional
class TaskService(
    private val taskRepository: TaskRepository,
    private val projectRepository: com.taskflow.repository.ProjectRepository,
    private val userRepository: com.taskflow.repository.UserRepository,
    private val notificationService: NotificationService,
    private val taskMetrics: TaskMetrics
) {
    private val logger = LoggerFactory.getLogger(javaClass)

    @Transactional(readOnly = true)
    @Cacheable(
        value = ["tasks"],
        key = "#userId + ':' + #status + ':' + #priority + ':' + #projectId + ':' + #pageable.pageNumber",
        unless = "#result.isEmpty"
    )
    fun getTasks(
        userId: Long,
        status: String?,
        priority: String?,
        projectId: Long?,
        pageable: Pageable
    ): Page<TaskResponse> {
        val taskStatus = status?.let { TaskStatus.valueOf(it.uppercase()) }
        val taskPriority = priority?.let { TaskPriority.valueOf(it.uppercase()) }

        return taskRepository.findByFilters(userId, taskStatus, taskPriority, projectId, pageable)
            .map { it.toResponse() }
    }

    @Transactional(readOnly = true)
    fun getTask(id: Long, userId: Long): TaskResponse {
        val task = findTaskAndVerifyAccess(id, userId)
        return task.toResponse()
    }

    @CacheEvict(value = ["tasks"], allEntries = true)
    fun createTask(request: CreateTaskRequest, creatorId: Long): TaskResponse {
        val project = projectRepository.findById(request.projectId)
            .orElseThrow { NotFoundException("Project not found: ${request.projectId}") }

        verifyProjectAccess(project, creatorId)

        val creator = userRepository.findById(creatorId)
            .orElseThrow { NotFoundException("User not found: $creatorId") }

        val assignee = request.assigneeId?.let { id ->
            userRepository.findById(id).orElseThrow { NotFoundException("Assignee not found: $id") }
        }

        val task = Task(
            project = project,
            creator = creator,
            assignee = assignee,
            title = request.title,
            description = request.description,
            status = TaskStatus.valueOf(request.status ?: "TODO"),
            priority = TaskPriority.valueOf(request.priority ?: "MEDIUM"),
            dueDate = request.dueDate
        )

        val savedTask = taskRepository.save(task)
        taskMetrics.recordTaskCreated(savedTask.priority.name)

        // ส่ง notification ถ้า task ถูก assign
        assignee?.let {
            try {
                notificationService.notifyTaskAssigned(savedTask, it)
            } catch (ex: Exception) {
                logger.warn("Failed to send task assignment notification", ex)
            }
        }

        logger.info("Task created: id={}, project={}, creator={}", savedTask.id, project.id, creatorId)
        return savedTask.toResponse()
    }

    @CacheEvict(value = ["tasks"], allEntries = true)
    fun updateTask(id: Long, request: UpdateTaskRequest, userId: Long): TaskResponse {
        val task = findTaskAndVerifyAccess(id, userId)

        request.title?.let { task.title = it }
        request.description?.let { task.description = it }
        request.priority?.let { task.priority = TaskPriority.valueOf(it) }
        request.dueDate?.let { task.dueDate = it }
        request.assigneeId?.let { assigneeId ->
            task.assignee = userRepository.findById(assigneeId).orElse(null)
        }

        return taskRepository.save(task).toResponse()
    }

    @CacheEvict(value = ["tasks"], allEntries = true)
    fun updateStatus(id: Long, status: String, userId: Long): TaskResponse {
        val task = findTaskAndVerifyAccess(id, userId)
        val newStatus = TaskStatus.valueOf(status.uppercase())
        val oldStatus = task.status

        task.status = newStatus
        if (newStatus == TaskStatus.DONE && task.completedAt == null) {
            task.completedAt = LocalDateTime.now()
            taskMetrics.recordTaskCompleted(
                task.priority.name,
                task.createdAt?.let { java.time.Duration.between(it, LocalDateTime.now()).toMinutes() } ?: 0
            )
        }

        logger.info("Task {} status changed: {} -> {}", id, oldStatus, newStatus)
        return taskRepository.save(task).toResponse()
    }

    @CacheEvict(value = ["tasks"], allEntries = true)
    fun deleteTask(id: Long, userId: Long) {
        val task = findTaskAndVerifyAccess(id, userId)
        taskRepository.delete(task)
        logger.info("Task deleted: id={}", id)
    }

    @Transactional(readOnly = true)
    fun getOverdueTasks(userId: Long): List<TaskResponse> {
        return taskRepository.findOverdueTasksByUser(userId, LocalDate.now())
            .map { it.toResponse() }
    }

    @Transactional(readOnly = true)
    fun getStats(userId: Long): TaskStats {
        return TaskStats(
            total = taskRepository.countByUserId(userId),
            todo = taskRepository.countByUserIdAndStatus(userId, TaskStatus.TODO),
            inProgress = taskRepository.countByUserIdAndStatus(userId, TaskStatus.IN_PROGRESS),
            review = taskRepository.countByUserIdAndStatus(userId, TaskStatus.REVIEW),
            done = taskRepository.countByUserIdAndStatus(userId, TaskStatus.DONE),
            overdue = taskRepository.countOverdueByUser(userId, LocalDate.now())
        )
    }

    private fun findTaskAndVerifyAccess(id: Long, userId: Long): Task {
        val task = taskRepository.findById(id)
            .orElseThrow { NotFoundException("Task not found: $id") }

        if (task.creator?.id != userId && task.assignee?.id != userId &&
            task.project?.owner?.id != userId) {
            throw AccessDeniedException("Access denied to task: $id")
        }

        return task
    }

    private fun verifyProjectAccess(project: com.taskflow.entity.Project, userId: Long) {
        if (project.owner?.id != userId) {
            throw AccessDeniedException("Access denied to project: ${project.id}")
        }
    }

    private fun Task.toResponse() = TaskResponse(
        id = id,
        title = title,
        description = description,
        status = status.name,
        priority = priority.name,
        dueDate = dueDate,
        completedAt = completedAt,
        projectId = project?.id,
        projectName = project?.name,
        assigneeId = assignee?.id,
        assigneeName = assignee?.let { "${it.firstName} ${it.lastName}" },
        creatorId = creator?.id,
        tags = tags.map { it.name },
        createdAt = createdAt,
        updatedAt = updatedAt
    )
}
```

---

## 🧪 8. Integration Tests

```kotlin
// test/integration/TaskIntegrationTest.kt
package com.taskflow.integration

import io.restassured.RestAssured
import io.restassured.http.ContentType
import io.restassured.module.kotlin.extensions.Extract
import io.restassured.module.kotlin.extensions.Given
import io.restassured.module.kotlin.extensions.Then
import io.restassured.module.kotlin.extensions.When
import org.hamcrest.Matchers.*
import org.junit.jupiter.api.*
import org.springframework.boot.test.context.SpringBootTest
import org.springframework.boot.test.web.server.LocalServerPort
import org.springframework.test.context.DynamicPropertyRegistry
import org.springframework.test.context.DynamicPropertySource
import org.testcontainers.containers.PostgreSQLContainer
import org.testcontainers.junit.jupiter.Container
import org.testcontainers.junit.jupiter.Testcontainers

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Testcontainers
@TestMethodOrder(MethodOrderer.OrderAnnotation::class)
class TaskIntegrationTest {

    @LocalServerPort
    private var port: Int = 0

    companion object {
        private var authToken: String = ""
        private var createdTaskId: Long = 0
        private var projectId: Long = 0

        @Container
        val postgres = PostgreSQLContainer<Nothing>("postgres:16").apply {
            withDatabaseName("taskflow_test")
            withUsername("test")
            withPassword("test")
        }

        @DynamicPropertySource
        @JvmStatic
        fun configureProperties(registry: DynamicPropertyRegistry) {
            registry.add("spring.datasource.url") { postgres.jdbcUrl }
            registry.add("spring.datasource.username") { postgres.username }
            registry.add("spring.datasource.password") { postgres.password }
        }
    }

    @BeforeEach
    fun setUp() {
        RestAssured.port = port
    }

    @Test
    @Order(1)
    fun `should register new user`() {
        Given {
            contentType(ContentType.JSON)
            body("""{"email": "test@taskflow.com", "password": "Test@123!", "firstName": "Test", "lastName": "User"}""")
        } When {
            post("/auth/register")
        } Then {
            statusCode(201)
            body("email", equalTo("test@taskflow.com"))
        }
    }

    @Test
    @Order(2)
    fun `should login and receive JWT token`() {
        authToken = Given {
            contentType(ContentType.JSON)
            body("""{"email": "test@taskflow.com", "password": "Test@123!"}""")
        } When {
            post("/auth/login")
        } Then {
            statusCode(200)
            body("accessToken", notNullValue())
        } Extract {
            path("accessToken")
        }

        assert(authToken.isNotBlank())
    }

    @Test
    @Order(3)
    fun `should create project`() {
        projectId = Given {
            contentType(ContentType.JSON)
            header("Authorization", "Bearer $authToken")
            body("""{"name": "Test Project", "description": "Integration test project"}""")
        } When {
            post("/api/v1/projects")
        } Then {
            statusCode(201)
            body("name", equalTo("Test Project"))
        } Extract {
            path<Int>("id").toLong()
        }
    }

    @Test
    @Order(4)
    fun `should create task in project`() {
        createdTaskId = Given {
            contentType(ContentType.JSON)
            header("Authorization", "Bearer $authToken")
            body("""
                {
                    "projectId": $projectId,
                    "title": "Implement authentication",
                    "description": "Add JWT auth to the API",
                    "priority": "HIGH",
                    "dueDate": "2025-12-31"
                }
            """.trimIndent())
        } When {
            post("/api/v1/tasks")
        } Then {
            statusCode(201)
            body("title", equalTo("Implement authentication"))
            body("priority", equalTo("HIGH"))
            body("status", equalTo("TODO"))
        } Extract {
            path<Int>("id").toLong()
        }
    }

    @Test
    @Order(5)
    fun `should get task by id`() {
        Given {
            header("Authorization", "Bearer $authToken")
        } When {
            get("/api/v1/tasks/$createdTaskId")
        } Then {
            statusCode(200)
            body("id", equalTo(createdTaskId.toInt()))
            body("title", equalTo("Implement authentication"))
        }
    }

    @Test
    @Order(6)
    fun `should update task status to IN_PROGRESS`() {
        Given {
            contentType(ContentType.JSON)
            header("Authorization", "Bearer $authToken")
            body("""{"status": "IN_PROGRESS"}""")
        } When {
            patch("/api/v1/tasks/$createdTaskId/status")
        } Then {
            statusCode(200)
            body("status", equalTo("IN_PROGRESS"))
        }
    }

    @Test
    @Order(7)
    fun `should get task statistics`() {
        Given {
            header("Authorization", "Bearer $authToken")
        } When {
            get("/api/v1/tasks/stats")
        } Then {
            statusCode(200)
            body("total", greaterThanOrEqualTo(1))
            body("inProgress", greaterThanOrEqualTo(1))
        }
    }

    @Test
    @Order(8)
    fun `should return 401 without token`() {
        When {
            get("/api/v1/tasks")
        } Then {
            statusCode(401)
        }
    }

    @Test
    @Order(9)
    fun `should rate limit after too many requests`() {
        // ส่ง request 25 ครั้ง (เกิน rate limit สำหรับ public tier)
        repeat(25) {
            Given {
                header("Authorization", "Bearer $authToken")
            } When {
                get("/api/v1/tasks")
            }
        }
        // Request ล่าสุดควรถูก rate limited
        Given {
            // ไม่มี token = public tier ที่ limit ต่ำกว่า
        } When {
            get("/api/v1/tasks")
        } Then {
            // 401 หรือ 429 depending on rate limit vs auth
            statusCode(oneOf(401, 429))
        }
    }
}
```

---

## 🐳 9. Dockerfile

```dockerfile
# Dockerfile
# Multi-stage build
FROM eclipse-temurin:21-jdk-alpine AS builder

WORKDIR /app

# Copy gradle files
COPY gradle gradle
COPY gradlew .
COPY build.gradle.kts .
COPY settings.gradle.kts .

# Download dependencies (cached layer)
RUN ./gradlew dependencies --no-daemon 2>/dev/null || true

# Copy source and build
COPY src src
RUN ./gradlew bootJar --no-daemon -x test

# Final image
FROM eclipse-temurin:21-jre-alpine

WORKDIR /app

# Security: non-root user
RUN addgroup -g 1001 -S appgroup && \
    adduser -u 1001 -S appuser -G appgroup

# Health check dependency
RUN apk add --no-cache curl

COPY --from=builder /app/build/libs/*.jar app.jar

# JVM optimization flags
ENV JAVA_OPTS="-XX:+UseContainerSupport \
               -XX:MaxRAMPercentage=75 \
               -XX:+UseG1GC \
               -XX:+HeapDumpOnOutOfMemoryError \
               -Djava.security.egd=file:/dev/./urandom"

USER appuser:appgroup

EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=10s --start-period=60s --retries=3 \
    CMD curl -f http://localhost:8080/actuator/health || exit 1

ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar app.jar"]
```

---

## 🐋 10. docker-compose.yml

```yaml
version: '3.8'

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "8080:8080"
    environment:
      SPRING_PROFILES_ACTIVE: prod
      DB_HOST: postgres
      DB_PORT: 5432
      DB_NAME: taskflow
      DB_USERNAME: taskflow
      DB_PASSWORD: ${DB_PASSWORD}
      REDIS_HOST: redis
      REDIS_PORT: 6379
      JWT_SECRET: ${JWT_SECRET}
      ZIPKIN_URL: http://zipkin:9411/api/v2/spans
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    restart: unless-stopped
    networks:
      - taskflow-network

  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: taskflow
      POSTGRES_USER: taskflow
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - postgres-data:/var/lib/postgresql/data
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U taskflow"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - taskflow-network

  redis:
    image: redis:7-alpine
    command: redis-server --appendonly yes --maxmemory 256mb --maxmemory-policy allkeys-lru
    volumes:
      - redis-data:/data
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - taskflow-network

  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./monitoring/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus-data:/prometheus
    networks:
      - taskflow-network

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    environment:
      GF_SECURITY_ADMIN_PASSWORD: ${GRAFANA_PASSWORD:-admin}
    volumes:
      - grafana-data:/var/lib/grafana
    depends_on:
      - prometheus
    networks:
      - taskflow-network

  zipkin:
    image: openzipkin/zipkin:latest
    ports:
      - "9411:9411"
    networks:
      - taskflow-network

volumes:
  postgres-data:
  redis-data:
  prometheus-data:
  grafana-data:

networks:
  taskflow-network:
    driver: bridge
```

---

## 📊 สรุปเนื้อหา Mini Project 3

| Feature | Technology | Status |
|---------|-----------|--------|
| Authentication | JWT + Refresh Tokens | ✅ |
| Authorization | Spring Security | ✅ |
| Database | PostgreSQL + JPA | ✅ |
| Cache | Redis | ✅ |
| Rate Limiting | Bucket4j | ✅ |
| Fault Tolerance | Resilience4j | ✅ |
| API Docs | springdoc-openapi | ✅ |
| DB Migration | Flyway | ✅ |
| Monitoring | Micrometer + Prometheus | ✅ |
| Tracing | Zipkin | ✅ |
| Testing | JUnit5 + Testcontainers | ✅ |
| Containerization | Docker + docker-compose | ✅ |
| Code Coverage | JaCoCo (80%+) | ✅ |

### API Endpoints Summary:

| Method | Endpoint | ประโยชน์ |
|--------|---------|---------|
| POST | /auth/register | สร้างบัญชี |
| POST | /auth/login | เข้าสู่ระบบ |
| POST | /auth/refresh | ต่ออายุ token |
| GET | /api/v1/tasks | รายการ task |
| POST | /api/v1/tasks | สร้าง task ใหม่ |
| GET | /api/v1/tasks/{id} | ดู task |
| PUT | /api/v1/tasks/{id} | แก้ไข task |
| PATCH | /api/v1/tasks/{id}/status | เปลี่ยน status |
| DELETE | /api/v1/tasks/{id} | ลบ task |
| GET | /api/v1/tasks/stats | สถิติ task |

### 🎓 สิ่งที่ได้เรียนรู้จาก Parts 41-50:

1. **API Versioning** - จัดการ breaking changes อย่างปลอดภัย
2. **Rate Limiting** - ป้องกัน abuse และ ensure fair usage
3. **Circuit Breaker** - สร้าง fault-tolerant system
4. **API Documentation** - สร้าง docs ที่ดีสำหรับ developers
5. **Database Migration** - จัดการ schema changes อย่างมีระบบ
6. **Performance** - แก้ N+1, caching, indexing
7. **Advanced Security** - OWASP Top 10, XSS, SQL injection
8. **OAuth2/SSO** - Social login ด้วย Google/GitHub
9. **Monitoring** - เห็น system behavior ใน production
10. **Mini Project** - รวมทุกอย่างเป็น production-ready API

---

*Part 50/100+ | Kotlin & Spring Boot Complete Course*
