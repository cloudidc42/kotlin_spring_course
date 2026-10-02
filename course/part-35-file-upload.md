# Part 35: File Upload และ Storage
## จัดการไฟล์ใน Spring Boot

---

## 🎯 เป้าหมายของ Part นี้

- MultipartFile handling ใน Spring Boot
- Local file storage
- AWS S3 integration
- File validation (type, size)
- Image resizing (optional)
- ตัวอย่าง: Profile photo upload

---

## 📁 1. พื้นฐาน File Upload

Spring Boot รองรับ file upload ผ่าน `MultipartFile` interface ซึ่งรองรับทั้ง single และ multiple files

### Flow ของ File Upload
```
Client → HTTP POST (multipart/form-data) → Controller → Service → Storage
                                                                ↓
                                                    Local/S3/Cloud Storage
```

---

## ⚙️ 2. Configuration

```yaml
# application.yml
spring:
  servlet:
    multipart:
      enabled: true
      max-file-size: 10MB        # ขนาดไฟล์สูงสุด
      max-request-size: 20MB     # ขนาด request สูงสุด
      file-size-threshold: 2KB   # ขนาดที่จะ buffer ใน memory

app:
  upload:
    local:
      base-path: /uploads
      url-prefix: /files
    allowed-types:
      - image/jpeg
      - image/png
      - image/gif
      - image/webp
      - application/pdf
    max-file-size: 10485760  # 10 MB in bytes

aws:
  s3:
    bucket-name: ${AWS_S3_BUCKET:my-app-bucket}
    region: ${AWS_REGION:ap-southeast-1}
    access-key: ${AWS_ACCESS_KEY:}
    secret-key: ${AWS_SECRET_KEY:}
    cdn-url: ${AWS_CDN_URL:}
```

---

## 📦 3. Dependencies

```kotlin
// build.gradle.kts
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-web")

    // AWS S3
    implementation("software.amazon.awssdk:s3:2.21.0")
    implementation("software.amazon.awssdk:s3-transfer-manager:2.21.0")

    // Image processing (optional)
    implementation("net.coobird:thumbnailator:0.4.20")

    // File type detection
    implementation("org.apache.tika:tika-core:2.9.0")

    // Testing
    testImplementation("org.springframework.boot:spring-boot-test-autoconfigure")
    testImplementation("org.testcontainers:localstack:1.19.0")
}
```

---

## 🏗️ 4. File Storage Interface

```kotlin
// src/main/kotlin/com/example/storage/FileStorage.kt
package com.example.storage

import org.springframework.web.multipart.MultipartFile

interface FileStorage {
    fun store(file: MultipartFile, directory: String = ""): StoredFile
    fun load(filename: String): ByteArray
    fun delete(filename: String): Boolean
    fun exists(filename: String): Boolean
    fun getUrl(filename: String): String
}

data class StoredFile(
    val filename: String,       // ชื่อไฟล์ที่เก็บ (อาจ rename)
    val originalFilename: String,  // ชื่อไฟล์ต้นฉบับ
    val contentType: String,
    val size: Long,
    val url: String,
    val path: String            // path ใน storage
)
```

---

## 💾 5. Local File Storage

