# Part 91: Real World Project 1 - Social Media API
## สร้าง Social Media Platform API

---

## 🎯 เป้าหมายของ Part นี้

- สร้าง complete Social Media API
- Users, Posts, Likes, Comments, Follows
- Newsfeed generation
- Real-time notifications
- Image upload

---

## 📋 1. Requirements

```
Features:
- User registration, login, profile management
- Create/edit/delete posts (text + images)
- Like/unlike posts
- Comment on posts
- Follow/unfollow users
- View newsfeed (posts from followed users)
- Notifications (likes, comments, follows)
- Search users and posts
- Trending hashtags
```

---

## 🏗️ 2. Project Structure

```
social-media-api/
├── src/main/kotlin/com/example/social/
│   ├── config/
│   │   ├── SecurityConfig.kt
│   │   ├── RedisConfig.kt
│   │   └── WebConfig.kt
│   ├── entity/
│   │   ├── User.kt
│   │   ├── Post.kt
│   │   ├── Comment.kt
│   │   ├── Like.kt
│   │   ├── Follow.kt
│   │   └── Notification.kt
│   ├── repository/
│   │   ├── UserRepository.kt
│   │   ├── PostRepository.kt
│   │   └── ...
│   ├── service/
│   │   ├── UserService.kt
│   │   ├── PostService.kt
│   │   ├── FeedService.kt
│   │   └── NotificationService.kt
│   ├── controller/
│   │   ├── AuthController.kt
│   │   ├── UserController.kt
│   │   ├── PostController.kt
│   │   └── FeedController.kt
│   ├── dto/
│   └── SocialMediaApp.kt
└── build.gradle.kts
```

---

## 📦 3. build.gradle.kts

```kotlin
plugins {
    kotlin("jvm") version "1.9.21"
    kotlin("plugin.spring") version "1.9.21"
    kotlin("plugin.jpa") version "1.9.21"
    id("org.springframework.boot") version "3.2.1"
    id("io.spring.dependency-management") version "1.1.4"
}

dependencies {
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
    implementation("org.springframework.boot:spring-boot-starter-security")
    implementation("org.springframework.boot:spring-boot-starter-data-redis")
    implementation("org.springframework.boot:spring-boot-starter-validation")
    implementation("org.springframework.boot:spring-boot-starter-actuator")
    implementation("org.springframework.boot:spring-boot-starter-cache")
    implementation("com.auth0:java-jwt:4.4.0")
    implementation("com.fasterxml.jackson.module:jackson-module-kotlin")
    implementation("org.jetbrains.kotlin:kotlin-reflect")
    runtimeOnly("org.postgresql:postgresql")
    testImplementation("org.springframework.boot:spring-boot-starter-test")
    testImplementation("org.springframework.security:spring-security-test")
}
```

---

## 🗃️ 4. Entities

```kotlin
// entity/User.kt
@Entity
@Table(name = "users")
data class User(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long = 0,

    @Column(unique = true, nullable = false)
    val username: String,

    @Column(unique = true, nullable = false)
    val email: String,

    @Column(nullable = false)
    val passwordHash: String,

    val displayName: String = username,
    val bio: String? = null,
    val avatarUrl: String? = null,
    val isVerified: Boolean = false,
    val isPrivate: Boolean = false,

    @Column(updatable = false)
    val createdAt: Instant = Instant.now(),

    @OneToMany(mappedBy = "author", cascade = [CascadeType.ALL], fetch = FetchType.LAZY)
    val posts: List<Post> = emptyList(),

    @OneToMany(mappedBy = "follower", fetch = FetchType.LAZY)
    val followings: List<Follow> = emptyList(),

    @OneToMany(mappedBy = "following", fetch = FetchType.LAZY)
    val followers: List<Follow> = emptyList()
)
```

```kotlin
// entity/Post.kt
@Entity
@Table(name = "posts")
data class Post(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long = 0,

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id", nullable = false)
    val author: User,

    @Column(nullable = false, length = 2200)
    val content: String,

    @ElementCollection
    @CollectionTable(name = "post_images", joinColumns = [JoinColumn(name = "post_id")])
    val imageUrls: List<String> = emptyList(),

    @ElementCollection
    @CollectionTable(name = "post_hashtags", joinColumns = [JoinColumn(name = "post_id")])
    val hashtags: List<String> = emptyList(),

    val likesCount: Int = 0,
    val commentsCount: Int = 0,
    val sharesCount: Int = 0,

    @Enumerated(EnumType.STRING)
    val visibility: PostVisibility = PostVisibility.PUBLIC,

    @Column(updatable = false)
    val createdAt: Instant = Instant.now(),
    val updatedAt: Instant = Instant.now(),

    @OneToMany(mappedBy = "post", cascade = [CascadeType.ALL])
    val likes: List<Like> = emptyList(),

    @OneToMany(mappedBy = "post", cascade = [CascadeType.ALL])
    val comments: List<Comment> = emptyList()
)

enum class PostVisibility { PUBLIC, FOLLOWERS_ONLY, PRIVATE }
```

