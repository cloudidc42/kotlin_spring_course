# Part 70: Performance Profiling

## Performance Profiling — วิเคราะห์และปรับปรุงประสิทธิภาพ JVM Application

---

## 🎯 เป้าหมายของ Part นี้

- เข้าใจ JVM Memory Management
- GC tuning สำหรับ Spring Boot
- Heap dump analysis
- CPU profiling ด้วย Async Profiler
- Spring Boot performance tips
- ปรับปรุง High-traffic API

---

## 📖 1. JVM Memory Structure

```
JVM Memory
├── Heap
│   ├── Young Generation
│   │   ├── Eden Space
│   │   ├── Survivor S0
│   │   └── Survivor S1
│   └── Old Generation (Tenured)
├── Non-Heap
│   ├── Metaspace (Java 8+)
│   └── Code Cache
└── Stack (per thread)
    └── Stack Frames
```

### Memory Regions

- **Eden Space** — new objects เริ่มต้นที่นี่
- **Survivor spaces** — objects ที่รอดจาก Minor GC
- **Old Generation** — long-lived objects
- **Metaspace** — class metadata (แทน PermGen)

---

## ⚙️ 2. JVM Flags สำหรับ Spring Boot

```bash
# เปิด application พร้อม JVM flags
java -jar app.jar \
  -Xms512m \
  -Xmx2g \
  -XX:+UseG1GC \
  -XX:MaxGCPauseMillis=200 \
  -XX:G1HeapRegionSize=16m \
  -XX:+PrintGCDetails \
  -XX:+PrintGCDateStamps \
  -Xloggc:/var/log/gc.log \
  -XX:+HeapDumpOnOutOfMemoryError \
  -XX:HeapDumpPath=/var/log/heap-dump.hprof
```

### application.yml สำหรับ Spring Boot

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
      connection-timeout: 30000
      idle-timeout: 600000
      max-lifetime: 1800000
      
  jpa:
    properties:
      hibernate:
        jdbc:
          batch_size: 50
          fetch_size: 100
        order_inserts: true
        order_updates: true
        generate_statistics: false

management:
  endpoints:
    web:
      exposure:
        include: health,metrics,prometheus
  metrics:
    export:
      prometheus:
        enabled: true
```

---

## 🗑️ 3. Garbage Collectors

### G1GC (แนะนำสำหรับ production)

```bash
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200      # target pause time
-XX:G1NewSizePercent=20       # young gen min %
-XX:G1MaxNewSizePercent=40    # young gen max %
-XX:G1HeapRegionSize=16m      # region size
-XX:G1MixedGCCountTarget=8    # mixed GC cycles
```

### ZGC (สำหรับ low-latency, Java 15+)

```bash
-XX:+UseZGC
-XX:SoftMaxHeapSize=1g        # soft heap limit
-Xmx2g                        # hard limit
```

### Shenandoah GC

```bash
-XX:+UseShenandoahGC
-XX:ShenandoahGCMode=iu       # incremental-update mode
```

---

## 📊 4. Monitoring ด้วย Micrometer + Prometheus

```kotlin
import io.micrometer.core.instrument.*
import io.micrometer.core.annotation.Timed
import org.springframework.stereotype.Service

@Service
class ProductService(
    private val meterRegistry: MeterRegistry,
    private val productRepository: ProductRepository
) {
    private val searchCounter = meterRegistry.counter("product.search.count")
    private val searchLatency = meterRegistry.timer("product.search.latency")

    @Timed(value = "product.service.findAll", description = "Time to find all products")
    fun findAllProducts(): List<ProductResponse> {
        return productRepository.findAll().map { it.toResponse() }
    }

    fun searchProducts(query: String): List<ProductResponse> {
        searchCounter.increment()
        return searchLatency.record {
            productRepository.search(query).map { it.toResponse() }
        }
    }

    fun recordInventoryGauge() {
        meterRegistry.gauge("product.inventory.total") {
            productRepository.count().toDouble()
        }
    }
}
```

### Custom Metrics

```kotlin
import io.micrometer.core.instrument.*

