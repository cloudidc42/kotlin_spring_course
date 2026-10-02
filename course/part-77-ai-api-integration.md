# Part 77: AI API Integration

## AI API Integration — Claude, OpenAI, Spring AI และ RAG

---

## 🎯 เป้าหมายของ Part นี้

- Anthropic Claude API
- OpenAI API integration
- Spring AI framework
- RAG (Retrieval Augmented Generation)
- สร้าง AI-powered customer support

---

## 📖 1. Spring AI Framework

Spring AI เป็น abstraction layer สำหรับ AI APIs ต่างๆ ช่วยให้ switch ระหว่าง providers ได้ง่าย

### Dependencies

```kotlin
// build.gradle.kts
dependencyManagement {
    imports {
        mavenBom("org.springframework.ai:spring-ai-bom:1.0.0")
    }
}

dependencies {
    // Anthropic Claude
    implementation("org.springframework.ai:spring-ai-anthropic-spring-boot-starter")
    
    // OpenAI
    implementation("org.springframework.ai:spring-ai-openai-spring-boot-starter")
    
    // Vector Store (pgvector)
    implementation("org.springframework.ai:spring-ai-pgvector-store-spring-boot-starter")
    
    // PDF document reader
    implementation("org.springframework.ai:spring-ai-pdf-document-reader")
}
```

### application.yml

```yaml
spring:
  ai:
    anthropic:
      api-key: ${ANTHROPIC_API_KEY}
      chat:
        options:
          model: claude-opus-4-5
          max-tokens: 4096
          temperature: 0.7
    openai:
      api-key: ${OPENAI_API_KEY}
      chat:
        options:
          model: gpt-4o
          temperature: 0.7
    vectorstore:
      pgvector:
        initialize-schema: true
        dimensions: 1536
```

---

## 🤖 2. Anthropic Claude API

### Direct API Call

```kotlin
import org.springframework.ai.anthropic.AnthropicChatModel
import org.springframework.ai.chat.messages.UserMessage
import org.springframework.ai.chat.messages.SystemMessage
import org.springframework.ai.chat.prompt.Prompt
import org.springframework.stereotype.Service

@Service
class ClaudeService(private val anthropicChatModel: AnthropicChatModel) {

    fun chat(userMessage: String): String {
        val prompt = Prompt(
            listOf(
                SystemMessage("คุณเป็น AI ผู้ช่วยที่เป็นมิตรและช่วยเหลือลูกค้า ตอบเป็นภาษาไทย"),
                UserMessage(userMessage)
            )
        )
        return anthropicChatModel.call(prompt).result.output.content
    }

    fun streamChat(userMessage: String): Flux<String> {
        val prompt = Prompt(UserMessage(userMessage))
        return anthropicChatModel.stream(prompt)
            .mapNotNull { it.result?.output?.content }
            .filter { it.isNotEmpty() }
    }
}
```

### Structured Output

```kotlin
import org.springframework.ai.converter.BeanOutputConverter
import org.springframework.ai.chat.prompt.PromptTemplate

data class ProductAnalysis(
    val sentiment: String,
    val keyFeatures: List<String>,
    val targetAudience: String,
    val priceRange: String,
    val competitiveAdvantages: List<String>
)

@Service
class ProductAnalysisService(private val chatModel: ChatModel) {

    fun analyzeProduct(productDescription: String): ProductAnalysis {
        val outputConverter = BeanOutputConverter(ProductAnalysis::class.java)

        val template = PromptTemplate(
            """
            วิเคราะห์ผลิตภัณฑ์ต่อไปนี้และให้ข้อมูลตาม format ที่กำหนด:
            
            คำอธิบายผลิตภัณฑ์: {description}
            
            {format}
            """.trimIndent()
        )

        val prompt = template.create(
            mapOf(
                "description" to productDescription,
                "format" to outputConverter.getFormat()
            )
        )

        val response = chatModel.call(prompt)
        return outputConverter.convert(response.result.output.content)!!
    }
}
```

---

## 🔗 3. RAG (Retrieval Augmented Generation)

RAG เป็นเทคนิคที่ดึงข้อมูลจาก knowledge base มาประกอบกับ prompt เพื่อให้ AI ตอบได้แม่นยำขึ้น

