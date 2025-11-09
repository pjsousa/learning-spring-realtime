# Fine-Grained STOMP Control: Identifying Clients and Implementing Targeted Server-to-Client Messaging

*Estimated reading time: 25 minutes · Target word count: ~1,900*

## Overview

In previous articles, we've explored broadcasting messages to all subscribers (`/topic/public`) and sending private messages to specific users (`/user/queue/private`). But real-world applications require even more precision: sometimes you need to target a specific user, other times a specific connection, and occasionally you need to broadcast to everyone. This article delves into the crucial mechanisms Spring uses to handle security, identification, and precise message delivery in a STOMP-over-WebSocket environment.

We'll explain how the server reliably identifies the user (identity) and the connection (physical path) for incoming messages, and detail the distinct programming models available in Spring for publishing messages to all connected users versus targeting a single user across potentially multiple sessions, or even a single specific session.

By the end, you'll have a comprehensive notification system that demonstrates all three messaging patterns: broadcast, user-specific, and session-specific delivery. You'll understand when to use each approach and how to access both the authenticated user identity and the session ID in your controllers.

## Learning Goals

- Understand how Spring identifies users (Principal) versus connections (Session ID) in STOMP messages
- Master three messaging patterns: broadcast, user-specific, and session-specific delivery
- Use `SimpMessagingTemplate` for programmatic message sending
- Implement `@SendToUser` annotation with `broadcast=false` for session-specific replies
- Access `Principal` and `@Header("simpSessionId")` in controller methods
- Build a notification system that demonstrates all messaging patterns
- Create a browser client that can test all three delivery modes

## Routing Nuances: Distinguishing Users and Connections

Message routing in Spring STOMP relies on three primary channels: the `clientInboundChannel` (receiving from clients), the `clientOutboundChannel` (sending to clients), and the `brokerChannel` (internal and external message exchange).

### Distinguishing the User and Connection on Client Inbound Messages

When a client sends a SEND frame, the server identifies the sender based on information captured during the initial HTTP handshake:

#### 1. Identifying the User (Identity)

The STOMP messaging session begins with an HTTP request (the WebSocket handshake or SockJS transport requests). The authenticated user (`Principal`), typically retrieved from the HTTP session, is automatically associated with the subsequent WebSocket/SockJS session. This authenticated user identity is then applied as a user header on every subsequent STOMP Message flowing into the application.

In a method handling a client message (an `@MessageMapping` method), the authenticated user can be accessed via the `java.security.Principal` argument:

```java
@MessageMapping("/notify")
public void handleNotification(@Payload NotificationRequest request, Principal principal) {
    String username = principal != null ? principal.getName() : "anonymous";
    // Use username for user-specific routing
}
```

#### 2. Identifying the Connection (Session ID)