@Component
class BusinessMetrics(private val meterRegistry: MeterRegistry) {

    fun recordOrderPlaced(amount: Double, currency: String) {
        meterRegistry.counter(
            "orders.placed",
            "currency", currency
        ).increment()

        meterRegistry.summary(
            "orders.amount",
            "currency", currency
        ).record(amount)
    }

    fun recordApiLatency(endpoint: String, method: String, status: Int, durationMs: Long) {
        meterRegistry.timer(
            "http.request.duration",
            "endpoint", endpoint,
            "method", method,
            "status", status.toString()
        ).record(durationMs, java.util.concurrent.TimeUnit.MILLISECONDS)
    }
}
```

---

## 🔍 5. Heap Dump Analysis

### สร้าง Heap Dump

```bash
# ขณะ application ทำงาน
jmap -dump:format=b,file=heap.hprof <pid>

# หรือใช้ jcmd
jcmd <pid> GC.heap_dump /tmp/heap.hprof

# automatic เมื่อ OOM
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/var/log/
```

### วิเคราะห์ด้วย Eclipse MAT

```
1. เปิด Eclipse Memory Analyzer (MAT)
2. File -> Open Heap Dump -> เลือก .hprof file
3. ดู "Leak Suspects Report"
4. ตรวจสอบ "Dominator Tree"
5. หา objects ที่มี retained heap สูงผิดปกติ
```

### โค้ดที่ทำให้ Memory Leak

```kotlin
// ❌ Memory leak — List ไม่ถูกลบ
class CacheService {
    private val cache = HashMap<String, ByteArray>()

    fun cache(key: String, data: ByteArray) {
        cache[key] = data // ไม่มีการลบ!
    }
}

// ✅ แก้ไข — ใช้ LRU Cache หรือ expire
import com.github.benmanes.caffeine.cache.Caffeine
import java.time.Duration

@Service
class CacheService {
    private val cache = Caffeine.newBuilder()
        .maximumSize(10_000)
        .expireAfterWrite(Duration.ofMinutes(30))
        .recordStats()
        .build<String, ByteArray>()

    fun cache(key: String, data: ByteArray) {
        cache.put(key, data)
    }

    fun get(key: String): ByteArray? = cache.getIfPresent(key)
}
```

---

## ⚡ 6. CPU Profiling ด้วย Async Profiler

```bash
# ดาวน์โหลด async-profiler
wget https://github.com/async-profiler/async-profiler/releases/download/v3.0/async-profiler-3.0-linux-x64.tar.gz
tar -xzf async-profiler-3.0-linux-x64.tar.gz

# Profile CPU เป็น flame graph
./profiler.sh -d 30 -f flamegraph.html <pid>

# Profile allocation
./profiler.sh -e alloc -d 30 -f alloc.html <pid>

# Profile lock contention
./profiler.sh -e lock -d 30 -f lock.html <pid>
```

### Spring Boot Actuator + Continuous Profiling

```kotlin
@RestController
@RequestMapping("/actuator/profiling")
class ProfilingController {

    @PostMapping("/start")
    fun startProfiling(@RequestParam duration: Int = 30): ResponseEntity<String> {
        // ใช้ async-profiler Java API
        AsyncProfiler.getInstance().execute("start,event=cpu,file=/tmp/profile.jfr")
        return ResponseEntity.ok("Profiling started for ${duration}s")
    }

    @PostMapping("/stop")
    fun stopProfiling(): ResponseEntity<String> {
        AsyncProfiler.getInstance().execute("stop")
        return ResponseEntity.ok("Profiling stopped")
    }
}
```

---

## 🚀 7. Spring Boot Performance Tips

### Database Query Optimization

```kotlin
// ❌ N+1 Problem
@Service
class OrderService(private val orderRepository: OrderRepository) {
    fun getAllOrders(): List<OrderDto> {
        return orderRepository.findAll().map { order ->
            OrderDto(
                id = order.id,
                items = order.items.map { it.toDto() } // ทุก order load items แยก
            )
        }
    }
}

