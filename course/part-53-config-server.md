# Part 53: Spring Cloud Config Server
## Centralized Configuration Management

---

## 🎯 เป้าหมายของ Part นี้

- ทำความเข้าใจ Config Server
- ตั้งค่า Config Server
- ตั้งค่า Config Clients
- Encryption/Decryption secrets
- Config refresh ไม่ต้อง restart
- ตัวอย่าง: Microservices shared configuration

---

## 🏗️ 1. Config Server Setup

```kotlin
// config-server/build.gradle.kts
dependencies {
    implementation("org.springframework.cloud:spring-cloud-config-server")
    implementation("org.springframework.cloud:spring-cloud-starter-netflix-eureka-client")
}

// ConfigServerApplication.kt
@SpringBootApplication
@EnableConfigServer
class ConfigServerApplication

fun main(args: Array<String>) {
    runApplication<ConfigServerApplication>(*args)
}
```

```yaml
# config-server/application.yml
server:
  port: 8888

spring:
  application:
    name: config-server
  cloud:
    config:
      server:
        git:
          uri: https://github.com/myorg/config-repo  # Git repo สำหรับ config
          search-paths: "{application}"               # ใช้ชื่อ app เป็น folder
          default-label: main
          clone-on-start: true
        
        # หรือใช้ local filesystem (development)
        # native:
        #   search-locations: classpath:/config

eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
```

---

## 📂 2. Config Repository Structure

```
config-repo/                        # Git repository
├── application.yml                 # Shared by all services
├── application-dev.yml             # Shared dev config
├── application-prod.yml            # Shared prod config
├── user-service/
│   ├── user-service.yml            # user-service default
│   ├── user-service-dev.yml
│   └── user-service-prod.yml
├── order-service/
│   ├── order-service.yml
│   └── order-service-prod.yml
└── api-gateway/
    └── api-gateway.yml
```

```yaml
# config-repo/application.yml (shared)
app:
  default-page-size: 20
  max-page-size: 100

logging:
  pattern:
    console: "%d{yyyy-MM-dd HH:mm:ss} [%thread] %-5level %logger{36} - %msg%n"

---
# config-repo/user-service/user-service.yml
app:
  security:
    jwt-expiration-ms: 3600000

spring:
  jpa:
    show-sql: false

---
# config-repo/user-service/user-service-dev.yml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/userdb
  jpa:
    show-sql: true
```

---

## 🔌 3. Config Client

```kotlin
// user-service/build.gradle.kts
dependencies {
    implementation("org.springframework.cloud:spring-cloud-starter-config")
    implementation("org.springframework.cloud:spring-cloud-starter-netflix-eureka-client")
    implementation("org.springframework.boot:spring-boot-starter-actuator")
}
```

```yaml
# user-service/application.yml
spring:
  application:
    name: user-service
  config:
    import: "optional:configserver:http://localhost:8888"
  profiles:
    active: dev

# user-service/bootstrap.yml (สำหรับ Spring Boot 2.x)
# Spring Boot 3.x ใช้ spring.config.import แทน
```

---

## 🔄 4. Dynamic Config Refresh

```kotlin
// ไม่ต้อง restart service เมื่อ config เปลี่ยน

// 1. เพิ่ม @RefreshScope ที่ bean ที่ใช้ config
@RestController
@RefreshScope  // reload เมื่อ config refresh
class FeatureController(
    private val appProperties: AppProperties
) {
    @GetMapping("/features")
    fun getFeatures(): Map<String, Boolean> = mapOf(
        "newUi" to appProperties.features.enableNewUi,
        "beta" to appProperties.features.enableBeta
    )
}

// 2. Trigger refresh (manual)
// POST http://user-service:8080/actuator/refresh
// หรือใช้ Spring Cloud Bus เพื่อ broadcast refresh ทุก service
```

```yaml
# application.yml
management:
  endpoints:
    web:
      exposure:
        include: refresh,health,info
```

```bash
# Refresh single service
curl -X POST http://user-service:8080/actuator/refresh

# Refresh all services (Spring Cloud Bus + Kafka)
curl -X POST http://config-server:8888/actuator/busrefresh
```

---

## 🔐 5. Encrypting Secrets

```yaml
# config-server/application.yml
encrypt:
  key: "my-encryption-key-must-be-32-chars-long"
  # หรือ ใช้ keystore:
  # key-store:
  #   location: classpath:keystore.jks
  #   password: keystorepass
  #   alias: mykey
```

```bash
# Encrypt a value
curl -X POST http://config-server:8888/encrypt \
  -d "my-secret-password"
# Returns: {cipher}AbCdEfGhIjKlMnOp...

# Store encrypted value ใน config file
# application.yml:
spring:
  datasource:
    password: '{cipher}AbCdEfGhIjKlMnOp...'

# Decrypt
curl -X POST http://config-server:8888/decrypt \
  -d "AbCdEfGhIjKlMnOp..."
# Returns: my-secret-password
```

---

## 🐳 6. Docker Compose กับ Config Server

```yaml
# docker-compose.yml
services:
  config-server:
    build: ./config-server
    ports: ["8888:8888"]
    environment:
      SPRING_CLOUD_CONFIG_SERVER_GIT_URI: https://github.com/myorg/config-repo
      SPRING_CLOUD_CONFIG_SERVER_GIT_USERNAME: ${GIT_USERNAME}
      SPRING_CLOUD_CONFIG_SERVER_GIT_PASSWORD: ${GIT_TOKEN}
    depends_on: [eureka]
  
  user-service:
    build: ./user-service
    environment:
      SPRING_CONFIG_IMPORT: "configserver:http://config-server:8888"
      SPRING_PROFILES_ACTIVE: dev
    depends_on: [config-server, user-db]
```

---

## 📝 สรุป Part 53

| แนวคิด | รายละเอียด |
|--------|-----------|
| Config Server | Central config repository |
| `@EnableConfigServer` | เปิด Config Server |
| `spring.config.import` | Fetch config from server |
| `@RefreshScope` | Hot reload config |
| Encryption | เก็บ secrets ใน cipher text |
| Git backend | Audit trail สำหรับ config changes |

---

## ➡️ ถัดไป: Part 54 - Distributed Tracing

---
*Part 53/100+ | Kotlin & Spring Boot Complete Course*
