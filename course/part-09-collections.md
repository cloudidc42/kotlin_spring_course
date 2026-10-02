# Part 09: Collections
## List, Set, Map และ Collection Operations ใน Kotlin

---

## 🎯 เป้าหมายของ Part นี้

- เข้าใจความแตกต่างระหว่าง Immutable และ Mutable Collections
- ใช้ List, Set, Map ได้อย่างคล่องแคล่ว
- ใช้ collection operations: filter, map, flatMap, groupBy, partition, zip, associate, reduce, fold
- เรียงลำดับข้อมูลหลากหลายรูปแบบ
- เข้าใจ Sequence กับ lazy evaluation
- สร้างระบบ Product Inventory ด้วย Collections

---

## 📦 1. List - รายการที่มีลำดับ

`List` คือ collection ที่เก็บข้อมูลเป็นลำดับ อนุญาตให้มีค่าซ้ำได้

### 1.1 Immutable List

```kotlin
fun main() {
    // สร้าง List แบบต่างๆ
    val fruits = listOf("Apple", "Banana", "Cherry", "Apple")
    val numbers = listOf(1, 2, 3, 4, 5)
    val mixed = listOf(1, "hello", 3.14, true)  // List<Any>
    val empty = emptyList<String>()
    val single = listOf("only")
    
    // การเข้าถึงสมาชิก
    println(fruits[0])          // Apple
    println(fruits.get(1))      // Banana
    println(fruits.first())     // Apple
    println(fruits.last())      // Apple
    println(fruits.getOrNull(10))  // null (ไม่ throw exception)
    println(fruits.getOrElse(10) { "default" })  // default
    
    // Properties พื้นฐาน
    println(fruits.size)        // 4
    println(fruits.isEmpty())   // false
    println(fruits.isNotEmpty()) // true
    println(fruits.count())     // 4
    
    // การตรวจสอบ
    println("Apple" in fruits)       // true
    println(fruits.contains("Mango")) // false
    println(fruits.containsAll(listOf("Apple", "Banana"))) // true
    
    // Index operations
    println(fruits.indexOf("Apple"))     // 0
    println(fruits.lastIndexOf("Apple")) // 3
    
    // Slice
    println(fruits.subList(1, 3))   // [Banana, Cherry]
    println(numbers.take(3))         // [1, 2, 3]
    println(numbers.drop(2))         // [3, 4, 5]
    println(numbers.takeLast(2))     // [4, 5]
    println(numbers.dropLast(1))     // [1, 2, 3, 4]
}
```

### 1.2 MutableList - List ที่แก้ไขได้

```kotlin
fun main() {
    // สร้าง MutableList
    val colors = mutableListOf("Red", "Green", "Blue")
    val scores: MutableList<Int> = mutableListOf()
    val fromArray = ArrayList<String>()
    
    // เพิ่มสมาชิก
    colors.add("Yellow")              // เพิ่มท้าย
    colors.add(0, "White")            // เพิ่มที่ index 0
    colors.addAll(listOf("Purple", "Orange"))  // เพิ่มหลายตัว
    println(colors)  // [White, Red, Green, Blue, Yellow, Purple, Orange]
    
    // แก้ไขสมาชิก
    colors[0] = "Black"               // เปลี่ยน index 0
    colors.set(1, "Pink")             // เปลี่ยน index 1
    println(colors)  // [Black, Pink, Green, Blue, Yellow, Purple, Orange]
    
    // ลบสมาชิก
    colors.remove("Green")            // ลบตามค่า (ลบตัวแรกที่เจอ)
    colors.removeAt(0)                // ลบตาม index
    colors.removeAll(listOf("Purple", "Orange"))  // ลบหลายตัว
    println(colors)  // [Pink, Blue, Yellow]
    
    // ล้างทั้งหมด
    val temp = mutableListOf(1, 2, 3)
    temp.clear()
    println(temp)    // []
    
    // Iteration
    for (color in colors) {
        println(color)
    }
    
    colors.forEachIndexed { index, color ->
        println("$index: $color")
    }
    
    // Convert
    val immutable: List<String> = colors.toList()
    val sorted: List<String> = colors.sorted()
    
    // Scores example
    scores.addAll(listOf(85, 90, 78, 92, 88))
    println("Average: ${scores.average()}")   // 86.6
    println("Max: ${scores.max()}")            // 92
    println("Min: ${scores.min()}")            // 78
    println("Sum: ${scores.sum()}")            // 433
}
```

### 1.3 List Builder Pattern

```kotlin
fun main() {
    // buildList - สร้าง List ด้วย builder
    val oddNumbers = buildList {
        for (i in 1..10) {
            if (i % 2 != 0) add(i)
        }
    }
    println(oddNumbers)  // [1, 3, 5, 7, 9]
    
    // List จาก range
    val rangeList = (1..5).toList()
    println(rangeList)  // [1, 2, 3, 4, 5]
    
    // List ที่สร้างด้วย lambda
    val squares = List(5) { it * it }
    println(squares)  // [0, 1, 4, 9, 16]
    
    // Repeat elements
    val repeated = List(3) { "hello" }
    println(repeated)  // [hello, hello, hello]
}
```

---

## 🔷 2. Set - รายการที่ไม่ซ้ำกัน

`Set` คือ collection ที่ไม่อนุญาตให้มีค่าซ้ำ ไม่รับประกันลำดับ (ยกเว้น LinkedHashSet)

### 2.1 Immutable Set

