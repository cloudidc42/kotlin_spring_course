# Part 80: Mini Project Enterprise

## Enterprise SaaS API — โปรเจกต์จบ Phase 3

---

## 🎯 เป้าหมายของ Part นี้

- สร้าง Enterprise-grade SaaS API
- Multi-tenancy + Authentication
- Search + Caching + Messaging
- Full CI/CD pipeline
- Kubernetes deployment
- Architecture overview

---

## 🏗️ 1. Architecture Overview

```
                    ┌─────────────────────────────────────┐
                    │         Kubernetes Cluster           │
                    │                                     │
    Internet ──────►│  Ingress (nginx)                    │
                    │       ↓                             │
                    │  API Gateway (Spring Cloud Gateway) │
                    │       ↓                             │
                    │  ┌──────────┐  ┌──────────────────┐ │
                    │  │  Auth    │  │   Core API       │ │
                    │  │ Service  │  │  Spring Boot     │ │
                    │  └──────────┘  └────────┬─────────┘ │
                    │                         │           │
                    │  ┌──────────┐  ┌────────▼─────────┐ │
                    │  │ Search   │  │    PostgreSQL     │ │
                    │  │  (ES)    │  │  (StatefulSet)   │ │
                    │  └──────────┘  └──────────────────┘ │
                    │                                     │
                    │  ┌──────────┐  ┌──────────────────┐ │
                    │  │  Redis   │  │     Kafka        │ │
                    │  │ (Cache)  │  │  (Messaging)     │ │
                    │  └──────────┘  └──────────────────┘ │
                    └─────────────────────────────────────┘
```

---

## 📁 2. โครงสร้างโปรเจกต์

```
enterprise-saas/
├── src/main/kotlin/com/example/saas/
│   ├── SaasApplication.kt
│   ├── config/
│   │   ├── SecurityConfig.kt
│   │   ├── MultiTenantConfig.kt
│   │   ├── CacheConfig.kt
│   │   ├── KafkaConfig.kt
│   │   └── ElasticsearchConfig.kt
│   ├── tenant/
│   │   ├── TenantContext.kt
│   │   ├── TenantFilter.kt
│   │   ├── TenantRepository.kt
│   │   └── TenantService.kt
│   ├── auth/
│   │   ├── JwtService.kt
│   │   ├── AuthController.kt
│   │   └── UserDetailsService.kt
│   ├── product/
│   │   ├── Product.kt
│   │   ├── ProductRepository.kt
│   │   ├── ProductService.kt
│   │   ├── ProductController.kt
│   │   └── ProductDocument.kt (ES)
│   ├── order/
│   │   ├── Order.kt
│   │   ├── OrderService.kt
│   │   └── OrderController.kt
│   ├── notification/
│   │   ├── NotificationService.kt
│   │   └── OrderEventConsumer.kt
│   └── analytics/
│       ├── AnalyticsService.kt
│       └── AnalyticsController.kt
├── src/main/resources/
│   ├── application.yml
│   ├── application-kubernetes.yml
│   └── i18n/
│       ├── messages.properties
│       └── messages_th.properties
├── k8s/
│   ├── namespace.yaml
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── hpa.yaml
│   └── configmap.yaml
├── helm/
│   └── enterprise-saas/
│       ├── Chart.yaml
│       ├── values.yaml
│       └── templates/
├── .github/
│   └── workflows/
│       ├── ci.yml
│       └── deploy.yml
└── docker-compose.yml
```

---

## ⚡ 3. Core Application

```kotlin
@SpringBootApplication
@EnableCaching
@EnableAsync
@EnableScheduling
class SaasApplication

fun main(args: Array<String>) {
    runApplication<SaasApplication>(*args)
}
```

### Security Configuration

