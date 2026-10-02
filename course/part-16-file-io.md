# Part 16: File I/O
## File Input/Output in Kotlin

---

## 🎯 เป้าหมายของ Part นี้

- อ่านและเขียนไฟล์ด้วย Kotlin
- ใช้ Java I/O API ใน Kotlin
- ใช้ Kotlin extension functions สำหรับ File
- Path และ directory operations
- JSON serialization/deserialization
- CSV reading/writing

---

## 📂 1. การอ่านไฟล์

### อ่านไฟล์ทั้งหมด

```kotlin
import java.io.File

fun main() {
    val file = File("data.txt")
    
    // อ่านทั้งไฟล์เป็น String
    val content = file.readText()
    println(content)
    
    // อ่านเป็น List<String> (แต่ละบรรทัด)
    val lines = file.readLines()
    lines.forEachIndexed { i, line ->
        println("${i + 1}: $line")
    }
    
    // อ่านทีละบรรทัดแบบ lazy (ประหยัด memory)
    file.forEachLine { line ->
        println(line)
    }
    
    // อ่านเป็น bytes
    val bytes = file.readBytes()
    println("File size: ${bytes.size} bytes")
}
```

### อ่านไฟล์แบบ safe

```kotlin
import java.io.File

fun readFileSafely(path: String): Result<String> = runCatching {
    File(path).readText(Charsets.UTF_8)
}

fun main() {
    val result = readFileSafely("config.txt")
    
    result.fold(
        onSuccess = { content ->
            println("File content:\n$content")
        },
        onFailure = { error ->
            println("Error reading file: ${error.message}")
        }
    )
    
    // หรือแบบสั้น
    readFileSafely("data.txt")
        .onSuccess { println(it) }
        .onFailure { println("Error: ${it.message}") }
}
```

### ใช้ BufferedReader (ไฟล์ใหญ่)

```kotlin
import java.io.File

fun processLargeFile(path: String) {
    var lineCount = 0
    var wordCount = 0
    var charCount = 0
    
    File(path).bufferedReader().use { reader ->
        reader.lineSequence().forEach { line ->
            lineCount++
            wordCount += line.split(Regex("\\s+")).filter { it.isNotEmpty() }.size
            charCount += line.length
        }
    }
    
    println("Lines: $lineCount")
    println("Words: $wordCount")
    println("Characters: $charCount")
}
```

---

## ✏️ 2. การเขียนไฟล์

```kotlin
import java.io.File

fun main() {
    // เขียน String ทั้งหมด (overwrite)
    File("output.txt").writeText("Hello, World!\nKotlin is awesome!")
    
    // เขียนแบบ append (ต่อท้าย)
    File("log.txt").appendText("New log entry\n")
    
    // เขียนหลาย lines
    val lines = listOf("Line 1", "Line 2", "Line 3")
    File("lines.txt").writeText(lines.joinToString("\n"))
    
    // เขียนแบบ buffered (ไฟล์ใหญ่)
    File("large.txt").bufferedWriter().use { writer ->
        for (i in 1..1_000_000) {
            writer.write("Record $i: ${System.currentTimeMillis()}\n")
        }
    }
    
    // เขียน bytes
    val data = byteArrayOf(0x48, 0x65, 0x6C, 0x6C, 0x6F)  // "Hello"
    File("binary.bin").writeBytes(data)
    
    println("Files written successfully")
}
```

---

## 📁 3. Directory Operations

```kotlin
import java.io.File
import java.nio.file.Files
import java.nio.file.Path
import java.nio.file.Paths

fun main() {
    // สร้าง directory
    val dir = File("my-directory")
    dir.mkdir()         // สร้าง single directory
    
    File("parent/child/grandchild").mkdirs()  // สร้างทั้ง tree
    
    // ตรวจสอบ
    println(dir.exists())       // true
    println(dir.isDirectory)    // true
    println(dir.isFile)         // false
    
    // list files
    val currentDir = File(".")
    currentDir.listFiles()?.forEach { file ->
        val type = if (file.isDirectory) "DIR" else "FILE"
        println("$type: ${file.name} (${file.length()} bytes)")
    }
    
    // filter files
    val kotlinFiles = File(".").walk()
        .filter { it.isFile && it.extension == "kt" }
        .toList()
    
    println("Kotlin files found: ${kotlinFiles.size}")
    kotlinFiles.forEach { println("  ${it.relativeTo(File(".")).path}") }
    
    // ลบ
    File("temp.txt").delete()
    File("empty-dir").deleteRecursively()  // ลบ directory และทุกอย่างใน
    
    // เปลี่ยนชื่อ/ย้าย
    File("old.txt").renameTo(File("new.txt"))
}
```

