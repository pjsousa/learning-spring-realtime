# Deciphering STOMP Messaging: The Vectors of Real-Time Partitioning

*Estimated reading time: 25 minutes · Target word count: ~1,900*

## Overview

In previous articles, we've used STOMP destinations like `/topic/public` and `/user/queue/private` without fully unpacking how Spring and the STOMP protocol organize and route messages. Unlike raw WebSockets, which are merely byte streams, STOMP introduces semantic layers that enable complex, scalable messaging architectures. This article analyzes the six critical partitioning and routing vectors—**Topic**, **Queue**, **User**, **Subscription**, **Connection**, and **Session**—that allow messages to be directed precisely where they need to go.

By the end of this article, you'll understand how these vectors work individually and together, enabling both broadcast and targeted private messaging patterns. We'll build a comprehensive demo application that showcases all six vectors in action: a multi-room chat system with private messaging, user presence tracking, and session management. This layer of abstraction frees developers from dealing with low-level connection management and raw message handling, letting you think in terms of destinations, users, and subscriptions rather than TCP sockets and frame parsing.

## Learning Goals

- Understand the six partitioning vectors in STOMP communication and their semantic meanings.
- Distinguish between Topic (publish-subscribe) and Queue (point-to-point) destination patterns.
- Grasp how Subscription IDs enable multiple concurrent subscriptions per connection.
- Learn how User destinations abstract session management for authenticated users.
- Explore the relationship between Connection (physical transport) and Session (application context).
- Build a practical application that demonstrates all six vectors working together.
- Use Spring's messaging APIs to inspect and manipulate these vectors programmatically.

## Why Understanding Vectors Matters

When building real-time applications, you need to answer fundamental routing questions:

- **Who should receive this message?** (Topic vs Queue vs User)
- **How do we track which client is listening?** (Subscription)
- **How do we identify the sender?** (User + Session)
- **What happens when a user has multiple browser tabs?** (Connection + Session relationship)

Without understanding these vectors, you might find yourself manually tracking session IDs, reinventing subscription management, or struggling to scale beyond simple broadcast scenarios. The STOMP protocol and Spring's abstractions provide elegant answers to these questions—if you know how to use them.

## The Six Partitioning Vectors

Let's define each vector and its role in STOMP messaging:

| Vector | STOMP Command/Header | Function & Semantic Meaning |
|--------|---------------------|---------------------------|
| **1. Destination (Topic/Queue)** | `destination` header in SEND and SUBSCRIBE frames | Highest level of partitioning, defining who should potentially receive the message. Topics (`/topic/**`) imply publish-subscribe (one-to-many). Queues (`/queue/**`) imply point-to-point (one-to-one). |
| **2. Subscription** | `id` header in SUBSCRIBE/UNSUBSCRIBE frames | The specific request made by a client to listen to a destination. Each connection can have multiple open subscriptions, and the `id` must be unique per connection. The server uses the `subscription` header in MESSAGE frames to match the client's `id`. |
| **3. User** | Authenticated Principal associated with the session | Represents the authenticated human or identity layer. Spring uses User Destinations (e.g., `/user/queue/**`) to target messages to this authenticated identity, regardless of how many connections or sessions they have open. |
| **4. Session** | `WebSocketSession` object or `simpSessionId` header | Represents the application-level context associated with a single client connection, maintained by Spring. It holds attributes and is crucial for linking the authenticated user (Principal) to the physical pipe. |
| **5. Connection** | The underlying TCP socket or WebSocket connection | The physical network conduit over which STOMP frames flow. All messages for a session share and flow on the same connection. |

These vectors form a hierarchy: **Connection** → **Session** → **User** → **Subscription** → **Destination (Topic/Queue)**. Understanding this hierarchy is key to building robust real-time applications.

## Prerequisites and Project Setup

This article assumes familiarity with:

- Basic Spring Boot STOMP setup (Articles 1-2)
- User destinations (Article 7)
- Spring Security integration (Article 6)

We'll build a comprehensive demo that requires authentication, so ensure you have Spring Security configured. If you're starting fresh, follow the setup below.

### Generate the Project

Create a new Spring Boot 3.2+ project with the following dependencies:

```bash
curl https://start.spring.io/starter.zip \
  -d dependencies=websocket,web,security \
  -d name=stomp-vectors-demo \
  -d javaVersion=21 \
  -d type=maven-project \
  -d packageName=com.example.stompvectors \
  -o stomp-vectors-demo.zip

unzip stomp-vectors-demo.zip
cd stomp-vectors-demo
```

Your `pom.xml` should include:

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-websocket</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-security</artifactId>
    </dependency>
</dependencies>
```

## Step 1: Configure WebSocket with All Vector Support

We'll configure the WebSocket broker to support topics, queues, and user destinations—demonstrating three of our six vectors at the configuration level.

Create `src/main/java/com/example/stompvectors/config/WebSocketConfig.java`:

```java
package com.example.stompvectors.config;

import org.springframework.context.annotation.Configuration;
import org.springframework.messaging.simp.config.MessageBrokerRegistry;
import org.springframework.web.socket.config.annotation.EnableWebSocketMessageBroker;
import org.springframework.web.socket.config.annotation.StompEndpointRegistry;
import org.springframework.web.socket.config.annotation.WebSocketMessageBrokerConfigurer;

