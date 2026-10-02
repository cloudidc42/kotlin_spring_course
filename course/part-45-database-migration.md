# Part 45: Database Migration (Flyway)
## การจัดการ Schema Changes อย่างปลอดภัยด้วย Flyway

---

## 🎯 เป้าหมายของ Part นี้

- เข้าใจแนวคิด Database Migration
- ติดตั้งและ configure Flyway
- Migration scripts naming convention
- Versioned migrations (V1__, V2__)
- Repeatable migrations (R__)
- Undo migrations
- Best practices สำหรับ production
- ตัวอย่างจริง: Production-safe migrations

---

## 📚 1. ทำไมต้องใช้ Database Migration?

ปัญหาที่เกิดขึ้นบ่อยโดยไม่ใช้ migration tool:

```
Developer A: เพิ่ม column `phone_number` ใน users table
Developer B: ไม่รู้ เลยสร้าง column `phone` แทน

Production: มีทั้ง phone_number และ phone ที่ไม่ได้ใช้
→ ข้อมูลกระจัดกระจาย, bug ยาก debug
```

Flyway แก้ปัญหานี้ด้วย:
- **Version control สำหรับ database schema**
- **Automatic migration เมื่อ application start**
- **Checksum verification** - ป้องกันการแก้ไข migration เก่า
- **Migration history** - รู้ว่า database อยู่ที่ version ไหน

---

## 🔧 2. Dependencies Setup

```kotlin
// build.gradle.kts
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
    implementation("org.flywaydb:flyway-core")
    implementation("org.flywaydb:flyway-mysql")  // สำหรับ MySQL
    // หรือ
    // implementation("org.flywaydb:flyway-database-postgresql")  // สำหรับ PostgreSQL
    
    runtimeOnly("com.mysql:mysql-connector-j")
    // หรือ
    // runtimeOnly("org.postgresql:postgresql")
    
    testImplementation("org.testcontainers:mysql:1.19.3")
    testImplementation("org.testcontainers:junit-jupiter:1.19.3")
}
```

```yaml
# application.yml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/myapp?useSSL=false&allowPublicKeyRetrieval=true
    username: ${DB_USERNAME:root}
    password: ${DB_PASSWORD:password}
    driver-class-name: com.mysql.cj.jdbc.Driver
    hikari:
      maximum-pool-size: 10
      minimum-idle: 5
      connection-timeout: 30000
  
  jpa:
    hibernate:
      ddl-auto: validate  # สำคัญ! ใช้ validate แทน create-drop เมื่อใช้ Flyway
    show-sql: false
    properties:
      hibernate:
        format_sql: true
        dialect: org.hibernate.dialect.MySQLDialect
  
  flyway:
    enabled: true
    locations: classpath:db/migration
    baseline-on-migrate: true     # สำหรับ database ที่มีอยู่แล้ว
    baseline-version: 0
    validate-on-migrate: true     # ตรวจสอบ checksum ก่อน migrate
    out-of-order: false           # ไม่อนุญาต migration out-of-order
    clean-disabled: true          # ป้องกัน flyway:clean ใน production
```

---

## 📁 3. Migration File Structure

```
src/
└── main/
    └── resources/
        └── db/
            └── migration/
                ├── V1__create_initial_schema.sql
                ├── V2__add_user_profile.sql
                ├── V3__create_products_table.sql
                ├── V4__add_order_tables.sql
                ├── V5__add_indexes.sql
                ├── V6__add_audit_columns.sql
                ├── V7__add_soft_delete.sql
                └── R__create_views.sql    (Repeatable)
```

### Naming Convention:

```
Versioned:   V{version}__{description}.sql
Repeatable:  R__{description}.sql
Undo:        U{version}__{description}.sql  (Flyway Pro)

ตัวอย่าง:
V1__create_initial_schema.sql   ✅
V1.1__add_index.sql             ✅ (version 1.1)
V2__add_column.sql              ✅
v2__wrong.sql                   ❌ (lowercase v)
V2-wrong.sql                    ❌ (- แทน __)
```

