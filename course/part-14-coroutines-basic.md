# Part 14: Coroutines พื้นฐาน
## Asynchronous Programming ด้วย Kotlin Coroutines

---

## 🎯 เป้าหมายของ Part นี้

- เข้าใจว่า Coroutines คืออะไร และต่างจาก Thread อย่างไร
- เขียน suspend function ได้
- ใช้ CoroutineScope, GlobalScope
- เข้าใจความต่างของ launch กับ async
- ใช้ await และ Deferred
- เลือก Dispatchers ได้ถูกต้อง
- จัดการ Job และ cancellation
- ใช้ runBlocking, withContext
- สร้าง parallel API calls และ file downloads

---

## 🤔 1. Coroutines คืออะไร?

### 1.1 Thread vs Coroutine

```
Thread (Thread-based):
┌────────────────────────────────────────┐
│ Thread 1: ████████░░░░░░████████░░░░░░ │  (blocked ระหว่างรอ I/O)
│ Thread 2: ░░░░████████░░░░░░████████░░ │
│ Thread 3: ░░░░░░░░████████░░░░░░░░░░░░ │
└────────────────────────────────────────┘
- สร้าง Thread ใหม่ทุกครั้ง = ใช้หน่วยความจำมาก
- Thread blocked = CPU ว่างแต่ Thread ยังถูกใช้งาน

Coroutine (Coroutine-based):
┌────────────────────────────────────────┐
│ Thread 1: A1 B1 A2 C1 B2 A3 C2 D1 ... │  (ทำงานหลายงานบน Thread เดียว)
└────────────────────────────────────────┘
- Coroutine ถูก suspend เมื่อรอ I/O (ไม่บล็อก Thread)
- Thread ว่างไปทำงานอื่นได้
- สร้างได้หลายหมื่น coroutines ต่อ Thread
```

### 1.2 Setup

```kotlin
// build.gradle.kts
dependencies {
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:1.7.3")
    // สำหรับ Android
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-android:1.7.3")
    // สำหรับ Spring Boot
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-reactor:1.7.3")
}
```

---

## ⏸️ 2. suspend function

`suspend` function คือฟังก์ชันที่สามารถ "หยุด" การทำงานชั่วคราวและกลับมาทำงานต่อได้ โดยไม่บล็อก Thread

### 2.1 พื้นฐาน suspend function

```kotlin
import kotlinx.coroutines.*

// suspend function - เรียกได้แค่จาก coroutine หรือ suspend function อื่น
suspend fun fetchData(): String {
    delay(1000)  // หยุด 1 วินาที (ไม่บล็อก Thread!)
    return "Data from server"
}

suspend fun processData(data: String): String {
    delay(500)   // simulate processing
    return "Processed: $data"
}

// เปรียบเทียบ delay vs Thread.sleep
fun blockingExample() {
    // Thread.sleep(1000) - บล็อก Thread ทั้งหมด
    println("This blocks the thread!")
}

suspend fun nonBlockingExample() {
    // delay(1000) - suspend coroutine แต่ Thread ว่าง
    delay(1000)
    println("This doesn't block the thread!")
}

fun main() = runBlocking {
    val data = fetchData()
    val result = processData(data)
    println(result)  // Processed: Data from server
}
```

### 2.2 suspend function Chain

```kotlin
import kotlinx.coroutines.*

suspend fun getUser(userId: Int): String {
    delay(200)  // simulate network call
    return "User_$userId"
}

suspend fun getUserOrders(userId: Int): List<String> {
    delay(300)  // simulate database query
    return listOf("Order_1", "Order_2", "Order_3")
}

suspend fun calculateTotal(orders: List<String>): Double {
    delay(100)  // simulate calculation
    return orders.size * 1500.0
}

fun main() = runBlocking {
    val startTime = System.currentTimeMillis()
    
    // Sequential (ทำทีละอย่าง)
    val user = getUser(1)
    val orders = getUserOrders(1)
    val total = calculateTotal(orders)
    
    val elapsed = System.currentTimeMillis() - startTime
    println("User: $user")
    println("Orders: $orders")
    println("Total: $total")
    println("Time: ${elapsed}ms")  // ~600ms (200+300+100)
}
```

---

## 🌍 3. CoroutineScope และ GlobalScope

