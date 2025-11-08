# Fine-Grained STOMP Control: Identifying Clients and Implementing Targeted Server-to-Client Messaging

*Estimated reading time: 25 minutes · Target word count: ~1,900*

## Overview

In previous articles, we've explored broadcasting messages to all subscribers (`/topic/public`) and sending private messages to specific users (`/user/queue/...`). But real-world applications often need more granular control: distinguishing between a user's identity and their physical connection, targeting specific sessions, and choosing the right messaging pattern for each scenario.

This article delves into the crucial mechanisms Spring uses to handle security, identification, and precise message delivery in a STOMP-over-WebSocket environment. We explain how the server reliably identifies the user (identity) and the connection (physical path) for incoming messages, and detail the distinct programming models available in Spring for publishing messages to all connected users versus targeting a single user across potentially multiple sessions.

By the end, you'll understand the difference between user identity and session identity, know when to use broadcast topics versus user destinations, and be able to implement session-specific messaging when needed. We'll build a practical example: a notification system that can send messages to all users, to a specific user (all their sessions), or to a single session.

## Learning Goals

- Understand how Spring identifies users (Principal) vs connections (Session ID) in STOMP messages
- Access `Principal` and session ID in `@MessageMapping` methods
- Distinguish between broadcasting to all users and targeting specific users
- Use `@SendToUser` annotation with `broadcast` attribute for session-specific delivery
- Implement targeted messaging using `SimpMessagingTemplate.convertAndSendToUser()`
- Choose the appropriate messaging pattern for different use cases
- Build a practical notification system demonstrating all messaging patterns

## Why This Matters

Consider a real-world scenario: Alice logs into your application on her laptop and her phone. Both devices establish separate WebSocket sessions, but both are authenticated as "alice". When Bob sends Alice a private message, you want both devices to receive it. However, if Alice is viewing a settings page on her laptop and you want to notify her that "Settings saved successfully," you might only want to send that to the specific session that triggered the action, not both devices.

Spring provides three distinct mechanisms for this:

1. **Broadcast to all users** (`/topic/**`):** Everyone subscribed receives the message
2. **Target a specific user** (`/user/**`):** All sessions for that user receive the message
3. **Target a specific session** (`@SendToUser` with `broadcast=false`):** Only one session receives the message

Understanding when and how to use each pattern is essential for building sophisticated real-time applications.

## Prerequisites and Project Setup

This article assumes familiarity with:
- Basic Spring Boot STOMP setup (Articles 1-2)
- Authentication with Spring Security (Article 6)
- User destinations (Article 7)
- Maven or Gradle project structure

### Dependencies

Start with a Spring Boot 3.2+ project including:

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
        <artifactId>spring-boot-starter-validation</artifactId>
    </dependency>
