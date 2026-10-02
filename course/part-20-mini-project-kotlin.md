# Part 20: ทบทวนและ Mini Project
## Review & Mini Project: Task Management System

---

## 🎯 เป้าหมายของ Part นี้

- ทบทวนทุกสิ่งที่เรียนมา (Part 01-19)
- สร้าง Task Management System ครบฟีเจอร์
- ประยุกต์ใช้ทุก concept ในโปรเจกต์จริง
- เตรียมความพร้อมสำหรับ Spring Boot

---

## 📋 1. ทบทวน Kotlin Concepts

### Checklist

```
✅ Variables & Types    - val/var, nullable, type inference
✅ Operators            - arithmetic, comparison, string templates
✅ Control Flow         - if/when/for/while
✅ Functions            - default params, extension, higher-order
✅ Classes              - constructors, properties, companion
✅ Inheritance          - open, abstract, interface
✅ Data/Sealed/Enum     - data class, sealed class, enum
✅ Collections          - List, Map, Set, operations
✅ Lambda & HOF         - filter, map, reduce, scope functions
✅ Null Safety          - ?., ?:, let
✅ Generics             - type parameters, variance
✅ Coroutines           - basic async programming
✅ Exceptions           - try/catch, Result
✅ File I/O             - read/write files
✅ Functional           - composition, lazy sequences
✅ Scope Functions      - let, run, with, apply, also
✅ Delegation           - by keyword, property delegates
```

---

## 🏗️ 2. Mini Project: Task Management System

สร้างระบบจัดการงาน (Task Management System) ครบฟีเจอร์ที่ใช้ทุก concept จาก Part 01-19

### โครงสร้างโปรเจกต์

```
task-manager/
├── src/
│   ├── model/
│   │   ├── Task.kt
│   │   ├── Project.kt
│   │   └── User.kt
│   ├── repository/
│   │   ├── TaskRepository.kt
│   │   └── ProjectRepository.kt
│   ├── service/
│   │   ├── TaskService.kt
│   │   └── ProjectService.kt
│   ├── util/
│   │   ├── Extensions.kt
│   │   └── Validators.kt
│   └── Main.kt
```

---

## 📦 3. Models

```kotlin
// Task.kt
import java.time.LocalDateTime
import java.time.format.DateTimeFormatter

enum class Priority { LOW, MEDIUM, HIGH, CRITICAL }

enum class Status {
    TODO, IN_PROGRESS, REVIEW, DONE, CANCELLED;
    
    fun canTransitionTo(next: Status): Boolean = when (this) {
        TODO        -> next in listOf(IN_PROGRESS, CANCELLED)
        IN_PROGRESS -> next in listOf(REVIEW, TODO, CANCELLED)
        REVIEW      -> next in listOf(DONE, IN_PROGRESS, CANCELLED)
        DONE        -> false
        CANCELLED   -> false
    }
    
    val isTerminal: Boolean get() = this in listOf(DONE, CANCELLED)
    val displayName: String get() = name.replace("_", " ")
}

data class Tag(val name: String) {
    init { require(name.isNotBlank()) { "Tag name cannot be blank" } }
    override fun toString() = "#$name"
}

data class Task(
    val id: String,
    val title: String,
    val description: String = "",
    val priority: Priority = Priority.MEDIUM,
    var status: Status = Status.TODO,
    val projectId: String? = null,
    val assigneeId: String? = null,
    val tags: Set<Tag> = emptySet(),
    val createdAt: LocalDateTime = LocalDateTime.now(),
    var updatedAt: LocalDateTime = LocalDateTime.now(),
    var dueDate: LocalDateTime? = null,
    var completedAt: LocalDateTime? = null
) {
    val isOverdue: Boolean get() = 
        dueDate?.let { it < LocalDateTime.now() && !status.isTerminal } ?: false
    
    val age: Long get() = 
        java.time.Duration.between(createdAt, LocalDateTime.now()).toDays()
    
    fun withStatus(newStatus: Status): Task {
        require(status.canTransitionTo(newStatus)) {
            "Cannot transition from $status to $newStatus"
        }
        return copy(
            status = newStatus,
            updatedAt = LocalDateTime.now(),
            completedAt = if (newStatus == Status.DONE) LocalDateTime.now() else completedAt
        )
    }
    
    fun withPriority(newPriority: Priority) = copy(
        priority = newPriority,
        updatedAt = LocalDateTime.now()
    )
    
    fun withTag(tag: Tag) = copy(
        tags = tags + tag,
        updatedAt = LocalDateTime.now()
    )
    
    fun withDueDate(date: LocalDateTime) = copy(
        dueDate = date,
        updatedAt = LocalDateTime.now()
    )
    
    override fun toString(): String {
        val formatter = DateTimeFormatter.ofPattern("dd/MM/yyyy")
        return buildString {
            append("[${status.displayName}] $title")
            append(" (${priority.name})")
            dueDate?.let { append(" due: ${it.format(formatter)}") }
            if (isOverdue) append(" ⚠️ OVERDUE")
            if (tags.isNotEmpty()) append(" ${tags.joinToString(" ")}")
        }
    }
}

// Project.kt
data class Project(
    val id: String,
    val name: String,
    val description: String = "",
    val ownerId: String,
    val createdAt: LocalDateTime = LocalDateTime.now(),
    var isActive: Boolean = true
) {
    override fun toString() = "[$id] $name"
}

// User.kt
data class User(
    val id: String,
    val name: String,
    val email: String,
    val role: UserRole = UserRole.MEMBER
) {
    enum class UserRole { ADMIN, MANAGER, MEMBER, VIEWER }
    
    val displayName: String get() = "$name ($role)"
    
    init {
        require(email.contains("@")) { "Invalid email: $email" }
        require(name.isNotBlank()) { "Name cannot be blank" }
    }
}
```