---

## 📜 4. Migration Scripts

### V1 - Initial Schema

```sql
-- db/migration/V1__create_initial_schema.sql
-- Version: 1
-- Author: Dev Team
-- Date: 2024-01-01
-- Description: สร้าง initial schema สำหรับระบบ user management

-- Users table
CREATE TABLE IF NOT EXISTS users (
    id          BIGINT          NOT NULL AUTO_INCREMENT,
    first_name  VARCHAR(50)     NOT NULL,
    last_name   VARCHAR(50)     NOT NULL,
    email       VARCHAR(255)    NOT NULL,
    password    VARCHAR(255)    NOT NULL,
    active      BOOLEAN         NOT NULL DEFAULT TRUE,
    created_at  DATETIME        NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at  DATETIME        NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    PRIMARY KEY (id),
    UNIQUE KEY uk_users_email (email)
);

-- Roles table
CREATE TABLE IF NOT EXISTS roles (
    id          BIGINT          NOT NULL AUTO_INCREMENT,
    name        VARCHAR(50)     NOT NULL,
    description VARCHAR(255),
    
    PRIMARY KEY (id),
    UNIQUE KEY uk_roles_name (name)
);

-- User-Role junction table
CREATE TABLE IF NOT EXISTS user_roles (
    user_id BIGINT NOT NULL,
    role_id BIGINT NOT NULL,
    
    PRIMARY KEY (user_id, role_id),
    FOREIGN KEY fk_user_roles_user (user_id) REFERENCES users (id) ON DELETE CASCADE,
    FOREIGN KEY fk_user_roles_role (role_id) REFERENCES roles (id) ON DELETE CASCADE
);

-- Insert default roles
INSERT INTO roles (name, description) VALUES
    ('ROLE_USER', 'Standard user'),
    ('ROLE_ADMIN', 'Administrator'),
    ('ROLE_MODERATOR', 'Content moderator');
```

### V2 - Add User Profile

```sql
-- db/migration/V2__add_user_profile.sql
-- Description: เพิ่ม profile information ให้ users

ALTER TABLE users
    ADD COLUMN phone_number     VARCHAR(20)     NULL            AFTER email,
    ADD COLUMN birth_date       DATE            NULL            AFTER phone_number,
    ADD COLUMN bio              TEXT            NULL            AFTER birth_date,
    ADD COLUMN avatar_url       VARCHAR(500)    NULL            AFTER bio;

-- สร้าง table สำหรับ address
CREATE TABLE IF NOT EXISTS user_addresses (
    id          BIGINT          NOT NULL AUTO_INCREMENT,
    user_id     BIGINT          NOT NULL,
    address_line1 VARCHAR(255)  NOT NULL,
    address_line2 VARCHAR(255)  NULL,
    city        VARCHAR(100)    NOT NULL,
    province    VARCHAR(100)    NOT NULL,
    postal_code VARCHAR(10)     NOT NULL,
    country     VARCHAR(2)      NOT NULL DEFAULT 'TH',
    is_default  BOOLEAN         NOT NULL DEFAULT FALSE,
    created_at  DATETIME        NOT NULL DEFAULT CURRENT_TIMESTAMP,
    
    PRIMARY KEY (id),
    FOREIGN KEY fk_addresses_user (user_id) REFERENCES users (id) ON DELETE CASCADE,
    INDEX idx_user_addresses_user (user_id)
);
```

### V3 - Products Table

