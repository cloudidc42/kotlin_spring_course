# Part 30: Docker และ Deployment
## Containerize และ Deploy Spring Boot Application

---

## 🎯 เป้าหมายของ Part นี้

- สร้าง Dockerfile สำหรับ Spring Boot
- Docker Compose สำหรับ dev environment
- Multi-stage builds
- Environment variables และ secrets
- Deploy บน Railway/Render/Fly.io
- Health checks

---

## 🐳 1. Dockerfile

```dockerfile
# Dockerfile
# Multi-stage build: แยก build stage กับ runtime stage

# Stage 1: Build
FROM eclipse-temurin:17-jdk-alpine AS builder
WORKDIR /app

# Copy Gradle files
COPY gradlew .
COPY gradle gradle
COPY build.gradle.kts .
COPY settings.gradle.kts .

# Download dependencies (cache layer)
RUN ./gradlew dependencies --no-daemon

# Copy source และ build
COPY src src
RUN ./gradlew bootJar --no-daemon -x test

# Stage 2: Runtime (smaller image)
FROM eclipse-temurin:17-jre-alpine
WORKDIR /app

# Security: non-root user
RUN addgroup -S spring && adduser -S spring -G spring
USER spring:spring

# Copy JAR from builder
COPY --from=builder /app/build/libs/*.jar app.jar

# Expose port
EXPOSE 8080

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=30s --retries=3 \
    CMD wget -q --spider http://localhost:8080/actuator/health || exit 1

# Run
ENTRYPOINT ["java", \
    "-XX:+UseContainerSupport", \
    "-XX:MaxRAMPercentage=75.0", \
    "-jar", "app.jar"]
```

Build and run:
```bash
# Build image
docker build -t myapp:latest .

# Run container
docker run -p 8080:8080 \
    -e SPRING_PROFILES_ACTIVE=prod \
    -e DATABASE_URL=jdbc:postgresql://host:5432/db \
    myapp:latest

# Run in background
docker run -d \
    --name myapp \
    -p 8080:8080 \
    --restart unless-stopped \
    myapp:latest
```

---

## 🎼 2. Docker Compose (Development)

```yaml
# docker-compose.yml
version: '3.8'

services:
  app:
    build: .
    ports:
      - "8080:8080"
    environment:
      - SPRING_PROFILES_ACTIVE=dev
      - SPRING_DATASOURCE_URL=jdbc:postgresql://db:5432/appdb
      - SPRING_DATASOURCE_USERNAME=appuser
      - SPRING_DATASOURCE_PASSWORD=apppass
      - SPRING_JPA_HIBERNATE_DDL_AUTO=update
    depends_on:
      db:
        condition: service_healthy
    restart: unless-stopped
    networks:
      - app-network

  db:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: appdb
      POSTGRES_USER: appuser
      POSTGRES_PASSWORD: apppass
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U appuser -d appdb"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - app-network

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    networks:
      - app-network

volumes:
  postgres_data:

networks:
  app-network:
    driver: bridge
```

Commands:
```bash
# Start all services
docker compose up -d

# View logs
docker compose logs -f app

# Stop
docker compose down

# Rebuild after code changes
docker compose up -d --build app
```

---

## 🌍 3. Environment Configuration

```yaml
# application-prod.yml
spring:
  datasource:
    url: ${DATABASE_URL}
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
    hikari:
      maximum-pool-size: 10
      minimum-idle: 2
      connection-timeout: 30000
  
  jpa:
    show-sql: false
    hibernate:
      ddl-auto: validate
  
  cache:
    type: redis
  
  redis:
    host: ${REDIS_HOST:localhost}
    port: ${REDIS_PORT:6379}

server:
  port: ${PORT:8080}
  shutdown: graceful

management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics

logging:
  level:
    root: WARN
    com.example: INFO
```

---

## ☁️ 4. Deploy บน Render

```yaml
# render.yaml
services:
  - type: web
    name: myapp
    runtime: docker
    dockerfilePath: ./Dockerfile
    envVars:
      - key: SPRING_PROFILES_ACTIVE
        value: prod
      - key: DATABASE_URL
        fromDatabase:
          name: myapp-db
          property: connectionString
      - key: APP_SECURITY_JWT_SECRET
        generateValue: true
    healthCheckPath: /actuator/health

databases:
  - name: myapp-db
    databaseName: appdb
    user: appuser
    plan: free
```

---

## 🚂 5. Deploy บน Railway

```bash
# Install Railway CLI
npm i -g @railway/cli

# Login
railway login

# Initialize project
railway init

# Deploy
railway up

# Set environment variables
railway variables set DATABASE_URL=...
railway variables set SPRING_PROFILES_ACTIVE=prod

# View logs
railway logs
```

---

## 🩺 6. Health Checks ใน Spring Boot

```kotlin
// actuator endpoints
// application.yml
management:
  endpoints:
    web:
      exposure:
        include: health,info,readiness,liveness
  endpoint:
    health:
      show-details: always
      probes:
        enabled: true  # เปิด liveness/readiness probes

// Custom Health Indicator
@Component
class DatabaseHealthIndicator(private val dataSource: DataSource) : HealthIndicator {
    
    override fun health(): Health {
        return try {
            dataSource.connection.use { conn ->
                if (conn.isValid(1)) {
                    Health.up()
                        .withDetail("database", "PostgreSQL")
                        .withDetail("status", "connected")
                        .build()
                } else {
                    Health.down().withDetail("reason", "Connection invalid").build()
                }
            }
        } catch (e: Exception) {
            Health.down(e).build()
        }
    }
}
```

```bash
# Health endpoints
curl http://localhost:8080/actuator/health
# {"status":"UP","components":{"db":{"status":"UP"},...}}

curl http://localhost:8080/actuator/health/liveness
# {"status":"UP"}

curl http://localhost:8080/actuator/health/readiness
# {"status":"UP"}
```

---

## 🔒 7. Kubernetes Deployment (Bonus)

```yaml
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 2
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: myapp
          image: myapp:latest
          ports:
            - containerPort: 8080
          env:
            - name: SPRING_PROFILES_ACTIVE
              value: "prod"
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: db-secret
                  key: url
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "500m"
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  selector:
    app: myapp
  ports:
    - port: 80
      targetPort: 8080
  type: LoadBalancer
```

---

## 📝 สรุป Part 30

| แนวคิด | รายละเอียด |
|--------|-----------|
| Multi-stage Dockerfile | Build stage + slim runtime |
| Docker Compose | Local development stack |
| Environment vars | `${DATABASE_URL}` |
| Health checks | `/actuator/health/liveness` |
| Render/Railway | Easy cloud deployment |
| Kubernetes | Production-grade orchestration |
| Graceful shutdown | `server.shutdown: graceful` |

---

## 🎓 สรุป Phase 2 (Parts 21-30)

เราได้เรียนรู้ Spring Boot ครบถ้วนแล้ว:

| Part | เนื้อหา |
|------|---------|
| 21 | Spring Boot + IoC/DI fundamentals |
| 22 | Kotlin-specific Spring setup |
| 23 | REST API CRUD |
| 24 | Spring Data JPA + Relationships |
| 25 | Validation |
| 26 | Exception Handling + RFC 7807 |
| 27 | Spring Security |
| 28 | JWT Authentication |
| 29 | Testing (Unit, Integration, E2E) |
| 30 | Docker + Deployment |

---

## ➡️ ถัดไป: Part 31 - Caching ด้วย Redis

Phase 3 จะครอบคลุม: Caching, Messaging, Microservices, และ Advanced Topics

---
*Part 30/100+ | Kotlin & Spring Boot Complete Course*
