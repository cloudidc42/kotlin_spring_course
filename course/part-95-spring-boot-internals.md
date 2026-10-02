# Part 95: Spring Boot Internals
## เข้าใจ Spring Boot จากภายใน

---

## 🎯 เป้าหมายของ Part นี้

- Auto-configuration mechanism
- Bean lifecycle ในรายละเอียด
- @Conditional annotations
- Spring AOT internals
- สร้าง Custom Spring Boot Starter

---

## 📖 1. Auto-configuration Mechanism

### Spring Boot Startup Sequence

```
1. SpringApplication.run() เรียก
2. SpringFactoriesLoader โหลด META-INF/spring/factories
3. AutoConfigurationImportSelector เลือก @Configuration classes
4. @Conditional เงื่อนไข evaluate
5. Beans ที่ผ่านเงื่อนไขถูกสร้าง
6. ApplicationContext refresh สมบูรณ์
```

### ดู Auto-configuration ที่ทำงาน

```bash
# เปิด debug mode
./gradlew bootRun --args='--debug'

# หรือใน application.properties
debug=true

# Output จะแสดง:
# Positive matches:
#   DataSourceAutoConfiguration matched:
#     - @ConditionalOnClass found required class 'javax.sql.DataSource'
#
# Negative matches:
#   ActiveMQAutoConfiguration:
#     - @ConditionalOnClass did not find required class 'javax.jms.ConnectionFactory'
```

---

## 🔧 2. @Conditional Annotations

### Built-in @Conditional Annotations

```kotlin
// @ConditionalOnClass - เปิดเมื่อ class อยู่ใน classpath
@Configuration
@ConditionalOnClass(DataSource::class)
class DataSourceAutoConfiguration {
    @Bean
    @ConditionalOnMissingBean  // สร้างเฉพาะเมื่อยังไม่มี DataSource bean
    fun dataSource(): DataSource = // ...
}

// @ConditionalOnProperty - เปิดเมื่อ property มีค่าที่กำหนด
@Configuration
@ConditionalOnProperty(
    name = ["app.cache.enabled"],
    havingValue = "true",
    matchIfMissing = false
)
class CacheAutoConfiguration {
    @Bean
    fun cacheManager(): CacheManager = // ...
}

// @ConditionalOnBean / @ConditionalOnMissingBean
@Bean
@ConditionalOnBean(EmailService::class)  // เปิดเฉพาะเมื่อมี EmailService bean
fun emailNotificationService(emailService: EmailService): NotificationService {
    return EmailNotificationService(emailService)
}

@Bean
@ConditionalOnMissingBean(NotificationService::class)  // fallback
fun noopNotificationService(): NotificationService {
    return NoopNotificationService()
}

// @ConditionalOnWebApplication
@Configuration
@ConditionalOnWebApplication(type = ConditionalOnWebApplication.Type.SERVLET)
class WebMvcAutoConfiguration { }

// Custom @Conditional
class OnProductionEnvironmentCondition : Condition {
    override fun matches(context: ConditionContext, metadata: AnnotatedTypeMetadata): Boolean {
        val activeProfiles = context.environment.activeProfiles
        return "production" in activeProfiles
    }
}

@Target(AnnotationTarget.CLASS, AnnotationTarget.FUNCTION)
@Retention(AnnotationRetention.RUNTIME)
@Conditional(OnProductionEnvironmentCondition::class)
annotation class ConditionalOnProduction
```

---

## 🔄 3. Bean Lifecycle