```sql
-- db/migration/V3__create_products_table.sql
-- Description: สร้าง product catalog

CREATE TABLE IF NOT EXISTS categories (
    id          BIGINT          NOT NULL AUTO_INCREMENT,
    name        VARCHAR(100)    NOT NULL,
    slug        VARCHAR(100)    NOT NULL,
    parent_id   BIGINT          NULL,
    active      BOOLEAN         NOT NULL DEFAULT TRUE,
    sort_order  INT             NOT NULL DEFAULT 0,
    
    PRIMARY KEY (id),
    UNIQUE KEY uk_categories_slug (slug),
    FOREIGN KEY fk_categories_parent (parent_id) REFERENCES categories (id) ON DELETE SET NULL,
    INDEX idx_categories_parent (parent_id)
);

CREATE TABLE IF NOT EXISTS products (
    id              BIGINT          NOT NULL AUTO_INCREMENT,
    category_id     BIGINT          NOT NULL,
    name            VARCHAR(255)    NOT NULL,
    slug            VARCHAR(255)    NOT NULL,
    description     TEXT            NULL,
    price           DECIMAL(10, 2)  NOT NULL,
    stock_quantity  INT             NOT NULL DEFAULT 0,
    sku             VARCHAR(100)    NULL,
    active          BOOLEAN         NOT NULL DEFAULT TRUE,
    created_at      DATETIME        NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at      DATETIME        NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    PRIMARY KEY (id),
    UNIQUE KEY uk_products_slug (slug),
    UNIQUE KEY uk_products_sku (sku),
    FOREIGN KEY fk_products_category (category_id) REFERENCES categories (id),
    INDEX idx_products_category (category_id),
    INDEX idx_products_price (price),
    INDEX idx_products_active (active)
);

-- Insert sample categories
INSERT INTO categories (name, slug, sort_order) VALUES
    ('Electronics', 'electronics', 1),
    ('Clothing', 'clothing', 2),
    ('Books', 'books', 3),
    ('Food & Beverage', 'food-beverage', 4);

INSERT INTO categories (name, slug, parent_id, sort_order) VALUES
    ('Smartphones', 'smartphones', 1, 1),
    ('Laptops', 'laptops', 1, 2),
    ('Tablets', 'tablets', 1, 3);
```

### V4 - Order Tables

```sql
-- db/migration/V4__add_order_tables.sql
-- Description: สร้าง order management tables

CREATE TABLE IF NOT EXISTS orders (
    id              BIGINT          NOT NULL AUTO_INCREMENT,
    user_id         BIGINT          NOT NULL,
    order_number    VARCHAR(50)     NOT NULL,
    status          ENUM(
                        'PENDING',
                        'CONFIRMED',
                        'PROCESSING',
                        'SHIPPED',
                        'DELIVERED',
                        'CANCELLED',
                        'REFUNDED'
                    )               NOT NULL DEFAULT 'PENDING',
    subtotal        DECIMAL(10, 2)  NOT NULL,
    shipping_fee    DECIMAL(10, 2)  NOT NULL DEFAULT 0.00,
    discount_amount DECIMAL(10, 2)  NOT NULL DEFAULT 0.00,
    total_amount    DECIMAL(10, 2)  NOT NULL,
    payment_method  VARCHAR(50)     NULL,
    payment_status  ENUM('PENDING', 'PAID', 'FAILED', 'REFUNDED') NOT NULL DEFAULT 'PENDING',
    shipping_address_id BIGINT      NULL,
    notes           TEXT            NULL,
    created_at      DATETIME        NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at      DATETIME        NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    PRIMARY KEY (id),
    UNIQUE KEY uk_orders_number (order_number),
    FOREIGN KEY fk_orders_user (user_id) REFERENCES users (id),
    INDEX idx_orders_user (user_id),
    INDEX idx_orders_status (status),
    INDEX idx_orders_created (created_at)
);

CREATE TABLE IF NOT EXISTS order_items (
    id              BIGINT          NOT NULL AUTO_INCREMENT,
    order_id        BIGINT          NOT NULL,
    product_id      BIGINT          NOT NULL,
    product_name    VARCHAR(255)    NOT NULL,  -- snapshot ของ product name ณ เวลาที่ order
    product_price   DECIMAL(10, 2)  NOT NULL,  -- snapshot ของ price
    quantity        INT             NOT NULL,
    subtotal        DECIMAL(10, 2)  NOT NULL,
    
    PRIMARY KEY (id),
    FOREIGN KEY fk_items_order (order_id) REFERENCES orders (id) ON DELETE CASCADE,
    FOREIGN KEY fk_items_product (product_id) REFERENCES products (id),
    INDEX idx_order_items_order (order_id)
);
```

