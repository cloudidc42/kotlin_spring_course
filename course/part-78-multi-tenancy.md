# Part 78: Multi-tenancy

## Multi-tenancy — สร้าง SaaS Application รองรับหลาย Tenant

---

## 🎯 เป้าหมายของ Part นี้

- Single vs Multi database strategy
- Schema-per-tenant
- Row-level security
- Tenant isolation
- สร้าง SaaS application

---

## 📖 1. Multi-tenancy Strategies

### 1. Single Database, Shared Schema (Row-level)
- ทุก tenant อยู่ใน database และ tables เดียวกัน
- แบ่งด้วย `tenant_id` column
- ง่ายที่สุด แต่ data isolation น้อยที่สุด

### 2. Single Database, Separate Schemas
- แต่ละ tenant มี schema ของตัวเอง
- สมดุลระหว่าง isolation และ resource usage
- แนะนำสำหรับ SaaS ส่วนใหญ่

### 3. Separate Databases
- แต่ละ tenant มี database ของตัวเอง
- Data isolation สูงสุด
- ค่าใช้จ่ายสูงสุด

```
                    Isolation | Cost | Complexity
Row-level:              ต่ำ   |  ต่ำ  |     ต่ำ
Schema-per-tenant:     กลาง   | กลาง |    กลาง
Separate DB:            สูง   |  สูง  |     สูง
```

---

## ⚙️ 2. Tenant Context

```kotlin
object TenantContext {
    private val currentTenant = ThreadLocal<String>()

    fun setCurrentTenant(tenantId: String) {
        currentTenant.set(tenantId)
    }

    fun getCurrentTenant(): String {
        return currentTenant.get()
            ?: throw TenantNotSetException("No tenant set in current context")
    }

    fun getCurrentTenantOrNull(): String? = currentTenant.get()

    fun clear() = currentTenant.remove()
}

class TenantNotSetException(message: String) : RuntimeException(message)
```

### Tenant Resolution

```kotlin
@Component
class TenantResolver {

    // จาก subdomain: tenant.example.com
    fun resolveFromSubdomain(request: HttpServletRequest): String? {
        val host = request.serverName
        val parts = host.split(".")
        return if (parts.size >= 3) parts[0] else null
    }

    // จาก header: X-Tenant-Id
    fun resolveFromHeader(request: HttpServletRequest): String? {
        return request.getHeader("X-Tenant-Id")
    }

    // จาก JWT token
    fun resolveFromJwt(request: HttpServletRequest): String? {
        val token = request.getHeader("Authorization")?.removePrefix("Bearer ")
        return token?.let { jwtService.extractTenantId(it) }
    }

    // จาก path: /api/{tenantId}/...
    fun resolveFromPath(request: HttpServletRequest): String? {
        val path = request.requestURI
        val regex = Regex("/api/([^/]+)/")
        return regex.find(path)?.groupValues?.get(1)
    }
}
```

---

## 🔒 3. Schema-per-Tenant Implementation

### Tenant DataSource Router

```kotlin
import org.springframework.jdbc.datasource.lookup.AbstractRoutingDataSource

class TenantRoutingDataSource : AbstractRoutingDataSource() {

    override fun determineCurrentLookupKey(): Any {
        return TenantContext.getCurrentTenantOrNull() ?: "default"
    }
}

@Configuration
class MultiTenantDataSourceConfig(
    private val tenantRepository: TenantRepository
) {

    @Bean
    fun dataSource(): DataSource {
        val routingDataSource = TenantRoutingDataSource()

        val tenantDataSources: Map<Any, Any> = tenantRepository.findAll()
            .associate { tenant ->
                tenant.id to createDataSourceForTenant(tenant)
            }

        routingDataSource.setTargetDataSources(tenantDataSources)
        routingDataSource.setDefaultTargetDataSource(defaultDataSource())
        routingDataSource.afterPropertiesSet()

        return routingDataSource
    }

    private fun createDataSourceForTenant(tenant: Tenant): DataSource {
        return HikariDataSource(HikariConfig().apply {
            jdbcUrl = "jdbc:postgresql://localhost:5432/saas_db?currentSchema=${tenant.schemaName}"
            username = System.getenv("DB_USERNAME")
            password = System.getenv("DB_PASSWORD")
            maximumPoolSize = 5  // เล็กลงต่อ tenant
            poolName = "tenant-${tenant.id}-pool"
        })
    }
}
```

