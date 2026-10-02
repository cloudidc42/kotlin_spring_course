# Part 56: GraphQL กับ Spring Boot - Blog API

## บทนำ

**GraphQL** คือภาษาสำหรับ querying API ที่ Facebook พัฒนาขึ้นในปี 2012 และเปิดตัวสู่สาธารณะในปี 2015 GraphQL แก้ปัญหาของ REST API อย่าง **Over-fetching** (ดึงข้อมูลมากกว่าที่ต้องการ) และ **Under-fetching** (ต้องเรียก API หลายครั้ง) โดยให้ client ระบุได้ว่าต้องการข้อมูลอะไรบ้าง

## เปรียบเทียบ REST vs GraphQL

### REST API (Over-fetching)
```
GET /api/posts/1
Response: { id, title, content, authorId, tags, createdAt, updatedAt, ... }
// แต่เราต้องการแค่ title และ author name
```

### GraphQL (Precise fetching)
```graphql
query {
  post(id: 1) {
    title
    author {
      name
    }
  }
}
```

## การตั้งค่าโปรเจกต์

```kotlin
// build.gradle.kts
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-graphql")
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
    implementation("org.springframework.boot:spring-boot-starter-validation")
    
    // GraphQL subscriptions support
    implementation("org.springframework.boot:spring-boot-starter-websocket")
    
    // Testing
    testImplementation("org.springframework.graphql:spring-graphql-test")
    testImplementation("org.springframework.boot:spring-boot-starter-test")
}
```

```yaml
# application.yml
spring:
  graphql:
    graphiql:
      enabled: true  # เปิด GraphiQL playground ที่ /graphiql
    path: /graphql
    websocket:
      path: /graphql-ws
  jpa:
    hibernate:
      ddl-auto: create-drop
    show-sql: true
```

## Schema Definition

### GraphQL Schema

```graphql
# src/main/resources/graphql/schema.graphqls

# Types
type Post {
    id: ID!
    title: String!
    content: String!
    author: Author!
    tags: [Tag!]!
    comments: [Comment!]!
    published: Boolean!
    createdAt: String!
    updatedAt: String!
}

type Author {
    id: ID!
    name: String!
    email: String!
    bio: String
    posts: [Post!]!
}

type Tag {
    id: ID!
    name: String!
    posts: [Post!]!
}

type Comment {
    id: ID!
    content: String!
    author: Author!
    post: Post!
    createdAt: String!
}

# Pagination types
type PostPage {
    content: [Post!]!
    totalElements: Int!
    totalPages: Int!
    currentPage: Int!
    hasNext: Boolean!
    hasPrevious: Boolean!
}

# Input types
input CreatePostInput {
    title: String!
    content: String!
    tagIds: [ID!]
    published: Boolean = false
}

input UpdatePostInput {
    title: String
    content: String
    tagIds: [ID!]
    published: Boolean
}

input CreateCommentInput {
    postId: ID!
    content: String!
}

input PostFilterInput {
    authorId: ID
    tagIds: [ID!]
    published: Boolean
    searchTerm: String
}

# Queries
type Query {
    # Posts
    post(id: ID!): Post
    posts(
        page: Int = 0
        size: Int = 10
        filter: PostFilterInput
    ): PostPage!
    
    # Authors
    author(id: ID!): Author
    authors: [Author!]!
    
    # Tags
    tags: [Tag!]!
    tag(id: ID!): Tag
    
    # Comments
    commentsByPost(postId: ID!): [Comment!]!
}

# Mutations
type Mutation {
    # Post mutations
    createPost(input: CreatePostInput!): Post!
    updatePost(id: ID!, input: UpdatePostInput!): Post!
    deletePost(id: ID!): Boolean!
    publishPost(id: ID!): Post!
    
    # Comment mutations
    addComment(input: CreateCommentInput!): Comment!
    deleteComment(id: ID!): Boolean!
    
    # Tag mutations
    createTag(name: String!): Tag!
}

# Subscriptions
type Subscription {
    newPostPublished: Post!
    newCommentOnPost(postId: ID!): Comment!
}
```