@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {

    @Override
    public void configureMessageBroker(MessageBrokerRegistry registry) {
        // Vector 1: Topics (broadcast) and Queues (point-to-point)
        registry.enableSimpleBroker("/topic", "/queue");
        
        // Application destination prefix for client-to-server messages
        registry.setApplicationDestinationPrefixes("/app");
        
        // Vector 3: User destination prefix for user-specific routing
        registry.setUserDestinationPrefix("/user");
    }

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        // Vector 5: Connection endpoint - the physical WebSocket connection
        registry.addEndpoint("/ws-vectors")
                .setAllowedOriginPatterns("*")
                .withSockJS();
    }
}
```

**Key points:**

- **Topics (`/topic/**`)**: Broadcast destinations where all subscribers receive every message.
- **Queues (`/queue/**`)**: Point-to-point destinations where typically one subscriber receives each message.
- **User destinations (`/user/**`)**: Spring translates these to user-specific queues, abstracting session management.
- **Connection endpoint (`/ws-vectors`)**: The physical connection point where clients establish WebSocket connections.

## Step 2: Set Up Minimal Security

We need authenticated users to demonstrate the User vector. Create a simple security configuration:

Create `src/main/java/com/example/stompvectors/config/SecurityConfig.java`:

```java
package com.example.stompvectors.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.Customizer;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.core.userdetails.User;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.security.provisioning.InMemoryUserDetailsManager;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
public class SecurityConfig {

    @Bean
    SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/css/**", "/js/**", "/login", "/favicon.ico").permitAll()
                .anyRequest().authenticated()
            )
            .formLogin(Customizer.withDefaults())
            .logout(logout -> logout.logoutSuccessUrl("/login?logout").permitAll())
            .csrf(csrf -> csrf.ignoringRequestMatchers("/ws-vectors/**"));
        
        return http.build();
    }

    @Bean
    UserDetailsService userDetailsService(PasswordEncoder passwordEncoder) {
        return new InMemoryUserDetailsManager(
            User.withUsername("alice")
                .password(passwordEncoder.encode("password"))
                .roles("USER")
                .build(),
            User.withUsername("bob")
                .password(passwordEncoder.encode("password"))
                .roles("USER")
                .build(),
            User.withUsername("charlie")
                .password(passwordEncoder.encode("password"))
                .roles("USER")
                .build()
        );
    }

    @Bean
    PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

This provides three test users (alice, bob, charlie) all with password "password".

## Step 3: Model Messages for Multi-Vector Demo

We'll create message DTOs that support our multi-room chat with private messaging:

Create `src/main/java/com/example/stompvectors/model/ChatMessage.java`:

```java
package com.example.stompvectors.model;

import java.time.Instant;

public class ChatMessage {
    private String from;
    private String content;
    private String room;
    private Instant timestamp;
    private MessageType type;

    public enum MessageType {
        CHAT, JOIN, LEAVE, PRIVATE
    }

    public ChatMessage() {
        this.timestamp = Instant.now();
    }

    public ChatMessage(String from, String content, String room, MessageType type) {
        this.from = from;
        this.content = content;
        this.room = room;
        this.type = type;
        this.timestamp = Instant.now();
    }

    // Getters and setters
    public String getFrom() {
        return from;
    }

    public void setFrom(String from) {
        this.from = from;
    }

    public String getContent() {
        return content;
    }

    public void setContent(String content) {
        this.content = content;
    }

    public String getRoom() {
        return room;
    }

    public void setRoom(String room) {
        this.room = room;
    }

    public Instant getTimestamp() {
        return timestamp;
    }

    public void setTimestamp(Instant timestamp) {
        this.timestamp = timestamp;
    }

    public MessageType getType() {
        return type;
    }

    public void setType(MessageType type) {
        this.type = type;
    }
}
```

Create `src/main/java/com/example/stompvectors/model/PresenceUpdate.java`:

```java
package com.example.stompvectors.model;

import java.time.Instant;
import java.util.Set;

public class PresenceUpdate {
    private String room;
    private Set<String> activeUsers;
    private Instant timestamp;

    public PresenceUpdate() {
        this.timestamp = Instant.now();
    }

    public PresenceUpdate(String room, Set<String> activeUsers) {
        this.room = room;
        this.activeUsers = activeUsers;
        this.timestamp = Instant.now();
    }

    // Getters and setters
    public String getRoom() {
        return room;
    }

    public void setRoom(String room) {
        this.room = room;
    }

    public Set<String> getActiveUsers() {
        return activeUsers;
    }

    public void setActiveUsers(Set<String> activeUsers) {
        this.activeUsers = activeUsers;
    }

    public Instant getTimestamp() {
        return timestamp;
    }

    public void setTimestamp(Instant timestamp) {
        this.timestamp = timestamp;
    }
}
```

## Step 4: Implement Vector-Aware Controllers

Now we'll create controllers that demonstrate all six vectors. This is where the magic happens—we'll inspect and use each vector programmatically.

Create `src/main/java/com/example/stompvectors/controller/ChatController.java`:

```java
package com.example.stompvectors.controller;

import com.example.stompvectors.model.ChatMessage;
import com.example.stompvectors.model.PresenceUpdate;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.messaging.handler.annotation.Header;
import org.springframework.messaging.handler.annotation.MessageMapping;
import org.springframework.messaging.handler.annotation.Payload;
import org.springframework.messaging.simp.SimpMessagingTemplate;
import org.springframework.messaging.simp.annotation.SubscribeMapping;
import org.springframework.messaging.simp.stomp.StompHeaderAccessor;
import org.springframework.stereotype.Controller;

import java.security.Principal;
import java.time.Instant;
import java.util.Set;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.ConcurrentMap;

@Controller
public class ChatController {

    private static final Logger log = LoggerFactory.getLogger(ChatController.class);
    
    // Track active users per room (Vector 3: User, Vector 1: Topic)
    private final ConcurrentMap<String, Set<String>> roomUsers = new ConcurrentHashMap<>();
    
    private final SimpMessagingTemplate messagingTemplate;

    public ChatController(SimpMessagingTemplate messagingTemplate) {
        this.messagingTemplate = messagingTemplate;
    }

    /**
     * Vector 1: Topic - Broadcast to all subscribers of /topic/room/{roomName}
     * Vector 2: Subscription - Each client subscription has a unique ID
     * Vector 3: User - Principal identifies the authenticated user
     * Vector 4: Session - simpSessionId identifies the application session
     * Vector 5: Connection - Underlying WebSocket connection (implicit)
     */
    @MessageMapping("/chat.send")
    public void sendMessage(
            @Payload ChatMessage message,
            Principal principal,
            @Header("simpSessionId") String sessionId,
            StompHeaderAccessor accessor) {
        
        // Vector 3: Extract user from Principal
        String username = principal != null ? principal.getName() : "anonymous";
        message.setFrom(username);
        message.setTimestamp(Instant.now());
        
        // Vector 4: Access session attributes (stored in the Session vector)
        accessor.getSessionAttributes().put("lastActivity", Instant.now());
        
        String room = message.getRoom();
        log.info("Message from user {} (session {}) to room {}", username, sessionId, room);
        
        // Vector 1: Broadcast to topic - all subscribers receive this
        messagingTemplate.convertAndSend("/topic/room/" + room, message);
    }

    /**
     * Vector 2: Subscription - Client subscribes with a unique subscription ID
     * Vector 1: Topic - Subscribe to room broadcasts
     * Vector 3: User - Track user presence in room
     * Vector 4: Session - Store room membership in session attributes
     */
    @SubscribeMapping("/topic/room/{roomName}")
    public PresenceUpdate onRoomSubscribe(
            @Header("simpSessionId") String sessionId,
            @Header("simpDestination") String destination,
            Principal principal,
            StompHeaderAccessor accessor) {
        
        String roomName = extractRoomName(destination);
        String username = principal != null ? principal.getName() : "anonymous";
        
        // Vector 4: Store room in session attributes
        accessor.getSessionAttributes().put("currentRoom", roomName);
        
        // Vector 3: Track user presence
        roomUsers.computeIfAbsent(roomName, k -> ConcurrentHashMap.newKeySet()).add(username);
        
        log.info("User {} (session {}) subscribed to room {} (subscription active)", 
                username, sessionId, roomName);
        
        // Return initial presence data (Vector 2: Direct reply to this subscription)
        return new PresenceUpdate(roomName, roomUsers.get(roomName));
    }

    /**
     * Vector 1: Queue - Point-to-point messaging via user destinations
     * Vector 3: User - Target specific user regardless of their sessions
     */
    @MessageMapping("/private.send")
    public void sendPrivateMessage(
            @Payload ChatMessage message,
            Principal principal) {
        
        String sender = principal != null ? principal.getName() : "anonymous";
        message.setFrom(sender);
        message.setType(ChatMessage.MessageType.PRIVATE);
        message.setTimestamp(Instant.now());
        
        String recipient = message.getContent().split(":", 2)[0].trim();
        String content = message.getContent().contains(":") 
            ? message.getContent().split(":", 2)[1].trim() 
            : message.getContent();
        
        message.setContent(content);
        
        log.info("Private message from {} to {}", sender, recipient);
        
        // Vector 1: Queue + Vector 3: User - Send to user's queue
        messagingTemplate.convertAndSendToUser(
            recipient,
            "/queue/private",
            message
        );
    }

    private String extractRoomName(String destination) {
        // Extract room name from /topic/room/{roomName}
        if (destination != null && destination.startsWith("/topic/room/")) {
            return destination.substring("/topic/room/".length());
        }
        return "general";
    }
}
```

**Vector usage breakdown:**

1. **Topic (`/topic/room/{roomName}`)**: Broadcast messages to all room subscribers.
2. **Queue (`/queue/private`)**: Point-to-point private messages via user destinations.
3. **User (Principal)**: Identifies the authenticated sender and recipient.
4. **Session (`simpSessionId`)**: Tracks individual connections and stores attributes.
5. **Subscription**: Implicit in `@SubscribeMapping`—each client subscription has a unique ID.
6. **Connection**: The underlying WebSocket transport (handled by Spring).

## Step 5: Track Session Lifecycle Events

To demonstrate Connection and Session vectors, we'll listen for lifecycle events:

Create `src/main/java/com/example/stompvectors/listener/SessionLifecycleListener.java`:

```java
package com.example.stompvectors.listener;

