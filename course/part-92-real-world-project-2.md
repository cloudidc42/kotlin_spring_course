# Part 92: Real World Project 2 - E-Learning Platform API
## สร้าง Online Learning Platform

---

## 🎯 เป้าหมายของ Part นี้

- สร้าง E-Learning Platform API
- Courses, Lessons, Enrollments
- Progress tracking
- Quiz และ Assessment
- Certificate generation

---

## 📋 1. Requirements

```
Features:
- Instructors สร้าง courses และ lessons
- Students enroll ใน courses (free/paid)
- Progress tracking ต่อ lesson
- Quiz system พร้อม auto-grading
- Certificate เมื่อ complete course
- Review and rating system
- Course search และ categories
- Discussion forums
```

---

## 🏗️ 2. Domain Model

```kotlin
// entity/Course.kt
@Entity
@Table(name = "courses")
data class Course(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long = 0,

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "instructor_id", nullable = false)
    val instructor: User,

    @Column(nullable = false)
    val title: String,

    @Column(nullable = false, length = 5000)
    val description: String,

    val thumbnailUrl: String? = null,
    val previewVideoUrl: String? = null,

    @Column(nullable = false, precision = 10, scale = 2)
    val price: BigDecimal = BigDecimal.ZERO,

    @Enumerated(EnumType.STRING)
    val level: CourseLevel = CourseLevel.BEGINNER,

    @Enumerated(EnumType.STRING)
    val status: CourseStatus = CourseStatus.DRAFT,

    @ElementCollection
    @CollectionTable(name = "course_categories")
    val categories: Set<String> = emptySet(),

    @ElementCollection
    @CollectionTable(name = "course_tags")
    val tags: Set<String> = emptySet(),

    @ElementCollection
    @CollectionTable(name = "course_objectives")
    val learningObjectives: List<String> = emptyList(),

    @ElementCollection
    @CollectionTable(name = "course_requirements")
    val requirements: List<String> = emptyList(),

    val totalDurationMinutes: Int = 0,
    val enrollmentCount: Int = 0,
    val averageRating: Double = 0.0,
    val ratingCount: Int = 0,

    @Column(updatable = false)
    val createdAt: Instant = Instant.now(),
    val publishedAt: Instant? = null,

    @OneToMany(mappedBy = "course", cascade = [CascadeType.ALL], fetch = FetchType.LAZY)
    @OrderBy("sortOrder ASC")
    val sections: List<Section> = emptyList()
)

enum class CourseLevel { BEGINNER, INTERMEDIATE, ADVANCED }
enum class CourseStatus { DRAFT, PUBLISHED, ARCHIVED }
```

```kotlin
// entity/Section.kt
@Entity
data class Section(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long = 0,

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "course_id")
    val course: Course,

    val title: String,
    val description: String? = null,
    val sortOrder: Int = 0,

    @OneToMany(mappedBy = "section", cascade = [CascadeType.ALL])
    @OrderBy("sortOrder ASC")
    val lessons: List<Lesson> = emptyList()
)
```

```kotlin
// entity/Lesson.kt
@Entity
data class Lesson(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long = 0,

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "section_id")
    val section: Section,

    val title: String,
    val description: String? = null,

    @Enumerated(EnumType.STRING)
    val type: LessonType,

    val videoUrl: String? = null,
    val articleContent: String? = null,
    val attachmentUrls: List<String> = emptyList(),

    val durationMinutes: Int = 0,
    val sortOrder: Int = 0,
    val isFreePreview: Boolean = false
)

enum class LessonType { VIDEO, ARTICLE, QUIZ, ASSIGNMENT }
```

```kotlin
// entity/Enrollment.kt
@Entity
@Table(
    name = "enrollments",
    uniqueConstraints = [UniqueConstraint(columnNames = ["user_id", "course_id"])]
)
data class Enrollment(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long = 0,

    @ManyToOne(fetch = FetchType.LAZY)
    val user: User,

    @ManyToOne(fetch = FetchType.LAZY)
    val course: Course,

    @Enumerated(EnumType.STRING)
    val status: EnrollmentStatus = EnrollmentStatus.ACTIVE,

    val enrolledAt: Instant = Instant.now(),
    val completedAt: Instant? = null,
    val lastAccessedAt: Instant = Instant.now(),

    val progressPercent: Int = 0,
    val completedLessonsCount: Int = 0,

    // Payment info
    val amountPaid: BigDecimal = BigDecimal.ZERO,
    val paymentId: String? = null
)

enum class EnrollmentStatus { ACTIVE, COMPLETED, CANCELLED, REFUNDED }
```

---

## 🔧 3. Progress Tracking