### 3.1 GlobalScope - ไม่แนะนำ

```kotlin
import kotlinx.coroutines.*

fun main() {
    // GlobalScope - coroutine ทำงานตลอด lifetime ของ application
    // ไม่ดี: ไม่มี structured concurrency, อาจ leak
    GlobalScope.launch {
        delay(1000)
        println("GlobalScope coroutine done")
    }
    
    // ต้องรอ เพราะ main function จบก่อน coroutine
    Thread.sleep(2000)
    println("Main done")
}
```

### 3.2 runBlocking - สำหรับ main และ tests

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    // runBlocking สร้าง CoroutineScope และบล็อก thread จนกว่า coroutines จะจบ
    // ใช้ใน: main function, unit tests
    
    println("Start: ${Thread.currentThread().name}")
    
    delay(1000)
    println("After 1 second")
    
    launch {
        delay(500)
        println("Child coroutine done")
    }
    
    println("End of runBlocking body")
    // runBlocking รอจน child coroutines ทั้งหมดจบ
}
```

### 3.3 coroutineScope - สำหรับ suspend functions

```kotlin
import kotlinx.coroutines.*

suspend fun doWork() = coroutineScope {
    // coroutineScope สร้าง scope ใหม่และรอจนทุก children จบ
    // ถ้า child ใด fail, scope ทั้งหมด cancel
    
    val job1 = launch {
        delay(1000)
        println("Job 1 done")
    }
    
    val job2 = launch {
        delay(500)
        println("Job 2 done")
    }
    
    println("Waiting for jobs...")
    // รอทั้ง job1 และ job2
}

fun main() = runBlocking {
    doWork()
    println("All work done")
}
```

### 3.4 LifecycleScope / ViewModelScope (Android)

```kotlin
// Android LifecycleScope
class MyFragment : Fragment() {
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        // cancel อัตโนมัติเมื่อ Fragment destroyed
        lifecycleScope.launch {
            val data = fetchData()
            updateUI(data)
        }
    }
}

// Spring Boot coroutineScope
@Service
class UserService {
    suspend fun getUserWithOrders(userId: Int) = coroutineScope {
        val user = async { fetchUser(userId) }
        val orders = async { fetchOrders(userId) }
        Pair(user.await(), orders.await())
    }
}
```

---

## 🚀 4. launch vs async

### 4.1 launch - Fire and Forget

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    println("Main start")
    
    // launch - เริ่ม coroutine ที่ไม่คืนค่า
    val job = launch {
        println("Coroutine start")
        delay(1000)
        println("Coroutine end")
    }
    
    println("After launch (coroutine is still running)")
    job.join()  // รอจน coroutine จบ
    println("Main end")
}
// Output:
// Main start
// After launch (coroutine is still running)
// Coroutine start
// Coroutine end
// Main end
```

### 4.2 async - Concurrent with Result

```kotlin
import kotlinx.coroutines.*

suspend fun fetchPrice(symbol: String): Double {
    delay(500)  // simulate API call
    return when (symbol) {
        "BTC" -> 2_500_000.0
        "ETH" -> 150_000.0
        "ADA" -> 25.0
        else  -> 0.0
    }
}

fun main() = runBlocking {
    // Sequential - รวม 1500ms
    val start1 = System.currentTimeMillis()
    val btc1 = fetchPrice("BTC")
    val eth1 = fetchPrice("ETH")
    val ada1 = fetchPrice("ADA")
    println("Sequential: ${System.currentTimeMillis() - start1}ms")
    println("BTC: $btc1, ETH: $eth1, ADA: $ada1")
    
    println()
    
    // Parallel ด้วย async - รวม ~500ms
    val start2 = System.currentTimeMillis()
    val btcDeferred = async { fetchPrice("BTC") }
    val ethDeferred = async { fetchPrice("ETH") }
    val adaDeferred = async { fetchPrice("ADA") }
    
    // await ทั้ง 3 พร้อมกัน
    val btc2 = btcDeferred.await()
    val eth2 = ethDeferred.await()
    val ada2 = adaDeferred.await()
    println("Parallel: ${System.currentTimeMillis() - start2}ms")
    println("BTC: $btc2, ETH: $eth2, ADA: $ada2")
}
```

### 4.3 awaitAll