```kotlin
// src/main/kotlin/com/example/storage/LocalFileStorage.kt
package com.example.storage

import org.springframework.beans.factory.annotation.Value
import org.springframework.core.io.Resource
import org.springframework.core.io.UrlResource
import org.springframework.stereotype.Service
import org.springframework.web.multipart.MultipartFile
import java.nio.file.Files
import java.nio.file.Path
import java.nio.file.Paths
import java.nio.file.StandardCopyOption
import java.util.UUID

@Service("localFileStorage")
class LocalFileStorage(
    @Value("\${app.upload.local.base-path:/uploads}") private val basePath: String,
    @Value("\${app.upload.local.url-prefix:/files}") private val urlPrefix: String
) : FileStorage {

    init {
        // สร้าง directory ถ้ายังไม่มี
        Files.createDirectories(Paths.get(basePath))
    }

    override fun store(file: MultipartFile, directory: String): StoredFile {
        val originalFilename = file.originalFilename
            ?: throw IllegalArgumentException("File must have a name")

        // สร้างชื่อไฟล์ unique ป้องกัน overwrite
        val extension = originalFilename.substringAfterLast('.', "")
        val storedFilename = "${UUID.randomUUID()}.${extension}"

        // สร้าง subdirectory ถ้าระบุ
        val targetDirectory = if (directory.isNotEmpty()) {
            Paths.get(basePath, directory).also { Files.createDirectories(it) }
        } else {
            Paths.get(basePath)
        }

        val targetPath = targetDirectory.resolve(storedFilename)

        // บันทึกไฟล์
        Files.copy(file.inputStream, targetPath, StandardCopyOption.REPLACE_EXISTING)

        val relativePath = if (directory.isNotEmpty()) "$directory/$storedFilename" else storedFilename

        return StoredFile(
            filename = storedFilename,
            originalFilename = originalFilename,
            contentType = file.contentType ?: "application/octet-stream",
            size = file.size,
            url = "$urlPrefix/$relativePath",
            path = relativePath
        )
    }

    override fun load(filename: String): ByteArray {
        val path = Paths.get(basePath, filename)
        return Files.readAllBytes(path)
    }

    fun loadAsResource(filename: String): Resource {
        val path = Paths.get(basePath, filename)
        return UrlResource(path.toUri())
    }

    override fun delete(filename: String): Boolean {
        val path = Paths.get(basePath, filename)
        return Files.deleteIfExists(path)
    }

    override fun exists(filename: String): Boolean {
        val path = Paths.get(basePath, filename)
        return Files.exists(path)
    }

    override fun getUrl(filename: String): String {
        return "$urlPrefix/$filename"
    }
}
```

---

## ☁️ 6. AWS S3 Storage

```kotlin
// src/main/kotlin/com/example/config/S3Config.kt
package com.example.config

import org.springframework.beans.factory.annotation.Value
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import software.amazon.awssdk.auth.credentials.AwsBasicCredentials
import software.amazon.awssdk.auth.credentials.StaticCredentialsProvider
import software.amazon.awssdk.regions.Region
import software.amazon.awssdk.services.s3.S3Client
import software.amazon.awssdk.services.s3.presigner.S3Presigner

@Configuration
class S3Config(
    @Value("\${aws.s3.region}") private val region: String,
    @Value("\${aws.s3.access-key:}") private val accessKey: String,
    @Value("\${aws.s3.secret-key:}") private val secretKey: String
) {

    @Bean
    fun s3Client(): S3Client {
        return if (accessKey.isNotEmpty() && secretKey.isNotEmpty()) {
            // ใช้ credentials ที่ระบุ
            S3Client.builder()
                .region(Region.of(region))
                .credentialsProvider(
                    StaticCredentialsProvider.create(
                        AwsBasicCredentials.create(accessKey, secretKey)
                    )
                )
                .build()
        } else {
            // ใช้ default credentials chain (IAM role, env vars, etc.)
            S3Client.builder()
                .region(Region.of(region))
                .build()
        }
    }

    @Bean
    fun s3Presigner(): S3Presigner {
        return S3Presigner.builder()
            .region(Region.of(region))
            .build()
    }
}
```