```kotlin
// service/ProgressService.kt
@Service
@Transactional
class ProgressService(
    private val lessonProgressRepository: LessonProgressRepository,
    private val enrollmentRepository: EnrollmentRepository,
    private val certificateService: CertificateService
) {
    fun markLessonComplete(userId: Long, lessonId: Long): ProgressResult {
        // Find enrollment
        val lesson = lessonRepository.findById(lessonId)
            .orElseThrow { NotFoundException("Lesson not found") }

        val courseId = lesson.section.course.id
        val enrollment = enrollmentRepository.findByUserIdAndCourseId(userId, courseId)
            ?: throw NotEnrolledException("User not enrolled in this course")

        // Mark lesson as complete
        val progress = lessonProgressRepository.findByUserIdAndLessonId(userId, lessonId)
            ?: LessonProgress(userId = userId, lesson = lesson)

        val updated = progress.copy(
            completed = true,
            completedAt = Instant.now(),
            watchedPercent = 100
        )
        lessonProgressRepository.save(updated)

        // Recalculate course progress
        val totalLessons = lesson.section.course.totalLessonsCount()
        val completedLessons = lessonProgressRepository.countCompletedByUserIdAndCourseId(userId, courseId)
        val progressPercent = (completedLessons * 100 / totalLessons).toInt()

        enrollmentRepository.updateProgress(enrollment.id, progressPercent, completedLessons.toInt())

        // Check if course completed
        if (progressPercent == 100) {
            completeEnrollment(enrollment, userId, courseId)
        }

        return ProgressResult(
            lessonId = lessonId,
            courseProgress = progressPercent,
            isCoursCompleted = progressPercent == 100
        )
    }

    fun updateVideoProgress(userId: Long, lessonId: Long, watchedSeconds: Int, totalSeconds: Int) {
        val watchedPercent = ((watchedSeconds.toDouble() / totalSeconds) * 100).toInt().coerceIn(0, 100)

        val progress = lessonProgressRepository.findByUserIdAndLessonId(userId, lessonId)
            ?: LessonProgress(userId = userId, lesson = lessonRepository.getReferenceById(lessonId))

        // Only update if watched more
        if (watchedPercent > progress.watchedPercent) {
            lessonProgressRepository.save(
                progress.copy(
                    watchedPercent = watchedPercent,
                    completed = watchedPercent >= 90,  // 90% = completed
                    lastWatchedAt = Instant.now()
                )
            )
        }
    }

    private fun completeEnrollment(enrollment: Enrollment, userId: Long, courseId: Long) {
        enrollmentRepository.save(
            enrollment.copy(
                status = EnrollmentStatus.COMPLETED,
                completedAt = Instant.now()
            )
        )

        // Generate certificate
        certificateService.generateCertificate(userId, courseId)

        // Publish completion event
        eventPublisher.publishEvent(CourseCompletedEvent(userId, courseId))
    }
}
```

---

## 📝 4. Quiz System

```kotlin
// entity/Quiz.kt
@Entity
data class Quiz(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long = 0,

    @OneToOne
    val lesson: Lesson,

    val title: String,
    val description: String? = null,
    val passingScore: Int = 70,
    val maxAttempts: Int = 3,
    val timeLimitMinutes: Int? = null,

    @OneToMany(mappedBy = "quiz", cascade = [CascadeType.ALL])
    val questions: List<Question> = emptyList()
)

@Entity
data class Question(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long = 0,

    @ManyToOne
    val quiz: Quiz,

    val text: String,

    @Enumerated(EnumType.STRING)
    val type: QuestionType,

    @ElementCollection
    val options: List<String> = emptyList(),

    @ElementCollection
    val correctAnswers: List<String> = emptyList(),

    val explanation: String? = null,
    val points: Int = 1
)

enum class QuestionType { SINGLE_CHOICE, MULTIPLE_CHOICE, TRUE_FALSE, SHORT_ANSWER }

// service/QuizService.kt
@Service
@Transactional
class QuizService(
    private val quizRepository: QuizRepository,
    private val quizAttemptRepository: QuizAttemptRepository
) {
    fun submitQuiz(userId: Long, quizId: Long, answers: Map<Long, List<String>>): QuizResult {
        val quiz = quizRepository.findById(quizId)
            .orElseThrow { NotFoundException("Quiz not found") }

        // Check attempt count
        val attemptCount = quizAttemptRepository.countByUserIdAndQuizId(userId, quizId)
        if (attemptCount >= quiz.maxAttempts) {
            throw QuizException("Maximum attempts (${quiz.maxAttempts}) reached")
        }

        // Grade quiz
        var totalPoints = 0
        var earnedPoints = 0
        val questionResults = mutableListOf<QuestionResult>()

        quiz.questions.forEach { question ->
            totalPoints += question.points
            val userAnswers = answers[question.id] ?: emptyList()
            val isCorrect = checkAnswer(question, userAnswers)

            if (isCorrect) earnedPoints += question.points

            questionResults.add(
                QuestionResult(
                    questionId = question.id,
                    userAnswers = userAnswers,
                    correctAnswers = question.correctAnswers,
                    isCorrect = isCorrect,
                    explanation = question.explanation
                )
            )
        }

        val score = if (totalPoints > 0) (earnedPoints * 100 / totalPoints) else 0
        val passed = score >= quiz.passingScore

        // Save attempt
        quizAttemptRepository.save(
            QuizAttempt(
                userId = userId,
                quiz = quiz,
                score = score,
                passed = passed,
                submittedAt = Instant.now()
            )
        )

        return QuizResult(
            score = score,
            passed = passed,
            passingScore = quiz.passingScore,
            questionResults = questionResults,
            attemptsUsed = attemptCount + 1,
            attemptsRemaining = quiz.maxAttempts - attemptCount - 1
        )
    }

    private fun checkAnswer(question: Question, userAnswers: List<String>): Boolean {
        return when (question.type) {
            QuestionType.SINGLE_CHOICE, QuestionType.TRUE_FALSE ->
                userAnswers.size == 1 && userAnswers.first() in question.correctAnswers

            QuestionType.MULTIPLE_CHOICE ->
                userAnswers.toSet() == question.correctAnswers.toSet()

            QuestionType.SHORT_ANSWER ->
                userAnswers.firstOrNull()?.lowercase()?.trim() in
                    question.correctAnswers.map { it.lowercase().trim() }
        }
    }
}
```

