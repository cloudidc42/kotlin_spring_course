# Part 89: System Design Patterns
## ออกแบบ System ระดับ Production

---

## 🎯 เป้าหมายของ Part นี้

- Load Balancing strategies
- Caching patterns
- Database patterns
- API patterns
- Fault tolerance
- ตัวอย่าง: Design Twitter-scale system

---

## 📖 1. Load Balancing

### Load Balancing Algorithms

```
Round Robin:
  Request 1 → Server A
  Request 2 → Server B
  Request 3 → Server C
  Request 4 → Server A (วนซ้ำ)
  เหมาะกับ: servers ที่มี capacity เท่ากัน

Weighted Round Robin:
  Server A weight=3, Server B weight=1
  A, A, A, B, A, A, A, B ...
  เหมาะกับ: servers ที่มี capacity ต่างกัน

Least Connections:
  ส่งไปที่ server ที่มี active connections น้อยที่สุด
  เหมาะกับ: requests ที่ใช้เวลาต่างกัน (long-running tasks)

IP Hash:
  hash(client_ip) % num_servers
  เหมาะกับ: stateful sessions (user ไปที่ server เดิมเสมอ)
```

### Spring Boot กับ Load Balancer

```kotlin
// Nginx Load Balancer Configuration
// nginx.conf
upstream spring_backends {
    # Round robin (default)
    server backend1:8080;
    server backend2:8080;
    server backend3:8080;
    
    # หรือ Least Connections
    # least_conn;
    
    # หรือ IP Hash (sticky sessions)
    # ip_hash;
    
    keepalive 32;
}

server {
    listen 80;
    
    location /api/ {
        proxy_pass http://spring_backends;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
    
    # Health check
    location /health {
        proxy_pass http://spring_backends/actuator/health;
    }
}
```

```kotlin
// Spring Cloud LoadBalancer (Microservices)
@Configuration
class LoadBalancerConfig {
    @Bean
    @LoadBalanced
    fun restTemplate(): RestTemplate = RestTemplate()
}

@Service
class OrderService(
    @LoadBalanced
    private val restTemplate: RestTemplate
) {
    fun getProduct(productId: Long): Product {
        // "product-service" จะถูก resolve ด้วย load balancer
        return restTemplate.getForObject(
            "http://product-service/api/products/$productId",
            Product::class.java
        )!!
    }
}
```

---

## 🗄️ 2. Caching Patterns

### Cache-Aside (Lazy Loading)

```kotlin
@Service
class ProductService(
    private val productRepository: ProductRepository,
    private val redisTemplate: RedisTemplate<String, Product>
) {
    private val cacheTTL = Duration.ofMinutes(30)

    fun getProduct(id: Long): Product {
        val cacheKey = "product:$id"

        // 1. Check cache first
        val cached = redisTemplate.opsForValue().get(cacheKey)
        if (cached != null) {
            return cached  // Cache hit!
        }

        // 2. Cache miss - get from DB
        val product = productRepository.findById(id)
            .orElseThrow { NotFoundException("Product $id not found") }

        // 3. Store in cache
        redisTemplate.opsForValue().set(cacheKey, product, cacheTTL)

        return product
    }

    fun updateProduct(id: Long, product: Product): Product {
        val updated = productRepository.save(product)

        // Invalidate cache
        redisTemplate.delete("product:$id")

        return updated
    }
}
```

### Write-Through Cache

```kotlin
@Service
class UserService(
    private val userRepository: UserRepository,
    private val cache: Cache
) {
    // Write to both cache and DB simultaneously
    fun updateUser(id: Long, user: User): User {
        val updated = userRepository.save(user)

        // Always update cache on write
        cache.put("user:$id", updated)

        return updated
    }

    fun getUser(id: Long): User {
        // Always hit cache first
        return cache.get("user:$id") as? User
            ?: userRepository.findById(id)
                .orElseThrow { NotFoundException("User not found") }
    }
}
```

### Read-Through with Spring Cache

```kotlin
@Service
class ProductCacheService(
    private val productRepository: ProductRepository
) {
    @Cacheable(value = ["products"], key = "#id")
    fun getProduct(id: Long): Product {
        println("Cache miss - loading from DB for id: $id")
        return productRepository.findById(id)
            .orElseThrow { NotFoundException("Product $id not found") }
    }

    @CachePut(value = ["products"], key = "#result.id")
    fun updateProduct(id: Long, product: Product): Product {
        return productRepository.save(product)
    }

    @CacheEvict(value = ["products"], key = "#id")
    fun deleteProduct(id: Long) {
        productRepository.deleteById(id)
    }

    @CacheEvict(value = ["products"], allEntries = true)
    fun clearAllProductCache() {
        println("Clearing all product caches")
    }
}
```

---

## 🗃️ 3. Database Patterns

### Read Replicas

