# Part 75: Data Streaming

## Data Streaming — Real-time Analytics ด้วย Kafka Streams และ Spring Cloud Stream

---

## 🎯 เป้าหมายของ Part นี้

- Spring Cloud Stream
- Kafka Streams
- Windowing operations
- State stores
- สร้าง Real-time analytics pipeline

---

## 📖 1. Streaming vs Batch Processing

```
Batch Processing:
ข้อมูล → รวบรวม → ประมวลผลทั้งหมด → ผลลัพธ์
(ทุกชั่วโมง / ทุกวัน)

Stream Processing:
ข้อมูล → event → ประมวลผลทันที → ผลลัพธ์ real-time
(milliseconds)
```

### Use Cases

- Real-time fraud detection
- Live analytics dashboard
- IoT sensor processing
- Event-driven microservices
- Real-time recommendations

---

## ⚙️ 2. Spring Cloud Stream Setup

### Dependencies

```kotlin
// build.gradle.kts
dependencies {
    implementation("org.springframework.cloud:spring-cloud-starter-stream-kafka")
    implementation("org.springframework.kafka:spring-kafka")
    implementation("org.apache.kafka:kafka-streams")
}
```

### application.yml

```yaml
spring:
  cloud:
    stream:
      kafka:
        binder:
          brokers: localhost:9092
          auto-create-topics: true
        streams:
          binder:
            configuration:
              default:
                key:
                  serde: org.apache.kafka.common.serialization.Serdes$StringSerde
                value:
                  serde: org.springframework.kafka.support.serializer.JsonSerde
      bindings:
        orderInput-in-0:
          destination: orders
          group: analytics-group
        analyticsOutput-out-0:
          destination: order-analytics
        processOrder-in-0:
          destination: orders
        processOrder-out-0:
          destination: order-results
```

---

## 📨 3. Producer และ Consumer

### Event Classes

```kotlin
data class OrderEvent(
    val orderId: String,
    val userId: String,
    val amount: Double,
    val currency: String,
    val category: String,
    val timestamp: Long = System.currentTimeMillis()
)

data class OrderAnalyticsEvent(
    val windowStart: Long,
    val windowEnd: Long,
    val category: String,
    val orderCount: Long,
    val totalAmount: Double,
    val averageAmount: Double
)
```

### Producer

```kotlin
import org.springframework.cloud.stream.function.StreamBridge
import org.springframework.stereotype.Service

@Service
class OrderEventProducer(private val streamBridge: StreamBridge) {

    fun publishOrderCreated(order: Order) {
        val event = OrderEvent(
            orderId = order.id.toString(),
            userId = order.userId.toString(),
            amount = order.total,
            currency = "THB",
            category = order.category
        )
        streamBridge.send("orders", event)
    }
}
```

### Consumer

```kotlin
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import java.util.function.Consumer

@Configuration
class OrderStreamConfig {

    @Bean
    fun orderInput(): Consumer<OrderEvent> = Consumer { event ->
        println("Received order: ${event.orderId}, amount: ${event.amount}")
        // process event
    }
}
```

---

## 🌊 4. Kafka Streams — DSL

```kotlin
import org.apache.kafka.streams.StreamsBuilder
import org.apache.kafka.streams.kstream.*
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import java.time.Duration

@Configuration
class KafkaStreamsConfig {

    @Bean
    fun orderProcessingTopology(builder: StreamsBuilder): KStream<String, OrderEvent> {
        val orders: KStream<String, OrderEvent> = builder.stream("orders")

        // Filter valid orders
        val validOrders = orders.filter { _, event ->
            event.amount > 0 && event.userId.isNotBlank()
        }

        // Branch by amount
        val branches = validOrders.split(Named.`as`("split-"))
            .branch(
                { _, event -> event.amount > 10000 },
                Branched.`as`("high-value")
            )
            .branch(
                { _, event -> event.amount > 1000 },
                Branched.`as`("medium-value")
            )
            .defaultBranch(Branched.`as`("low-value"))

        branches["split-high-value"]?.to("high-value-orders")
        branches["split-medium-value"]?.to("medium-value-orders")

        return validOrders
    }
}
```

---

## ⏱️ 5. Windowing Operations

```kotlin
@Configuration
class WindowingConfig {

    @Bean
    fun orderAggregationTopology(builder: StreamsBuilder): KTable<Windowed<String>, OrderStats> {
        val orders: KStream<String, OrderEvent> = builder.stream("orders")

        // Tumbling Window — 5 นาที ไม่ซ้อนกัน
        val tumblingWindow = TimeWindows.ofSizeWithNoGrace(Duration.ofMinutes(5))

        return orders
            .groupBy { _, event ->
                KeyValue(event.category, event)
            }
            .windowedBy(tumblingWindow)
            .aggregate(
                { OrderStats() },
                { _, event, stats ->
                    stats.apply {
                        count++
                        totalAmount += event.amount
                        averageAmount = totalAmount / count
                    }
                },
                Materialized.`as`<String, OrderStats, WindowStore<Bytes, ByteArray>>("order-stats-store")
                    .withKeySerde(Serdes.String())
                    .withValueSerde(JsonSerde(OrderStats::class.java))
            )
    }
}

data class OrderStats(
    var count: Long = 0,
    var totalAmount: Double = 0.0,
    var averageAmount: Double = 0.0
)
```