While the `Principal` identifies who the user is, the session ID identifies which physical connection or session they are currently using (e.g., distinguishing a user's phone session from their desktop session). The session ID is exposed in the inbound message headers. The server-side method can capture this specific connection context using the `@Header("simpSessionId")` annotation argument:

```java
@MessageMapping("/notify")
public void handleNotification(
    @Payload NotificationRequest request,
    Principal principal,
    @Header("simpSessionId") String sessionId
) {
    String username = principal != null ? principal.getName() : "anonymous";
    // Use sessionId for connection-specific routing
    log.info("Message from user {} on session {}", username, sessionId);
}
```

This distinction is crucial: a user can have multiple active sessions (multiple browser tabs, mobile app, desktop app), and you may need to target a specific session rather than all of them.

## Publishing Messages from the Server

The server publishes messages using the `SimpMessagingTemplate` (also known as "brokerMessagingTemplate"). Spring provides three distinct patterns for message delivery:

### 1. Publishing from Server to All Users (Broadcast)

To broadcast a message to a group of users who are all listening to a public destination, the server sends the message to a Topic:

**Mechanism**: Direct the message to a broker-managed destination, typically prefixed `/topic/`. The broker is responsible for finding all matching subscriptions associated with that topic and relaying the message to each one.

**Implementation**: Inject the `SimpMessagingTemplate` and use the standard `convertAndSend` method:

```java
@Service
public class NotificationService {
    private final SimpMessagingTemplate messagingTemplate;
    
    public NotificationService(SimpMessagingTemplate messagingTemplate) {
        this.messagingTemplate = messagingTemplate;
    }
    
    public void broadcastAnnouncement(String message) {
        Notification notification = new Notification("System", message, Instant.now());
        messagingTemplate.convertAndSend("/topic/announcements", notification);
    }
}
```

All clients subscribed to `/topic/announcements` will receive this message, regardless of who they are.

### 2. Publishing from Server to One Specific User

To send a message privately, the server must use User Destinations, which rely on the authenticated user's name:

**Mechanism**: Spring's `UserDestinationMessageHandler` automatically translates a logical destination (like `/user/queue/updates`) into a unique, session-specific destination (e.g., `/queue/updates-user123`). This ensures only the specified user receives the message, and it works even if that user has multiple sessions connected.

**Implementation (Using SimpMessagingTemplate)**: Inject the template and use `convertAndSendToUser()`. This method requires the username and the generic destination prefix:

```java
public void sendUserNotification(String username, String message) {
    Notification notification = new Notification("System", message, Instant.now());
    messagingTemplate.convertAndSendToUser(
        username,
        "/queue/notifications",
        notification
    );
}
```

The full destination pattern used by the server internally is typically `/user/{username}/queue/notifications`, which Spring then translates to session-specific queues for all of that user's active sessions.

**Implementation (Using Annotation)**: Inside an `@MessageMapping` method, the return value can be targeted back to the sender using `@SendToUser`. By default, this targets all sessions associated with that user. The `broadcast=false` attribute can be set to target only the specific session that sent the original message:

```java
@MessageMapping("/request.status")
@SendToUser("/queue/status")  // Sends to all sessions of the sender
public StatusResponse getStatus(@Payload StatusRequest request, Principal principal) {
    return new StatusResponse("Processing", Instant.now());
}

@MessageMapping("/request.status.specific")
@SendToUser(value = "/queue/status", broadcast = false)  // Sends only to this session
public StatusResponse getStatusSpecific(@Payload StatusRequest request, Principal principal) {
    return new StatusResponse("Processing", Instant.now());
}
```

### 3. Publishing to a Specific Session

Sometimes you need to target a specific connection, not just a user. This is useful when you want to send a message only to the session that triggered an action, even if the user has other active sessions.

**Implementation**: Use `SimpMessagingTemplate` with the session ID directly:

```java
public void sendToSession(String sessionId, String destination, Object payload) {
    messagingTemplate.convertAndSend(
        "/user/queue/session-specific",
        payload,
        Collections.singletonMap("simpSessionId", sessionId)
    );
}
```

However, a more common pattern is to use `@SendToUser` with `broadcast=false` in a controller method, which automatically targets the session that sent the message.

## Analogy: Mail Service Partitioning

Think of the STOMP system as a sophisticated corporate mail delivery service:

- **Topic** is like a company-wide email distribution list (e.g., "All Hands Announcements"). Everyone subscribed receives a copy.
- **User Destinations** are like having a personal mail slot labeled only with your employee ID, regardless of whether you pick up mail at your desk, the loading dock, or the break room. The mail system knows exactly where all your currently active receiving locations (sessions/connections) are, and delivers the private message only to them.
- **The Session/Connection** is the delivery driver's physical route and vehicle. You need to know the User (employee ID) and the Session (current vehicle route) to ensure the message gets delivered securely and quickly.

## Prerequisites and Project Setup

This article builds on concepts from Articles 6 (Security) and 7 (User Destinations). You should be familiar with:

- Spring Boot STOMP setup with `@EnableWebSocketMessageBroker`
- Authentication integration (Principal propagation)
- User destinations (`/user/**` prefixes)
- Basic `SimpMessagingTemplate` usage

### Dependencies

Start with a Spring Boot 3.2+ project including:

- `spring-boot-starter-websocket`
- `spring-boot-starter-web`
- `spring-boot-starter-security`
- `spring-boot-starter-thymeleaf` (for serving HTML)

For Maven, your `pom.xml` should include:

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
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-thymeleaf</artifactId>
    </dependency>
</dependencies>
```

## Step 1: Configure WebSocket with User Destinations

Create `src/main/java/com/example/finegrained/config/WebSocketConfig.java`:

```java
package com.example.finegrained.config;

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
        // Enable simple broker for topics and queues
        registry.enableSimpleBroker("/topic", "/queue");
        
        // Application destination prefix for client-to-server messages
        registry.setApplicationDestinationPrefixes("/app");
        
        // User destination prefix - enables /user/** routing
        registry.setUserDestinationPrefix("/user");
    }

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/ws")
                .setAllowedOriginPatterns("*")
                .withSockJS();
    }
}
```

## Step 2: Set Up Authentication

Create `src/main/java/com/example/finegrained/config/SecurityConfig.java`:

```java
package com.example.finegrained.config;

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
            .csrf(csrf -> csrf.ignoringRequestMatchers("/ws/**"));
        
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