```kotlin
// entity/Follow.kt
@Entity
@Table(
    name = "follows",
    uniqueConstraints = [UniqueConstraint(columnNames = ["follower_id", "following_id"])]
)
data class Follow(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long = 0,

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "follower_id", nullable = false)
    val follower: User,

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "following_id", nullable = false)
    val following: User,

    val createdAt: Instant = Instant.now()
)
```

---

## 🔧 5. Services

```kotlin
// service/PostService.kt
@Service
@Transactional
class PostService(
    private val postRepository: PostRepository,
    private val userRepository: UserRepository,
    private val notificationService: NotificationService,
    private val hashtagService: HashtagService,
    private val feedInvalidator: FeedInvalidator
) {
    fun createPost(userId: Long, request: CreatePostRequest): PostDto {
        val author = userRepository.findById(userId)
            .orElseThrow { UserNotFoundException("User $userId not found") }

        val hashtags = extractHashtags(request.content)

        val post = postRepository.save(
            Post(
                author = author,
                content = request.content,
                imageUrls = request.imageUrls ?: emptyList(),
                hashtags = hashtags,
                visibility = request.visibility ?: PostVisibility.PUBLIC
            )
        )

        // Update hashtag trending
        hashtagService.incrementHashtags(hashtags)

        // Invalidate followers' feed cache
        feedInvalidator.invalidateFollowerFeeds(userId)

        return post.toDto(currentUserId = userId)
    }

    fun likePost(userId: Long, postId: Long): LikeResult {
        val post = postRepository.findById(postId)
            .orElseThrow { PostNotFoundException("Post $postId not found") }

        val existingLike = likeRepository.findByUserIdAndPostId(userId, postId)

        return if (existingLike != null) {
            // Unlike
            likeRepository.delete(existingLike)
            postRepository.decrementLikesCount(postId)
            LikeResult(liked = false, likesCount = post.likesCount - 1)
        } else {
            // Like
            likeRepository.save(Like(userId = userId, post = post))
            postRepository.incrementLikesCount(postId)

            // Notify post author
            if (post.author.id != userId) {
                notificationService.createLikeNotification(userId, post)
            }

            LikeResult(liked = true, likesCount = post.likesCount + 1)
        }
    }

    private fun extractHashtags(content: String): List<String> {
        val regex = Regex("#(\\w+)")
        return regex.findAll(content)
            .map { it.groupValues[1].lowercase() }
            .distinct()
            .take(30)  // Max 30 hashtags
    }

    @Transactional(readOnly = true)
    fun getPost(postId: Long, currentUserId: Long?): PostDto {
        val post = postRepository.findById(postId)
            .orElseThrow { PostNotFoundException("Post $postId not found") }

        // Check visibility
        if (post.visibility == PostVisibility.PRIVATE &&
            post.author.id != currentUserId) {
            throw ForbiddenException("Cannot access private post")
        }

        return post.toDto(currentUserId = currentUserId)
    }
}
```

```kotlin
// service/FeedService.kt
@Service
class FeedService(
    private val postRepository: PostRepository,
    private val followRepository: FollowRepository,
    private val redisTemplate: RedisTemplate<String, String>
) {
    @Transactional(readOnly = true)
    fun getHomeFeed(userId: Long, page: Int = 0, size: Int = 20): PageResponse<PostDto> {
        val cacheKey = "feed:home:$userId:$page"

        // Try cache first
        val cachedIds = redisTemplate.opsForValue().get(cacheKey)
        if (cachedIds != null) {
            val postIds = cachedIds.split(",").map { it.toLong() }
            val posts = postRepository.findAllById(postIds)
            return PageResponse(
                content = posts.map { it.toDto(userId) },
                page = page,
                size = size,
                totalElements = posts.size.toLong()
            )
        }

        // Get following IDs
        val followingIds = followRepository.findFollowingIdsByUserId(userId)

        // Include own posts
        val authorIds = followingIds + userId

        // Get recent posts
        val pageable = PageRequest.of(page, size, Sort.by("createdAt").descending())
        val posts = postRepository.findByAuthorIdInAndVisibilityIn(
            authorIds,
            listOf(PostVisibility.PUBLIC, PostVisibility.FOLLOWERS_ONLY),
            pageable
        )

        // Cache for 5 minutes
        val postIdStr = posts.content.joinToString(",") { it.id.toString() }
        redisTemplate.opsForValue().set(cacheKey, postIdStr, Duration.ofMinutes(5))

        return PageResponse(
            content = posts.content.map { it.toDto(userId) },
            page = page,
            size = posts.totalPages,
            totalElements = posts.totalElements
        )
    }
}
```

---

## 🌐 6. Controllers