```kotlin
fun main() {
    // สร้าง Set
    val uniqueNumbers = setOf(1, 2, 3, 2, 1, 4)  // ค่าซ้ำถูกตัดออก
    println(uniqueNumbers)  // [1, 2, 3, 4]
    
    val letters = setOf('a', 'b', 'c', 'd')
    val emptySet = emptySet<String>()
    
    // การตรวจสอบ
    println(2 in uniqueNumbers)         // true
    println(uniqueNumbers.contains(5))  // false
    println(uniqueNumbers.size)         // 4
    
    // Set operations (ทฤษฎีเซต)
    val setA = setOf(1, 2, 3, 4, 5)
    val setB = setOf(3, 4, 5, 6, 7)
    
    // Union (สหภาพ)
    val union = setA union setB
    println(union)  // [1, 2, 3, 4, 5, 6, 7]
    
    // Intersection (อินเตอร์เซกชัน)
    val intersection = setA intersect setB
    println(intersection)  // [3, 4, 5]
    
    // Difference (ผลต่าง)
    val difference = setA subtract setB
    println(difference)  // [1, 2]
    
    // ใช้สำหรับหาค่าซ้ำ
    val list1 = listOf(1, 2, 3, 4, 5)
    val list2 = listOf(3, 4, 5, 6, 7)
    val common = list1.toSet() intersect list2.toSet()
    println(common)  // [3, 4, 5]
}
```

### 2.2 MutableSet

```kotlin
fun main() {
    val tags = mutableSetOf("kotlin", "android", "java")
    
    // เพิ่มสมาชิก
    val added = tags.add("spring")       // true - เพิ่มสำเร็จ
    val duplicate = tags.add("kotlin")   // false - มีอยู่แล้ว
    println(tags)    // [kotlin, android, java, spring]
    println(added)   // true
    println(duplicate) // false
    
    tags.addAll(setOf("gradle", "maven"))
    
    // ลบสมาชิก
    tags.remove("java")
    tags.removeAll(setOf("gradle", "maven"))
    println(tags)    // [kotlin, android, spring]
    
    // LinkedHashSet - รักษาลำดับการใส่
    val ordered = linkedSetOf("banana", "apple", "cherry")
    println(ordered)  // [banana, apple, cherry]
    
    // TreeSet (ผ่าน sortedSetOf) - เรียงลำดับ
    val sorted = sortedSetOf("banana", "apple", "cherry")
    println(sorted)  // [apple, banana, cherry]
    
    // ตัวอย่าง: หา unique visitors
    val visitors = mutableSetOf<String>()
    val pageVisits = listOf("Alice", "Bob", "Alice", "Charlie", "Bob", "Alice")
    pageVisits.forEach { visitors.add(it) }
    println("Unique visitors: ${visitors.size}")  // 3
    println(visitors)  // [Alice, Bob, Charlie]
}
```

---

## 🗺️ 3. Map - คู่ Key-Value

`Map` คือ collection ที่เก็บข้อมูลเป็นคู่ key-value โดย key ต้องไม่ซ้ำ

### 3.1 Immutable Map

```kotlin
fun main() {
    // สร้าง Map
    val capitals = mapOf(
        "Thailand" to "Bangkok",
        "Japan" to "Tokyo",
        "France" to "Paris",
        "Germany" to "Berlin"
    )
    
    val scores = mapOf("Alice" to 95, "Bob" to 87, "Charlie" to 92)
    
    // การเข้าถึง
    println(capitals["Thailand"])           // Bangkok
    println(capitals.get("Japan"))          // Tokyo
    println(capitals["Unknown"])            // null
    println(capitals.getOrDefault("Unknown", "N/A"))  // N/A
    println(capitals.getOrElse("Unknown") { "Not found" })  // Not found
    
    // Properties
    println(capitals.size)          // 4
    println(capitals.isEmpty())     // false
    println(capitals.keys)          // [Thailand, Japan, France, Germany]
    println(capitals.values)        // [Bangkok, Tokyo, Paris, Berlin]
    println(capitals.entries)       // map entries
    
    // การตรวจสอบ
    println("Thailand" in capitals)          // true
    println(capitals.containsKey("Spain"))   // false
    println(capitals.containsValue("Tokyo")) // true
    
    // Iteration
    for ((country, capital) in capitals) {
        println("$country: $capital")
    }
    
    capitals.forEach { (country, capital) ->
        println("Capital of $country is $capital")
    }
    
    // Map ที่ซ้อนกัน
    val nested = mapOf(
        "fruits" to mapOf("apple" to 100, "banana" to 50),
        "veggies" to mapOf("carrot" to 80, "potato" to 60)
    )
    println(nested["fruits"]?.get("apple"))  // 100
}
```

### 3.2 MutableMap

```kotlin
fun main() {
    val inventory = mutableMapOf<String, Int>()
    
    // เพิ่ม/แก้ไข
    inventory["Apple"] = 100
    inventory["Banana"] = 50
    inventory.put("Cherry", 75)
    inventory.putAll(mapOf("Date" to 30, "Elder" to 20))
    println(inventory)
    
    // แก้ไข
    inventory["Apple"] = 120     // เปลี่ยนค่า
    inventory["Banana"] = (inventory["Banana"] ?: 0) + 10  // เพิ่ม 10
    
    // getOrPut - ได้ค่าหรือสร้างใหม่
    val count = inventory.getOrPut("Mango") { 0 }
    println(count)  // 0
    
    // ลบ
    inventory.remove("Elder")
    inventory.remove("Date", 30)   // ลบถ้า value ตรง (conditional remove)
    
    // Update ด้วย merge
    inventory.merge("Apple", 50) { old, new -> old + new }  // Apple: 120 + 50 = 170
    
    // compute
    inventory.compute("Banana") { _, v -> (v ?: 0) * 2 }   // Banana * 2
    
    println(inventory)
    
    // LinkedHashMap - รักษาลำดับการใส่
    val ordered = linkedMapOf("z" to 26, "a" to 1, "m" to 13)
    println(ordered)  // {z=26, a=1, m=13}
    
    // sortedMapOf - เรียงตาม key
    val sorted = sortedMapOf("z" to 26, "a" to 1, "m" to 13)
    println(sorted)  // {a=1, m=13, z=26}
}
```