### V5 - Performance Indexes

```sql
-- db/migration/V5__add_indexes.sql
-- Description: เพิ่ม indexes สำหรับ query performance

-- Users
CREATE INDEX idx_users_email ON users (email);
CREATE INDEX idx_users_active ON users (active);
CREATE INDEX idx_users_created ON users (created_at);

-- Products full-text search
ALTER TABLE products ADD FULLTEXT INDEX ft_products_search (name, description);

-- Orders compound indexes สำหรับ common queries
CREATE INDEX idx_orders_user_status ON orders (user_id, status);
CREATE INDEX idx_orders_payment ON orders (payment_status, created_at);
```

### V6 - Audit Columns

```sql
-- db/migration/V6__add_audit_columns.sql
-- Description: เพิ่ม audit trail columns

ALTER TABLE users
    ADD COLUMN created_by VARCHAR(255) NULL AFTER updated_at,
    ADD COLUMN updated_by VARCHAR(255) NULL AFTER created_by,
    ADD COLUMN version    BIGINT       NOT NULL DEFAULT 0 AFTER updated_by;

ALTER TABLE products
    ADD COLUMN created_by VARCHAR(255) NULL AFTER updated_at,
    ADD COLUMN updated_by VARCHAR(255) NULL AFTER created_by,
    ADD COLUMN version    BIGINT       NOT NULL DEFAULT 0 AFTER updated_by;

ALTER TABLE orders
    ADD COLUMN created_by VARCHAR(255) NULL AFTER updated_at,
    ADD COLUMN updated_by VARCHAR(255) NULL AFTER created_by;

-- Audit log table
CREATE TABLE IF NOT EXISTS audit_logs (
    id          BIGINT          NOT NULL AUTO_INCREMENT,
    entity_type VARCHAR(100)    NOT NULL,
    entity_id   BIGINT          NOT NULL,
    action      ENUM('CREATE', 'UPDATE', 'DELETE') NOT NULL,
    old_values  JSON            NULL,
    new_values  JSON            NULL,
    changed_by  VARCHAR(255)    NOT NULL,
    changed_at  DATETIME        NOT NULL DEFAULT CURRENT_TIMESTAMP,
    
    PRIMARY KEY (id),
    INDEX idx_audit_entity (entity_type, entity_id),
    INDEX idx_audit_changed_by (changed_by),
    INDEX idx_audit_changed_at (changed_at)
);
```

### V7 - Soft Delete

```sql
-- db/migration/V7__add_soft_delete.sql
-- Description: เปลี่ยนจาก active flag เป็น deleted_at สำหรับ soft delete

-- Users: เพิ่ม deleted_at (null = ไม่ได้ถูกลบ)
ALTER TABLE users
    ADD COLUMN deleted_at DATETIME NULL AFTER updated_at;

-- Migration ข้อมูล: users ที่ active=false จะถูกตั้งเป็น deleted
UPDATE users
SET deleted_at = NOW()
WHERE active = FALSE;

-- Products
ALTER TABLE products
    ADD COLUMN deleted_at DATETIME NULL AFTER updated_at;

-- เพิ่ม index สำหรับ soft delete queries
CREATE INDEX idx_users_deleted ON users (deleted_at);
CREATE INDEX idx_products_deleted ON products (deleted_at);
```

### R - Repeatable Migration (Views)

