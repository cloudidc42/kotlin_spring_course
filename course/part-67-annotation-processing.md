# Part 67: Annotation Processing (KAPT/KSP)

## Annotation Processing — สร้าง Code Generation อัตโนมัติด้วย KSP

---

## 🎯 เป้าหมายของ Part นี้

- เข้าใจความแตกต่างระหว่าง KAPT และ KSP
- เรียนรู้หลักการ Kotlin Symbol Processing (KSP)
- สร้าง custom annotations
- สร้าง code generator ด้วย KSP
- สร้าง Builder pattern อัตโนมัติ

---

## 📖 1. KAPT vs KSP

### KAPT (Kotlin Annotation Processing Tool)
KAPT เป็น annotation processor รุ่นเก่าที่ทำงานผ่าน Java APT

ข้อเสีย:
- ช้ากว่าเพราะต้อง compile เป็น Java stubs ก่อน
- ไม่รองรับ incremental processing ได้ดี
- อนาคตจะถูกแทนที่ด้วย KSP

### KSP (Kotlin Symbol Processing)
KSP เป็น API ใหม่ที่ออกแบบมาสำหรับ Kotlin โดยเฉพาะ

ข้อดี:
- เร็วกว่า KAPT 2-3 เท่า
- รองรับ incremental processing
- เข้าใจ Kotlin types ได้โดยตรง (nullability, suspend, etc.)
- รองรับ multiplatform

---

## ⚙️ 2. ตั้งค่า KSP

### build.gradle.kts (processor module)

```kotlin
plugins {
    kotlin("jvm") version "1.9.20"
    id("com.google.devtools.ksp") version "1.9.20-1.0.14"
}

dependencies {
    implementation("com.google.devtools.ksp:symbol-processing-api:1.9.20-1.0.14")
}
```

### build.gradle.kts (app module)

```kotlin
plugins {
    kotlin("jvm") version "1.9.20"
    id("com.google.devtools.ksp") version "1.9.20-1.0.14"
}

dependencies {
    ksp(project(":processor"))
    implementation(project(":annotations"))
}
```

### settings.gradle.kts

```kotlin
rootProject.name = "ksp-demo"
include(":app", ":processor", ":annotations")
```

---

## 🏷️ 3. Custom Annotations

สร้าง module `annotations` สำหรับ annotation definitions

```kotlin
// annotations/src/main/kotlin/AutoBuilder.kt

/**
 * สร้าง Builder class อัตโนมัติสำหรับ data class
 */
@Target(AnnotationTarget.CLASS)
@Retention(AnnotationRetention.SOURCE)
annotation class AutoBuilder

/**
 * สร้าง toString() method แบบ custom
 */
@Target(AnnotationTarget.CLASS)
@Retention(AnnotationRetention.SOURCE)
annotation class AutoToString(
    val includeFields: Array<String> = [],
    val excludeFields: Array<String> = []
)

/**
 * Mark field ว่าไม่ต้องรวมใน Builder
 */
@Target(AnnotationTarget.PROPERTY)
@Retention(AnnotationRetention.SOURCE)
annotation class Exclude

/**
 * กำหนด default value สำหรับ Builder field
 */
@Target(AnnotationTarget.PROPERTY)
@Retention(AnnotationRetention.SOURCE)
annotation class Default(val value: String)
```

---

## 🔧 4. KSP Processor