---

## ⚙️ 4. Collection Operations

### 4.1 filter - กรองข้อมูล

```kotlin
data class Product(
    val id: Int,
    val name: String,
    val category: String,
    val price: Double,
    val stock: Int,
    val isActive: Boolean
)

fun main() {
    val products = listOf(
        Product(1, "iPhone 15", "Electronics", 35000.0, 50, true),
        Product(2, "Samsung TV", "Electronics", 25000.0, 30, true),
        Product(3, "Running Shoes", "Sports", 2500.0, 100, true),
        Product(4, "Yoga Mat", "Sports", 800.0, 75, false),
        Product(5, "Coffee Maker", "Kitchen", 3500.0, 20, true),
        Product(6, "Blender", "Kitchen", 1500.0, 0, true)
    )
    
    // filter พื้นฐาน
    val expensive = products.filter { it.price > 10000 }
    println("Expensive: ${expensive.map { it.name }}")
    // [iPhone 15, Samsung TV]
    
    // filterNot - กรองแบบตรงข้าม
    val active = products.filterNot { !it.isActive }
    println("Active: ${active.size}")  // 5
    
    // filterIsInstance - กรองตาม type
    val items: List<Any> = listOf(1, "hello", 2.5, "world", 3, true)
    val strings = items.filterIsInstance<String>()
    println(strings)  // [hello, world]
    
    // filterNotNull - กรอง null ออก
    val nullableList = listOf(1, null, 3, null, 5)
    val nonNull = nullableList.filterNotNull()
    println(nonNull)  // [1, 3, 5]
    
    // filter ซับซ้อน
    val inStockActive = products.filter { it.stock > 0 && it.isActive }
    println("In stock & active: ${inStockActive.size}")  // 5
    
    // ตรวจสอบเงื่อนไข
    println(products.any { it.price > 30000 })   // true
    println(products.all { it.price > 0 })        // true
    println(products.none { it.stock < 0 })       // true
    println(products.count { it.category == "Electronics" })  // 2
}
```

### 4.2 map - แปลงข้อมูล

```kotlin
fun main() {
    val products = listOf(
        Product(1, "iPhone 15", "Electronics", 35000.0, 50, true),
        Product(2, "Samsung TV", "Electronics", 25000.0, 30, true),
        Product(3, "Running Shoes", "Sports", 2500.0, 100, true)
    )
    
    // map พื้นฐาน - แปลงทุก element
    val names = products.map { it.name }
    println(names)  // [iPhone 15, Samsung TV, Running Shoes]
    
    val pricesWithTax = products.map { it.price * 1.07 }
    println(pricesWithTax)  // [37450.0, 26750.0, 2675.0]
    
    // map เป็น data class ใหม่
    data class ProductSummary(val name: String, val price: String)
    
    val summaries = products.map { product ->
        ProductSummary(
            name = product.name,
            price = "฿${String.format("%.2f", product.price)}"
        )
    }
    summaries.forEach { println("${it.name}: ${it.price}") }
    
    // mapNotNull - แปลงและกรอง null ออก
    val numbers = listOf("1", "two", "3", "four", "5")
    val parsed = numbers.mapNotNull { it.toIntOrNull() }
    println(parsed)  // [1, 3, 5]
    
    // mapIndexed - แปลงพร้อม index
    val ranked = products.mapIndexed { index, product ->
        "#${index + 1} ${product.name}"
    }
    println(ranked)  // [#1 iPhone 15, #2 Samsung TV, #3 Running Shoes]
    
    // mapKeys, mapValues สำหรับ Map
    val prices = mapOf("apple" to 30, "banana" to 15, "cherry" to 50)
    val upperKeys = prices.mapKeys { it.key.uppercase() }
    println(upperKeys)  // {APPLE=30, BANANA=15, CHERRY=50}
    
    val discounted = prices.mapValues { (_, price) -> price * 0.9 }
    println(discounted)  // {apple=27.0, banana=13.5, cherry=45.0}
}
```

### 4.3 flatMap - แปลงและ flatten

```kotlin
fun main() {
    val departments = listOf(
        Pair("Engineering", listOf("Alice", "Bob", "Charlie")),
        Pair("Marketing", listOf("Dave", "Eve")),
        Pair("HR", listOf("Frank", "Grace", "Heidi"))
    )
    
    // flatMap - แปลงแต่ละ element เป็น list แล้วรวมเป็น list เดียว
    val allEmployees = departments.flatMap { (_, employees) -> employees }
    println(allEmployees)
    // [Alice, Bob, Charlie, Dave, Eve, Frank, Grace, Heidi]
    
    // เปรียบเทียบกับ map
    val mapped = departments.map { (_, employees) -> employees }
    println(mapped)
    // [[Alice, Bob, Charlie], [Dave, Eve], [Frank, Grace, Heidi]]
    
    // ตัวอย่างจริง: รวม tags จากหลาย products
    data class Article(val title: String, val tags: List<String>)
    
    val articles = listOf(
        Article("Kotlin Guide", listOf("kotlin", "programming", "jvm")),
        Article("Spring Boot", listOf("spring", "java", "backend")),
        Article("Android Dev", listOf("android", "kotlin", "mobile"))
    )
    
    val allTags = articles.flatMap { it.tags }
    println(allTags)
    // [kotlin, programming, jvm, spring, java, backend, android, kotlin, mobile]
    
    val uniqueTags = allTags.toSet()
    println(uniqueTags)
    // [kotlin, programming, jvm, spring, java, backend, android, mobile]
    
    // flatten - รวม List<List<T>> เป็น List<T>
    val nestedLists = listOf(listOf(1, 2, 3), listOf(4, 5), listOf(6, 7, 8))
    val flat = nestedLists.flatten()
    println(flat)  // [1, 2, 3, 4, 5, 6, 7, 8]
}
```

