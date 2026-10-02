# Part 04: Control Flow - if, when, loops
## Control Flow: Branching & Iteration

---

## 🎯 เป้าหมายของ Part นี้

- ใช้ `if/else` เป็น expression
- ใช้ `when` แทน switch-case
- วนซ้ำด้วย `for`, `while`, `do-while`
- ใช้ `break`, `continue`, `return`
- ใช้ labeled loops
- เข้าใจ scope ของตัวแปร

---

## 🔀 1. if Expression

### if พื้นฐาน

```kotlin
val score = 85

// แบบ statement
if (score >= 60) {
    println("ผ่าน")
} else {
    println("ไม่ผ่าน")
}

// Single line
if (score >= 60) println("ผ่าน")
else println("ไม่ผ่าน")
```

### if เป็น Expression (คืนค่าได้!)

```kotlin
val score = 85

// if ที่คืนค่า
val result = if (score >= 60) "ผ่าน" else "ไม่ผ่าน"
println(result)  // ผ่าน

// แทน ternary operator (?: ใน Java)
val max = if (10 > 20) 10 else 20   // max = 20

// if-else if-else
val grade = when {
    score >= 90 -> "A"
    score >= 80 -> "B"
    score >= 70 -> "C"
    score >= 60 -> "D"
    else        -> "F"
}
// เป็น when จริงๆ (ดูหัวข้อถัดไป)

// if expression กับ block
val message = if (score >= 90) {
    val bonus = "ยอดเยี่ยม!"
    "ได้เกรด A $bonus"  // ค่าสุดท้ายของ block
} else if (score >= 80) {
    "ได้เกรด B"
} else {
    "ต้องพยายามมากขึ้น"
}
println(message)
```

### if กับ Nullable

```kotlin
val name: String? = "Alice"

// แบบธรรมดา
if (name != null) {
    println("Hello, ${name.uppercase()}")  // Smart cast!
}

// ใน expression
val greeting = if (name != null) "Hello, $name!" else "Hello, stranger!"

// ดีกว่าด้วย let
name?.let { println("Hello, ${it.uppercase()}!") }
```

---

## 🎯 2. when Expression

### when แทน switch

```kotlin
val day = 3

// แบบ switch ใน Java
when (day) {
    1 -> println("จันทร์")
    2 -> println("อังคาร")
    3 -> println("พุธ")
    4 -> println("พฤหัส")
    5 -> println("ศุกร์")
    6 -> println("เสาร์")
    7 -> println("อาทิตย์")
    else -> println("วันที่ไม่ถูกต้อง")
}
// พุธ
```

### when เป็น Expression

```kotlin
val dayName = when (day) {
    1 -> "จันทร์"
    2 -> "อังคาร"
    3 -> "พุธ"
    4 -> "พฤหัส"
    5 -> "ศุกร์"
    6, 7 -> "วันหยุด"  // หลายค่า
    else -> "ไม่ถูกต้อง"
}
println(dayName)  // พุธ
```

### when กับ ranges

```kotlin
val score = 75

val grade = when (score) {
    in 90..100 -> "A"
    in 80..89  -> "B"
    in 70..79  -> "C"
    in 60..69  -> "D"
    in 0..59   -> "F"
    else       -> "ไม่ถูกต้อง"
}
println(grade)  // C
```

### when กับ type checking

```kotlin
fun describe(obj: Any): String = when (obj) {
    is Int    -> "Int: $obj"
    is Double -> "Double: ${"%.2f".format(obj)}"
    is String -> "String(${obj.length}): \"$obj\""
    is Boolean-> "Boolean: $obj"
    is List<*> -> "List[${obj.size}]: $obj"
    null      -> "null value"
    else      -> "Unknown: ${obj::class.simpleName}"
}

println(describe(42))
println(describe(3.14))
println(describe("Hello"))
println(describe(true))
println(describe(listOf(1, 2, 3)))
println(describe(null))
```