### Schema Migration per Tenant

```kotlin
@Service
class TenantSchemaService(
    private val dataSource: DataSource,
    private val flywayConfig: FlywayConfig
) {

    fun createTenantSchema(tenantId: String): Boolean {
        return try {
            val jdbcTemplate = JdbcTemplate(dataSource)

            // สร้าง schema
            jdbcTemplate.execute("CREATE SCHEMA IF NOT EXISTS tenant_$tenantId")

            // รัน migrations สำหรับ schema ใหม่
            val flyway = Flyway.configure()
                .dataSource(dataSource)
                .schemas("tenant_$tenantId")
                .locations("classpath:db/tenant-migrations")
                .load()

            flyway.migrate()
            true
        } catch (ex: Exception) {
            false
        }
    }

    fun deleteTenantSchema(tenantId: String): Boolean {
        return try {
            val jdbcTemplate = JdbcTemplate(dataSource)
            jdbcTemplate.execute("DROP SCHEMA IF EXISTS tenant_$tenantId CASCADE")
            true
        } catch (ex: Exception) {
            false
        }
    }
}
```

---

## 🛡️ 4. Row-Level Security (PostgreSQL RLS)

```sql
-- เปิด RLS สำหรับ table
ALTER TABLE products ENABLE ROW LEVEL SECURITY;

-- สร้าง policy
CREATE POLICY tenant_isolation ON products
    USING (tenant_id = current_setting('app.current_tenant')::uuid);

-- สร้าง function เพื่อ set tenant
CREATE OR REPLACE FUNCTION set_tenant(tenant_id UUID)
RETURNS VOID AS $$
BEGIN
    PERFORM set_config('app.current_tenant', tenant_id::TEXT, TRUE);
END;
$$ LANGUAGE plpgsql;
```

### Spring Boot RLS Integration

```kotlin
@Component
class RLSInterceptor : HandlerInterceptor {

    private val jdbcTemplate: JdbcTemplate

    override fun preHandle(
        request: HttpServletRequest,
        response: HttpServletResponse,
        handler: Any
    ): Boolean {
        val tenantId = TenantContext.getCurrentTenantOrNull()
        if (tenantId != null) {
            jdbcTemplate.execute("SELECT set_tenant('$tenantId'::uuid)")
        }
        return true
    }

    override fun afterCompletion(
        request: HttpServletRequest,
        response: HttpServletResponse,
        handler: Any,
        ex: Exception?
    ) {
        TenantContext.clear()
    }
}
```

---

## 🏗️ 5. Multi-tenant Entity และ Repository

```kotlin
@MappedSuperclass
abstract class TenantAwareEntity {
    @Column(name = "tenant_id", nullable = false)
    var tenantId: String = ""

    @PrePersist
    fun setTenant() {
        tenantId = TenantContext.getCurrentTenant()
    }
}

@Entity
@Table(name = "products")
data class Product(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long = 0,
    val name: String,
    val price: Double
) : TenantAwareEntity()

@Repository
interface ProductRepository : JpaRepository<Product, Long> {

    // Spring Data auto-filters by tenantId
    fun findByTenantId(tenantId: String): List<Product>

    @Query("SELECT p FROM Product p WHERE p.tenantId = :#{T(com.example.TenantContext).getCurrentTenant()}")
    fun findAllForCurrentTenant(): List<Product>
}
```

---

## 🌐 6. Multi-tenant Filter