### Walk directory tree

```kotlin
import java.io.File

fun analyzeDirectory(path: String) {
    val root = File(path)
    var fileCount = 0
    var dirCount = 0
    var totalSize = 0L
    
    root.walk().forEach { entry ->
        when {
            entry.isFile -> {
                fileCount++
                totalSize += entry.length()
            }
            entry.isDirectory && entry != root -> {
                dirCount++
            }
        }
    }
    
    println("Directory: $path")
    println("Files: $fileCount")
    println("Directories: $dirCount")
    println("Total size: ${totalSize / 1024} KB")
}

// Print directory tree
fun printTree(dir: File, indent: String = "") {
    println("$indent${dir.name}/")
    dir.listFiles()?.sortedWith(compareBy({ !it.isDirectory }, { it.name }))?.forEach { file ->
        if (file.isDirectory) {
            printTree(file, "$indent  ")
        } else {
            println("$indent  ${file.name} (${file.length()} bytes)")
        }
    }
}
```

---

## 📊 4. CSV Reading/Writing

```kotlin
import java.io.File

data class Employee(
    val id: Int,
    val name: String,
    val department: String,
    val salary: Double
)

object CsvProcessor {
    fun readEmployees(filename: String): List<Employee> {
        return File(filename).readLines()
            .drop(1)  // skip header
            .filter { it.isNotBlank() }
            .map { line ->
                val cols = line.split(",")
                Employee(
                    id = cols[0].trim().toInt(),
                    name = cols[1].trim(),
                    department = cols[2].trim(),
                    salary = cols[3].trim().toDouble()
                )
            }
    }
    
    fun writeEmployees(employees: List<Employee>, filename: String) {
        File(filename).bufferedWriter().use { writer ->
            writer.write("ID,Name,Department,Salary\n")
            employees.forEach { emp ->
                writer.write("${emp.id},${emp.name},${emp.department},${emp.salary}\n")
            }
        }
    }
}

fun main() {
    // สร้างข้อมูลตัวอย่าง
    val employees = listOf(
        Employee(1, "Alice Smith", "Engineering", 120000.0),
        Employee(2, "Bob Johnson", "Marketing", 85000.0),
        Employee(3, "Charlie Lee", "Engineering", 110000.0),
        Employee(4, "Diana Chen", "HR", 75000.0)
    )
    
    // เขียน CSV
    CsvProcessor.writeEmployees(employees, "employees.csv")
    println("CSV written")
    
    // อ่านกลับ
    val loaded = CsvProcessor.readEmployees("employees.csv")
    loaded.forEach { println(it) }
    
    // วิเคราะห์
    val avgSalary = loaded.map { it.salary }.average()
    val byDept = loaded.groupBy { it.department }
    
    println("\nAverage salary: ${"%.2f".format(avgSalary)}")
    println("\nBy department:")
    byDept.forEach { (dept, emps) ->
        val deptAvg = emps.map { it.salary }.average()
        println("  $dept: ${emps.size} employees, avg salary = ${"%.2f".format(deptAvg)}")
    }
}
```

---

## 🔧 5. Properties Files

```kotlin
import java.io.File
import java.util.Properties

class AppProperties(filename: String) {
    private val props = Properties()
    private val file = File(filename)
    
    init {
        if (file.exists()) {
            file.inputStream().use { props.load(it) }
        }
    }
    
    operator fun get(key: String): String? = props.getProperty(key)
    
    operator fun set(key: String, value: String) {
        props.setProperty(key, value)
    }
    
    fun getOrDefault(key: String, default: String): String = 
        props.getProperty(key, default)
    
    fun save(comment: String = "") {
        file.outputStream().use { props.store(it, comment) }
    }
    
    fun toMap(): Map<String, String> = props.stringPropertyNames()
        .associateWith { props.getProperty(it) }
}

fun main() {
    val config = AppProperties("app.properties")
    
    config["database.host"] = "localhost"
    config["database.port"] = "5432"
    config["database.name"] = "myapp"
    config["app.debug"] = "true"
    
    config.save("Application Configuration")
    
    println("DB Host: ${config["database.host"]}")
    println("Debug: ${config.getOrDefault("app.debug", "false")}")
    println("All: ${config.toMap()}")
}
```

---

## 📝 6. JSON กับ kotlinx.serialization

