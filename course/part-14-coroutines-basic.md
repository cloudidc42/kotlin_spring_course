# Part 14: Coroutines พื้นฐาน

## Coroutines คืออะไร?

Coroutines เป็นวิธีการทำ asynchronous programming ที่ทรงพลังของ Kotlin ช่วยให้เราเขียน code แบบ asynchronous ได้เหมือนเขียน synchronous code ธรรมดา โดยไม่ต้องใช้ callback หรือ complex thread management

### เปรียบเทียบกับ Thread

```
Thread:
- ทำงานบน OS thread (หนัก ~1MB per thread)
- Context switching ช้า
- จำนวนจำกัด (~1000 threads)
- Blocking ทำให้ thread ว่าง

Coroutine:
- ทำงานบน coroutine infrastructure (เบา ~1KB)
- Context switching เร็ว
- สามารถมีได้นับล้าน
- Suspending ไม่บล็อก thread
```

```kotlin
// Thread แบบดั้งเดิม
fun fetchDataWithThread(url: String): String {
    var result = ""
    val thread = Thread {
        // blocking call - thread ถูกบล็อกทั้งหมด
        result = URL(url).readText()
    }
    thread.start()
    thread.join()  // รอ thread
    return result
}

// Callback Hell แบบ Java/JavaScript
fun fetchWithCallback(url: String, callback: (String) -> Unit) {
    Thread {
        val data = URL(url).readText()
        callback(data)
    }.start()
}

fetchWithCallback("http://api1.com") { data1 ->
    // nested callback
    fetchWithCallback("http://api2.com?data=$data1") { data2 ->
        fetchWithCallback("http://api3.com?data=$data2") { data3 ->
            println(data3)  // Callback hell!
        }
    }
}

// Coroutines - อ่านง่ายเหมือน synchronous
suspend fun fetchData(url: String): String {
    return withContext(Dispatchers.IO) {
        URL(url).readText()
    }
}

suspend fun fetchChainedData() {
    val data1 = fetchData("http://api1.com")
    val data2 = fetchData("http://api2.com?data=$data1")
    val data3 = fetchData("http://api3.com?data=$data2")
    println(data3)  // Clean and sequential-looking!
}
```

---

## 14.1 Coroutines Setup

เพิ่ม dependencies ใน `build.gradle.kts`:

```kotlin
dependencies {
    // Kotlin Coroutines
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:1.7.3")
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-reactor:1.7.3")
    
    // สำหรับ Spring Boot
    implementation("org.springframework.boot:spring-boot-starter-webflux")
    
    // Testing
    testImplementation("org.jetbrains.kotlinx:kotlinx-coroutines-test:1.7.3")
}
```

---

## 14.2 suspend Functions

`suspend` function คือฟังก์ชันที่สามารถ suspend (หยุดชั่วคราว) การทำงานโดยไม่บล็อก thread

```kotlin
import kotlinx.coroutines.*

// suspend function พื้นฐาน
suspend fun doSomething(): String {
    delay(1000)  // suspend (ไม่ block thread)
    return "Done!"
}

// suspend function ต้องถูกเรียกใน coroutine หรือ suspend function อื่น
fun main() = runBlocking {  // สร้าง coroutine scope
    println("Before")
    val result = doSomething()  // เรียก suspend function
    println("Result: $result")
    println("After")
}
// Before
// (รอ 1 วินาที)
// Result: Done!
// After
```

### เปรียบเทียบ delay vs Thread.sleep

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    // delay - suspend โดยไม่บล็อก thread
    // Thread สามารถทำงานอื่นได้ระหว่างนี้
    val job1 = launch {
        println("Coroutine 1 start")
        delay(1000)
        println("Coroutine 1 end")
    }
    
    val job2 = launch {
        println("Coroutine 2 start")
        delay(500)
        println("Coroutine 2 end")
    }
    
    joinAll(job1, job2)
}
// Coroutine 1 start
// Coroutine 2 start
// (500ms)
// Coroutine 2 end
// (500ms)
// Coroutine 1 end