```kotlin
// Bean Lifecycle phases:
// 1. Instantiation
// 2. Property population (DI)
// 3. BeanNameAware.setBeanName()
// 4. BeanFactoryAware.setBeanFactory()
// 5. ApplicationContextAware.setApplicationContext()
// 6. BeanPostProcessor.postProcessBeforeInitialization()
// 7. @PostConstruct (InitializingBean.afterPropertiesSet)
// 8. Custom init-method
// 9. BeanPostProcessor.postProcessAfterInitialization()
// 10. Bean ready for use
// 11. @PreDestroy (DisposableBean.destroy)
// 12. Custom destroy-method

@Service
class DatabaseConnectionPool : InitializingBean, DisposableBean {
    private var connections: List<Connection> = emptyList()

    // Phase 7: @PostConstruct
    @PostConstruct
    fun init() {
        println("@PostConstruct: Initializing connection pool")
        connections = (1..10).map { createConnection() }
    }

    // Phase 7 alternative: InitializingBean interface
    override fun afterPropertiesSet() {
        println("afterPropertiesSet: Additional initialization")
    }

    // Phase 11: @PreDestroy
    @PreDestroy
    fun cleanup() {
        println("@PreDestroy: Closing connections")
        connections.forEach { it.close() }
    }

    // Phase 11 alternative: DisposableBean interface
    override fun destroy() {
        println("destroy: Final cleanup")
    }

    private fun createConnection(): Connection {
        return DriverManager.getConnection("jdbc:h2:mem:test")
    }
}

// BeanPostProcessor - ดัก bean initialization
@Component
class LoggingBeanPostProcessor : BeanPostProcessor {
    override fun postProcessBeforeInitialization(bean: Any, beanName: String): Any? {
        println("Before init: $beanName")
        return bean
    }

    override fun postProcessAfterInitialization(bean: Any, beanName: String): Any? {
        println("After init: $beanName")
        return bean  // อาจ return proxy แทน bean เดิม (Spring AOP ทำแบบนี้!)
    }
}
```

---

## 📦 4. สร้าง Custom Spring Boot Starter

### Starter Structure

```
my-starter/
├── my-starter-autoconfigure/     <- autoconfiguration module
│   ├── src/main/kotlin/
│   │   └── com/example/
│   │       ├── MyServiceAutoConfiguration.kt
│   │       └── MyServiceProperties.kt
│   └── src/main/resources/
│       └── META-INF/spring/
│           └── org.springframework.boot.autoconfigure.AutoConfiguration.imports
└── my-spring-boot-starter/       <- starter (empty, just dependencies)
    └── build.gradle.kts
```

### 1. Properties

```kotlin
// my-starter-autoconfigure/src/main/kotlin/com/example/MyServiceProperties.kt
package com.example

import org.springframework.boot.context.properties.ConfigurationProperties

@ConfigurationProperties(prefix = "my.service")
data class MyServiceProperties(
    var enabled: Boolean = true,
    var apiUrl: String = "https://api.example.com",
    var apiKey: String = "",
    var timeoutSeconds: Int = 30,
    var maxRetries: Int = 3
)
```

### 2. Service Interface & Implementation

```kotlin
// my-starter-autoconfigure/.../MyService.kt
interface MyService {
    fun getData(id: String): MyData
}

data class MyData(val id: String, val value: String)

class DefaultMyService(
    private val properties: MyServiceProperties
) : MyService {
    override fun getData(id: String): MyData {
        // Use properties.apiUrl, properties.apiKey
        return MyData(id = id, value = "fetched data")
    }
}
```

### 3. AutoConfiguration

```kotlin
// my-starter-autoconfigure/.../MyServiceAutoConfiguration.kt
package com.example

import org.springframework.boot.autoconfigure.AutoConfiguration
import org.springframework.boot.autoconfigure.condition.ConditionalOnClass
import org.springframework.boot.autoconfigure.condition.ConditionalOnMissingBean
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty
import org.springframework.boot.context.properties.EnableConfigurationProperties
import org.springframework.context.annotation.Bean

@AutoConfiguration
@ConditionalOnClass(MyService::class)
@ConditionalOnProperty(
    prefix = "my.service",
    name = ["enabled"],
    havingValue = "true",
    matchIfMissing = true
)
@EnableConfigurationProperties(MyServiceProperties::class)
class MyServiceAutoConfiguration(
    private val properties: MyServiceProperties
) {
    @Bean
    @ConditionalOnMissingBean(MyService::class)
    fun myService(): MyService = DefaultMyService(properties)
}
```

### 4. Register AutoConfiguration

```
# META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
com.example.MyServiceAutoConfiguration
```

### 5. Starter build.gradle.kts