**ผลลัพธ์:**
```
Int: 42
Double: 3.14
String(5): "Hello"
Boolean: true
List[3]: [1, 2, 3]
null value
```

### when โดยไม่มี argument

```kotlin
val x = 5
val y = 10

val result = when {
    x > y   -> "$x มากกว่า $y"
    x < y   -> "$x น้อยกว่า $y"
    x == y  -> "เท่ากัน"
    else    -> "ไม่รู้"
}
println(result)  // 5 น้อยกว่า 10

// ตัวอย่างที่ซับซ้อนขึ้น
val temperature = 35.0

when {
    temperature < 0   -> println("หนาวมาก! ❄️")
    temperature < 15  -> println("เย็น 🌬️")
    temperature < 25  -> println("สบาย ☀️")
    temperature < 35  -> println("อบอุ่น 🌤️")
    temperature < 40  -> println("ร้อน 🔥")
    else              -> println("ร้อนจัด! 🌋")
}
// ร้อน 🔥
```

### when กับ sealed class

```kotlin
sealed class Shape {
    data class Circle(val radius: Double) : Shape()
    data class Rectangle(val width: Double, val height: Double) : Shape()
    data class Triangle(val base: Double, val height: Double) : Shape()
}

fun area(shape: Shape): Double = when (shape) {
    is Shape.Circle    -> Math.PI * shape.radius * shape.radius
    is Shape.Rectangle -> shape.width * shape.height
    is Shape.Triangle  -> 0.5 * shape.base * shape.height
    // ไม่ต้องมี else เพราะ sealed class ครบทุก case แล้ว
}

fun main() {
    val shapes = listOf(
        Shape.Circle(5.0),
        Shape.Rectangle(4.0, 6.0),
        Shape.Triangle(3.0, 4.0)
    )
    
    for (shape in shapes) {
        println("$shape → area = ${"%.2f".format(area(shape))}")
    }
}
```

---

## 🔄 3. for Loop

### for ทั่วไป

```kotlin
// วนซ้ำใน range
for (i in 1..5) {
    print("$i ")  // 1 2 3 4 5
}
println()

// วนซ้ำแบบลดลง
for (i in 10 downTo 1) {
    print("$i ")  // 10 9 8 7 6 5 4 3 2 1
}
println()

// วนซ้ำแบบข้าม
for (i in 0..20 step 5) {
    print("$i ")  // 0 5 10 15 20
}
println()

// ไม่รวม end
for (i in 0 until 5) {
    print("$i ")  // 0 1 2 3 4
}
println()
```

### for กับ Collection

```kotlin
val fruits = listOf("apple", "banana", "cherry", "durian")

// วนทีละตัว
for (fruit in fruits) {
    println(fruit)
}

// วนพร้อม index
for ((index, fruit) in fruits.withIndex()) {
    println("$index: $fruit")
}
// 0: apple
// 1: banana
// 2: cherry
// 3: durian

// วนแบบ forEach (functional)
fruits.forEach { fruit ->
    println(fruit.uppercase())
}

// forEach กับ index
fruits.forEachIndexed { index, fruit ->
    println("$index. $fruit")
}
```

### for กับ Map

```kotlin
val scores = mapOf(
    "Alice" to 95,
    "Bob" to 87,
    "Charlie" to 72
)

// วน Map
for ((name, score) in scores) {
    println("$name: $score")
}

// วนแค่ keys
for (name in scores.keys) {
    println(name)
}

// วนแค่ values
for (score in scores.values) {
    print("$score ")
}
println()
```

### for กับ String

```kotlin
val text = "Hello, Kotlin!"

// วนทีละตัวอักษร
for (char in text) {
    if (char.isLetter()) print(char)
}
println()  // HelloKotlin

// ใช้ indices
for (i in text.indices) {
    if (text[i].isUpperCase()) print("[$i:${text[i]}] ")
}
println()  // [0:H] [7:K]
```

---

## 🔁 4. while Loop