### 4.4 groupBy - จัดกลุ่ม

```kotlin
fun main() {
    val products = listOf(
        Product(1, "iPhone 15", "Electronics", 35000.0, 50, true),
        Product(2, "Samsung TV", "Electronics", 25000.0, 30, true),
        Product(3, "Running Shoes", "Sports", 2500.0, 100, true),
        Product(4, "Yoga Mat", "Sports", 800.0, 75, false),
        Product(5, "Coffee Maker", "Kitchen", 3500.0, 20, true),
        Product(6, "Blender", "Kitchen", 1500.0, 0, true)
    )
    
    // groupBy พื้นฐาน
    val byCategory = products.groupBy { it.category }
    byCategory.forEach { (category, items) ->
        println("$category: ${items.map { it.name }}")
    }
    // Electronics: [iPhone 15, Samsung TV]
    // Sports: [Running Shoes, Yoga Mat]
    // Kitchen: [Coffee Maker, Blender]
    
    // groupBy แล้วแปลงค่า
    val categoryNames = products.groupBy(
        keySelector = { it.category },
        valueTransform = { it.name }
    )
    println(categoryNames)
    
    // นับจำนวนต่อกลุ่ม
    val countByCategory = products.groupBy { it.category }.mapValues { it.value.size }
    println(countByCategory)  // {Electronics=2, Sports=2, Kitchen=2}
    
    // หาราคาเฉลี่ยต่อ category
    val avgPriceByCategory = products
        .groupBy { it.category }
        .mapValues { (_, items) -> items.map { it.price }.average() }
    println(avgPriceByCategory)
    
    // groupingBy + aggregate
    val totalStockByCategory = products
        .groupingBy { it.category }
        .fold(0) { acc, product -> acc + product.stock }
    println(totalStockByCategory)
    // {Electronics=80, Sports=175, Kitchen=20}
}
```

### 4.5 partition - แบ่งเป็น 2 กลุ่ม

```kotlin
fun main() {
    val numbers = listOf(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)
    
    // partition - แยกเป็น Pair<List, List>
    val (evens, odds) = numbers.partition { it % 2 == 0 }
    println("Evens: $evens")  // [2, 4, 6, 8, 10]
    println("Odds: $odds")    // [1, 3, 5, 7, 9]
    
    // ตัวอย่างจริง
    val products = listOf(
        Product(1, "iPhone 15", "Electronics", 35000.0, 50, true),
        Product(2, "Samsung TV", "Electronics", 25000.0, 30, true),
        Product(3, "Running Shoes", "Sports", 2500.0, 100, true),
        Product(4, "Yoga Mat", "Sports", 800.0, 75, false),
        Product(5, "Coffee Maker", "Kitchen", 3500.0, 20, true),
        Product(6, "Blender", "Kitchen", 1500.0, 0, true)
    )
    
    val (active, inactive) = products.partition { it.isActive }
    println("Active: ${active.map { it.name }}")
    println("Inactive: ${inactive.map { it.name }}")
    
    val (inStock, outOfStock) = products.partition { it.stock > 0 }
    println("In stock: ${inStock.size}")
    println("Out of stock: ${outOfStock.map { it.name }}")
}
```

### 4.6 zip - รวม 2 Lists

```kotlin
fun main() {
    val names = listOf("Alice", "Bob", "Charlie")
    val scores = listOf(95, 87, 92)
    val grades = listOf("A", "B+", "A")
    
    // zip พื้นฐาน - สร้าง List<Pair>
    val nameScore = names zip scores
    println(nameScore)
    // [(Alice, 95), (Bob, 87), (Charlie, 92)]
    
    // zip พร้อม transform
    val result = names.zip(scores) { name, score ->
        "$name: $score"
    }
    println(result)
    // [Alice: 95, Bob: 87, Charlie: 92]
    
    // unzip - แยก List<Pair> กลับเป็น Pair<List, List>
    val pairs = listOf(Pair("Alice", 95), Pair("Bob", 87))
    val (nameList, scoreList) = pairs.unzip()
    println(nameList)   // [Alice, Bob]
    println(scoreList)  // [95, 87]
    
    // zipWithNext - zip กับ element ถัดไป
    val temperatures = listOf(20, 22, 25, 23, 21, 24)
    val changes = temperatures.zipWithNext { current, next -> next - current }
    println(changes)  // [2, 3, -2, -2, 3]
    
    // zip หยุดที่ list ที่สั้นกว่า
    val short = listOf(1, 2, 3)
    val long = listOf(10, 20, 30, 40, 50)
    println(short.zip(long))  // [(1, 10), (2, 20), (3, 30)]
}
```

### 4.7 associate - สร้าง Map จาก Collection