```kotlin
// processor/src/main/kotlin/AutoBuilderProcessor.kt

import com.google.devtools.ksp.processing.*
import com.google.devtools.ksp.symbol.*
import com.google.devtools.ksp.validate
import java.io.OutputStream

class AutoBuilderProcessor(
    private val codeGenerator: CodeGenerator,
    private val logger: KSPLogger
) : SymbolProcessor {

    override fun process(resolver: Resolver): List<KSAnnotated> {
        val symbols = resolver
            .getSymbolsWithAnnotation("AutoBuilder")
            .filterIsInstance<KSClassDeclaration>()

        val unprocessed = mutableListOf<KSAnnotated>()

        symbols.forEach { classDeclaration ->
            if (classDeclaration.validate()) {
                classDeclaration.accept(AutoBuilderVisitor(), Unit)
            } else {
                unprocessed.add(classDeclaration)
            }
        }

        return unprocessed
    }

    inner class AutoBuilderVisitor : KSVisitorVoid() {
        override fun visitClassDeclaration(classDeclaration: KSClassDeclaration, data: Unit) {
            if (classDeclaration.classKind != ClassKind.CLASS &&
                classDeclaration.classKind != ClassKind.DATA_CLASS) {
                logger.error("@AutoBuilder can only be applied to classes", classDeclaration)
                return
            }

            val packageName = classDeclaration.packageName.asString()
            val className = classDeclaration.simpleName.asString()
            val builderName = "${className}Builder"

            val file = codeGenerator.createNewFile(
                dependencies = Dependencies(false, classDeclaration.containingFile!!),
                packageName = packageName,
                fileName = builderName
            )

            generateBuilder(file, classDeclaration, packageName, className, builderName)
        }

        private fun generateBuilder(
            file: OutputStream,
            classDeclaration: KSClassDeclaration,
            packageName: String,
            className: String,
            builderName: String
        ) {
            val properties = classDeclaration.getAllProperties()
                .filter { prop ->
                    prop.annotations.none { ann ->
                        ann.shortName.asString() == "Exclude"
                    }
                }
                .toList()

            file.write("""
package $packageName

class $builderName {
${properties.joinToString("\n") { prop ->
    val propName = prop.simpleName.asString()
    val propType = prop.type.resolve()
    val isNullable = propType.isMarkedNullable
    val typeName = propType.declaration.simpleName.asString()
    val nullMark = if (isNullable) "?" else ""
    "    private var $propName: $typeName$nullMark = ${getDefaultValue(prop, typeName, isNullable)}"
}}

${properties.joinToString("\n") { prop ->
    val propName = prop.simpleName.asString()
    val propType = prop.type.resolve()
    val typeName = propType.declaration.simpleName.asString()
    val isNullable = propType.isMarkedNullable
    val nullMark = if (isNullable) "?" else ""
    "    fun $propName(value: $typeName$nullMark): $builderName = apply { this.$propName = value }"
}}

    fun build(): $className = $className(
${properties.joinToString(",\n") { prop ->
    val propName = prop.simpleName.asString()
    "        $propName = $propName"
}}
    )
}

fun $className.Companion.builder(): $builderName = $builderName()
""".trimIndent().toByteArray())
        }

        private fun getDefaultValue(
            prop: KSPropertyDeclaration,
            typeName: String,
            isNullable: Boolean
        ): String {
            val defaultAnnotation = prop.annotations
                .find { it.shortName.asString() == "Default" }

            if (defaultAnnotation != null) {
                return defaultAnnotation.arguments.first().value.toString()
            }

            if (isNullable) return "null"

            return when (typeName) {
                "String" -> "\"\""
                "Int" -> "0"
                "Long" -> "0L"
                "Double" -> "0.0"
                "Float" -> "0.0f"
                "Boolean" -> "false"
                "List" -> "emptyList()"
                else -> "TODO(\"provide default for $typeName\")"
            }
        }
    }
}
```

### Provider สำหรับ Processor

```kotlin
// processor/src/main/kotlin/AutoBuilderProcessorProvider.kt

import com.google.devtools.ksp.processing.SymbolProcessor
import com.google.devtools.ksp.processing.SymbolProcessorEnvironment
import com.google.devtools.ksp.processing.SymbolProcessorProvider

class AutoBuilderProcessorProvider : SymbolProcessorProvider {
    override fun create(environment: SymbolProcessorEnvironment): SymbolProcessor {
        return AutoBuilderProcessor(
            codeGenerator = environment.codeGenerator,
            logger = environment.logger
        )
    }
}
```

### ลงทะเบียน Provider

```
# processor/src/main/resources/META-INF/services/
# com.google.devtools.ksp.processing.SymbolProcessorProvider

com.example.processor.AutoBuilderProcessorProvider
```

---

## 📝 5. ใช้งาน @AutoBuilder

```kotlin
// app/src/main/kotlin/models/User.kt

@AutoBuilder
data class User(
    val id: Long,
    val name: String,
    val email: String,
    @Default("USER")
    val role: String,
    @Exclude
    val passwordHash: String = ""
)
```

โค้ดที่ถูก generate อัตโนมัติ:

```kotlin
// build/generated/ksp/main/kotlin/UserBuilder.kt (generated)

class UserBuilder {
    private var id: Long = 0L
    private var name: String = ""
    private var email: String = ""
    private var role: String = "USER"

    fun id(value: Long): UserBuilder = apply { this.id = value }
    fun name(value: String): UserBuilder = apply { this.name = value }
    fun email(value: String): UserBuilder = apply { this.email = value }
    fun role(value: String): UserBuilder = apply { this.role = value }

    fun build(): User = User(
        id = id,
        name = name,
        email = email,
        role = role
    )
}

fun User.Companion.builder(): UserBuilder = UserBuilder()
```