```kotlin
import kotlinx.coroutines.*

suspend fun fetchUserData(userId: Int): String {
    delay((200..500).random().toLong())
    return "Data for user $userId"
}

fun main() = runBlocking {
    val userIds = listOf(1, 2, 3, 4, 5)
    
    // สร้าง Deferred list
    val deferreds = userIds.map { id ->
        async { fetchUserData(id) }
    }
    
    // รอทั้งหมดพร้อมกัน
    val results = deferreds.awaitAll()
    results.forEach { println(it) }
    
    // หรือใช้ coroutineScope + async สะอาดกว่า
    val results2 = coroutineScope {
        userIds.map { id ->
            async { fetchUserData(id) }
        }.awaitAll()
    }
    println("All ${results2.size} users loaded")
}
```

---

## ⚙️ 5. Dispatchers

Dispatcher กำหนดว่า coroutine จะทำงานบน thread ไหน

### 5.1 ประเภท Dispatchers

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    // Dispatchers.Default - CPU-intensive tasks
    // ใช้ thread pool ขนาด = จำนวน CPU cores
    launch(Dispatchers.Default) {
        // เหมาะกับ: sorting, parsing, computation
        println("Default: ${Thread.currentThread().name}")
        val result = (1..1_000_000).sum()
        println("Sum: $result")
    }
    
    // Dispatchers.IO - I/O operations
    // ใช้ thread pool ขนาด 64 หรือมากกว่า
    launch(Dispatchers.IO) {
        // เหมาะกับ: database, network, file operations
        println("IO: ${Thread.currentThread().name}")
        // simulate file read
        delay(100)
        println("File read complete")
    }
    
    // Dispatchers.Main - UI updates (Android/JavaFX)
    // ต้องมี Main dispatcher ใน project
    // launch(Dispatchers.Main) { updateUI() }
    
    // Dispatchers.Unconfined - ไม่กำหนด thread (ไม่แนะนำ)
    launch(Dispatchers.Unconfined) {
        println("Unconfined start: ${Thread.currentThread().name}")
        delay(100)
        println("Unconfined after delay: ${Thread.currentThread().name}")
    }
    
    delay(500)
}
```

### 5.2 withContext - เปลี่ยน Dispatcher

```kotlin
import kotlinx.coroutines.*
import java.io.File

suspend fun readFile(path: String): String = withContext(Dispatchers.IO) {
    // ทำงานบน IO dispatcher
    File(path).readText()
}

suspend fun parseJson(json: String): Map<String, Any> = withContext(Dispatchers.Default) {
    // ทำงานบน Default dispatcher (CPU intensive)
    // simulate parsing
    delay(100)
    mapOf("data" to json)
}

suspend fun processUserRequest(userId: Int): String {
    // 1. ดึงข้อมูลจาก DB (IO)
    val userData = withContext(Dispatchers.IO) {
        println("Fetching from DB on: ${Thread.currentThread().name}")
        delay(200)
        "user_data_$userId"
    }
    
    // 2. ประมวลผล (Default)
    val processed = withContext(Dispatchers.Default) {
        println("Processing on: ${Thread.currentThread().name}")
        delay(100)
        "processed_$userData"
    }
    
    return processed
}

fun main() = runBlocking {
    val result = processUserRequest(1)
    println("Result: $result")
}
```

### 5.3 Custom Dispatcher

```kotlin
import kotlinx.coroutines.*
import java.util.concurrent.Executors

fun main() = runBlocking {
    // สร้าง custom dispatcher จาก ExecutorService
    val customDispatcher = Executors.newFixedThreadPool(4).asCoroutineDispatcher()
    
    try {
        val jobs = (1..10).map { i ->
            launch(customDispatcher) {
                println("Task $i on: ${Thread.currentThread().name}")
                delay(100)
            }
        }
        jobs.forEach { it.join() }
    } finally {
        customDispatcher.close()  // สำคัญ! ต้อง close เสมอ
    }
}
```

---

## 📋 6. Job และ Cancellation

### 6.1 Job พื้นฐาน

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    // Job คือ handle ของ coroutine
    val job = launch {
        repeat(10) { i ->
            println("Working step $i...")
            delay(500)
        }
    }
    
    // Job states: New -> Active -> Completing -> Completed
    //                          -> Cancelling -> Cancelled
    
    println("Job is active: ${job.isActive}")    // true
    println("Job is completed: ${job.isCompleted}") // false
    
    delay(1500)  // รอ 1.5 วินาที (ทำงานได้ 3 รอบ)
    
    job.cancel()  // ยกเลิก
    job.join()    // รอจนยกเลิกสมบูรณ์
    
    println("Job is cancelled: ${job.isCancelled}") // true
    println("Job is active: ${job.isActive}")        // false
}
```