---

## 💾 4. Repository

```kotlin
// TaskRepository.kt
import java.time.LocalDateTime

sealed class RepositoryResult<out T> {
    data class Success<T>(val data: T) : RepositoryResult<T>()
    data class Failure(val error: String) : RepositoryResult<Nothing>()
    object NotFound : RepositoryResult<Nothing>()
}

interface Repository<T, ID> {
    fun findById(id: ID): RepositoryResult<T>
    fun findAll(): List<T>
    fun save(entity: T): RepositoryResult<T>
    fun delete(id: ID): Boolean
    fun exists(id: ID): Boolean
}

class TaskRepository : Repository<Task, String> {
    private val tasks = mutableMapOf<String, Task>()
    
    override fun findById(id: String): RepositoryResult<Task> {
        return tasks[id]?.let { RepositoryResult.Success(it) } 
            ?: RepositoryResult.NotFound
    }
    
    override fun findAll(): List<Task> = tasks.values.toList()
    
    override fun save(entity: Task): RepositoryResult<Task> {
        tasks[entity.id] = entity
        return RepositoryResult.Success(entity)
    }
    
    override fun delete(id: String): Boolean = tasks.remove(id) != null
    
    override fun exists(id: String): Boolean = id in tasks
    
    // Query methods
    fun findByStatus(status: Status): List<Task> =
        tasks.values.filter { it.status == status }
    
    fun findByPriority(priority: Priority): List<Task> =
        tasks.values.filter { it.priority == priority }
    
    fun findByProject(projectId: String): List<Task> =
        tasks.values.filter { it.projectId == projectId }
    
    fun findByAssignee(userId: String): List<Task> =
        tasks.values.filter { it.assigneeId == userId }
    
    fun findOverdue(): List<Task> =
        tasks.values.filter { it.isOverdue }
    
    fun findByTag(tag: String): List<Task> =
        tasks.values.filter { task -> task.tags.any { it.name == tag } }
    
    fun search(query: String): List<Task> =
        tasks.values.filter { 
            it.title.contains(query, ignoreCase = true) || 
            it.description.contains(query, ignoreCase = true)
        }
    
    fun statistics(): Map<Status, Int> =
        Status.values().associateWith { status ->
            tasks.values.count { it.status == status }
        }
}

class ProjectRepository : Repository<Project, String> {
    private val projects = mutableMapOf<String, Project>()
    
    override fun findById(id: String) = 
        projects[id]?.let { RepositoryResult.Success(it) } ?: RepositoryResult.NotFound
    
    override fun findAll() = projects.values.toList()
    
    override fun save(entity: Project) = 
        RepositoryResult.Success(entity.also { projects[entity.id] = it })
    
    override fun delete(id: String) = projects.remove(id) != null
    
    override fun exists(id: String) = id in projects
    
    fun findActive() = projects.values.filter { it.isActive }
}
```

---

## ⚙️ 5. Service

