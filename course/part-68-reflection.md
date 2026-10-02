# Part 68: Reflection ใน Kotlin

## Kotlin Reflection — ตรวจสอบและเรียกใช้งาน Code ณ Runtime

---

## 🎯 เป้าหมายของ Part นี้

- เข้าใจ Kotlin Reflection API (KClass, KFunction, KProperty)
- เรียกใช้งาน functions และ properties ผ่าน reflection
- อ่าน annotations ณ runtime
- สร้าง generic mapper ด้วย reflection
- Data class introspection

---

## 📖 1. Kotlin Reflection คืออะไร?

Reflection คือความสามารถของโปรแกรมในการ inspect และ modify ตัวเองณ runtime โดยไม่รู้ข้อมูลณ compile time

```kotlin
// เพิ่ม dependency ใน build.gradle.kts
dependencies {
    implementation(kotlin("reflect"))
}
```

---

## 🔍 2. KClass — Class Reference

```kotlin
import kotlin.reflect.*
import kotlin.reflect.full.*

data class Person(
    val name: String,
    val age: Int,
    val email: String
)

fun main() {
    // วิธีได้ KClass
    val clazz1 = Person::class
    val clazz2 = Person("สมชาย", 30, "test@test.com")::class
    val clazz3 = Class.forName("Person").kotlin

    println(clazz1.simpleName)        // Person
    println(clazz1.qualifiedName)     // com.example.Person
    println(clazz1.isData)            // true
    println(clazz1.isAbstract)        // false
    println(clazz1.isFinal)           // true

    // Primary constructor
    val constructor = clazz1.primaryConstructor
    println(constructor?.parameters?.map { it.name })
    // [name, age, email]

    // Properties
    clazz1.memberProperties.forEach { prop ->
        println("${prop.name}: ${prop.returnType}")
    }
}
```

---

## ⚡ 3. KFunction — Function Reference

```kotlin
import kotlin.reflect.*
import kotlin.reflect.full.*

class Calculator {
    fun add(a: Int, b: Int): Int = a + b
    fun multiply(a: Int, b: Int): Int = a * b
    suspend fun asyncCompute(value: Int): String = "Result: $value"
}

fun main() {
    val calc = Calculator()
    val calcClass = Calculator::class

    // เรียกใช้ function ผ่าน reference
    val addFunc = calcClass.memberFunctions.find { it.name == "add" }!!
    val result = addFunc.call(calc, 5, 3)
    println(result) // 8

    // ดูข้อมูล function
    calcClass.memberFunctions.forEach { func ->
        println("${func.name}(${func.parameters.drop(1).joinToString { it.name ?: "?" }}) -> ${func.returnType}")
    }

    // Function reference แบบ direct
    val addRef: (Int, Int) -> Int = calc::add
    println(addRef(10, 20)) // 30

    // Top-level function reference
    fun greet(name: String): String = "Hello, $name!"
    val greetRef = ::greet
    println(greetRef.name)       // greet
    println(greetRef("World"))   // Hello, World!
}
```

### callBy() — Named Arguments

```kotlin
import kotlin.reflect.full.*

data class Config(
    val host: String = "localhost",
    val port: Int = 8080,
    val debug: Boolean = false
)

fun main() {
    val constructor = Config::class.primaryConstructor!!

    // callBy ใช้ named parameters
    val params = mapOf(
        constructor.parameters.find { it.name == "host" }!! to "db.example.com",
        constructor.parameters.find { it.name == "port" }!! to 5432
        // debug ใช้ค่า default
    )

    val config = constructor.callBy(params)
    println(config) // Config(host=db.example.com, port=5432, debug=false)
}
```

---

## 🏷️ 4. KProperty — Property Reference

