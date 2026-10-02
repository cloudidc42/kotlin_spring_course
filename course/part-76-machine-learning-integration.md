# Part 76: Machine Learning Integration

## Machine Learning Integration — เชื่อมต่อ ML Models กับ Spring Boot

---

## 🎯 เป้าหมายของ Part นี้

- เรียก ML models จาก Spring Boot
- REST API ไปยัง Python ML service
- Feature engineering
- A/B testing สำหรับ ML models
- สร้าง Product recommendation API

---

## 📖 1. Architecture Overview

```
Client Request
      ↓
Spring Boot API
      ↓
Feature Engineering
      ↓
ML Service (Python/FastAPI)
      ↓
Model Prediction
      ↓
Post-processing
      ↓
Response to Client
```

### ML Service Patterns

- **Embedded**: บรรจุ model ไว้ใน JVM (ONNX, DJL)
- **Sidecar**: ML container อยู่ใน pod เดียวกัน
- **Microservice**: แยก ML service ออกมาเฉพาะ
- **Cloud API**: เรียก cloud ML services (AWS SageMaker, GCP Vertex AI)

---

## 🐍 2. Python ML Service (FastAPI)

```python
# ml_service/main.py
from fastapi import FastAPI
from pydantic import BaseModel
import numpy as np
from sklearn.preprocessing import StandardScaler
import joblib
import pandas as pd

app = FastAPI(title="ML Recommendation Service")

# Load models at startup
scaler = joblib.load("models/scaler.pkl")
recommendation_model = joblib.load("models/recommendation_model.pkl")

class UserFeatures(BaseModel):
    user_id: str
    age: int
    purchase_history: list[str]
    browsing_history: list[str]
    total_spent: float
    last_purchase_days_ago: int

class RecommendationResponse(BaseModel):
    user_id: str
    recommended_product_ids: list[str]
    scores: list[float]
    model_version: str

@app.post("/recommend", response_model=RecommendationResponse)
async def recommend_products(features: UserFeatures):
    # Feature engineering
    feature_vector = engineer_features(features)
    
    # Scale features
    scaled = scaler.transform([feature_vector])
    
    # Get predictions
    scores = recommendation_model.predict_proba(scaled)[0]
    top_indices = np.argsort(scores)[-10:][::-1]
    
    product_ids = [str(idx) for idx in top_indices]
    product_scores = [float(scores[i]) for i in top_indices]
    
    return RecommendationResponse(
        user_id=features.user_id,
        recommended_product_ids=product_ids,
        scores=product_scores,
        model_version="v1.2.0"
    )

def engineer_features(features: UserFeatures) -> list:
    return [
        features.age,
        len(features.purchase_history),
        len(features.browsing_history),
        features.total_spent,
        features.last_purchase_days_ago,
        min(features.total_spent / max(len(features.purchase_history), 1), 10000),  # avg order value
        1 if features.last_purchase_days_ago < 30 else 0,  # active user flag
    ]
```

---

## ☕ 3. Spring Boot ML Client

### ML Client Interface

```kotlin
import com.fasterxml.jackson.annotation.JsonProperty

data class UserFeatures(
    @JsonProperty("user_id") val userId: String,
    val age: Int,
    @JsonProperty("purchase_history") val purchaseHistory: List<String>,
    @JsonProperty("browsing_history") val browsingHistory: List<String>,
    @JsonProperty("total_spent") val totalSpent: Double,
    @JsonProperty("last_purchase_days_ago") val lastPurchaseDaysAgo: Int
)

data class RecommendationResponse(
    @JsonProperty("user_id") val userId: String,
    @JsonProperty("recommended_product_ids") val recommendedProductIds: List<String>,
    val scores: List<Double>,
    @JsonProperty("model_version") val modelVersion: String
)
```

### WebClient สำหรับเรียก ML Service