```kotlin
// while loop พื้นฐาน
var i = 1
while (i <= 5) {
    print("$i ")
    i++
}
println()  // 1 2 3 4 5

// while กับ input
fun getUserInput(): String {
    while (true) {
        print("ใส่ตัวเลข (0-100): ")
        val input = readLine() ?: continue
        val num = input.toIntOrNull()
        if (num != null && num in 0..100) {
            return input
        }
        println("ข้อมูลไม่ถูกต้อง กรุณาลองใหม่")
    }
}

// while กับเงื่อนไขซับซ้อน
var balance = 1000
var months = 0
val interestRate = 0.05

while (balance < 2000) {
    balance = (balance * (1 + interestRate)).toInt()
    months++
}
println("ใช้เวลา $months เดือน เงินเพิ่มเป็น $balance บาท")
```

### do-while Loop

```kotlin
// do-while ทำงานก่อน แล้วค่อยตรวจเงื่อนไข
var count = 0
do {
    println("Count: $count")
    count++
} while (count < 3)
// Count: 0
// Count: 1
// Count: 2

// ตัวอย่าง: เมนูโปรแกรม
fun showMenu() {
    do {
        println("\n=== เมนู ===")
        println("1. เพิ่มข้อมูล")
        println("2. ลบข้อมูล")
        println("3. แสดงข้อมูล")
        println("0. ออก")
        print("เลือก: ")
        
        val choice = readLine()?.toIntOrNull() ?: -1
        
        when (choice) {
            1 -> println("เพิ่มข้อมูล...")
            2 -> println("ลบข้อมูล...")
            3 -> println("แสดงข้อมูล...")
            0 -> println("ออกจากโปรแกรม")
            else -> println("ตัวเลือกไม่ถูกต้อง")
        }
        
        if (choice == 0) break
    } while (true)
}
```

---

## 🛑 5. break, continue, return

### break - หยุด loop

```kotlin
// break ออกจาก loop
for (i in 1..10) {
    if (i == 5) break
    print("$i ")  // 1 2 3 4
}
println()

// break ใน while
var x = 0
while (true) {
    if (x >= 3) break
    println("x = $x")
    x++
}
```

### continue - ข้าม iteration

```kotlin
// continue ข้าม iteration นี้
for (i in 1..10) {
    if (i % 2 == 0) continue  // ข้ามเลขคู่
    print("$i ")  // 1 3 5 7 9
}
println()

// กรองข้อมูล
val numbers = listOf(1, -2, 3, -4, 5, -6, 7)
for (num in numbers) {
    if (num < 0) continue
    print("$num ")  // 1 3 5 7
}
println()
```

### return - คืนค่าจาก function

```kotlin
fun findFirst(list: List<Int>, target: Int): Int {
    for ((index, value) in list.withIndex()) {
        if (value == target) return index  // คืนค่าทันที
    }
    return -1  // ไม่พบ
}

println(findFirst(listOf(3, 1, 4, 1, 5, 9), 4))  // 2
println(findFirst(listOf(3, 1, 4, 1, 5, 9), 7))  // -1
```

---

## 🏷️ 6. Labeled Loops

### break กับ label

```kotlin
// ออกจาก outer loop ด้วย label
outer@ for (i in 1..3) {
    for (j in 1..3) {
        if (i == 2 && j == 2) break@outer
        println("$i,$j")
    }
}
// 1,1
// 1,2
// 1,3
// 2,1
```

### continue กับ label

```kotlin
// ข้าม outer loop ด้วย label
outer@ for (i in 1..3) {
    for (j in 1..3) {
        if (j == 2) continue@outer  // ข้ามไป outer loop ถัดไป
        println("$i,$j")
    }
}
// 1,1
// 2,1
// 3,1
```

### return จาก lambda

```kotlin
fun processNumbers(numbers: List<Int>) {
    numbers.forEach label@{ num ->
        if (num < 0) return@label  // return จาก lambda เท่านั้น (continue)
        println(num)
    }
    println("Done!")
}

processNumbers(listOf(1, -2, 3, -4, 5))
// 1
// 3
// 5
// Done!

// เปรียบเทียบ: return ออกจาก function
fun processNumbers2(numbers: List<Int>) {
    numbers.forEach { num ->
        if (num < 0) return  // return ออกจาก function เลย!
        println(num)
    }
    println("Done!")  // จะไม่ถึงบรรทัดนี้ถ้ามี negative
}
```