## Domain Models

```kotlin
// Post.kt
package com.blog.graphql.domain

import jakarta.persistence.*
import java.time.LocalDateTime

@Entity
@Table(name = "posts")
data class Post(
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long = 0,
    
    @Column(nullable = false)
    val title: String,
    
    @Column(columnDefinition = "TEXT", nullable = false)
    val content: String,
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "author_id", nullable = false)
    val author: Author,
    
    @ManyToMany(fetch = FetchType.LAZY)
    @JoinTable(
        name = "post_tags",
        joinColumns = [JoinColumn(name = "post_id")],
        inverseJoinColumns = [JoinColumn(name = "tag_id")]
    )
    val tags: MutableList<Tag> = mutableListOf(),
    
    @OneToMany(mappedBy = "post", cascade = [CascadeType.ALL], fetch = FetchType.LAZY)
    val comments: MutableList<Comment> = mutableListOf(),
    
    val published: Boolean = false,
    
    val createdAt: LocalDateTime = LocalDateTime.now(),
    val updatedAt: LocalDateTime = LocalDateTime.now()
)
```

```kotlin
// Author.kt
package com.blog.graphql.domain

import jakarta.persistence.*

@Entity
@Table(name = "authors")
data class Author(
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long = 0,
    
    @Column(nullable = false)
    val name: String,
    
    @Column(unique = true, nullable = false)
    val email: String,
    
    val bio: String? = null,
    
    @OneToMany(mappedBy = "author", fetch = FetchType.LAZY)
    val posts: List<Post> = emptyList()
)
```

```kotlin
// Tag.kt
package com.blog.graphql.domain

import jakarta.persistence.*

@Entity
@Table(name = "tags")
data class Tag(
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long = 0,
    
    @Column(unique = true, nullable = false)
    val name: String,
    
    @ManyToMany(mappedBy = "tags")
    val posts: List<Post> = emptyList()
)
```

```kotlin
// Comment.kt
package com.blog.graphql.domain

import jakarta.persistence.*
import java.time.LocalDateTime

@Entity
@Table(name = "comments")
data class Comment(
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long = 0,
    
    @Column(columnDefinition = "TEXT", nullable = false)
    val content: String,
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "author_id", nullable = false)
    val author: Author,
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "post_id", nullable = false)
    val post: Post,
    
    val createdAt: LocalDateTime = LocalDateTime.now()
)
```

## GraphQL Controllers

### Query Controller

```kotlin
// PostQueryController.kt
package com.blog.graphql.controller

import com.blog.graphql.domain.Post
import com.blog.graphql.dto.*
import com.blog.graphql.service.PostService
import org.springframework.graphql.data.method.annotation.Argument
import org.springframework.graphql.data.method.annotation.QueryMapping
import org.springframework.stereotype.Controller

@Controller
class PostQueryController(
    private val postService: PostService
) {

    @QueryMapping
    fun post(@Argument id: Long): Post? {
        return postService.findById(id)
    }

    @QueryMapping
    fun posts(
        @Argument page: Int,
        @Argument size: Int,
        @Argument filter: PostFilterInput?
    ): PostPage {
        return postService.findAll(page, size, filter)
    }
}
```

```kotlin
// AuthorQueryController.kt
package com.blog.graphql.controller

import com.blog.graphql.domain.Author
import com.blog.graphql.service.AuthorService
import org.springframework.graphql.data.method.annotation.Argument
import org.springframework.graphql.data.method.annotation.QueryMapping
import org.springframework.stereotype.Controller

@Controller
class AuthorQueryController(
    private val authorService: AuthorService
) {

    @QueryMapping
    fun author(@Argument id: Long): Author? {
        return authorService.findById(id)
    }

    @QueryMapping
    fun authors(): List<Author> {
        return authorService.findAll()
    }
}
```