### 6.2 Cancellation และ isActive

```kotlin
import kotlinx.coroutines.*

suspend fun longRunningTask(): String = coroutineScope {
    var progress = 0
    while (isActive && progress < 100) {  // ตรวจสอบ isActive เสมอ!
        delay(100)
        progress += 10
        println("Progress: $progress%")
    }
    if (!isActive) "Cancelled at $progress%" else "Completed"
}

fun main() = runBlocking {
    val job = launch {
        val result = longRunningTask()
        println("Result: $result")
    }
    
    delay(350)
    job.cancel()
    job.join()
    println("Done")
}
// Output:
// Progress: 10%
// Progress: 20%
// Progress: 30%
// Result: Cancelled at 30%
// Done
```

### 6.3 CancellationException

```kotlin
import kotlinx.coroutines.*

suspend fun fetchData(): String {
    try {
        delay(2000)  // simulate long operation
        return "data"
    } catch (e: CancellationException) {
        println("fetchData cancelled, cleaning up...")
        throw e  // ต้อง rethrow CancellationException เสมอ!
    }
}

fun main() = runBlocking {
    val job = launch {
        try {
            val data = fetchData()
            println("Got: $data")
        } catch (e: CancellationException) {
            println("Job cancelled")
        } finally {
            println("Cleanup resources")  // always runs
        }
    }
    
    delay(500)
    job.cancel(CancellationException("User cancelled"))
    job.join()
    println("Main done")
}
```

### 6.4 withTimeout

```kotlin
import kotlinx.coroutines.*

suspend fun slowApiCall(): String {
    delay(3000)
    return "result"
}

fun main() = runBlocking {
    // withTimeout - throw TimeoutCancellationException ถ้าเกินเวลา
    try {
        val result = withTimeout(1000) {
            slowApiCall()
        }
        println(result)
    } catch (e: TimeoutCancellationException) {
        println("Request timed out!")
    }
    
    // withTimeoutOrNull - คืน null แทนที่จะ throw
    val result = withTimeoutOrNull(1000) {
        slowApiCall()
    }
    println(result ?: "Timeout - got null")
    
    // Practical: retry with timeout
    suspend fun fetchWithRetry(maxRetries: Int): String? {
        repeat(maxRetries) { attempt ->
            val result = withTimeoutOrNull(500) {
                slowApiCall()  // จะ timeout ทุกครั้ง
            }
            if (result != null) return result
            println("Attempt ${attempt + 1} timed out, retrying...")
        }
        return null
    }
    
    val data = fetchWithRetry(3)
    println("Final result: $data")
}
```

---

## 🔄 7. runBlocking และ withContext

### 7.1 runBlocking ใช้งานจริง

```kotlin
import kotlinx.coroutines.*

// ใช้ใน main function
fun main() = runBlocking {
    println("Start")
    
    val result = async { computeAnswer() }
    
    println("Waiting...")
    println("Answer: ${result.await()}")
}

suspend fun computeAnswer(): Int {
    delay(1000)
    return 42
}
```

### 7.2 withContext - เปลี่ยน context

```kotlin
import kotlinx.coroutines.*

class UserRepository {
    // Database operations บน IO thread
    suspend fun findById(id: Int): User = withContext(Dispatchers.IO) {
        // simulate DB query
        delay(100)
        User(id, "User_$id", "user$id@example.com")
    }
    
    suspend fun findAll(): List<User> = withContext(Dispatchers.IO) {
        delay(200)
        (1..5).map { User(it, "User_$it", "user$it@example.com") }
    }
    
    suspend fun save(user: User): User = withContext(Dispatchers.IO) {
        delay(150)
        user.copy(id = user.id)
    }
}

data class User(val id: Int, val name: String, val email: String)

class UserService(private val repo: UserRepository) {
    // Business logic อาจทำบน Default dispatcher
    suspend fun getActiveUsers(): List<User> {
        val users = repo.findAll()
        return withContext(Dispatchers.Default) {
            users.filter { it.name.isNotEmpty() }
                 .sortedBy { it.name }
        }
    }
}

fun main() = runBlocking {
    val repo = UserRepository()
    val service = UserService(repo)
    
    val users = service.getActiveUsers()
    users.forEach { println("${it.id}: ${it.name} - ${it.email}") }
}
```

