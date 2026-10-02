# Part 49: Monitoring และ Observability
## การติดตามและวิเคราะห์ระบบด้วย Micrometer, Prometheus และ Grafana

---

## 🎯 เป้าหมายของ Part นี้

- Micrometer Metrics Framework
- Spring Boot Actuator setup
- Prometheus integration
- Grafana dashboards
- Distributed Tracing ด้วย Micrometer Tracing
- Custom metrics
- Alerts setup
- ตัวอย่างจริง: API Monitoring Dashboard

---

## 📚 1. Observability 3 Pillars

Observability ประกอบด้วย 3 ส่วนหลัก:

```
┌─────────────────────────────────────────────────────────┐
│                    OBSERVABILITY                        │
├───────────────┬───────────────────┬─────────────────────┤
│    METRICS    │      TRACES       │       LOGS          │
│               │                   │                     │
│ Numbers over  │ Request journey   │ Timestamped         │
│ time          │ across services   │ text events         │
│               │                   │                     │
│ Prometheus    │ Zipkin/Jaeger     │ ELK Stack           │
│ Grafana       │ Tempo             │ Loki                │
└───────────────┴───────────────────┴─────────────────────┘
```

### ทำไมต้อง Monitor?

- รู้ก่อน user ว่าระบบมีปัญหา
- หา bottleneck และ optimize performance
- ทำ capacity planning
- Compliance และ audit requirements
- Debug production issues

---

## 🔧 2. Dependencies Setup

```kotlin
// build.gradle.kts
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
    
    // Spring Boot Actuator
    implementation("org.springframework.boot:spring-boot-starter-actuator")
    
    // Micrometer + Prometheus
    implementation("io.micrometer:micrometer-registry-prometheus")
    
    // Micrometer Tracing (Zipkin/Brave)
    implementation("io.micrometer:micrometer-tracing-bridge-brave")
    implementation("io.zipkin.reporter2:zipkin-reporter-brave")
    
    // หรือ OpenTelemetry
    // implementation("io.micrometer:micrometer-tracing-bridge-otel")
    // implementation("io.opentelemetry:opentelemetry-exporter-otlp")
    
    implementation("com.fasterxml.jackson.module:jackson-module-kotlin")
    implementation("org.jetbrains.kotlin:kotlin-reflect")
    runtimeOnly("com.h2database:h2")
    testImplementation("org.springframework.boot:spring-boot-starter-test")
}
```

```yaml
# application.yml
spring:
  application:
    name: monitoring-demo

management:
  endpoints:
    web:
      exposure:
        include: health, info, metrics, prometheus, env, beans, loggers, httptrace, threaddump, heapdump
  endpoint:
    health:
      show-details: always
      show-components: always
      probes:
        enabled: true  # Kubernetes liveness/readiness probes
    prometheus:
      enabled: true
    metrics:
      enabled: true
  metrics:
    tags:
      application: ${spring.application.name}
      environment: ${ENVIRONMENT:development}
    distribution:
      percentiles-histogram:
        http.server.requests: true
      percentiles:
        http.server.requests: 0.5, 0.95, 0.99
      slo:
        http.server.requests: 50ms, 100ms, 200ms, 500ms
  tracing:
    sampling:
      probability: 1.0  # 100% sampling สำหรับ dev (production ใช้ 0.1-0.2)
    zipkin:
      tracing:
        endpoint: http://localhost:9411/api/v2/spans
```

---

## 📊 3. Custom Metrics