```yaml
# application.yml - Multiple DataSources
spring:
  datasource:
    primary:
      url: jdbc:postgresql://primary-db:5432/mydb
      username: ${DB_USER}
      password: ${DB_PASS}
    replica:
      url: jdbc:postgresql://replica-db:5432/mydb
      username: ${DB_USER_RO}
      password: ${DB_PASS_RO}
```

```kotlin
@Configuration
class DataSourceConfig {

    @Primary
    @Bean("primaryDataSource")
    fun primaryDataSource(): DataSource = DataSourceBuilder.create()
        .url("jdbc:postgresql://primary:5432/mydb")
        .build()

    @Bean("replicaDataSource")
    fun replicaDataSource(): DataSource = DataSourceBuilder.create()
        .url("jdbc:postgresql://replica:5432/mydb")
        .build()
}

// Routing DataSource
@Component
class RoutingDataSource(
    @Qualifier("primaryDataSource") private val primary: DataSource,
    @Qualifier("replicaDataSource") private val replica: DataSource
) : AbstractRoutingDataSource() {

    override fun determineCurrentLookupKey(): Any {
        return if (TransactionSynchronizationManager.isCurrentTransactionReadOnly()) {
            "REPLICA"
        } else {
            "PRIMARY"
        }
    }
}

// ใช้งาน
@Service
@Transactional(readOnly = true)  // ← ไปที่ replica
class ProductQueryService(private val productRepository: ProductRepository) {
    fun getAllProducts() = productRepository.findAll()
}

@Service
@Transactional  // ← ไปที่ primary
class ProductCommandService(private val productRepository: ProductRepository) {
    fun createProduct(product: Product) = productRepository.save(product)
}
```

### Database Sharding (ระดับ Advanced)

```kotlin
// Horizontal sharding by user_id
@Component
class ShardRouter {
    private val shardCount = 4

    fun getShardId(userId: Long): Int = (userId % shardCount).toInt()

    fun getDataSource(userId: Long): DataSource {
        val shardId = getShardId(userId)
        return dataSourcePool[shardId]
    }
}
```

---

## 🔄 4. CQRS Pattern

```
CQRS = Command Query Responsibility Segregation
แยก operation อ่าน (Query) ออกจากการเขียน (Command)

Write side: PostgreSQL (normalized, ACID)
Read side: Elasticsearch / Redis (denormalized, fast)
```

```kotlin
// Command Side
data class CreateOrderCommand(
    val userId: Long,
    val items: List<OrderItem>
)

@Service
class OrderCommandService(
    private val orderRepository: OrderRepository,
    private val eventPublisher: ApplicationEventPublisher
) {
    fun handle(command: CreateOrderCommand): Long {
        val order = Order(
            userId = command.userId,
            items = command.items,
            status = "PENDING"
        )
        val saved = orderRepository.save(order)

        // Publish event สำหรับ read side
        eventPublisher.publishEvent(OrderCreatedEvent(saved))

        return saved.id
    }
}

// Query Side (Read Model)
@Document(indexName = "orders")
data class OrderDocument(
    @Id val id: String,
    val userId: Long,
    val userName: String,
    val items: List<OrderItemView>,
    val total: Double,
    val status: String,
    val createdAt: Instant
)

@Repository
interface OrderSearchRepository : ElasticsearchRepository<OrderDocument, String>

@Service
class OrderQueryService(
    private val orderSearchRepository: OrderSearchRepository
) {
    fun findOrdersByUser(userId: Long): List<OrderDocument> =
        orderSearchRepository.findByUserId(userId)

    fun searchOrders(query: String): List<OrderDocument> =
        orderSearchRepository.findByItemsNameContaining(query)
}

// Event Handler - sync write side to read side
@Component
class OrderEventHandler(
    private val orderSearchRepository: OrderSearchRepository
) {
    @EventListener
    fun on(event: OrderCreatedEvent) {
        val document = OrderDocument(
            id = event.order.id.toString(),
            userId = event.order.userId,
            userName = event.order.user.name,
            items = event.order.items.map { OrderItemView(it.name, it.quantity, it.price) },
            total = event.order.total,
            status = event.order.status,
            createdAt = event.order.createdAt
        )
        orderSearchRepository.save(document)
    }
}
```

---

## 🛡️ 5. Fault Tolerance Patterns

### Bulkhead Pattern

```kotlin
// แยก thread pools สำหรับ different services
// ป้องกัน 1 service ที่ slow ทำให้ service อื่น slow ด้วย

@Configuration
class ThreadPoolConfig {

    @Bean("orderThreadPool")
    fun orderExecutor(): Executor = ThreadPoolTaskExecutor().apply {
        corePoolSize = 10
        maxPoolSize = 20
        queueCapacity = 50
        setThreadNamePrefix("order-")
        initialize()
    }

    @Bean("paymentThreadPool")
    fun paymentExecutor(): Executor = ThreadPoolTaskExecutor().apply {
        corePoolSize = 5
        maxPoolSize = 10
        queueCapacity = 25
        setThreadNamePrefix("payment-")
        initialize()
    }
}

@Service
class OrderService(
    @Qualifier("orderThreadPool") private val executor: Executor
) {
    fun processOrderAsync(order: Order): CompletableFuture<OrderResult> {
        return CompletableFuture.supplyAsync({ processOrder(order) }, executor)
    }
}
```

