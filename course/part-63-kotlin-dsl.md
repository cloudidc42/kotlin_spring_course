# Part 63: Kotlin DSL - Type-safe Builders

## บทนำ

**DSL (Domain-Specific Language)** คือภาษาพิเศษที่ออกแบบมาสำหรับ domain เฉพาะ Kotlin รองรับการสร้าง **internal DSL** ได้อย่างสวยงามด้วยฟีเจอร์ต่างๆ เช่น lambda with receiver, extension functions, operator overloading และ infix functions

## ทำไมต้อง DSL?

### ปัญหาของ Builder Pattern ธรรมดา

```kotlin
// Traditional Builder - verbose and error-prone
val query = QueryBuilder()
    .select("id", "name", "email")
    .from("users")
    .where("active = true")
    .and("age > 18")
    .orderBy("name")
    .limit(10)
    .build()
```

### Kotlin DSL - Cleaner and Type-safe

```kotlin
// Kotlin DSL - readable and type-safe
val query = query {
    select("id", "name", "email")
    from("users")
    where {
        "active" eq true
        "age" gt 18
    }
    orderBy("name")
    limit(10)
}
```

## 1. Lambda with Receiver

พื้นฐานของ DSL คือ **lambda with receiver** (Function Type with Receiver):

```kotlin
// T.() -> Unit คือ lambda ที่มี T เป็น receiver
fun buildHtml(block: HtmlBuilder.() -> Unit): HtmlBuilder {
    val builder = HtmlBuilder()
    builder.block()
    return builder
}

class HtmlBuilder {
    private val content = StringBuilder()
    
    fun div(block: DivBuilder.() -> Unit) {
        val div = DivBuilder()
        div.block()
        content.append("<div>${div.build()}</div>")
    }
    
    fun p(text: String) {
        content.append("<p>$text</p>")
    }
    
    fun build(): String = content.toString()
}
```

## 2. @DslMarker - ป้องกัน DSL Scope Problems

```kotlin
@DslMarker
annotation class HtmlDsl

@HtmlDsl
class HtmlBuilder {
    // ...
}

@HtmlDsl
class DivBuilder {
    // ...
}

// @DslMarker ป้องกัน implicit access to outer receiver
html {
    div {
        div {
            // this คือ innermost DivBuilder เท่านั้น
            // ไม่สามารถเรียก html {} จากที่นี่โดยตรง
        }
    }
}
```

## 3. HTML DSL