---

## 🎮 7. ตัวอย่างโปรแกรมจริง

### โปรแกรมตรวจสอบจำนวนเฉพาะ

```kotlin
fun isPrime(n: Int): Boolean {
    if (n < 2) return false
    if (n == 2) return true
    if (n % 2 == 0) return false
    
    for (i in 3..Math.sqrt(n.toDouble()).toInt() step 2) {
        if (n % i == 0) return false
    }
    return true
}

fun main() {
    print("จำนวนเฉพาะตั้งแต่ 1 ถึง 100: ")
    for (i in 1..100) {
        if (isPrime(i)) print("$i ")
    }
    println()
    
    // นับจำนวนเฉพาะ
    val primeCount = (1..1000).count { isPrime(it) }
    println("จำนวนเฉพาะ 1-1000: $primeCount ตัว")
}
```

**ผลลัพธ์:**
```
จำนวนเฉพาะตั้งแต่ 1 ถึง 100: 2 3 5 7 11 13 17 19 23 29 31 37 41 43 47 53 59 61 67 71 73 79 83 89 97 
จำนวนเฉพาะ 1-1000: 168 ตัว
```

### FizzBuzz

```kotlin
fun fizzBuzz(n: Int): String = when {
    n % 15 == 0 -> "FizzBuzz"
    n % 3 == 0  -> "Fizz"
    n % 5 == 0  -> "Buzz"
    else        -> "$n"
}

fun main() {
    for (i in 1..30) {
        print("${fizzBuzz(i)} ")
    }
    println()
}
```

**ผลลัพธ์:**
```
1 2 Fizz 4 Buzz Fizz 7 8 Fizz Buzz 11 Fizz 13 14 FizzBuzz 16 17 Fizz 19 Buzz Fizz 22 23 Fizz Buzz 26 Fizz 28 29 FizzBuzz 
```

### Multiplication Table (ตารางสูตรคูณ)

```kotlin
fun multiplicationTable(n: Int) {
    println("=== ตารางสูตรคูณ แม่ $n ===")
    for (i in 1..12) {
        println("$n × $i = ${n * i}")
    }
}

fun main() {
    for (table in 1..9) {
        multiplicationTable(table)
        println()
    }
}
```

### Pattern Printing

```kotlin
fun main() {
    val n = 5
    
    // สามเหลี่ยม
    println("=== สามเหลี่ยม ===")
    for (i in 1..n) {
        println("*".repeat(i))
    }
    
    // สามเหลี่ยมกลับ
    println("\n=== สามเหลี่ยมกลับ ===")
    for (i in n downTo 1) {
        println("*".repeat(i))
    }
    
    // สามเหลี่ยมกลาง
    println("\n=== สามเหลี่ยมกลาง ===")
    for (i in 1..n) {
        print(" ".repeat(n - i))
        println("*".repeat(2 * i - 1))
    }
    
    // สี่เหลี่ยม
    println("\n=== สี่เหลี่ยม ===")
    for (i in 1..n) {
        for (j in 1..n) {
            print(if (i == 1 || i == n || j == 1 || j == n) "* " else "  ")
        }
        println()
    }
    
    // Checkerboard
    println("\n=== Checkerboard ===")
    for (i in 1..n) {
        for (j in 1..n) {
            print(if ((i + j) % 2 == 0) "# " else ". ")
        }
        println()
    }
}
```

**ผลลัพธ์:**
```
=== สามเหลี่ยม ===
*
**
***
****
*****

=== สามเหลี่ยมกลาง ===
    *
   ***
  *****
 *******
*********

=== สี่เหลี่ยม ===
* * * * * 
*       * 
*       * 
*       * 
* * * * * 

=== Checkerboard ===
# . # . # 
. # . # . 
# . # . # 
. # . # . 
# . # . # 
```