// ✅ เพิ่ม JOIN FETCH
@Repository
interface OrderRepository : JpaRepository<Order, Long> {
    @Query("SELECT o FROM Order o JOIN FETCH o.items WHERE o.userId = :userId")
    fun findByUserIdWithItems(@Param("userId") userId: Long): List<Order>
    
    @EntityGraph(attributePaths = ["items", "items.product"])
    fun findAll(): List<Order>
}
```

### Async Processing

```kotlin
@Service
class NotificationService {

    @Async("taskExecutor")
    fun sendEmailAsync(email: String, message: String): CompletableFuture<Void> {
        // ส่ง email แบบ async ไม่บล็อก main thread
        emailClient.send(email, message)
        return CompletableFuture.completedFuture(null)
    }
}

@Configuration
class AsyncConfig {

    @Bean("taskExecutor")
    fun taskExecutor(): Executor = ThreadPoolTaskExecutor().apply {
        corePoolSize = 10
        maxPoolSize = 50
        queueCapacity = 500
        threadNamePrefix = "async-task-"
        initialize()
    }
}
```

### Response Compression

```yaml
server:
  compression:
    enabled: true
    mime-types: application/json,application/xml,text/html
    min-response-size: 1024
```

---

## 📈 8. Optimize High-Traffic API

```kotlin
@RestController
@RequestMapping("/api/v1/products")
class ProductController(
    private val productService: ProductService,
    private val cacheManager: CacheManager
) {

    // Cache responses
    @GetMapping
    @Cacheable(value = ["products"], key = "#page + '-' + #size")
    fun getProducts(
        @RequestParam(defaultValue = "0") page: Int,
        @RequestParam(defaultValue = "20") size: Int
    ): Page<ProductResponse> {
        return productService.findAll(PageRequest.of(page, size))
    }

    // ETag for conditional requests
    @GetMapping("/{id}")
    fun getProduct(@PathVariable id: Long, request: HttpServletRequest): ResponseEntity<ProductResponse> {
        val product = productService.findById(id)
        val eTag = "\"${product.hashCode()}\""
        
        if (request.getHeader("If-None-Match") == eTag) {
            return ResponseEntity.status(HttpStatus.NOT_MODIFIED).build()
        }
        
        return ResponseEntity.ok()
            .eTag(eTag)
            .cacheControl(CacheControl.maxAge(60, TimeUnit.SECONDS))
            .body(product)
    }
}
```

---

## 📊 9. สรุปตาราง Performance Tools

| เครื่องมือ | ใช้สำหรับ | Command/Config |
|-----------|----------|---------------|
| JFR (Java Flight Recorder) | Low-overhead profiling | `-XX:+FlightRecorder` |
| Async Profiler | CPU, memory, lock profiling | `./profiler.sh -d 30` |
| Eclipse MAT | Heap dump analysis | GUI tool |
| JVisualVM | General JVM monitoring | `jvisualvm` |
| Micrometer + Prometheus | Metrics collection | Spring dependency |
| Grafana | Metrics visualization | Dashboard |
| G1GC | Production GC | `-XX:+UseG1GC` |
| ZGC | Low-latency GC | `-XX:+UseZGC` |

---

## 💡 Best Practices

1. **วัดก่อนปรับ** — อย่า optimize โดยไม่มี data
2. **Set JVM flags** ผ่าน environment variables ใน production
3. **Monitor GC logs** อย่างสม่ำเสมอ หา GC pressure
4. **Connection pool size** ควรตรงกับ DB connections ที่รองรับ
5. **Cache อย่างระวัง** — cache invalidation เป็นปัญหา

---

*Part 70/100+ | Kotlin & Spring Boot Complete Course*
