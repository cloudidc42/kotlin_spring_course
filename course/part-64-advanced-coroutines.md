# Part 64: Advanced Coroutines - Real-time Data Pipeline

## บทนำ

ใน Part นี้เราจะศึกษา Kotlin Coroutines ในระดับ Advanced โดยเน้นที่ **Flow** operators, **SharedFlow** vs **StateFlow**, **Channel** types, **Structured Concurrency** และการทดสอบ coroutines

## 1. Flow Operators ขั้นสูง

### transform - Custom Operator

```kotlin
import kotlinx.coroutines.flow.*

// transform ยืดหยุ่นกว่า map - สามารถ emit หลายค่าหรือไม่ emit เลยก็ได้
fun <T, R> Flow<T>.filterAndMap(
    predicate: (T) -> Boolean,
    transform: (T) -> R
): Flow<R> = this.transform { value ->
    if (predicate(value)) {
        emit(transform(value))
    }
}

// ตัวอย่างการใช้
suspend fun transformExample() {
    flowOf(1, 2, 3, 4, 5, 6)
        .filterAndMap(
            predicate = { it % 2 == 0 },
            transform = { it * it }
        )
        .collect { println(it) }
    // Output: 4, 16, 36
}

// flatMapConcat - แปลงแต่ละ element เป็น Flow และ concat ต่อกัน
suspend fun flatMapConcatExample() {
    flowOf(1, 2, 3)
        .flatMapConcat { num ->
            flow {
                emit("${num}a")
                emit("${num}b")
            }
        }
        .collect { print("$it ") }
    // Output: 1a 1b 2a 2b 3a 3b
}

// flatMapMerge - แปลงเป็น Flow และ merge พร้อมกัน (concurrency)
suspend fun flatMapMergeExample() {
    flowOf(1, 2, 3)
        .flatMapMerge(concurrency = 2) { num ->
            flow {
                delay(num * 100L)
                emit(num)
            }
        }
        .collect { print("$it ") }
    // Output: 1 2 3 (order depends on completion time)
}
```

### combine และ zip

```kotlin
// zip - รอทั้งคู่แล้วจับคู่ตาม order
suspend fun zipExample() {
    val names = flowOf("Alice", "Bob", "Charlie")
    val scores = flowOf(90, 85, 95)
    
    names.zip(scores) { name, score ->
        "$name: $score"
    }.collect { println(it) }
    // Output: Alice: 90, Bob: 85, Charlie: 95
}

// combine - เมื่อ flow ใดก็ตาม emit ค่าใหม่ จะ combine ค่าล่าสุดของทุก flow
suspend fun combineExample() {
    val temperature = MutableStateFlow(25.0)
    val humidity = MutableStateFlow(60.0)
    
    temperature.combine(humidity) { temp, hum ->
        "Temperature: ${temp}°C, Humidity: ${hum}%"
    }.collect { println(it) }
    
    // อัปเดตค่า
    temperature.value = 28.0  // trigger combine
    humidity.value = 65.0     // trigger combine
}

// merge - รวม flows หลาย flows เป็น flow เดียว
suspend fun mergeExample() {
    val hot = flow { 
        repeat(3) { 
            emit("hot-$it")
            delay(300) 
        } 
    }
    val cold = flow { 
        repeat(3) { 
            emit("cold-$it")
            delay(200) 
        } 
    }
    
    merge(hot, cold).collect { println(it) }
    // Output: interleaved based on timing
}
```

### Windowing Operators

