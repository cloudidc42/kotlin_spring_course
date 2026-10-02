# Part 57: WebSocket กับ Spring Boot - Real-time Chat

## บทนำ

**WebSocket** คือ protocol ที่ให้ full-duplex communication ระหว่าง client และ server ผ่าน TCP connection เดียวกัน ต่างจาก HTTP ที่เป็น request-response pattern WebSocket ช่วยให้ server ส่งข้อมูลให้ client ได้เลยโดยไม่ต้อง client poll ตลอดเวลา

## ความแตกต่างระหว่าง HTTP และ WebSocket

```
HTTP (Polling):
Client ──request──▶ Server
Client ◀──response── Server
Client ──request──▶ Server  (ต้องถามซ้ำ)
Client ◀──response── Server

WebSocket:
Client ──upgrade──▶ Server   (handshake ครั้งเดียว)
Client ◀──message── Server   (server push ได้เลย)
Client ──message──▶ Server   (bidirectional)
Client ◀──message── Server
```

## STOMP Protocol

**STOMP (Simple Text Oriented Messaging Protocol)** คือ messaging protocol ที่ทำงานบน WebSocket ช่วยให้ใช้งาน pub/sub pattern ได้ง่ายขึ้น

```
STOMP Message Flow:
Client → SUBSCRIBE /topic/chat  (subscribe ไปยัง topic)
Client → SEND /app/chat          (ส่ง message)
Server → PUBLISH /topic/chat     (broadcast ให้ทุก subscriber)
```

## การตั้งค่าโปรเจกต์

```kotlin
// build.gradle.kts
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-websocket")
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-security")
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
    
    // JSON processing
    implementation("com.fasterxml.jackson.module:jackson-module-kotlin")
}
```

## WebSocket Configuration

```kotlin
// WebSocketConfig.kt
package com.chat.websocket.config

import org.springframework.context.annotation.Configuration
import org.springframework.messaging.simp.config.ChannelRegistration
import org.springframework.messaging.simp.config.MessageBrokerRegistry
import org.springframework.web.socket.config.annotation.*

@Configuration
@EnableWebSocketMessageBroker
class WebSocketConfig : WebSocketMessageBrokerConfigurer {

    override fun configureMessageBroker(registry: MessageBrokerRegistry) {
        // กำหนด prefix สำหรับ topics ที่ server จะ broadcast
        registry.enableSimpleBroker("/topic", "/queue")
        
        // กำหนด prefix สำหรับ messages ที่ client ส่งมา
        registry.setApplicationDestinationPrefixes("/app")
        
        // กำหนด prefix สำหรับ user-specific messages
        registry.setUserDestinationPrefix("/user")
    }

    override fun registerStompEndpoints(registry: StompEndpointRegistry) {
        registry.addEndpoint("/ws")
            .setAllowedOriginPatterns("*")
            .withSockJS()  // รองรับ SockJS fallback สำหรับ browser เก่า
    }

    override fun configureClientInboundChannel(registration: ChannelRegistration) {
        registration.interceptors(AuthChannelInterceptor())
    }
}
```

## Authentication Interceptor

```kotlin
// AuthChannelInterceptor.kt
package com.chat.websocket.config

import org.springframework.messaging.Message
import org.springframework.messaging.MessageChannel
import org.springframework.messaging.simp.stomp.StompCommand
import org.springframework.messaging.simp.stomp.StompHeaderAccessor
import org.springframework.messaging.support.ChannelInterceptor
import org.springframework.messaging.support.MessageHeaderAccessor
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken
import org.springframework.stereotype.Component

@Component
class AuthChannelInterceptor : ChannelInterceptor {

    override fun preSend(message: Message<*>, channel: MessageChannel): Message<*>? {
        val accessor = MessageHeaderAccessor.getAccessor(message, StompHeaderAccessor::class.java)
        
        if (accessor?.command == StompCommand.CONNECT) {
            val token = accessor.getFirstNativeHeader("Authorization")
            
            if (token != null && token.startsWith("Bearer ")) {
                val jwt = token.substring(7)
                val authentication = validateToken(jwt)
                accessor.user = authentication
            }
        }
        
        return message
    }

    private fun validateToken(token: String): UsernamePasswordAuthenticationToken {
        // Validate JWT and return authentication
        // ...implementation...
        return UsernamePasswordAuthenticationToken("user", null, emptyList())
    }
}
```

## Domain Models