// Thread.sleep - บล็อก thread ทั้งหมด
fun withThreadSleep() = runBlocking {
    val job1 = launch {
        println("Thread 1 start")
        Thread.sleep(1000)  // บล็อก thread!
        println("Thread 1 end")
    }
    
    // job2 จะไม่ทำงานจนกว่า job1 จะเสร็จ (ถ้าใช้ single thread)
    val job2 = launch {
        println("Thread 2 start")
        Thread.sleep(500)
        println("Thread 2 end")
    }
}
```

---

## 14.3 launch และ async

### launch - Fire and Forget

`launch` ใช้สำหรับ start coroutine ที่ไม่ต้องการ return value

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    println("Start: ${Thread.currentThread().name}")
    
    // launch - return Job (ไม่มี value)
    val job = launch {
        delay(1000)
        println("Launched coroutine done: ${Thread.currentThread().name}")
    }
    
    println("After launch (before join)")
    job.join()  // รอให้ coroutine เสร็จ
    println("End")
}

// launch หลายๆ อัน
fun main2() = runBlocking {
    val startTime = System.currentTimeMillis()
    
    val jobs = (1..5).map { i ->
        launch {
            delay(1000)
            println("Job $i done")
        }
    }
    
    jobs.joinAll()
    
    val elapsed = System.currentTimeMillis() - startTime
    println("All jobs done in ${elapsed}ms")  // ประมาณ 1000ms (parallel!)
}
```

### async - Deferred Value

`async` ใช้สำหรับ coroutine ที่ต้องการ return value

```kotlin
import kotlinx.coroutines.*

suspend fun fetchUserName(id: Int): String {
    delay(1000)  // simulate network call
    return "User$id"
}

suspend fun fetchUserAge(id: Int): Int {
    delay(800)   // simulate network call
    return 20 + id
}

fun main() = runBlocking {
    // Sequential (ช้า)
    val startSeq = System.currentTimeMillis()
    val name1 = fetchUserName(1)  // รอ 1000ms
    val age1 = fetchUserAge(1)    // รอ 800ms
    println("Sequential: ${name1}, ${age1} in ${System.currentTimeMillis() - startSeq}ms")
    // Sequential: User1, 21 in 1800ms
    
    // Parallel ด้วย async (เร็ว)
    val startPar = System.currentTimeMillis()
    val nameDeferred = async { fetchUserName(2) }  // เริ่มทันที
    val ageDeferred = async { fetchUserAge(2) }    // เริ่มทันที
    
    val name2 = nameDeferred.await()  // รอผลลัพธ์
    val age2 = ageDeferred.await()    // รอผลลัพธ์ (อาจเสร็จแล้ว)
    println("Parallel: ${name2}, ${age2} in ${System.currentTimeMillis() - startPar}ms")
    // Parallel: User2, 22 in 1000ms (เร็วกว่า!)
}
```

### async/await pattern ขั้นสูง

```kotlin
import kotlinx.coroutines.*

data class UserProfile(val name: String, val age: Int, val email: String)

suspend fun fetchName(id: Int): String { delay(500); return "Alice" }
suspend fun fetchAge(id: Int): Int { delay(300); return 25 }
suspend fun fetchEmail(id: Int): String { delay(400); return "alice@example.com" }

fun main() = runBlocking {
    val id = 1
    val start = System.currentTimeMillis()
    
    // Parallel fetch ทุก field
    val (name, age, email) = Triple(
        async { fetchName(id) },
        async { fetchAge(id) },
        async { fetchEmail(id) }
    ).let { (n, a, e) -> Triple(n.await(), a.await(), e.await()) }
    
    val profile = UserProfile(name, age, email)
    println("Profile: $profile in ${System.currentTimeMillis() - start}ms")
    // Profile: UserProfile(name=Alice, age=25, email=alice@example.com) in ~500ms
    
    // ใช้ coroutineScope สำหรับ structured concurrency
    val profile2 = coroutineScope {
        val nameD = async { fetchName(id) }
        val ageD = async { fetchAge(id) }
        val emailD = async { fetchEmail(id) }
        UserProfile(nameD.await(), ageD.await(), emailD.await())
    }
    println("Profile2: $profile2")
}
```

---

## 14.4 Coroutine Scope

Scope กำหนด lifecycle ของ coroutine