```sql
-- db/migration/R__create_views.sql
-- Repeatable migrations จะ run อีกครั้งถ้า checksum เปลี่ยน
-- ใช้กับ views, stored procedures, functions

-- Active users view
CREATE OR REPLACE VIEW v_active_users AS
SELECT
    u.id,
    u.first_name,
    u.last_name,
    CONCAT(u.first_name, ' ', u.last_name) AS full_name,
    u.email,
    u.phone_number,
    u.created_at,
    GROUP_CONCAT(r.name ORDER BY r.name) AS roles
FROM users u
LEFT JOIN user_roles ur ON u.id = ur.user_id
LEFT JOIN roles r ON ur.role_id = r.id
WHERE u.deleted_at IS NULL
GROUP BY u.id;

-- Order summary view
CREATE OR REPLACE VIEW v_order_summary AS
SELECT
    o.id,
    o.order_number,
    CONCAT(u.first_name, ' ', u.last_name) AS customer_name,
    u.email AS customer_email,
    o.status,
    o.total_amount,
    o.payment_status,
    o.created_at,
    COUNT(oi.id) AS item_count
FROM orders o
JOIN users u ON o.user_id = u.id
LEFT JOIN order_items oi ON o.id = oi.order_id
GROUP BY o.id;

-- Product stats view
CREATE OR REPLACE VIEW v_product_stats AS
SELECT
    p.id,
    p.name,
    c.name AS category_name,
    p.price,
    p.stock_quantity,
    COALESCE(SUM(oi.quantity), 0) AS total_sold,
    COALESCE(SUM(oi.subtotal), 0) AS total_revenue
FROM products p
JOIN categories c ON p.category_id = c.id
LEFT JOIN order_items oi ON p.id = oi.product_id
LEFT JOIN orders o ON oi.order_id = o.id AND o.status = 'DELIVERED'
WHERE p.deleted_at IS NULL
GROUP BY p.id;
```

---

## 🔒 5. Flyway ใน Kotlin/Spring

```kotlin
// config/FlywayConfig.kt
package com.example.migration.config

import org.flywaydb.core.Flyway
import org.springframework.boot.autoconfigure.flyway.FlywayMigrationInitializer
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.context.annotation.DependsOn
import javax.sql.DataSource

@Configuration
class FlywayConfig {

    /**
     * Custom Flyway configuration
     * สำหรับ cases ที่ต้องการ control มากกว่า default
     */
    @Bean(initMethod = "migrate")
    fun flyway(dataSource: DataSource): Flyway {
        return Flyway.configure()
            .dataSource(dataSource)
            .locations("classpath:db/migration", "classpath:db/test-data")
            .baselineOnMigrate(true)
            .validateOnMigrate(true)
            .outOfOrder(false)
            .load()
    }
}
```

```kotlin
// service/DatabaseMigrationService.kt
package com.example.migration.service

import org.flywaydb.core.Flyway
import org.flywaydb.core.api.output.MigrateResult
import org.springframework.stereotype.Service

@Service
class DatabaseMigrationService(private val flyway: Flyway) {

    fun getMigrationInfo(): List<Map<String, Any?>> {
        return flyway.info().all().map { info ->
            mapOf(
                "version" to info.version?.version,
                "description" to info.description,
                "state" to info.state.name,
                "installedOn" to info.installedOn,
                "executionTime" to info.executionTime,
                "checksum" to info.checksum
            )
        }
    }

    fun getCurrentVersion(): String? {
        return flyway.info().current()?.version?.version
    }

    fun getPendingMigrations(): Int {
        return flyway.info().pending().size
    }

    fun validate() {
        flyway.validate()
    }
}
```

---

## 🧪 6. Testing Migrations