```kotlin
import kotlin.reflect.*
import kotlin.reflect.full.*

data class User(
    val id: Long,
    var name: String,
    var email: String,
    val createdAt: Long = System.currentTimeMillis()
)

fun main() {
    val user = User(1L, "สมชาย", "somchai@example.com")

    // อ่านค่า property
    val nameProperty = User::name
    println(nameProperty.get(user)) // สมชาย
    println(nameProperty.name)       // name
    println(nameProperty.returnType) // kotlin.String

    // เขียนค่า property (ต้องเป็น KMutableProperty)
    if (nameProperty is KMutableProperty1) {
        nameProperty.set(user, "วิทยา ใจดี")
        println(user.name) // วิทยา ใจดี
    }

    // ดู visibility
    User::class.memberProperties.forEach { prop ->
        println("${prop.name}: visibility=${prop.visibility}, mutable=${prop is KMutableProperty1}")
    }
}
```

---

## 📌 5. Annotations Reflection

```kotlin
import kotlin.reflect.*
import kotlin.reflect.full.*

@Target(AnnotationTarget.CLASS, AnnotationTarget.PROPERTY)
@Retention(AnnotationRetention.RUNTIME)
annotation class Validate(val message: String = "Invalid value")

@Target(AnnotationTarget.PROPERTY)
@Retention(AnnotationRetention.RUNTIME)
annotation class MinLength(val value: Int)

@Target(AnnotationTarget.PROPERTY)
@Retention(AnnotationRetention.RUNTIME)
annotation class Email

@Validate
data class UserRequest(
    @MinLength(2)
    val name: String,
    
    @Email
    val email: String,
    
    @MinLength(8)
    val password: String
)

class ReflectionValidator {

    fun validate(obj: Any): List<String> {
        val errors = mutableListOf<String>()
        val kClass = obj::class

        // ตรวจสอบ class-level annotation
        val classValidate = kClass.findAnnotation<Validate>()
        if (classValidate == null) {
            return listOf("Class ${kClass.simpleName} is not marked with @Validate")
        }

        kClass.memberProperties.forEach { prop ->
            val value = prop.getter.call(obj)

            // ตรวจสอบ @MinLength
            prop.findAnnotation<MinLength>()?.let { annotation ->
                val strValue = value?.toString() ?: ""
                if (strValue.length < annotation.value) {
                    errors.add("${prop.name} must be at least ${annotation.value} characters")
                }
            }

            // ตรวจสอบ @Email
            prop.findAnnotation<Email>()?.let {
                val strValue = value?.toString() ?: ""
                if (!strValue.contains("@")) {
                    errors.add("${prop.name} must be a valid email address")
                }
            }
        }

        return errors
    }
}

fun main() {
    val validator = ReflectionValidator()

    val invalidUser = UserRequest(
        name = "A",
        email = "not-an-email",
        password = "short"
    )

    val errors = validator.validate(invalidUser)
    errors.forEach { println("Error: $it") }
}
```

---

## 🗺️ 6. Data Class Introspection

```kotlin
import kotlin.reflect.*
import kotlin.reflect.full.*

// Generic utility สำหรับ data class
object DataClassUtils {

    fun <T : Any> toMap(obj: T): Map<String, Any?> {
        require(obj::class.isData) { "${obj::class.simpleName} is not a data class" }
        return obj::class.memberProperties.associate { prop ->
            prop.name to prop.getter.call(obj)
        }
    }

    fun <T : Any> copyWith(obj: T, changes: Map<String, Any?>): T {
        val kClass = obj::class
        require(kClass.isData) { "${kClass.simpleName} is not a data class" }

        val constructor = kClass.primaryConstructor!!
        val params = constructor.parameters.associateWith { param ->
            changes.getOrElse(param.name!!) {
                kClass.memberProperties.find { it.name == param.name }?.getter?.call(obj)
            }
        }

        @Suppress("UNCHECKED_CAST")
        return constructor.callBy(params) as T
    }

    fun <T : Any> diff(obj1: T, obj2: T): Map<String, Pair<Any?, Any?>> {
        val kClass = obj1::class
        require(kClass.isData) { "${kClass.simpleName} is not a data class" }

        return kClass.memberProperties
            .filter { prop -> prop.getter.call(obj1) != prop.getter.call(obj2) }
            .associate { prop ->
                prop.name to (prop.getter.call(obj1) to prop.getter.call(obj2))
            }
    }
}

data class Product(
    val id: Long,
    val name: String,
    val price: Double,
    val stock: Int
)

fun main() {
    val product = Product(1L, "Laptop", 25000.0, 50)

    // แปลงเป็น Map
    val map = DataClassUtils.toMap(product)
    println(map) // {id=1, name=Laptop, price=25000.0, stock=50}

    // Copy with changes
    val updated = DataClassUtils.copyWith(product, mapOf("price" to 22000.0, "stock" to 48))
    println(updated) // Product(id=1, name=Laptop, price=22000.0, stock=48)

    // เปรียบเทียบความแตกต่าง
    val diff = DataClassUtils.diff(product, updated)
    diff.forEach { (field, change) ->
        println("$field: ${change.first} -> ${change.second}")
    }
}
```

