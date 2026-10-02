# Part 88: Technical Debt Management
## จัดการ Technical Debt อย่างมีระบบ

---

## 🎯 เป้าหมายของ Part นี้

- เข้าใจ Technical Debt คืออะไร
- วิธีระบุและวัด Technical Debt
- Strategies ในการ pay down debt
- Boy Scout Rule
- Strangler Fig Pattern

---

## 📖 1. Technical Debt คืออะไร?

Ward Cunningham บัญญัติคำว่า "Technical Debt" ในปี 1992 เพื่ออธิบายการที่เราเลือก solution ที่เร็วในตอนนี้ แต่ต้องจ่าย "ดอกเบี้ย" ในรูปแบบของ complexity และ maintenance cost ในอนาคต

```
Technical Debt = รายงานทุกเรื่องที่ต้อง fix/refactor แต่ถูกเลื่อนออกไป

ดอกเบี้ย = เวลาที่ใช้เพิ่มขึ้นทุกครั้งที่ต้อง work กับโค้ดนั้น

เหมือนหนี้เงิน:
- ยิ่งปล่อยนาน → ดอกเบี้ยสะสม → ใช้เวลา + เงินมากขึ้น
- ถ้าไม่จ่าย → bankruptcy (code ไม่สามารถ maintain ได้เลย)
```

### ประเภทของ Technical Debt

```
1. Deliberate (ตั้งใจ)
   "เราจะทำ shortcut ตอนนี้ก่อน แล้วค่อย refactorทีหลัง"
   → บางครั้ง OK ถ้า document ไว้และ plan จะ fix

2. Inadvertent (ไม่ได้ตั้งใจ)
   "เราไม่รู้ว่าวิธีที่ดีกว่าคืออะไร"
   → เกิดจาก lack of knowledge/experience

3. Reckless (ประมาท)
   "เราไม่มีเวลาสำหรับ design ที่ดี"
   → อันตรายที่สุด, สะสมเร็วมาก
```

---

## 🔍 2. วิธีระบุ Technical Debt

### Code Indicators

```kotlin
// 1. Long Parameter Lists
// ❌ Too many parameters = bad design
fun createOrder(
    userId: Long,
    productId: Long,
    quantity: Int,
    discountCode: String?,
    shippingAddress: String,
    billingAddress: String,
    paymentMethod: String,
    notes: String?
): Order { ... }

// ✅ Use value objects / data classes
data class CreateOrderRequest(
    val userId: Long,
    val productId: Long,
    val quantity: Int,
    val discountCode: String? = null,
    val shippingAddress: Address,
    val billingAddress: Address,
    val paymentMethod: PaymentMethod,
    val notes: String? = null
)
fun createOrder(request: CreateOrderRequest): Order { ... }

// 2. God Class - ทำทุกอย่าง
class UserManager {
    // 2000 lines of code doing everything related to users
    // authentication, authorization, profile, settings, orders, payments...
    // → ต้อง split ออกเป็น multiple classes
}

// 3. Copy-Paste Code
// ถ้าเห็น code block เดียวกันในหลายที่ = debt
// → Extract to function/class

// 4. Commented-out Code
// fun oldProcessOrder() {
//     // This was removed in v2 but nobody deleted it
// }
// → ลบออก! git history เก็บไว้อยู่แล้ว
```

### เครื่องมือวัด Technical Debt

```bash
# SonarQube - คำนวณ debt เป็น hours
# Login to SonarQube → Projects → Technical Debt

# Detekt - count code smells
./gradlew detekt

# JaCoCo - code coverage (low coverage = debt)
./gradlew test jacocoTestReport
open build/reports/jacoco/test/html/index.html

# Complexity metrics
./gradlew detekt | grep "Complexity"
```

### Technical Debt Register (Spreadsheet/Jira)

```markdown
# Technical Debt Register

| ID | Description | Impact | Effort | Priority | Owner | Target Sprint |
|----|-------------|--------|--------|----------|-------|---------------|
| TD-001 | UserService ทำงาน 3 responsibilities | High | M | P1 | Alice | Sprint 12 |
| TD-002 | Hardcoded configs ใน OrderController | Medium | S | P2 | Bob | Sprint 13 |
| TD-003 | No integration tests สำหรับ Payment | High | L | P1 | Carol | Sprint 12 |
| TD-004 | Inconsistent error handling | Medium | M | P2 | Dave | Sprint 14 |
```

---

## 🧹 3. Boy Scout Rule

> "Always leave the code better than you found it"
> — Robert C. Martin

```kotlin
// สมมติเจอ code นี้ขณะแก้ bug อื่น
fun getUserOrders(userId: Long): List<Order> {
    val user = db.query("SELECT * FROM users WHERE id=" + userId)  // SQL injection!
    val orders = db.query("SELECT * FROM orders WHERE user_id=" + userId)
    var result = ArrayList<Order>()
    for (i in 0..orders.size) {  // off-by-one bug!
        result.add(orders.get(i))
    }
    return result
}

// Boy Scout Rule: แก้ problems เล็กๆ ขณะผ่าน
fun getUserOrders(userId: Long): List<Order> {
    // ✅ Fix SQL injection
    // ✅ Fix off-by-one
    // ✅ Use idiomatic Kotlin
    return orderRepository.findByUserId(userId)
}
```

### Boy Scout ใน Practice