---

## 🐦 6. ออกแบบ Twitter-Scale System

### Requirements

```
ผู้ใช้: 300 million active users/day
Tweets: 500 million tweets/day
Read/Write ratio: 100:1 (mostly reads)
Storage: ต้องเก็บ tweets ตลอดไป
Latency: < 200ms สำหรับ home timeline
```

### High-Level Design

```
[Client] → [CDN] → [Load Balancer]
                         ↓
              [API Gateway / Rate Limiter]
                    /    |    \
            [User]  [Tweet] [Timeline]
            Service  Service  Service
               |        |        |
           [PostgreSQL] [Cassandra] [Redis Cache]
                           |
                     [Fanout Service]
                    /             \
            [Push Model]     [Pull Model]
            (active users)   (celebrities)
```

### Tweet Service

```kotlin
@Entity
data class Tweet(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long = 0,
    val userId: Long,
    val content: String,  // max 280 chars
    val mediaUrls: List<String> = emptyList(),
    val replyToId: Long? = null,
    val retweetId: Long? = null,
    val createdAt: Instant = Instant.now()
)

@Service
class TweetService(
    private val tweetRepository: TweetRepository,
    private val timelineService: TimelineService,
    private val fanoutService: FanoutService
) {
    fun createTweet(userId: Long, content: String): Tweet {
        require(content.length <= 280) { "Tweet too long" }

        val tweet = tweetRepository.save(
            Tweet(userId = userId, content = content)
        )

        // Fanout to followers' timelines
        fanoutService.fanout(tweet)

        return tweet
    }
}
```

### Timeline Fanout

```kotlin
@Service
class FanoutService(
    private val followerService: FollowerService,
    private val redisTemplate: RedisTemplate<String, Long>
) {
    companion object {
        const val CELEBRITY_THRESHOLD = 1_000_000  // 1M followers
        const val TIMELINE_MAX_SIZE = 800L
    }

    fun fanout(tweet: Tweet) {
        val followers = followerService.getFollowerIds(tweet.userId)

        if (followers.size > CELEBRITY_THRESHOLD) {
            // Pull model for celebrities - don't fanout
            // Followers ต้อง pull เอง
            return
        }

        // Push model - fanout to all followers
        followers.forEach { followerId ->
            val timelineKey = "timeline:$followerId"
            redisTemplate.opsForZSet().add(
                timelineKey,
                tweet.id,
                tweet.createdAt.epochSecond.toDouble()
            )
            // Keep only latest 800 tweets
            redisTemplate.opsForZSet().removeRange(timelineKey, 0, -TIMELINE_MAX_SIZE - 1)
        }
    }
}

@Service
class TimelineService(
    private val redisTemplate: RedisTemplate<String, Long>,
    private val tweetRepository: TweetRepository,
    private val followerService: FollowerService
) {
    fun getHomeTimeline(userId: Long, page: Int = 0, size: Int = 20): List<Tweet> {
        val timelineKey = "timeline:$userId"
        val start = (page * size).toLong()
        val end = start + size - 1

        // Get tweet IDs from timeline cache
        val tweetIds = redisTemplate.opsForZSet()
            .reverseRange(timelineKey, start, end)
            ?: emptySet()

        // Include tweets from celebrities (pull model)
        val celebrityTweets = getCelebrityTweets(userId)

        // Merge and sort
        val allTweetIds = (tweetIds + celebrityTweets.map { it.id }).distinct()

        return tweetRepository.findAllById(allTweetIds)
            .sortedByDescending { it.createdAt }
            .take(size)
    }

    private fun getCelebrityTweets(userId: Long): List<Tweet> {
        val followings = followerService.getFollowingIds(userId)
        val celebrities = followings.filter {
            followerService.getFollowerCount(it) > FanoutService.CELEBRITY_THRESHOLD
        }
        return celebrities.flatMap { celebId ->
            tweetRepository.findByUserIdOrderByCreatedAtDesc(celebId)
                .take(20)
        }
    }
}
```

---

## 📋 สรุป

| Pattern | วัตถุประสงค์ | เมื่อใช้ |
|---------|------------|---------|
| Load Balancing | กระจาย traffic | เมื่อ horizontal scale |
| Cache-Aside | Reduce DB load | Read-heavy workload |
| Write-Through | Cache always fresh | Write + immediate read |
| Read Replica | Scale reads | 80%+ queries are reads |
| CQRS | Separate read/write | Complex domains |
| Bulkhead | Isolate failures | Multiple external services |
| Fanout | Distribute updates | Social media timelines |

---

*Part 89/100+ | Kotlin & Spring Boot Complete Course*