```
User Question
      ↓
Query Embedding
      ↓
Vector Similarity Search
      ↓
Retrieve Relevant Documents
      ↓
Build Context Prompt
      ↓
LLM Response
      ↓
Answer with Citations
```

### Document Ingestion

```kotlin
import org.springframework.ai.document.Document
import org.springframework.ai.embedding.EmbeddingModel
import org.springframework.ai.vectorstore.VectorStore
import org.springframework.ai.reader.pdf.PagePdfDocumentReader
import org.springframework.stereotype.Service

@Service
class DocumentIngestionService(
    private val vectorStore: VectorStore,
    private val embeddingModel: EmbeddingModel
) {

    fun ingestPdfDocument(filePath: String, metadata: Map<String, Any>) {
        val reader = PagePdfDocumentReader(filePath)
        val documents = reader.get()

        // เพิ่ม metadata
        val enrichedDocs = documents.map { doc ->
            Document(
                doc.content,
                metadata + mapOf(
                    "source" to filePath,
                    "ingested_at" to System.currentTimeMillis()
                )
            )
        }

        vectorStore.add(enrichedDocs)
        println("Ingested ${enrichedDocs.size} documents from $filePath")
    }

    fun ingestFAQs(faqs: List<Pair<String, String>>) {
        val documents = faqs.map { (question, answer) ->
            Document(
                "Q: $question\nA: $answer",
                mapOf(
                    "type" to "faq",
                    "question" to question
                )
            )
        }
        vectorStore.add(documents)
    }

    fun ingestProductCatalog(products: List<Product>) {
        val documents = products.map { product ->
            val content = """
                ชื่อสินค้า: ${product.name}
                หมวดหมู่: ${product.category}
                ราคา: ${product.price} บาท
                คำอธิบาย: ${product.description}
                คุณสมบัติ: ${product.features.joinToString(", ")}
            """.trimIndent()

            Document(content, mapOf(
                "product_id" to product.id,
                "type" to "product"
            ))
        }
        vectorStore.add(documents)
    }
}
```

### RAG Query Service

```kotlin
import org.springframework.ai.vectorstore.SearchRequest
import org.springframework.ai.chat.prompt.PromptTemplate

@Service
class RAGQueryService(
    private val vectorStore: VectorStore,
    private val chatModel: ChatModel
) {

    fun query(userQuestion: String, topK: Int = 5): RAGResponse {
        // 1. ค้นหา relevant documents
        val searchRequest = SearchRequest.query(userQuestion)
            .withTopK(topK)
            .withSimilarityThreshold(0.7)

        val relevantDocs = vectorStore.similaritySearch(searchRequest)

        if (relevantDocs.isEmpty()) {
            return RAGResponse(
                answer = "ขออภัย ไม่พบข้อมูลที่เกี่ยวข้องกับคำถามของคุณ",
                sources = emptyList(),
                confidence = 0.0
            )
        }

        // 2. สร้าง context จาก documents ที่พบ
        val context = relevantDocs.joinToString("\n\n---\n\n") { doc ->
            doc.content
        }

        // 3. สร้าง prompt พร้อม context
        val promptTemplate = PromptTemplate(
            """
            คุณเป็น AI ผู้ช่วยสำหรับลูกค้า ใช้ข้อมูลต่อไปนี้เพื่อตอบคำถาม:
            
            ข้อมูลอ้างอิง:
            {context}
            
            คำถามของลูกค้า: {question}
            
            กรุณาตอบโดยอ้างอิงจากข้อมูลที่ให้มาเท่านั้น ถ้าไม่มีข้อมูลที่เกี่ยวข้อง ให้บอกว่าไม่ทราบ
            """.trimIndent()
        )

        val prompt = promptTemplate.create(
            mapOf(
                "context" to context,
                "question" to userQuestion
            )
        )

        // 4. รับคำตอบจาก LLM
        val response = chatModel.call(prompt)
        val answer = response.result.output.content

        return RAGResponse(
            answer = answer,
            sources = relevantDocs.map { doc ->
                DocumentSource(
                    content = doc.content.take(200) + "...",
                    metadata = doc.metadata,
                    score = doc.metadata["distance"] as? Double ?: 0.0
                )
            },
            confidence = relevantDocs.firstOrNull()?.metadata?.get("distance") as? Double ?: 0.0
        )
    }
}

data class RAGResponse(
    val answer: String,
    val sources: List<DocumentSource>,
    val confidence: Double
)

data class DocumentSource(
    val content: String,
    val metadata: Map<String, Any>,
    val score: Double
)
```