</dependencies>
```

## Step 1: Configure WebSocket with User Destinations

We'll set up a WebSocket configuration that supports both broadcast topics and user-specific destinations.

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
        // Enable simple broker for topics (broadcast) and queues (point-to-point)
        registry.enableSimpleBroker("/topic", "/queue");
        
        // Application destination prefix for client-to-server messages
        registry.setApplicationDestinationPrefixes("/app");
        
        // User destination prefix for user-specific messaging
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

This configuration enables:
- `/topic/**` for broadcasting to all subscribers
- `/queue/**` for point-to-point messaging
- `/user/**` for user-specific destinations that automatically resolve to session-specific queues

## Step 2: Set Up Minimal Security

We need authenticated users to demonstrate identity-based messaging. Create a simple security configuration:

`src/main/java/com/example/finegrained/config/SecurityConfig.java`:

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

This provides three test users (alice, bob, charlie) all with password "password".

## Step 3: Understanding User vs Connection Identification

Before implementing our notification system, let's clarify how Spring identifies users and connections.

### Identifying the User (Identity)

When a client sends a STOMP `SEND` frame, the server identifies the sender based on information captured during the initial HTTP handshake:

1. **HTTP Handshake:** The STOMP messaging session begins with an HTTP request (the WebSocket handshake or SockJS transport requests).

2. **Principal Association:** The authenticated user (`Principal`), typically retrieved from the HTTP session, is automatically associated with the subsequent WebSocket/SockJS session.

3. **User Header:** This authenticated user identity is then applied as a `simpUser` header on every subsequent STOMP message flowing into the application.

4. **Accessing in Controllers:** In a method handling a client message (an `@MessageMapping` method), the authenticated user can be accessed via the `java.security.Principal` argument.

### Identifying the Connection (Session ID)

While the `Principal` identifies **who** the user is, the session ID identifies **which physical connection** or session they are currently using. This is crucial for distinguishing:
- A user's phone session from their desktop session
- Multiple browser tabs from the same user
- Different devices or applications connected simultaneously

The session ID is exposed in the inbound message headers. The server-side method can capture this specific connection context using the `@Header("simpSessionId")` annotation argument.

## Step 4: Model Notification Messages

Let's create DTOs for our notification system:

`src/main/java/com/example/finegrained/model/Notification.java`:

```java
package com.example.finegrained.model;

import java.time.Instant;

public class Notification {
    
    public enum NotificationType {
        BROADCAST,      // Sent to all users
        USER_SPECIFIC,  // Sent to all sessions of a specific user
        SESSION_SPECIFIC // Sent to a single session
    }
    
    private NotificationType type;
    private String from;
    private String to;  // Username for user-specific, session ID for session-specific
    private String message;
    private Instant timestamp;
    
    public Notification() {
        this.timestamp = Instant.now();
    }
    
    public Notification(NotificationType type, String from, String to, String message) {
        this.type = type;
        this.from = from;
        this.to = to;
        this.message = message;
        this.timestamp = Instant.now();
    }

    // Getters and setters
    public NotificationType getType() {
        return type;
    }

    public void setType(NotificationType type) {
        this.type = type;
    }

    public String getFrom() {
        return from;
    }

    public void setFrom(String from) {
        this.from = from;
    }

    public String getTo() {
        return to;
    }

    public void setTo(String to) {
        this.to = to;
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
}
```

## Step 5: Implement the Notification Controller

Now we'll create a controller that demonstrates all three messaging patterns and shows how to access both `Principal` and session ID:

`src/main/java/com/example/finegrained/controller/NotificationController.java`:

```java
package com.example.finegrained.controller;

import com.example.finegrained.model.Notification;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.messaging.handler.annotation.Header;
import org.springframework.messaging.handler.annotation.MessageMapping;
import org.springframework.messaging.handler.annotation.Payload;
import org.springframework.messaging.handler.annotation.SendToUser;
import org.springframework.messaging.simp.SimpMessagingTemplate;
import org.springframework.messaging.simp.annotation.SendToUser;
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
     * Example 1: Broadcasting to all users
     * Any client subscribed to /topic/announcements receives this message
     */
    @MessageMapping("/notify.broadcast")
    public void broadcastNotification(
            @Payload Notification notification,
            Principal principal) {
        
        String sender = principal != null ? principal.getName() : "System";
        notification.setFrom(sender);
        notification.setType(Notification.NotificationType.BROADCAST);
        notification.setTimestamp(Instant.now());
        
        log.info("Broadcasting notification from {}: {}", sender, notification.getMessage());
        
        // Send to all subscribers of /topic/announcements
        messagingTemplate.convertAndSend("/topic/announcements", notification);
    }

    /**
     * Example 2: Sending to a specific user (all their sessions)
     * Uses convertAndSendToUser() which delivers to all sessions for that user
     */
    @MessageMapping("/notify.user")
    public void sendToUser(
            @Payload Notification notification,
            Principal principal) {
        
        String sender = principal != null ? principal.getName() : "System";
        notification.setFrom(sender);
        notification.setType(Notification.NotificationType.USER_SPECIFIC);
        notification.setTimestamp(Instant.now());
        
        String recipient = notification.getTo();
        if (recipient == null || recipient.isBlank()) {
            log.warn("User notification from {} has no recipient", sender);
            return;
        }
        
        log.info("Sending user-specific notification from {} to {}: {}", 
                sender, recipient, notification.getMessage());
        
        // This delivers to ALL sessions for the recipient user
        messagingTemplate.convertAndSendToUser(
            recipient,
            "/queue/notifications",
            notification
        );
    }

    /**
     * Example 3: Sending to a specific session only
     * Uses @SendToUser with broadcast=false to target only the session that sent the message
     */
    @MessageMapping("/notify.session")
    @SendToUser(value = "/queue/notifications", broadcast = false)
    public Notification sendToSession(
            @Payload Notification notification,
            Principal principal,
            @Header("simpSessionId") String sessionId) {
        
        String sender = principal != null ? principal.getName() : "System";
        notification.setFrom(sender);
        notification.setTo(sessionId); // Store session ID for reference
        notification.setType(Notification.NotificationType.SESSION_SPECIFIC);
        notification.setTimestamp(Instant.now());
        
        log.info("Sending session-specific notification from {} to session {}: {}", 
                sender, sessionId, notification.getMessage());
        
        // @SendToUser with broadcast=false sends only to the session that triggered this method
        return notification;
    }

    /**
     * Example 4: Demonstrating Principal and Session ID access
     * This method shows how to access both user identity and connection identity
     */
    @MessageMapping("/notify.info")
    public void requestSessionInfo(
            @Payload String request,
            Principal principal,
            @Header("simpSessionId") String sessionId) {
        
        String username = principal != null ? principal.getName() : "anonymous";
        
        Notification info = new Notification(
            Notification.NotificationType.SESSION_SPECIFIC,
            "System",
            sessionId,
            String.format("User: %s, Session: %s, Request: %s", username, sessionId, request)
        );
        
        log.info("Session info requested by user {} on session {}", username, sessionId);
        
        // Send back to the requesting session only
        messagingTemplate.convertAndSendToUser(
            username,
            "/queue/info",
            info
        );
    }
}
```

**Key Points:**

1. **Broadcast (`/topic/announcements`):** Uses `convertAndSend()` to send to all subscribers regardless of user identity.

2. **User-Specific (`convertAndSendToUser()`):** Delivers to all sessions for a specific user. If Alice has two browser tabs open, both receive the message.

3. **Session-Specific (`@SendToUser` with `broadcast=false`):** Returns a value that is sent only to the session that triggered the method. This is useful for acknowledgments, confirmations, or session-specific feedback.

4. **Accessing Both Identities:** The `requestSessionInfo` method demonstrates accessing both `Principal` (user identity) and `@Header("simpSessionId")` (connection identity) in the same method.

## Step 6: Understanding @SendToUser Annotation

The `@SendToUser` annotation provides a declarative way to send messages back to the user who sent the original message. It has two important attributes:

- **`value`**: The destination path (relative to the user prefix)
- **`broadcast`**: 
  - `true` (default): Sends to all sessions for that user
  - `false`: Sends only to the specific session that triggered the method

### When to Use Each Pattern

**Broadcast (`/topic/**`):**
- System announcements
- Public chat messages
- Live scoreboards
- Any message that should reach all connected clients

**User-Specific (`/user/**` with `broadcast=true` or `convertAndSendToUser()`):**
- Private messages
- User notifications
- Personalized updates
- Any message that should reach a specific user across all their devices/sessions

**Session-Specific (`@SendToUser` with `broadcast=false`):**
- Form submission confirmations
- Action acknowledgments
- Session-specific state updates
- Any message that should only go to the session that triggered it

## Step 7: Create a Service for Server-Initiated Messages

Let's add a service that demonstrates server-initiated messaging patterns:

`src/main/java/com/example/finegrained/service/NotificationService.java`:

```java
package com.example.finegrained.service;

import com.example.finegrained.model.Notification;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.messaging.simp.SimpMessagingTemplate;
import org.springframework.messaging.simp.user.SimpUser;
import org.springframework.messaging.simp.user.SimpUserRegistry;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Service;

import java.time.Instant;
import java.util.List;
import java.util.Random;

@Service
public class NotificationService {

    private static final Logger log = LoggerFactory.getLogger(NotificationService.class);
    
    private final SimpMessagingTemplate messagingTemplate;
    private final SimpUserRegistry userRegistry;
    private final Random random = new Random();
    private final List<String> sampleUsers = List.of("alice", "bob", "charlie");
    
    private int broadcastCounter = 0;
    private int userNotificationCounter = 0;

    public NotificationService(SimpMessagingTemplate messagingTemplate, 
                              SimpUserRegistry userRegistry) {
        this.messagingTemplate = messagingTemplate;
        this.userRegistry = userRegistry;
    }

    /**
     * Broadcast a system announcement every 60 seconds
     */
    @Scheduled(initialDelay = 10000, fixedRate = 60000)
    public void sendPeriodicBroadcast() {
        broadcastCounter++;
        
        Notification announcement = new Notification(
            Notification.NotificationType.BROADCAST,
            "System",
            null,
            "System Announcement #" + broadcastCounter + ": All systems operational!"
        );
        
        log.info("Broadcasting system announcement #{}", broadcastCounter);
        
        // Send to all subscribers
        messagingTemplate.convertAndSend("/topic/announcements", announcement);
    }

    /**
     * Send a notification to a random user every 30 seconds
     */
    @Scheduled(initialDelay = 15000, fixedRate = 30000)
    public void sendPeriodicUserNotification() {
        String recipient = sampleUsers.get(random.nextInt(sampleUsers.size()));
        userNotificationCounter++;
        
        // Check if user is online
        SimpUser user = userRegistry.getUser(recipient);
        if (user == null || user.getSessions().isEmpty()) {
            log.debug("User {} is not online, skipping notification", recipient);
            return;
        }
        
        Notification notification = new Notification(
            Notification.NotificationType.USER_SPECIFIC,
            "System",
            recipient,
            "Personal Notification #" + userNotificationCounter + ": You have a new update!"
        );
        
        log.info("Sending user notification #{} to {}", userNotificationCounter, recipient);
        
        // This will deliver to ALL sessions for the recipient
        messagingTemplate.convertAndSendToUser(
            recipient,
            "/queue/notifications",
            notification
        );
    }
}
```

Don't forget to enable scheduling in your main application class:

```java
@SpringBootApplication
@EnableScheduling
public class FineGrainedApplication {
    public static void main(String[] args) {
        SpringApplication.run(FineGrainedApplication.class, args);
    }
}
```

## Step 8: Build the Browser Test Client

Create a comprehensive HTML client that demonstrates all messaging patterns:

`src/main/resources/templates/index.html`:

```html
<!DOCTYPE html>
<html lang="en" xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>Fine-Grained STOMP Control Demo</title>
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
            padding: 1.5rem;
        }
        h2 {
            margin-top: 0;
            font-size: 1.25rem;
        }
        .messages {
            height: 250px;
            overflow-y: auto;
            border: 1px solid #eee;
            padding: 0.75rem;
            margin-bottom: 1rem;
            background: #fafafa;
            border-radius: 4px;
        }
        .message {
            margin-bottom: 0.75rem;
            padding: 0.75rem;
            background: white;
            border-radius: 4px;
            border-left: 4px solid #007bff;
        }
        .message.broadcast {
            border-left-color: #28a745;
        }
        .message.user-specific {
            border-left-color: #ffc107;
        }
        .message.session-specific {
            border-left-color: #dc3545;
        }
        .message-header {
            font-weight: 600;
            margin-bottom: 0.25rem;
            color: #007bff;
        }
        .message-content {
            margin-top: 0.5rem;
        }
        .message-meta {
            font-size: 0.75rem;
            color: #666;
            margin-top: 0.25rem;
        }
        form {
            display: flex;
            flex-direction: column;
            gap: 0.75rem;
        }
        input[type="text"] {
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
        .info-panel {
            background: #e7f3ff;
            padding: 1rem;
            border-radius: 4px;
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

    <div class="info-panel">
        <strong>Session Information:</strong>
        <div id="session-info">Not connected</div>
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
                <label>Broadcast to All Users:</label>
                <input type="text" id="broadcast-message" placeholder="Message for everyone">
                <button type="submit">Broadcast</button>
            </form>
            <form id="user-form">
                <label>Send to Specific User:</label>
                <input type="text" id="user-recipient" placeholder="Username (alice, bob, charlie)">
                <input type="text" id="user-message" placeholder="Message">
                <button type="submit">Send to User</button>
            </form>
            <form id="session-form">
                <label>Session-Specific (to this session only):</label>
                <input type="text" id="session-message" placeholder="Message for this session">
                <button type="submit">Send to Session</button>
            </form>
            <form id="info-form">
                <label>Request Session Info:</label>
                <input type="text" id="info-request" placeholder="Request text">
                <button type="submit">Get Info</button>
            </form>
        </section>

        <section>
            <h2>System Announcements (Broadcast)</h2>
            <div id="broadcast-messages" class="messages"></div>
        </section>
    </div>

    <div class="container">
        <section>
            <h2>User Notifications (All My Sessions)</h2>
            <div id="user-messages" class="messages"></div>
        </section>

        <section>
            <h2>Session-Specific Messages (This Session Only)</h2>
            <div id="session-messages" class="messages"></div>
        </section>
    </div>

    <script src="https://cdn.jsdelivr.net/npm/sockjs-client@1/dist/sockjs.min.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/stompjs@2.3.3/lib/stomp.min.js"></script>
    <script>
        const username = /*[[${username}]]*/ 'anonymous';
        let stompClient = null;
        let sessionId = null;

        const connectBtn = document.getElementById('connect-btn');
        const disconnectBtn = document.getElementById('disconnect-btn');
        const statusEl = document.getElementById('status');
        const sessionInfoEl = document.getElementById('session-info');

        const broadcastMessagesEl = document.getElementById('broadcast-messages');
        const userMessagesEl = document.getElementById('user-messages');
        const sessionMessagesEl = document.getElementById('session-messages');

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

                // Extract session ID from connection headers if available
                // Note: Session ID is typically available server-side, 
                // but we can request it via the info endpoint
                requestSessionInfo('Connection established');

                // Subscribe to broadcast announcements
                stompClient.subscribe('/topic/announcements', function(message) {
                    const notification = JSON.parse(message.body);
                    appendMessage(notification, broadcastMessagesEl, 'broadcast');
                });

                // Subscribe to user-specific notifications
                // This will receive messages sent to this user across all sessions
                stompClient.subscribe('/user/queue/notifications', function(message) {
                    const notification = JSON.parse(message.body);
                    appendMessage(notification, userMessagesEl, 'user-specific');
                });

                // Subscribe to session-specific messages
                // This receives messages sent only to this specific session
                stompClient.subscribe('/user/queue/notifications', function(message) {
                    // Note: Session-specific messages also come through /user/queue/notifications
                    // The server determines delivery based on @SendToUser(broadcast=false)
                    const notification = JSON.parse(message.body);
                    if (notification.type === 'SESSION_SPECIFIC') {
                        appendMessage(notification, sessionMessagesEl, 'session-specific');
                    }
                });

                // Subscribe to info responses
                stompClient.subscribe('/user/queue/info', function(message) {
                    const info = JSON.parse(message.body);
                    sessionInfoEl.textContent = info.message;
                    appendMessage(info, sessionMessagesEl, 'session-specific');
                });

                console.log('Subscribed to all destinations');
            }, function(error) {
                console.error('STOMP error: ' + error);
                setConnected(false);
            });
        }

        function disconnect() {
            if (stompClient !== null) {
                stompClient.disconnect(function() {
                    setConnected(false);
                    sessionInfoEl.textContent = 'Not connected';
                    console.log('Disconnected');
                });
            }
        }

        function requestSessionInfo(requestText) {
            if (!stompClient || !stompClient.connected) return;
            stompClient.send('/app/notify.info', {}, JSON.stringify(requestText));
        }

        function sendBroadcast(event) {
            event.preventDefault();
            const message = document.getElementById('broadcast-message').value.trim();
            if (!message || !stompClient || !stompClient.connected) return;

            const notification = {
                message: message
            };

            stompClient.send('/app/notify.broadcast', {}, JSON.stringify(notification));
            document.getElementById('broadcast-message').value = '';
        }

        function sendToUser(event) {
            event.preventDefault();
            const recipient = document.getElementById('user-recipient').value.trim();
            const message = document.getElementById('user-message').value.trim();
            if (!recipient || !message || !stompClient || !stompClient.connected) return;

            const notification = {
                to: recipient,
                message: message
            };

            stompClient.send('/app/notify.user', {}, JSON.stringify(notification));
            document.getElementById('user-recipient').value = '';
            document.getElementById('user-message').value = '';
        }

        function sendToSession(event) {
            event.preventDefault();
            const message = document.getElementById('session-message').value.trim();
            if (!message || !stompClient || !stompClient.connected) return;

            const notification = {
                message: message
            };

            stompClient.send('/app/notify.session', {}, JSON.stringify(notification));
            document.getElementById('session-message').value = '';
        }

        function appendMessage(notification, container, type) {
            const messageDiv = document.createElement('div');
            messageDiv.className = `message ${type}`;

            const header = document.createElement('div');
            header.className = 'message-header';
            header.textContent = `${notification.from || 'System'} → ${notification.to || 'All'}`;
            messageDiv.appendChild(header);

            const content = document.createElement('div');
            content.className = 'message-content';
            content.textContent = notification.message;
            messageDiv.appendChild(content);

            if (notification.timestamp) {
                const meta = document.createElement('div');
                meta.className = 'message-meta';
                meta.textContent = `Type: ${notification.type} • ${new Date(notification.timestamp).toLocaleString()}`;
                messageDiv.appendChild(meta);
            }

            container.appendChild(messageDiv);
            container.scrollTop = container.scrollHeight;
        }

        connectBtn.addEventListener('click', connect);
        disconnectBtn.addEventListener('click', disconnect);
        document.getElementById('broadcast-form').addEventListener('submit', sendBroadcast);
        document.getElementById('user-form').addEventListener('submit', sendToUser);
        document.getElementById('session-form').addEventListener('submit', sendToSession);
        document.getElementById('info-form').addEventListener('submit', function(e) {
            e.preventDefault();
            const request = document.getElementById('info-request').value.trim();
            if (request) {
                requestSessionInfo(request);
                document.getElementById('info-request').value = '';
            }
        });
    </script>