```kotlin
@Component
@Order(1)
class TenantFilter(
    private val tenantResolver: TenantResolver,
    private val tenantRepository: TenantRepository
) : OncePerRequestFilter() {

    override fun doFilterInternal(
        request: HttpServletRequest,
        response: HttpServletResponse,
        filterChain: FilterChain
    ) {
        val tenantId = tenantResolver.resolveFromHeader(request)
            ?: tenantResolver.resolveFromSubdomain(request)
            ?: run {
                filterChain.doFilter(request, response)
                return
            }

        // Validate tenant exists and is active
        val tenant = tenantRepository.findById(tenantId).orElse(null)
        if (tenant == null || !tenant.isActive) {
            response.sendError(HttpServletResponse.SC_UNAUTHORIZED, "Invalid or inactive tenant")
            return
        }

        try {
            TenantContext.setCurrentTenant(tenantId)
            filterChain.doFilter(request, response)
        } finally {
            TenantContext.clear()
        }
    }

    override fun shouldNotFilter(request: HttpServletRequest): Boolean {
        return request.requestURI.startsWith("/public/")
    }
}
```

---

## 🏢 7. Tenant Management

```kotlin
@RestController
@RequestMapping("/api/v1/tenants")
@PreAuthorize("hasRole('SUPER_ADMIN')")
class TenantManagementController(
    private val tenantService: TenantManagementService
) {

    @PostMapping
    fun createTenant(@RequestBody @Valid request: CreateTenantRequest): ResponseEntity<TenantResponse> {
        val tenant = tenantService.createTenant(request)
        return ResponseEntity.status(HttpStatus.CREATED).body(tenant)
    }

    @PutMapping("/{tenantId}/plan")
    fun upgradePlan(
        @PathVariable tenantId: String,
        @RequestBody request: UpgradePlanRequest
    ): ResponseEntity<TenantResponse> {
        val updated = tenantService.upgradePlan(tenantId, request.plan)
        return ResponseEntity.ok(updated)
    }

    @DeleteMapping("/{tenantId}")
    fun deleteTenant(@PathVariable tenantId: String): ResponseEntity<Void> {
        tenantService.deleteTenant(tenantId)
        return ResponseEntity.noContent().build()
    }
}

@Service
@Transactional
class TenantManagementService(
    private val tenantRepository: TenantRepository,
    private val schemaService: TenantSchemaService,
    private val billingService: BillingService
) {

    fun createTenant(request: CreateTenantRequest): TenantResponse {
        // สร้าง tenant record
        val tenant = tenantRepository.save(
            Tenant(
                id = generateTenantId(request.companyName),
                companyName = request.companyName,
                adminEmail = request.adminEmail,
                plan = request.plan,
                schemaName = "tenant_${generateTenantId(request.companyName)}",
                isActive = true
            )
        )

        // สร้าง schema และ run migrations
        schemaService.createTenantSchema(tenant.id)

        // Setup billing
        billingService.createSubscription(tenant.id, request.plan)

        return tenant.toResponse()
    }

    private fun generateTenantId(name: String): String {
        return name.lowercase().replace(Regex("[^a-z0-9]"), "_")
            .let { "${it}_${System.currentTimeMillis() % 10000}" }
    }
}
```

---

## 📊 8. สรุปตาราง Multi-tenancy Approaches

| Approach | Isolation | Cost/Tenant | Scalability | เมื่อใช้ |
|---------|----------|------------|------------|---------|
| Shared schema | ต่ำ | ต่ำมาก | สูง | Small SaaS, startups |
| Schema-per-tenant | กลาง | ต่ำ-กลาง | ดี | Mid-size SaaS |
| DB-per-tenant | สูงมาก | สูง | ซับซ้อน | Enterprise, regulated |
| Hybrid | ยืดหยุ่น | ยืดหยุ่น | ดี | Multi-tier SaaS |

---

## 💡 Best Practices

1. **Tenant ID ใน JWT** — ไม่ต้องทำ DB lookup ทุก request
2. **Cross-tenant query protection** — ทดสอบว่า tenant A ไม่เห็นข้อมูล tenant B
3. **Tenant-specific rate limiting** — แต่ละ tenant มี quota
4. **Soft delete** — อย่าลบข้อมูล tenant ทันที เก็บไว้ 30 วัน
5. **Tenant onboarding automation** — สร้าง schema, seed data, ส่ง welcome email

---

## 🧪 9. Testing Multi-tenancy