```kotlin
// chunked - รวม N elements เป็น List
suspend fun chunkedExample() {
    (1..10).asFlow()
        .chunked(3)
        .collect { println(it) }
    // Output: [1,2,3], [4,5,6], [7,8,9], [10]
}

// Custom sliding window
fun <T> Flow<T>.slidingWindow(size: Int): Flow<List<T>> = flow {
    val buffer = ArrayDeque<T>()
    collect { element ->
        buffer.addLast(element)
        if (buffer.size > size) buffer.removeFirst()
        if (buffer.size == size) emit(buffer.toList())
    }
}

// Custom debounce + batch
fun <T> Flow<T>.batchWithTimeout(
    maxSize: Int,
    timeout: Long
): Flow<List<T>> = channelFlow {
    val buffer = mutableListOf<T>()
    var job: kotlinx.coroutines.Job? = null
    
    collect { element ->
        buffer.add(element)
        job?.cancel()
        
        if (buffer.size >= maxSize) {
            send(buffer.toList())
            buffer.clear()
        } else {
            job = kotlinx.coroutines.launch {
                delay(timeout)
                if (buffer.isNotEmpty()) {
                    send(buffer.toList())
                    buffer.clear()
                }
            }
        }
    }
    
    job?.join()
    if (buffer.isNotEmpty()) send(buffer.toList())
}
```

## 2. SharedFlow vs StateFlow

### StateFlow

```kotlin
import kotlinx.coroutines.flow.*

class WeatherViewModel {
    // StateFlow - always has a value, replays last value to new subscribers
    private val _temperature = MutableStateFlow(0.0)
    val temperature: StateFlow<Double> = _temperature.asStateFlow()
    
    private val _weatherData = MutableStateFlow<WeatherState>(WeatherState.Loading)
    val weatherData: StateFlow<WeatherState> = _weatherData.asStateFlow()

    suspend fun loadWeather(city: String) {
        _weatherData.value = WeatherState.Loading
        
        try {
            val data = fetchWeatherData(city)
            _weatherData.value = WeatherState.Success(data)
            _temperature.value = data.temperature
        } catch (e: Exception) {
            _weatherData.value = WeatherState.Error(e.message ?: "Unknown error")
        }
    }

    // StateFlow.update - thread-safe update
    fun incrementTemperature() {
        _temperature.update { current -> current + 1.0 }
    }
    
    // updateAndGet - get the new value
    fun adjustTemperature(delta: Double): Double {
        return _temperature.updateAndGet { current -> current + delta }
    }
}

sealed class WeatherState {
    object Loading : WeatherState()
    data class Success(val data: WeatherData) : WeatherState()
    data class Error(val message: String) : WeatherState()
}

data class WeatherData(val temperature: Double, val humidity: Double, val description: String)
```

### SharedFlow

```kotlin
class EventBus {
    // SharedFlow - กระจาย events ให้หลาย subscribers
    // replay = 0: ไม่ replay ให้ subscriber ใหม่
    // replay = N: replay N events ล่าสุด
    private val _events = MutableSharedFlow<AppEvent>(
        replay = 0,
        extraBufferCapacity = 64,
        onBufferOverflow = BufferOverflow.DROP_OLDEST
    )
    val events: SharedFlow<AppEvent> = _events.asSharedFlow()

    suspend fun publish(event: AppEvent) {
        _events.emit(event)
    }

    fun tryPublish(event: AppEvent): Boolean {
        return _events.tryEmit(event)
    }
}

sealed class AppEvent {
    data class UserLoggedIn(val userId: String) : AppEvent()
    data class OrderPlaced(val orderId: String, val amount: Double) : AppEvent()
    data class PaymentReceived(val paymentId: String) : AppEvent()
    object SystemMaintenanceStarted : AppEvent()
}

// การใช้งาน EventBus
class OrderService(private val eventBus: EventBus) {
    suspend fun placeOrder(userId: String, amount: Double): String {
        val orderId = createOrder(userId, amount)
        eventBus.publish(AppEvent.OrderPlaced(orderId, amount))
        return orderId
    }
}

class NotificationService(private val eventBus: EventBus) {
    suspend fun listenForEvents() {
        eventBus.events.collect { event ->
            when (event) {
                is AppEvent.OrderPlaced -> sendOrderConfirmation(event.orderId)
                is AppEvent.PaymentReceived -> sendPaymentReceipt(event.paymentId)
                else -> Unit
            }
        }
    }
}
```

