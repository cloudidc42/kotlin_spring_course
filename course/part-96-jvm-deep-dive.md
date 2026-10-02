# Part 96: JVM Deep Dive
## เข้าใจ JVM จากภายในเพื่อ Production Performance

---

## 🎯 เป้าหมายของ Part นี้

- JVM Memory Model
- Garbage Collection algorithms
- JIT Compilation
- Class Loading
- JVM Flags สำหรับ Production
- Monitoring JVM ใน Production

---

## 📖 1. JVM Memory Model

```
JVM Memory Layout:
┌─────────────────────────────────────────────────────┐
│                    JVM Process                       │
│  ┌──────────────────────────────────────────────┐   │
│  │              Java Heap                        │   │
│  │  ┌─────────────────┐  ┌────────────────────┐ │   │
│  │  │   Young Gen     │  │      Old Gen       │ │   │
│  │  │  ┌───┐ ┌──────┐ │  │  (Tenured Space)  │ │   │
│  │  │  │ E │ │  S0  │ │  │                   │ │   │
│  │  │  │ d │ │      │ │  │   Long-lived       │ │   │
│  │  │  │ e │ │  S1  │ │  │   objects          │ │   │
│  │  │  │ n │ └──────┘ │  │                   │ │   │
│  │  │  └───┘          │  └────────────────────┘ │   │
│  │  └─────────────────┘                          │   │
│  └──────────────────────────────────────────────┘   │
│  ┌──────────────┐  ┌───────┐  ┌─────────────────┐  │
│  │  Metaspace   │  │ Stack │  │  Native Memory  │  │
│  │ (Class Meta) │  │(each) │  │  (off-heap)     │  │
│  └──────────────┘  └───────┘  └─────────────────┘  │
└─────────────────────────────────────────────────────┘
```

### Heap Regions

```
Young Generation:
  Eden Space: สร้าง objects ใหม่ทั้งหมด
  Survivor 0 (S0): objects ที่รอด GC ครั้งแรก
  Survivor 1 (S1): objects ที่รอด GC ครั้งที่สอง

เมื่อ Eden เต็ม → Minor GC เกิดขึ้น
Objects ที่รอดพอ GC rounds → ย้ายไป Old Gen

Old Generation:
  Long-lived objects
  เมื่อเต็ม → Major GC (หรือ Full GC) เกิดขึ้น
  Full GC ช้ามาก! (Stop-the-world)

Metaspace:
  Class metadata (ไม่ใช่ heap)
  Method bytecode
  ไม่มี fixed limit (ขยายได้ตาม OS memory)
```

---

## 🗑️ 2. Garbage Collection Algorithms

### Serial GC

```bash
# สำหรับ small applications, single-threaded
java -XX:+UseSerialGC -jar app.jar

# ไม่เหมาะกับ production server applications
```

### G1 GC (Garbage First) - Default ใน Java 11+

```bash
# G1GC - balanced throughput และ low latency
java -XX:+UseG1GC \
     -XX:MaxGCPauseMillis=200 \    # target pause time
     -XX:G1HeapRegionSize=8m \     # region size
     -jar app.jar

# G1 แบ่ง heap เป็น regions เท่าๆ กัน
# Collect regions ที่มี garbage เยอะที่สุดก่อน (G1 = Garbage First)
```

### ZGC - Low Latency (Java 15+)

```bash
# ZGC - sub-millisecond pauses
java -XX:+UseZGC \
     -XX:ConcGCThreads=6 \
     -jar app.jar

# ดีที่สุดสำหรับ latency-sensitive applications
# Pause time < 1ms โดยไม่ขึ้นกับ heap size
```

### Shenandoah GC (Java 12+)

```bash
# คล้าย ZGC แต่ใช้ memory น้อยกว่า
java -XX:+UseShenandoahGC -jar app.jar
```

### GC Comparison

