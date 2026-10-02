# Part 87: Team Collaboration
## Git Workflow, Code Review, และ ADR

---

## 🎯 เป้าหมายของ Part นี้

- เข้าใจ Git Workflow ต่างๆ
- ทำ Code Review อย่างมีประสิทธิภาพ
- เขียน Architecture Decision Records (ADR)
- Documentation standards
- Pair Programming techniques

---

## 📖 1. Git Workflow Strategies

### GitFlow

```
GitFlow เหมาะกับ:
- Scheduled releases
- Multiple versions ที่ต้อง support
- Team ใหญ่

Branch structure:
  main          ← production code (tagged releases)
  develop       ← integration branch
  feature/*     ← new features
  release/*     ← release preparation
  hotfix/*      ← urgent production fixes
```

```bash
# GitFlow Example

# Start new feature
git checkout develop
git checkout -b feature/user-authentication

# ... code, commit, code, commit ...

# Finish feature - merge back to develop
git checkout develop
git merge --no-ff feature/user-authentication
git branch -d feature/user-authentication
git push origin develop

# Create release
git checkout develop
git checkout -b release/1.2.0

# ... fix bugs, update version numbers ...

# Finish release
git checkout main
git merge --no-ff release/1.2.0
git tag -a v1.2.0 -m "Release version 1.2.0"
git checkout develop
git merge --no-ff release/1.2.0
git branch -d release/1.2.0
git push origin main develop --tags

# Hotfix
git checkout main
git checkout -b hotfix/critical-security-fix

# ... fix ...

git checkout main
git merge --no-ff hotfix/critical-security-fix
git tag -a v1.2.1 -m "Hotfix version 1.2.1"
git checkout develop
git merge --no-ff hotfix/critical-security-fix
git branch -d hotfix/critical-security-fix
git push origin main develop --tags
```

### Trunk-Based Development (แนะนำสำหรับ CI/CD)

```
Trunk-Based เหมาะกับ:
- Continuous deployment
- Feature flags
- High-frequency releases
- Experienced teams

Branch structure:
  main          ← everything goes here (trunk)
  feature/*     ← short-lived (1-2 days max)
  release/*     ← ถ้าต้องการ (optional)
```

```bash
# Trunk-Based Development

# Create short-lived feature branch
git checkout main
git pull origin main
git checkout -b feature/add-discount-api

# ทำงาน 1-2 วันสูงสุด แล้ว merge กลับ main
git add .
git commit -m "feat(discount): add percentage discount calculation"
git push origin feature/add-discount-api

# Create PR → Review → Merge to main
# หลัง merge ลบ branch ทันที
git branch -d feature/add-discount-api

# Release ด้วย tags
git tag -a v2.3.0 -m "Release v2.3.0"
git push origin v2.3.0
```

### Feature Flags กับ Trunk-Based

```kotlin
// src/main/kotlin/com/example/config/FeatureFlags.kt
package com.example.config

import org.springframework.boot.context.properties.ConfigurationProperties
import org.springframework.stereotype.Component

@Component
@ConfigurationProperties(prefix = "features")
class FeatureFlags {
    var newDiscountEngine: Boolean = false
    var aiRecommendations: Boolean = false
    var betaCheckout: Boolean = false
}

// ใช้งาน
@Service
class OrderService(
    private val featureFlags: FeatureFlags,
    private val oldDiscountService: OldDiscountService,
    private val newDiscountEngine: NewDiscountEngine
) {
    fun calculateDiscount(order: Order): Double {
        return if (featureFlags.newDiscountEngine) {
            newDiscountEngine.calculate(order)  // Feature ใหม่
        } else {
            oldDiscountService.calculate(order)  // Feature เดิม
        }
    }
}
```

```yaml
# application.yml - เปิด/ปิด features
features:
  new-discount-engine: ${FEATURE_NEW_DISCOUNT:false}
  ai-recommendations: ${FEATURE_AI_RECS:false}
  beta-checkout: ${FEATURE_BETA_CHECKOUT:false}
```

---

## 👀 2. Code Review Process

### PR Template