```kotlin
// src/main/kotlin/com/example/storage/S3FileStorage.kt
package com.example.storage

import org.springframework.beans.factory.annotation.Value
import org.springframework.context.annotation.Primary
import org.springframework.stereotype.Service
import org.springframework.web.multipart.MultipartFile
import software.amazon.awssdk.core.sync.RequestBody
import software.amazon.awssdk.services.s3.S3Client
import software.amazon.awssdk.services.s3.model.*
import software.amazon.awssdk.services.s3.presigner.S3Presigner
import software.amazon.awssdk.services.s3.presigner.model.GetObjectPresignRequest
import java.time.Duration
import java.util.UUID

@Primary  // ใช้ S3 เป็น default เมื่อมี @Autowired FileStorage
@Service("s3FileStorage")
class S3FileStorage(
    private val s3Client: S3Client,
    private val s3Presigner: S3Presigner,
    @Value("\${aws.s3.bucket-name}") private val bucketName: String,
    @Value("\${aws.s3.cdn-url:}") private val cdnUrl: String
) : FileStorage {

    override fun store(file: MultipartFile, directory: String): StoredFile {
        val originalFilename = file.originalFilename
            ?: throw IllegalArgumentException("File must have a name")

        val extension = originalFilename.substringAfterLast('.', "")
        val storedFilename = "${UUID.randomUUID()}.${extension}"
        val s3Key = if (directory.isNotEmpty()) "$directory/$storedFilename" else storedFilename

        val contentType = file.contentType ?: "application/octet-stream"

        // Upload ไปยัง S3
        val putRequest = PutObjectRequest.builder()
            .bucket(bucketName)
            .key(s3Key)
            .contentType(contentType)
            .contentLength(file.size)
            .build()

        s3Client.putObject(putRequest, RequestBody.fromInputStream(file.inputStream, file.size))

        return StoredFile(
            filename = storedFilename,
            originalFilename = originalFilename,
            contentType = contentType,
            size = file.size,
            url = getUrl(s3Key),
            path = s3Key
        )
    }

    override fun load(filename: String): ByteArray {
        val request = GetObjectRequest.builder()
            .bucket(bucketName)
            .key(filename)
            .build()

        return s3Client.getObjectAsBytes(request).asByteArray()
    }

    override fun delete(filename: String): Boolean {
        return try {
            val request = DeleteObjectRequest.builder()
                .bucket(bucketName)
                .key(filename)
                .build()
            s3Client.deleteObject(request)
            true
        } catch (e: Exception) {
            false
        }
    }

    override fun exists(filename: String): Boolean {
        return try {
            val request = HeadObjectRequest.builder()
                .bucket(bucketName)
                .key(filename)
                .build()
            s3Client.headObject(request)
            true
        } catch (e: NoSuchKeyException) {
            false
        }
    }

    override fun getUrl(filename: String): String {
        return if (cdnUrl.isNotEmpty()) {
            "$cdnUrl/$filename"
        } else {
            "https://$bucketName.s3.amazonaws.com/$filename"
        }
    }

    // สร้าง presigned URL (เพื่อ download แบบ private)
    fun generatePresignedUrl(filename: String, expiryMinutes: Long = 60): String {
        val presignRequest = GetObjectPresignRequest.builder()
            .signatureDuration(Duration.ofMinutes(expiryMinutes))
            .getObjectRequest { req ->
                req.bucket(bucketName).key(filename)
            }
            .build()

        return s3Presigner.presignGetObject(presignRequest).url().toString()
    }

    // สร้าง presigned URL สำหรับ upload จาก client โดยตรง
    fun generatePresignedUploadUrl(filename: String, contentType: String): String {
        val presignRequest = software.amazon.awssdk.services.s3.presigner.model.PutObjectPresignRequest.builder()
            .signatureDuration(Duration.ofMinutes(10))
            .putObjectRequest { req ->
                req.bucket(bucketName).key(filename).contentType(contentType)
            }
            .build()

        return s3Presigner.presignPutObject(presignRequest).url().toString()
    }
}
```

---

## ✅ 7. File Validation

```kotlin
// src/main/kotlin/com/example/validation/FileValidator.kt
package com.example.validation

import org.apache.tika.Tika
import org.springframework.beans.factory.annotation.Value
import org.springframework.stereotype.Component
import org.springframework.web.multipart.MultipartFile

@Component
class FileValidator(
    @Value("\${app.upload.max-file-size:10485760}") private val maxFileSize: Long,
    @Value("\${app.upload.allowed-types}") private val allowedTypes: List<String>
) {

    private val tika = Tika()

    data class ValidationResult(
        val isValid: Boolean,
        val errors: List<String> = emptyList()
    )

    fun validate(file: MultipartFile): ValidationResult {
        val errors = mutableListOf<String>()

        // ตรวจสอบว่าไฟล์ว่างเปล่าหรือไม่
        if (file.isEmpty) {
            errors.add("File is empty")
            return ValidationResult(false, errors)
        }

        // ตรวจสอบขนาดไฟล์
        if (file.size > maxFileSize) {
            val maxSizeMb = maxFileSize / 1024 / 1024
            errors.add("File size exceeds maximum allowed size of ${maxSizeMb}MB")
        }

        // ตรวจสอบ content type ด้วย Apache Tika (ปลอดภัยกว่าดูจาก extension)
        val detectedType = tika.detect(file.bytes)
        if (detectedType !in allowedTypes) {
            errors.add("File type '$detectedType' is not allowed. Allowed types: ${allowedTypes.joinToString(", ")}")
        }

        // ตรวจสอบชื่อไฟล์ (ป้องกัน path traversal)
        val filename = file.originalFilename ?: ""
        if (filename.contains("..") || filename.contains("/") || filename.contains("\\")) {
            errors.add("Invalid filename")
        }

        return ValidationResult(errors.isEmpty(), errors)
    }

    fun validateImage(file: MultipartFile): ValidationResult {
        val imageTypes = listOf("image/jpeg", "image/png", "image/gif", "image/webp")
        val validationResult = validate(file)

        if (!validationResult.isValid) return validationResult

        val detectedType = tika.detect(file.bytes)
        if (detectedType !in imageTypes) {
            return ValidationResult(false, listOf("Only image files are allowed"))
        }

        return ValidationResult(true)
    }
}
```