```kotlin
// ChatMessage.kt
package com.chat.websocket.domain

import jakarta.persistence.*
import java.time.LocalDateTime

@Entity
@Table(name = "messages")
data class ChatMessage(
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long = 0,
    
    @Column(nullable = false)
    val content: String,
    
    @Column(nullable = false)
    val sender: String,
    
    @Column(nullable = false)
    val roomId: String,
    
    @Enumerated(EnumType.STRING)
    val type: MessageType = MessageType.CHAT,
    
    val timestamp: LocalDateTime = LocalDateTime.now()
)

enum class MessageType {
    CHAT,   // ข้อความปกติ
    JOIN,   // คนเข้ามาใน room
    LEAVE,  // คนออกจาก room
    TYPING, // กำลังพิมพ์
    READ    // อ่านแล้ว
}
```

```kotlin
// ChatRoom.kt
package com.chat.websocket.domain

import jakarta.persistence.*
import java.time.LocalDateTime

@Entity
@Table(name = "chat_rooms")
data class ChatRoom(
    @Id
    val id: String,
    
    @Column(nullable = false)
    val name: String,
    
    @Enumerated(EnumType.STRING)
    val type: RoomType = RoomType.PUBLIC,
    
    val createdAt: LocalDateTime = LocalDateTime.now()
)

enum class RoomType {
    PUBLIC,  // ทุกคนเข้าได้
    PRIVATE, // เชิญเท่านั้น
    DIRECT   // 1:1 chat
}
```

## DTOs

```kotlin
// MessageDtos.kt
package com.chat.websocket.dto

import com.chat.websocket.domain.MessageType
import java.time.LocalDateTime

data class ChatMessageRequest(
    val content: String,
    val roomId: String,
    val type: MessageType = MessageType.CHAT
)

data class ChatMessageResponse(
    val id: Long,
    val content: String,
    val sender: String,
    val roomId: String,
    val type: MessageType,
    val timestamp: LocalDateTime
)

data class TypingIndicator(
    val roomId: String,
    val username: String,
    val isTyping: Boolean
)

data class UserStatus(
    val username: String,
    val online: Boolean,
    val lastSeen: LocalDateTime?
)

data class JoinRoomRequest(
    val roomId: String
)

data class PrivateMessageRequest(
    val recipientUsername: String,
    val content: String
)
```

## WebSocket Controller

```kotlin
// ChatController.kt
package com.chat.websocket.controller

import com.chat.websocket.dto.*
import com.chat.websocket.service.ChatService
import org.springframework.messaging.handler.annotation.*
import org.springframework.messaging.simp.SimpMessageHeaderAccessor
import org.springframework.messaging.simp.SimpMessagingTemplate
import org.springframework.messaging.simp.annotation.SendToUser
import org.springframework.stereotype.Controller
import java.security.Principal

@Controller
class ChatController(
    private val chatService: ChatService,
    private val messagingTemplate: SimpMessagingTemplate
) {

    /**
     * รับ message จาก client และ broadcast ไปยังทุกคนใน room
     * Client ส่งมาที่ /app/chat.sendMessage
     * Server broadcast ไปที่ /topic/room/{roomId}
     */
    @MessageMapping("/chat.sendMessage")
    @SendTo("/topic/room/{roomId}")
    fun sendMessage(
        @Payload message: ChatMessageRequest,
        @DestinationVariable roomId: String,
        principal: Principal
    ): ChatMessageResponse {
        return chatService.saveAndBroadcastMessage(message, principal.name)
    }

    /**
     * เมื่อ user เข้าร่วม room
     */
    @MessageMapping("/chat.join")
    fun joinRoom(
        @Payload request: JoinRoomRequest,
        headerAccessor: SimpMessageHeaderAccessor,
        principal: Principal
    ) {
        val username = principal.name
        
        // เก็บ roomId ไว้ใน session
        headerAccessor.sessionAttributes?.put("roomId", request.roomId)
        headerAccessor.sessionAttributes?.put("username", username)

        // แจ้งทุกคนใน room ว่ามีคนเข้ามา
        val joinMessage = ChatMessageResponse(
            id = 0,
            content = "$username joined the room",
            sender = "System",
            roomId = request.roomId,
            type = MessageType.JOIN,
            timestamp = LocalDateTime.now()
        )

        messagingTemplate.convertAndSend("/topic/room/${request.roomId}", joinMessage)
        
        // ส่งประวัติข้อความให้ user ที่เพิ่งเข้ามา
        val history = chatService.getMessageHistory(request.roomId, 50)
        messagingTemplate.convertAndSendToUser(
            username,
            "/queue/history",
            history
        )
    }

    /**
     * Private message - ส่งให้คนคนเดียว
     */
    @MessageMapping("/chat.private")
    fun sendPrivateMessage(
        @Payload request: PrivateMessageRequest,
        principal: Principal
    ) {
        val message = chatService.savePrivateMessage(request, principal.name)
        
        // ส่งให้ผู้รับ
        messagingTemplate.convertAndSendToUser(
            request.recipientUsername,
            "/queue/private",
            message
        )
        
        // ส่งยืนยันให้ผู้ส่งด้วย
        messagingTemplate.convertAndSendToUser(
            principal.name,
            "/queue/private",
            message
        )
    }

    /**
     * Typing indicator
     */
    @MessageMapping("/chat.typing")
    fun broadcastTypingStatus(
        @Payload indicator: TypingIndicator,
        principal: Principal
    ) {
        val typingStatus = indicator.copy(username = principal.name)
        messagingTemplate.convertAndSend(
            "/topic/room/${indicator.roomId}/typing",
            typingStatus
        )
    }

    /**
     * Read receipt
     */
    @MessageMapping("/chat.read")
    fun markAsRead(
        @Payload messageId: Long,
        principal: Principal
    ) {
        chatService.markMessageAsRead(messageId, principal.name)
        
        val readReceipt = mapOf(
            "messageId" to messageId,
            "readBy" to principal.name,
            "readAt" to LocalDateTime.now()
        )
        
        messagingTemplate.convertAndSend("/topic/receipts", readReceipt)
    }
}
```