```kotlin
import org.springframework.stereotype.Component
import org.springframework.web.reactive.function.client.WebClient
import org.springframework.web.reactive.function.client.awaitBody
import reactor.util.retry.Retry
import java.time.Duration

@Component
class MLServiceClient(
    private val mlServiceWebClient: WebClient
) {

    suspend fun getRecommendations(features: UserFeatures): RecommendationResponse {
        return mlServiceWebClient
            .post()
            .uri("/recommend")
            .bodyValue(features)
            .retrieve()
            .awaitBody<RecommendationResponse>()
    }

    suspend fun getRecommendationsWithFallback(features: UserFeatures): RecommendationResponse {
        return try {
            getRecommendations(features)
        } catch (ex: Exception) {
            // Fallback to popular products
            RecommendationResponse(
                userId = features.userId,
                recommendedProductIds = getPopularProducts(),
                scores = List(10) { 0.5 },
                modelVersion = "fallback"
            )
        }
    }

    private fun getPopularProducts(): List<String> {
        // Return hardcoded popular products as fallback
        return listOf("101", "205", "312", "418", "521", "630", "742", "856", "963", "1074")
    }
}

@Configuration
class MLServiceConfig {

    @Bean
    fun mlServiceWebClient(): WebClient = WebClient.builder()
        .baseUrl("http://ml-service:8000")
        .defaultHeader("Content-Type", "application/json")
        .codecs { it.defaultCodecs().maxInMemorySize(1024 * 1024) }
        .filter(RetryFilter())
        .build()
}
```

---

## 🔧 4. Feature Engineering ใน Kotlin

```kotlin
@Service
class FeatureEngineeringService(
    private val userRepository: UserRepository,
    private val orderRepository: OrderRepository,
    private val browsingRepository: BrowsingHistoryRepository
) {

    suspend fun buildUserFeatures(userId: Long): UserFeatures {
        val user = userRepository.findById(userId)
            ?: throw UserNotFoundException(userId)

        val recentOrders = orderRepository.findRecentByUserId(userId, limit = 50)
        val browsingHistory = browsingRepository.findRecentByUserId(userId, limit = 100)

        val totalSpent = recentOrders.sumOf { it.total }
        val purchaseHistory = recentOrders.flatMap { it.items.map { item -> item.productId.toString() } }
        val browsingProductIds = browsingHistory.map { it.productId.toString() }

        val lastPurchaseDaysAgo = recentOrders.maxByOrNull { it.createdAt }
            ?.let { java.time.temporal.ChronoUnit.DAYS.between(it.createdAt.toLocalDate(), java.time.LocalDate.now()).toInt() }
            ?: 999

        return UserFeatures(
            userId = userId.toString(),
            age = user.age,
            purchaseHistory = purchaseHistory.distinct().take(20),
            browsingHistory = browsingProductIds.distinct().take(30),
            totalSpent = totalSpent,
            lastPurchaseDaysAgo = lastPurchaseDaysAgo
        )
    }
}
```

---

## 🧪 5. A/B Testing สำหรับ ML Models