```kotlin
import kotlinx.coroutines.*

// runBlocking - สร้าง blocking coroutine (ใช้ใน main/test)
fun main() = runBlocking {
    println("runBlocking scope")
    delay(100)
}

// coroutineScope - suspend function ที่รอลูก coroutines ทั้งหมด
suspend fun processAll() = coroutineScope {
    val job1 = launch { delay(1000); println("Job 1") }
    val job2 = launch { delay(500); println("Job 2") }
    // coroutineScope จะ return เมื่อ job1 และ job2 เสร็จ
}

// GlobalScope - ใช้ใน application lifetime (ระวัง! อาจ leak)
val globalJob = GlobalScope.launch {
    delay(1000)
    println("Global scope")
}

// Custom CoroutineScope
class DataLoader {
    private val scope = CoroutineScope(Dispatchers.IO + SupervisorJob())
    
    fun loadData() {
        scope.launch {
            println("Loading data...")
            delay(1000)
            println("Data loaded!")
        }
    }
    
    fun cleanup() {
        scope.cancel()  // ยกเลิก coroutines ทั้งหมดใน scope
    }
}

// CoroutineScope lifecycle ที่ถูกต้อง
class UserViewModel : CoroutineScope {
    private val job = Job()
    override val coroutineContext = Dispatchers.Main + job
    
    fun loadUser(id: Int) {
        launch {
            try {
                val user = fetchUser(id)
                // update UI
            } catch (e: Exception) {
                // handle error
            }
        }
    }
    
    fun onDestroy() {
        job.cancel()  // ยกเลิกเมื่อ ViewModel ถูก destroy
    }
    
    private suspend fun fetchUser(id: Int): String {
        delay(500)
        return "User $id"
    }
}
```

### Structured Concurrency

```kotlin
import kotlinx.coroutines.*

// Structured Concurrency หมายถึง child coroutines อยู่ใน scope ของ parent
fun main() = runBlocking {
    coroutineScope {
        launch {
            delay(1000)
            println("Child 1 done")
        }
        launch {
            delay(500)
            println("Child 2 done")
        }
        // coroutineScope รอ children ทั้งหมด
    }
    println("All children done")
}

// ถ้า child throw exception, parent จะ cancel ด้วย
fun main2() = runBlocking {
    try {
        coroutineScope {
            launch {
                delay(500)
                throw RuntimeException("Child failed!")
            }
            launch {
                delay(1000)
                println("This won't print")
            }
        }
    } catch (e: Exception) {
        println("Caught: ${e.message}")  // Caught: Child failed!
    }
}
```

---

## 14.5 Job และ Deferred

### Job

`Job` แทน coroutine ที่ไม่มี return value

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    val job = launch {
        println("Job started")
        delay(2000)
        println("Job completed")
    }
    
    println("Job state: ${job.isActive}")    // true
    println("Job state: ${job.isCompleted}") // false
    println("Job state: ${job.isCancelled}") // false
    
    delay(500)
    job.cancel()  // ยกเลิก job
    
    println("After cancel: ${job.isCancelled}")  // true
    
    job.join()  // รอให้ job เสร็จสิ้น (แม้จะ cancelled)
    println("Final state: ${job.isCompleted}")   // true
}
```

### Job Hierarchy

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    val parent = Job()
    
    val child1 = launch(parent) {
        delay(2000)
        println("Child 1 done")
    }
    
    val child2 = launch(parent) {
        delay(1000)
        println("Child 2 done")
    }
    
    delay(500)
    parent.cancel()  // ยกเลิก parent ทำให้ children ถูกยกเลิกด้วย
    
    println("Parent: ${parent.isCancelled}")  // true
    println("Child1: ${child1.isCancelled}")  // true
    println("Child2: ${child2.isCancelled}")  // true
}
```

### Deferred

`Deferred` แทน coroutine ที่มี return value (result ของ `async`)

```kotlin
import kotlinx.coroutines.*

suspend fun computeValue(): Int {
    delay(1000)
    return 42
}

fun main() = runBlocking {
    val deferred: Deferred<Int> = async {
        computeValue()
    }
    
    println("Deferred created, doing other work...")
    delay(500)  // ทำงานอื่นระหว่างรอ
    
    // await() รอผลลัพธ์
    val result = deferred.await()
    println("Result: $result")  // Result: 42
    
    // Deferred states
    println("Completed: ${deferred.isCompleted}")  // true
    println("Value: ${deferred.getCompleted()}")   // 42 (ไม่ต้อง await ถ้า completed แล้ว)
}
```