---

## 💬 4. AI Customer Support System

```kotlin
@Service
class CustomerSupportService(
    private val ragQueryService: RAGQueryService,
    private val chatModel: ChatModel,
    private val conversationRepository: ConversationRepository
) {

    fun handleCustomerMessage(
        sessionId: String,
        userId: Long?,
        message: String
    ): SupportResponse {
        val conversation = conversationRepository.findBySessionId(sessionId)
            ?: createNewConversation(sessionId, userId)

        // ตรวจสอบ intent
        val intent = detectIntent(message)

        val response = when (intent) {
            SupportIntent.ORDER_STATUS -> handleOrderStatusQuery(userId, message)
            SupportIntent.PRODUCT_INQUIRY -> handleProductInquiry(message)
            SupportIntent.COMPLAINT -> handleComplaint(userId, message, conversation)
            SupportIntent.GENERAL -> handleGeneralQuery(message, conversation)
        }

        // บันทึก conversation
        conversation.messages.add(
            ConversationMessage(
                role = "user",
                content = message,
                timestamp = java.time.LocalDateTime.now()
            )
        )
        conversation.messages.add(
            ConversationMessage(
                role = "assistant",
                content = response.message,
                timestamp = java.time.LocalDateTime.now()
            )
        )
        conversationRepository.save(conversation)

        return response
    }

    private fun detectIntent(message: String): SupportIntent {
        val prompt = """
            จัดประเภทข้อความต่อไปนี้เป็นหนึ่งในประเภท: ORDER_STATUS, PRODUCT_INQUIRY, COMPLAINT, GENERAL
            ข้อความ: "$message"
            ตอบด้วยประเภทเดียวเท่านั้น ไม่มีคำอธิบายเพิ่มเติม
        """.trimIndent()

        val response = chatModel.call(prompt).trim().uppercase()
        return try {
            SupportIntent.valueOf(response)
        } catch (e: Exception) {
            SupportIntent.GENERAL
        }
    }

    private fun handleGeneralQuery(
        message: String,
        conversation: Conversation
    ): SupportResponse {
        val ragResponse = ragQueryService.query(message)
        return SupportResponse(
            message = ragResponse.answer,
            intent = SupportIntent.GENERAL,
            requiresHumanAgent = ragResponse.confidence < 0.5,
            sources = ragResponse.sources.map { it.content }
        )
    }
}

enum class SupportIntent {
    ORDER_STATUS, PRODUCT_INQUIRY, COMPLAINT, GENERAL
}

data class SupportResponse(
    val message: String,
    val intent: SupportIntent,
    val requiresHumanAgent: Boolean = false,
    val sources: List<String> = emptyList(),
    val suggestedActions: List<String> = emptyList()
)
```

---

## 🌐 5. REST API Controller

```kotlin
@RestController
@RequestMapping("/api/v1/support")
class CustomerSupportController(
    private val customerSupportService: CustomerSupportService,
    private val documentIngestionService: DocumentIngestionService
) {

    @PostMapping("/chat")
    fun chat(
        @RequestBody request: ChatRequest,
        @AuthenticationPrincipal user: UserDetails?
    ): ResponseEntity<SupportResponse> {
        val sessionId = request.sessionId ?: java.util.UUID.randomUUID().toString()
        val userId = (user as? AppUserDetails)?.userId

        val response = customerSupportService.handleCustomerMessage(
            sessionId = sessionId,
            userId = userId,
            message = request.message
        )

        return ResponseEntity.ok(response)
    }

    @PostMapping("/ingest/faq")
    @PreAuthorize("hasRole('ADMIN')")
    fun ingestFAQs(@RequestBody faqs: List<FAQRequest>): ResponseEntity<String> {
        documentIngestionService.ingestFAQs(faqs.map { it.question to it.answer })
        return ResponseEntity.ok("Ingested ${faqs.size} FAQs")
    }

    @GetMapping("/chat/stream")
    fun streamChat(
        @RequestParam message: String,
        @RequestParam(defaultValue = "default") sessionId: String
    ): Flux<ServerSentEvent<String>> {
        return customerSupportService.streamResponse(message, sessionId)
            .map { chunk ->
                ServerSentEvent.builder(chunk)
                    .event("message")
                    .build()
            }
    }
}

data class ChatRequest(
    val message: String,
    val sessionId: String? = null
)

data class FAQRequest(
    val question: String,
    val answer: String
)
```