---

## 📡 8. ตัวอย่าง: Parallel API Calls

```kotlin
import kotlinx.coroutines.*
import kotlin.system.measureTimeMillis

// Simulated API clients
object ApiClient {
    suspend fun getUser(userId: Int): User {
        delay(300)  // simulate network latency
        return User(userId, "User $userId", "$userId@example.com")
    }
    
    suspend fun getUserPosts(userId: Int): List<Post> {
        delay(400)
        return (1..3).map { Post(it, userId, "Post $it by user $userId") }
    }
    
    suspend fun getUserFollowers(userId: Int): List<Int> {
        delay(200)
        return listOf(userId + 1, userId + 2, userId + 3)
    }
    
    suspend fun getWeather(city: String): Weather {
        delay(500)
        return Weather(city, 28.5, "Sunny")
    }
    
    suspend fun getExchangeRate(from: String, to: String): Double {
        delay(300)
        return when ("$from-$to") {
            "USD-THB" -> 35.5
            "EUR-THB" -> 38.2
            else -> 1.0
        }
    }
}

data class Post(val id: Int, val userId: Int, val content: String)
data class Weather(val city: String, val temp: Double, val condition: String)
data class UserProfile(
    val user: User,
    val posts: List<Post>,
    val followerCount: Int
)

// ดึงข้อมูล Profile แบบ parallel
suspend fun getUserProfile(userId: Int): UserProfile = coroutineScope {
    val userDeferred = async { ApiClient.getUser(userId) }
    val postsDeferred = async { ApiClient.getUserPosts(userId) }
    val followersDeferred = async { ApiClient.getUserFollowers(userId) }
    
    UserProfile(
        user = userDeferred.await(),
        posts = postsDeferred.await(),
        followerCount = followersDeferred.await().size
    )
}

// ดึงหลาย profiles พร้อมกัน
suspend fun getMultipleProfiles(userIds: List<Int>): List<UserProfile> = coroutineScope {
    userIds.map { id ->
        async { getUserProfile(id) }
    }.awaitAll()
}

// Dashboard data - หลาย API พร้อมกัน
suspend fun getDashboardData(userId: Int, city: String): Map<String, Any> = coroutineScope {
    val profileDeferred = async { getUserProfile(userId) }
    val weatherDeferred = async { ApiClient.getWeather(city) }
    val usdRateDeferred = async { ApiClient.getExchangeRate("USD", "THB") }
    val eurRateDeferred = async { ApiClient.getExchangeRate("EUR", "THB") }
    
    mapOf(
        "profile" to profileDeferred.await(),
        "weather" to weatherDeferred.await(),
        "rates" to mapOf(
            "USD/THB" to usdRateDeferred.await(),
            "EUR/THB" to eurRateDeferred.await()
        )
    )
}

fun main() = runBlocking {
    println("=== User Profile (Parallel Fetch) ===")
    val time1 = measureTimeMillis {
        val profile = getUserProfile(1)
        println("User: ${profile.user.name}")
        println("Posts: ${profile.posts.size}")
        println("Followers: ${profile.followerCount}")
    }
    println("Time: ${time1}ms (expected ~400ms, not 900ms)")
    
    println("\n=== Multiple Profiles ===")
    val time2 = measureTimeMillis {
        val profiles = getMultipleProfiles(listOf(1, 2, 3))
        profiles.forEach { p ->
            println("${p.user.name}: ${p.posts.size} posts, ${p.followerCount} followers")
        }
    }
    println("Time: ${time2}ms (3 profiles in parallel)")
    
    println("\n=== Dashboard Data ===")
    val time3 = measureTimeMillis {
        val dashboard = getDashboardData(1, "Bangkok")
        val profile = dashboard["profile"] as UserProfile
        val weather = dashboard["weather"] as Weather
        val rates = dashboard["rates"] as Map<*, *>
        
        println("User: ${profile.user.name}")
        println("Weather in ${weather.city}: ${weather.temp}°C, ${weather.condition}")
        println("Rates: $rates")
    }
    println("Time: ${time3}ms (fetched 4 different APIs in parallel)")
}
```