```kotlin
// build.gradle.kts
// plugins { kotlin("plugin.serialization") version "1.9.22" }
// implementation("org.jetbrains.kotlinx:kotlinx-serialization-json:1.6.2")

import kotlinx.serialization.*
import kotlinx.serialization.json.*

@Serializable
data class User(
    val id: Int,
    val name: String,
    val email: String,
    val age: Int,
    val active: Boolean = true
)

@Serializable
data class ApiResponse<T>(
    val success: Boolean,
    val data: T?,
    val message: String = ""
)

fun main() {
    val user = User(1, "Alice", "alice@example.com", 25)
    
    // Serialize to JSON
    val json = Json.encodeToString(user)
    println(json)
    // {"id":1,"name":"Alice","email":"alice@example.com","age":25,"active":true}
    
    // Deserialize from JSON
    val decoded = Json.decodeFromString<User>(json)
    println(decoded)
    
    // Pretty print
    val prettyJson = Json { prettyPrint = true }
    println(prettyJson.encodeToString(user))
    
    // Serialize list
    val users = listOf(
        User(1, "Alice", "alice@example.com", 25),
        User(2, "Bob", "bob@example.com", 30)
    )
    val jsonList = Json.encodeToString(users)
    println(jsonList)
    
    // Write to file
    val file = java.io.File("users.json")
    file.writeText(prettyJson.encodeToString(users))
    
    // Read from file
    val loadedUsers = Json.decodeFromString<List<User>>(file.readText())
    loadedUsers.forEach { println(it) }
}
```

---

## 🏋️ 7. แบบฝึกหัด

### ข้อ 1: Log File Analyzer
```kotlin
import java.io.File
import java.time.LocalDateTime
import java.time.format.DateTimeFormatter

data class LogEntry(
    val timestamp: LocalDateTime,
    val level: String,
    val message: String
)

fun parseLogFile(filename: String): List<LogEntry> {
    val formatter = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss")
    return File(filename).readLines()
        .filter { it.isNotBlank() }
        .mapNotNull { line ->
            runCatching {
                val parts = line.split(" | ", limit = 3)
                LogEntry(
                    LocalDateTime.parse(parts[0].trim(), formatter),
                    parts[1].trim(),
                    parts[2].trim()
                )
            }.getOrNull()
        }
}

fun analyzeLog(entries: List<LogEntry>) {
    val byLevel = entries.groupBy { it.level }
    println("=== Log Analysis ===")
    println("Total entries: ${entries.size}")
    byLevel.forEach { (level, logs) ->
        println("$level: ${logs.size}")
    }
    
    val errors = byLevel["ERROR"] ?: emptyList()
    if (errors.isNotEmpty()) {
        println("\nRecent errors:")
        errors.takeLast(5).forEach { println("  ${it.timestamp}: ${it.message}") }
    }
}
```

### ข้อ 2: Config File Manager
```kotlin
class ConfigManager(private val filename: String) {
    private val data = mutableMapOf<String, String>()
    
    init { load() }
    
    private fun load() {
        val file = java.io.File(filename)
        if (!file.exists()) return
        
        file.readLines()
            .filter { it.isNotBlank() && !it.startsWith("#") }
            .forEach { line ->
                val (key, value) = line.split("=", limit = 2)
                data[key.trim()] = value.trim()
            }
    }
    
    fun save() {
        java.io.File(filename).bufferedWriter().use { writer ->
            writer.write("# Auto-generated config\n")
            data.forEach { (k, v) -> writer.write("$k=$v\n") }
        }
    }
    
    fun get(key: String, default: String = "") = data.getOrDefault(key, default)
    fun set(key: String, value: String) { data[key] = value }
    fun getInt(key: String, default: Int = 0) = get(key).toIntOrNull() ?: default
    fun getBoolean(key: String, default: Boolean = false) = 
        when (get(key).lowercase()) { "true", "1", "yes" -> true; "false", "0", "no" -> false; else -> default }
}

fun main() {
    val config = ConfigManager("app.conf")
    config.set("server.port", "8080")
    config.set("server.host", "0.0.0.0")
    config.set("debug", "true")
    config.save()
    
    println("Port: ${config.getInt("server.port", 3000)}")
    println("Debug: ${config.getBoolean("debug")}")
}
```

---

## 📝 สรุป Part 16

| ฟังก์ชัน | การใช้งาน |
|---------|---------|
| `File.readText()` | อ่านทั้งไฟล์เป็น String |
| `File.readLines()` | อ่านทุก line |
| `File.forEachLine` | อ่านทีละ line (lazy) |
| `File.writeText()` | เขียนทั้งหมด |
| `File.appendText()` | ต่อท้าย |
| `File.bufferedWriter()` | เขียนแบบ buffered |
| `File.walk()` | เดิน directory tree |
| `use {}` | auto-close resource |

---

## ➡️ ถัดไป: Part 17 - Functional Programming

---
*Part 16/100+ | Kotlin & Spring Boot Complete Course*