```markdown
<!-- .github/PULL_REQUEST_TEMPLATE.md -->

## 📋 Summary
<!-- อธิบาย change ใน 2-3 ประโยค -->

## 🔧 Type of Change
- [ ] Bug fix (non-breaking change)
- [ ] New feature (non-breaking change)
- [ ] Breaking change
- [ ] Documentation update
- [ ] Refactoring
- [ ] Performance improvement

## 🧪 Testing
- [ ] Unit tests เพิ่ม/อัพเดต
- [ ] Integration tests ผ่าน
- [ ] Manual testing ทำแล้ว

## 📸 Screenshots (ถ้ามี UI change)

## 📝 Checklist
- [ ] Code ตาม coding standards
- [ ] Self-reviewed แล้ว
- [ ] Comments เพิ่มใน complex logic
- [ ] Documentation อัพเดต (ถ้าจำเป็น)
- [ ] ไม่มี breaking changes (หรือ document ไว้แล้ว)

## 🔗 Related Issues
Closes #XXX
```

### Code Review Guidelines

```kotlin
// ✅ Good PR - เล็ก, focused, ทดสอบได้
// PR: "Add email validation to user registration"
// Changed files: 3
// Lines added: 45, deleted: 12

// ❌ Bad PR - ใหญ่เกินไป, หลาย concerns
// PR: "Refactor entire user module + add OAuth + fix bugs"
// Changed files: 47
// Lines added: 1,234, deleted: 567
```

### Reviewer Guidelines

```markdown
# Code Review Guidelines for Reviewers

## เวลา
- Review ภายใน 24 ชั่วโมง
- ถ้า review นานกว่า 2 ชั่วโมง ให้แจ้งทีม

## Comment Tone
✅ "Consider using a map here for O(1) lookup instead of list search"
✅ "Nit: variable name could be more descriptive"
✅ "Question: why do we need this check here?"
❌ "This is wrong"
❌ "You should have known better"

## Severity Levels (prefix comments)
- "Must fix:" - ต้อง fix ก่อน merge
- "Should:" - ควร fix แต่ไม่ block
- "Consider:" - suggestion
- "Nit:" - minor style
- "Question:" - ขอคำอธิบาย

## Focus Areas
1. Correctness (logic bugs)
2. Security vulnerabilities
3. Performance issues
4. Maintainability
5. Test coverage
```

---

## 📋 3. Architecture Decision Records (ADR)

ADR คือ document ที่บันทึกการตัดสินใจทาง architecture พร้อมเหตุผล

### ADR Template

```markdown
# ADR-001: ใช้ PostgreSQL เป็น Primary Database

## Status
Accepted

## Date
2024-01-15

## Context
เราต้องการ database สำหรับ system ใหม่ที่จะ handle:
- 10,000 concurrent users
- Complex relational queries
- JSON data (product metadata)
- Full-text search

## Decision
เลือกใช้ **PostgreSQL 16** เป็น primary database

## Alternatives Considered

### MySQL 8
- ✅ Familiar to team
- ❌ JSON support น้อยกว่า PostgreSQL
- ❌ Full-text search อ่อนกว่า

### MongoDB
- ✅ Flexible schema
- ❌ ไม่มี ACID transactions แบบสมบูรณ์
- ❌ Team ไม่มีประสบการณ์

### DynamoDB
- ✅ Fully managed, scalable
- ❌ Vendor lock-in
- ❌ Complex query patterns ยาก

## Consequences
### ดี
- ACID compliance เต็มรูปแบบ
- Excellent JSON support (JSONB)
- Strong full-text search
- Rich extension ecosystem (PostGIS, etc.)
- Active community

### แย่
- Team ต้องเรียนรู้ PostgreSQL specifics
- ต้องดูแล infrastructure เอง (ไม่ใช่ managed)
- Horizontal scaling ซับซ้อนกว่า NoSQL
```

### ADR Structure สำหรับทีม

```
docs/
├── adr/
│   ├── README.md              # Index ของ ADRs ทั้งหมด
│   ├── ADR-001-database.md
│   ├── ADR-002-auth-strategy.md
│   ├── ADR-003-api-versioning.md
│   └── ADR-004-caching-layer.md
```