### เปรียบเทียบ StateFlow vs SharedFlow

```kotlin
// StateFlow:
// - มี current value เสมอ (initial value required)
// - replay = 1 เสมอ (new subscriber ได้ค่าล่าสุด)
// - conflates (ถ้า emit เร็วกว่า collect จะเก็บแค่ล่าสุด)
// - เหมาะกับ: UI state, current value

// SharedFlow:
// - ไม่มี initial value
// - replay configurable (0 to N)
// - ไม่ conflate (เก็บทุก event)
// - เหมาะกับ: events, one-shot operations

fun comparison() {
    // StateFlow - UI state
    val uiState = MutableStateFlow(UiState.Loading)
    // Subscribers เสมอรู้ current state

    // SharedFlow - Events
    val events = MutableSharedFlow<UiEvent>()
    // Events ไม่มี "current value" - แค่ stream ของ events
}
```

## 3. Channels

### Channel Types

```kotlin
import kotlinx.coroutines.channels.*

suspend fun channelTypesExample() {
    // Rendezvous Channel - ไม่มี buffer
    // Sender รอจนกว่า receiver จะรับ
    val rendezvous = Channel<Int>() // Channel<Int>(0)
    
    // Buffered Channel - มี buffer ขนาดที่กำหนด
    val buffered = Channel<Int>(100)
    
    // Conflated Channel - เก็บแค่ค่าล่าสุด
    val conflated = Channel<Int>(Channel.CONFLATED)
    
    // Unlimited Channel - buffer ไม่จำกัด (ระวัง OOM)
    val unlimited = Channel<Int>(Channel.UNLIMITED)
    
    // Custom behavior
    val withDrop = Channel<Int>(
        capacity = 10,
        onBufferOverflow = BufferOverflow.DROP_OLDEST
    )
}
```

### Producer-Consumer Pattern

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.channels.*

// Producer Coroutine
fun CoroutineScope.produceNumbers(): ReceiveChannel<Int> = produce {
    var n = 1
    while (true) {
        send(n++)
        delay(100)
    }
}

// Pipeline Pattern
fun CoroutineScope.square(numbers: ReceiveChannel<Int>): ReceiveChannel<Int> = produce {
    for (n in numbers) {
        send(n * n)
    }
}

fun CoroutineScope.filter(
    numbers: ReceiveChannel<Int>,
    predicate: (Int) -> Boolean
): ReceiveChannel<Int> = produce {
    for (n in numbers) {
        if (predicate(n)) send(n)
    }
}

suspend fun pipelineExample() = coroutineScope {
    val numbers = produceNumbers()
    val squares = square(numbers)
    val evenSquares = filter(squares) { it % 2 == 0 }
    
    repeat(5) {
        println(evenSquares.receive())
    }
    
    coroutineContext.cancelChildren()
}
```

### Fan-out และ Fan-in

```kotlin
// Fan-out: หลาย consumers รับจาก channel เดียว
fun CoroutineScope.launchWorker(
    id: Int,
    channel: ReceiveChannel<Int>
): Job = launch {
    for (item in channel) {
        println("Worker $id processing: $item")
        delay(100)
    }
}

suspend fun fanOutExample() = coroutineScope {
    val channel = produce {
        repeat(20) { send(it) }
    }
    
    val workers = List(4) { id ->
        launchWorker(id, channel)
    }
    
    workers.forEach { it.join() }
}

// Fan-in: หลาย producers ส่งไปยัง channel เดียว
fun CoroutineScope.mergeChannels(
    vararg channels: ReceiveChannel<String>
): ReceiveChannel<String> = produce {
    channels.forEach { channel ->
        launch {
            for (element in channel) {
                send(element)
            }
        }
    }
}
```

## 4. Structured Concurrency Patterns

### SupervisorJob

```kotlin
import kotlinx.coroutines.*