---

## 14.6 Dispatchers

Dispatchers กำหนดว่า coroutine จะทำงานบน thread ไหน

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    // Dispatchers.Default - สำหรับ CPU-intensive work
    // จำนวน threads = CPU cores (min 2)
    launch(Dispatchers.Default) {
        println("Default: ${Thread.currentThread().name}")
        // ทำงาน computation-heavy
        val sum = (1..1_000_000).sum()
        println("Sum: $sum")
    }
    
    // Dispatchers.IO - สำหรับ I/O operations
    // thread pool ใหญ่กว่า (64 threads หรือมากกว่า)
    launch(Dispatchers.IO) {
        println("IO: ${Thread.currentThread().name}")
        // อ่านไฟล์, network calls, database
        delay(100)  // simulate I/O
    }
    
    // Dispatchers.Main - สำหรับ UI thread (Android/JavaFX)
    // launch(Dispatchers.Main) { ... }
    
    // Dispatchers.Unconfined - ทำงาน thread ไหนก็ได้ (ระวัง!)
    launch(Dispatchers.Unconfined) {
        println("Unconfined start: ${Thread.currentThread().name}")
        delay(100)
        println("Unconfined after delay: ${Thread.currentThread().name}")
        // อาจเปลี่ยน thread หลัง delay
    }
}
```

### withContext - เปลี่ยน Dispatcher

```kotlin
import kotlinx.coroutines.*
import java.io.File

suspend fun readFileContent(path: String): String {
    // เปลี่ยนไปทำงานบน IO thread
    return withContext(Dispatchers.IO) {
        File(path).readText()
    }
}

suspend fun processData(data: String): String {
    // เปลี่ยนไปทำงานบน CPU thread
    return withContext(Dispatchers.Default) {
        data.uppercase().reversed()
    }
}

// Spring Boot pattern: suspend function ใน service
// @Service
class FileProcessingService {
    suspend fun processFile(path: String): String {
        val content = withContext(Dispatchers.IO) {
            // I/O operation
            "file content"  // simulate
        }
        
        val processed = withContext(Dispatchers.Default) {
            // CPU-intensive processing
            content.uppercase()
        }
        
        return processed
    }
}

fun main() = runBlocking {
    val service = FileProcessingService()
    println(service.processFile("/tmp/test.txt"))
}
```

### Custom Dispatcher

```kotlin
import kotlinx.coroutines.*
import java.util.concurrent.Executors

// สร้าง custom thread pool
val customDispatcher = Executors.newFixedThreadPool(4).asCoroutineDispatcher()

fun main() = runBlocking {
    repeat(8) { i ->
        launch(customDispatcher) {
            println("Task $i on: ${Thread.currentThread().name}")
            delay(100)
        }
    }
    // customDispatcher.close()  // ต้อง close เมื่อไม่ใช้แล้ว
}
```

---

## 14.7 Coroutine Context

Context ประกอบด้วย elements ต่างๆ ที่กำหนดพฤติกรรมของ coroutine

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    // Coroutine context = Job + Dispatcher + CoroutineName + ...
    launch(Dispatchers.IO + CoroutineName("MyCoroutine")) {
        println("Name: ${coroutineContext[CoroutineName]?.name}")
        println("Dispatcher: ${coroutineContext[ContinuationInterceptor]}")
    }
    
    // ดู context ปัจจุบัน
    val context = coroutineContext
    println("Job: ${context[Job]}")
    
    // สืบทอด context
    val parentContext = Dispatchers.IO + CoroutineName("Parent")
    launch(parentContext) {
        println("Parent: ${coroutineContext[CoroutineName]?.name}")
        
        // Child สืบทอด context จาก parent
        launch {
            println("Child: ${coroutineContext[CoroutineName]?.name}")
        }
        
        // Child เปลี่ยน dispatcher แต่สืบทอด name
        launch(Dispatchers.Default) {
            println("Child2 name: ${coroutineContext[CoroutineName]?.name}")
        }
    }
}
```