```markdown
# ADR Index

| ID | Title | Status | Date |
|----|-------|--------|------|
| ADR-001 | PostgreSQL as Primary Database | Accepted | 2024-01-15 |
| ADR-002 | JWT Authentication Strategy | Accepted | 2024-01-20 |
| ADR-003 | URL Path API Versioning | Accepted | 2024-02-01 |
| ADR-004 | Redis for Session Caching | Proposed | 2024-02-15 |
```

---

## 📝 4. Documentation Standards

### Code Comments

```kotlin
// ✅ Comment อธิบาย WHY ไม่ใช่ WHAT
// Bad: increment counter by 1
counter++

// Good: Retry up to 3 times to handle transient network failures
// See: https://example.com/retry-policy
repeat(MAX_RETRIES) { attempt ->
    try {
        return makeApiCall()
    } catch (e: NetworkException) {
        if (attempt == MAX_RETRIES - 1) throw e
        Thread.sleep(RETRY_DELAY_MS * (attempt + 1))
    }
}

// ✅ KDoc สำหรับ public API
/**
 * Processes a payment for the given order.
 *
 * This function handles the complete payment lifecycle including:
 * - Payment gateway interaction
 * - Idempotency checking (prevents double charges)
 * - Order status update
 *
 * @param orderId The unique identifier of the order to pay
 * @param paymentMethod The payment method to charge
 * @return [PaymentResult] containing transaction ID and status
 * @throws PaymentException if the payment gateway rejects the charge
 * @throws OrderNotFoundException if the order doesn't exist
 *
 * @since 2.0.0
 */
fun processPayment(orderId: Long, paymentMethod: PaymentMethod): PaymentResult {
    // implementation
}
```

### README Structure

```markdown
# Project Name

Brief description of what this project does.

## Getting Started

### Prerequisites
- JDK 21+
- Docker Desktop
- PostgreSQL 16 (or use Docker)

### Installation
\`\`\`bash
git clone https://github.com/org/project.git
cd project
./gradlew build
\`\`\`

### Running Locally
\`\`\`bash
docker-compose up -d  # Start dependencies
./gradlew bootRun
\`\`\`

API available at: http://localhost:8080
Swagger UI: http://localhost:8080/swagger-ui.html

## Architecture
[Link to Architecture docs or diagram]

## Contributing
See [CONTRIBUTING.md](CONTRIBUTING.md)

## Deployment
See [DEPLOYMENT.md](docs/DEPLOYMENT.md)
```

---

## 👥 5. Pair Programming

### Pair Programming Formats

```
Driver-Navigator (ทั่วไป):
  - Driver: พิมพ์โค้ด
  - Navigator: คิดกลยุทธ์, review, suggest
  - Switch ทุก 25-30 นาที

Ping-Pong (TDD):
  - Person A: เขียน failing test
  - Person B: ทำให้ test pass + เขียน test ต่อไป
  - Person A: ทำให้ test pass + เขียน test ต่อไป
  - สลับไปเรื่อยๆ

Strong-Style:
  - Navigator บอก idea
  - Driver implement ทันที
  - เหมาะสำหรับ junior + senior
```

### Remote Pairing Tools

```bash
# VS Code Live Share
code --install-extension MS-vsliveshare.vsliveshare

# IntelliJ Code With Me
# Settings → Tools → Code With Me → Enable

# tmux (terminal sharing)
tmux new-session -s pair
# share session name กับ partner
tmux attach -t pair
```

---

## 📋 สรุป

| หัวข้อ | Best Practice |
|--------|--------------|
| GitFlow | เหมาะกับ scheduled releases, multiple versions |
| Trunk-Based | เหมาะกับ CI/CD, high-frequency deployment |
| Feature Flags | Deploy code แยกจาก release feature |
| PR Size | เล็กๆ focused, ง่ายต่อ review |
| ADR | บันทึก architectural decisions ทุกครั้ง |
| Code Comments | อธิบาย WHY ไม่ใช่ WHAT |
| Pair Programming | เพิ่ม code quality, knowledge sharing |

---

*Part 87/100+ | Kotlin & Spring Boot Complete Course*