```kotlin
// test/FlywayMigrationTest.kt
package com.example.migration

import org.flywaydb.core.Flyway
import org.junit.jupiter.api.Test
import org.springframework.beans.factory.annotation.Autowired
import org.springframework.boot.test.context.SpringBootTest
import org.springframework.jdbc.core.JdbcTemplate
import org.testcontainers.containers.MySQLContainer
import org.testcontainers.junit.jupiter.Container
import org.testcontainers.junit.jupiter.Testcontainers
import kotlin.test.assertEquals
import kotlin.test.assertNotNull

@SpringBootTest
@Testcontainers
class FlywayMigrationTest {

    companion object {
        @Container
        val mysql = MySQLContainer<Nothing>("mysql:8.0").apply {
            withDatabaseName("testdb")
            withUsername("test")
            withPassword("test")
        }
    }

    @Autowired
    private lateinit var flyway: Flyway

    @Autowired
    private lateinit var jdbcTemplate: JdbcTemplate

    @Test
    fun `all migrations should succeed`() {
        val info = flyway.info()
        val pending = info.pending()
        assertEquals(0, pending.size, "ควรไม่มี pending migrations")

        val applied = info.applied()
        assert(applied.isNotEmpty()) { "ควรมี applied migrations" }
    }

    @Test
    fun `users table should have correct columns`() {
        val columns = jdbcTemplate.queryForList("""
            SELECT COLUMN_NAME, DATA_TYPE, IS_NULLABLE
            FROM INFORMATION_SCHEMA.COLUMNS
            WHERE TABLE_NAME = 'users'
            ORDER BY ORDINAL_POSITION
        """)

        val columnNames = columns.map { it["COLUMN_NAME"] }
        assert("id" in columnNames)
        assert("first_name" in columnNames)
        assert("last_name" in columnNames)
        assert("email" in columnNames)
        assert("phone_number" in columnNames)  // เพิ่มใน V2
        assert("deleted_at" in columnNames)    // เพิ่มใน V7
    }

    @Test
    fun `default roles should be seeded`() {
        val roleCount = jdbcTemplate.queryForObject(
            "SELECT COUNT(*) FROM roles", Int::class.java
        )
        assert(roleCount!! >= 3) { "ควรมีอย่างน้อย 3 roles" }
    }

    @Test
    fun `views should exist`() {
        val views = jdbcTemplate.queryForList("""
            SELECT TABLE_NAME FROM INFORMATION_SCHEMA.VIEWS
            WHERE TABLE_SCHEMA = DATABASE()
        """).map { it["TABLE_NAME"] }

        assert("v_active_users" in views)
        assert("v_order_summary" in views)
    }
}
```

---

## 🏭 7. Production Migration Strategy

```yaml
# application-prod.yml
spring:
  flyway:
    enabled: true
    clean-disabled: true      # ห้าม clean production DB
    validate-on-migrate: true
    baseline-on-migrate: false  # production ไม่ควรใช้ baseline
    out-of-order: false
    # ล็อค migration เพื่อกัน concurrent deployment
    lock-retry-count: 10

# application-dev.yml
spring:
  flyway:
    enabled: true
    clean-disabled: false      # development ทำ clean ได้
```

```bash
# Scripts สำหรับ manual migration (ถ้าจำเป็น)

# Check migration status
./mvnw flyway:info -Dflyway.url=jdbc:mysql://prod:3306/myapp

# Validate migrations
./mvnw flyway:validate

# Repair (แก้ failed migrations)
./mvnw flyway:repair

# ห้าม! ใน production:
# ./mvnw flyway:clean  ← จะลบข้อมูลทั้งหมด!
```

---

## 📊 สรุปเนื้อหา

| ประเภท Migration | Prefix | เมื่อไหร่ run |
|-----------------|--------|-------------|
| Versioned | V1__, V2__ | รัน 1 ครั้งเท่านั้น |
| Repeatable | R__ | รันทุกครั้งที่ checksum เปลี่ยน |
| Undo (Pro) | U1__, U2__ | Manual rollback |

### Migration States:

| State | ความหมาย |
|-------|---------|
| PENDING | ยังไม่ได้ run |
| SUCCESS | Run สำเร็จ |
| FAILED | Run ล้มเหลว |
| MISSING | ไม่พบ script (อาจถูกลบ) |
| IGNORED | ถูก ignore |

### Best Practices:

1. **Never modify** applied migrations - สร้างใหม่แทน
2. **Test locally** ก่อน push ทุกครั้ง
3. **Backup** database ก่อน run migration ใน production
4. **Idempotent** - ใช้ `CREATE TABLE IF NOT EXISTS`
5. **Small migrations** - แต่ละ migration ควรทำสิ่งเดียว
6. **Data migrations** แยกจาก schema changes

---

*Part 45/100+ | Kotlin & Spring Boot Complete Course*