---

## 👤 8. Profile Photo Upload Service

```kotlin
// src/main/kotlin/com/example/service/ProfilePhotoService.kt
package com.example.service

import com.example.dto.PhotoUploadResult
import com.example.entity.UserProfile
import com.example.repository.UserProfileRepository
import com.example.storage.FileStorage
import com.example.validation.FileValidator
import net.coobird.thumbnailator.Thumbnails
import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional
import org.springframework.web.multipart.MultipartFile
import java.io.ByteArrayInputStream
import java.io.ByteArrayOutputStream
import javax.imageio.ImageIO

@Service
class ProfilePhotoService(
    private val fileStorage: FileStorage,
    private val fileValidator: FileValidator,
    private val userProfileRepository: UserProfileRepository
) {

    companion object {
        const val PHOTO_DIRECTORY = "profiles"
        const val THUMBNAIL_WIDTH = 150
        const val THUMBNAIL_HEIGHT = 150
        const val MAX_IMAGE_WIDTH = 800
        const val MAX_IMAGE_HEIGHT = 800
    }

    @Transactional
    fun uploadProfilePhoto(userId: Long, file: MultipartFile): PhotoUploadResult {
        // 1. Validate file
        val validation = fileValidator.validateImage(file)
        if (!validation.isValid) {
            throw InvalidFileException(validation.errors.joinToString(", "))
        }

        // 2. ลบรูปเก่า (ถ้ามี)
        val existingProfile = userProfileRepository.findByUserId(userId)
        existingProfile?.photoPath?.let { oldPath ->
            fileStorage.delete(oldPath)
            existingProfile.thumbnailPath?.let { fileStorage.delete(it) }
        }

        // 3. Resize รูปก่อน upload
        val resizedImageBytes = resizeImage(
            file.bytes,
            file.contentType ?: "image/jpeg",
            MAX_IMAGE_WIDTH,
            MAX_IMAGE_HEIGHT
        )

        // 4. สร้าง thumbnail
        val thumbnailBytes = createThumbnail(file.bytes, file.contentType ?: "image/jpeg")

        // 5. สร้าง MultipartFile จาก resized bytes
        val resizedFile = createMultipartFile(
            resizedImageBytes,
            file.originalFilename ?: "photo.jpg",
            file.contentType ?: "image/jpeg"
        )
        val thumbnailFile = createMultipartFile(
            thumbnailBytes,
            "thumb_${file.originalFilename ?: "photo.jpg"}",
            file.contentType ?: "image/jpeg"
        )

        // 6. Upload ไป storage
        val storedPhoto = fileStorage.store(resizedFile, PHOTO_DIRECTORY)
        val storedThumbnail = fileStorage.store(thumbnailFile, "$PHOTO_DIRECTORY/thumbnails")

        // 7. บันทึก path ลง DB
        val profile = existingProfile ?: UserProfile(userId = userId)
        profile.apply {
            photoPath = storedPhoto.path
            photoUrl = storedPhoto.url
            thumbnailPath = storedThumbnail.path
            thumbnailUrl = storedThumbnail.url
        }
        userProfileRepository.save(profile)

        return PhotoUploadResult(
            photoUrl = storedPhoto.url,
            thumbnailUrl = storedThumbnail.url,
            size = resizedImageBytes.size.toLong(),
            message = "Profile photo uploaded successfully"
        )
    }

    fun deleteProfilePhoto(userId: Long) {
        val profile = userProfileRepository.findByUserId(userId)
            ?: throw NotFoundException("Profile not found for user $userId")

        profile.photoPath?.let { fileStorage.delete(it) }
        profile.thumbnailPath?.let { fileStorage.delete(it) }

        profile.photoPath = null
        profile.photoUrl = null
        profile.thumbnailPath = null
        profile.thumbnailUrl = null
        userProfileRepository.save(profile)
    }

    private fun resizeImage(
        imageBytes: ByteArray,
        contentType: String,
        maxWidth: Int,
        maxHeight: Int
    ): ByteArray {
        val inputStream = ByteArrayInputStream(imageBytes)
        val outputStream = ByteArrayOutputStream()
        val format = contentType.substringAfter("/")

        Thumbnails.of(inputStream)
            .size(maxWidth, maxHeight)
            .keepAspectRatio(true)
            .outputFormat(format)
            .toOutputStream(outputStream)

        return outputStream.toByteArray()
    }

    private fun createThumbnail(imageBytes: ByteArray, contentType: String): ByteArray {
        val inputStream = ByteArrayInputStream(imageBytes)
        val outputStream = ByteArrayOutputStream()
        val format = contentType.substringAfter("/")

        Thumbnails.of(inputStream)
            .size(THUMBNAIL_WIDTH, THUMBNAIL_HEIGHT)
            .keepAspectRatio(false)  // crop to exact size
            .crop(net.coobird.thumbnailator.geometry.Positions.CENTER)
            .outputFormat(format)
            .toOutputStream(outputStream)

        return outputStream.toByteArray()
    }

    private fun createMultipartFile(
        bytes: ByteArray,
        filename: String,
        contentType: String
    ): MultipartFile {
        return object : MultipartFile {
            override fun getName() = "file"
            override fun getOriginalFilename() = filename
            override fun getContentType() = contentType
            override fun isEmpty() = bytes.isEmpty()
            override fun getSize() = bytes.size.toLong()
            override fun getBytes() = bytes
            override fun getInputStream() = ByteArrayInputStream(bytes)
            override fun transferTo(dest: java.io.File) = dest.writeBytes(bytes)
        }
    }
}
```