### Sliding Window — สำหรับ Fraud Detection

```kotlin
@Bean
fun fraudDetectionTopology(builder: StreamsBuilder): KStream<String, OrderEvent> {
    val orders: KStream<String, OrderEvent> = builder.stream("orders")

    // Sliding Window — 10 นาที ย้อนหลัง
    val slidingWindow = SlidingWindows.ofTimeDifferenceWithNoGrace(Duration.ofMinutes(10))

    val userOrderCounts = orders
        .groupByKey()
        .windowedBy(slidingWindow)
        .count(
            Materialized.`as`<String, Long, WindowStore<Bytes, ByteArray>>("user-order-counts")
        )

    // Alert เมื่อ user สั่งมากกว่า 10 ครั้งใน 10 นาที
    return userOrderCounts
        .toStream()
        .filter { key, count -> count > 10 }
        .map { key, count ->
            KeyValue(
                key.key(),
                FraudAlert(userId = key.key(), orderCount = count, windowStart = key.window().start())
            )
        }
        .also { it.to("fraud-alerts") }
        .mapValues { alert -> orders.filter { _, e -> e.userId == alert.userId }.peek { k, v -> } }
        .flatMapValues { emptyList<OrderEvent>() }
}
```

---

## 🗃️ 6. State Stores

```kotlin
import org.apache.kafka.streams.state.*

@Configuration
class StateStoreConfig {

    @Bean
    fun orderStatsTopology(builder: StreamsBuilder): KTable<String, UserOrderStats> {
        // สร้าง persistent state store
        val storeBuilder: StoreBuilder<KeyValueStore<String, UserOrderStats>> =
            Stores.keyValueStoreBuilder(
                Stores.persistentKeyValueStore("user-order-stats"),
                Serdes.String(),
                JsonSerde(UserOrderStats::class.java)
            )

        builder.addStateStore(storeBuilder)

        val orders: KStream<String, OrderEvent> = builder.stream("orders")

        return orders
            .groupByKey()
            .aggregate(
                { UserOrderStats() },
                { userId, event, stats ->
                    stats.apply {
                        orderCount++
                        totalSpent += event.amount
                        lastOrderAt = event.timestamp
                        if (orderCount == 1L) firstOrderAt = event.timestamp
                        favoriteCategory = updateFavoriteCategory(event.category)
                    }
                },
                Materialized.`as`<String, UserOrderStats, KeyValueStore<Bytes, ByteArray>>("user-stats")
                    .withKeySerde(Serdes.String())
                    .withValueSerde(JsonSerde(UserOrderStats::class.java))
            )
    }
}

data class UserOrderStats(
    var orderCount: Long = 0,
    var totalSpent: Double = 0.0,
    var firstOrderAt: Long = 0,
    var lastOrderAt: Long = 0,
    var favoriteCategory: String = "",
    val categoryCount: MutableMap<String, Int> = mutableMapOf()
) {
    fun updateFavoriteCategory(category: String): String {
        categoryCount[category] = (categoryCount[category] ?: 0) + 1
        return categoryCount.maxByOrNull { it.value }?.key ?: category
    }
}
```

### Query State Store

```kotlin
@RestController
@RequestMapping("/api/v1/analytics")
class AnalyticsController(
    private val streamsBuilderFactoryBean: StreamsBuilderFactoryBean
) {

    @GetMapping("/user/{userId}/stats")
    fun getUserStats(@PathVariable userId: String): UserOrderStats? {
        val streams = streamsBuilderFactoryBean.kafkaStreams
        val store: ReadOnlyKeyValueStore<String, UserOrderStats> =
            streams.store(
                StoreQueryParameters.fromNameAndType("user-stats", QueryableStoreTypes.keyValueStore())
            )
        return store.get(userId)
    }

    @GetMapping("/window-stats")
    fun getWindowedStats(
        @RequestParam category: String,
        @RequestParam from: Long,
        @RequestParam to: Long
    ): List<WindowedStats> {
        val streams = streamsBuilderFactoryBean.kafkaStreams
        val store: ReadOnlyWindowStore<String, OrderStats> =
            streams.store(
                StoreQueryParameters.fromNameAndType("order-stats-store", QueryableStoreTypes.windowStore())
            )

        val results = mutableListOf<WindowedStats>()
        val iterator = store.fetch(category, java.time.Instant.ofEpochMilli(from), java.time.Instant.ofEpochMilli(to))

        while (iterator.hasNext()) {
            val next = iterator.next()
            results.add(
                WindowedStats(
                    windowStart = next.key.window().start(),
                    windowEnd = next.key.window().end(),
                    stats = next.value
                )
            )
        }

        return results
    }
}
```