```kotlin
@Configuration
@EnableWebSecurity
@EnableMethodSecurity
class SecurityConfig(
    private val jwtAuthFilter: JwtAuthFilter,
    private val tenantFilter: TenantFilter
) {

    @Bean
    fun securityFilterChain(http: HttpSecurity): SecurityFilterChain {
        return http
            .csrf { it.disable() }
            .sessionManagement { it.sessionCreationPolicy(SessionCreationPolicy.STATELESS) }
            .authorizeHttpRequests { auth ->
                auth.requestMatchers("/api/v1/auth/**").permitAll()
                auth.requestMatchers("/actuator/health").permitAll()
                auth.requestMatchers("/api/v1/admin/**").hasRole("SUPER_ADMIN")
                auth.anyRequest().authenticated()
            }
            .addFilterBefore(tenantFilter, UsernamePasswordAuthenticationFilter::class.java)
            .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter::class.java)
            .build()
    }
}
```

---

## 🏪 4. Product Service กับ Cache และ Search

```kotlin
@Service
@Transactional
class ProductService(
    private val productRepository: ProductRepository,
    private val searchService: ProductSearchService,
    private val cacheManager: CacheManager,
    private val eventPublisher: ApplicationEventPublisher,
    private val messageService: MessageService
) {

    @Cacheable(value = ["products"], key = "#id + ':' + T(com.example.TenantContext).getCurrentTenant()")
    fun findById(id: Long): Product {
        return productRepository.findByIdAndTenantId(id, TenantContext.getCurrentTenant())
            ?: throw ProductNotFoundException(
                messageService.getMessage("product.not.found", id)
            )
    }

    @CacheEvict(value = ["products"], key = "#result.id + ':' + T(com.example.TenantContext).getCurrentTenant()")
    fun create(request: CreateProductRequest): Product {
        val product = productRepository.save(
            Product(
                name = request.name,
                price = request.price,
                description = request.description,
                category = request.category,
                stock = request.stock,
                tenantId = TenantContext.getCurrentTenant()
            )
        )
        eventPublisher.publishEvent(ProductCreatedEvent(product))
        return product
    }

    @CacheEvict(value = ["products"], key = "#id + ':' + T(com.example.TenantContext).getCurrentTenant()")
    fun update(id: Long, request: UpdateProductRequest): Product {
        val product = findById(id)
        val updated = product.copy(
            name = request.name ?: product.name,
            price = request.price ?: product.price,
            description = request.description ?: product.description
        )
        val saved = productRepository.save(updated)
        eventPublisher.publishEvent(ProductUpdatedEvent(saved))
        return saved
    }

    fun search(query: ProductSearchRequest): ProductSearchResponse {
        return searchService.search(query.copy(
            tenantId = TenantContext.getCurrentTenant()
        ))
    }
}
```

---

## 📦 5. Order Service กับ Kafka

```kotlin
@Service
@Transactional
class OrderService(
    private val orderRepository: OrderRepository,
    private val inventoryService: InventoryService,
    private val kafkaTemplate: KafkaTemplate<String, OrderEvent>,
    private val messageService: MessageService
) {

    fun placeOrder(userId: Long, request: PlaceOrderRequest): Order {
        // ตรวจสอบและ reserve inventory
        inventoryService.reserveItems(request.items.map { it.productId to it.quantity })

        val order = orderRepository.save(
            Order(
                userId = userId,
                tenantId = TenantContext.getCurrentTenant(),
                items = request.items.map { item ->
                    OrderItem(
                        productId = item.productId,
                        quantity = item.quantity,
                        price = item.price
                    )
                },
                status = OrderStatus.CONFIRMED,
                total = request.items.sumOf { it.price * it.quantity }
            )
        )

        // ส่ง event ไป Kafka
        kafkaTemplate.send(
            "order-events",
            order.id.toString(),
            OrderEvent(
                orderId = order.id,
                userId = userId,
                tenantId = order.tenantId,
                status = order.status.name,
                total = order.total,
                timestamp = System.currentTimeMillis()
            )
        )

        return order
    }
}
```

---

## 📊 6. Analytics Controller