```kotlin
// metrics/ApiMetrics.kt
package com.example.monitoring.metrics

import io.micrometer.core.instrument.Counter
import io.micrometer.core.instrument.Gauge
import io.micrometer.core.instrument.MeterRegistry
import io.micrometer.core.instrument.Timer
import org.springframework.stereotype.Component
import java.util.concurrent.atomic.AtomicInteger

@Component
class ApiMetrics(private val meterRegistry: MeterRegistry) {

    // Counter: นับจำนวนครั้ง
    private val registrationCounter: Counter = Counter.builder("user.registrations.total")
        .description("Total number of user registrations")
        .tag("source", "api")
        .register(meterRegistry)

    private val loginCounter: Counter = Counter.builder("user.logins.total")
        .description("Total number of user logins")
        .register(meterRegistry)

    private val failedLoginCounter: Counter = Counter.builder("user.logins.failed.total")
        .description("Total number of failed login attempts")
        .register(meterRegistry)

    // Gauge: ค่าปัจจุบัน (เปลี่ยนขึ้นลงได้)
    private val activeUsers = AtomicInteger(0)
    private val activeUsersGauge: Gauge = Gauge.builder("users.active.current", activeUsers) { it.toDouble() }
        .description("Current number of active users")
        .register(meterRegistry)

    // Timer: วัดเวลา
    private val orderProcessingTimer: Timer = Timer.builder("order.processing.duration")
        .description("Time taken to process an order")
        .publishPercentiles(0.5, 0.95, 0.99)
        .publishPercentileHistogram()
        .register(meterRegistry)

    // Counter สำหรับแต่ละ payment method
    fun recordPaymentMethod(method: String) {
        meterRegistry.counter(
            "payments.total",
            "method", method,
            "status", "success"
        ).increment()
    }

    fun recordRegistration() = registrationCounter.increment()

    fun recordLogin(success: Boolean) {
        if (success) loginCounter.increment()
        else failedLoginCounter.increment()
    }

    fun setActiveUsers(count: Int) = activeUsers.set(count)

    fun recordOrderProcessing(block: () -> Unit) {
        orderProcessingTimer.record(block)
    }

    // Custom distribution summary (สำหรับ order amounts)
    fun recordOrderAmount(amount: Double) {
        meterRegistry.summary(
            "order.amount",
            "currency", "THB"
        ).record(amount)
    }
}
```

---

## 🔍 4. Distributed Tracing

```kotlin
// service/OrderService.kt
package com.example.monitoring.service

import com.example.monitoring.metrics.ApiMetrics
import io.micrometer.observation.Observation
import io.micrometer.observation.ObservationRegistry
import io.micrometer.tracing.Tracer
import org.slf4j.LoggerFactory
import org.springframework.stereotype.Service

@Service
class OrderService(
    private val apiMetrics: ApiMetrics,
    private val observationRegistry: ObservationRegistry,
    private val tracer: Tracer
) {
    private val logger = LoggerFactory.getLogger(javaClass)

    data class OrderRequest(
        val userId: Long,
        val items: List<String>,
        val totalAmount: Double
    )

    data class OrderResponse(
        val orderId: String,
        val status: String,
        val traceId: String?
    )

    fun createOrder(request: OrderRequest): OrderResponse {
        // ใช้ Observation API (ครอบทั้ง metrics + tracing)
        return Observation.createNotStarted("order.create", observationRegistry)
            .lowCardinalityKeyValue("user.tier", getUserTier(request.userId))
            .highCardinalityKeyValue("user.id", request.userId.toString())
            .observe {
                processOrder(request)
            }
    }

    private fun processOrder(request: OrderRequest): OrderResponse {
        val currentSpan = tracer.currentSpan()
        val traceId = currentSpan?.context()?.traceId()

        logger.info(
            "Processing order for user={}, items={}, traceId={}",
            request.userId, request.items.size, traceId
        )

        apiMetrics.recordOrderProcessing {
            // Simulate processing
            validateInventory(request)
            processPayment(request)
            sendConfirmation(request)
        }

        apiMetrics.recordOrderAmount(request.totalAmount)

        return OrderResponse(
            orderId = "ORD-${System.currentTimeMillis()}",
            status = "CONFIRMED",
            traceId = traceId
        )
    }

    private fun validateInventory(request: OrderRequest) {
        // Nested span สำหรับ sub-operation
        val newSpan = tracer.nextSpan().name("validate.inventory").start()
        try {
            Thread.sleep(10)  // simulate work
            logger.debug("Inventory validated for {} items", request.items.size)
        } finally {
            newSpan.end()
        }
    }

    private fun processPayment(request: OrderRequest) {
        val newSpan = tracer.nextSpan().name("process.payment").start()
        try {
            Thread.sleep(50)  // simulate payment processing
            logger.debug("Payment processed: THB {}", request.totalAmount)
        } finally {
            newSpan.end()
        }
    }

    private fun sendConfirmation(request: OrderRequest) {
        logger.debug("Confirmation sent to user {}", request.userId)
    }

    private fun getUserTier(userId: Long): String {
        return if (userId % 3 == 0L) "premium" else "standard"
    }
}
```

---

## 🏥 5. Custom Health Indicators

