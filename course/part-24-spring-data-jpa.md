# Part 24: Spring Data JPA
## Database Access with Spring Data JPA & Kotlin

---

## 🎯 เป้าหมายของ Part นี้

- เข้าใจ JPA (Java Persistence API) และ Hibernate
- สร้าง Entity classes
- ใช้ Spring Data JPA Repository
- สร้าง relationships (One-to-One, One-to-Many, Many-to-Many)
- Custom queries (JPQL, Native SQL)
- Pagination และ Sorting
- ตัวอย่าง: Blog System

---

## 🏗️ 1. Entity Basics

```kotlin
// Entity พื้นฐาน
import jakarta.persistence.*
import java.time.LocalDateTime

@Entity
@Table(name = "products")
data class Product(
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long = 0,
    
    @Column(nullable = false, length = 200)
    val name: String,
    
    @Column(columnDefinition = "TEXT")
    val description: String = "",
    
    @Column(nullable = false, precision = 10, scale = 2)
    val price: Double,
    
    @Column(name = "stock_quantity")
    var stockQuantity: Int = 0,
    
    @Column(unique = true)
    val sku: String,
    
    @Column(name = "is_active")
    var isActive: Boolean = true,
    
    @Column(name = "created_at", updatable = false)
    val createdAt: LocalDateTime = LocalDateTime.now(),
    
    @Column(name = "updated_at")
    var updatedAt: LocalDateTime = LocalDateTime.now()
)
```

### JPA Annotations ที่ต้องรู้

```kotlin
@Entity                     // เป็น JPA entity
@Table(name = "table_name") // ชื่อ table
@Id                         // Primary key
@GeneratedValue             // Auto-generate ID
  strategy = IDENTITY       // DB auto-increment
  strategy = SEQUENCE       // Sequence object
  strategy = UUID            // Random UUID

@Column(
    name = "col_name",      // ชื่อ column
    nullable = false,       // NOT NULL
    unique = true,          // UNIQUE constraint
    length = 100,           // VARCHAR length
    precision = 10,         // Decimal total digits
    scale = 2               // Decimal fractional digits
    updatable = false,      // ห้ามอัปเดต
    insertable = false      // ห้าม insert
)

@Transient                  // ไม่บันทึกลง DB
@Enumerated(EnumType.STRING) // บันทึก enum เป็น String
@Lob                        // Large Object (TEXT, BLOB)
@CreationTimestamp          // Hibernate: ตั้งค่าตอน create
@UpdateTimestamp            // Hibernate: ตั้งค่าตอน update
```

---

## 📚 2. Repository Pattern

```kotlin
// JpaRepository มี methods ให้ใช้ทันที:
// save(entity), findById(id), findAll(), deleteById(id)
// existsById(id), count(), saveAll(list), deleteAll()

@Repository
interface ProductRepository : JpaRepository<Product, Long> {
    
    // Spring Data generates SQL from method names!
    
    // SELECT * FROM products WHERE name = ?
    fun findByName(name: String): List<Product>
    
    // SELECT * FROM products WHERE price BETWEEN ? AND ?
    fun findByPriceBetween(min: Double, max: Double): List<Product>
    
    // SELECT * FROM products WHERE price <= ? AND is_active = true
    fun findByPriceLessThanEqualAndIsActiveTrue(maxPrice: Double): List<Product>
    
    // SELECT * FROM products WHERE name LIKE %?%
    fun findByNameContainingIgnoreCase(keyword: String): List<Product>
    
    // SELECT * FROM products WHERE sku = ?
    fun findBySku(sku: String): Product?
    
    // SELECT EXISTS(SELECT 1 FROM products WHERE sku = ?)
    fun existsBySku(sku: String): Boolean
    
    // SELECT * FROM products WHERE is_active = true ORDER BY price
    fun findByIsActiveTrueOrderByPrice(): List<Product>
    
    // Custom JPQL query
    @Query("SELECT p FROM Product p WHERE p.price > :minPrice AND p.stockQuantity > 0")
    fun findAvailableAbovePrice(@Param("minPrice") minPrice: Double): List<Product>
    
    // Native SQL
    @Query(
        value = "SELECT * FROM products WHERE MATCH(name, description) AGAINST(:keyword)",
        nativeQuery = true
    )
    fun fullTextSearch(@Param("keyword") keyword: String): List<Product>
    
    // Modifying query
    @Modifying
    @Transactional
    @Query("UPDATE Product p SET p.stockQuantity = p.stockQuantity - :qty WHERE p.id = :id")
    fun decrementStock(@Param("id") id: Long, @Param("qty") qty: Int): Int
    
    // Pagination
    fun findByIsActiveTrue(pageable: Pageable): Page<Product>
}
```

### Method Naming Keywords