```kotlin
// HtmlDsl.kt
package com.dsl.html

@DslMarker
annotation class HtmlDsl

@HtmlDsl
abstract class HtmlElement {
    protected val children = mutableListOf<HtmlElement>()
    protected val attributes = mutableMapOf<String, String>()

    fun render(indent: Int = 0): String {
        val sb = StringBuilder()
        val spaces = "  ".repeat(indent)
        val attrStr = if (attributes.isEmpty()) "" else " " + 
            attributes.entries.joinToString(" ") { (k, v) -> "$k=\"$v\"" }
        
        if (children.isEmpty()) {
            sb.append("$spaces<${tagName()}$attrStr/>")
        } else {
            sb.append("$spaces<${tagName()}$attrStr>\n")
            children.forEach { sb.append(it.render(indent + 1) + "\n") }
            sb.append("$spaces</${tagName()}>")
        }
        return sb.toString()
    }

    abstract fun tagName(): String
}

@HtmlDsl
class HTML : HtmlElement() {
    override fun tagName() = "html"

    fun head(block: HEAD.() -> Unit) {
        val head = HEAD()
        head.block()
        children.add(head)
    }

    fun body(block: BODY.() -> Unit) {
        val body = BODY()
        body.block()
        children.add(body)
    }
}

@HtmlDsl
class HEAD : HtmlElement() {
    override fun tagName() = "head"
    
    fun title(text: String) {
        children.add(TextElement("title", text))
    }
    
    fun meta(name: String, content: String) {
        val meta = MetaElement()
        meta.attributes["name"] = name
        meta.attributes["content"] = content
        children.add(meta)
    }
    
    fun link(rel: String, href: String) {
        val link = LinkElement()
        link.attributes["rel"] = rel
        link.attributes["href"] = href
        children.add(link)
    }
}

@HtmlDsl
class BODY : HtmlElement() {
    override fun tagName() = "body"

    fun div(vararg classes: String, block: DIV.() -> Unit) {
        val div = DIV()
        if (classes.isNotEmpty()) {
            div.attributes["class"] = classes.joinToString(" ")
        }
        div.block()
        children.add(div)
    }

    fun h1(text: String, vararg classes: String) {
        val h = HeadingElement("h1")
        if (classes.isNotEmpty()) h.attributes["class"] = classes.joinToString(" ")
        h.text = text
        children.add(h)
    }

    fun p(text: String) {
        children.add(TextElement("p", text))
    }

    fun a(href: String, text: String) {
        val a = LinkTextElement()
        a.attributes["href"] = href
        a.text = text
        children.add(a)
    }

    fun ul(block: UL.() -> Unit) {
        val ul = UL()
        ul.block()
        children.add(ul)
    }
}

@HtmlDsl
class DIV : HtmlElement() {
    override fun tagName() = "div"
    
    fun p(text: String) {
        children.add(TextElement("p", text))
    }
    
    fun span(text: String, vararg classes: String) {
        val span = TextElement("span", text)
        if (classes.isNotEmpty()) span.attributes["class"] = classes.joinToString(" ")
        children.add(span)
    }

    fun img(src: String, alt: String = "") {
        val img = ImgElement()
        img.attributes["src"] = src
        img.attributes["alt"] = alt
        children.add(img)
    }
}

@HtmlDsl
class UL : HtmlElement() {
    override fun tagName() = "ul"
    
    fun li(text: String) {
        children.add(TextElement("li", text))
    }
    
    fun li(block: LI.() -> Unit) {
        val li = LI()
        li.block()
        children.add(li)
    }
}

@HtmlDsl
class LI : HtmlElement() {
    override fun tagName() = "li"
    fun text(content: String) { children.add(RawTextElement(content)) }
}

// Entry point function
fun html(block: HTML.() -> Unit): HTML {
    val html = HTML()
    html.block()
    return html
}

// การใช้งาน
fun main() {
    val page = html {
        head {
            title("My Kotlin Page")
            meta("viewport", "width=device-width, initial-scale=1.0")
            link("stylesheet", "styles.css")
        }
        body {
            div("container", "main") {
                h1("Welcome to Kotlin DSL", "title")
                p("This page was generated using a Kotlin HTML DSL")
                
                div("products") {
                    p("Product List:")
                }
            }
            
            div("footer") {
                p("Footer content")
                a("https://kotlinlang.org", "Visit Kotlin")
            }
        }
    }
    
    println(page.render())
}
```

## 4. Query DSL