```
Algorithm   | Throughput | Latency | Memory | Use Case
Serial      | Low        | High    | Low    | Small apps
Parallel    | High       | Medium  | Medium | Batch processing
G1          | High       | Low     | Medium | General purpose
ZGC         | High       | VLow    | High   | Low-latency services
Shenandoah  | Medium     | VLow    | Medium | Low-latency, less memory
```

---

## ⚡ 3. JIT Compilation

```
JIT (Just-In-Time) compilation:
  Bytecode → Machine code (เฉพาะ hot methods)
  
Tiered Compilation (Java 8+):
  Level 0: Interpreter
  Level 1: C1 (client) - fast compile, no optimization
  Level 2: C1 + light profiling
  Level 3: C1 + full profiling  
  Level 4: C2 (server) - slow compile, heavy optimization
```

```bash
# ดู JIT compilation activity
java -XX:+PrintCompilation -jar app.jar 2>&1 | head -50

# Output:
#    1    1  n  java.lang.System::arraycopy (0 bytes)  (native)
#  127   17    com.example.UserService::findById (45 bytes)
#  234   17%  com.example.OrderService::processOrders (789 bytes)  (OSR)
```

### JIT Optimization Examples

```kotlin
// JIT ทำ optimizations เช่น:

// 1. Method Inlining
fun add(a: Int, b: Int) = a + b

fun calculate(x: Int, y: Int): Int {
    return add(x, y)  // JIT จะ inline add() ไปใน calculate
    // กลายเป็น: return x + y (ไม่มี method call overhead)
}

// 2. Dead Code Elimination
fun example(debug: Boolean) {
    if (debug) {
        println("Debug: ...")  // ถ้า debug always false → JIT ลบออก
    }
    doActualWork()
}

// 3. Loop Unrolling
for (i in 0..3) {
    process(i)
    // JIT อาจเปลี่ยนเป็น:
    // process(0); process(1); process(2); process(3);
}
```

---

## 📂 4. Class Loading

```
Class Loading Process:
1. Loading: อ่าน .class file จาก classpath
2. Linking:
   a. Verification: ตรวจสอบ bytecode ถูกต้อง
   b. Preparation: จัดสรร memory สำหรับ static fields
   c. Resolution: แก้ symbolic references
3. Initialization: รัน static initializers

ClassLoader Hierarchy:
Bootstrap ClassLoader → ExtClass/Platform ClassLoader → App ClassLoader → Custom ClassLoaders
```

```kotlin
// ดู ClassLoader
class MyClass

fun main() {
    println(MyClass::class.java.classLoader)
    // jdk.internal.loader.ClassLoaders$AppClassLoader@XXX

    println(String::class.java.classLoader)
    // null = Bootstrap ClassLoader (เขียนด้วย C++)

    // Custom ClassLoader
    val urlClassLoader = URLClassLoader(
        arrayOf(URL("file:///path/to/plugins/")),
        Thread.currentThread().contextClassLoader
    )
    val pluginClass = urlClassLoader.loadClass("com.plugin.MyPlugin")
}
```

---

## ⚙️ 5. JVM Flags สำหรับ Production

### Heap Size

```bash
# ตั้ง initial = max เพื่อหลีกเลี่ยง heap resize
java -Xms2g -Xmx2g \

# หรือใน container (% ของ container memory)
java -XX:InitialRAMPercentage=50 \
     -XX:MaxRAMPercentage=75 \
```

### GC Tuning

```bash
# G1GC Production Config
java \
  -XX:+UseG1GC \
  -XX:MaxGCPauseMillis=200 \
  -XX:+ParallelRefProcEnabled \
  -XX:G1HeapWastePercent=5 \
  -XX:G1MixedGCCountTarget=4 \
  -XX:InitiatingHeapOccupancyPercent=45 \
  -jar app.jar
```

### GC Logging

```bash
# Java 11+ GC logging
java \
  -Xlog:gc*:file=/var/log/app/gc.log:time,pid:filecount=5,filesize=20m \
  -jar app.jar

# Parse GC logs with GCEasy or GCViewer
```

### Complete Production JVM Flags