---

## 🔄 7. Real-time Analytics Pipeline

```kotlin
@Configuration
class AnalyticsPipelineConfig {

    @Bean
    fun analyticsPipeline(builder: StreamsBuilder): KStream<String, OrderEvent> {
        val orders: KStream<String, OrderEvent> = builder.stream("orders")

        // 1. Enrich events
        val enriched = orders.mapValues { event ->
            event.copy(
                // เพิ่มข้อมูล derived fields
            )
        }

        // 2. Real-time order count per minute
        enriched
            .groupBy { _, _ -> "global" }
            .windowedBy(TimeWindows.ofSizeWithNoGrace(Duration.ofMinutes(1)))
            .count(Materialized.`as`("orders-per-minute"))
            .toStream()
            .map { key, count -> KeyValue("orders-per-minute", "{\"count\": $count}") }
            .to("metrics")

        // 3. Revenue per category
        enriched
            .groupBy { _, event -> event.category }
            .windowedBy(TimeWindows.ofSizeWithNoGrace(Duration.ofHours(1)))
            .aggregate(
                { 0.0 },
                { _, event, total -> total + event.amount },
                Materialized.`as`("revenue-per-category")
            )
            .toStream()
            .map { key, revenue ->
                KeyValue(
                    key.key(),
                    """{"category": "${key.key()}", "revenue": $revenue, "windowStart": ${key.window().start()}}"""
                )
            }
            .to("category-revenue")

        // 4. Detect large orders
        enriched
            .filter { _, event -> event.amount > 50000 }
            .mapValues { event ->
                LargeOrderAlert(
                    orderId = event.orderId,
                    userId = event.userId,
                    amount = event.amount,
                    alertTime = System.currentTimeMillis()
                )
            }
            .to("large-order-alerts")

        return enriched
    }
}

data class LargeOrderAlert(
    val orderId: String,
    val userId: String,
    val amount: Double,
    val alertTime: Long
)
```

---

## 📊 8. สรุปตาราง Kafka Streams Windows

| Window Type | คำอธิบาย | ใช้เมื่อ |
|-------------|---------|---------|
| Tumbling | Fixed size, no overlap | Hourly/Daily reports |
| Hopping | Fixed size, overlapping | Moving averages |
| Sliding | Event-time based, contiguous | Fraud detection |
| Session | Inactivity gap-based | User sessions |

---

## 💡 Best Practices

1. **ตั้ง replication factor** อย่างน้อย 3 สำหรับ production topics
2. **State store backup** ด้วย changelog topics
3. **Dead letter queue** สำหรับ processing errors
4. **Graceful shutdown** — drain inflight messages ก่อน stop
5. **Schema registry** (Confluent) เพื่อ manage message schemas

---

## 📊 9. Kafka Monitoring

```yaml
# JMX metrics สำหรับ Kafka Streams
spring:
  kafka:
    streams:
      properties:
        metrics.recording.level: INFO
        built.in.metrics.version: latest
```

```kotlin
@Component
class KafkaMetricsExporter(
    private val streamsBuilderFactoryBean: StreamsBuilderFactoryBean,
    private val meterRegistry: MeterRegistry
) {

    @Scheduled(fixedRate = 15000)
    fun exportMetrics() {
        val streams = streamsBuilderFactoryBean.kafkaStreams
        val metrics = streams.metrics()

        metrics.forEach { (metricName, metric) ->
            val value = metric.metricValue()
            if (value is Number) {
                meterRegistry.gauge(
                    "kafka.streams.${metricName.name()}",
                    value.toDouble()
                )
            }
        }
    }
}
```

---

## 🛡️ 10. Error Handling in Streams

```kotlin
@Configuration
class StreamsErrorConfig {

    @Bean
    fun defaultProductionExceptionHandler() = object : ProductionExceptionHandler {
        override fun handle(
            record: ProducerRecord<ByteArray, ByteArray>,
            exception: Exception
        ): ProductionExceptionHandler.ProductionExceptionHandlerResponse {
            logger.error("Failed to produce record to ${record.topic()}", exception)
            // ส่งไป dead letter topic
            return ProductionExceptionHandler.ProductionExceptionHandlerResponse.CONTINUE
        }
    }

    @Bean
    fun defaultDeserializationExceptionHandler() = object : DeserializationExceptionHandler {
        override fun handle(
            context: ProcessorContext,
            record: ConsumerRecord<ByteArray, ByteArray>,
            exception: Exception
        ): DeserializationExceptionHandler.DeserializationHandlerResponse {
            logger.error("Failed to deserialize record from ${record.topic()}", exception)
            return DeserializationExceptionHandler.DeserializationHandlerResponse.CONTINUE
        }
    }
}
```

---

*Part 75/100+ | Kotlin & Spring Boot Complete Course*