```kotlin
// QueryDsl.kt
package com.dsl.query

@DslMarker
annotation class QueryDsl

data class Column(val name: String, val alias: String? = null) {
    override fun toString() = if (alias != null) "$name AS $alias" else name
}

sealed class Condition {
    data class Equals(val column: String, val value: Any?) : Condition()
    data class NotEquals(val column: String, val value: Any?) : Condition()
    data class GreaterThan(val column: String, val value: Any) : Condition()
    data class LessThan(val column: String, val value: Any) : Condition()
    data class Like(val column: String, val pattern: String) : Condition()
    data class In(val column: String, val values: List<Any>) : Condition()
    data class IsNull(val column: String) : Condition()
    data class IsNotNull(val column: String) : Condition()
    data class And(val conditions: List<Condition>) : Condition()
    data class Or(val conditions: List<Condition>) : Condition()
    data class Not(val condition: Condition) : Condition()
}

enum class OrderDirection { ASC, DESC }
data class OrderByClause(val column: String, val direction: OrderDirection = OrderDirection.ASC)
data class JoinClause(val table: String, val type: JoinType, val condition: String)
enum class JoinType { INNER, LEFT, RIGHT, FULL }

@QueryDsl
class WhereBuilder {
    internal val conditions = mutableListOf<Condition>()

    infix fun String.eq(value: Any?): Condition {
        val cond = Condition.Equals(this, value)
        conditions.add(cond)
        return cond
    }

    infix fun String.neq(value: Any?): Condition {
        val cond = Condition.NotEquals(this, value)
        conditions.add(cond)
        return cond
    }

    infix fun String.gt(value: Any): Condition {
        val cond = Condition.GreaterThan(this, value)
        conditions.add(cond)
        return cond
    }

    infix fun String.lt(value: Any): Condition {
        val cond = Condition.LessThan(this, value)
        conditions.add(cond)
        return cond
    }

    infix fun String.like(pattern: String): Condition {
        val cond = Condition.Like(this, pattern)
        conditions.add(cond)
        return cond
    }

    infix fun String.`in`(values: List<Any>): Condition {
        val cond = Condition.In(this, values)
        conditions.add(cond)
        return cond
    }

    fun String.isNull(): Condition {
        val cond = Condition.IsNull(this)
        conditions.add(cond)
        return cond
    }

    fun String.isNotNull(): Condition {
        val cond = Condition.IsNotNull(this)
        conditions.add(cond)
        return cond
    }

    fun and(block: WhereBuilder.() -> Unit): Condition {
        val nested = WhereBuilder()
        nested.block()
        val cond = Condition.And(nested.conditions)
        conditions.add(cond)
        return cond
    }

    fun or(block: WhereBuilder.() -> Unit): Condition {
        val nested = WhereBuilder()
        nested.block()
        val cond = Condition.Or(nested.conditions)
        conditions.add(cond)
        return cond
    }
}

@QueryDsl
class SelectQueryBuilder {
    private val columns = mutableListOf<Column>()
    private var table: String = ""
    private val joins = mutableListOf<JoinClause>()
    private var whereConditions: Condition? = null
    private val orderByClauses = mutableListOf<OrderByClause>()
    private var groupByColumns = mutableListOf<String>()
    private var havingCondition: String? = null
    private var limitValue: Int? = null
    private var offsetValue: Int? = null

    fun select(vararg cols: String) {
        columns.addAll(cols.map { Column(it) })
    }

    fun selectAs(column: String, alias: String) {
        columns.add(Column(column, alias))
    }

    fun from(tableName: String) {
        this.table = tableName
    }

    fun join(table: String, condition: String) {
        joins.add(JoinClause(table, JoinType.INNER, condition))
    }

    fun leftJoin(table: String, condition: String) {
        joins.add(JoinClause(table, JoinType.LEFT, condition))
    }

    fun where(block: WhereBuilder.() -> Unit) {
        val builder = WhereBuilder()
        builder.block()
        whereConditions = if (builder.conditions.size == 1) {
            builder.conditions.first()
        } else {
            Condition.And(builder.conditions)
        }
    }

    fun orderBy(column: String, direction: OrderDirection = OrderDirection.ASC) {
        orderByClauses.add(OrderByClause(column, direction))
    }

    fun groupBy(vararg cols: String) {
        groupByColumns.addAll(cols)
    }

    fun having(condition: String) {
        havingCondition = condition
    }

    fun limit(value: Int) {
        limitValue = value
    }

    fun offset(value: Int) {
        offsetValue = value
    }

    fun build(): String {
        val sb = StringBuilder()
        
        val selectClause = if (columns.isEmpty()) "*" else columns.joinToString(", ")
        sb.append("SELECT $selectClause")
        sb.append(" FROM $table")
        
        joins.forEach { join ->
            sb.append(" ${join.type} JOIN ${join.table} ON ${join.condition}")
        }
        
        whereConditions?.let { sb.append(" WHERE ${renderCondition(it)}") }
        
        if (groupByColumns.isNotEmpty()) {
            sb.append(" GROUP BY ${groupByColumns.joinToString(", ")}")
        }
        
        havingCondition?.let { sb.append(" HAVING $it") }
        
        if (orderByClauses.isNotEmpty()) {
            sb.append(" ORDER BY " + orderByClauses.joinToString(", ") { 
                "${it.column} ${it.direction}" 
            })
        }
        
        limitValue?.let { sb.append(" LIMIT $it") }
        offsetValue?.let { sb.append(" OFFSET $it") }
        
        return sb.toString()
    }

    private fun renderCondition(condition: Condition): String = when (condition) {
        is Condition.Equals -> {
            val value = if (condition.value is String) "'${condition.value}'" else condition.value
            "${condition.column} = $value"
        }
        is Condition.NotEquals -> {
            val value = if (condition.value is String) "'${condition.value}'" else condition.value
            "${condition.column} != $value"
        }
        is Condition.GreaterThan -> "${condition.column} > ${condition.value}"
        is Condition.LessThan -> "${condition.column} < ${condition.value}"
        is Condition.Like -> "${condition.column} LIKE '${condition.pattern}'"
        is Condition.In -> {
            val values = condition.values.joinToString(", ") { 
                if (it is String) "'$it'" else it.toString() 
            }
            "${condition.column} IN ($values)"
        }
        is Condition.IsNull -> "${condition.column} IS NULL"
        is Condition.IsNotNull -> "${condition.column} IS NOT NULL"
        is Condition.And -> condition.conditions.joinToString(" AND ") { 
            "(${renderCondition(it)})" 
        }
        is Condition.Or -> condition.conditions.joinToString(" OR ") { 
            "(${renderCondition(it)})" 
        }
        is Condition.Not -> "NOT (${renderCondition(condition.condition)})"
    }
}

// Entry point
fun query(block: SelectQueryBuilder.() -> Unit): String {
    val builder = SelectQueryBuilder()
    builder.block()
    return builder.build()
}

// การใช้งาน Query DSL
fun examples() {
    val simpleQuery = query {
        select("id", "name", "email")
        from("users")
        where {
            "active" eq true
            "age" gt 18
        }
        orderBy("name")
        limit(10)
    }
    
    println(simpleQuery)
    // SELECT id, name, email FROM users WHERE (active = true) AND (age > 18) ORDER BY name ASC LIMIT 10

    val complexQuery = query {
        select("u.id", "u.name", "o.total")
        selectAs("COUNT(o.id)", "order_count")
        from("users u")
        leftJoin("orders o", "u.id = o.user_id")
        where {
            "u.active" eq true
            or {
                "u.role" eq "ADMIN"
                "u.role" eq "MODERATOR"
            }
        }
        groupBy("u.id", "u.name", "o.total")
        having("COUNT(o.id) > 0")
        orderBy("order_count", OrderDirection.DESC)
        limit(20)
        offset(40)
    }
    
    println(complexQuery)
}
```