---

## 📥 9. ตัวอย่าง: File Downloads

```kotlin
import kotlinx.coroutines.*
import kotlin.random.Random

data class DownloadResult(
    val url: String,
    val sizeKB: Int,
    val timeMs: Long,
    val success: Boolean,
    val error: String? = null
)

object Downloader {
    suspend fun download(url: String): DownloadResult {
        val startTime = System.currentTimeMillis()
        
        return try {
            // simulate download (random size 100-5000 KB)
            val sizeKB = Random.nextInt(100, 5000)
            val downloadTimeMs = (sizeKB / 10).toLong()  // 10 KB/ms
            
            delay(downloadTimeMs)
            
            // simulate occasional failures
            if (Random.nextFloat() < 0.2) {
                throw RuntimeException("Connection timeout for $url")
            }
            
            DownloadResult(
                url = url,
                sizeKB = sizeKB,
                timeMs = System.currentTimeMillis() - startTime,
                success = true
            )
        } catch (e: CancellationException) {
            throw e  // rethrow cancellation!
        } catch (e: Exception) {
            DownloadResult(
                url = url,
                sizeKB = 0,
                timeMs = System.currentTimeMillis() - startTime,
                success = false,
                error = e.message
            )
        }
    }
}

class DownloadManager(private val maxConcurrent: Int = 3) {
    
    // Download หลายไฟล์พร้อมกัน แต่จำกัดจำนวน concurrent
    suspend fun downloadAll(urls: List<String>): List<DownloadResult> = coroutineScope {
        val semaphore = kotlinx.coroutines.sync.Semaphore(maxConcurrent)
        
        urls.map { url ->
            async {
                semaphore.withPermit {
                    println("Starting: $url")
                    val result = Downloader.download(url)
                    val status = if (result.success) "✓" else "✗"
                    println("$status Finished: $url (${result.sizeKB}KB in ${result.timeMs}ms)")
                    result
                }
            }
        }.awaitAll()
    }
    
    // Download พร้อม progress
    suspend fun downloadWithProgress(
        urls: List<String>,
        onProgress: (completed: Int, total: Int) -> Unit
    ): List<DownloadResult> = coroutineScope {
        val total = urls.size
        var completed = 0
        val mutex = kotlinx.coroutines.sync.Mutex()
        
        urls.map { url ->
            async {
                val result = Downloader.download(url)
                mutex.withLock {
                    completed++
                    onProgress(completed, total)
                }
                result
            }
        }.awaitAll()
    }
}

fun main() = runBlocking {
    val urls = listOf(
        "https://example.com/file1.zip",
        "https://example.com/file2.pdf",
        "https://example.com/file3.mp4",
        "https://example.com/file4.jpg",
        "https://example.com/file5.doc",
        "https://example.com/file6.png"
    )
    
    val manager = DownloadManager(maxConcurrent = 3)
    
    println("=== Downloading ${urls.size} files (max 3 concurrent) ===")
    val startTime = System.currentTimeMillis()
    
    val results = manager.downloadWithProgress(urls) { completed, total ->
        val percentage = (completed.toFloat() / total * 100).toInt()
        println("Progress: $completed/$total ($percentage%)")
    }
    
    val elapsed = System.currentTimeMillis() - startTime
    
    println("\n=== Summary ===")
    val successful = results.filter { it.success }
    val failed = results.filter { !it.success }
    
    println("Successful: ${successful.size}")
    println("Failed: ${failed.size}")
    println("Total downloaded: ${successful.sumOf { it.sizeKB }} KB")
    println("Total time: ${elapsed}ms")
    
    if (failed.isNotEmpty()) {
        println("\nFailed downloads:")
        failed.forEach { println("  ${it.url}: ${it.error}") }
    }
    
    // Retry failed downloads
    if (failed.isNotEmpty()) {
        println("\n=== Retrying failed downloads ===")
        val retryResults = manager.downloadAll(failed.map { it.url })
        val retrySuccess = retryResults.count { it.success }
        println("Retry successful: $retrySuccess/${failed.size}")
    }
}
```