```kotlin
fun main() {
    val products = listOf(
        Product(1, "iPhone 15", "Electronics", 35000.0, 50, true),
        Product(2, "Samsung TV", "Electronics", 25000.0, 30, true),
        Product(3, "Running Shoes", "Sports", 2500.0, 100, true)
    )
    
    // associate - สร้าง Map จาก lambda ที่คืน Pair
    val idToProduct = products.associate { it.id to it }
    println(idToProduct[1]?.name)  // iPhone 15
    
    // associateBy - ใช้ property เป็น key
    val byId = products.associateBy { it.id }
    println(byId[2]?.name)  // Samsung TV
    
    // associateBy with value transform
    val idToName = products.associateBy(
        keySelector = { it.id },
        valueTransform = { it.name }
    )
    println(idToName)  // {1=iPhone 15, 2=Samsung TV, 3=Running Shoes}
    
    // associateWith - ใช้ element เป็น key, lambda เป็น value
    val nameToPrice = products.associateWith { it.price }
    println(nameToPrice[products[0]])  // 35000.0
    
    // ตัวอย่างจริง: สร้าง lookup table
    val words = listOf("apple", "banana", "cherry", "date")
    val wordLengths = words.associateWith { it.length }
    println(wordLengths)  // {apple=5, banana=6, cherry=6, date=4}
    
    val firstLetterToWords = words.groupBy { it.first() }
    println(firstLetterToWords)
    // {a=[apple], b=[banana], c=[cherry], d=[date]}
}
```

### 4.8 reduce และ fold - คำนวณสะสม

```kotlin
fun main() {
    val numbers = listOf(1, 2, 3, 4, 5)
    
    // reduce - รวมค่าทั้งหมด (ไม่มี initial value)
    val sum = numbers.reduce { acc, num -> acc + num }
    println("Sum: $sum")  // 15
    
    val product = numbers.reduce { acc, num -> acc * num }
    println("Product: $product")  // 120
    
    val largest = numbers.reduce { acc, num -> maxOf(acc, num) }
    println("Max: $largest")  // 5
    
    // fold - เหมือน reduce แต่มี initial value
    val sumWithFold = numbers.fold(0) { acc, num -> acc + num }
    println("Sum (fold): $sumWithFold")  // 15
    
    val sumFrom100 = numbers.fold(100) { acc, num -> acc + num }
    println("Sum from 100: $sumFrom100")  // 115
    
    // fold กับ type ที่ต่างกัน
    val words = listOf("Hello", "World", "Kotlin")
    val sentence = words.fold("") { acc, word ->
        if (acc.isEmpty()) word else "$acc $word"
    }
    println(sentence)  // Hello World Kotlin
    
    // สร้าง Map ด้วย fold
    val wordCount = words.fold(mutableMapOf<Int, MutableList<String>>()) { map, word ->
        val len = word.length
        map.getOrPut(len) { mutableListOf() }.add(word)
        map
    }
    println(wordCount)  // {5=[Hello, World], 6=[Kotlin]}
    
    // reduceRight, foldRight - ทำจากขวาไปซ้าย
    val rightFold = words.foldRight("") { word, acc ->
        if (acc.isEmpty()) word else "$word $acc"
    }
    println(rightFold)  // Hello World Kotlin
    
    // runningFold - เก็บ intermediate results
    val running = numbers.runningFold(0) { acc, num -> acc + num }
    println(running)  // [0, 1, 3, 6, 10, 15]
    
    // scan - เหมือน runningFold
    val cumulative = numbers.scan(0) { acc, num -> acc + num }
    println(cumulative)  // [0, 1, 3, 6, 10, 15]
}
```

---

## 🔢 5. Sorting - การเรียงลำดับ

```kotlin
data class Student(val name: String, val grade: Int, val score: Double)

fun main() {
    // เรียงลำดับพื้นฐาน
    val numbers = listOf(3, 1, 4, 1, 5, 9, 2, 6)
    println(numbers.sorted())          // [1, 1, 2, 3, 4, 5, 6, 9]
    println(numbers.sortedDescending()) // [9, 6, 5, 4, 3, 2, 1, 1]
    
    // เรียง String
    val fruits = listOf("Banana", "apple", "Cherry", "date")
    println(fruits.sorted())           // [Banana, Cherry, apple, date] (case-sensitive)
    println(fruits.sortedWith(compareBy(String::lowercase)))  // [apple, Banana, Cherry, date]
    
    // sortedBy - เรียงตาม property
    val students = listOf(
        Student("Charlie", 3, 88.5),
        Student("Alice", 1, 95.0),
        Student("Bob", 2, 87.0),
        Student("David", 1, 92.0)
    )
    
    val byName = students.sortedBy { it.name }
    byName.forEach { println("${it.name}: ${it.score}") }
    
    val byScoreDesc = students.sortedByDescending { it.score }
    byScoreDesc.forEach { println("${it.name}: ${it.score}") }
    
    // sortedWith + compareBy - เรียงหลาย criteria
    val byGradeThenScore = students.sortedWith(
        compareBy<Student> { it.grade }.thenByDescending { it.score }
    )
    byGradeThenScore.forEach { println("Grade ${it.grade}: ${it.name} (${it.score})") }
    
    // MutableList sort (in-place)
    val mutableNumbers = mutableListOf(3, 1, 4, 1, 5)
    mutableNumbers.sort()
    println(mutableNumbers)  // [1, 1, 3, 4, 5]
    
    mutableNumbers.sortDescending()
    println(mutableNumbers)  // [5, 4, 3, 1, 1]
    
    mutableNumbers.sortWith(compareByDescending { it })
    
    // Comparator
    val comparator = Comparator<Student> { a, b ->
        when {
            a.grade != b.grade -> a.grade - b.grade
            else -> b.score.compareTo(a.score)
        }
    }
    val custom = students.sortedWith(comparator)
    custom.forEach { println("Grade ${it.grade}: ${it.name} (${it.score})") }
}
```

---

## ⚡ 6. Sequence vs Collection - Lazy Evaluation

### 6.1 ความแตกต่าง