---

## 🗄️ 7. Generic Mapper ด้วย Reflection

```kotlin
import kotlin.reflect.*
import kotlin.reflect.full.*

class ReflectionMapper {

    fun <FROM : Any, TO : Any> map(source: FROM, targetClass: KClass<TO>): TO {
        val sourceProps = source::class.memberProperties
            .associate { it.name to it.getter.call(source) }

        val constructor = targetClass.primaryConstructor
            ?: throw IllegalArgumentException("${targetClass.simpleName} has no primary constructor")

        val params = constructor.parameters.associateWith { param ->
            val value = sourceProps[param.name]
            if (value != null && !param.type.isMarkedNullable) {
                convertType(value, param.type)
            } else {
                value
            }
        }

        return constructor.callBy(params)
    }

    private fun convertType(value: Any, targetType: KType): Any {
        val targetClass = targetType.classifier as? KClass<*> ?: return value
        if (targetClass.isInstance(value)) return value

        return when (targetClass) {
            String::class -> value.toString()
            Int::class -> value.toString().toInt()
            Long::class -> value.toString().toLong()
            Double::class -> value.toString().toDouble()
            Boolean::class -> value.toString().toBoolean()
            else -> value
        }
    }
}

// Entity และ DTO
data class UserEntity(
    val id: Long,
    val firstName: String,
    val lastName: String,
    val emailAddress: String
)

data class UserDto(
    val id: Long,
    val firstName: String,
    val lastName: String,
    val emailAddress: String
)

fun main() {
    val mapper = ReflectionMapper()
    
    val entity = UserEntity(1L, "สมชาย", "ใจดี", "somchai@example.com")
    val dto = mapper.map(entity, UserDto::class)
    
    println(dto) // UserDto(id=1, firstName=สมชาย, lastName=ใจดี, emailAddress=somchai@example.com)
}
```

---

## 🔗 8. Spring Boot Integration

```kotlin
import kotlin.reflect.*
import kotlin.reflect.full.*
import org.springframework.stereotype.Component
import org.springframework.core.convert.converter.Converter

@Component
class GenericEntityMapper {

    final inline fun <reified FROM : Any, reified TO : Any> map(source: FROM): TO {
        return map(source, TO::class)
    }

    fun <FROM : Any, TO : Any> map(source: FROM, targetClass: KClass<TO>): TO {
        val sourceMap = extractProperties(source)
        return instantiate(targetClass, sourceMap)
    }

    fun <T : Any> mapList(sources: List<T>, targetClass: KClass<*>): List<Any> {
        return sources.map { source -> map(source, targetClass as KClass<Any>) }
    }

    private fun <T : Any> extractProperties(obj: T): Map<String, Any?> {
        return obj::class.memberProperties.associate { prop ->
            prop.name to prop.getter.call(obj)
        }
    }

    private fun <T : Any> instantiate(kClass: KClass<T>, values: Map<String, Any?>): T {
        val constructor = kClass.primaryConstructor
            ?: throw IllegalStateException("No primary constructor for ${kClass.simpleName}")

        val args = constructor.parameters.associateWith { param ->
            values[param.name]
        }

        return constructor.callBy(args)
    }
}
```

---

## 📊 9. Performance Considerations