### CoroutineExceptionHandler

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    val handler = CoroutineExceptionHandler { context, exception ->
        println("Caught in handler: ${exception.message}")
        println("Coroutine: ${context[CoroutineName]?.name}")
    }
    
    val scope = CoroutineScope(Dispatchers.Default + handler)
    
    val job = scope.launch(CoroutineName("ErrorJob")) {
        delay(100)
        throw RuntimeException("Something went wrong!")
    }
    
    job.join()
    delay(500)  // รอให้ handler ทำงาน
}
```

---

## 14.8 Cancellation

การยกเลิก coroutine เป็นเรื่องสำคัญมากในการจัดการ resources

```kotlin
import kotlinx.coroutines.*

// Coroutine ต้อง cooperative กับ cancellation
fun main() = runBlocking {
    val job = launch {
        repeat(1000) { i ->
            // isActive ตรวจสอบ cancellation
            if (!isActive) return@launch
            
            println("Working $i")
            delay(100)  // delay เป็น cancellation point
        }
    }
    
    delay(550)
    println("Cancelling...")
    job.cancel()
    job.join()
    println("Cancelled")
}
// Working 0
// Working 1
// ...
// Working 4
// Cancelling...
// Cancelled
```

### CancellationException

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    val job = launch {
        try {
            repeat(10) { i ->
                println("Work $i")
                delay(200)
            }
        } catch (e: CancellationException) {
            println("Caught CancellationException: ${e.message}")
            // ไม่ควร swallow CancellationException!
            throw e  // re-throw
        } finally {
            // finally block ทำงานแม้ cancel
            println("Cleanup in finally")
        }
    }
    
    delay(500)
    job.cancelAndJoin()
    println("Done")
}
```

### withTimeout

```kotlin
import kotlinx.coroutines.*

suspend fun slowOperation(): String {
    delay(2000)
    return "Result"
}

fun main() = runBlocking {
    // withTimeout - throw TimeoutCancellationException
    try {
        val result = withTimeout(1000) {
            slowOperation()
        }
        println(result)
    } catch (e: TimeoutCancellationException) {
        println("Timed out!")
    }
    
    // withTimeoutOrNull - return null แทน throw
    val result = withTimeoutOrNull(1000) {
        slowOperation()
    }
    println(result ?: "Timed out (null)")
}
```

---

## 14.9 Flow - Asynchronous Streams

Flow เป็น cold asynchronous stream ที่ emit values หลายๆ ค่า

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

// สร้าง Flow
fun simpleFlow(): Flow<Int> = flow {
    for (i in 1..5) {
        delay(100)
        emit(i)  // emit value
    }
}

fun main() = runBlocking {
    simpleFlow().collect { value ->
        println("Collected: $value")
    }
}

// Flow operators (คล้าย Collection operators)
fun main2() = runBlocking {
    (1..10).asFlow()
        .filter { it % 2 == 0 }           // เฉพาะเลขคู่
        .map { it * it }                   // ยกกำลัง 2
        .take(3)                           // เอาแค่ 3 ค่า
        .collect { println("Value: $it") }
    // Value: 4
    // Value: 16
    // Value: 36
}
```

---

## 14.10 ตัวอย่าง: Async API Calls

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

// Data models
data class Post(val id: Int, val title: String, val userId: Int)
data class User(val id: Int, val name: String, val email: String)
data class Comment(val postId: Int, val id: Int, val body: String)

// API Service (simulate)
object ApiService {
    suspend fun getUser(id: Int): User {
        delay(500)  // simulate network
        return User(id, "User $id", "user$id@example.com")
    }
    
    suspend fun getPosts(userId: Int): List<Post> {
        delay(300)
        return (1..3).map { Post(userId * 10 + it, "Post $it of user $userId", userId) }
    }
    
    suspend fun getComments(postId: Int): List<Comment> {
        delay(200)
        return (1..2).map { Comment(postId, postId * 10 + it, "Comment $it on post $postId") }
    }
}

// Sequential API calls (ช้า)
suspend fun loadUserDataSequential(userId: Int) {
    val start = System.currentTimeMillis()
    
    val user = ApiService.getUser(userId)  // 500ms
    val posts = ApiService.getPosts(userId)  // 300ms
    
    // load comments for each post sequentially
    val allComments = posts.map { post ->
        ApiService.getComments(post.id)  // 200ms * 3 = 600ms
    }
    
    val elapsed = System.currentTimeMillis() - start
    println("Sequential: ${user.name}, ${posts.size} posts, loaded in ${elapsed}ms")
    // ~1400ms
}

// Parallel API calls (เร็ว)
suspend fun loadUserDataParallel(userId: Int) {
    val start = System.currentTimeMillis()
    
    coroutineScope {
        val userDeferred = async { ApiService.getUser(userId) }
        val postsDeferred = async { ApiService.getPosts(userId) }
        
        val user = userDeferred.await()
        val posts = postsDeferred.await()
        
        // load comments ทุก post พร้อมกัน
        val allComments = posts.map { post ->
            async { ApiService.getComments(post.id) }
        }.awaitAll()
        
        val elapsed = System.currentTimeMillis() - start
        println("Parallel: ${user.name}, ${posts.size} posts, ${allComments.flatten().size} comments, loaded in ${elapsed}ms")
        // ~500ms (bottleneck คือ getUser)
    }
}

fun main() = runBlocking {
    println("Loading user data...")
    
    loadUserDataSequential(1)
    loadUserDataParallel(1)
}
```

