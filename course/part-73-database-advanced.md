# Part 73: Advanced Database

## Advanced Database — Concurrency, Sharding, Replicas สำหรับ High-traffic Systems

---

## 🎯 เป้าหมายของ Part นี้

- Stored procedures กับ JPA
- Database sharding concepts
- Read replicas
- Optimistic vs Pessimistic locking
- Advanced connection pooling
- สร้าง High-concurrency inventory system

---

## 📖 1. Stored Procedures กับ JPA

### สร้าง Stored Procedure

```sql
-- PostgreSQL stored procedure
CREATE OR REPLACE PROCEDURE process_order(
    p_user_id BIGINT,
    p_product_ids BIGINT[],
    p_quantities INTEGER[],
    OUT p_order_id BIGINT,
    OUT p_total DECIMAL(10,2)
)
LANGUAGE plpgsql AS $$
DECLARE
    v_product RECORD;
    v_index INTEGER;
BEGIN
    -- สร้าง order
    INSERT INTO orders (user_id, status, created_at)
    VALUES (p_user_id, 'PENDING', NOW())
    RETURNING id INTO p_order_id;
    
    p_total := 0;
    
    -- เพิ่ม order items
    FOR v_index IN 1..array_length(p_product_ids, 1) LOOP
        SELECT * INTO v_product FROM products WHERE id = p_product_ids[v_index];
        
        -- ตรวจสอบ stock
        IF v_product.stock < p_quantities[v_index] THEN
            RAISE EXCEPTION 'Insufficient stock for product %', p_product_ids[v_index];
        END IF;
        
        -- เพิ่ม order item
        INSERT INTO order_items (order_id, product_id, quantity, price)
        VALUES (p_order_id, p_product_ids[v_index], p_quantities[v_index], v_product.price);
        
        -- อัพเดต stock
        UPDATE products SET stock = stock - p_quantities[v_index]
        WHERE id = p_product_ids[v_index];
        
        p_total := p_total + (v_product.price * p_quantities[v_index]);
    END LOOP;
    
    -- อัพเดต order total
    UPDATE orders SET total = p_total WHERE id = p_order_id;
    
    COMMIT;
END;
$$;
```

### เรียก Stored Procedure ด้วย JPA

```kotlin
import jakarta.persistence.*

@Entity
@Table(name = "orders")
@NamedStoredProcedureQuery(
    name = "Order.processOrder",
    procedureName = "process_order",
    parameters = [
        StoredProcedureParameter(name = "p_user_id", mode = ParameterMode.IN, type = Long::class),
        StoredProcedureParameter(name = "p_product_ids", mode = ParameterMode.IN, type = Array<Long>::class),
        StoredProcedureParameter(name = "p_quantities", mode = ParameterMode.IN, type = Array<Int>::class),
        StoredProcedureParameter(name = "p_order_id", mode = ParameterMode.OUT, type = Long::class),
        StoredProcedureParameter(name = "p_total", mode = ParameterMode.OUT, type = Double::class)
    ]
)
class Order(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long = 0,
    val userId: Long,
    var status: String = "PENDING",
    var total: Double = 0.0
)
```

```kotlin
@Service
@Transactional
class OrderService(
    @PersistenceContext private val entityManager: EntityManager
) {

    fun processOrder(userId: Long, items: List<Pair<Long, Int>>): OrderResult {
        val productIds = items.map { it.first }.toLongArray()
        val quantities = items.map { it.second }.toIntArray()

        val proc = entityManager.createNamedStoredProcedureQuery("Order.processOrder")
        proc.setParameter("p_user_id", userId)
        proc.setParameter("p_product_ids", productIds)
        proc.setParameter("p_quantities", quantities)

        proc.execute()

        val orderId = proc.getOutputParameterValue("p_order_id") as Long
        val total = proc.getOutputParameterValue("p_total") as Double

        return OrderResult(orderId = orderId, total = total)
    }
}
```

---

## 🔀 2. Database Sharding Concepts

### Horizontal Sharding Strategy

```kotlin
// Shard Router
class ShardRouter(private val shardCount: Int = 4) {

    fun getShardId(userId: Long): Int {
        return (userId % shardCount).toInt()
    }

    fun getShardId(key: String): Int {
        return Math.abs(key.hashCode() % shardCount)
    }
}

// DataSource Router
@Component
class ShardDataSourceRouter(
    private val shardRouter: ShardRouter,
    private val dataSources: Map<Int, DataSource>
) {

    fun getDataSource(userId: Long): DataSource {
        val shardId = shardRouter.getShardId(userId)
        return dataSources[shardId] 
            ?: throw IllegalStateException("No datasource for shard $shardId")
    }
}
```

### Consistent Hashing