```kotlin
@RestController
@RequestMapping("/api/v1/analytics")
@PreAuthorize("hasRole('ADMIN')")
class AnalyticsController(
    private val analyticsService: AnalyticsService
) {

    @GetMapping("/dashboard")
    fun getDashboard(
        @RequestParam(defaultValue = "30") days: Int
    ): ResponseEntity<DashboardResponse> {
        val tenantId = TenantContext.getCurrentTenant()
        val dashboard = analyticsService.getDashboard(tenantId, days)
        return ResponseEntity.ok(dashboard)
    }

    @GetMapping("/orders/trend")
    fun getOrderTrend(
        @RequestParam @DateTimeFormat(iso = DateTimeFormat.ISO.DATE) from: java.time.LocalDate,
        @RequestParam @DateTimeFormat(iso = DateTimeFormat.ISO.DATE) to: java.time.LocalDate
    ): ResponseEntity<List<DailyOrderStats>> {
        val tenantId = TenantContext.getCurrentTenant()
        return ResponseEntity.ok(analyticsService.getOrderTrend(tenantId, from, to))
    }
}

data class DashboardResponse(
    val totalRevenue: Double,
    val orderCount: Long,
    val activeUsers: Long,
    val topProducts: List<ProductStats>,
    val revenueByCategory: Map<String, Double>,
    val period: String
)
```

---

## 🚀 7. Kubernetes Deployment

```yaml
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: enterprise-saas
  namespace: saas-production
  labels:
    app: enterprise-saas
    version: "1.0.0"
spec:
  replicas: 3
  selector:
    matchLabels:
      app: enterprise-saas
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: enterprise-saas
        version: "1.0.0"
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
        prometheus.io/path: "/actuator/prometheus"
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchLabels:
                    app: enterprise-saas
                topologyKey: kubernetes.io/hostname
      containers:
        - name: enterprise-saas
          image: ghcr.io/company/enterprise-saas:1.0.0
          ports:
            - containerPort: 8080
          env:
            - name: SPRING_PROFILES_ACTIVE
              value: kubernetes
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: db-password
            - name: JWT_SECRET
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: jwt-secret
            - name: KAFKA_BOOTSTRAP_SERVERS
              value: "kafka-service:9092"
            - name: ELASTICSEARCH_URIS
              value: "http://elasticsearch-service:9200"
            - name: REDIS_HOST
              value: "redis-service"
          resources:
            requests:
              memory: "512Mi"
              cpu: "250m"
            limits:
              memory: "1Gi"
              cpu: "1000m"
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
            failureThreshold: 3
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            initialDelaySeconds: 60
            periodSeconds: 30
```

---

## 🔄 8. Full CI/CD Pipeline Summary

```yaml
# .github/workflows/full-pipeline.yml
# CI: test → lint → build → docker → push
# CD: deploy-staging → smoke-test → approval → deploy-production → health-check → notify
```

---

## 📋 9. Feature Checklist

```
Enterprise SaaS API Features:
✅ Multi-tenancy (schema-per-tenant)
✅ JWT Authentication + Refresh Tokens  
✅ Role-based Access Control (RBAC)
✅ Product CRUD + Full-text Search (Elasticsearch)
✅ Order Management + Inventory Control
✅ Redis Caching (L2 cache)
✅ Kafka Event Streaming
✅ Async Notifications
✅ Real-time Analytics
✅ Internationalization (Thai/English)
✅ Rate Limiting per Tenant
✅ API Versioning
✅ Comprehensive Error Handling
✅ Health Checks + Metrics (Prometheus)
✅ Distributed Tracing (Zipkin)
✅ Docker + Kubernetes Deployment
✅ Helm Charts
✅ GitHub Actions CI/CD
✅ Automated Testing (Unit + Integration)
✅ API Documentation (OpenAPI/Swagger)
```

---

## 📊 10. สรุปตาราง Phase 3 — Enterprise Patterns