---

## 🔀 10. Structured Concurrency

```kotlin
import kotlinx.coroutines.*

// Structured Concurrency: parent รอ children เสมอ
// ถ้า child fail -> parent และ siblings ถูก cancel

suspend fun riskyOperation(id: Int): String {
    delay(100L * id)
    if (id == 3) throw RuntimeException("Operation 3 failed!")
    return "Result $id"
}

fun main() = runBlocking {
    // ตัวอย่าง: ถ้า child ใดใด fail ทั้ง coroutineScope ยกเลิก
    try {
        coroutineScope {
            val results = (1..5).map { id ->
                async { riskyOperation(id) }
            }
            results.awaitAll()
        }
    } catch (e: RuntimeException) {
        println("One failed: ${e.message}")
    }
    
    // supervisorScope: sibling ไม่ถูก cancel เมื่อ child อื่น fail
    supervisorScope {
        val results = (1..5).map { id ->
            async {
                try {
                    riskyOperation(id)
                } catch (e: Exception) {
                    "Failed: ${e.message}"
                }
            }
        }
        results.awaitAll().forEach { println(it) }
    }
}
```

---

## 🏋️ แบบฝึกหัด

### ระดับ 1 (ง่าย)

1. เขียน suspend function `calculateAsync(a: Int, b: Int, op: String): Int` ที่:
   - delay 500ms เพื่อ simulate processing
   - ทำการคำนวณตาม op: "+", "-", "*", "/"
   - เรียก 3 การคำนวณแบบ sequential แล้วแสดงเวลาที่ใช้
   - จากนั้น refactor เป็น parallel ด้วย async และเปรียบเทียบเวลา

2. สร้าง coroutine ที่ทำงาน 10 วินาที แต่มี timeout 3 วินาที โดยใช้ `withTimeoutOrNull`

### ระดับ 2 (กลาง)

3. เขียน `parallelMap` extension function:
   ```kotlin
   suspend fun <T, R> List<T>.parallelMap(
       dispatcher: CoroutineDispatcher = Dispatchers.Default,
       transform: suspend (T) -> R
   ): List<R>
   ```
   และทดสอบกับการ fetch ข้อมูล user 20 คนพร้อมกัน

4. สร้าง `RateLimiter` coroutine ที่อนุญาตให้ทำ operation ได้แค่ N ครั้งต่อวินาที

### ระดับ 3 (ท้าทาย)

5. สร้าง `RetryPolicy` ที่รองรับ:
   - maxRetries
   - exponential backoff (1s, 2s, 4s, ...)
   - เฉพาะ retry สำหรับ exception types ที่กำหนด
   - ยกเลิกถ้า coroutine ถูก cancel

6. สร้าง `DownloadQueue` ที่:
   - รับ URLs เพิ่มได้ตลอดเวลา
   - ทำงาน max N concurrent downloads
   - มี pause/resume
   - รายงาน progress แต่ละไฟล์

---

## 📊 สรุป

| Concept | Description | ใช้เมื่อ |
|---------|-------------|---------|
| `suspend` | ฟังก์ชันที่ suspend ได้ | ทุก async operation |
| `runBlocking` | บล็อก thread รอ coroutines | main(), tests |
| `launch` | fire and forget | ไม่ต้องการผลลัพธ์ |
| `async/await` | parallel with result | ต้องการผลลัพธ์ |
| `coroutineScope` | สร้าง scope ชั่วคราว | suspend functions |
| `withContext` | เปลี่ยน dispatcher | เปลี่ยน thread pool |
| `Job.cancel()` | ยกเลิก coroutine | interrupt long work |
| `withTimeout` | timeout | prevent hanging |

| Dispatcher | Thread Pool | ใช้สำหรับ |
|-----------|------------|---------|
| `Default` | CPU cores | คำนวณ, parsing |
| `IO` | Up to 64 | network, file, DB |
| `Main` | UI thread | UI updates |
| `Unconfined` | caller's thread | testing mostly |

---

## ➡️ ถัดไป: Part 15 - Exception Handling

---
*Part 14/100+ | Kotlin & Spring Boot Complete Course*