### Mutation Controller

```kotlin
// PostMutationController.kt
package com.blog.graphql.controller

import com.blog.graphql.domain.Post
import com.blog.graphql.dto.*
import com.blog.graphql.service.PostService
import org.springframework.graphql.data.method.annotation.Argument
import org.springframework.graphql.data.method.annotation.MutationMapping
import org.springframework.stereotype.Controller

@Controller
class PostMutationController(
    private val postService: PostService
) {

    @MutationMapping
    fun createPost(@Argument input: CreatePostInput): Post {
        return postService.createPost(input)
    }

    @MutationMapping
    fun updatePost(
        @Argument id: Long,
        @Argument input: UpdatePostInput
    ): Post {
        return postService.updatePost(id, input)
    }

    @MutationMapping
    fun deletePost(@Argument id: Long): Boolean {
        return postService.deletePost(id)
    }

    @MutationMapping
    fun publishPost(@Argument id: Long): Post {
        return postService.publishPost(id)
    }
}
```

### Subscription Controller

```kotlin
// PostSubscriptionController.kt
package com.blog.graphql.controller

import com.blog.graphql.domain.Comment
import com.blog.graphql.domain.Post
import com.blog.graphql.service.PostEventPublisher
import org.springframework.graphql.data.method.annotation.Argument
import org.springframework.graphql.data.method.annotation.SubscriptionMapping
import org.springframework.stereotype.Controller
import reactor.core.publisher.Flux

@Controller
class PostSubscriptionController(
    private val postEventPublisher: PostEventPublisher
) {

    @SubscriptionMapping
    fun newPostPublished(): Flux<Post> {
        return postEventPublisher.getNewPostFlux()
    }

    @SubscriptionMapping
    fun newCommentOnPost(@Argument postId: Long): Flux<Comment> {
        return postEventPublisher.getCommentFlux(postId)
    }
}
```

## N+1 Problem และ DataLoader

### ปัญหา N+1

เมื่อ query posts พร้อม author จะเกิด N+1 queries:
```graphql
query {
  posts {
    title
    author {   # ← เรียก DB N ครั้งสำหรับแต่ละ post!
      name
    }
  }
}
```

### แก้ปัญหาด้วย DataLoader

```kotlin
// AuthorDataLoader.kt
package com.blog.graphql.dataloader

import com.blog.graphql.domain.Author
import com.blog.graphql.repository.AuthorRepository
import org.dataloader.BatchLoaderEnvironment
import org.dataloader.MappedBatchLoaderWithContext
import org.springframework.stereotype.Component
import java.util.concurrent.CompletableFuture
import java.util.concurrent.CompletableFuture.supplyAsync

@Component("authorDataLoader")
class AuthorDataLoader(
    private val authorRepository: AuthorRepository
) : MappedBatchLoaderWithContext<Long, Author> {

    override fun load(
        keys: Set<Long>,
        environment: BatchLoaderEnvironment
    ): CompletableFuture<Map<Long, Author>> {
        return supplyAsync {
            // โหลด authors ทีเดียวหมดด้วย IN query แทนที่จะโหลดทีละ row
            authorRepository.findAllById(keys)
                .associateBy { it.id }
        }
    }
}
```

```kotlin
// DataLoaderRegistrar.kt
package com.blog.graphql.dataloader

import org.dataloader.DataLoaderFactory
import org.dataloader.DataLoaderOptions
import org.springframework.graphql.execution.BatchLoaderRegistry
import org.springframework.stereotype.Component

@Component
class DataLoaderRegistrar(
    private val authorDataLoader: AuthorDataLoader
) {

    fun register(registry: BatchLoaderRegistry) {
        val options = DataLoaderOptions.newOptions()
            .setBatchingEnabled(true)
            .setCachingEnabled(true)
            .setMaxBatchSize(100)

        registry.forTypePair(Long::class.java, Author::class.java)
            .withName("authorDataLoader")
            .registerMappedBatchLoader { keys, env ->
                authorDataLoader.load(keys, env)
            }
    }
}
```