class DataProcessor(private val scope: CoroutineScope) {

    // SupervisorJob: child failures ไม่กระทบ siblings
    private val supervisorScope = CoroutineScope(
        scope.coroutineContext + SupervisorJob()
    )

    fun processMultipleItems(items: List<Int>): List<Deferred<String>> {
        return items.map { item ->
            supervisorScope.async {
                processItem(item)
            }
        }
    }

    private suspend fun processItem(item: Int): String {
        delay(100)
        if (item == 3) throw Exception("Item 3 failed!")
        return "Processed: $item"
    }
}

suspend fun supervisorExample() {
    val scope = CoroutineScope(Dispatchers.Default)
    val processor = DataProcessor(scope)
    
    val results = processor.processMultipleItems(listOf(1, 2, 3, 4, 5))
    
    results.forEachIndexed { index, deferred ->
        try {
            println("Item ${index + 1}: ${deferred.await()}")
        } catch (e: Exception) {
            println("Item ${index + 1} failed: ${e.message}")
        }
    }
}
```

### coroutineScope vs supervisorScope

```kotlin
// coroutineScope: หาก child ใด fail → cancel ทั้งหมด
suspend fun withCoroutineScope() {
    try {
        coroutineScope {
            launch {
                delay(100)
                throw Exception("Child 1 failed")
            }
            launch {
                delay(200)
                println("Child 2 completed") // จะไม่ถูก execute
            }
        }
    } catch (e: Exception) {
        println("Caught: ${e.message}")
    }
}

// supervisorScope: child ใด fail → ไม่กระทบคนอื่น
suspend fun withSupervisorScope() {
    supervisorScope {
        val job1 = launch {
            delay(100)
            throw Exception("Child 1 failed")
        }
        
        val job2 = launch {
            delay(200)
            println("Child 2 completed") // จะ execute ได้ปกติ
        }
        
        job1.join() // ไม่ throw exception ที่นี่
        job2.join()
    }
}
```

### withTimeout และ withTimeoutOrNull

```kotlin
suspend fun timeoutExamples() {
    // withTimeout - throw CancellationException เมื่อ timeout
    try {
        val result = withTimeout(1000L) {
            delay(500)  // ทำงานสำเร็จ
            "Success"
        }
        println(result)
    } catch (e: TimeoutCancellationException) {
        println("Timeout!")
    }

    // withTimeoutOrNull - return null เมื่อ timeout
    val result = withTimeoutOrNull(1000L) {
        delay(500)
        "Success"
    }
    println(result ?: "Timed out")

    // หลาย operations พร้อมกัน (รอทุกอัน)
    val (a, b, c) = coroutineScope {
        val deferredA = async { fetchDataA() }
        val deferredB = async { fetchDataB() }
        val deferredC = async { fetchDataC() }
        Triple(deferredA.await(), deferredB.await(), deferredC.await())
    }

    // รอแค่อันแรกที่เสร็จ
    val fastest = select<String> {
        async { fetchDataA() }.onAwait { "A: $it" }
        async { fetchDataB() }.onAwait { "B: $it" }
        async { fetchDataC() }.onAwait { "C: $it" }
    }
}
```

## 5. Real-time Data Pipeline

```kotlin
// DataPipeline.kt
package com.pipeline

import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*
import kotlin.time.Duration.Companion.seconds

data class SensorReading(
    val sensorId: String,
    val value: Double,
    val timestamp: Long = System.currentTimeMillis()
)

data class Alert(
    val sensorId: String,
    val message: String,
    val severity: AlertSeverity
)

enum class AlertSeverity { LOW, MEDIUM, HIGH, CRITICAL }