| Pattern | ใช้ใน | เครื่องมือ |
|---------|------|---------|
| Multi-tenancy | SaaS isolation | Schema routing |
| CQRS | Read/Write separation | Spring Events |
| Event Sourcing | Audit trail | Kafka |
| Circuit Breaker | Resilience | Resilience4j |
| Bulkhead | Resource isolation | Thread pools |
| Saga | Distributed transactions | Kafka + Outbox |
| CQRS + ES | Complex domains | Axon Framework |
| Feature Flags | Gradual rollout | Flipt/Unleash |
| Blue-Green Deploy | Zero downtime | Kubernetes |
| GitOps | Declarative deployments | Argo CD |

---

## 🎓 สรุป Phase 3 (Part 51-80)

Phase 3 ครอบคลุม:

1. **Microservices** — Service mesh, inter-service communication
2. **Advanced Patterns** — CQRS, Event Sourcing, Saga
3. **Infrastructure** — Kubernetes, Helm, GitOps
4. **Data Engineering** — Elasticsearch, Kafka Streams
5. **AI/ML Integration** — LLM APIs, RAG, Feature engineering
6. **Enterprise Patterns** — Multi-tenancy, i18n, Advanced testing
7. **DevOps** — CI/CD, Performance profiling, Security

---

## 🚀 Phase 4 Preview (Part 81-100)

ใน Part ต่อไป:
- **GraphQL API** — Flexible querying
- **WebSocket** — Real-time features
- **gRPC** — High-performance internal APIs
- **Reactive Programming** — WebFlux
- **Advanced Security** — OAuth2, Zero Trust
- **Observability** — OpenTelemetry
- **Final Capstone Project** — Full production system

---

## 🔐 11. JWT กับ Multi-tenant

```kotlin
@Service
class JwtService(
    @Value("\${app.jwt.secret}") private val jwtSecret: String,
    @Value("\${app.jwt.expiration:86400}") private val expiration: Long
) {

    fun generateToken(user: User, tenantId: String): String {
        return Jwts.builder()
            .setSubject(user.id.toString())
            .claim("email", user.email)
            .claim("role", user.role)
            .claim("tenantId", tenantId)  // เพิ่ม tenantId ใน token
            .setIssuedAt(java.util.Date())
            .setExpiration(java.util.Date(System.currentTimeMillis() + expiration * 1000))
            .signWith(getSignKey(), SignatureAlgorithm.HS256)
            .compact()
    }

    fun extractTenantId(token: String): String {
        return extractClaim(token) { it.get("tenantId", String::class.java) }
    }

    private fun getSignKey(): javax.crypto.SecretKey {
        return io.jsonwebtoken.security.Keys.hmacShaKeyFor(
            java.util.Base64.getDecoder().decode(jwtSecret)
        )
    }

    private fun <T> extractClaim(token: String, resolver: (Claims) -> T): T {
        val claims = Jwts.parserBuilder()
            .setSigningKey(getSignKey())
            .build()
            .parseClaimsJws(token)
            .body
        return resolver(claims)
    }
}
```

---

## 📈 12. Application Configuration

```yaml
# application-kubernetes.yml
spring:
  datasource:
    url: jdbc:postgresql://${DB_HOST:postgres-service}:5432/${DB_NAME:saas_db}
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
    hikari:
      maximum-pool-size: 20
  
  redis:
    host: ${REDIS_HOST:redis-service}
    port: 6379
    timeout: 2000ms
  
  kafka:
    bootstrap-servers: ${KAFKA_BOOTSTRAP_SERVERS:kafka-service:9092}
    producer:
      retries: 3
      acks: all
    consumer:
      auto-offset-reset: earliest
      enable-auto-commit: false

  elasticsearch:
    uris: ${ELASTICSEARCH_URIS:http://elasticsearch-service:9200}

app:
  jwt:
    secret: ${JWT_SECRET}
    expiration: 86400  # 24 hours
  
  multitenancy:
    enabled: true
    default-tenant: system
  
  cache:
    ttl: 3600
    max-size: 10000

management:
  endpoints:
    web:
      exposure:
        include: health,metrics,prometheus,info
  metrics:
    export:
      prometheus:
        enabled: true
```

---

*Part 80/100+ | Kotlin & Spring Boot Complete Course*