## Service Layer

```kotlin
// ChatService.kt
package com.chat.websocket.service

import com.chat.websocket.domain.ChatMessage
import com.chat.websocket.domain.MessageType
import com.chat.websocket.dto.*
import com.chat.websocket.repository.ChatMessageRepository
import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional
import java.time.LocalDateTime

@Service
class ChatService(
    private val messageRepository: ChatMessageRepository
) {

    @Transactional
    fun saveAndBroadcastMessage(
        request: ChatMessageRequest,
        sender: String
    ): ChatMessageResponse {
        val message = ChatMessage(
            content = request.content,
            sender = sender,
            roomId = request.roomId,
            type = request.type
        )

        val saved = messageRepository.save(message)
        return saved.toResponse()
    }

    fun getMessageHistory(roomId: String, limit: Int): List<ChatMessageResponse> {
        return messageRepository
            .findTopByRoomIdOrderByTimestampDesc(roomId, limit)
            .reversed()
            .map { it.toResponse() }
    }

    @Transactional
    fun savePrivateMessage(
        request: PrivateMessageRequest,
        sender: String
    ): ChatMessageResponse {
        val directRoomId = createDirectRoomId(sender, request.recipientUsername)
        
        val message = ChatMessage(
            content = request.content,
            sender = sender,
            roomId = directRoomId,
            type = MessageType.CHAT
        )

        return messageRepository.save(message).toResponse()
    }

    fun markMessageAsRead(messageId: Long, username: String) {
        // อัปเดต read status
        messageRepository.findById(messageId).ifPresent { message ->
            // บันทึกว่า username ได้อ่าน messageId แล้ว
        }
    }

    private fun createDirectRoomId(user1: String, user2: String): String {
        return listOf(user1, user2).sorted().joinToString("_")
    }
}

fun ChatMessage.toResponse() = ChatMessageResponse(
    id = id,
    content = content,
    sender = sender,
    roomId = roomId,
    type = type,
    timestamp = timestamp
)
```

## Event Listener สำหรับ Connection Events