```bash
#!/bin/bash
# start.sh - production JVM settings

exec java \
  # Memory
  -Xms512m \
  -Xmx2g \
  -XX:MetaspaceSize=256m \
  -XX:MaxMetaspaceSize=512m \
  \
  # GC
  -XX:+UseG1GC \
  -XX:MaxGCPauseMillis=200 \
  -XX:+ExplicitGCInvokesConcurrent \
  \
  # JIT
  -XX:+TieredCompilation \
  -XX:ReservedCodeCacheSize=256m \
  \
  # OOM handling
  -XX:+HeapDumpOnOutOfMemoryError \
  -XX:HeapDumpPath=/var/log/app/heap-dump.hprof \
  -XX:+ExitOnOutOfMemoryError \
  \
  # GC Logging
  -Xlog:gc*:file=/var/log/app/gc.log:time,pid:filecount=5,filesize=20m \
  \
  # JMX Monitoring
  -Dcom.sun.management.jmxremote=true \
  -Dcom.sun.management.jmxremote.port=9999 \
  -Dcom.sun.management.jmxremote.authenticate=false \
  -Dcom.sun.management.jmxremote.ssl=false \
  \
  # Application
  -jar app.jar "$@"
```

---

## 📊 6. Monitoring JVM ใน Production

### Micrometer + Prometheus + Grafana

```kotlin
// build.gradle.kts
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-actuator")
    implementation("io.micrometer:micrometer-registry-prometheus")
}

// application.yml
management:
  endpoints:
    web:
      exposure:
        include: prometheus,health,info,metrics
  metrics:
    export:
      prometheus:
        enabled: true
    tags:
      application: ${spring.application.name}
      environment: ${spring.profiles.active:default}
```

### Custom JVM Metrics

```kotlin
@Component
class JvmMetrics(private val meterRegistry: MeterRegistry) {

    @PostConstruct
    fun registerMetrics() {
        // Heap usage
        val runtime = Runtime.getRuntime()

        Gauge.builder("jvm.heap.used", runtime) { rt ->
            ((rt.totalMemory() - rt.freeMemory()) / 1024 / 1024).toDouble()
        }
        .description("JVM heap used in MB")
        .baseUnit("megabytes")
        .register(meterRegistry)

        // GC info
        ManagementFactory.getGarbageCollectorMXBeans().forEach { gcBean ->
            Gauge.builder("jvm.gc.collections", gcBean) { gc ->
                gc.collectionCount.toDouble()
            }
            .tag("gc", gcBean.name)
            .register(meterRegistry)
        }
    }
}
```

### Grafana Dashboard Queries

```promql
# Heap Usage %
100 * (jvm_memory_used_bytes{area="heap"} / jvm_memory_max_bytes{area="heap"})

# GC Pause Rate
rate(jvm_gc_pause_seconds_sum[5m])

# Thread Count
jvm_threads_live_threads

# Class Loading
jvm_classes_loaded_classes
```

### Flight Recorder สำหรับ Profiling

```bash
# เริ่ม JFR recording
java -XX:+FlightRecorder \
     -XX:StartFlightRecording=duration=60s,filename=/tmp/recording.jfr \
     -jar app.jar

# Dump recording ของ running process
jcmd <pid> JFR.start duration=60s filename=/tmp/app.jfr
jcmd <pid> JFR.dump filename=/tmp/app-dump.jfr

# Analyze กับ JDK Mission Control
jmc /tmp/app.jfr
```

---

## 📋 สรุป

| หัวข้อ | Key Points |
|--------|-----------|
| Heap | Young Gen (Eden + Survivors) + Old Gen |
| Metaspace | Class metadata ไม่ใน heap |
| G1GC | Default, balanced throughput/latency |
| ZGC | Sub-millisecond pause, Java 15+ |
| JIT | Tiered compilation, hot method optimization |
| Production Flags | Heap size, GC logging, heap dump on OOM |
| Monitoring | Prometheus + Grafana + JFR |

---

*Part 96/100+ | Kotlin & Spring Boot Complete Course*