## 5. Test DSL

```kotlin
// TestDsl.kt
package com.dsl.test

@DslMarker
annotation class TestDsl

data class HttpRequest(
    val method: String,
    val url: String,
    val headers: Map<String, String>,
    val body: String?,
    val queryParams: Map<String, String>
)

data class HttpResponse(
    val statusCode: Int,
    val headers: Map<String, String>,
    val body: String?
)

@TestDsl
class HttpRequestBuilder {
    var method: String = "GET"
    var url: String = ""
    private val headers = mutableMapOf<String, String>()
    var body: String? = null
    private val queryParams = mutableMapOf<String, String>()

    fun header(name: String, value: String) {
        headers[name] = value
    }

    fun authorization(token: String) {
        headers["Authorization"] = "Bearer $token"
    }

    fun contentType(type: String = "application/json") {
        headers["Content-Type"] = type
    }

    fun param(name: String, value: String) {
        queryParams[name] = value
    }

    fun build() = HttpRequest(method, url, headers, body, queryParams)
}

@TestDsl
class AssertionsBuilder(private val response: HttpResponse) {
    
    fun statusCode(expected: Int) {
        assert(response.statusCode == expected) {
            "Expected status code $expected but got ${response.statusCode}"
        }
    }

    fun ok() = statusCode(200)
    fun created() = statusCode(201)
    fun noContent() = statusCode(204)
    fun badRequest() = statusCode(400)
    fun unauthorized() = statusCode(401)
    fun forbidden() = statusCode(403)
    fun notFound() = statusCode(404)

    fun bodyContains(text: String) {
        assert(response.body?.contains(text) == true) {
            "Expected body to contain '$text' but got: ${response.body}"
        }
    }

    fun bodyJson(block: JsonAssertBuilder.() -> Unit) {
        val jsonBuilder = JsonAssertBuilder(response.body ?: "{}")
        jsonBuilder.block()
    }

    fun header(name: String, expectedValue: String) {
        assert(response.headers[name] == expectedValue) {
            "Expected header '$name' to be '$expectedValue' but got '${response.headers[name]}'"
        }
    }
}

@TestDsl
class JsonAssertBuilder(private val json: String) {
    fun field(path: String, expected: Any?) {
        // Parse JSON and check field
        println("Checking $path == $expected in JSON")
    }
    
    fun arraySize(path: String, expectedSize: Int) {
        println("Checking array size at $path == $expectedSize")
    }
}

@TestDsl
class ApiTestBuilder {
    private var requestBuilder = HttpRequestBuilder()
    private lateinit var response: HttpResponse

    fun given(block: HttpRequestBuilder.() -> Unit): ApiTestBuilder {
        requestBuilder.block()
        return this
    }

    fun `when`(block: HttpRequestBuilder.() -> Unit): ApiTestBuilder {
        requestBuilder.block()
        response = executeRequest(requestBuilder.build())
        return this
    }

    fun then(block: AssertionsBuilder.() -> Unit): ApiTestBuilder {
        val assertions = AssertionsBuilder(response)
        assertions.block()
        return this
    }

    private fun executeRequest(request: HttpRequest): HttpResponse {
        // Execute the actual HTTP request (using RestTemplate, WebClient, etc.)
        return HttpResponse(200, mapOf(), "{\"status\":\"ok\"}")
    }
}

fun apiTest(block: ApiTestBuilder.() -> Unit) {
    val builder = ApiTestBuilder()
    builder.block()
}

// การใช้งาน Test DSL
class UserApiTest {

    fun `should create user successfully`() {
        apiTest {
            given {
                method = "POST"
                url = "/api/users"
                contentType()
                body = """{"email":"test@test.com","password":"Password123","firstName":"John","lastName":"Doe"}"""
            }
            `when` {
                // Execute request
            }
            then {
                created()
                bodyJson {
                    field("id", null)  // id exists
                    field("email", "test@test.com")
                    field("firstName", "John")
                }
            }
        }
    }

    fun `should return 404 for unknown user`() {
        apiTest {
            given {
                method = "GET"
                url = "/api/users/999"
                authorization("valid-token")
            }
            `when` { }
            then {
                notFound()
                bodyContains("User not found")
            }
        }
    }
}
```