import com.example.stompvectors.model.ChatMessage;
import com.example.stompvectors.model.PresenceUpdate;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.context.event.EventListener;
import org.springframework.messaging.simp.SimpMessagingTemplate;
import org.springframework.messaging.simp.stomp.StompHeaderAccessor;
import org.springframework.stereotype.Component;
import org.springframework.web.socket.messaging.SessionConnectedEvent;
import org.springframework.web.socket.messaging.SessionDisconnectEvent;
import org.springframework.web.socket.messaging.SessionSubscribeEvent;
import org.springframework.web.socket.messaging.SessionUnsubscribeEvent;

import java.security.Principal;
import java.util.Set;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.ConcurrentMap;

@Component
public class SessionLifecycleListener {

    private static final Logger log = LoggerFactory.getLogger(SessionLifecycleListener.class);
    
    // Track sessions per user (Vector 4: Session, Vector 3: User)
    private final ConcurrentMap<String, Set<String>> userSessions = new ConcurrentHashMap<>();
    
    // Track room memberships (Vector 1: Topic subscriptions)
    private final ConcurrentMap<String, Set<String>> roomUsers = new ConcurrentHashMap<>();
    
    private final SimpMessagingTemplate messagingTemplate;

    public SessionLifecycleListener(SimpMessagingTemplate messagingTemplate) {
        this.messagingTemplate = messagingTemplate;
    }