---

## 14.11 ตัวอย่าง: Parallel Processing

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

// Processing items in parallel with limited concurrency
suspend fun processItem(item: Int): Int {
    delay((100..500).random().toLong())  // simulate variable processing time
    return item * item
}

suspend fun processAllItems(items: List<Int>, parallelism: Int = 4): List<Int> {
    return items
        .chunked(parallelism)
        .flatMap { chunk ->
            coroutineScope {
                chunk.map { item ->
                    async { processItem(item) }
                }.awaitAll()
            }
        }
}

// Producer-Consumer pattern
fun producer(channel: kotlinx.coroutines.channels.Channel<Int>) = CoroutineScope(Dispatchers.Default).launch {
    for (i in 1..10) {
        println("Producing: $i")
        channel.send(i)
        delay(100)
    }
    channel.close()
}

suspend fun processInBatches(
    items: List<Int>,
    batchSize: Int,
    process: suspend (List<Int>) -> List<Int>
): List<Int> {
    return items.chunked(batchSize).flatMap { batch ->
        process(batch)
    }
}

// Fan-out: หนึ่ง producer หลาย consumers
suspend fun fanOut() = coroutineScope {
    val channel = kotlinx.coroutines.channels.Channel<Int>(capacity = 10)
    
    // Producer
    launch {
        for (i in 1..20) {
            channel.send(i)
        }
        channel.close()
    }
    
    // Multiple consumers
    val consumers = (1..4).map { consumerId ->
        launch {
            for (item in channel) {
                delay(100)
                println("Consumer $consumerId processed: $item")
            }
        }
    }
    
    consumers.joinAll()
}

fun main() = runBlocking {
    val items = (1..20).toList()
    
    val start = System.currentTimeMillis()
    val results = processAllItems(items, parallelism = 4)
    println("Processed in ${System.currentTimeMillis() - start}ms")
    println("Results: $results")
}
```

---

## 14.12 Coroutines ใน Spring Boot

```kotlin
// build.gradle.kts
// implementation("org.jetbrains.kotlinx:kotlinx-coroutines-reactor:1.7.3")

import kotlinx.coroutines.*
import org.springframework.web.bind.annotation.*
import org.springframework.stereotype.Service

// Repository (non-blocking)
@Repository
interface UserRepository : CoroutineCrudRepository<User, Long>

// Service ที่ใช้ coroutines
@Service
class UserService(
    private val userRepository: UserRepository,
    private val externalApiClient: ExternalApiClient
) {
    // suspend function ใน service
    suspend fun getUserWithProfile(id: Long): UserWithProfile {
        return coroutineScope {
            val userDeferred = async { userRepository.findById(id) }
            val profileDeferred = async { externalApiClient.fetchProfile(id) }
            
            val user = userDeferred.await() 
                ?: throw NotFoundException("User $id not found")
            val profile = profileDeferred.await()
            
            UserWithProfile(user, profile)
        }
    }
    
    // Flow สำหรับ streaming
    fun getAllUsersFlow(): Flow<User> = userRepository.findAll()
    
    // Batch processing
    suspend fun processAllUsers(processor: suspend (User) -> Unit) {
        userRepository.findAll()
            .collect { user ->
                withContext(Dispatchers.Default) {
                    processor(user)
                }
            }
    }
}