```kotlin
@SpringBootTest
class MultiTenancyTest {

    @Autowired
    private lateinit var productRepository: ProductRepository

    @Test
    fun `tenant A cannot see tenant B data`() {
        // สร้าง products สำหรับ tenant A
        TenantContext.setCurrentTenant("tenant_a")
        val productA = productRepository.save(Product(name = "Tenant A Product", price = 100.0))

        // สร้าง products สำหรับ tenant B
        TenantContext.setCurrentTenant("tenant_b")
        val productB = productRepository.save(Product(name = "Tenant B Product", price = 200.0))

        // Tenant A ควรเห็นเฉพาะ product ของตัวเอง
        TenantContext.setCurrentTenant("tenant_a")
        val tenantAProducts = productRepository.findAll()
        assertThat(tenantAProducts).hasSize(1)
        assertThat(tenantAProducts.first().name).isEqualTo("Tenant A Product")

        // Cleanup
        TenantContext.clear()
    }

    @Test
    fun `request without tenant header returns 401`() {
        // ทดสอบว่า requests ที่ไม่มี tenant header ถูก reject
        val mockMvc = MockMvcBuilders.webAppContextSetup(applicationContext).build()
        mockMvc.perform(
            MockMvcRequestBuilders.get("/api/v1/products")
                .header("Authorization", "Bearer validtoken")
                // ไม่มี X-Tenant-Id header
        ).andExpect(MockMvcResultMatchers.status().isUnauthorized)
    }
}
```

---

## 🔒 10. Tenant Subscription Plans

```kotlin
@Entity
@Table(name = "tenants")
data class Tenant(
    @Id
    val id: String,
    val companyName: String,
    val adminEmail: String,
    
    @Enumerated(EnumType.STRING)
    val plan: TenantPlan = TenantPlan.STARTER,
    
    val schemaName: String,
    val isActive: Boolean = true,
    val maxUsers: Int = 10,
    val maxProducts: Int = 100,
    val storageGb: Int = 1
)

enum class TenantPlan(
    val maxUsers: Int,
    val maxProducts: Int,
    val storageGb: Int,
    val features: Set<String>
) {
    STARTER(10, 100, 1, setOf("basic_search")),
    PROFESSIONAL(50, 10_000, 10, setOf("basic_search", "advanced_analytics", "api_access")),
    ENTERPRISE(Int.MAX_VALUE, Int.MAX_VALUE, 1000, setOf("basic_search", "advanced_analytics", "api_access", "custom_domain", "sso", "dedicated_support"))
}
```

### Plan Enforcement

```kotlin
@Aspect
@Component
class PlanEnforcementAspect(
    private val tenantRepository: TenantRepository,
    private val productRepository: ProductRepository
) {

    @Before("@annotation(requiresPlan)")
    fun checkPlanFeature(joinPoint: JoinPoint, requiresPlan: RequiresPlan) {
        val tenantId = TenantContext.getCurrentTenant()
        val tenant = tenantRepository.findById(tenantId)
            ?: throw TenantNotFoundException(tenantId)

        if (requiresPlan.feature !in tenant.plan.features) {
            throw FeatureNotAvailableException(
                "Feature '${requiresPlan.feature}' requires ${requiresPlan.minimumPlan} plan or higher"
            )
        }
    }

    @Before("execution(* com.example.ProductService.create(..))")
    fun checkProductLimit() {
        val tenantId = TenantContext.getCurrentTenant()
        val tenant = tenantRepository.findById(tenantId) ?: return
        val currentCount = productRepository.countByTenantId(tenantId)

        if (currentCount >= tenant.maxProducts) {
            throw PlanLimitExceededException(
                "Product limit (${tenant.maxProducts}) reached for plan ${tenant.plan}"
            )
        }
    }
}

@Target(AnnotationTarget.FUNCTION)
@Retention(AnnotationRetention.RUNTIME)
annotation class RequiresPlan(
    val feature: String,
    val minimumPlan: String = "PROFESSIONAL"
)
```

---

*Part 78/100+ | Kotlin & Spring Boot Complete Course*