```kotlin
fun main() {
    // Collection - Eager evaluation (คำนวณทันที ทีละขั้นตอน)
    val collection = (1..1_000_000)
        .toList()
        .filter { it % 2 == 0 }   // สร้าง List ใหม่ขนาด 500,000
        .map { it * it }           // สร้าง List ใหม่ขนาด 500,000
        .take(5)                   // สร้าง List ใหม่ขนาด 5
    println(collection)
    
    // Sequence - Lazy evaluation (คำนวณเมื่อต้องการ element)
    val sequence = (1..1_000_000)
        .asSequence()
        .filter { it % 2 == 0 }   // ยังไม่คำนวณ!
        .map { it * it }           // ยังไม่คำนวณ!
        .take(5)                   // ยังไม่คำนวณ!
        .toList()                  // คำนวณตอนนี้เท่านั้น! แค่ 5 elements
    println(sequence)
    
    // Sequence ประหยัดกว่ามากเมื่อ:
    // 1. มีข้อมูลจำนวนมาก
    // 2. ใช้ operations หลายขั้นตอน
    // 3. ต้องการแค่บางส่วนของผลลัพธ์
}
```

### 6.2 การสร้าง Sequence

```kotlin
fun main() {
    // จาก Collection
    val seqFromList = listOf(1, 2, 3, 4, 5).asSequence()
    
    // จาก elements โดยตรง
    val seqOf = sequenceOf(1, 2, 3, 4, 5)
    
    // generateSequence - infinite sequence
    val naturals = generateSequence(1) { it + 1 }
    val first10 = naturals.take(10).toList()
    println(first10)  // [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
    
    // Fibonacci sequence
    val fibonacci = generateSequence(Pair(0, 1)) { (a, b) -> Pair(b, a + b) }
        .map { it.first }
    val first15Fib = fibonacci.take(15).toList()
    println(first15Fib)  // [0, 1, 1, 2, 3, 5, 8, 13, 21, 34, 55, 89, 144, 233, 377]
    
    // sequence {} builder
    val primes = sequence {
        var nums = generateSequence(2) { it + 1 }
        while (true) {
            val prime = nums.first()
            yield(prime)
            nums = nums.drop(1).filter { it % prime != 0 }
        }
    }
    println(primes.take(10).toList())  // [2, 3, 5, 7, 11, 13, 17, 19, 23, 29]
    
    // Sequence operations
    val result = generateSequence(1) { it + 1 }
        .filter { it % 3 == 0 }
        .map { it * it }
        .takeWhile { it < 1000 }
        .toList()
    println(result)  // [9, 36, 81, 144, 225, 324, 441, 576, 729, 900]
}
```

### 6.3 Performance Comparison

```kotlin
import kotlin.system.measureTimeMillis

fun main() {
    val data = (1..1_000_000).toList()
    
    // Collection approach
    val timeCollection = measureTimeMillis {
        data.filter { it % 2 == 0 }
            .map { it.toString() }
            .filter { it.length > 3 }
            .first()
    }
    
    // Sequence approach
    val timeSequence = measureTimeMillis {
        data.asSequence()
            .filter { it % 2 == 0 }
            .map { it.toString() }
            .filter { it.length > 3 }
            .first()
    }
    
    println("Collection: ${timeCollection}ms")
    println("Sequence: ${timeSequence}ms")
    // Sequence จะเร็วกว่ามาก!
    
    // กฎการเลือกใช้
    // ใช้ Collection เมื่อ:
    // - ข้อมูลน้อย (< 1000 elements)
    // - ต้องการ access element หลายครั้ง
    // - ต้องการ random access
    
    // ใช้ Sequence เมื่อ:
    // - ข้อมูลมาก
    // - มี operations หลายขั้นตอน
    // - ต้องการแค่บางส่วน (first, take, etc.)
    // - infinite sequences
}
```

---

## 🏪 7. ตัวอย่างจริง: Product Inventory System

