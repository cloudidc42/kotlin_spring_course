# Part 72: CI/CD Pipeline

## CI/CD Pipeline — Automate Build, Test, Deploy ด้วย GitHub Actions

---

## 🎯 เป้าหมายของ Part นี้

- สร้าง GitHub Actions workflow
- Build, test, Docker build และ push
- Automated deployment
- Environment secrets management
- Rollback strategies
- Full CI/CD pipeline

---

## 📖 1. CI/CD Overview

```
Developer pushes code
         ↓
    GitHub Actions triggers
         ↓
    Run Tests (unit + integration)
         ↓
    Code Quality Checks (lint, coverage)
         ↓
    Build Docker Image
         ↓
    Push to Registry
         ↓
    Deploy to Staging
         ↓
    Smoke Tests
         ↓
    Manual Approval (Production)
         ↓
    Deploy to Production
         ↓
    Health Check
         ↓
    Notify Team
```

---

## 🔧 2. GitHub Actions Workflow

### .github/workflows/ci.yml

```yaml
name: CI Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  JAVA_VERSION: '21'
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  test:
    name: Run Tests
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
          POSTGRES_DB: testdb
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
      
      redis:
        image: redis:7
        ports:
          - 6379:6379
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up JDK
        uses: actions/setup-java@v4
        with:
          java-version: ${{ env.JAVA_VERSION }}
          distribution: 'temurin'
          cache: gradle

      - name: Grant execute permission to gradlew
        run: chmod +x gradlew

      - name: Run unit tests
        run: ./gradlew test

      - name: Run integration tests
        run: ./gradlew integrationTest
        env:
          SPRING_DATASOURCE_URL: jdbc:postgresql://localhost:5432/testdb
          SPRING_DATASOURCE_USERNAME: test
          SPRING_DATASOURCE_PASSWORD: test
          SPRING_REDIS_HOST: localhost

      - name: Generate test report
        uses: dorny/test-reporter@v1
        if: success() || failure()
        with:
          name: Test Results
          path: build/test-results/**/*.xml
          reporter: java-junit

      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v4
        with:
          token: ${{ secrets.CODECOV_TOKEN }}
          files: build/reports/jacoco/test/jacocoTestReport.xml

  code-quality:
    name: Code Quality
    runs-on: ubuntu-latest
    needs: test
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up JDK
        uses: actions/setup-java@v4
        with:
          java-version: ${{ env.JAVA_VERSION }}
          distribution: 'temurin'
          cache: gradle

      - name: Run Ktlint
        run: ./gradlew ktlintCheck

      - name: Run Detekt
        run: ./gradlew detekt

      - name: SonarQube analysis
        run: ./gradlew sonarqube
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}

  build-image:
    name: Build Docker Image
    runs-on: ubuntu-latest
    needs: [test, code-quality]
    if: github.ref == 'refs/heads/main' || github.ref == 'refs/heads/develop'
    
    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}
      image-digest: ${{ steps.build.outputs.digest }}
    
    steps:
      - uses: actions/checkout@v4

      - name: Set up JDK
        uses: actions/setup-java@v4
        with:
          java-version: ${{ env.JAVA_VERSION }}
          distribution: 'temurin'
          cache: gradle

      - name: Build JAR
        run: ./gradlew bootJar

      - name: Set up QEMU
        uses: docker/setup-qemu-action@v3

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Log in to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=ref,event=branch
            type=sha,prefix={{branch}}-
            type=semver,pattern={{version}}

      - name: Build and push Docker image
        id: build
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          platforms: linux/amd64,linux/arm64

      - name: Scan for vulnerabilities
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ steps.meta.outputs.version }}
          format: 'sarif'
          output: 'trivy-results.sarif'
          severity: 'CRITICAL,HIGH'
```

---

## 🚀 3. Deployment Workflow