---

## 🌐 9. File Upload Controller

```kotlin
// src/main/kotlin/com/example/controller/FileController.kt
package com.example.controller

import com.example.dto.PhotoUploadResult
import com.example.service.ProfilePhotoService
import com.example.storage.FileStorage
import com.example.storage.LocalFileStorage
import org.springframework.core.io.Resource
import org.springframework.http.HttpHeaders
import org.springframework.http.MediaType
import org.springframework.http.ResponseEntity
import org.springframework.security.core.annotation.AuthenticationPrincipal
import org.springframework.web.bind.annotation.*
import org.springframework.web.multipart.MultipartFile

@RestController
@RequestMapping("/api")
class FileController(
    private val profilePhotoService: ProfilePhotoService,
    private val localFileStorage: LocalFileStorage
) {

    // Upload profile photo
    @PostMapping(
        path = ["/users/me/photo"],
        consumes = [MediaType.MULTIPART_FORM_DATA_VALUE]
    )
    fun uploadProfilePhoto(
        @RequestParam("file") file: MultipartFile,
        @AuthenticationPrincipal userId: Long
    ): ResponseEntity<PhotoUploadResult> {
        val result = profilePhotoService.uploadProfilePhoto(userId, file)
        return ResponseEntity.ok(result)
    }

    // Upload หลายไฟล์พร้อมกัน
    @PostMapping(
        path = ["/upload/multiple"],
        consumes = [MediaType.MULTIPART_FORM_DATA_VALUE]
    )
    fun uploadMultipleFiles(
        @RequestParam("files") files: Array<MultipartFile>
    ): ResponseEntity<List<Map<String, String>>> {
        val results = files.map { file ->
            val stored = localFileStorage.store(file, "uploads")
            mapOf(
                "filename" to stored.filename,
                "originalName" to stored.originalFilename,
                "url" to stored.url,
                "size" to stored.size.toString()
            )
        }
        return ResponseEntity.ok(results)
    }

    // Download file
    @GetMapping("/files/{filename:.+}")
    fun downloadFile(@PathVariable filename: String): ResponseEntity<Resource> {
        val resource = localFileStorage.loadAsResource(filename)

        if (!resource.exists()) {
            return ResponseEntity.notFound().build()
        }

        val contentType = determineContentType(filename)

        return ResponseEntity.ok()
            .header(HttpHeaders.CONTENT_DISPOSITION, "attachment; filename=\"${resource.filename}\"")
            .contentType(MediaType.parseMediaType(contentType))
            .body(resource)
    }

    // ดู/แสดงผลไฟล์ inline
    @GetMapping("/files/view/{filename:.+}")
    fun viewFile(@PathVariable filename: String): ResponseEntity<Resource> {
        val resource = localFileStorage.loadAsResource(filename)

        if (!resource.exists()) {
            return ResponseEntity.notFound().build()
        }

        val contentType = determineContentType(filename)

        return ResponseEntity.ok()
            .header(HttpHeaders.CONTENT_DISPOSITION, "inline; filename=\"${resource.filename}\"")
            .contentType(MediaType.parseMediaType(contentType))
            .body(resource)
    }

    // ลบไฟล์
    @DeleteMapping("/users/me/photo")
    fun deleteProfilePhoto(@AuthenticationPrincipal userId: Long): ResponseEntity<Map<String, String>> {
        profilePhotoService.deleteProfilePhoto(userId)
        return ResponseEntity.ok(mapOf("message" to "Photo deleted successfully"))
    }

    private fun determineContentType(filename: String): String {
        return when (filename.substringAfterLast('.').lowercase()) {
            "jpg", "jpeg" -> "image/jpeg"
            "png" -> "image/png"
            "gif" -> "image/gif"
            "pdf" -> "application/pdf"
            "mp4" -> "video/mp4"
            else -> "application/octet-stream"
        }
    }
}
```