## 6. Configuration DSL (Spring-style)

```kotlin
// ConfigDsl.kt
package com.dsl.config

@DslMarker
annotation class ConfigDsl

data class DatabaseConfig(
    val host: String,
    val port: Int,
    val name: String,
    val username: String,
    val password: String,
    val maxPoolSize: Int,
    val connectionTimeout: Long
)

data class CacheConfig(
    val host: String,
    val port: Int,
    val ttl: Long,
    val maxSize: Int
)

data class AppConfig(
    val database: DatabaseConfig,
    val cache: CacheConfig?,
    val features: Map<String, Boolean>
)

@ConfigDsl
class DatabaseConfigBuilder {
    var host: String = "localhost"
    var port: Int = 5432
    var name: String = ""
    var username: String = ""
    var password: String = ""
    var maxPoolSize: Int = 10
    var connectionTimeout: Long = 30000L

    fun build() = DatabaseConfig(host, port, name, username, password, maxPoolSize, connectionTimeout)
}

@ConfigDsl
class CacheConfigBuilder {
    var host: String = "localhost"
    var port: Int = 6379
    var ttl: Long = 3600L
    var maxSize: Int = 1000

    fun build() = CacheConfig(host, port, ttl, maxSize)
}

@ConfigDsl
class AppConfigBuilder {
    private var databaseConfig: DatabaseConfig? = null
    private var cacheConfig: CacheConfig? = null
    private val features = mutableMapOf<String, Boolean>()

    fun database(block: DatabaseConfigBuilder.() -> Unit) {
        databaseConfig = DatabaseConfigBuilder().apply(block).build()
    }

    fun cache(block: CacheConfigBuilder.() -> Unit) {
        cacheConfig = CacheConfigBuilder().apply(block).build()
    }

    fun feature(name: String, enabled: Boolean = true) {
        features[name] = enabled
    }

    fun build() = AppConfig(
        database = databaseConfig ?: throw IllegalStateException("Database config is required"),
        cache = cacheConfig,
        features = features
    )
}

fun appConfig(block: AppConfigBuilder.() -> Unit): AppConfig {
    return AppConfigBuilder().apply(block).build()
}

// การใช้งาน Config DSL
val config = appConfig {
    database {
        host = "db.example.com"
        port = 5432
        name = "myapp"
        username = "admin"
        password = "secret"
        maxPoolSize = 20
        connectionTimeout = 5000L
    }
    
    cache {
        host = "redis.example.com"
        port = 6379
        ttl = 7200L
        maxSize = 5000
    }
    
    feature("dark-mode", enabled = true)
    feature("new-checkout", enabled = false)
    feature("ai-recommendations", enabled = true)
}
```

## สรุปเทคนิค DSL ใน Kotlin

| เทคนิค | ใช้เพื่อ | ตัวอย่าง |
|--------|---------|---------|
| Lambda with Receiver | Block ที่มี context | `html { body { } }` |
| @DslMarker | ป้องกัน scope confusion | `@HtmlDsl annotation class` |
| Infix Function | ทำให้อ่านเหมือน English | `"name" eq "John"` |
| Operator Overloading | Custom operators | `query + filter` |
| Extension Functions | เพิ่ม DSL methods | `String.isNull()` |
| Type-safe Builders | Compile-time validation | `select("id")` |
| Sealed Classes | Represent variants | `Condition.Equals` |

## ข้อดีของ Kotlin DSL

1. **Type Safety** - Compiler ตรวจสอบ syntax ให้
2. **IDE Support** - Autocomplete ใช้ได้
3. **Readability** - โค้ดอ่านเหมือน config file
4. **Refactoring** - Rename ได้ปลอดภัย
5. **No Learning Curve** - ยังเป็น Kotlin code

*Part 63/100+ | Kotlin & Spring Boot Complete Course*