```kotlin
import kotlin.reflect.*
import kotlin.reflect.full.*

// Cache reflection results เพื่อประสิทธิภาพ
object ReflectionCache {
    private val propertyCache = HashMap<KClass<*>, List<KProperty1<Any, *>>>()
    private val constructorCache = HashMap<KClass<*>, KFunction<*>>()

    @Suppress("UNCHECKED_CAST")
    fun <T : Any> getProperties(kClass: KClass<T>): List<KProperty1<T, *>> {
        return propertyCache.getOrPut(kClass) {
            kClass.memberProperties.toList() as List<KProperty1<Any, *>>
        } as List<KProperty1<T, *>>
    }

    @Suppress("UNCHECKED_CAST")
    fun <T : Any> getConstructor(kClass: KClass<T>): KFunction<T> {
        return constructorCache.getOrPut(kClass) {
            kClass.primaryConstructor
                ?: throw IllegalStateException("No primary constructor for ${kClass.simpleName}")
        } as KFunction<T>
    }
}
```

---

## 📊 10. สรุปตาราง Kotlin Reflection API

| API | คำอธิบาย | ตัวอย่าง |
|-----|----------|---------|
| `KClass<T>` | Class metadata | `String::class` |
| `KFunction<R>` | Function metadata | `::println` |
| `KProperty<T, V>` | Property metadata | `User::name` |
| `KMutableProperty<T, V>` | Mutable property | `User::age` |
| `.call(vararg args)` | เรียก function/constructor | `func.call(obj, arg)` |
| `.callBy(params)` | เรียกพร้อม named params | `ctor.callBy(map)` |
| `.getter.call(obj)` | อ่านค่า property | `prop.getter.call(user)` |
| `.setter?.call(obj, v)` | เขียนค่า property | `prop.setter?.call(user, "new")` |
| `.findAnnotation<A>()` | ค้นหา annotation | `prop.findAnnotation<Valid>()` |
| `.memberProperties` | Properties ทั้งหมดของ class | `clazz.memberProperties` |
| `.isData` | ตรวจสอบว่าเป็น data class | `clazz.isData` |

---

## 💡 Best Practices

1. **Cache reflection results** — reflection มี overhead สูง อย่า reflect ซ้ำๆ
2. **ใช้ `@Retention(RUNTIME)`** บน annotations ที่ต้องอ่านณ runtime
3. **ระวัง type safety** — reflection bypass type checking ของ compiler
4. **เพิ่ม `kotlin-reflect` dependency** — ไม่ได้ include โดย default
5. **พิจารณา KSP แทน** — ถ้าสามารถทำ compile-time ได้ จะเร็วกว่า reflection

---

## 📝 11. Reflection Use Cases ใน Spring Boot

| Use Case | ตัวอย่าง | ระดับ Performance Impact |
|---------|---------|--------------------------|
| Generic mapper | Entity → DTO | กลาง (cache ได้) |
| Dynamic validation | @Validate annotations | ต่ำ (ถ้า cache) |
| Plugin loading | Load class by name | ต่ำ (ทำครั้งเดียว) |
| Serialization | JSON ↔ Object | สูง (ใช้ library) |
| Dependency injection | Spring IoC | ต่ำ (startup only) |
| Test frameworks | JUnit reflection | ไม่สำคัญใน tests |

### สรุป Reflection APIs ที่ใช้บ่อย

```kotlin
// ตัวอย่าง cheat sheet
val kClass = MyClass::class               // KClass reference
val instance = kClass.createInstance()    // no-arg constructor
val prop = kClass.memberProperties.first() // first property
val func = kClass.memberFunctions.first() // first function
val annotations = kClass.annotations      // class annotations
val isData = kClass.isData                // data class check
val superTypes = kClass.supertypes        // parent classes/interfaces
val typeParams = kClass.typeParameters    // generic type params
val visibility = prop.visibility          // PUBLIC, PROTECTED, etc.
val isNullable = prop.returnType.isMarkedNullable
```

---

*Part 68/100+ | Kotlin & Spring Boot Complete Course*