---

## 🏆 5. Certificate Generation

```kotlin
// service/CertificateService.kt
@Service
class CertificateService(
    private val certificateRepository: CertificateRepository,
    private val pdfGenerator: PdfCertificateGenerator
) {
    fun generateCertificate(userId: Long, courseId: Long): Certificate {
        val user = userRepository.findById(userId).orElseThrow()
        val course = courseRepository.findById(courseId).orElseThrow()
        val enrollment = enrollmentRepository.findByUserIdAndCourseId(userId, courseId)!!

        val certificateId = UUID.randomUUID().toString()
        val pdfUrl = pdfGenerator.generate(
            certificateId = certificateId,
            studentName = user.displayName,
            courseName = course.title,
            instructorName = course.instructor.displayName,
            completedAt = enrollment.completedAt ?: Instant.now()
        )

        return certificateRepository.save(
            Certificate(
                certificateId = certificateId,
                userId = userId,
                courseId = courseId,
                pdfUrl = pdfUrl,
                issuedAt = Instant.now()
            )
        )
    }

    fun verifyCertificate(certificateId: String): CertificateVerification {
        val certificate = certificateRepository.findByCertificateId(certificateId)
            ?: return CertificateVerification(valid = false)

        return CertificateVerification(
            valid = true,
            studentName = certificate.user.displayName,
            courseName = certificate.course.title,
            issuedAt = certificate.issuedAt
        )
    }
}
```

---

## 🌐 6. API Endpoints

```
# Courses
GET    /api/v1/courses                      # list/search courses
GET    /api/v1/courses/{id}                 # course details
POST   /api/v1/courses                      # create course (instructor)
PUT    /api/v1/courses/{id}                 # update course
POST   /api/v1/courses/{id}/publish         # publish course

# Sections & Lessons
POST   /api/v1/courses/{id}/sections        # add section
POST   /api/v1/sections/{id}/lessons        # add lesson
PUT    /api/v1/lessons/{id}                 # update lesson

# Enrollment
POST   /api/v1/courses/{id}/enroll          # enroll in course
GET    /api/v1/my/enrollments               # my enrolled courses

# Progress
POST   /api/v1/lessons/{id}/complete        # mark lesson complete
POST   /api/v1/lessons/{id}/progress        # update video progress

# Quiz
GET    /api/v1/quizzes/{id}                 # get quiz (no answers)
POST   /api/v1/quizzes/{id}/submit          # submit quiz answers
GET    /api/v1/quizzes/{id}/attempts        # my quiz attempts

# Reviews
POST   /api/v1/courses/{id}/reviews         # add review
GET    /api/v1/courses/{id}/reviews         # list reviews

# Certificates
GET    /api/v1/certificates/{id}            # verify certificate
GET    /api/v1/my/certificates              # my certificates
```

---

## 📋 สรุป

| Feature | Key Design Decision |
|---------|-------------------|
| Course Structure | Course → Sections → Lessons (hierarchical) |
| Progress Tracking | Per-lesson progress + course aggregate |
| Quiz | Auto-grading + multiple attempts |
| Certificate | UUID-based verification link |
| Payments | Enum-based enrollment status |
| Search | Category + tags + full-text |

---

*Part 92/100+ | Kotlin & Spring Boot Complete Course*