    /**
     * Vector 5: Connection established
     * Vector 4: Session created
     * Vector 3: User associated with session
     */
    @EventListener
    public void handleSessionConnected(SessionConnectedEvent event) {
        StompHeaderAccessor accessor = StompHeaderAccessor.wrap(event.getMessage());
        String sessionId = accessor.getSessionId();
        Principal principal = accessor.getUser();
        
        if (principal != null) {
            String username = principal.getName();
            userSessions.computeIfAbsent(username, k -> ConcurrentHashMap.newKeySet()).add(sessionId);
            log.info("Connection established: User {} now has {} active session(s) [Session: {}]",
                    username, userSessions.get(username).size(), sessionId);
        } else {
            log.info("Connection established: Anonymous session [Session: {}]", sessionId);
        }
    }

    /**
     * Vector 2: Subscription created
     * Vector 1: Destination (Topic/Queue) subscribed
     */
    @EventListener
    public void handleSessionSubscribe(SessionSubscribeEvent event) {
        StompHeaderAccessor accessor = StompHeaderAccessor.wrap(event.getMessage());
        String sessionId = accessor.getSessionId();
        String destination = accessor.getDestination();
        String subscriptionId = accessor.getSubscriptionId();
        Principal principal = accessor.getUser();
        
        String username = principal != null ? principal.getName() : "anonymous";
        
        log.info("Subscription created: User {} [Session: {}] subscribed to {} [Subscription ID: {}]",
                username, sessionId, destination, subscriptionId);
        
        // Track room subscriptions
        if (destination != null && destination.startsWith("/topic/room/")) {
            String roomName = destination.substring("/topic/room/".length());
            roomUsers.computeIfAbsent(roomName, k -> ConcurrentHashMap.newKeySet()).add(username);
            
            // Broadcast updated presence (Vector 1: Topic broadcast)
            messagingTemplate.convertAndSend("/topic/room/" + roomName,
                    new PresenceUpdate(roomName, roomUsers.get(roomName)));
        }
    }

    /**
     * Vector 2: Subscription removed
     */
    @EventListener
    public void handleSessionUnsubscribe(SessionUnsubscribeEvent event) {
        StompHeaderAccessor accessor = StompHeaderAccessor.wrap(event.getMessage());
        String sessionId = accessor.getSessionId();
        String subscriptionId = accessor.getSubscriptionId();
        Principal principal = accessor.getUser();
        
        String username = principal != null ? principal.getName() : "anonymous";
        
        log.info("Subscription removed: User {} [Session: {}] unsubscribed [Subscription ID: {}]",
                username, sessionId, subscriptionId);
    }

    /**
     * Vector 5: Connection closed
     * Vector 4: Session destroyed
     * Vector 3: User session removed
     */
    @EventListener
    public void handleSessionDisconnect(SessionDisconnectEvent event) {
        StompHeaderAccessor accessor = StompHeaderAccessor.wrap(event.getMessage());
        String sessionId = accessor.getSessionId();
        Principal principal = accessor.getUser();
        
        if (principal != null) {
            String username = principal.getName();
            Set<String> sessions = userSessions.get(username);
            if (sessions != null) {
                sessions.remove(sessionId);
                log.info("Connection closed: User {} now has {} active session(s) [Session: {}]",
                        username, sessions.size(), sessionId);
                
                // Remove from all rooms
                roomUsers.values().forEach(users -> users.remove(username));
                
                // Notify rooms of user departure
                roomUsers.forEach((room, users) -> {
                    if (users.contains(username)) {
                        users.remove(username);
                        messagingTemplate.convertAndSend("/topic/room/" + room,
                                new PresenceUpdate(room, users));
                    }
                });
            }
        } else {
            log.info("Connection closed: Anonymous session [Session: {}]", sessionId);
        }
    }
}
```

This listener demonstrates how to track all six vectors through Spring's event system:

- **Connection**: `SessionConnectedEvent` and `SessionDisconnectEvent`
- **Session**: `simpSessionId` in all events
- **User**: `Principal` from the session
- **Subscription**: `SessionSubscribeEvent` and `SessionUnsubscribeEvent` with subscription IDs
- **Destination**: Extracted from subscribe events
- **Topic/Queue**: Determined by destination prefix

## Step 6: Create a Vector Inspection Service

Let's create a service that exposes vector information for debugging and monitoring:

Create `src/main/java/com/example/stompvectors/service/VectorInspectionService.java`:

```java
package com.example.stompvectors.service;