### ใช้งาน Builder ที่ generate แล้ว

```kotlin
fun main() {
    val user = User.builder()
        .id(1L)
        .name("สมชาย ใจดี")
        .email("somchai@example.com")
        .role("ADMIN")
        .build()

    println(user)
    // User(id=1, name=สมชาย ใจดี, email=somchai@example.com, role=ADMIN, passwordHash=)
}
```

---

## 🏗️ 6. Advanced: Multiple Annotations

```kotlin
// สร้าง annotation สำหรับ Spring Repository generation
@Target(AnnotationTarget.CLASS)
@Retention(AnnotationRetention.SOURCE)
annotation class GenerateRepository(
    val entityClass: KClass<*>,
    val generateFindByName: Boolean = true,
    val generateFindByEmail: Boolean = false
)

// สร้าง annotation สำหรับ DTO generation
@Target(AnnotationTarget.CLASS)
@Retention(AnnotationRetention.SOURCE)
annotation class GenerateDto(
    val suffix: String = "Dto",
    val excludeFields: Array<String> = []
)
```

```kotlin
// ตัวอย่างการใช้งาน
@AutoBuilder
@GenerateDto(excludeFields = ["passwordHash", "internalNotes"])
data class Product(
    val id: Long,
    val name: String,
    val price: Double,
    val description: String,
    val passwordHash: String = "",
    val internalNotes: String = ""
)
```

---

## 🔬 7. KSP Type System

```kotlin
// การทำงานกับ KSP Type System
class TypeAnalyzer {

    fun analyzeProperty(prop: KSPropertyDeclaration): PropertyInfo {
        val resolvedType = prop.type.resolve()
        
        return PropertyInfo(
            name = prop.simpleName.asString(),
            typeName = resolvedType.declaration.simpleName.asString(),
            packageName = resolvedType.declaration.packageName.asString(),
            isNullable = resolvedType.isMarkedNullable,
            typeArguments = resolvedType.arguments.map { arg ->
                arg.type?.resolve()?.declaration?.simpleName?.asString() ?: "*"
            },
            annotations = prop.annotations.map { ann ->
                ann.shortName.asString()
            }.toList()
        )
    }

    fun isKotlinBuiltIn(packageName: String): Boolean {
        return packageName.startsWith("kotlin") || packageName.startsWith("java")
    }
}

data class PropertyInfo(
    val name: String,
    val typeName: String,
    val packageName: String,
    val isNullable: Boolean,
    val typeArguments: List<String>,
    val annotations: List<String>
)
```

---

## 📊 8. สรุปตาราง KAPT vs KSP

| Feature | KAPT | KSP |
|---------|------|-----|
| ความเร็ว | ช้า (Java stubs) | เร็ว 2-3x |
| Incremental | จำกัด | รองรับเต็มที่ |
| Kotlin types | ผ่าน Java mirror | โดยตรง |
| Nullability | ไม่รู้ | รู้ |
| Multiplatform | ไม่รองรับ | รองรับ |
| API | Java APT | Kotlin-first |
| Status | Legacy | แนะนำ |

---

## 💡 Best Practices

1. **ใช้ KSP แทน KAPT** สำหรับ processor ใหม่ทั้งหมด
2. **แยก module** annotations, processor, และ app ออกจากกัน
3. **Generate source files** ไปที่ `build/generated` ไม่ใช่ source tree
4. **Validate symbols** ก่อน process เสมอเพื่อหลีกเลี่ยง errors
5. **ใช้ incremental processing** เพื่อความเร็วใน build

---

## 🧪 9. Testing Annotation Processors

