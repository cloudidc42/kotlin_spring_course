# Part 98: Open Source Contribution
## มีส่วนร่วมใน Open Source Ecosystem

---

## 🎯 เป้าหมายของ Part นี้

- หา projects ที่เหมาะสมสำหรับ contribute
- เข้าใจ codebase ใหม่อย่างรวดเร็ว
- First PR strategies
- Kotlin/Spring open source ecosystem
- สร้าง library ของตัวเอง

---

## 📖 1. ทำไมต้อง Contribute Open Source?

```
ประโยชน์สำหรับตัวคุณ:
✅ เรียนรู้จาก best engineers ในโลก
✅ Portfolio ที่แสดงถึง real-world skills
✅ Network กับ engineers ทั่วโลก
✅ Credibility ใน community
✅ อาจได้รับ job offer จาก maintainers

ประโยชน์สำหรับ ecosystem:
✅ ช่วย fix bugs ที่คุณเจอ
✅ เพิ่ม documentation
✅ แปลเป็นภาษาต่างๆ
✅ ช่วยตอบ issues
```

---

## 🔍 2. หา Projects ที่เหมาะสม

### เริ่มจาก Tools ที่คุณใช้จริง

```bash
# ดู dependencies ของ project ตัวเอง
./gradlew dependencies

# Projects ที่น่า contribute:
# 1. Spring Boot (สร้างด้วย Java/Kotlin)
#    https://github.com/spring-projects/spring-boot

# 2. Kotlinx Coroutines
#    https://github.com/Kotlin/kotlinx.coroutines

# 3. Arrow-kt (Functional programming)
#    https://github.com/arrow-kt/arrow

# 4. Ktor (Kotlin web framework)
#    https://github.com/ktorio/ktor

# 5. Detekt (Static analysis)
#    https://github.com/detekt/detekt

# 6. MockK (Mocking library)
#    https://github.com/mockk/mockk
```

### GitHub Search สำหรับ Beginner Issues

```bash
# Search syntax:
# label:"good first issue" language:kotlin
# label:"help wanted" language:kotlin
# label:"bug" label:"good first issue" language:kotlin

# ตัวอย่าง URLs:
# https://github.com/search?q=label%3A%22good+first+issue%22+language%3Akotlin&type=Issues

# หรือใช้ websites:
# - goodfirstissue.dev
# - up-for-grabs.net
# - firsttimersonly.com
```

---

## 📚 3. เข้าใจ Codebase ใหม่

### Systematic Approach

```bash
# Step 1: อ่าน README และ CONTRIBUTING.md
cat README.md
cat CONTRIBUTING.md

# Step 2: ดู project structure
find . -name "*.kt" | head -20
ls -la

# Step 3: Build project
./gradlew build
./gradlew test

# Step 4: อ่าน tests ก่อน implementation
# Tests อธิบาย behavior ได้ดีที่สุด!
ls src/test/

# Step 5: Trace code จาก entry point
# หา main classes, look at how tests call code
```

### Code Reading Techniques

```kotlin
// เมื่อเจอ code ไม่เข้าใจ:

// 1. ดู test สำหรับ function นั้น
// 2. ดู git log สำหรับ history
// git log --follow -p src/main/kotlin/SomeClass.kt

// 3. ดู usages ด้วย IDE (Find Usages)
// IntelliJ: Alt+F7 / Cmd+F7

// 4. ดู PR ที่เพิ่ม feature นั้น
// git log --all --grep="feature name"

// 5. ถาม maintainer ใน issue/discussion
```

---

## 🚀 4. First PR Strategies

### Documentation PR (ง่ายที่สุด)

```bash
# หา documentation issues
# label:"documentation" or "docs"

# ตัวอย่าง:
# - Fix typo ใน README
# - เพิ่ม example code
# - อัพเดต outdated documentation
# - แปลเป็นภาษาอื่น
# - เพิ่ม missing KDoc

# PR ง่ายๆ แต่ valuable:
```

```kotlin
// Before: missing documentation
fun processOrder(order: Order): OrderResult {
    // implementation
}

// After: added KDoc
/**
 * Processes an order through the complete fulfillment pipeline.
 *
 * @param order The order to process. Must not be null.
 * @return [OrderResult] containing the processing outcome
 * @throws OrderValidationException if order data is invalid
 * @throws InsufficientStockException if product is out of stock
 */
fun processOrder(order: Order): OrderResult {
    // implementation
}
```

### Bug Fix PR

```bash
# 1. Reproduce bug locally
# 2. Write failing test
# 3. Fix the bug
# 4. Verify test passes
# 5. Create PR with:
#    - Description of bug
#    - How to reproduce
#    - Fix explanation
#    - Tests added
```