import org.springframework.messaging.simp.user.SimpUser;
import org.springframework.messaging.simp.user.SimpUserRegistry;
import org.springframework.stereotype.Service;

import java.util.HashMap;
import java.util.Map;
import java.util.stream.Collectors;

@Service
public class VectorInspectionService {

    private final SimpUserRegistry userRegistry;

    public VectorInspectionService(SimpUserRegistry userRegistry) {
        this.userRegistry = userRegistry;
    }

    /**
     * Inspect all active vectors in the system
     */
    public Map<String, Object> inspectVectors() {
        Map<String, Object> vectors = new HashMap<>();
        
        // Vector 3: Users
        Map<String, Object> users = new HashMap<>();
        for (SimpUser user : userRegistry.getUsers()) {
            Map<String, Object> userInfo = new HashMap<>();
            userInfo.put("username", user.getName());
            userInfo.put("sessionCount", user.getSessions().size());
            userInfo.put("sessions", user.getSessions().stream()
                    .map(session -> {
                        Map<String, Object> sessionInfo = new HashMap<>();
                        sessionInfo.put("id", session.getId());
                        sessionInfo.put("subscriptionCount", session.getSubscriptions().size());
                        sessionInfo.put("subscriptions", session.getSubscriptions().stream()
                                .map(sub -> {
                                    Map<String, Object> subInfo = new HashMap<>();
                                    subInfo.put("id", sub.getId());
                                    subInfo.put("destination", sub.getDestination());
                                    return subInfo;
                                })
                                .collect(Collectors.toList()));
                        return sessionInfo;
                    })
                    .collect(Collectors.toList()));
            users.put(user.getName(), userInfo);
        }
        vectors.put("users", users);
        vectors.put("totalUsers", userRegistry.getUserCount());
        
        return vectors;
    }
}
```

## Step 7: Add REST Endpoint for Vector Inspection

Create a REST controller to expose vector information:

Create `src/main/java/com/example/stompvectors/controller/VectorInspectionController.java`:

```java
package com.example.stompvectors.controller;

import com.example.stompvectors.service.VectorInspectionService;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import java.util.Map;

@RestController
@RequestMapping("/api/vectors")
public class VectorInspectionController {

    private final VectorInspectionService inspectionService;

    public VectorInspectionController(VectorInspectionService inspectionService) {
        this.inspectionService = inspectionService;
    }