```kotlin
data class Product(
    val id: Int,
    val name: String,
    val category: String,
    val price: Double,
    val stock: Int,
    val isActive: Boolean,
    val tags: List<String>
)

data class Order(
    val orderId: Int,
    val customerId: Int,
    val productId: Int,
    val quantity: Int,
    val totalPrice: Double
)

class InventorySystem(private val products: List<Product>) {
    
    // 1. ค้นหาสินค้า
    fun searchByName(query: String): List<Product> =
        products.filter { it.name.contains(query, ignoreCase = true) }
    
    fun searchByCategory(category: String): List<Product> =
        products.filter { it.category.equals(category, ignoreCase = true) }
    
    fun searchByPriceRange(min: Double, max: Double): List<Product> =
        products.filter { it.price in min..max }
    
    fun searchByTags(tags: List<String>): List<Product> =
        products.filter { product ->
            tags.any { tag -> tag in product.tags }
        }
    
    // 2. สถิติสินค้า
    fun getCategoryStats(): Map<String, CategoryStats> {
        return products
            .groupBy { it.category }
            .mapValues { (_, items) ->
                CategoryStats(
                    count = items.size,
                    totalStock = items.sumOf { it.stock },
                    avgPrice = items.map { it.price }.average(),
                    minPrice = items.minOf { it.price },
                    maxPrice = items.maxOf { it.price }
                )
            }
    }
    
    // 3. สินค้าที่ต้องเติม stock
    fun getLowStockProducts(threshold: Int = 10): List<Product> =
        products.filter { it.isActive && it.stock <= threshold }
                .sortedBy { it.stock }
    
    // 4. สินค้ายอดนิยม (จาก orders)
    fun getTopSellingProducts(orders: List<Order>, limit: Int = 5): List<Pair<Product, Int>> {
        val salesByProduct = orders
            .groupBy { it.productId }
            .mapValues { (_, orders) -> orders.sumOf { it.quantity } }
        
        return products
            .mapNotNull { product ->
                val sold = salesByProduct[product.id] ?: return@mapNotNull null
                Pair(product, sold)
            }
            .sortedByDescending { it.second }
            .take(limit)
    }
    
    // 5. Revenue analysis
    fun getRevenueByCategory(orders: List<Order>): Map<String, Double> {
        val productById = products.associateBy { it.id }
        
        return orders
            .mapNotNull { order ->
                val product = productById[order.productId] ?: return@mapNotNull null
                Pair(product.category, order.totalPrice)
            }
            .groupBy { it.first }
            .mapValues { (_, pairs) -> pairs.sumOf { it.second } }
    }
    
    // 6. สร้าง catalog report
    fun generateCatalogReport(): String {
        val sb = StringBuilder()
        sb.appendLine("=== Product Catalog Report ===")
        sb.appendLine("Total Products: ${products.size}")
        sb.appendLine("Active Products: ${products.count { it.isActive }}")
        sb.appendLine()
        
        val byCategory = products.groupBy { it.category }
        byCategory.forEach { (category, items) ->
            sb.appendLine("--- $category ---")
            items.sortedBy { it.name }.forEach { product ->
                val status = if (product.isActive) "✓" else "✗"
                sb.appendLine("  $status ${product.name} - ฿${product.price} (stock: ${product.stock})")
            }
        }
        
        return sb.toString()
    }
}

data class CategoryStats(
    val count: Int,
    val totalStock: Int,
    val avgPrice: Double,
    val minPrice: Double,
    val maxPrice: Double
)

fun main() {
    val products = listOf(
        Product(1, "iPhone 15 Pro", "Electronics", 45000.0, 25, true,
            listOf("phone", "apple", "smartphone")),
        Product(2, "Samsung Galaxy S24", "Electronics", 38000.0, 30, true,
            listOf("phone", "samsung", "android")),
        Product(3, "MacBook Pro M3", "Electronics", 85000.0, 15, true,
            listOf("laptop", "apple", "computer")),
        Product(4, "Nike Running Shoes", "Sports", 3500.0, 80, true,
            listOf("shoes", "nike", "running")),
        Product(5, "Adidas Yoga Mat", "Sports", 900.0, 50, false,
            listOf("yoga", "adidas", "fitness")),
        Product(6, "Nespresso Machine", "Kitchen", 7500.0, 12, true,
            listOf("coffee", "kitchen", "appliance")),
        Product(7, "Vitamix Blender", "Kitchen", 15000.0, 8, true,
            listOf("blender", "kitchen", "appliance")),
        Product(8, "Standing Desk", "Furniture", 12000.0, 5, true,
            listOf("desk", "office", "furniture"))
    )
    
    val orders = listOf(
        Order(1, 101, 1, 2, 90000.0),
        Order(2, 102, 4, 3, 10500.0),
        Order(3, 103, 1, 1, 45000.0),
        Order(4, 101, 2, 2, 76000.0),
        Order(5, 104, 4, 5, 17500.0),
        Order(6, 102, 6, 1, 7500.0),
        Order(7, 105, 3, 1, 85000.0),
        Order(8, 103, 4, 2, 7000.0)
    )
    
    val inventory = InventorySystem(products)
    
    // ค้นหา
    println("=== Search Results ===")
    println("Electronics: ${inventory.searchByCategory("Electronics").map { it.name }}")
    println("Apple products: ${inventory.searchByTags(listOf("apple")).map { it.name }}")
    println("Under 10000: ${inventory.searchByPriceRange(0.0, 10000.0).map { it.name }}")
    
    // สถิติ
    println("\n=== Category Stats ===")
    inventory.getCategoryStats().forEach { (cat, stats) ->
        println("$cat: ${stats.count} items, avg ฿${String.format("%.0f", stats.avgPrice)}")
    }
    
    // Low stock
    println("\n=== Low Stock Alert (<=15) ===")
    inventory.getLowStockProducts(15).forEach {
        println("${it.name}: ${it.stock} remaining")
    }
    
    // Top sellers
    println("\n=== Top 3 Sellers ===")
    inventory.getTopSellingProducts(orders, 3).forEach { (product, sold) ->
        println("${product.name}: $sold units sold")
    }
    
    // Revenue by category
    println("\n=== Revenue by Category ===")
    inventory.getRevenueByCategory(orders).entries
        .sortedByDescending { it.value }
        .forEach { (cat, revenue) ->
            println("$cat: ฿${String.format("%.2f", revenue)}")
        }
    
    // Catalog report
    println("\n${inventory.generateCatalogReport()}")
}
```

---

## 📚 8. Utility Functions อื่นๆ ที่มีประโยชน์

