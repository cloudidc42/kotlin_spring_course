# Part 97: Career Roadmap
## เส้นทางอาชีพ Junior → World-Class Engineer

---

## 🎯 เป้าหมายของ Part นี้

- Career levels และ expectations แต่ละ level
- Skills ที่ต้องมีในแต่ละ level
- Portfolio building
- Open source contribution
- Tech communities ไทยและสากล

---

## 📖 1. Career Levels

### Junior Engineer (0-2 ปี)

```
Responsibilities:
✅ ทำ tasks ที่กำหนดให้ชัดเจน
✅ แก้ bugs ที่ขอบเขตชัดเจน
✅ เรียนรู้ codebase และ patterns
✅ ทำตาม code review feedback
✅ ถามคำถามเมื่อไม่เข้าใจ

Technical Skills:
- เขียน Kotlin/Spring Boot ได้
- ทำ CRUD API ได้
- เข้าใจ Git workflow
- เขียน unit tests ได้
- ใช้ IDE อย่างคล่อง

Red Flags:
❌ ไม่ถามคำถาม (กลัวดูไม่รู้เรื่อง)
❌ ทำงานนานเกิน 1-2 วันโดยไม่ขอ help
❌ ไม่ทำ code review ที่ได้รับ
```

### Mid-level Engineer (2-5 ปี)

```
Responsibilities:
✅ ทำ features จากระดับ requirements
✅ ออกแบบ solution อย่างอิสระ
✅ Mentor junior engineers
✅ Participate ใน technical decisions
✅ Estimate tasks อย่างแม่นยำ

Technical Skills:
- ออกแบบ REST APIs ที่ดี
- ทำ database design
- เข้าใจ caching, async patterns
- เขียน integration tests
- Performance optimization
- Security awareness

Soft Skills:
- Communicate technical ideas ชัดเจน
- ประเมิน risk ได้
- Proactive ในการแก้ปัญหา
```

### Senior Engineer (5-8 ปี)

```
Responsibilities:
✅ ออกแบบ systems ที่ซับซ้อน
✅ Define technical standards สำหรับทีม
✅ Lead ทาง technical ของ features/projects
✅ Identify และจัดการ technical risks
✅ Hire และ onboard engineers

Technical Skills:
- System design (distributed systems)
- Performance profiling และ optimization
- Security architecture
- Cost optimization
- Cross-team technical coordination

Impact:
- ทำให้ทีม 3-5 คน productive มากขึ้น
- Decisions มีผลต่อ system หลายๆ ส่วน
```

### Principal/Staff Engineer (8+ ปี)

```
Responsibilities:
✅ Technical strategy ของ organization
✅ Cross-team technical decisions
✅ Identify long-term technical bets
✅ External representation (conferences, blog)
✅ Shape engineering culture

Impact:
- ทำให้ทีม 20-50+ คน effective มากขึ้น
- Decisions มีผลต่อ roadmap หลายปี
- Influence ที่กว้างกว่า company
```

---

## 📚 2. Learning Path

### Year 1-2: Foundation

```
Quarter 1: Kotlin Fundamentals
  - Complete this course! (Part 1-20)
  - Build 2-3 personal projects
  - Get comfortable with Spring Boot

Quarter 2: Web Development
  - REST APIs (Part 21-30)
  - Security + Testing
  - Deploy to cloud (AWS/GCP)

Quarter 3: Intermediate Patterns
  - Caching, Async, Microservices
  - Database optimization
  - CI/CD basics

Quarter 4: First Job
  - Apply to companies
  - Build portfolio
  - Contribute to open source
```

### Year 2-4: Growth

```
Focus Areas:
- Deep dive ใน 1-2 domains (e-commerce, fintech, etc.)
- System design fundamentals
- Performance optimization
- Mentoring junior members
- Lead features independently

Projects to build:
- Production-level API with proper error handling
- System with high availability requirements
- Something with significant traffic
```

### Year 4+: Senior Track

```
Technical:
- Distributed systems (Kafka, Kubernetes)
- Large-scale architecture
- Platform engineering
- Cross-cutting concerns (observability, security)

Leadership:
- Technical writing (blog, documentation)
- Conference talks
- Open source project ownership
- Mentoring multiple engineers
```

---

## 🗂️ 3. Portfolio Building

### GitHub Portfolio

```markdown
# GitHub Profile Tips

1. Pinned Repositories (6 repos)
   - 1-2 serious projects (production-quality)
   - 1-2 learning projects (showing concepts)
   - 1-2 unique/interesting experiments

2. README.md บน GitHub Profile
   - ใส่ bio, skills, current focus
   - GitHub stats (optional)
   - Links to blog, LinkedIn

3. Green squares (Contribution graph)
   - Commit consistently
   - แม้แต่ README updates นับ

4. Star projects you actually use
   - แสดงว่าคุณ active ใน community
```