```yaml
# .github/workflows/deploy.yml
name: Deploy Pipeline

on:
  workflow_run:
    workflows: ["CI Pipeline"]
    types: [completed]
    branches: [main]

jobs:
  deploy-staging:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    if: ${{ github.event.workflow_run.conclusion == 'success' }}
    environment: staging
    
    steps:
      - uses: actions/checkout@v4

      - name: Set up kubectl
        uses: azure/setup-kubectl@v4
        with:
          version: 'v1.28.0'

      - name: Configure kubeconfig
        run: |
          mkdir -p $HOME/.kube
          echo "${{ secrets.KUBECONFIG_STAGING }}" | base64 -d > $HOME/.kube/config

      - name: Deploy to staging
        run: |
          kubectl set image deployment/spring-app \
            spring-app=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:main-${{ github.sha }} \
            -n staging
          kubectl rollout status deployment/spring-app -n staging --timeout=300s

      - name: Run smoke tests
        run: |
          sleep 30
          curl -f https://staging.example.com/actuator/health || exit 1

  deploy-production:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: deploy-staging
    environment:
      name: production
      url: https://app.example.com
    
    steps:
      - uses: actions/checkout@v4

      - name: Set up kubectl
        uses: azure/setup-kubectl@v4

      - name: Configure kubeconfig
        run: |
          echo "${{ secrets.KUBECONFIG_PRODUCTION }}" | base64 -d > $HOME/.kube/config

      - name: Blue-Green Deploy
        run: |
          # อัพเดต image สำหรับ deployment
          kubectl set image deployment/spring-app-green \
            spring-app=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:main-${{ github.sha }} \
            -n production
          
          # รอให้ green deployment พร้อม
          kubectl rollout status deployment/spring-app-green -n production --timeout=300s
          
          # Switch traffic จาก blue ไป green
          kubectl patch service spring-app-service \
            -n production \
            -p '{"spec": {"selector": {"deployment": "green"}}}'

      - name: Health check
        run: |
          for i in {1..5}; do
            response=$(curl -s -o /dev/null -w "%{http_code}" https://app.example.com/actuator/health)
            if [ "$response" == "200" ]; then
              echo "Health check passed"
              exit 0
            fi
            sleep 10
          done
          echo "Health check failed, rolling back..."
          exit 1

      - name: Notify Slack on success
        if: success()
        uses: slackapi/slack-github-action@v1.24.0
        with:
          payload: |
            {
              "text": "Deployment to production succeeded! :rocket:",
              "blocks": [
                {
                  "type": "section",
                  "text": {
                    "type": "mrkdwn",
                    "text": "*Production Deployment Successful* :white_check_mark:\nVersion: `${{ github.sha }}`\nDeployed by: ${{ github.actor }}"
                  }
                }
              ]
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

---

## 🔙 4. Rollback Strategies

### Manual Rollback Script

```bash
#!/bin/bash
# rollback.sh

NAMESPACE="${1:-production}"
DEPLOYMENT="${2:-spring-app}"
REVISION="${3:-1}"

echo "Rolling back $DEPLOYMENT in $NAMESPACE to revision $REVISION"

# ดู revision history
kubectl rollout history deployment/$DEPLOYMENT -n $NAMESPACE

# Rollback
kubectl rollout undo deployment/$DEPLOYMENT -n $NAMESPACE --to-revision=$REVISION

# รอให้ rollback เสร็จ
kubectl rollout status deployment/$DEPLOYMENT -n $NAMESPACE --timeout=300s

echo "Rollback completed"
```

### Automatic Rollback in Workflow

```yaml
- name: Deploy with automatic rollback
  run: |
    # บันทึก revision ปัจจุบัน
    CURRENT_REVISION=$(kubectl rollout history deployment/spring-app -n production | tail -2 | head -1 | awk '{print $1}')
    
    # Deploy
    kubectl set image deployment/spring-app \
      spring-app=$NEW_IMAGE \
      -n production
    
    # รอและตรวจสอบ
    if ! kubectl rollout status deployment/spring-app -n production --timeout=120s; then
      echo "Deployment failed, rolling back..."
      kubectl rollout undo deployment/spring-app -n production
      exit 1
    fi
    
    # Smoke test
    if ! curl -f https://app.example.com/actuator/health; then
      echo "Health check failed, rolling back..."
      kubectl rollout undo deployment/spring-app -n production
      exit 1
    fi
```

---

## 🔑 5. Environment Secrets Management

### GitHub Secrets ที่ต้องตั้งค่า

```
Repository Secrets:
├── KUBECONFIG_STAGING        (base64 encoded kubeconfig)
├── KUBECONFIG_PRODUCTION     (base64 encoded kubeconfig)
├── REGISTRY_USERNAME         (container registry)
├── REGISTRY_PASSWORD         (container registry)
├── SONAR_TOKEN               (SonarQube)
├── CODECOV_TOKEN             (Codecov)
└── SLACK_WEBHOOK_URL         (Slack notifications)