```kotlin
fun main() {
    val numbers = listOf(3, 1, 4, 1, 5, 9, 2, 6, 5, 3, 5)
    
    // distinct - ตัดค่าซ้ำ
    println(numbers.distinct())  // [3, 1, 4, 5, 9, 2, 6]
    
    // distinctBy - ตัดซ้ำตาม criteria
    data class Person(val name: String, val age: Int)
    val people = listOf(
        Person("Alice", 30), Person("Bob", 25),
        Person("Charlie", 30), Person("David", 25)
    )
    val uniqueAges = people.distinctBy { it.age }
    println(uniqueAges)  // [Person(name=Alice, age=30), Person(name=Bob, age=25)]
    
    // chunked - แบ่งเป็น chunks
    println(numbers.chunked(3))  // [[3, 1, 4], [1, 5, 9], [2, 6, 5], [3, 5]]
    val chunkedSum = numbers.chunked(3) { it.sum() }
    println(chunkedSum)  // [8, 15, 13, 8]
    
    // windowed - sliding window
    println(numbers.windowed(3))  // [[3, 1, 4], [1, 4, 1], [4, 1, 5], ...]
    val movingAvg = numbers.windowed(3) { it.average() }
    println(movingAvg.map { String.format("%.2f", it) })
    
    // flatten related
    val nested = listOf(listOf(1, 2), listOf(3, 4), listOf(5))
    println(nested.flatten())  // [1, 2, 3, 4, 5]
    
    // max/min operations
    println(numbers.max())           // 9
    println(numbers.min())           // 1
    println(numbers.maxOrNull())     // 9 (null ถ้า empty)
    println(numbers.minOrNull())     // 1
    
    val people2 = listOf(Person("Alice", 30), Person("Bob", 25))
    println(people2.maxByOrNull { it.age })  // Person(name=Alice, age=30)
    println(people2.minByOrNull { it.age })  // Person(name=Bob, age=25)
    
    // sumOf - หาผลรวม
    println(numbers.sumOf { it.toLong() })  // 44
    
    // joinToString - แปลงเป็น String
    println(numbers.joinToString())                    // 3, 1, 4, 1, 5, 9, 2, 6, 5, 3, 5
    println(numbers.joinToString(separator = " | "))   // 3 | 1 | ...
    println(numbers.joinToString(
        prefix = "[",
        postfix = "]",
        separator = ", "
    ))  // [3, 1, 4, 1, 5, 9, 2, 6, 5, 3, 5]
    
    // reversed
    println(numbers.reversed())  // [5, 3, 5, 6, 2, 9, 5, 1, 4, 1, 3]
    
    // shuffled (random order)
    val shuffled = numbers.shuffled()
    println(shuffled)
    
    // indices
    println(numbers.indices)         // 0..10
    println(numbers.lastIndex)       // 10
    
    // withIndex
    numbers.withIndex().forEach { (index, value) ->
        print("[$index:$value] ")
    }
    println()
}
```

---

## 🏋️ แบบฝึกหัด

### ระดับ 1 (ง่าย)

1. สร้าง `List<String>` ของชื่อ 10 ประเทศ จากนั้น:
   - กรองเฉพาะประเทศที่ชื่อยาวกว่า 5 ตัวอักษร
   - แปลงเป็นตัวพิมพ์ใหญ่ทั้งหมด
   - เรียงตามตัวอักษร
   - แสดงผล 5 อันดับแรก

2. สร้าง `Map<String, Int>` ที่เก็บคะแนนนักเรียน 5 คน แล้ว:
   - หาค่าเฉลี่ย
   - หาคนที่ได้คะแนนสูงสุดและต่ำสุด
   - แยกเป็น 2 กลุ่ม: ผ่าน (>= 50) และไม่ผ่าน (< 50)

### ระดับ 2 (กลาง)

3. สร้าง data class `Employee(id, name, department, salary)` และ list 10+ คน จากนั้น:
   - หา department ที่มีพนักงานมากที่สุด
   - หาค่าเฉลี่ยเงินเดือนต่อ department
   - หา top 3 เงินเดือนสูงสุดในแต่ละ department
   - สร้าง report string ด้วย `fold`

4. เขียนฟังก์ชัน `wordFrequency(text: String): Map<String, Int>` ที่:
   - แยกคำจาก text
   - นับความถี่ของแต่ละคำ
   - คืน Map เรียงตามความถี่จากมากไปน้อย

### ระดับ 3 (ท้าทาย)

5. สร้าง Sequence ที่ generate prime numbers แบบ infinite แล้วใช้มันเพื่อ:
   - หา prime ลำดับที่ 100
   - หา prime ทั้งหมดที่น้อยกว่า 1000
   - หา twin primes (prime คู่ที่ต่างกัน 2) 10 คู่แรก

6. เขียน `groupAnagrams(words: List<String>): List<List<String>>` ที่จัดกลุ่ม anagrams ด้วย `groupBy`
   - "eat", "tea", "tan", "ate", "nat", "bat"  
   - ควรได้ [["eat", "tea", "ate"], ["tan", "nat"], ["bat"]]

---

## 📊 สรุป

| Collection | Ordered | Duplicates | Mutable | Use Case |
|-----------|---------|------------|---------|----------|
| `listOf` | ✓ | ✓ | ✗ | ข้อมูลทั่วไปที่มีลำดับ |
| `mutableListOf` | ✓ | ✓ | ✓ | ต้องการเพิ่ม/ลบ/แก้ไข |
| `setOf` | ✗ | ✗ | ✗ | ค่าไม่ซ้ำ ตรวจสอบ membership |
| `mutableSetOf` | ✗ | ✗ | ✓ | เพิ่ม/ลบค่าไม่ซ้ำ |
| `mapOf` | ✗ | key: ✗ | ✗ | key-value lookup |
| `mutableMapOf` | ✗ | key: ✗ | ✓ | dynamic key-value |
| `linkedMapOf` | insertion | key: ✗ | ✓ | รักษาลำดับการใส่ |
| `sortedMapOf` | key-sorted | key: ✗ | ✓ | เรียงตาม key |

| Operation | Description | Returns |
|-----------|-------------|---------|
| `filter` | กรองตามเงื่อนไข | `List<T>` |
| `map` | แปลงทุก element | `List<R>` |
| `flatMap` | แปลง + flatten | `List<R>` |
| `groupBy` | จัดกลุ่ม | `Map<K, List<T>>` |
| `partition` | แบ่ง 2 กลุ่ม | `Pair<List, List>` |
| `reduce` | รวมเป็นค่าเดียว | `T` |
| `fold` | รวมพร้อม initial value | `R` |
| `associate` | สร้าง Map | `Map<K, V>` |
| `zip` | รวม 2 lists | `List<Pair>` |
| `sorted` | เรียงลำดับ | `List<T>` |

---

## ➡️ ถัดไป: Part 10 - Lambda และ Higher-Order Functions

---
*Part 09/100+ | Kotlin & Spring Boot Complete Course*