```kotlin
// controller/PostController.kt
@RestController
@RequestMapping("/api/v1/posts")
class PostController(
    private val postService: PostService
) {
    @PostMapping
    fun createPost(
        @AuthenticationPrincipal user: UserPrincipal,
        @Valid @RequestBody request: CreatePostRequest
    ): ResponseEntity<PostDto> {
        val post = postService.createPost(user.id, request)
        val uri = URI.create("/api/v1/posts/${post.id}")
        return ResponseEntity.created(uri).body(post)
    }

    @GetMapping("/{id}")
    fun getPost(
        @PathVariable id: Long,
        @AuthenticationPrincipal user: UserPrincipal?
    ): ResponseEntity<PostDto> {
        return ResponseEntity.ok(postService.getPost(id, user?.id))
    }

    @PostMapping("/{id}/like")
    fun likePost(
        @PathVariable id: Long,
        @AuthenticationPrincipal user: UserPrincipal
    ): ResponseEntity<LikeResult> {
        return ResponseEntity.ok(postService.likePost(user.id, id))
    }

    @PostMapping("/{id}/comments")
    fun addComment(
        @PathVariable id: Long,
        @AuthenticationPrincipal user: UserPrincipal,
        @Valid @RequestBody request: CreateCommentRequest
    ): ResponseEntity<CommentDto> {
        val comment = postService.addComment(user.id, id, request)
        return ResponseEntity.status(HttpStatus.CREATED).body(comment)
    }

    @DeleteMapping("/{id}")
    fun deletePost(
        @PathVariable id: Long,
        @AuthenticationPrincipal user: UserPrincipal
    ): ResponseEntity<Void> {
        postService.deletePost(user.id, id)
        return ResponseEntity.noContent().build()
    }
}
```

```kotlin
// controller/UserController.kt
@RestController
@RequestMapping("/api/v1/users")
class UserController(
    private val userService: UserService,
    private val feedService: FeedService
) {
    @GetMapping("/{username}")
    fun getProfile(@PathVariable username: String): ResponseEntity<UserProfileDto> =
        ResponseEntity.ok(userService.getProfileByUsername(username))

    @PutMapping("/me")
    fun updateProfile(
        @AuthenticationPrincipal user: UserPrincipal,
        @Valid @RequestBody request: UpdateProfileRequest
    ): ResponseEntity<UserProfileDto> =
        ResponseEntity.ok(userService.updateProfile(user.id, request))

    @PostMapping("/{id}/follow")
    fun followUser(
        @PathVariable id: Long,
        @AuthenticationPrincipal user: UserPrincipal
    ): ResponseEntity<FollowResult> {
        if (id == user.id) {
            throw BadRequestException("Cannot follow yourself")
        }
        return ResponseEntity.ok(userService.followUser(user.id, id))
    }

    @GetMapping("/{id}/followers")
    fun getFollowers(
        @PathVariable id: Long,
        @RequestParam(defaultValue = "0") page: Int,
        @RequestParam(defaultValue = "20") size: Int
    ): ResponseEntity<PageResponse<UserSummaryDto>> =
        ResponseEntity.ok(userService.getFollowers(id, page, size))

    @GetMapping("/{id}/posts")
    fun getUserPosts(
        @PathVariable id: Long,
        @AuthenticationPrincipal currentUser: UserPrincipal?,
        @RequestParam(defaultValue = "0") page: Int
    ): ResponseEntity<PageResponse<PostDto>> =
        ResponseEntity.ok(postService.getUserPosts(id, currentUser?.id, page))
}
```

---

## 📊 7. Database Schema

```sql
-- schema.sql
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    display_name VARCHAR(100),
    bio TEXT,
    avatar_url VARCHAR(500),
    is_verified BOOLEAN DEFAULT FALSE,
    is_private BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE TABLE posts (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    content TEXT NOT NULL,
    likes_count INT DEFAULT 0,
    comments_count INT DEFAULT 0,
    visibility VARCHAR(20) DEFAULT 'PUBLIC',
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE TABLE follows (
    id BIGSERIAL PRIMARY KEY,
    follower_id BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    following_id BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    UNIQUE(follower_id, following_id)
);

-- Indexes for performance
CREATE INDEX idx_posts_user_id ON posts(user_id);
CREATE INDEX idx_posts_created_at ON posts(created_at DESC);
CREATE INDEX idx_follows_follower ON follows(follower_id);
CREATE INDEX idx_follows_following ON follows(following_id);
```

---

## 📋 สรุป

| Feature | Implementation |
|---------|---------------|
| Authentication | JWT + Spring Security |
| Posts | CRUD + Media Upload |
| Social Graph | Follow/Unfollow + counting |
| Newsfeed | Fanout + Redis cache |
| Likes | Toggle + count denormalization |
| Notifications | Async event-driven |
| Search | Full-text search (PostgreSQL) |

---

*Part 91/100+ | Kotlin & Spring Boot Complete Course*