```kotlin
// PostResolver.kt - ใช้ DataLoader ใน SchemaMapping
package com.blog.graphql.resolver

import com.blog.graphql.domain.Author
import com.blog.graphql.domain.Post
import org.springframework.graphql.data.method.annotation.SchemaMapping
import org.springframework.graphql.execution.ReactorContextManager
import org.springframework.stereotype.Controller
import graphql.schema.DataFetchingEnvironment
import org.dataloader.DataLoader
import java.util.concurrent.CompletableFuture

@Controller
class PostResolver {

    @SchemaMapping(typeName = "Post", field = "author")
    fun author(
        post: Post,
        dataFetchingEnvironment: DataFetchingEnvironment
    ): CompletableFuture<Author> {
        val dataLoader: DataLoader<Long, Author> = 
            dataFetchingEnvironment.getDataLoader("authorDataLoader")
        
        // ใช้ DataLoader แทนการ query ตรงๆ
        return dataLoader.load(post.author.id)
    }
}
```

## Service Layer

```kotlin
// PostService.kt
package com.blog.graphql.service

import com.blog.graphql.domain.Post
import com.blog.graphql.dto.*
import com.blog.graphql.exception.PostNotFoundException
import com.blog.graphql.repository.PostRepository
import com.blog.graphql.repository.TagRepository
import org.springframework.data.domain.PageRequest
import org.springframework.data.domain.Sort
import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional

@Service
class PostService(
    private val postRepository: PostRepository,
    private val tagRepository: TagRepository,
    private val postEventPublisher: PostEventPublisher
) {

    fun findById(id: Long): Post? {
        return postRepository.findById(id).orElse(null)
    }

    fun findAll(page: Int, size: Int, filter: PostFilterInput?): PostPage {
        val pageable = PageRequest.of(page, size, Sort.by("createdAt").descending())
        
        val postPage = if (filter != null) {
            postRepository.findAllWithFilter(
                authorId = filter.authorId,
                tagIds = filter.tagIds,
                published = filter.published,
                searchTerm = filter.searchTerm,
                pageable = pageable
            )
        } else {
            postRepository.findAll(pageable)
        }

        return PostPage(
            content = postPage.content,
            totalElements = postPage.totalElements.toInt(),
            totalPages = postPage.totalPages,
            currentPage = page,
            hasNext = postPage.hasNext(),
            hasPrevious = postPage.hasPrevious()
        )
    }

    @Transactional
    fun createPost(input: CreatePostInput): Post {
        val author = getCurrentUser() // ดึง user ปัจจุบัน
        val tags = input.tagIds?.let { tagRepository.findAllById(it) } ?: emptyList()
        
        val post = Post(
            title = input.title,
            content = input.content,
            author = author,
            tags = tags.toMutableList(),
            published = input.published
        )

        return postRepository.save(post)
    }

    @Transactional
    fun publishPost(id: Long): Post {
        val post = postRepository.findById(id)
            .orElseThrow { PostNotFoundException("Post $id not found") }
        
        val publishedPost = post.copy(published = true)
        val saved = postRepository.save(publishedPost)
        
        // ส่ง event สำหรับ subscription
        postEventPublisher.publishNewPost(saved)
        
        return saved
    }

    @Transactional
    fun updatePost(id: Long, input: UpdatePostInput): Post {
        val post = postRepository.findById(id)
            .orElseThrow { PostNotFoundException("Post $id not found") }

        val updatedPost = post.copy(
            title = input.title ?: post.title,
            content = input.content ?: post.content,
            published = input.published ?: post.published
        )

        return postRepository.save(updatedPost)
    }

    fun deletePost(id: Long): Boolean {
        return if (postRepository.existsById(id)) {
            postRepository.deleteById(id)
            true
        } else {
            false
        }
    }
}
```