class SensorDataPipeline(
    private val scope: CoroutineScope = CoroutineScope(Dispatchers.Default + SupervisorJob())
) {

    // Simulate sensor data stream
    fun sensorDataFlow(sensorId: String): Flow<SensorReading> = flow {
        while (true) {
            emit(SensorReading(
                sensorId = sensorId,
                value = (20..35).random().toDouble() + Math.random()
            ))
            delay(1000) // อ่านข้อมูลทุก 1 วินาที
        }
    }.flowOn(Dispatchers.IO)

    // Moving average ด้วย sliding window
    fun Flow<SensorReading>.movingAverage(windowSize: Int): Flow<Pair<String, Double>> {
        val windows = mutableMapOf<String, ArrayDeque<Double>>()
        
        return transform { reading ->
            val window = windows.getOrPut(reading.sensorId) { ArrayDeque() }
            window.addLast(reading.value)
            if (window.size > windowSize) window.removeFirst()
            
            if (window.size == windowSize) {
                emit(Pair(reading.sensorId, window.average()))
            }
        }
    }

    // Detect anomalies
    fun Flow<Pair<String, Double>>.detectAnomalies(threshold: Double): Flow<Alert> {
        val baselines = mutableMapOf<String, Double>()
        var count = 0
        
        return transform { (sensorId, avg) ->
            val baseline = baselines.getOrPut(sensorId) { avg }
            
            // อัปเดต baseline ทุกๆ 10 readings
            if (++count % 10 == 0) {
                baselines[sensorId] = (baseline * 0.9 + avg * 0.1)
            }
            
            val deviation = Math.abs(avg - baseline) / baseline
            
            if (deviation > threshold * 2) {
                emit(Alert(sensorId, "Critical deviation: ${String.format("%.2f", deviation * 100)}%", AlertSeverity.CRITICAL))
            } else if (deviation > threshold) {
                emit(Alert(sensorId, "High deviation: ${String.format("%.2f", deviation * 100)}%", AlertSeverity.HIGH))
            }
        }
    }

    // Aggregate data from multiple sensors
    fun createPipeline(sensorIds: List<String>): Flow<ProcessedData> {
        val sensorFlows = sensorIds.map { sensorDataFlow(it) }
        
        return merge(*sensorFlows.toTypedArray())
            .movingAverage(windowSize = 5)
            .buffer(capacity = 100)
            .runningFold(mutableMapOf<String, Double>()) { acc, (sensorId, avg) ->
                acc.apply { put(sensorId, avg) }
            }
            .filter { it.size == sensorIds.size } // รอให้ครบทุก sensor
            .map { readings ->
                ProcessedData(
                    readings = readings.toMap(),
                    timestamp = System.currentTimeMillis(),
                    summary = calculateSummary(readings)
                )
            }
            .sample(2.seconds) // Sample ทุก 2 วินาที
    }

    fun startMonitoring(sensorIds: List<String>) {
        scope.launch {
            // Separate flows for different purposes
            val sensorFlow = merge(*sensorIds.map { sensorDataFlow(it) }.toTypedArray())
                .shareIn(scope, SharingStarted.WhileSubscribed(5000), replay = 1)

            // Flow 1: Alert monitoring
            launch {
                sensorFlow
                    .movingAverage(windowSize = 5)
                    .detectAnomalies(threshold = 0.2)
                    .collect { alert ->
                        handleAlert(alert)
                    }
            }

            // Flow 2: Data logging
            launch {
                sensorFlow
                    .buffer(capacity = 50)
                    .collect { reading ->
                        logReading(reading)
                    }
            }

            // Flow 3: Processed data for dashboard
            launch {
                createPipeline(sensorIds).collect { data ->
                    updateDashboard(data)
                }
            }
        }
    }

    private fun calculateSummary(readings: Map<String, Double>): DataSummary {
        return DataSummary(
            min = readings.values.minOrNull() ?: 0.0,
            max = readings.values.maxOrNull() ?: 0.0,
            average = readings.values.average()
        )
    }

    private suspend fun handleAlert(alert: Alert) {
        println("[ALERT] ${alert.severity}: ${alert.sensorId} - ${alert.message}")
    }

    private suspend fun logReading(reading: SensorReading) {
        // บันทึกข้อมูลลง database
    }

    private suspend fun updateDashboard(data: ProcessedData) {
        // ส่งข้อมูลไปยัง dashboard ผ่าน WebSocket
    }
}