```
// Keywords:
// find... / get... / read... / query...  → SELECT
// count...                               → SELECT COUNT
// exists...                             → SELECT EXISTS
// delete... / remove...                 → DELETE

// Conditions:
// By            → WHERE
// And           → AND
// Or            → OR
// Not           → NOT
// Is/Equals     → =
// LessThan      → <
// LessThanEqual → <=
// GreaterThan   → >
// Between       → BETWEEN
// Like          → LIKE
// StartingWith  → LIKE x%
// EndingWith    → LIKE %x
// Containing    → LIKE %x%
// In            → IN
// NotIn         → NOT IN
// True/False    → = true/false
// Null/NotNull  → IS NULL/IS NOT NULL
// After/Before  → > / <  (for dates)
// IgnoreCase    → UPPER(x) = UPPER(?)

// Sort/Limit:
// OrderBy...Asc/Desc
// First/Top (limit 1)
```

---

## 🔗 3. Relationships

### One-to-One

```kotlin
@Entity
@Table(name = "users")
data class User(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long = 0,
    val name: String,
    val email: String,
    
    @OneToOne(
        cascade = [CascadeType.ALL],
        fetch = FetchType.LAZY,  // ไม่โหลดจนกว่าจะเรียกใช้
        orphanRemoval = true
    )
    @JoinColumn(name = "profile_id")
    var profile: UserProfile? = null
)

@Entity
@Table(name = "user_profiles")
data class UserProfile(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long = 0,
    val bio: String = "",
    val avatarUrl: String = "",
    val website: String = "",
    
    @OneToOne(mappedBy = "profile")
    val user: User? = null
)
```

### One-to-Many / Many-to-One

```kotlin
@Entity
@Table(name = "categories")
data class Category(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long = 0,
    val name: String,
    
    @OneToMany(
        mappedBy = "category",
        cascade = [CascadeType.PERSIST, CascadeType.MERGE],
        fetch = FetchType.LAZY
    )
    val products: MutableList<Product> = mutableListOf()
)

@Entity
@Table(name = "products")
data class Product(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long = 0,
    val name: String,
    val price: Double,
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "category_id")
    val category: Category? = null
)
```

### Many-to-Many

```kotlin
@Entity
@Table(name = "posts")
data class Post(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long = 0,
    val title: String,
    val content: String,
    
    @ManyToMany(cascade = [CascadeType.PERSIST, CascadeType.MERGE])
    @JoinTable(
        name = "post_tags",
        joinColumns = [JoinColumn(name = "post_id")],
        inverseJoinColumns = [JoinColumn(name = "tag_id")]
    )
    val tags: MutableSet<Tag> = mutableSetOf()
)

@Entity
@Table(name = "tags")
data class Tag(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long = 0,
    
    @Column(unique = true)
    val name: String,
    
    @ManyToMany(mappedBy = "tags")
    val posts: MutableSet<Post> = mutableSetOf()
)
```

---

## 📑 4. Pagination และ Sorting

```kotlin
@RestController
@RequestMapping("/api/products")
class ProductController(private val productRepository: ProductRepository) {
    
    @GetMapping
    fun getProducts(
        @RequestParam(defaultValue = "0") page: Int,
        @RequestParam(defaultValue = "10") size: Int,
        @RequestParam(defaultValue = "id") sortBy: String,
        @RequestParam(defaultValue = "asc") direction: String
    ): Page<Product> {
        val sort = if (direction == "desc") 
            Sort.by(sortBy).descending() 
        else 
            Sort.by(sortBy).ascending()
        
        val pageable = PageRequest.of(page, size, sort)
        return productRepository.findAll(pageable)
    }
}
```

ผลลัพธ์ของ Page:
```json
{
    "content": [...],
    "pageable": {
        "pageNumber": 0,
        "pageSize": 10,
        "sort": {"sorted": true}
    },
    "totalElements": 100,
    "totalPages": 10,
    "last": false,
    "first": true
}
```

---

## 🏪 5. ตัวอย่างจริง: Blog System