### โครงสร้าง Portfolio Project ที่ดี

```markdown
# Good Portfolio Project Structure

## README.md ต้องมี:
- Project description (ทำอะไร ทำไม)
- Demo link หรือ screenshots
- Tech stack
- Architecture diagram (ถ้า complex)
- How to run locally
- API documentation
- What you learned

## Code Quality:
- Proper error handling
- Tests (unit + integration)
- Clean git history
- No hardcoded secrets
- CI/CD (GitHub Actions)

## Example Projects:
1. "E-commerce API" - Spring Boot + Kotlin + PostgreSQL
   Shows: REST API design, auth, caching, testing
   
2. "Real-time Chat" - WebSocket + Redis pub/sub
   Shows: async communication, scalability patterns
   
3. "URL Shortener" - With analytics
   Shows: system design thinking, performance
```

---

## 🌍 4. Open Source Contribution

### ขั้นตอนเริ่มต้น

```bash
# 1. หา project ที่คุณใช้จริง
# Examples:
# - spring-boot, kotlinx.coroutines, Arrow-kt
# - HTTP libraries, testing frameworks

# 2. อ่าน CONTRIBUTING.md ก่อนเสมอ
cat CONTRIBUTING.md

# 3. หา good first issues
# GitHub: filter by label "good first issue" or "help wanted"

# 4. Fork และ clone
git clone https://github.com/YOUR-USERNAME/project.git
cd project

# 5. สร้าง branch สำหรับ fix/feature
git checkout -b fix/issue-description

# 6. Make changes + tests
# 7. Push และ create PR
```

### PR Tips สำหรับ Open Source

```markdown
# Good First PR:
- Fix a typo ใน documentation
- Add missing test case
- Fix warning ที่ compiler แสดง
- Improve error message
- Add example code

# PR Description Template:
## What does this PR do?
Fixes #123

## Why is this change needed?
...

## How was it tested?
- Added unit test for X
- Manually tested by doing Y

## Checklist
- [ ] Tests pass
- [ ] Documentation updated (if needed)
- [ ] Follows code style
```

---

## 👥 5. Tech Communities

### ไทย

```
Online:
- Facebook: "Kotlin Thailand", "Spring Boot Thailand"
- Discord: Thailand Developers
- LINE: Programming groups ต่างๆ

Events:
- BKK.JS / BKK.Android
- Barcamp Bangkok
- AWS User Group Thailand
- Google Developer Group Bangkok
- Devcon Thailand

Job Boards:
- Blognone Jobs
- IT Jobs Thailand
- JobsDB, LinkedIn
```

### สากล

```
Online:
- Kotlin Slack (kotlinlang.slack.com)
- Spring Community Forums
- Stack Overflow
- Reddit: r/Kotlin, r/SpringBoot
- Dev.to, Medium

Conferences:
- KotlinConf
- SpringOne
- Devoxx
- QCon
- Fosdem

Certifications (optional):
- Spring Professional Certification
- AWS Solutions Architect
- CKA (Kubernetes)
```

---

## 💰 6. Salary Expectations (Thailand)

```
Level         | Years | Salary Range (THB/month)
Junior        | 0-2   | 25,000 - 45,000
Mid-level     | 2-5   | 50,000 - 90,000
Senior        | 5-8   | 90,000 - 150,000
Lead/Principal| 8+    | 150,000 - 300,000+

Remote/International:
Junior        | 0-2   | $40,000 - $70,000/year
Mid-level     | 2-5   | $70,000 - $120,000/year
Senior        | 5-8   | $120,000 - $200,000/year
Staff/Principal| 8+   | $200,000 - $400,000+/year

หมายเหตุ: ตัวเลขเหล่านี้เป็น rough estimate ขึ้นกับ:
- Company size/type (startup vs enterprise vs FAANG)
- Skills และ domain expertise
- Location/remote status
- Negotiation skills
```

---

## 📋 สรุป

| Level | Focus | Timeline |
|-------|-------|---------|
| Junior | Learn, deliver assigned tasks | Year 0-2 |
| Mid | Lead features, mentor | Year 2-5 |
| Senior | Design systems, define standards | Year 5-8 |
| Principal | Strategy, cross-team impact | Year 8+ |

### Key Success Factors

```
1. Code ดี: ต้องทำงาน, maintainable, tested
2. Communication: อธิบาย technical ให้ non-technical ฟังเข้าใจ
3. Proactive: หา problem ก่อนมีคนถาม
4. Continuous learning: tech เปลี่ยนเร็ว
5. Build relationships: networking ช่วยมาก
```

---

*Part 97/100+ | Kotlin & Spring Boot Complete Course*