data class ProcessedData(
    val readings: Map<String, Double>,
    val timestamp: Long,
    val summary: DataSummary
)

data class DataSummary(val min: Double, val max: Double, val average: Double)
```

## 6. Coroutine Testing

```kotlin
// CoroutineTests.kt
package com.test

import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*
import kotlinx.coroutines.test.*
import org.junit.jupiter.api.Test

class CoroutineTests {

    @Test
    fun `test with TestCoroutineScope`() = runTest {
        val results = mutableListOf<Int>()
        
        flowOf(1, 2, 3)
            .map { it * 2 }
            .collect { results.add(it) }
        
        assert(results == listOf(2, 4, 6))
    }

    @Test
    fun `test time-based flow operations`() = runTest {
        val results = mutableListOf<Int>()
        
        flow {
            emit(1)
            delay(1000)
            emit(2)
            delay(2000)
            emit(3)
        }
        .collect { results.add(it) }
        
        assert(results == listOf(1, 2, 3))
        // runTest ทำให้ delay ไม่ใช้เวลาจริง
    }

    @Test
    fun `test StateFlow`() = runTest {
        val stateFlow = MutableStateFlow(0)
        val collected = mutableListOf<Int>()
        
        val job = launch {
            stateFlow
                .take(4)
                .collect { collected.add(it) }
        }
        
        advanceUntilIdle()
        stateFlow.value = 1
        stateFlow.value = 2
        stateFlow.value = 3
        
        job.join()
        
        assert(collected == listOf(0, 1, 2, 3))
    }

    @Test
    fun `test SharedFlow events`() = runTest {
        val sharedFlow = MutableSharedFlow<String>()
        val received = mutableListOf<String>()
        
        val job = launch {
            sharedFlow.take(3).collect { received.add(it) }
        }
        
        advanceUntilIdle()
        sharedFlow.emit("event1")
        sharedFlow.emit("event2")
        sharedFlow.emit("event3")
        
        job.join()
        
        assert(received == listOf("event1", "event2", "event3"))
    }

    @Test
    fun `test timeout handling`() = runTest {
        val result = withTimeoutOrNull(1000L) {
            delay(500)
            "completed"
        }
        
        assert(result == "completed")
        
        val timedOut = withTimeoutOrNull(500L) {
            delay(1000)
            "completed"
        }
        
        assert(timedOut == null)
    }

    @Test
    fun `test concurrent coroutines`() = runTest {
        val results = mutableListOf<Int>()
        
        coroutineScope {
            val jobs = (1..5).map { num ->
                async {
                    delay(num * 100L)
                    num
                }
            }
            
            jobs.awaitAll().forEach { results.add(it) }
        }
        
        assert(results == listOf(1, 2, 3, 4, 5))
    }
}
```

## สรุป Advanced Coroutines

| Feature | ใช้เมื่อ | ตัวอย่าง |
|---------|---------|---------|
| Flow.transform | Custom transformation logic | filterAndMap |
| Flow.combine | รวมหลาย flows ด้วยค่าล่าสุด | temperature + humidity |
| Flow.zip | จับคู่ elements ตาม order | names + scores |
| StateFlow | Current state ที่มี value เสมอ | UI state |
| SharedFlow | Events/broadcasts | AppEvent bus |
| Channel | Direct communication ระหว่าง coroutines | producer-consumer |
| SupervisorJob | Child failures ไม่กระทบ siblings | batch processing |
| supervisorScope | Local supervisor scope | independent tasks |
| runTest | Testing coroutines | unit tests |

*Part 64/100+ | Kotlin & Spring Boot Complete Course*