```kotlin
// TaskService.kt
import java.time.LocalDateTime
import java.util.UUID

class TaskService(
    private val taskRepo: TaskRepository,
    private val projectRepo: ProjectRepository
) {
    // Create
    fun createTask(
        title: String,
        description: String = "",
        priority: Priority = Priority.MEDIUM,
        projectId: String? = null,
        assigneeId: String? = null,
        tags: Set<String> = emptySet(),
        dueDays: Int? = null
    ): Result<Task> = runCatching {
        require(title.isNotBlank()) { "Title cannot be blank" }
        require(title.length <= 200) { "Title too long (max 200 chars)" }
        
        // Validate project exists
        projectId?.let { pid ->
            require(projectRepo.exists(pid)) { "Project $pid not found" }
        }
        
        val task = Task(
            id = UUID.randomUUID().toString().take(8),
            title = title.trim(),
            description = description,
            priority = priority,
            projectId = projectId,
            assigneeId = assigneeId,
            tags = tags.map { Tag(it) }.toSet(),
            dueDate = dueDays?.let { LocalDateTime.now().plusDays(it.toLong()) }
        )
        
        when (val result = taskRepo.save(task)) {
            is RepositoryResult.Success -> result.data
            else -> throw RuntimeException("Failed to save task")
        }
    }
    
    // Update status
    fun updateStatus(taskId: String, newStatus: Status): Result<Task> = runCatching {
        val task = getTaskOrThrow(taskId)
        val updated = task.withStatus(newStatus)
        taskRepo.save(updated)
        updated
    }
    
    // Assign task
    fun assignTask(taskId: String, userId: String): Result<Task> = runCatching {
        val task = getTaskOrThrow(taskId)
        val updated = task.copy(assigneeId = userId, updatedAt = LocalDateTime.now())
        taskRepo.save(updated)
        updated
    }
    
    // Delete
    fun deleteTask(taskId: String): Result<Boolean> = runCatching {
        val task = getTaskOrThrow(taskId)
        require(!task.status.isTerminal || task.status == Status.CANCELLED) {
            "Cannot delete completed tasks"
        }
        taskRepo.delete(taskId)
    }
    
    // Queries
    fun getTasksByStatus(status: Status) = taskRepo.findByStatus(status)
    fun getOverdueTasks() = taskRepo.findOverdue()
    fun getTasksByProject(projectId: String) = taskRepo.findByProject(projectId)
    fun searchTasks(query: String) = taskRepo.search(query)
    
    // Statistics
    fun generateReport(): TaskReport {
        val all = taskRepo.findAll()
        val stats = taskRepo.statistics()
        
        return TaskReport(
            total = all.size,
            byStatus = stats,
            byPriority = Priority.values().associateWith { p ->
                all.count { it.priority == p }
            },
            overdue = taskRepo.findOverdue().size,
            completionRate = if (all.isEmpty()) 0.0 else
                all.count { it.status == Status.DONE }.toDouble() / all.size * 100
        )
    }
    
    private fun getTaskOrThrow(id: String): Task = when (val result = taskRepo.findById(id)) {
        is RepositoryResult.Success -> result.data
        is RepositoryResult.NotFound -> throw NoSuchElementException("Task $id not found")
        is RepositoryResult.Failure -> throw RuntimeException(result.error)
    }
}

data class TaskReport(
    val total: Int,
    val byStatus: Map<Status, Int>,
    val byPriority: Map<Priority, Int>,
    val overdue: Int,
    val completionRate: Double
) {
    fun print() {
        println("=".repeat(50))
        println("           TASK REPORT")
        println("=".repeat(50))
        println("Total tasks: $total")
        println("\nBy Status:")
        byStatus.forEach { (status, count) ->
            val bar = "█".repeat(count)
            println("  %-15s %3d %s".format(status.displayName, count, bar))
        }
        println("\nBy Priority:")
        byPriority.forEach { (priority, count) ->
            println("  %-10s : $count".format(priority.name))
        }
        println("\nOverdue: $overdue")
        println("Completion Rate: ${"%.1f".format(completionRate)}%")
        println("=".repeat(50))
    }
}
```

---

## 🛠️ 6. Utilities