```kotlin
// WebSocketEventListener.kt
package com.chat.websocket.listener

import com.chat.websocket.dto.ChatMessageResponse
import com.chat.websocket.domain.MessageType
import org.springframework.context.event.EventListener
import org.springframework.messaging.simp.SimpMessageSendingOperations
import org.springframework.messaging.simp.stomp.StompHeaderAccessor
import org.springframework.stereotype.Component
import org.springframework.web.socket.messaging.SessionConnectedEvent
import org.springframework.web.socket.messaging.SessionDisconnectEvent
import java.time.LocalDateTime

@Component
class WebSocketEventListener(
    private val messagingTemplate: SimpMessageSendingOperations
) {

    @EventListener
    fun handleWebSocketConnectListener(event: SessionConnectedEvent) {
        val accessor = StompHeaderAccessor.wrap(event.message)
        val username = accessor.user?.name ?: return
        
        println("User Connected: $username")
        
        // แจ้ง online status
        messagingTemplate.convertAndSend(
            "/topic/users/status",
            mapOf("username" to username, "online" to true)
        )
    }

    @EventListener
    fun handleWebSocketDisconnectListener(event: SessionDisconnectEvent) {
        val accessor = StompHeaderAccessor.wrap(event.message)
        val username = accessor.user?.name ?: return
        val roomId = accessor.sessionAttributes?.get("roomId") as? String
        
        println("User Disconnected: $username")

        if (roomId != null) {
            val leaveMessage = ChatMessageResponse(
                id = 0,
                content = "$username left the room",
                sender = "System",
                roomId = roomId,
                type = MessageType.LEAVE,
                timestamp = LocalDateTime.now()
            )
            messagingTemplate.convertAndSend("/topic/room/$roomId", leaveMessage)
        }

        // อัปเดต offline status
        messagingTemplate.convertAndSend(
            "/topic/users/status",
            mapOf(
                "username" to username,
                "online" to false,
                "lastSeen" to LocalDateTime.now()
            )
        )
    }
}
```

## WebSocket Security

```kotlin
// WebSocketSecurityConfig.kt
package com.chat.websocket.config

import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.security.config.annotation.web.socket.EnableWebSocketSecurity
import org.springframework.security.messaging.access.intercept.MessageMatcherDelegatingAuthorizationManager

@Configuration
@EnableWebSocketSecurity
class WebSocketSecurityConfig {

    @Bean
    fun messageAuthorizationManager(
        messages: MessageMatcherDelegatingAuthorizationManager.Builder
    ): MessageMatcherDelegatingAuthorizationManager {
        return messages
            .nullDestMatcher().authenticated()
            .simpSubscribeDestMatchers("/user/**").authenticated()
            .simpSubscribeDestMatchers("/topic/room/**").authenticated()
            .simpDestMatchers("/app/**").authenticated()
            .anyMessage().denyAll()
            .build()
    }
}
```

## REST Controller สำหรับ Room Management

```kotlin
// RoomController.kt
package com.chat.websocket.controller

import com.chat.websocket.domain.ChatRoom
import com.chat.websocket.domain.RoomType
import com.chat.websocket.dto.CreateRoomRequest
import com.chat.websocket.service.RoomService
import org.springframework.http.ResponseEntity
import org.springframework.web.bind.annotation.*
import java.security.Principal

@RestController
@RequestMapping("/api/rooms")
class RoomController(
    private val roomService: RoomService
) {

    @GetMapping
    fun getPublicRooms(): ResponseEntity<List<ChatRoom>> {
        return ResponseEntity.ok(roomService.getPublicRooms())
    }

    @PostMapping
    fun createRoom(
        @RequestBody request: CreateRoomRequest,
        principal: Principal
    ): ResponseEntity<ChatRoom> {
        val room = roomService.createRoom(request, principal.name)
        return ResponseEntity.ok(room)
    }

    @GetMapping("/{roomId}/messages")
    fun getMessageHistory(
        @PathVariable roomId: String,
        @RequestParam(defaultValue = "50") limit: Int
    ): ResponseEntity<List<ChatMessageResponse>> {
        return ResponseEntity.ok(chatService.getMessageHistory(roomId, limit))
    }
}
```

## Frontend JavaScript Client