```kotlin
// health/DatabaseHealthIndicator.kt
package com.example.monitoring.health

import org.springframework.boot.actuate.health.Health
import org.springframework.boot.actuate.health.HealthIndicator
import org.springframework.jdbc.core.JdbcTemplate
import org.springframework.stereotype.Component

@Component
class DatabaseHealthIndicator(private val jdbcTemplate: JdbcTemplate) : HealthIndicator {

    override fun health(): Health {
        return try {
            val start = System.currentTimeMillis()
            jdbcTemplate.queryForObject("SELECT 1", Int::class.java)
            val elapsed = System.currentTimeMillis() - start

            if (elapsed > 1000) {
                Health.up()
                    .withDetail("status", "slow")
                    .withDetail("response_time_ms", elapsed)
                    .build()
            } else {
                Health.up()
                    .withDetail("response_time_ms", elapsed)
                    .build()
            }
        } catch (ex: Exception) {
            Health.down()
                .withException(ex)
                .withDetail("error", ex.message)
                .build()
        }
    }
}

// health/ExternalApiHealthIndicator.kt
@Component
class PaymentServiceHealthIndicator : HealthIndicator {

    override fun health(): Health {
        return try {
            // ทดสอบ connection ไปยัง payment service
            Health.up()
                .withDetail("payment_service", "available")
                .withDetail("url", "https://api.payment.example.com")
                .build()
        } catch (ex: Exception) {
            Health.down()
                .withDetail("payment_service", "unavailable")
                .withDetail("error", ex.message)
                .build()
        }
    }
}

// health/DiskSpaceHealthIndicator (Spring provides this, but custom version)
@Component
class CustomDiskSpaceHealthIndicator : HealthIndicator {

    override fun health(): Health {
        val file = java.io.File("/")
        val freeBytes = file.freeSpace
        val totalBytes = file.totalSpace
        val usedPercent = (1 - freeBytes.toDouble() / totalBytes) * 100

        return if (usedPercent > 90) {
            Health.down()
                .withDetail("used_percent", "%.1f%%".format(usedPercent))
                .withDetail("free_bytes", freeBytes)
                .build()
        } else {
            Health.up()
                .withDetail("used_percent", "%.1f%%".format(usedPercent))
                .withDetail("free_bytes", freeBytes)
                .build()
        }
    }
}
```

---

## 📈 6. Metrics Endpoint ที่สำคัญ

```kotlin
// controller/MetricsController.kt
package com.example.monitoring.controller

import io.micrometer.core.instrument.MeterRegistry
import org.springframework.http.ResponseEntity
import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.RequestMapping
import org.springframework.web.bind.annotation.RestController

@RestController
@RequestMapping("/api/internal/metrics")
class MetricsController(private val meterRegistry: MeterRegistry) {

    @GetMapping("/summary")
    fun getMetricsSummary(): ResponseEntity<Map<String, Any>> {
        val httpRequests = meterRegistry.find("http.server.requests").timer()
        val jvmMemory = meterRegistry.find("jvm.memory.used").gauge()

        return ResponseEntity.ok(
            mapOf(
                "http_requests" to mapOf(
                    "count" to (httpRequests?.count() ?: 0),
                    "mean_ms" to (httpRequests?.mean(java.util.concurrent.TimeUnit.MILLISECONDS) ?: 0.0)
                ),
                "jvm" to mapOf(
                    "memory_used_mb" to ((jvmMemory?.value() ?: 0.0) / (1024 * 1024))
                ),
                "uptime_seconds" to meterRegistry.find("process.uptime").timeGauge()?.value()
            )
        )
    }
}
```

---

## 🐋 7. Docker Compose Setup

```yaml
# docker-compose-monitoring.yml
version: '3.8'

services:
  app:
    build: .
    ports:
      - "8080:8080"
    environment:
      - SPRING_PROFILES_ACTIVE=docker
      - MANAGEMENT_ZIPKIN_TRACING_ENDPOINT=http://zipkin:9411/api/v2/spans
    depends_on:
      - prometheus
      - zipkin

  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./monitoring/prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus-data:/prometheus
    command:
      - --config.file=/etc/prometheus/prometheus.yml
      - --storage.tsdb.retention.time=15d
      - --web.enable-lifecycle
      - --web.enable-admin-api

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
      - GF_USERS_ALLOW_SIGN_UP=false
    volumes:
      - grafana-data:/var/lib/grafana
      - ./monitoring/grafana/provisioning:/etc/grafana/provisioning
    depends_on:
      - prometheus

  zipkin:
    image: openzipkin/zipkin:latest
    ports:
      - "9411:9411"

  alertmanager:
    image: prom/alertmanager:latest
    ports:
      - "9093:9093"
    volumes:
      - ./monitoring/alertmanager.yml:/etc/alertmanager/alertmanager.yml

volumes:
  prometheus-data:
  grafana-data:
```