</body>
</html>
```

Add Thymeleaf dependency and create a simple controller:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-thymeleaf</artifactId>
</dependency>
```

`src/main/java/com/example/finegrained/controller/HomeController.java`:

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

## Step 9: Testing the Implementation

### Manual Testing Steps

1. **Start the application:**
   ```bash
   ./mvnw spring-boot:run
   ```

2. **Open multiple browser windows:**
   - Window 1: Log in as `alice` (password: `password`)
   - Window 2: Log in as `alice` again (to test multiple sessions)
   - Window 3: Log in as `bob` (password: `password`)

3. **Connect in all windows:**
   - Click "Connect" in each window

4. **Test broadcast messaging:**
   - In Window 1 (alice), send a broadcast message
   - Observe that all three windows receive it (both alice sessions and bob's session)

5. **Test user-specific messaging:**
   - In Window 3 (bob), send a message to user "alice"
   - Observe that both Window 1 and Window 2 (alice's sessions) receive it
   - Window 3 (bob) does not receive it

6. **Test session-specific messaging:**
   - In Window 1 (alice), send a session-specific message
   - Observe that only Window 1 receives it, not Window 2 (alice's other session)

7. **Test server-initiated messages:**
   - Wait 10 seconds for the first system broadcast
   - Wait 15 seconds for the first user notification
   - Observe that broadcasts go to all windows, while user notifications go only to the targeted user's sessions

### Verification Points

- ✅ Broadcast messages appear in all connected clients
- ✅ User-specific messages appear in all sessions for that user
- ✅ Session-specific messages appear only in the triggering session
- ✅ Server-initiated broadcasts work correctly
- ✅ Server-initiated user notifications work correctly
- ✅ Session info request returns correct user and session information

## Understanding the Mail Service Analogy

Think of the STOMP system as a sophisticated corporate mail delivery service:

- **Topic (`/topic/**`)**: Like a company-wide email distribution list (e.g., "All Hands Announcements"). Everyone subscribed receives a copy, regardless of who they are.

- **User Destinations (`/user/**`)**: Like having a personal mail slot labeled only with your employee ID, regardless of whether you pick up mail at your desk, the loading dock, or the break room. The mail system knows exactly where all your currently active receiving locations (sessions/connections) are, and delivers the private message only to them.

- **Session/Connection**: The delivery driver's physical route and vehicle. You need to know both the User (employee ID) and the Session (current vehicle route) to ensure the message gets delivered securely and quickly to a specific location.

This analogy helps clarify when to use each pattern:
- Use **topics** for public announcements
- Use **user destinations** for personal messages that should reach all of a user's devices
- Use **session-specific** delivery for acknowledgments or context-specific updates

## Advanced Patterns

### Conditional Delivery Based on User State

You can check if a user is online before sending:

```java
@Autowired
private SimpUserRegistry userRegistry;

public void sendIfOnline(String username, String destination, Object payload) {
    SimpUser user = userRegistry.getUser(username);
    if (user != null && !user.getSessions().isEmpty()) {
        messagingTemplate.convertAndSendToUser(username, destination, payload);
    } else {
        log.info("User {} is offline, message queued for later", username);
        // Queue message for later delivery
    }
}
```

### Error Handling for Unknown Users

If you send to a user that doesn't exist or has no active sessions, Spring silently discards the message. Add validation:

```java
if (userRegistry.getUser(recipient) == null) {
    // Send error back to sender
    messagingTemplate.convertAndSendToUser(
        sender,
        "/queue/errors",
        new ErrorMessage("Recipient not found: " + recipient)
    );
    return;
}
```

## Troubleshooting

| Issue | Likely Cause | Solution |
|-------|--------------|----------|
| Session-specific messages not working | `broadcast=false` not set | Ensure `@SendToUser(broadcast=false)` is used |
| User messages only reaching one session | Using session ID instead of username | Use `convertAndSendToUser()` with username, not session ID |
| Principal is null | Not authenticated | Ensure Spring Security is configured and user is logged in |
| Session ID not accessible | Missing `@Header` annotation | Use `@Header("simpSessionId")` to access session ID |

## Looking Ahead

Fine-grained STOMP control enables sophisticated real-time features:

- **Multi-device synchronization**: Keep all user's devices in sync
- **Context-aware notifications**: Send session-specific confirmations
- **Presence-aware messaging**: Only send to online users
- **Selective broadcasting**: Target specific user groups

In future articles, we'll explore:
- Presence tracking and user registries
- Channel interceptors for message filtering
- Performance optimization for high-frequency messaging
- Integration with external message brokers

## Recap

You've learned:

- ✅ How Spring identifies users (`Principal`) vs connections (`Session ID`)
- ✅ Accessing both identities in `@MessageMapping` methods
- ✅ Three messaging patterns: broadcast, user-specific, session-specific
- ✅ Using `@SendToUser` with `broadcast` attribute
- ✅ When to use each messaging pattern
- ✅ Building a practical notification system

The key insight is understanding the distinction between user identity and session identity, and choosing the right messaging pattern based on your delivery requirements. This knowledge enables you to build sophisticated, real-time applications with precise control over message delivery.

---

**Test your implementation**: Run the app, log in as different users in multiple browser windows, and experiment with all three messaging patterns to see how messages are routed differently based on the delivery mechanism you choose.