```kotlin
// Example: Fix found in issue #123
// Bug: NumberFormatException when parsing empty string

// Before (buggy)
fun parseUserId(input: String): Long {
    return input.toLong()  // throws NumberFormatException if empty!
}

// After (fixed)
fun parseUserId(input: String): Long {
    require(input.isNotBlank()) { "User ID cannot be blank" }
    return input.toLongOrNull()
        ?: throw IllegalArgumentException("Invalid user ID format: $input")
}

// Test added
@Test
fun `parseUserId should throw for empty input`() {
    assertThrows<IllegalArgumentException> {
        parseUserId("")
    }
}

@Test
fun `parseUserId should throw for non-numeric input`() {
    assertThrows<IllegalArgumentException> {
        parseUserId("abc")
    }
}
```

### Feature PR

```bash
# สำคัญ: ต้องคุยกับ maintainer ก่อนเสมอ
# สร้าง issue ก่อน → รอ feedback → แล้วค่อย implement

# Template สำหรับ feature request:
# "I would like to add X because Y.
#  Here is my proposed implementation approach: Z.
#  Is this aligned with the project's direction?"
```

---

## 🔧 5. Kotlin/Spring Open Source Ecosystem

### Libraries น่า Contribute

```
Kotlin Core:
- kotlinx.coroutines - Coroutines library
- kotlinx.serialization - JSON/CBOR serialization
- kotlin-stdlib - Standard library

Spring Ecosystem:
- spring-boot - Main framework
- spring-data - Data access
- spring-security - Security
- spring-cloud - Distributed systems

Testing:
- kotest - Test framework for Kotlin
- mockk - Mocking library
- testcontainers-kotlin - Container testing

HTTP Clients:
- ktor - Kotlin web framework
- fuel - HTTP client

Utilities:
- arrow-kt - Functional programming
- exposed - SQL framework for Kotlin
- koin - DI framework
```

---

## 📦 6. สร้าง Library ของตัวเอง

### Library ที่น่าสนใจสร้าง

```
Ideas:
1. Kotlin extension สำหรับ Spring Boot
2. Validation library สำหรับ Kotlin
3. API client wrapper สำหรับ popular service
4. Utility functions สำหรับ domain ที่คุณรู้จัก
5. Spring Boot starter สำหรับ local service
```

### Library Structure

```kotlin
// my-kotlin-lib/build.gradle.kts
plugins {
    kotlin("jvm") version "1.9.21"
    `java-library`
    `maven-publish`
    signing
}

group = "io.github.yourusername"
version = "1.0.0"

publishing {
    publications {
        create<MavenPublication>("mavenJava") {
            from(components["java"])

            pom {
                name.set("My Kotlin Library")
                description.set("Useful utilities for Kotlin")
                url.set("https://github.com/yourusername/my-kotlin-lib")

                licenses {
                    license {
                        name.set("Apache 2.0")
                        url.set("https://opensource.org/licenses/Apache-2.0")
                    }
                }

                developers {
                    developer {
                        id.set("yourusername")
                        name.set("Your Name")
                    }
                }

                scm {
                    url.set("https://github.com/yourusername/my-kotlin-lib")
                }
            }
        }
    }

    repositories {
        maven {
            url = uri("https://s01.oss.sonatype.org/service/local/staging/deploy/maven2/")
            credentials {
                username = System.getenv("OSSRH_USERNAME")
                password = System.getenv("OSSRH_PASSWORD")
            }
        }
    }
}
```

### GitHub Actions สำหรับ CI/CD

```yaml
# .github/workflows/ci.yml
name: CI/CD

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
      - name: Run tests
        run: ./gradlew test

  publish:
    needs: test
    runs-on: ubuntu-latest
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
      - name: Publish to Maven Central
        run: ./gradlew publish
        env:
          OSSRH_USERNAME: ${{ secrets.OSSRH_USERNAME }}
          OSSRH_PASSWORD: ${{ secrets.OSSRH_PASSWORD }}
```

---

## 📋 สรุป

| หัวข้อ | Key Points |
|--------|-----------|
| เริ่มต้น | Contribute ไปที่ tools ที่ใช้จริง |
| First Issue | Documentation, typos, small bugs |
| Code Reading | อ่าน tests ก่อน implementation |
| Feature PR | คุยกับ maintainer ก่อนเสมอ |
| Library | สร้าง something useful ให้ community |
| Publishing | Maven Central สำหรับ JVM libraries |

### เส้นทางการ Contribute

```
1. ใช้ library / เจอ issue
2. อ่าน code และ CONTRIBUTING.md
3. เริ่มจาก documentation fix
4. ทำ small bug fix
5. Propose feature (issue first)
6. Implement feature
7. Become regular contributor
8. Become maintainer
```

---

*Part 98/100+ | Kotlin & Spring Boot Complete Course*