```kotlin
// Extensions.kt
import java.time.LocalDateTime
import java.time.format.DateTimeFormatter

fun LocalDateTime.toDisplayString(): String =
    format(DateTimeFormatter.ofPattern("dd MMM yyyy HH:mm"))

fun LocalDateTime.isToday(): Boolean {
    val today = LocalDateTime.now()
    return toLocalDate() == today.toLocalDate()
}

fun List<Task>.printSummary() {
    if (isEmpty()) {
        println("No tasks found")
        return
    }
    println("Found ${size} task(s):")
    forEachIndexed { i, task ->
        println("  ${i + 1}. $task")
    }
}

fun List<Task>.groupByPriority(): Map<Priority, List<Task>> =
    groupBy { it.priority }

fun List<Task>.sortByPriorityAndDue(): List<Task> =
    sortedWith(compareByDescending<Task> { it.priority.ordinal }
        .thenBy { it.dueDate ?: LocalDateTime.MAX })
```

---

## 🚀 7. Main Application

```kotlin
// Main.kt
import java.time.LocalDateTime

fun main() {
    println("=".repeat(60))
    println("    Task Management System - Kotlin Demo")
    println("=".repeat(60))
    println()
    
    // Setup
    val taskRepo = TaskRepository()
    val projectRepo = ProjectRepository()
    val taskService = TaskService(taskRepo, projectRepo)
    
    // Create projects
    val kotlinProject = Project("P001", "Kotlin Learning", ownerId = "U001")
    val webProject = Project("P002", "Web Application", ownerId = "U001")
    projectRepo.save(kotlinProject)
    projectRepo.save(webProject)
    
    // Create users
    val alice = User("U001", "Alice Smith", "alice@example.com", User.UserRole.MANAGER)
    val bob = User("U002", "Bob Jones", "bob@example.com")
    
    println("📋 Creating tasks...")
    println()
    
    // Create tasks
    val t1 = taskService.createTask(
        title = "Learn Kotlin basics",
        priority = Priority.HIGH,
        projectId = "P001",
        assigneeId = "U001",
        tags = setOf("kotlin", "learning"),
        dueDays = 7
    ).getOrThrow()
    
    val t2 = taskService.createTask(
        title = "Build REST API with Spring Boot",
        description = "Create CRUD endpoints for user management",
        priority = Priority.HIGH,
        projectId = "P002",
        assigneeId = "U001",
        tags = setOf("spring", "api"),
        dueDays = 14
    ).getOrThrow()
    
    val t3 = taskService.createTask(
        title = "Write unit tests",
        priority = Priority.MEDIUM,
        projectId = "P002",
        assigneeId = "U002",
        tags = setOf("testing"),
        dueDays = 10
    ).getOrThrow()
    
    val t4 = taskService.createTask(
        title = "Setup CI/CD pipeline",
        priority = Priority.LOW,
        projectId = "P002",
        tags = setOf("devops"),
        dueDays = -1  // already overdue!
    ).getOrThrow()
    
    val t5 = taskService.createTask(
        title = "Code review",
        priority = Priority.MEDIUM,
        projectId = "P002",
        assigneeId = "U002"
    ).getOrThrow()
    
    // Show all tasks
    println("📌 All Tasks:")
    taskRepo.findAll().sortByPriorityAndDue().printSummary()
    println()
    
    // Update statuses
    println("🔄 Updating task statuses...")
    taskService.updateStatus(t1.id, Status.IN_PROGRESS)
    taskService.updateStatus(t2.id, Status.IN_PROGRESS)
    taskService.updateStatus(t3.id, Status.REVIEW)
    taskService.updateStatus(t1.id, Status.REVIEW)
    taskService.updateStatus(t1.id, Status.DONE)
    println()
    
    // Try invalid transition
    println("❌ Testing invalid transition (DONE → IN_PROGRESS):")
    val invalidResult = taskService.updateStatus(t1.id, Status.IN_PROGRESS)
    invalidResult.onFailure { println("  Error: ${it.message}") }
    println()
    
    // Show tasks by status
    println("📊 Tasks by Status:")
    Status.values().forEach { status ->
        val tasks = taskService.getTasksByStatus(status)
        if (tasks.isNotEmpty()) {
            println("  ${status.displayName} (${tasks.size}):")
            tasks.forEach { println("    - $it") }
        }
    }
    println()
    
    // Overdue tasks
    val overdue = taskService.getOverdueTasks()
    if (overdue.isNotEmpty()) {
        println("⚠️  Overdue Tasks:")
        overdue.printSummary()
        println()
    }
    
    // Search
    println("🔍 Search for 'api':")
    taskService.searchTasks("api").printSummary()
    println()
    
    // Project tasks
    println("📁 Tasks in '${webProject.name}':")
    taskService.getTasksByProject("P002").printSummary()
    println()
    
    // Generate report
    println()
    taskService.generateReport().print()
}
```