Environment Secrets (production):
├── DB_PASSWORD               (database)
├── JWT_SECRET                (JWT signing)
└── API_KEYS                  (third-party APIs)
```

---

## 📦 6. Dockerfile

```dockerfile
# Multi-stage Dockerfile สำหรับ Spring Boot
FROM eclipse-temurin:21-jdk-alpine AS builder
WORKDIR /app
COPY gradlew .
COPY gradle gradle
COPY build.gradle.kts .
COPY settings.gradle.kts .
RUN ./gradlew dependencies --no-daemon

COPY src src
RUN ./gradlew bootJar --no-daemon

FROM eclipse-temurin:21-jre-alpine AS runtime
WORKDIR /app

# สร้าง non-root user
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

COPY --from=builder /app/build/libs/*.jar app.jar

EXPOSE 8080

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=60s --retries=3 \
  CMD wget -q --spider http://localhost:8080/actuator/health || exit 1

ENTRYPOINT ["java", \
  "-XX:+UseContainerSupport", \
  "-XX:MaxRAMPercentage=75.0", \
  "-Djava.security.egd=file:/dev/./urandom", \
  "-jar", "app.jar"]
```

---

## 📊 7. สรุปตาราง CI/CD Stages

| Stage | Actions | เมื่อ Fail |
|-------|---------|-----------|
| Build | Compile Kotlin | Block all |
| Unit Test | JUnit, Kotest | Block all |
| Integration Test | Spring Boot Test | Block all |
| Code Quality | Ktlint, Detekt | Block merge |
| Security Scan | Trivy, OWASP | Alert + Block |
| Docker Build | Multi-stage | Block deploy |
| Deploy Staging | kubectl set image | Block prod |
| Smoke Test | curl health check | Block prod + rollback |
| Deploy Production | Blue-green | Auto rollback |
| Notify | Slack | Continue |

---

## 💡 Best Practices

1. **Fail fast** — รัน test เร็วที่สุดก่อน deploy
2. **Immutable images** — tag ด้วย git SHA แทน `latest`
3. **Blue-Green deploy** — zero downtime และ rollback ง่าย
4. **Secrets rotation** — หมุนเวียน secrets สม่ำเสมอ
5. **Observability** — monitor deployment metrics ทันที

---

## 🔍 8. Security Scanning ใน CI

```yaml
# .github/workflows/security.yml
name: Security Scan

on:
  schedule:
    - cron: '0 6 * * 1'  # ทุกวันจันทร์เช้า
  push:
    branches: [main]

jobs:
  dependency-check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Run OWASP Dependency Check
        uses: dependency-check/Dependency-Check_Action@main
        with:
          project: "spring-app"
          path: "."
          format: "HTML"
          args: >
            --failOnCVSS 7
            --enableRetired

      - name: Upload results
        uses: actions/upload-artifact@v4
        with:
          name: dependency-check-report
          path: reports/

  container-scan:
    runs-on: ubuntu-latest
    needs: build-image
    steps:
      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ needs.build-image.outputs.image-tag }}
          format: 'table'
          exit-code: '1'
          ignore-unfixed: true
          severity: 'CRITICAL,HIGH'

  sast:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Run Semgrep
        uses: returntocorp/semgrep-action@v1
        with:
          config: >-
            p/kotlin
            p/spring
            p/security-audit
```

---

## 📊 9. Release Management

```yaml
# .github/workflows/release.yml
name: Release

on:
  push:
    tags:
      - 'v*.*.*'

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Extract version from tag
        id: version
        run: echo "version=${GITHUB_REF#refs/tags/v}" >> $GITHUB_OUTPUT

      - name: Update version in build.gradle.kts
        run: |
          sed -i "s/version = \".*\"/version = \"${{ steps.version.outputs.version }}\"/" build.gradle.kts

      - name: Build and push release image
        run: |
          docker build -t registry.example.com/spring-app:${{ steps.version.outputs.version }} .
          docker push registry.example.com/spring-app:${{ steps.version.outputs.version }}

      - name: Create GitHub Release
        uses: softprops/action-gh-release@v1
        with:
          tag_name: v${{ steps.version.outputs.version }}
          body: |
            ## Changes in v${{ steps.version.outputs.version }}
            
            See [CHANGELOG.md](CHANGELOG.md) for details.
          draft: false
          prerelease: false
```

---

*Part 72/100+ | Kotlin & Spring Boot Complete Course*