```javascript
// chat-client.js (ตัวอย่าง client-side code)
const stompClient = new StompJs.Client({
    brokerURL: 'ws://localhost:8080/ws',
    
    connectHeaders: {
        'Authorization': `Bearer ${getToken()}`
    },
    
    onConnect: (frame) => {
        console.log('Connected: ' + frame);
        
        // Subscribe to room messages
        stompClient.subscribe('/topic/room/general', (message) => {
            const chatMessage = JSON.parse(message.body);
            displayMessage(chatMessage);
        });
        
        // Subscribe to private messages
        stompClient.subscribe('/user/queue/private', (message) => {
            const privateMessage = JSON.parse(message.body);
            displayPrivateMessage(privateMessage);
        });
        
        // Subscribe to typing indicators
        stompClient.subscribe('/topic/room/general/typing', (message) => {
            const typing = JSON.parse(message.body);
            showTypingIndicator(typing);
        });
        
        // Join the room
        stompClient.publish({
            destination: '/app/chat.join',
            body: JSON.stringify({ roomId: 'general' })
        });
    }
});

// ส่ง message
function sendMessage(content) {
    stompClient.publish({
        destination: '/app/chat.sendMessage',
        body: JSON.stringify({
            content: content,
            roomId: 'general',
            type: 'CHAT'
        })
    });
}

// ส่ง typing indicator
let typingTimeout;
function onTyping() {
    stompClient.publish({
        destination: '/app/chat.typing',
        body: JSON.stringify({
            roomId: 'general',
            username: currentUser,
            isTyping: true
        })
    });
    
    clearTimeout(typingTimeout);
    typingTimeout = setTimeout(() => {
        stompClient.publish({
            destination: '/app/chat.typing',
            body: JSON.stringify({
                roomId: 'general',
                username: currentUser,
                isTyping: false
            })
        });
    }, 1000);
}

stompClient.activate();
```

## Testing WebSocket

```kotlin
// ChatWebSocketTest.kt
package com.chat.websocket

import org.junit.jupiter.api.Test
import org.springframework.boot.test.context.SpringBootTest
import org.springframework.boot.test.web.server.LocalServerPort
import org.springframework.messaging.converter.MappingJackson2MessageConverter
import org.springframework.messaging.simp.stomp.*
import org.springframework.web.socket.client.standard.StandardWebSocketClient
import org.springframework.web.socket.messaging.WebSocketStompClient
import org.springframework.web.socket.sockjs.client.SockJsClient
import org.springframework.web.socket.sockjs.client.WebSocketTransport
import java.lang.reflect.Type
import java.util.concurrent.CountDownLatch
import java.util.concurrent.TimeUnit

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class ChatWebSocketTest {

    @LocalServerPort
    private var port: Int = 0

    @Test
    fun `should send and receive chat message`() {
        val latch = CountDownLatch(1)
        var receivedMessage: ChatMessageResponse? = null

        val client = WebSocketStompClient(
            SockJsClient(listOf(WebSocketTransport(StandardWebSocketClient())))
        )
        client.messageConverter = MappingJackson2MessageConverter()

        val session = client.connectAsync(
            "ws://localhost:$port/ws",
            object : StompSessionHandlerAdapter() {}
        ).get(5, TimeUnit.SECONDS)

        // Subscribe to room
        session.subscribe("/topic/room/test", object : StompFrameHandler {
            override fun getPayloadType(headers: StompHeaders): Type {
                return ChatMessageResponse::class.java
            }

            override fun handleFrame(headers: StompHeaders, payload: Any?) {
                receivedMessage = payload as ChatMessageResponse
                latch.countDown()
            }
        })

        // Send message
        session.send(
            "/app/chat.sendMessage",
            ChatMessageRequest(content = "Hello!", roomId = "test")
        )

        assert(latch.await(5, TimeUnit.SECONDS)) { "Message not received in time" }
        assert(receivedMessage?.content == "Hello!")
    }
}
```

## สรุปแนวคิด WebSocket

| แนวคิด | คำอธิบาย |
|--------|---------|
| WebSocket Upgrade | HTTP request ที่ขอ upgrade เป็น WebSocket |
| STOMP | Protocol บน WebSocket สำหรับ messaging |
| /app prefix | Messages ที่ client ส่งมา (ผ่าน @MessageMapping) |
| /topic prefix | Broadcast channels (ทุกคน subscribe ได้) |
| /queue prefix | User-specific queues |
| /user prefix | ส่งให้ user คนเดียว |
| SockJS | Fallback สำหรับ browser ที่ไม่ support WebSocket |

## Use Cases ที่เหมาะกับ WebSocket

1. **Real-time Chat** - ตัวอย่างในบทนี้
2. **Live Notifications** - แจ้งเตือนแบบ real-time
3. **Collaborative Editing** - Google Docs style
4. **Live Dashboard** - ข้อมูลที่อัปเดตตลอดเวลา
5. **Online Gaming** - เกมที่ต้องการ low latency
6. **Live Stock Prices** - ราคาหุ้น real-time
7. **IoT Data Streaming** - ข้อมูลจาก sensors

*Part 57/100+ | Kotlin & Spring Boot Complete Course*