    @GetMapping("/inspect")
    public Map<String, Object> inspect() {
        return inspectionService.inspectVectors();
    }
}
```

## Step 8: Create the Browser Client

The client will demonstrate all six vectors through a multi-room chat interface with private messaging:

Create `src/main/resources/templates/index.html` (add Thymeleaf dependency if needed):

```html
<!DOCTYPE html>
<html lang="en" xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>STOMP Vectors Demo</title>
    <style>
        body {
            font-family: system-ui, sans-serif;
            margin: 0;
            padding: 1rem;
            background: #f5f5f5;
        }
        .container {
            max-width: 1200px;
            margin: 0 auto;
            display: grid;
            grid-template-columns: 250px 1fr 300px;
            gap: 1rem;
            height: calc(100vh - 2rem);
        }
        .sidebar {
            background: white;
            border-radius: 8px;
            padding: 1rem;
            box-shadow: 0 2px 4px rgba(0,0,0,0.1);
        }
        .main {
            background: white;
            border-radius: 8px;
            padding: 1rem;
            box-shadow: 0 2px 4px rgba(0,0,0,0.1);
            display: flex;
            flex-direction: column;
        }
        .info-panel {
            background: white;
            border-radius: 8px;
            padding: 1rem;
            box-shadow: 0 2px 4px rgba(0,0,0,0.1);
            font-size: 0.875rem;
        }
        h1 { margin: 0 0 1rem 0; font-size: 1.5rem; }
        h2 { margin: 0 0 0.5rem 0; font-size: 1.125rem; }
        .status {
            padding: 0.5rem;
            border-radius: 4px;
            margin-bottom: 1rem;
            font-weight: 600;
        }
        .status.connected { background: #d4edda; color: #155724; }
        .status.disconnected { background: #f8d7da; color: #721c24; }
        .room-list {
            list-style: none;
            padding: 0;
            margin: 0;
        }
        .room-item {
            padding: 0.5rem;
            margin-bottom: 0.5rem;
            background: #f8f9fa;
            border-radius: 4px;
            cursor: pointer;
        }
        .room-item.active {
            background: #007bff;
            color: white;
        }
        .messages {
            flex: 1;
            overflow-y: auto;
            border: 1px solid #ddd;
            border-radius: 4px;
            padding: 1rem;
            margin-bottom: 1rem;
            background: #fafafa;
        }
        .message {
            margin-bottom: 0.75rem;
            padding: 0.5rem;
            background: white;
            border-radius: 4px;
            border-left: 3px solid #007bff;
        }
        .message.private {
            border-left-color: #ffc107;
        }
        .message-sender {
            font-weight: 600;
            color: #007bff;
        }
        .message-content {
            margin-top: 0.25rem;
        }
        .message-timestamp {
            font-size: 0.75rem;
            color: #666;
            margin-top: 0.25rem;
        }
        form {
            display: flex;
            gap: 0.5rem;
        }
        input[type="text"] {
            flex: 1;
            padding: 0.5rem;
            border: 1px solid #ddd;
            border-radius: 4px;
        }
        button {
            padding: 0.5rem 1rem;
            background: #007bff;
            color: white;
            border: none;
            border-radius: 4px;
            cursor: pointer;
        }
        button:disabled {
            background: #ccc;
            cursor: not-allowed;
        }
        .vector-info {
            margin-bottom: 1rem;
            padding: 0.5rem;
            background: #f8f9fa;
            border-radius: 4px;
        }
        .vector-info strong {
            display: block;
            margin-bottom: 0.25rem;
        }
        .presence {
            margin-top: 1rem;
        }
        .presence ul {
            list-style: none;
            padding: 0;
            margin: 0.5rem 0 0 0;
        }
        .presence li {
            padding: 0.25rem;
            background: #e9ecef;
            margin-bottom: 0.25rem;
            border-radius: 4px;
        }
    </style>
</head>
<body>
    <div class="container">
        <div class="sidebar">
            <h2>Rooms</h2>
            <ul class="room-list" id="room-list">
                <li class="room-item" data-room="general">General</li>
                <li class="room-item" data-room="tech">Tech</li>
                <li class="room-item" data-room="random">Random</li>
            </ul>
            
            <div class="presence">
                <h3>Active Users</h3>
                <ul id="presence-list"></ul>
            </div>
        </div>
        
        <div class="main">
            <h1>STOMP Vectors Demo</h1>
            <div id="status" class="status disconnected">Disconnected</div>
            
            <div class="messages" id="messages"></div>
            
            <form id="message-form">
                <input type="text" id="message-input" placeholder="Type a message..." required>
                <button type="submit">Send</button>
            </form>
            
            <div style="margin-top: 1rem;">
                <h3>Private Message</h3>
                <form id="private-form">
                    <input type="text" id="private-recipient" placeholder="username: message" required>
                    <button type="submit">Send Private</button>
                </form>
            </div>
        </div>
        
        <div class="info-panel">
            <h2>Vector Information</h2>
            <div class="vector-info">
                <strong>Connection:</strong>
                <span id="connection-status">Not connected</span>
            </div>
            <div class="vector-info">
                <strong>Session ID:</strong>
                <span id="session-id">-</span>
            </div>
            <div class="vector-info">
                <strong>User:</strong>
                <span id="user-name" th:text="${username}">-</span>
            </div>
            <div class="vector-info">
                <strong>Active Subscriptions:</strong>
                <ul id="subscriptions-list" style="list-style: none; padding: 0; margin: 0.5rem 0 0 0;"></ul>
            </div>
            <div class="vector-info">
                <strong>Current Room (Topic):</strong>
                <span id="current-room">-</span>
            </div>
            
            <button onclick="inspectVectors()" style="margin-top: 1rem; width: 100%;">Inspect All Vectors</button>
            <pre id="vector-inspection" style="font-size: 0.75rem; overflow: auto; max-height: 300px; margin-top: 1rem;"></pre>
        </div>
    </div>

    <script src="https://cdn.jsdelivr.net/npm/sockjs-client@1/dist/sockjs.min.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/stompjs@2.3.3/lib/stomp.min.js"></script>
    <script>
        const username = /*[[${username}]]*/ 'anonymous';
        let stompClient = null;
        let currentRoom = 'general';
        let subscriptions = new Map(); // Vector 2: Track subscriptions

        const statusEl = document.getElementById('status');
        const messagesEl = document.getElementById('messages');
        const messageForm = document.getElementById('message-form');
        const messageInput = document.getElementById('message-input');
        const privateForm = document.getElementById('private-form');
        const privateRecipient = document.getElementById('private-recipient');
        const roomList = document.getElementById('room-list');
        const presenceList = document.getElementById('presence-list');
        const sessionIdEl = document.getElementById('session-id');
        const subscriptionsList = document.getElementById('subscriptions-list');
        const currentRoomEl = document.getElementById('current-room');
        const connectionStatusEl = document.getElementById('connection-status');

        function setConnected(connected) {
            statusEl.textContent = connected ? 'Connected' : 'Disconnected';
            statusEl.className = `status ${connected ? 'connected' : 'disconnected'}`;
            connectionStatusEl.textContent = connected ? 'Active' : 'Inactive';
        }

        function connect() {
            // Vector 5: Establish connection
            const socket = new SockJS('/ws-vectors');
            stompClient = Stomp.over(socket);
            stompClient.debug = null;

            stompClient.connect({}, function(frame) {
                console.log('Connected: ' + frame);
                setConnected(true);
                
                // Vector 4: Extract session ID from connection headers (if available)
                // Note: Session ID is typically available server-side, but we can track it
                // through subscription callbacks or by storing it in a custom header
                
                // Vector 1: Subscribe to current room topic
                subscribeToRoom(currentRoom);
                
                // Vector 1: Subscribe to private queue (Vector 3: User destination)
                const privateSub = stompClient.subscribe('/user/queue/private', function(message) {
                    const msg = JSON.parse(message.body);
                    appendMessage(msg, true);
                });
                subscriptions.set('private', privateSub);
                updateSubscriptionsList();
                
                console.log('Subscribed to /user/queue/private');
            }, function(error) {
                console.error('STOMP error: ' + error);
                setConnected(false);
            });
        }

        function subscribeToRoom(roomName) {
            if (!stompClient || !stompClient.connected) return;
            
            // Unsubscribe from previous room if exists
            if (subscriptions.has('room')) {
                subscriptions.get('room').unsubscribe();
                subscriptions.delete('room');
            }
            
            // Vector 1: Topic subscription
            // Vector 2: Each subscription gets a unique ID from STOMP
            const roomSub = stompClient.subscribe('/topic/room/' + roomName, function(message) {
                const payload = JSON.parse(message.body);
                
                // Check if it's a presence update
                if (payload.activeUsers) {
                    updatePresence(payload.activeUsers);
                } else {
                    appendMessage(payload, false);
                }
            });
            
            subscriptions.set('room', roomSub);
            currentRoom = roomName;
            currentRoomEl.textContent = roomName;
            updateSubscriptionsList();
            
            // Update UI
            document.querySelectorAll('.room-item').forEach(item => {
                item.classList.toggle('active', item.dataset.room === roomName);
            });
        }

        function updateSubscriptionsList() {
            subscriptionsList.innerHTML = '';
            subscriptions.forEach((sub, name) => {
                const li = document.createElement('li');
                li.textContent = `${name}: ${sub.id || 'active'}`;
                li.style.padding = '0.25rem';
                li.style.background = '#e9ecef';
                li.style.marginBottom = '0.25rem';
                li.style.borderRadius = '4px';
                subscriptionsList.appendChild(li);
            });
        }

        function updatePresence(users) {
            presenceList.innerHTML = '';
            users.forEach(user => {
                const li = document.createElement('li');
                li.textContent = user;
                presenceList.appendChild(li);
            });
        }

        function sendMessage(event) {
            event.preventDefault();
            const content = messageInput.value.trim();
            if (!content || !stompClient || !stompClient.connected) return;

            const message = {
                content: content,
                room: currentRoom
            };

            // Vector 1: Send to application destination (Topic will be used for broadcast)
            stompClient.send('/app/chat.send', {}, JSON.stringify(message));
            messageInput.value = '';
        }

        function sendPrivateMessage(event) {
            event.preventDefault();
            const input = privateRecipient.value.trim();
            if (!input || !stompClient || !stompClient.connected) return;

            const message = {
                content: input,
                room: 'private'
            };

            // Vector 1: Send to application destination (Queue + User will be used)
            stompClient.send('/app/private.send', {}, JSON.stringify(message));
            privateRecipient.value = '';
        }

        function appendMessage(msg, isPrivate) {
            const messageDiv = document.createElement('div');
            messageDiv.className = `message ${isPrivate ? 'private' : ''}`;

            const senderSpan = document.createElement('div');
            senderSpan.className = 'message-sender';
            senderSpan.textContent = isPrivate ? `🔒 ${msg.from}` : msg.from;
            messageDiv.appendChild(senderSpan);

            const contentDiv = document.createElement('div');
            contentDiv.className = 'message-content';
            contentDiv.textContent = msg.content;
            messageDiv.appendChild(contentDiv);

            if (msg.timestamp) {
                const timestampDiv = document.createElement('div');
                timestampDiv.className = 'message-timestamp';
                timestampDiv.textContent = new Date(msg.timestamp).toLocaleString();
                messageDiv.appendChild(timestampDiv);
            }

            messagesEl.appendChild(messageDiv);
            messagesEl.scrollTop = messagesEl.scrollHeight;
        }

        function inspectVectors() {
            fetch('/api/vectors/inspect')
                .then(response => response.json())
                .then(data => {
                    document.getElementById('vector-inspection').textContent = 
                        JSON.stringify(data, null, 2);
                })
                .catch(error => {
                    console.error('Error inspecting vectors:', error);
                });
        }

        // Event listeners
        messageForm.addEventListener('submit', sendMessage);
        privateForm.addEventListener('submit', sendPrivateMessage);
        
        roomList.addEventListener('click', function(event) {
            const roomItem = event.target.closest('.room-item');
            if (roomItem) {
                const roomName = roomItem.dataset.room;
                subscribeToRoom(roomName);
                messagesEl.innerHTML = ''; // Clear messages when switching rooms
            }
        });

        // Initialize
        connect();
    </script>
</body>
</html>
```

Add Thymeleaf dependency if you haven't already:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-thymeleaf</artifactId>
</dependency>
```

Create `src/main/java/com/example/stompvectors/controller/HomeController.java`:

```java
package com.example.stompvectors.controller;

import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.GetMapping;

import java.security.Principal;

@Controller
public class HomeController {

    @GetMapping("/")
    public String index(Principal principal, Model model) {
        model.addAttribute("username", principal != null ? principal.getName() : "anonymous");
        return "index";
    }
}
```

## Step 9: Running and Testing the Demo

1. **Start the application:**
   ```bash
   ./mvnw spring-boot:run
   ```

2. **Open multiple browser windows:**
   - Window 1: Log in as `alice` (password: `password`)
   - Window 2: Log in as `bob` (password: `password`)
   - Window 3: Log in as `charlie` (password: `password`)

3. **Observe the vectors in action:**
   - **Connection**: Status shows "Connected" when WebSocket is established
   - **Session**: Each browser window has a unique session (check server logs)
   - **User**: Username displayed in the info panel
   - **Subscription**: Active subscriptions listed with their IDs
   - **Topic**: Switch between rooms to see topic-based routing
   - **Queue**: Send private messages to see queue-based routing

4. **Test vector interactions:**
   - Send messages in different rooms (Topic vector)
   - Send private messages (Queue + User vectors)
   - Open multiple tabs for the same user (multiple Sessions, one User)
   - Switch rooms (Subscription changes)
   - Check the "Inspect All Vectors" button to see server-side vector state

## Understanding Vector Interactions

### Vector Hierarchy

The vectors form a clear hierarchy:

```
Connection (Physical)
  └── Session (Application Context)
      └── User (Identity)
          └── Subscription (Interest)
              └── Destination (Topic/Queue)
```

### Common Patterns

1. **Broadcast to Room (Topic + Subscription)**:
   - Client subscribes to `/topic/room/general` with subscription ID `sub-1`
   - Server broadcasts to `/topic/room/general`
   - All clients with active subscriptions receive the message

2. **Private Message (Queue + User + Session)**:
   - Alice sends to `/app/private.send` with recipient "bob"
   - Server uses `convertAndSendToUser("bob", "/queue/private", message)`
   - Spring finds all of Bob's sessions and delivers to each
   - Bob's client receives on `/user/queue/private` (translated internally)

3. **Multiple Sessions per User**:
   - Alice opens two browser tabs (two Connections → two Sessions)
   - Both sessions share the same User (Principal: "alice")
   - When Bob sends Alice a private message, both tabs receive it
   - This demonstrates User vector abstracting over Session vector

## Troubleshooting

| Issue | Likely Cause | Solution |
|-------|--------------|----------|
| Messages not received | Wrong destination or subscription | Verify subscription destination matches server broadcast destination |
| Private messages not working | User destination not configured | Check `setUserDestinationPrefix("/user")` in WebSocketConfig |
| Multiple sessions not receiving | User destination translation issue | Ensure `convertAndSendToUser()` uses exact username from Principal |
| Subscription ID not visible | Client-side limitation | Subscription IDs are primarily server-side; use server logs or inspection API |

## Advanced Vector Manipulation

### Custom Headers for Vector Tracking

You can add custom headers to track vectors across the message flow:

```java
@MessageMapping("/chat.send")
public void sendMessage(@Payload ChatMessage message, StompHeaderAccessor accessor) {
    // Add custom header to track session
    accessor.setHeader("custom-session-id", accessor.getSessionId());
    messagingTemplate.convertAndSend("/topic/room/" + message.getRoom(), message);
}
```

### Programmatic Subscription Management

You can programmatically manage subscriptions server-side:

```java
@Autowired
private SimpUserRegistry userRegistry;

public void forceUnsubscribe(String username, String destination) {
    SimpUser user = userRegistry.getUser(username);
    if (user != null) {
        user.getSessions().forEach(session -> {
            session.getSubscriptions().stream()
                .filter(sub -> sub.getDestination().equals(destination))
                .forEach(SimpSubscription::unsubscribe);
        });
    }
}
```

## Looking Ahead

Understanding these six vectors unlocks advanced real-time features:

- **Presence systems**: Track users across sessions and connections
- **Load balancing**: Distribute connections across instances while maintaining user identity
- **Analytics**: Monitor subscription patterns and message routing
- **Security**: Implement per-subscription authorization
- **Scaling**: Use broker relays while maintaining vector semantics

In the next article, we'll explore how these vectors interact with external message brokers (RabbitMQ, ActiveMQ) and how the abstraction holds up under horizontal scaling.

## Recap

You've learned:

- ✅ **Topic**: Broadcast destinations for one-to-many messaging
- ✅ **Queue**: Point-to-point destinations for targeted delivery
- ✅ **User**: Identity layer that abstracts over sessions
- ✅ **Subscription**: Client interest tracking with unique IDs
- ✅ **Session**: Application context per connection
- ✅ **Connection**: Physical WebSocket transport

The key insight is that these vectors work together to provide a rich, layered abstraction for real-time messaging. By understanding each vector's role, you can build sophisticated routing logic, implement presence tracking, and scale your applications effectively.

---

**Test your understanding**: Open multiple browser windows, send messages in different rooms, send private messages, and use the inspection API to see how all six vectors interact in real-time.