```kotlin
enum class ModelVariant { CONTROL, TREATMENT_A, TREATMENT_B }

@Service
class ABTestingService(
    private val userRepository: UserRepository,
    private val metricsService: MetricsService
) {

    fun assignVariant(userId: Long): ModelVariant {
        // Deterministic assignment based on userId
        return when (userId % 10) {
            in 0..6L -> ModelVariant.CONTROL      // 70%
            in 7..8L -> ModelVariant.TREATMENT_A  // 20%
            else -> ModelVariant.TREATMENT_B       // 10%
        }
    }
}

@Service
class RecommendationRouter(
    private val abTestingService: ABTestingService,
    private val mlServiceClient: MLServiceClient,
    private val featureEngineeringService: FeatureEngineeringService,
    private val metricsService: MetricsService
) {

    suspend fun getRecommendations(userId: Long): List<ProductResponse> {
        val variant = abTestingService.assignVariant(userId)
        val startTime = System.currentTimeMillis()

        return try {
            val recommendations = when (variant) {
                ModelVariant.CONTROL -> getControlRecommendations(userId)
                ModelVariant.TREATMENT_A -> getMLRecommendations(userId, "model-a")
                ModelVariant.TREATMENT_B -> getMLRecommendations(userId, "model-b")
            }

            metricsService.recordMLLatency(
                variant = variant.name,
                durationMs = System.currentTimeMillis() - startTime,
                success = true
            )

            recommendations
        } catch (ex: Exception) {
            metricsService.recordMLLatency(
                variant = variant.name,
                durationMs = System.currentTimeMillis() - startTime,
                success = false
            )
            getFallbackRecommendations(userId)
        }
    }

    private suspend fun getMLRecommendations(userId: Long, modelEndpoint: String): List<ProductResponse> {
        val features = featureEngineeringService.buildUserFeatures(userId)
        val response = mlServiceClient.getRecommendations(features)
        return response.recommendedProductIds
            .mapNotNull { productId -> productRepository.findById(productId.toLong()) }
            .map { it.toResponse() }
    }

    private fun getControlRecommendations(userId: Long): List<ProductResponse> {
        // Simple rule-based recommendations
        return popularProductRepository.findTop10()
            .map { it.toResponse() }
    }

    private fun getFallbackRecommendations(userId: Long): List<ProductResponse> {
        return popularProductRepository.findTop10().map { it.toResponse() }
    }
}
```

---

## 📦 6. Model Versioning

```kotlin
@Entity
@Table(name = "ml_model_versions")
data class ModelVersion(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long = 0,
    val modelName: String,
    val version: String,
    val endpoint: String,
    val isActive: Boolean = false,
    val accuracy: Double? = null,
    val deployedAt: java.time.LocalDateTime? = null,
    val createdAt: java.time.LocalDateTime = java.time.LocalDateTime.now()
)

@Service
class ModelVersionService(
    private val modelVersionRepository: ModelVersionRepository
) {

    fun getActiveEndpoint(modelName: String): String {
        return modelVersionRepository
            .findFirstByModelNameAndIsActiveOrderByDeployedAtDesc(modelName, true)
            ?.endpoint
            ?: throw IllegalStateException("No active model found for $modelName")
    }

    @Transactional
    fun promote(modelName: String, version: String) {
        modelVersionRepository.deactivateAll(modelName)
        val model = modelVersionRepository.findByModelNameAndVersion(modelName, version)
            ?: throw IllegalArgumentException("Model version not found")
        model.copy(isActive = true, deployedAt = java.time.LocalDateTime.now())
            .let { modelVersionRepository.save(it) }
    }
}
```

---

## 📊 7. สรุปตาราง ML Integration Patterns

| Pattern | ข้อดี | ข้อเสีย | เมื่อใช้ |
|---------|-------|---------|---------|
| Embedded (ONNX) | ไม่มี network hop | จำกัด frameworks | Simple models |
| REST API (Python) | ยืดหยุ่น | Network latency | Complex models |
| gRPC | เร็วกว่า REST | Complexity | High-throughput |
| Message Queue | Async, decoupled | ไม่ real-time | Batch scoring |
| Cloud ML APIs | Managed, scalable | ค่าใช้จ่ายสูง | Complex use cases |

---

## 💡 Best Practices

1. **Circuit breaker** สำหรับ ML service calls
2. **Fallback strategy** เสมอ — ML models ล้มเหลวได้
3. **Log predictions** สำหรับ monitoring และ debugging
4. **Feature store** เพื่อ consistency ระหว่าง training และ serving
5. **Shadow mode** — run new model แบบ shadow ก่อน fully deploy

---

## 🔁 8. Batch Scoring

บางครั้งเราต้องการ score ข้อมูลจำนวนมากในครั้งเดียว (batch scoring) แทนที่จะ score ทีละรายการ