```kotlin
// ใช้ compile-testing library
testImplementation("com.github.tschuchortdev:kotlin-compile-testing-ksp:1.5.0")

class AutoBuilderProcessorTest {

    @Test
    fun `generates builder for annotated class`() {
        val result = KotlinCompilation().apply {
            sources = listOf(
                SourceFile.kotlin("User.kt", """
                    @AutoBuilder
                    data class User(
                        val name: String,
                        val email: String
                    )
                """)
            )
            symbolProcessorProviders = listOf(AutoBuilderProcessorProvider())
            inheritClassPath = true
        }.compile()

        assertThat(result.exitCode).isEqualTo(KotlinCompilation.ExitCode.OK)
        
        // ตรวจสอบว่า generated file มีเนื้อหาที่ถูกต้อง
        val generatedFile = result.generatedFiles.find { it.name == "UserBuilder.kt" }
        assertThat(generatedFile).isNotNull()
        assertThat(generatedFile!!.readText()).contains("class UserBuilder")
        assertThat(generatedFile.readText()).contains("fun name(value: String)")
        assertThat(generatedFile.readText()).contains("fun email(value: String)")
        assertThat(generatedFile.readText()).contains("fun build(): User")
    }

    @Test
    fun `fails compilation when applied to interface`() {
        val result = KotlinCompilation().apply {
            sources = listOf(
                SourceFile.kotlin("MyInterface.kt", """
                    @AutoBuilder
                    interface MyInterface
                """)
            )
            symbolProcessorProviders = listOf(AutoBuilderProcessorProvider())
            inheritClassPath = true
        }.compile()

        assertThat(result.exitCode).isEqualTo(KotlinCompilation.ExitCode.COMPILATION_ERROR)
    }
}
```

---

## 🔍 10. Real-world KSP: Room-like ORM

สร้าง annotation processor ที่ generate SQL queries อัตโนมัติ

```kotlin
// annotations
@Target(AnnotationTarget.CLASS)
@Retention(AnnotationRetention.SOURCE)
annotation class Entity(val tableName: String = "")

@Target(AnnotationTarget.PROPERTY)
@Retention(AnnotationRetention.SOURCE)
annotation class PrimaryKey

@Target(AnnotationTarget.PROPERTY)
@Retention(AnnotationRetention.SOURCE)
annotation class Column(val name: String = "")

// example usage
@Entity("products")
data class Product(
    @PrimaryKey
    val id: Long = 0,
    
    @Column("product_name")
    val name: String,
    
    @Column("unit_price")
    val price: Double
)
```

### Processor generates SQL

```kotlin
class EntityProcessorVisitor(
    private val codeGenerator: CodeGenerator
) : KSVisitorVoid() {

    override fun visitClassDeclaration(classDeclaration: KSClassDeclaration, data: Unit) {
        val entityAnnotation = classDeclaration.annotations
            .find { it.shortName.asString() == "Entity" }!!
        
        val tableName = entityAnnotation.arguments
            .find { it.name?.asString() == "tableName" }
            ?.value?.toString()
            ?.takeIf { it.isNotBlank() }
            ?: classDeclaration.simpleName.asString().lowercase()

        val properties = classDeclaration.getAllProperties().toList()
        val pkProperty = properties.find { prop ->
            prop.annotations.any { it.shortName.asString() == "PrimaryKey" }
        }

        val className = classDeclaration.simpleName.asString()
        val packageName = classDeclaration.packageName.asString()

        val file = codeGenerator.createNewFile(
            Dependencies(false, classDeclaration.containingFile!!),
            packageName,
            "${className}Queries"
        )

        file.write(generateQueryClass(className, tableName, pkProperty, properties).toByteArray())
    }

    private fun generateQueryClass(
        className: String,
        tableName: String,
        pk: KSPropertyDeclaration?,
        properties: List<KSPropertyDeclaration>
    ): String {
        val columns = properties.map { prop ->
            val colAnnotation = prop.annotations.find { it.shortName.asString() == "Column" }
            val colName = colAnnotation?.arguments?.find { it.name?.asString() == "name" }
                ?.value?.toString()?.takeIf { it.isNotBlank() }
                ?: prop.simpleName.asString()
            prop.simpleName.asString() to colName
        }

        return """
            object ${className}Queries {
                const val TABLE = "$tableName"
                const val SELECT_ALL = "SELECT * FROM $tableName"
                const val SELECT_BY_ID = "SELECT * FROM $tableName WHERE ${pk?.simpleName?.asString() ?: "id"} = ?"
                const val INSERT = "INSERT INTO $tableName (${columns.map { it.second }.joinToString()}) VALUES (${columns.map { "?" }.joinToString()})"
                const val DELETE_BY_ID = "DELETE FROM $tableName WHERE ${pk?.simpleName?.asString() ?: "id"} = ?"
            }
        """.trimIndent()
    }
}
```

---

*Part 67/100+ | Kotlin & Spring Boot Complete Course*