---

## 📊 6. สรุปตาราง AI APIs

| Provider | Models | ข้อดี | ข้อเสีย |
|---------|--------|-------|---------|
| Anthropic Claude | claude-opus-4-5, claude-sonnet-4-5 | Reasoning, Safety | ราคาสูงกว่า |
| OpenAI | gpt-4o, gpt-4o-mini | Function calling | Context window จำกัด |
| Google Gemini | gemini-1.5-pro | Multimodal | API ใหม่กว่า |
| Local (Ollama) | llama3, mistral | Privacy, ฟรี | เร็วกว่า cloud น้อย |

---

## 💡 Best Practices

1. **Rate limiting** สำหรับ AI API calls — ป้องกัน cost overrun
2. **Prompt caching** (Anthropic) ลดค่าใช้จ่ายสำหรับ long system prompts
3. **Semantic cache** — cache responses สำหรับ similar questions
4. **Human-in-the-loop** สำหรับ low confidence responses
5. **Audit logging** — บันทึกทุก AI interaction เพื่อ compliance

---

## 📊 8. AI API Cost Tracking

```kotlin
@Service
class AIUsageTracker(
    private val meterRegistry: MeterRegistry,
    private val usageRepository: AIUsageRepository
) {

    fun trackUsage(
        provider: String,
        model: String,
        inputTokens: Int,
        outputTokens: Int,
        tenantId: String
    ) {
        val inputCost = calculateCost(provider, model, "input", inputTokens)
        val outputCost = calculateCost(provider, model, "output", outputTokens)

        meterRegistry.counter(
            "ai.tokens.used",
            "provider", provider,
            "model", model,
            "type", "input"
        ).increment(inputTokens.toDouble())

        meterRegistry.counter(
            "ai.cost.usd",
            "provider", provider,
            "model", model
        ).increment(inputCost + outputCost)

        usageRepository.save(
            AIUsage(
                provider = provider,
                model = model,
                inputTokens = inputTokens,
                outputTokens = outputTokens,
                costUsd = inputCost + outputCost,
                tenantId = tenantId,
                timestamp = java.time.Instant.now()
            )
        )
    }

    private fun calculateCost(provider: String, model: String, type: String, tokens: Int): Double {
        // ราคาต่อ 1M tokens (ตาม pricing ปัจจุบัน)
        val pricePerMillion = when ("$provider:$model:$type") {
            "anthropic:claude-opus-4-5:input" -> 15.0
            "anthropic:claude-opus-4-5:output" -> 75.0
            "anthropic:claude-sonnet-4-5:input" -> 3.0
            "anthropic:claude-sonnet-4-5:output" -> 15.0
            "openai:gpt-4o:input" -> 5.0
            "openai:gpt-4o:output" -> 15.0
            else -> 1.0
        }
        return tokens * pricePerMillion / 1_000_000
    }
}
```

---

## 🔄 9. Prompt Template Management

```kotlin
@Entity
@Table(name = "prompt_templates")
data class PromptTemplate(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long = 0,
    val name: String,
    val version: String,
    val systemPrompt: String,
    val userPromptTemplate: String,
    val isActive: Boolean = true,
    val variables: List<String> = emptyList()
)

@Service
class PromptTemplateService(
    private val templateRepository: PromptTemplateRepository
) {

    fun renderTemplate(templateName: String, variables: Map<String, String>): String {
        val template = templateRepository.findActiveByName(templateName)
            ?: throw IllegalArgumentException("Template not found: $templateName")

        var rendered = template.userPromptTemplate
        variables.forEach { (key, value) ->
            rendered = rendered.replace("{{$key}}", value)
        }
        return rendered
    }
}
```

---

*Part 77/100+ | Kotlin & Spring Boot Complete Course*