---

## 📊 8. Output ที่ควรได้

```
============================================================
    Task Management System - Kotlin Demo
============================================================

📋 Creating tasks...

📌 All Tasks:
Found 5 task(s):
  1. [TODO] Learn Kotlin basics (HIGH) due: 09/10/2026 #kotlin #learning
  2. [TODO] Build REST API with Spring Boot (HIGH) due: 16/10/2026 #spring #api
  3. [TODO] Write unit tests (MEDIUM) due: 12/10/2026 #testing
  4. [TODO] Code review (MEDIUM)
  5. [TODO] Setup CI/CD pipeline (LOW) due: 01/10/2026 ⚠️ OVERDUE #devops

🔄 Updating task statuses...

❌ Testing invalid transition (DONE → IN_PROGRESS):
  Error: Cannot transition from DONE to IN_PROGRESS

📊 Tasks by Status:
  TODO (1):
    - [TODO] Setup CI/CD pipeline (LOW) due: 01/10/2026 ⚠️ OVERDUE #devops
  IN PROGRESS (1):
    - [IN PROGRESS] Build REST API with Spring Boot (HIGH) due: 16/10/2026 #spring #api
  REVIEW (2):
    - [REVIEW] Write unit tests (MEDIUM) due: 12/10/2026 #testing
    - [REVIEW] Code review (MEDIUM)
  DONE (1):
    - [DONE] Learn Kotlin basics (HIGH) due: 09/10/2026 #kotlin #learning

⚠️  Overdue Tasks:
Found 1 task(s):
  1. [TODO] Setup CI/CD pipeline (LOW) due: 01/10/2026 ⚠️ OVERDUE #devops

🔍 Search for 'api':
Found 1 task(s):
  1. [IN PROGRESS] Build REST API with Spring Boot (HIGH) #spring #api

📁 Tasks in 'Web Application':
Found 4 task(s):
  ...

==================================================
           TASK REPORT
==================================================
Total tasks: 5

By Status:
  TODO            1 █
  IN PROGRESS     1 █
  REVIEW          2 ██
  DONE            1 █
  CANCELLED       0

By Priority:
  LOW        : 1
  MEDIUM     : 2
  HIGH       : 2
  CRITICAL   : 0

Overdue: 1
Completion Rate: 20.0%
==================================================
```

---

## 🏆 9. Challenge: ขยาย Project

ลองเพิ่มฟีเจอร์เหล่านี้:

```kotlin
// Challenge 1: Subtasks
data class SubTask(
    val id: String,
    val parentId: String,
    val title: String,
    var isDone: Boolean = false
)

// Challenge 2: Comments
data class Comment(
    val id: String,
    val taskId: String,
    val userId: String,
    val text: String,
    val createdAt: LocalDateTime = LocalDateTime.now()
)

// Challenge 3: Time tracking
data class TimeEntry(
    val taskId: String,
    val userId: String,
    val startTime: LocalDateTime,
    val endTime: LocalDateTime? = null
) {
    val duration: Long? get() = endTime?.let {
        java.time.Duration.between(startTime, it).toMinutes()
    }
}

// Challenge 4: Export ไปเป็น CSV/JSON
fun List<Task>.toCsv(): String {
    val header = "id,title,status,priority,assignee"
    val rows = map { "${it.id},\"${it.title}\",${it.status},${it.priority},${it.assigneeId ?: ""}" }
    return (listOf(header) + rows).joinToString("\n")
}

// Challenge 5: Import จาก JSON
// ใช้ kotlinx.serialization หรือ Gson
```

---

## 📝 สรุป Phase 1 (Part 01-20)

ตอนนี้คุณรู้จัก Kotlin ในระดับที่:
- เขียน Kotlin ได้คล่อง
- ใช้ OOP และ Functional programming
- จัดการ Null Safety
- ใช้ Collections อย่างมืออาชีพ
- เข้าใจ Coroutines พื้นฐาน
- สร้าง mini project จริงได้

---

## ➡️ ถัดไป: Part 21 - แนะนำ Spring Boot

ใน Part ถัดไปเราเริ่ม **Phase 2: Spring Boot** แล้ว!

จะเรียนรู้:
- Spring Framework คืออะไร
- Spring Boot vs Spring Framework
- ตั้งค่า project ด้วย Spring Initializr
- แนวคิด IoC และ Dependency Injection

---
*Part 20/100+ | Kotlin & Spring Boot Complete Course*