### Event Publisher สำหรับ Subscription

```kotlin
// PostEventPublisher.kt
package com.blog.graphql.service

import com.blog.graphql.domain.Comment
import com.blog.graphql.domain.Post
import org.springframework.stereotype.Component
import reactor.core.publisher.Flux
import reactor.core.publisher.Sinks

@Component
class PostEventPublisher {

    private val newPostSink = Sinks.many()
        .multicast()
        .onBackpressureBuffer<Post>()

    private val newCommentSinks = mutableMapOf<Long, Sinks.Many<Comment>>()

    fun publishNewPost(post: Post) {
        newPostSink.tryEmitNext(post)
    }

    fun publishNewComment(comment: Comment) {
        newCommentSinks[comment.post.id]?.tryEmitNext(comment)
    }

    fun getNewPostFlux(): Flux<Post> {
        return newPostSink.asFlux()
    }

    fun getCommentFlux(postId: Long): Flux<Comment> {
        return newCommentSinks.getOrPut(postId) {
            Sinks.many().multicast().onBackpressureBuffer()
        }.asFlux()
    }
}
```

## DTOs

```kotlin
// Dtos.kt
package com.blog.graphql.dto

import com.blog.graphql.domain.Post

data class CreatePostInput(
    val title: String,
    val content: String,
    val tagIds: List<Long>? = null,
    val published: Boolean = false
)

data class UpdatePostInput(
    val title: String? = null,
    val content: String? = null,
    val tagIds: List<Long>? = null,
    val published: Boolean? = null
)

data class CreateCommentInput(
    val postId: Long,
    val content: String
)

data class PostFilterInput(
    val authorId: Long? = null,
    val tagIds: List<Long>? = null,
    val published: Boolean? = null,
    val searchTerm: String? = null
)

data class PostPage(
    val content: List<Post>,
    val totalElements: Int,
    val totalPages: Int,
    val currentPage: Int,
    val hasNext: Boolean,
    val hasPrevious: Boolean
)
```

## Error Handling

```kotlin
// GraphQLExceptionHandler.kt
package com.blog.graphql.exception

import graphql.GraphQLError
import graphql.GraphqlErrorBuilder
import graphql.schema.DataFetchingEnvironment
import org.springframework.graphql.execution.DataFetcherExceptionResolverAdapter
import org.springframework.graphql.execution.ErrorType
import org.springframework.stereotype.Component

@Component
class GraphQLExceptionHandler : DataFetcherExceptionResolverAdapter() {

    override fun resolveToSingleError(
        ex: Throwable,
        env: DataFetchingEnvironment
    ): GraphQLError? {
        return when (ex) {
            is PostNotFoundException -> GraphqlErrorBuilder.newError()
                .errorType(ErrorType.NOT_FOUND)
                .message(ex.message)
                .path(env.executionStepInfo.path)
                .location(env.field.sourceLocation)
                .build()

            is UnauthorizedException -> GraphqlErrorBuilder.newError()
                .errorType(ErrorType.UNAUTHORIZED)
                .message("You are not authorized to perform this action")
                .path(env.executionStepInfo.path)
                .location(env.field.sourceLocation)
                .build()

            is ValidationException -> GraphqlErrorBuilder.newError()
                .errorType(ErrorType.BAD_REQUEST)
                .message(ex.message)
                .path(env.executionStepInfo.path)
                .location(env.field.sourceLocation)
                .build()

            else -> null
        }
    }
}
```

## Testing GraphQL