```kotlin
import java.util.TreeMap

class ConsistentHashRouter(private val virtualNodes: Int = 150) {
    private val ring = TreeMap<Long, String>()

    fun addNode(node: String) {
        repeat(virtualNodes) { i ->
            val hash = hash("$node-$i")
            ring[hash] = node
        }
    }

    fun removeNode(node: String) {
        repeat(virtualNodes) { i ->
            ring.remove(hash("$node-$i"))
        }
    }

    fun getNode(key: String): String? {
        if (ring.isEmpty()) return null
        val hash = hash(key)
        val entry = ring.ceilingEntry(hash) ?: ring.firstEntry()
        return entry?.value
    }

    private fun hash(key: String): Long {
        var h = 0L
        for (c in key) {
            h = h * 31 + c.code
        }
        return h
    }
}
```

---

## 📚 3. Read Replicas

### Spring Boot Multi-DataSource

```kotlin
@Configuration
class DataSourceConfig(
    @Value("\${db.primary.url}") private val primaryUrl: String,
    @Value("\${db.replica.url}") private val replicaUrl: String
) {

    @Bean(name = ["primaryDataSource"])
    @Primary
    fun primaryDataSource() = HikariDataSource(HikariConfig().apply {
        jdbcUrl = primaryUrl
        username = System.getenv("DB_USERNAME")
        password = System.getenv("DB_PASSWORD")
        maximumPoolSize = 20
        minimumIdle = 5
        poolName = "PrimaryPool"
    })

    @Bean(name = ["replicaDataSource"])
    fun replicaDataSource() = HikariDataSource(HikariConfig().apply {
        jdbcUrl = replicaUrl
        username = System.getenv("DB_USERNAME")
        password = System.getenv("DB_PASSWORD")
        maximumPoolSize = 30  // replica รับ read load มากกว่า
        minimumIdle = 10
        poolName = "ReplicaPool"
        isReadOnly = true
    })
}
```

### Transaction Routing

```kotlin
enum class DataSourceType { PRIMARY, REPLICA }

class DataSourceContextHolder {
    companion object {
        private val context = ThreadLocal<DataSourceType>()
        
        fun set(type: DataSourceType) = context.set(type)
        fun get(): DataSourceType = context.get() ?: DataSourceType.PRIMARY
        fun clear() = context.remove()
    }
}

@Target(AnnotationTarget.FUNCTION, AnnotationTarget.CLASS)
@Retention(AnnotationRetention.RUNTIME)
annotation class ReadOnly

@Aspect
@Component
class ReadOnlyRoutingAspect {
    
    @Around("@annotation(ReadOnly)")
    fun routeToReplica(joinPoint: ProceedingJoinPoint): Any? {
        DataSourceContextHolder.set(DataSourceType.REPLICA)
        return try {
            joinPoint.proceed()
        } finally {
            DataSourceContextHolder.clear()
        }
    }
}

@Service
class ProductService(private val productRepository: ProductRepository) {
    
    @ReadOnly
    fun findAllProducts(): List<Product> {
        return productRepository.findAll() // ไปที่ replica
    }

    @Transactional
    fun createProduct(request: CreateProductRequest): Product {
        return productRepository.save(request.toEntity()) // ไปที่ primary
    }
}
```

---

## 🔒 4. Optimistic vs Pessimistic Locking

### Optimistic Locking

```kotlin
@Entity
@Table(name = "inventory")
data class Inventory(
    @Id
    val productId: Long,
    
    var quantity: Int,
    
    @Version
    var version: Long = 0  // JPA version field
)

@Service
@Transactional
class InventoryService(private val inventoryRepository: InventoryRepository) {

    fun decrementStock(productId: Long, quantity: Int): Boolean {
        return try {
            val inventory = inventoryRepository.findById(productId)
                .orElseThrow { RuntimeException("Product not found") }
            
            if (inventory.quantity < quantity) {
                throw InsufficientStockException("Not enough stock")
            }
            
            inventory.quantity -= quantity
            inventoryRepository.save(inventory)
            true
        } catch (ex: OptimisticLockingFailureException) {
            // retry logic
            false
        }
    }

    // Retry wrapper
    @Retryable(
        value = [OptimisticLockingFailureException::class],
        maxAttempts = 3,
        backoff = Backoff(delay = 100)
    )
    fun decrementStockWithRetry(productId: Long, quantity: Int) {
        decrementStock(productId, quantity)
    }
}
```

### Pessimistic Locking

```kotlin
@Repository
interface InventoryRepository : JpaRepository<Inventory, Long> {

    // Lock row สำหรับ update
    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("SELECT i FROM Inventory i WHERE i.productId = :productId")
    fun findByProductIdWithLock(@Param("productId") productId: Long): Inventory?

    // Lock หลาย rows
    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("SELECT i FROM Inventory i WHERE i.productId IN :productIds")
    fun findByProductIdsWithLock(@Param("productIds") productIds: List<Long>): List<Inventory>
}

@Service
@Transactional
class InventoryServicePessimistic(private val inventoryRepository: InventoryRepository) {

    fun processOrderItems(items: List<OrderItem>): Boolean {
        // Lock ทุก product ที่ต้องการพร้อมกัน (เพื่อหลีกเลี่ยง deadlock ต้อง lock ตาม order)
        val sortedProductIds = items.map { it.productId }.sorted()
        val inventories = inventoryRepository.findByProductIdsWithLock(sortedProductIds)
            .associateBy { it.productId }

        for (item in items) {
            val inventory = inventories[item.productId]
                ?: throw RuntimeException("Product ${item.productId} not found")
            
            if (inventory.quantity < item.quantity) {
                throw InsufficientStockException("Insufficient stock for ${item.productId}")
            }
            
            inventory.quantity -= item.quantity
        }

        inventoryRepository.saveAll(inventories.values)
        return true
    }
}
```