```kotlin
// Entities
@Entity
@Table(name = "authors")
data class Author(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long = 0,
    val name: String,
    val email: String,
    
    @OneToMany(mappedBy = "author", cascade = [CascadeType.ALL])
    val posts: MutableList<BlogPost> = mutableListOf()
)

@Entity
@Table(name = "blog_posts")
data class BlogPost(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long = 0,
    val title: String,
    
    @Column(columnDefinition = "TEXT")
    val content: String,
    
    @Enumerated(EnumType.STRING)
    var status: PostStatus = PostStatus.DRAFT,
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "author_id")
    val author: Author,
    
    @ManyToMany(cascade = [CascadeType.PERSIST, CascadeType.MERGE])
    @JoinTable(name = "post_tag_mapping")
    val tags: MutableSet<PostTag> = mutableSetOf(),
    
    @OneToMany(mappedBy = "post", cascade = [CascadeType.ALL])
    val comments: MutableList<Comment> = mutableListOf(),
    
    val createdAt: LocalDateTime = LocalDateTime.now(),
    var publishedAt: LocalDateTime? = null
) {
    enum class PostStatus { DRAFT, PUBLISHED, ARCHIVED }
}

@Entity
data class PostTag(@Id @GeneratedValue val id: Long = 0, val name: String)

@Entity
data class Comment(
    @Id @GeneratedValue val id: Long = 0,
    val content: String,
    val authorName: String,
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "post_id")
    val post: BlogPost,
    
    val createdAt: LocalDateTime = LocalDateTime.now()
)

// Repository
@Repository
interface BlogPostRepository : JpaRepository<BlogPost, Long> {
    fun findByStatus(status: BlogPost.PostStatus, pageable: Pageable): Page<BlogPost>
    fun findByAuthorId(authorId: Long): List<BlogPost>
    fun findByTagsName(tagName: String): List<BlogPost>
    
    @Query("""
        SELECT p FROM BlogPost p 
        WHERE p.status = 'PUBLISHED' 
        AND (LOWER(p.title) LIKE LOWER(CONCAT('%', :q, '%'))
          OR LOWER(p.content) LIKE LOWER(CONCAT('%', :q, '%')))
        ORDER BY p.publishedAt DESC
    """)
    fun search(@Param("q") query: String, pageable: Pageable): Page<BlogPost>
}

// Service
@Service
@Transactional
class BlogService(
    private val postRepo: BlogPostRepository,
    private val authorRepo: JpaRepository<Author, Long>
) {
    fun createPost(authorId: Long, title: String, content: String, tags: List<String>): BlogPost {
        val author = authorRepo.findById(authorId).orElseThrow {
            NoSuchElementException("Author not found: $authorId")
        }
        
        val post = BlogPost(
            title = title,
            content = content,
            author = author,
            tags = tags.map { PostTag(name = it) }.toMutableSet()
        )
        
        return postRepo.save(post)
    }
    
    fun publish(postId: Long): BlogPost {
        val post = postRepo.findById(postId).orElseThrow {
            NoSuchElementException("Post not found: $postId")
        }
        
        require(post.status == BlogPost.PostStatus.DRAFT) {
            "Only DRAFT posts can be published"
        }
        
        return postRepo.save(post.copy(
            status = BlogPost.PostStatus.PUBLISHED,
            publishedAt = LocalDateTime.now()
        ))
    }
    
    @Transactional(readOnly = true)
    fun getPublishedPosts(page: Int = 0, size: Int = 10): Page<BlogPost> {
        return postRepo.findByStatus(
            BlogPost.PostStatus.PUBLISHED,
            PageRequest.of(page, size, Sort.by("publishedAt").descending())
        )
    }
}
```

---

## 🧪 6. Testing Repository

```kotlin
@DataJpaTest  // ใช้ H2 in-memory DB สำหรับ test
class ProductRepositoryTest {
    
    @Autowired
    private lateinit var productRepo: ProductRepository
    
    @BeforeEach
    fun setup() {
        productRepo.saveAll(listOf(
            Product(name = "Apple", price = 10.0, sku = "APL001", stockQuantity = 100),
            Product(name = "Banana", price = 5.0, sku = "BNN001", stockQuantity = 50),
            Product(name = "Apple Juice", price = 25.0, sku = "APJ001", stockQuantity = 0)
        ))
    }
    
    @Test
    fun `findByName should return products with exact name`() {
        val products = productRepo.findByName("Apple")
        assertThat(products).hasSize(1)
        assertThat(products[0].sku).isEqualTo("APL001")
    }
    
    @Test
    fun `findByNameContaining should return matching products`() {
        val products = productRepo.findByNameContainingIgnoreCase("apple")
        assertThat(products).hasSize(2)  // Apple + Apple Juice
    }
    
    @Test
    fun `findAvailableAbovePrice should exclude out of stock`() {
        val products = productRepo.findAvailableAbovePrice(5.0)
        assertThat(products).hasSize(1)  // Only Apple (Banana = 5.0 not > 5.0, Juice = out of stock)
    }
}
```

---

## 📝 สรุป Part 24

| แนวคิด | รายละเอียด |
|--------|-----------|
| `@Entity` | JPA entity class |
| `@Repository` | Data access interface |
| Method naming | Spring generates SQL |
| `@Query` | Custom JPQL/Native SQL |
| Relationships | One-to-One, One-to-Many, Many-to-Many |
| `Page<T>` | Paginated results |
| `@Transactional` | Database transaction |
| `@DataJpaTest` | Repository unit tests |

---

## ➡️ ถัดไป: Part 25 - Validation

---
*Part 24/100+ | Kotlin & Spring Boot Complete Course*