```kotlin
// PostGraphQLTest.kt
package com.blog.graphql.controller

import org.junit.jupiter.api.Test
import org.springframework.beans.factory.annotation.Autowired
import org.springframework.boot.test.autoconfigure.graphql.GraphQlTest
import org.springframework.graphql.test.tester.GraphQlTester
import org.springframework.boot.test.mock.mockito.MockBean
import org.mockito.BDDMockito.given

@GraphQlTest(PostQueryController::class)
class PostGraphQLTest {

    @Autowired
    private lateinit var graphQlTester: GraphQlTester

    @MockBean
    private lateinit var postService: PostService

    @Test
    fun `should return post by id`() {
        val mockPost = Post(
            id = 1L,
            title = "Test Post",
            content = "Test Content",
            author = Author(id = 1L, name = "John", email = "john@test.com")
        )
        
        given(postService.findById(1L)).willReturn(mockPost)

        graphQlTester.document("""
            query {
                post(id: "1") {
                    title
                    content
                    author {
                        name
                    }
                }
            }
        """)
        .execute()
        .path("post.title").entity(String::class.java).isEqualTo("Test Post")
        .path("post.author.name").entity(String::class.java).isEqualTo("John")
    }

    @Test
    fun `should create post`() {
        graphQlTester.document("""
            mutation {
                createPost(input: {
                    title: "New Post"
                    content: "Post content"
                    published: true
                }) {
                    id
                    title
                    published
                }
            }
        """)
        .execute()
        .path("createPost.title").entity(String::class.java).isEqualTo("New Post")
    }

    @Test
    fun `should return paginated posts`() {
        graphQlTester.document("""
            query {
                posts(page: 0, size: 10) {
                    content {
                        id
                        title
                    }
                    totalElements
                    hasNext
                }
            }
        """)
        .execute()
        .path("posts.totalElements").entity(Int::class.java)
        .path("posts.content").entityList(Map::class.java)
    }
}
```

## GraphQL Playground

หลังจาก run application สามารถเข้าใช้ GraphiQL ได้ที่ `http://localhost:8080/graphiql`

```graphql
# ตัวอย่าง Query
query GetBlogPosts {
  posts(page: 0, size: 5, filter: { published: true }) {
    content {
      id
      title
      author {
        name
        email
      }
      tags {
        name
      }
    }
    totalElements
    totalPages
    hasNext
  }
}

# ตัวอย่าง Mutation
mutation CreateBlogPost {
  createPost(input: {
    title: "Learning GraphQL with Kotlin"
    content: "GraphQL is amazing!"
    tagIds: [1, 2]
    published: false
  }) {
    id
    title
    published
    createdAt
  }
}

# ตัวอย่าง Subscription
subscription WatchNewPosts {
  newPostPublished {
    id
    title
    author {
      name
    }
  }
}
```

## สรุปแนวคิด GraphQL

| แนวคิด | คำอธิบาย | ตัวอย่าง |
|--------|---------|---------|
| Query | อ่านข้อมูล (เหมือน GET) | `query { post(id: 1) { title } }` |
| Mutation | เปลี่ยนแปลงข้อมูล (POST/PUT/DELETE) | `mutation { createPost(...) { id } }` |
| Subscription | รับข้อมูล real-time | `subscription { newPost { title } }` |
| Schema | กำหนดโครงสร้างข้อมูล | Types, Queries, Mutations |
| Resolver | ฟังก์ชันที่ดึงข้อมูลจริงๆ | @QueryMapping, @SchemaMapping |
| DataLoader | แก้ N+1 problem | BatchLoader สำหรับ relationship |
| Fragment | ใช้ field ซ้ำๆ ได้ | `fragment PostFields on Post { id title }` |

## ข้อดีของ GraphQL

1. **Flexible Queries** - Client เลือกข้อมูลที่ต้องการได้เอง
2. **Single Endpoint** - ใช้แค่ `/graphql` สำหรับทุกอย่าง
3. **Strongly Typed** - Schema ช่วย validate ก่อน execute
4. **Introspection** - Client รู้ได้ว่า API รองรับอะไร
5. **Real-time** - Subscription รองรับ real-time data
6. **Versioning-free** - เพิ่ม field ใหม่ไม่กระทบ client เก่า

*Part 56/100+ | Kotlin & Spring Boot Complete Course*