```kotlin
@Service
class BatchScoringService(
    private val mlServiceClient: MLServiceClient,
    private val featureEngineeringService: FeatureEngineeringService
) {

    @Scheduled(cron = "0 0 * * * *") // ทุกชั่วโมง
    fun scorePendingUsers() {
        val userIds = userRepository.findActiveUserIds()
        val batchSize = 100

        userIds.chunked(batchSize).forEach { batch ->
            try {
                val features = batch.map { userId ->
                    runBlocking { featureEngineeringService.buildUserFeatures(userId) }
                }
                val scores = mlServiceClient.batchScore(features)
                scoreRepository.saveAll(scores)
            } catch (ex: Exception) {
                logger.error("Batch scoring failed for batch starting with ${batch.first()}", ex)
            }
        }
    }

    fun scoreNewUser(userId: Long) {
        // Score ใหม่ทันทีเมื่อ user สมัครสมาชิก
        val features = runBlocking { featureEngineeringService.buildUserFeatures(userId) }
        val score = mlServiceClient.score(features)
        scoreRepository.save(UserScore(userId = userId, score = score, scoredAt = java.time.Instant.now()))
    }
}
```

---

## 🧩 9. Feature Store Integration

Feature Store เก็บ feature ที่คำนวณแล้ว เพื่อใช้ซ้ำระหว่าง training และ serving

```kotlin
@Service
class FeatureStoreService(
    private val redisTemplate: RedisTemplate<String, String>,
    private val objectMapper: ObjectMapper
) {

    fun getOrComputeFeatures(userId: Long): UserFeatures {
        val cacheKey = "features:user:$userId"
        val cached = redisTemplate.opsForValue().get(cacheKey)

        if (cached != null) {
            return objectMapper.readValue(cached, UserFeatures::class.java)
        }

        val features = computeFeatures(userId)
        redisTemplate.opsForValue().set(
            cacheKey,
            objectMapper.writeValueAsString(features),
            java.time.Duration.ofHours(1)
        )
        return features
    }

    fun invalidateFeatures(userId: Long) {
        redisTemplate.delete("features:user:$userId")
    }

    private fun computeFeatures(userId: Long): UserFeatures {
        // คำนวณ features จาก raw data
        val user = userRepository.findById(userId).orElseThrow()
        val orders = orderRepository.findRecentByUserId(userId, 50)
        return UserFeatures(
            userId = userId.toString(),
            age = user.age,
            purchaseHistory = orders.map { it.id.toString() },
            browsingHistory = emptyList(),
            totalSpent = orders.sumOf { it.total },
            lastPurchaseDaysAgo = 0
        )
    }
}
```

---

## 📈 10. Model Performance Monitoring

```kotlin
@Service
class MLModelMonitor(private val meterRegistry: MeterRegistry) {

    fun recordPrediction(
        modelName: String,
        modelVersion: String,
        latencyMs: Long,
        success: Boolean
    ) {
        meterRegistry.timer(
            "ml.prediction.latency",
            "model", modelName,
            "version", modelVersion
        ).record(latencyMs, java.util.concurrent.TimeUnit.MILLISECONDS)

        meterRegistry.counter(
            "ml.prediction.count",
            "model", modelName,
            "success", success.toString()
        ).increment()
    }

    fun recordPredictionQuality(modelName: String, actualLabel: String, predictedLabel: String) {
        val correct = actualLabel == predictedLabel
        meterRegistry.counter(
            "ml.prediction.quality",
            "model", modelName,
            "correct", correct.toString()
        ).increment()
    }
}
```

---

## 🐳 11. ML Service Docker Setup

```dockerfile
# ml_service/Dockerfile
FROM python:3.11-slim

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .
RUN python -c "import joblib; model = joblib.load('models/recommendation_model.pkl'); print('Model loaded')"

EXPOSE 8000
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000", "--workers", "4"]
```

```yaml
# docker-compose.yml additions
services:
  ml-service:
    build: ./ml_service
    ports:
      - "8000:8000"
    volumes:
      - ./ml_service/models:/app/models:ro
    environment:
      - MODEL_VERSION=v1.2.0
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
```

---

*Part 76/100+ | Kotlin & Spring Boot Complete Course*