// Controller ที่ใช้ coroutines (Spring WebFlux)
@RestController
@RequestMapping("/api/users")
class UserController(private val userService: UserService) {
    
    // suspend function เป็น endpoint ได้เลย
    @GetMapping("/{id}/profile")
    suspend fun getUserProfile(@PathVariable id: Long): UserWithProfile {
        return userService.getUserWithProfile(id)
    }
    
    // Flow สำหรับ streaming response
    @GetMapping("/stream")
    fun streamUsers(): Flow<User> = userService.getAllUsersFlow()
    
    // Parallel requests
    @GetMapping("/batch")
    suspend fun getUserBatch(@RequestParam ids: List<Long>): List<User> {
        return coroutineScope {
            ids.map { id -> async { userService.getUserById(id) } }.awaitAll()
        }
    }
}
```

---

## สรุปบทที่ 14

| Concept | ใช้งาน |
|---------|--------|
| `suspend` | ฟังก์ชันที่สามารถ suspend ได้ |
| `launch` | Start coroutine ไม่ต้องการ return value |
| `async/await` | Start coroutine ที่ return value |
| `coroutineScope` | สร้าง scope รอ children ทั้งหมด |
| `Dispatchers.IO` | สำหรับ I/O operations |
| `Dispatchers.Default` | สำหรับ CPU-heavy work |
| `withContext` | เปลี่ยน dispatcher |
| `Job` | Control coroutine lifecycle |
| `delay` | Suspend โดยไม่ block thread |
| `Flow` | Async streams ของ values |

**หลักการสำคัญ:**
- ใช้ structured concurrency เสมอ
- ระวัง GlobalScope
- `CancellationException` ต้อง re-throw
- ใช้ `coroutineScope {}` สำหรับ parallel operations
- Spring Boot + Coroutines = reactive without reactive complexity

---

## แบบฝึกหัดบทที่ 14

### ระดับง่าย

1. เขียน suspend function `downloadFile(url: String): ByteArray` ที่ simulate การ download ไฟล์ (delay 2 วินาที) จากนั้น download 3 ไฟล์พร้อมกันด้วย `async`

2. สร้าง `CountdownTimer` ที่ใช้ coroutine นับถอยหลังและ print ทุกวินาที สามารถ cancel ได้

3. เขียนฟังก์ชัน `retryWithDelay` ที่ retry coroutine สูงสุด n ครั้ง โดยรอเวลาระหว่าง retry

### ระดับกลาง

4. สร้าง `RateLimiter` ที่จำกัดจำนวน requests ต่อวินาที:
   ```kotlin
   val limiter = RateLimiter(maxPerSecond = 10)
   limiter.throttle { /* protected code */ }
   ```

5. Implement `Cache<K, V>` ที่ใช้ coroutines สำหรับ concurrent access:
   - `get(key)`: suspend เพื่อรอถ้ากำลัง loading
   - `load(key, loader)`: โหลดและเก็บ cache
   - Thread-safe โดยใช้ `Mutex` หรือ `Channel`

6. สร้าง pipeline ที่ process items แบบ parallel:
   ```kotlin
   val pipeline = Pipeline<String, Int>(
       parser = { it.toInt() },
       processor = { it * 2 },
       saver = { println(it) },
       parallelism = 4
   )
   pipeline.process(listOf("1", "2", "3", "4", "5"))
   ```

### ระดับยาก

7. Implement `BoundedConcurrency` ที่จำกัดจำนวน concurrent coroutines:
   ```kotlin
   val semaphore = BoundedConcurrency(maxConcurrent = 5)
   val results = items.map { item ->
       async { semaphore.withPermit { processItem(item) } }
   }.awaitAll()
   ```

8. สร้าง event-driven system ด้วย `Channel` และ `Flow`:
   - EventBus ที่ publish/subscribe events
   - Coroutine-safe
   - Support multiple subscribers
   - Backpressure handling

---

[ไปต่อ Part 15: Exception Handling →](part-15-exception-handling.md)

---

*Part 14/100+ | Kotlin & Spring Boot Complete Course*