```kotlin
// my-spring-boot-starter/build.gradle.kts
dependencies {
    api(project(":my-starter-autoconfigure"))
    // Auto-pulls dependencies needed
}
```

### 6. ใช้งานใน Application

```kotlin
// ใน build.gradle.kts ของ app
dependencies {
    implementation("com.example:my-spring-boot-starter:1.0.0")
}

// application.yml
my:
  service:
    enabled: true
    api-url: https://api.mycompany.com
    api-key: ${MY_API_KEY}
    timeout-seconds: 60

// ใน code
@Service
class MyController(
    private val myService: MyService  // Auto-injected!
) {
    @GetMapping("/data/{id}")
    fun getData(@PathVariable id: String) = myService.getData(id)
}
```

---

## 🚀 5. Spring AOT Processing

AOT (Ahead-of-Time) ใน Spring Boot 3.x เตรียม application ตอน build time เพื่อรองรับ GraalVM Native Image

```kotlin
// Spring AOT generates:
// 1. BeanDefinition registrations
// 2. Reflection hints
// 3. Resource hints
// 4. Proxy classes

// ดู generated AOT code
./gradlew processAot

// ดู files ใน
// build/generated/aotSources/
// build/generated/aotResources/

// Custom AOT hints
@Configuration
@ImportRuntimeHints(CustomAotHints::class)
class AppConfig {

    class CustomAotHints : RuntimeHintsRegistrar {
        override fun registerHints(hints: RuntimeHints, classLoader: ClassLoader?) {
            // Register for reflection
            hints.reflection()
                .registerType(MyDto::class.java) { it.withMembers() }

            // Register resources
            hints.resources()
                .registerPattern("templates/*.html")

            // Register JDK proxies
            hints.proxies()
                .registerJdkProxy(MyInterface::class.java)
        }
    }
}
```

---

## 🔍 6. Spring Internals Deep Dive

### ApplicationContext Hierarchy

```kotlin
// Parent context → Child contexts (ใน web app)
// ServletContext (root) ← parent
//   └── DispatcherServlet context ← child
//         └── RequestScope
//               └── SessionScope

// ดู beans ทั้งหมดใน context
@Component
class BeanInspector(private val context: ApplicationContext) {

    @PostConstruct
    fun inspect() {
        val beans = context.beanDefinitionNames
        println("Total beans: ${beans.size}")

        // Beans จาก auto-configuration
        val autoConfigBeans = beans.filter { name ->
            val bd = (context as ConfigurableApplicationContext)
                .beanFactory.getBeanDefinition(name)
            bd.factoryBeanName?.contains("AutoConfiguration") == true
        }
        println("Auto-configured beans: ${autoConfigBeans.size}")
    }
}
```

### Spring AOP Proxy

```kotlin
// Spring AOP สร้าง proxy รอบ @Transactional, @Cacheable ฯลฯ
// CGLIB proxy (default) - subclass bytecode generation
// JDK Dynamic Proxy - เฉพาะ interface

// ดู proxy type
@Service
@Transactional
class MyTransactionalService {
    fun doWork() {}
}

@Component
class ProxyInspector(private val service: MyTransactionalService) {
    @PostConstruct
    fun check() {
        println("Class: ${service::class.java.name}")
        // Output: Class: com.example.MyTransactionalService$$EnhancerBySpringCGLIB$$abc123
        println("Is CGLIB proxy: ${service::class.java.name.contains("CGLIB")}")
    }
}
```

---

## 📋 สรุป

| หัวข้อ | รายละเอียด |
|--------|-----------|
| Auto-configuration | SPI + @Conditional evaluation ตอน startup |
| Bean Lifecycle | 12 phases จาก instantiation ถึง destroy |
| @Conditional | เปิด/ปิด beans ตามเงื่อนไข |
| Custom Starter | autoconfigure module + starter module |
| Spring AOT | Compile-time analysis สำหรับ Native Image |
| AOP Proxy | CGLIB/JDK proxy รอบ @Transactional beans |

---

*Part 95/100+ | Kotlin & Spring Boot Complete Course*