### Collatz Conjecture

```kotlin
fun collatz(n: Long): List<Long> {
    val sequence = mutableListOf(n)
    var current = n
    
    while (current != 1L) {
        current = if (current % 2 == 0L) current / 2 else 3 * current + 1
        sequence.add(current)
    }
    
    return sequence
}

fun main() {
    for (start in listOf(6L, 27L, 100L)) {
        val seq = collatz(start)
        println("Collatz($start): ${seq.size} steps")
        if (seq.size <= 20) println("  $seq")
    }
    
    // หาตัวเลขที่มี steps มากที่สุดใน 1-1000
    val longest = (1L..1000L).maxByOrNull { collatz(it).size }
    println("\nตัวเลขที่มี steps มากที่สุดใน 1-1000: $longest (${collatz(longest!!).size} steps)")
}
```

---

## 🏋️ 8. แบบฝึกหัด

### ข้อ 1: Number pyramid
สร้างพีระมิดตัวเลข

```kotlin
fun main() {
    val rows = 5
    for (i in 1..rows) {
        // Spaces
        print(" ".repeat(rows - i))
        // Numbers ascending
        for (j in 1..i) print(j)
        // Numbers descending
        for (j in i - 1 downTo 1) print(j)
        println()
    }
}
// Output:
//     1
//    121
//   12321
//  1234321
// 123454321
```

### ข้อ 2: Fibonacci sequence
```kotlin
fun fibonacci(n: Int): List<Long> {
    val seq = mutableListOf(0L, 1L)
    for (i in 2 until n) {
        seq.add(seq[i-1] + seq[i-2])
    }
    return seq.take(n)
}

fun main() {
    println(fibonacci(15))
    // [0, 1, 1, 2, 3, 5, 8, 13, 21, 34, 55, 89, 144, 233, 377]
    
    // Fibonacci ที่ไม่เกิน 1000
    var a = 0L
    var b = 1L
    print("Fibonacci ≤ 1000: ")
    while (a <= 1000) {
        print("$a ")
        val temp = a + b
        a = b
        b = temp
    }
    println()
}
```

### ข้อ 3: Find max in grid
```kotlin
fun findMax2D(grid: Array<IntArray>): Triple<Int, Int, Int> {
    var maxVal = Int.MIN_VALUE
    var maxRow = 0
    var maxCol = 0
    
    for (i in grid.indices) {
        for (j in grid[i].indices) {
            if (grid[i][j] > maxVal) {
                maxVal = grid[i][j]
                maxRow = i
                maxCol = j
            }
        }
    }
    
    return Triple(maxVal, maxRow, maxCol)
}

fun main() {
    val grid = arrayOf(
        intArrayOf(3, 7, 2),
        intArrayOf(9, 1, 5),
        intArrayOf(4, 8, 6)
    )
    
    val (max, row, col) = findMax2D(grid)
    println("ค่ามากที่สุด: $max ที่ตำแหน่ง [$row][$col]")
    // ค่ามากที่สุด: 9 ที่ตำแหน่ง [1][0]
}
```

---

## 📝 สรุป Part 04

| โครงสร้าง | การใช้งาน |
|----------|---------|
| `if/else` | ตรรกะ, คืนค่าได้ |
| `when` | แทน switch, type check |
| `for..in` | วนซ้ำ range, collection |
| `while` | วนซ้ำตามเงื่อนไข |
| `do-while` | วนก่อน, ตรวจทีหลัง |
| `break` | หยุด loop |
| `continue` | ข้าม iteration |
| `label@` | ควบคุม nested loops |

---

## ➡️ ถัดไป: Part 05 - Functions

ใน Part ถัดไปเราจะเรียนรู้:
- Named functions และ single-expression functions
- Default parameters
- Named arguments
- Varargs
- Local functions
- Tail recursion

---
*Part 04/100+ | Kotlin & Spring Boot Complete Course*