```yaml
# monitoring/prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

alerting:
  alertmanagers:
    - static_configs:
        - targets: ['alertmanager:9093']

rule_files:
  - "alerts.yml"

scrape_configs:
  - job_name: 'spring-boot-app'
    metrics_path: '/actuator/prometheus'
    scrape_interval: 5s
    static_configs:
      - targets: ['app:8080']
    relabel_configs:
      - source_labels: [__address__]
        target_label: instance
```

```yaml
# monitoring/alerts.yml
groups:
  - name: spring-boot-alerts
    rules:
      # High error rate
      - alert: HighErrorRate
        expr: |
          (
            rate(http_server_requests_seconds_count{status=~"5.."}[5m]) /
            rate(http_server_requests_seconds_count[5m])
          ) > 0.05
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "High error rate detected"
          description: "Error rate is {{ $value | humanizePercentage }} for the last 5 minutes"

      # Slow response time
      - alert: SlowResponseTime
        expr: |
          histogram_quantile(0.95, 
            rate(http_server_requests_seconds_bucket[5m])
          ) > 1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Slow API response time"
          description: "P95 response time is {{ $value }}s"

      # High memory usage
      - alert: HighMemoryUsage
        expr: |
          (jvm_memory_used_bytes{area="heap"} / jvm_memory_max_bytes{area="heap"}) > 0.85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High JVM heap usage"
          description: "Heap usage is {{ $value | humanizePercentage }}"

      # Application down
      - alert: ApplicationDown
        expr: up{job="spring-boot-app"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Application is down"
          description: "Spring Boot application has been down for more than 1 minute"
```

---

## 📊 8. Grafana Dashboard Configuration

```json
// monitoring/grafana/dashboards/spring-boot-dashboard.json (บางส่วน)
{
  "dashboard": {
    "title": "Spring Boot Application Dashboard",
    "panels": [
      {
        "title": "Request Rate (RPS)",
        "type": "graph",
        "targets": [
          {
            "expr": "sum(rate(http_server_requests_seconds_count[1m])) by (uri)",
            "legendFormat": "{{uri}}"
          }
        ]
      },
      {
        "title": "P95 Response Time",
        "type": "graph",
        "targets": [
          {
            "expr": "histogram_quantile(0.95, sum(rate(http_server_requests_seconds_bucket[5m])) by (le, uri))",
            "legendFormat": "{{uri}}"
          }
        ]
      },
      {
        "title": "Error Rate",
        "type": "stat",
        "targets": [
          {
            "expr": "sum(rate(http_server_requests_seconds_count{status=~\"5..\"}[5m])) / sum(rate(http_server_requests_seconds_count[5m]))"
          }
        ]
      },
      {
        "title": "JVM Heap Usage",
        "type": "gauge",
        "targets": [
          {
            "expr": "jvm_memory_used_bytes{area=\"heap\"} / jvm_memory_max_bytes{area=\"heap\"}"
          }
        ]
      },
      {
        "title": "Active DB Connections",
        "type": "stat",
        "targets": [
          {
            "expr": "hikaricp_connections_active"
          }
        ]
      }
    ]
  }
}
```

---

## 📊 สรุปเนื้อหา

| Component | Tool | ประโยชน์ |
|-----------|------|---------|
| Metrics Collection | Micrometer | Abstraction layer |
| Metrics Storage | Prometheus | Time series DB |
| Visualization | Grafana | Dashboard |
| Distributed Tracing | Zipkin/Jaeger | Request tracing |
| Log Aggregation | ELK/Loki | Log search |
| Alerting | Alertmanager | Notifications |

### Actuator Endpoints ที่สำคัญ:

| Endpoint | ประโยชน์ |
|---------|---------|
| `/actuator/health` | Health status |
| `/actuator/metrics` | Metrics listing |
| `/actuator/prometheus` | Prometheus scrape endpoint |
| `/actuator/info` | App info |
| `/actuator/env` | Environment vars |
| `/actuator/loggers` | Log levels |
| `/actuator/threaddump` | Thread dump |
| `/actuator/heapdump` | Heap dump |

### Key Metrics to Monitor:

- **Availability**: HTTP 5xx rate
- **Latency**: P50, P95, P99 response times
- **Throughput**: Requests per second
- **Saturation**: DB connections, thread pool
- **Resource**: CPU, Memory, Disk

---

*Part 49/100+ | Kotlin & Spring Boot Complete Course*