## Step 3: Create Notification Models

Create `src/main/java/com/example/finegrained/model/Notification.java`:

```java
package com.example.finegrained.model;

import java.time.Instant;

public class Notification {
    private String from;
    private String message;
    private Instant timestamp;
    private String type; // "broadcast", "user", "session"
    
    public Notification() {
        this.timestamp = Instant.now();
    }
    
    public Notification(String from, String message, Instant timestamp) {
        this.from = from;
        this.message = message;
        this.timestamp = timestamp;
    }

    // Getters and setters
    public String getFrom() {
        return from;
    }

    public void setFrom(String from) {
        this.from = from;
    }

    public String getMessage() {
        return message;
    }

    public void setMessage(String message) {
        this.message = message;
    }

    public Instant getTimestamp() {
        return timestamp;
    }

    public void setTimestamp(Instant timestamp) {
        this.timestamp = timestamp;
    }

    public String getType() {
        return type;
    }

    public void setType(String type) {
        this.type = type;
    }
}
```

Create `src/main/java/com/example/finegrained/model/NotificationRequest.java`:

```java
package com.example.finegrained.model;

public class NotificationRequest {
    private String targetUser;
    private String message;
    private String deliveryType; // "broadcast", "user", "session"
    
    public NotificationRequest() {
    }

    public String getTargetUser() {
        return targetUser;
    }

    public void setTargetUser(String targetUser) {
        this.targetUser = targetUser;
    }

    public String getMessage() {
        return message;
    }

    public void setMessage(String message) {
        this.message = message;
    }

    public String getDeliveryType() {
        return deliveryType;
    }

    public void setDeliveryType(String deliveryType) {
        this.deliveryType = deliveryType;
    }
}
```

## Step 4: Implement the Notification Controller

This controller demonstrates all three messaging patterns and shows how to access both Principal and Session ID.

Create `src/main/java/com/example/finegrained/controller/NotificationController.java`:

```java
package com.example.finegrained.controller;

import com.example.finegrained.model.Notification;
import com.example.finegrained.model.NotificationRequest;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.messaging.handler.annotation.Header;
import org.springframework.messaging.handler.annotation.MessageMapping;
import org.springframework.messaging.handler.annotation.Payload;
import org.springframework.messaging.handler.annotation.SendToUser;
import org.springframework.messaging.simp.SimpMessagingTemplate;
import org.springframework.stereotype.Controller;

import java.security.Principal;
import java.time.Instant;

@Controller
public class NotificationController {

    private static final Logger log = LoggerFactory.getLogger(NotificationController.class);
    
    private final SimpMessagingTemplate messagingTemplate;

    public NotificationController(SimpMessagingTemplate messagingTemplate) {
        this.messagingTemplate = messagingTemplate;
    }

    /**
     * Demonstrates accessing Principal and Session ID
     * Sends a broadcast message to all subscribers
     */
    @MessageMapping("/notify.broadcast")
    public void handleBroadcastNotification(
            @Payload NotificationRequest request,
            Principal principal,
            @Header("simpSessionId") String sessionId) {
        
        String sender = principal != null ? principal.getName() : "anonymous";
        log.info("Broadcast request from user {} on session {}", sender, sessionId);
        
        Notification notification = new Notification(
            sender,
            request.getMessage(),
            Instant.now()
        );
        notification.setType("broadcast");
        
        // Broadcast to all subscribers of /topic/announcements
        messagingTemplate.convertAndSend("/topic/announcements", notification);
    }

    /**
     * Demonstrates user-specific messaging
     * Sends to all sessions of the target user
     */
    @MessageMapping("/notify.user")
    public void handleUserNotification(
            @Payload NotificationRequest request,
            Principal principal,
            @Header("simpSessionId") String sessionId) {
        
        String sender = principal != null ? principal.getName() : "anonymous";
        String targetUser = request.getTargetUser();
        
        if (targetUser == null || targetUser.isBlank()) {
            log.warn("User notification request from {} has no target user", sender);
            return;
        }
        
        log.info("User notification from {} to {} on session {}", sender, targetUser, sessionId);
        
        Notification notification = new Notification(
            sender,
            request.getMessage(),
            Instant.now()
        );
        notification.setType("user");
        
        // Send to all sessions of the target user
        messagingTemplate.convertAndSendToUser(
            targetUser,
            "/queue/notifications",
            notification
        );
    }

    /**
     * Demonstrates @SendToUser annotation
     * Returns a response to the sender (all their sessions by default)
     */
    @MessageMapping("/notify.self")
    @SendToUser("/queue/notifications")
    public Notification handleSelfNotification(
            @Payload NotificationRequest request,
            Principal principal,
            @Header("simpSessionId") String sessionId) {
        
        String sender = principal != null ? principal.getName() : "anonymous";
        log.info("Self notification request from {} on session {}", sender, sessionId);
        
        Notification notification = new Notification(
            "System",
            "You requested: " + request.getMessage(),
            Instant.now()
        );
        notification.setType("user");
        
        return notification;
    }

    /**
     * Demonstrates @SendToUser with broadcast=false
     * Returns a response only to the specific session that sent the message
     */
    @MessageMapping("/notify.self.session")
    @SendToUser(value = "/queue/notifications", broadcast = false)
    public Notification handleSessionSpecificNotification(
            @Payload NotificationRequest request,
            Principal principal,
            @Header("simpSessionId") String sessionId) {
        
        String sender = principal != null ? principal.getName() : "anonymous";
        log.info("Session-specific notification request from {} on session {}", sender, sessionId);
        
        Notification notification = new Notification(
            "System",
            "Session-specific response: " + request.getMessage() + " (Session: " + sessionId + ")",
            Instant.now()
        );
        notification.setType("session");
        
        return notification;
    }
}
```

**Key Points:**

1. **Principal Access**: The `Principal` parameter is automatically injected by Spring from the WebSocket session. It's `null` for unauthenticated sessions.
2. **Session ID Access**: The `@Header("simpSessionId")` annotation extracts the session ID from message headers.
3. **Broadcast Pattern**: `convertAndSend("/topic/announcements", ...)` sends to all subscribers.
4. **User Pattern**: `convertAndSendToUser(username, "/queue/notifications", ...)` sends to all sessions of that user.
5. **Session Pattern**: `@SendToUser(..., broadcast = false)` sends only to the session that triggered the message.

## Step 5: Add Server-Initiated Notifications

Create a service that periodically sends notifications to demonstrate server-initiated messaging:

Create `src/main/java/com/example/finegrained/service/NotificationService.java`:

```java
package com.example.finegrained.service;

import com.example.finegrained.model.Notification;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.messaging.simp.SimpMessagingTemplate;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Service;

import java.time.Instant;
import java.util.List;
import java.util.Random;

@Service
public class NotificationService {

    private static final Logger log = LoggerFactory.getLogger(NotificationService.class);
    
    private final SimpMessagingTemplate messagingTemplate;
    private final Random random = new Random();
    private final List<String> sampleUsers = List.of("alice", "bob", "charlie");
    
    private int broadcastCounter = 0;
    private int userCounter = 0;

    public NotificationService(SimpMessagingTemplate messagingTemplate) {
        this.messagingTemplate = messagingTemplate;
    }

    // Broadcast every 30 seconds
    @Scheduled(initialDelay = 15000, fixedRate = 30000)
    public void sendPeriodicBroadcast() {
        broadcastCounter++;
        Notification notification = new Notification(
            "System",
            "Periodic broadcast #" + broadcastCounter + ": Server announcement",
            Instant.now()
        );
        notification.setType("broadcast");
        
        log.info("Sending periodic broadcast #{}", broadcastCounter);
        messagingTemplate.convertAndSend("/topic/announcements", notification);
    }

    // Send to a random user every 45 seconds
    @Scheduled(initialDelay = 20000, fixedRate = 45000)
    public void sendPeriodicUserNotification() {
        String recipient = sampleUsers.get(random.nextInt(sampleUsers.size()));
        userCounter++;
        
        Notification notification = new Notification(
            "System",
            "Personal notification #" + userCounter + ": You have an update!",
            Instant.now()
        );
        notification.setType("user");
        
        log.info("Sending periodic user notification #{} to {}", userCounter, recipient);
        messagingTemplate.convertAndSendToUser(
            recipient,
            "/queue/notifications",
            notification
        );
    }
}
```

Enable scheduling in your main application class:

```java
package com.example.finegrained;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.scheduling.annotation.EnableScheduling;

@SpringBootApplication
@EnableScheduling
public class FineGrainedApplication {

    public static void main(String[] args) {
        SpringApplication.run(FineGrainedApplication.class, args);
    }
}
```

## Step 6: Create the Browser Client

Create `src/main/java/com/example/finegrained/controller/HomeController.java`:

```java
package com.example.finegrained.controller;

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

Create `src/main/resources/templates/index.html`:

```html
<!DOCTYPE html>
<html lang="en" xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>Fine-Grained STOMP Control</title>
    <style>
        body {
            font-family: system-ui, sans-serif;
            margin: 2rem;
            max-width: 1200px;
            margin-left: auto;
            margin-right: auto;
        }
        header {
            display: flex;
            justify-content: space-between;
            align-items: center;
            margin-bottom: 2rem;
            padding-bottom: 1rem;
            border-bottom: 2px solid #e0e0e0;
        }
        .status {
            padding: 0.5rem 1rem;
            border-radius: 4px;
            font-weight: 600;
        }
        .status.connected {
            background: #d4edda;
            color: #155724;
        }
        .status.disconnected {
            background: #f8d7da;
            color: #721c24;
        }
        .container {
            display: grid;
            grid-template-columns: 1fr 1fr;
            gap: 2rem;
            margin-bottom: 2rem;
        }
        section {
            border: 1px solid #ddd;
            border-radius: 8px;
            padding: 1rem;
        }
        h2 {
            margin-top: 0;
            font-size: 1.25rem;
        }
        .messages {
            height: 300px;
            overflow-y: auto;
            border: 1px solid #eee;
            padding: 0.5rem;
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
        .message.broadcast {
            border-left-color: #28a745;
        }
        .message.user {
            border-left-color: #ffc107;
        }
        .message.session {
            border-left-color: #dc3545;
        }
        .message-from {
            font-weight: 600;
            color: #007bff;
        }
        .message-content {
            margin-top: 0.25rem;
        }
        .message-type {
            font-size: 0.75rem;
            color: #666;
            margin-top: 0.25rem;
            font-weight: 600;
        }
        .message-timestamp {
            font-size: 0.75rem;
            color: #666;
            margin-top: 0.25rem;
        }
        form {
            display: flex;
            flex-direction: column;
            gap: 0.5rem;
        }
        input[type="text"], select {
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
        .info-box {
            background: #e7f3ff;
            border: 1px solid #b3d9ff;
            border-radius: 4px;
            padding: 1rem;
            margin-bottom: 1rem;
        }
    </style>
</head>
<body>
    <header>
        <h1>Fine-Grained STOMP Control Demo</h1>
        <div>
            <span>Signed in as: <strong th:text="${username}"></strong></span>
            <form action="/logout" method="post" style="display: inline; margin-left: 1rem;">
                <input type="hidden" th:name="${_csrf.parameterName}" th:value="${_csrf.token}">
                <button type="submit" style="background: #dc3545;">Logout</button>
            </form>
        </div>
    </header>

    <div class="info-box">
        <strong>Testing Instructions:</strong>
        <ul style="margin: 0.5rem 0;">
            <li>Open multiple browser windows/tabs and log in as different users</li>
            <li>Connect in all windows to see broadcast messages</li>
            <li>Use "Send User Notification" to target a specific user (all their sessions receive it)</li>
            <li>Use "Send Session Notification" to target only the current session</li>
            <li>Watch for periodic server-initiated broadcasts and user notifications</li>
        </ul>
    </div>

    <div style="margin-bottom: 1rem;">
        <button id="connect-btn">Connect</button>
        <button id="disconnect-btn" disabled>Disconnect</button>
        <span id="status" class="status disconnected">Disconnected</span>
    </div>

    <div class="container">
        <section>
            <h2>Send Notifications</h2>
            <form id="broadcast-form">
                <label>Broadcast (all users):</label>
                <input type="text" id="broadcast-message" placeholder="Message for everyone" required>
                <button type="submit">Send Broadcast</button>
            </form>
            <form id="user-form">
                <label>User Notification (all sessions of user):</label>
                <input type="text" id="user-target" placeholder="Target username" required>
                <input type="text" id="user-message" placeholder="Message" required>
                <button type="submit">Send User Notification</button>
            </form>
            <form id="session-form">
                <label>Session Notification (this session only):</label>
                <input type="text" id="session-message" placeholder="Message" required>
                <button type="submit">Send Session Notification</button>
            </form>
        </section>

        <section>
            <h2>Broadcast Messages</h2>
            <div id="broadcast-messages" class="messages"></div>
        </section>
    </div>

    <section>
        <h2>Personal Notifications</h2>
        <div id="notifications" class="messages"></div>
    </section>

    <script src="https://cdn.jsdelivr.net/npm/sockjs-client@1/dist/sockjs.min.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/stompjs@2.3.3/lib/stomp.min.js"></script>
    <script>
        const username = /*[[${username}]]*/ 'anonymous';
        let stompClient = null;

        const connectBtn = document.getElementById('connect-btn');
        const disconnectBtn = document.getElementById('disconnect-btn');
        const statusEl = document.getElementById('status');
        const broadcastMessagesEl = document.getElementById('broadcast-messages');
        const notificationsEl = document.getElementById('notifications');
        const broadcastForm = document.getElementById('broadcast-form');
        const userForm = document.getElementById('user-form');
        const sessionForm = document.getElementById('session-form');

        function setConnected(connected) {
            connectBtn.disabled = connected;
            disconnectBtn.disabled = !connected;
            statusEl.textContent = connected ? 'Connected' : 'Disconnected';
            statusEl.className = `status ${connected ? 'connected' : 'disconnected'}`;
        }

        function connect() {
            const socket = new SockJS('/ws');
            stompClient = Stomp.over(socket);
            stompClient.debug = null;

            stompClient.connect({}, function(frame) {
                console.log('Connected: ' + frame);
                setConnected(true);

                // Subscribe to broadcast messages
                stompClient.subscribe('/topic/announcements', function(message) {
                    const notification = JSON.parse(message.body);
                    appendMessage(notification, broadcastMessagesEl, 'broadcast');
                });

                // Subscribe to personal notifications
                stompClient.subscribe('/user/queue/notifications', function(message) {
                    const notification = JSON.parse(message.body);
                    appendMessage(notification, notificationsEl, notification.type || 'user');
                });

                console.log('Subscribed to /topic/announcements and /user/queue/notifications');
            }, function(error) {
                console.error('STOMP error: ' + error);
                setConnected(false);
            });
        }

        function disconnect() {
            if (stompClient !== null) {
                stompClient.disconnect(function() {
                    setConnected(false);
                    console.log('Disconnected');
                });
            }
        }

        function sendBroadcast(event) {
            event.preventDefault();
            const message = document.getElementById('broadcast-message').value.trim();
            if (!message || !stompClient || !stompClient.connected) {
                return;
            }
            stompClient.send('/app/notify.broadcast', {}, JSON.stringify({ message }));
            document.getElementById('broadcast-message').value = '';
        }

        function sendUserNotification(event) {
            event.preventDefault();
            const targetUser = document.getElementById('user-target').value.trim();
            const message = document.getElementById('user-message').value.trim();
            if (!targetUser || !message || !stompClient || !stompClient.connected) {
                return;
            }
            stompClient.send('/app/notify.user', {}, JSON.stringify({
                targetUser: targetUser,
                message: message
            }));
            document.getElementById('user-target').value = '';
            document.getElementById('user-message').value = '';
        }

        function sendSessionNotification(event) {
            event.preventDefault();
            const message = document.getElementById('session-message').value.trim();
            if (!message || !stompClient || !stompClient.connected) {
                return;
            }
            stompClient.send('/app/notify.self.session', {}, JSON.stringify({ message }));
            document.getElementById('session-message').value = '';
        }

        function appendMessage(notification, container, type) {
            const messageDiv = document.createElement('div');
            messageDiv.className = `message ${type}`;

            const fromDiv = document.createElement('div');
            fromDiv.className = 'message-from';
            fromDiv.textContent = `From: ${notification.from}`;
            messageDiv.appendChild(fromDiv);

            const contentDiv = document.createElement('div');
            contentDiv.className = 'message-content';
            contentDiv.textContent = notification.message;
            messageDiv.appendChild(contentDiv);

            const typeDiv = document.createElement('div');
            typeDiv.className = 'message-type';
            typeDiv.textContent = `Type: ${type.toUpperCase()}`;
            messageDiv.appendChild(typeDiv);

            if (notification.timestamp) {
                const timestampDiv = document.createElement('div');
                timestampDiv.className = 'message-timestamp';
                timestampDiv.textContent = new Date(notification.timestamp).toLocaleString();
                messageDiv.appendChild(timestampDiv);
            }

            container.appendChild(messageDiv);
            container.scrollTop = container.scrollHeight;
        }

        connectBtn.addEventListener('click', connect);
        disconnectBtn.addEventListener('click', disconnect);
        broadcastForm.addEventListener('submit', sendBroadcast);
        userForm.addEventListener('submit', sendUserNotification);
        sessionForm.addEventListener('submit', sendSessionNotification);
    </script>
</body>
</html>
```

## Step 7: Testing the Implementation

### Manual Testing Steps

1. **Start the application:**
   ```bash
   ./mvnw spring-boot:run
   ```

2. **Open multiple browser windows:**
   - Window 1: Log in as `alice` (password: `password`)
   - Window 2: Log in as `bob` (password: `password`)
   - Window 3: Log in as `alice` again (to test multiple sessions)

3. **Connect in all windows:**
   - Click "Connect" in each window to establish STOMP sessions.

4. **Test broadcast messages:**
   - In Window 1, send a broadcast message.
   - Observe that both Window 2 and Window 3 receive it (all connected users).

5. **Test user-specific messages:**
   - In Window 1, send a user notification targeting "bob".
   - Observe that Window 2 (bob) receives it, but Window 3 (alice's second session) does not.
   - Send a user notification targeting "alice".
   - Observe that both Window 1 and Window 3 receive it (all of alice's sessions).

6. **Test session-specific messages:**
   - In Window 1, send a session notification.
   - Observe that only Window 1 receives it, not Window 3 (alice's other session).

7. **Wait for periodic notifications:**
   - Within 30 seconds, all windows should receive a broadcast.
   - Within 45 seconds, a random user should receive a personal notification.

### Verification Points

- ✅ Broadcast messages appear in all connected clients
- ✅ User notifications appear only for the target user (all their sessions)
- ✅ Session notifications appear only in the session that sent the request
- ✅ Server-initiated periodic notifications work correctly
- ✅ Principal and Session ID are correctly logged in server logs

## Understanding the Patterns

### When to Use Broadcast (`/topic/**`)

- System-wide announcements
- Public chat rooms
- Live updates that everyone should see
- Real-time dashboards with shared data

### When to Use User Destinations (`/user/**`)

- Private messages between users
- Personalized notifications
- User-specific data updates
- Multi-session support (user has phone + desktop)

### When to Use Session-Specific (`broadcast=false`)

- Confirmation messages for actions
- Session-specific state updates
- Responses to requests that should only go to the requester
- Temporary session data

## Troubleshooting

| Issue | Likely Cause | Solution |
|-------|--------------|----------|
| Messages not received | Client not subscribed to correct destination | Ensure subscription uses `/topic/...` for broadcasts, `/user/queue/...` for user messages |
| User notifications not working | Username mismatch | Verify `convertAndSendToUser()` uses exact username from Principal |
| Session-specific not working | Missing `broadcast=false` | Check `@SendToUser` annotation has `broadcast=false` |
| Principal is null | Not authenticated | Ensure user is logged in before connecting WebSocket |

## Advanced Patterns

### 1. Conditional Delivery Based on User State

Check if a user is online before sending:

```java
@Autowired
private SimpUserRegistry userRegistry;

public void sendIfOnline(String username, String destination, Object payload) {
    SimpUser user = userRegistry.getUser(username);
    if (user != null && user.getSessions().size() > 0) {
        messagingTemplate.convertAndSendToUser(username, destination, payload);
    } else {
        // Queue for later or log
        log.info("User {} is offline, message queued", username);
    }
}
```

### 2. Combining Patterns

You can combine patterns in a single operation:

```java
// Send to a specific user AND broadcast a notification
messagingTemplate.convertAndSendToUser(username, "/queue/private", privateMessage);
messagingTemplate.convertAndSend("/topic/activity", new ActivityNotification(username, "sent a message"));
```

### 3. Accessing Session Attributes

Store and retrieve session-specific data:

```java
@MessageMapping("/store.data")
public void storeData(@Payload DataRequest request, StompHeaderAccessor accessor) {
    Map<String, Object> sessionAttributes = accessor.getSessionAttributes();
    sessionAttributes.put("customKey", request.getData());
    // Later, retrieve it in another handler
}
```

## Looking Ahead

Fine-grained control over message routing enables sophisticated real-time features:

- **Multi-device synchronization**: Update all of a user's devices simultaneously
- **Session-aware applications**: Provide different experiences per connection
- **Presence systems**: Track which sessions belong to which users
- **Collaborative editing**: Distinguish between user actions and session-specific UI state

In future articles, we'll explore presence tracking, session management, and advanced routing patterns that build on these fundamentals.

## Recap

You've implemented:

- ✅ Access to `Principal` (user identity) in controller methods
- ✅ Access to `@Header("simpSessionId")` (connection identity) in controller methods
- ✅ Broadcast messaging using `convertAndSend("/topic/...", ...)`
- ✅ User-specific messaging using `convertAndSendToUser(username, "/queue/...", ...)`
- ✅ Session-specific messaging using `@SendToUser(..., broadcast = false)`
- ✅ A comprehensive notification system demonstrating all patterns
- ✅ A browser client for testing all messaging modes

The key insight is understanding when to use each pattern: broadcasts for public data, user destinations for personalized messages across all sessions, and session-specific delivery for connection-scoped responses. This fine-grained control is essential for building production-ready real-time applications.

---

**Test your implementation**: Run the app, log in as different users in multiple windows, and experiment with all three messaging patterns to see how they behave differently.