---

## 🔗 5. Advanced Connection Pooling

```kotlin
@Configuration
class HikariConfig {

    @Bean
    @Primary
    fun dataSource(): DataSource = HikariDataSource(
        com.zaxxer.hikari.HikariConfig().apply {
            jdbcUrl = "jdbc:postgresql://localhost:5432/myapp"
            username = "user"
            password = "password"
            
            // Pool settings
            maximumPoolSize = 20
            minimumIdle = 5
            connectionTimeout = 30_000L       // 30 seconds
            idleTimeout = 600_000L            // 10 minutes
            maxLifetime = 1_800_000L          // 30 minutes
            keepaliveTime = 60_000L           // 1 minute
            
            // Performance
            addDataSourceProperty("cachePrepStmts", "true")
            addDataSourceProperty("prepStmtCacheSize", "250")
            addDataSourceProperty("prepStmtCacheSqlLimit", "2048")
            addDataSourceProperty("useServerPrepStmts", "true")
            
            // Monitoring
            metricRegistry = prometheusMeterRegistry()
            healthCheckRegistry = healthCheckRegistry()
        }
    )
}
```

---

## 🏭 6. High-Concurrency Inventory System

```kotlin
@RestController
@RequestMapping("/api/v1/inventory")
class InventoryController(private val inventoryService: InventoryService) {

    @PostMapping("/reserve")
    fun reserveItems(@RequestBody request: ReservationRequest): ResponseEntity<ReservationResponse> {
        val result = inventoryService.reserveWithOptimisticLocking(
            items = request.items.map { ItemRequest(it.productId, it.quantity) }
        )
        return ResponseEntity.ok(result)
    }
}

@Service
class InventoryService(
    private val inventoryRepository: InventoryRepository,
    private val reservationRepository: ReservationRepository
) {

    @Transactional(isolation = Isolation.READ_COMMITTED)
    @Retryable(
        value = [OptimisticLockingFailureException::class, PessimisticLockingFailureException::class],
        maxAttempts = 5,
        backoff = Backoff(delay = 50, multiplier = 2.0, maxDelay = 1000)
    )
    fun reserveWithOptimisticLocking(items: List<ItemRequest>): ReservationResponse {
        val productIds = items.map { it.productId }.sorted() // sort to prevent deadlock
        val inventories = inventoryRepository.findAllByProductIdIn(productIds)
            .associateBy { it.productId }

        // Validate all items
        val errors = items.mapNotNull { item ->
            val inv = inventories[item.productId]
            when {
                inv == null -> "Product ${item.productId} not found"
                inv.quantity < item.quantity -> "Insufficient stock for ${item.productId}: available=${inv.quantity}, requested=${item.quantity}"
                else -> null
            }
        }

        if (errors.isNotEmpty()) {
            throw InsufficientStockException(errors.joinToString(", "))
        }

        // Apply reservations
        items.forEach { item ->
            inventories[item.productId]!!.quantity -= item.quantity
        }

        inventoryRepository.saveAll(inventories.values)

        val reservation = reservationRepository.save(
            Reservation(
                items = items.map { ReservationItem(it.productId, it.quantity) },
                status = "RESERVED",
                expiresAt = LocalDateTime.now().plusMinutes(15)
            )
        )

        return ReservationResponse(
            reservationId = reservation.id,
            status = "RESERVED",
            expiresAt = reservation.expiresAt
        )
    }
}
```

---

## 📊 7. สรุปตาราง Locking Strategies

| Strategy | เมื่อใช้ | ข้อดี | ข้อเสีย |
|----------|---------|-------|---------|
| Optimistic | Low contention | ไม่บล็อก reads | Retry ถ้า conflict |
| Pessimistic | High contention | Guaranteed isolation | Block concurrent access |
| SELECT FOR UPDATE | Critical sections | Explicit control | ต้อง manage timeout |
| Stored Procedure | Complex operations | Atomic, performant | ยาก maintain |
| Queue (Redis) | Flash sales | No DB locking needed | Complexity เพิ่ม |

---

## 💡 Best Practices

1. **Prefer Optimistic** สำหรับ low-contention scenarios
2. **Sort IDs ก่อน lock** เพื่อป้องกัน deadlock
3. **Set lock timeout** เพื่อป้องกัน infinite wait
4. **Monitor HikariCP metrics** — pool size, timeout, pending threads
5. **Read replicas สำหรับ reporting** queries ที่ไม่ต้องการ strong consistency

---

*Part 73/100+ | Kotlin & Spring Boot Complete Course*