```bash
# สิ่งที่ทำได้ "ขณะผ่าน" (เล็กๆ):
- แก้ typo ใน comments/variable names
- เพิ่ม missing null check
- ลบ dead code
- แก้ minor code smell
- เพิ่ม missing test สำหรับ edge case

# สิ่งที่ควรเป็น separate task:
- Restructure entire module
- Change database schema
- Replace entire library
- Big refactoring
```

---

## 🌱 4. Strangler Fig Pattern

Strangler Fig Pattern ใช้สำหรับ migrate legacy system ทีละส่วน โดยไม่ต้อง rewrite ทั้งหมดพร้อมกัน

```
Strangler Fig (ต้นไม้เถาวัลย์):
  1. เถาวัลย์เติบโตรอบต้นไม้ที่แข็งแรงดั้งเดิม
  2. ค่อยๆ แทนที่ต้นไม้เดิมทีละส่วน
  3. สุดท้ายต้นไม้เดิมตาย แต่ไม่มีช่วง downtime

Legacy System Migration:
  1. สร้าง new service/module
  2. Route traffic ใหม่ไปที่ new code
  3. ค่อยๆ ลด traffic ไปที่ legacy
  4. ลบ legacy เมื่อ traffic = 0
```

### ตัวอย่าง Strangler Fig กับ Spring Boot

```kotlin
// Legacy Code - OrderController เดิม
@RestController
@RequestMapping("/api/orders")
class LegacyOrderController {
    // โค้ดเก่า ซับซ้อน ไม่มี tests
    @PostMapping
    fun createOrder(request: HttpServletRequest): String {
        val body = request.reader.readText()
        // Parse manually, no validation, direct DB calls
        // 500 lines of spaghetti code...
        return "OK"
    }
}

// New Code - Modern implementation
@RestController
@RequestMapping("/api/v2/orders")
class ModernOrderController(
    private val orderService: OrderService
) {
    @PostMapping
    fun createOrder(@Valid @RequestBody request: CreateOrderRequest): ResponseEntity<OrderDto> {
        val order = orderService.createOrder(request)
        return ResponseEntity.status(HttpStatus.CREATED).body(order)
    }
}

// Router/Proxy - ค่อยๆ migrate traffic
@Component
class OrderRouter(
    private val featureFlags: FeatureFlags
) {
    // ใช้ Feature Flag เพื่อ gradually migrate
    fun shouldUseNewOrderSystem(userId: Long): Boolean {
        return when {
            featureFlags.orderSystemV2Percentage == 100 -> true
            featureFlags.orderSystemV2Percentage == 0 -> false
            // Canary deployment: 10% users ใช้ system ใหม่
            else -> userId % 100 < featureFlags.orderSystemV2Percentage
        }
    }
}
```

### Migration Plan Template

```markdown
# Migration Plan: Order System V1 → V2

## Phase 1: Foundation (Sprint 10-11)
- [ ] สร้าง new OrderService with proper design
- [ ] เพิ่ม unit tests (>80% coverage)
- [ ] Integration tests

## Phase 2: Shadow Mode (Sprint 12)
- [ ] Route requests ไปที่ both old และ new
- [ ] Compare results (validation mode)
- [ ] ไม่ return results จาก new yet

## Phase 3: Canary (Sprint 13)
- [ ] 5% traffic → new system
- [ ] Monitor errors, latency
- [ ] เพิ่มเป็น 10%, 25%, 50%

## Phase 4: Migration (Sprint 14)
- [ ] 100% traffic → new system
- [ ] Legacy เป็น read-only fallback

## Phase 5: Cleanup (Sprint 15)
- [ ] ลบ legacy code
- [ ] Update documentation
- [ ] ปิด feature flags
```

---

## 📊 5. Debt Paydown Strategies

### The Debt Snowball (เล็กก่อน)

```
ลำดับ: แก้ debt ที่เล็กที่สุดก่อน
ข้อดี: เห็นผลเร็ว, motivate ทีม
เหมาะกับ: team ที่ขวัญเสีย, เริ่มต้น

Sprint 1: Fix 5 small code smells
Sprint 2: Remove 3 dead code sections
Sprint 3: Add tests for untested utility functions
```

### The Debt Avalanche (ดอกเบี้ยสูงก่อน)

```
ลำดับ: แก้ debt ที่มี impact สูงที่สุดก่อน
ข้อดี: ลด "ดอกเบี้ย" เร็วที่สุด
เหมาะกับ: team ที่มี discipline, long-term focus

Sprint 1: Refactor UserService (กระทบ 30% codebase)
Sprint 2: Add integration tests for Payment module (critical path)
Sprint 3: Fix security vulnerabilities
```

### 20% Time สำหรับ Debt

```bash
# Product + Engineering agreement:
# 80% features + bugs
# 20% technical improvement

# Tracking ใน Jira/Linear:
# Label: "tech-debt"
# Sprint goal includes debt items

# Reporting:
# - Debt items completed per sprint
# - SonarQube debt hours trend
# - Test coverage trend
```

---

## 📋 สรุป

| หัวข้อ | Key Points |
|--------|-----------|
| Technical Debt | Shortcut ตอนนี้ = cost ในอนาคต |
| Deliberate Debt | ยอมรับได้ถ้า document และ plan fix |
| Boy Scout Rule | ทำ code ดีขึ้นทุกครั้งที่ผ่าน |
| Strangler Fig | Migrate ทีละส่วน ไม่ต้อง big bang |
| Debt Register | Track debt อย่างเป็นระบบ |
| 20% Rule | จัดเวลาให้ debt reduction ทุก sprint |

---

*Part 88/100+ | Kotlin & Spring Boot Complete Course*