---

## 🧪 10. Testing

```kotlin
// src/test/kotlin/com/example/controller/FileControllerTest.kt
package com.example.controller

import org.junit.jupiter.api.Test
import org.springframework.beans.factory.annotation.Autowired
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest
import org.springframework.http.MediaType
import org.springframework.mock.web.MockMultipartFile
import org.springframework.test.web.servlet.MockMvc
import org.springframework.test.web.servlet.request.MockMvcRequestBuilders.multipart
import org.springframework.test.web.servlet.result.MockMvcResultMatchers.*

@WebMvcTest(FileController::class)
class FileControllerTest {

    @Autowired
    lateinit var mockMvc: MockMvc

    @Test
    fun `should upload image successfully`() {
        val imageContent = javaClass.getResourceAsStream("/test-image.jpg")?.readBytes()
            ?: "fake-image-content".toByteArray()

        val mockFile = MockMultipartFile(
            "file",
            "test.jpg",
            MediaType.IMAGE_JPEG_VALUE,
            imageContent
        )

        mockMvc.perform(
            multipart("/api/users/me/photo")
                .file(mockFile)
        )
            .andExpect(status().isOk)
            .andExpect(jsonPath("$.photoUrl").isNotEmpty)
            .andExpect(jsonPath("$.thumbnailUrl").isNotEmpty)
    }

    @Test
    fun `should reject oversized file`() {
        val largeContent = ByteArray(11 * 1024 * 1024)  // 11 MB (เกิน limit)
        val mockFile = MockMultipartFile(
            "file",
            "large.jpg",
            MediaType.IMAGE_JPEG_VALUE,
            largeContent
        )

        mockMvc.perform(
            multipart("/api/users/me/photo")
                .file(mockFile)
        )
            .andExpect(status().isBadRequest)
    }

    @Test
    fun `should reject invalid file type`() {
        val mockFile = MockMultipartFile(
            "file",
            "script.php",
            "application/x-php",
            "<?php echo 'hack'; ?>".toByteArray()
        )

        mockMvc.perform(
            multipart("/api/users/me/photo")
                .file(mockFile)
        )
            .andExpect(status().isBadRequest)
    }
}
```

---

## 📋 สรุป

### Storage Options Comparison

| Storage | ข้อดี | ข้อเสีย | เหมาะกับ |
|---------|-------|---------|---------|
| Local | เร็ว, ง่าย, ฟรี | ไม่ scale, ไม่ redundant | Development |
| AWS S3 | Scale ได้, CDN, ถูก | ต้องมี AWS account | Production |
| Google GCS | คล้าย S3 | ต้องมี GCP account | Production |

### File Security Best Practices

| ข้อควรปฏิบัติ | เหตุผล |
|-------------|--------|
| ตรวจ content type ด้วย Apache Tika | ป้องกันการปลอม extension |
| Rename ไฟล์เป็น UUID | ป้องกัน path traversal |
| จำกัดขนาดไฟล์ | ป้องกัน DoS |
| เก็บนอก web root | ป้องกัน direct access |
| Scan virus (prod) | ป้องกัน malware |

---

*Part 35/100+ | Kotlin & Spring Boot Complete Course*
