# Differentiate Clients: Sending Private Messages using User Destinations (/user/)

*Estimated reading time: 20 minutes · Target word count: ~1,800*

## Overview

In previous articles, we built chat rooms where messages broadcast to all subscribers via `/topic/public`. But real-world applications need **personalized communication**: direct messages between users, private notifications, per-user dashboards, and session-specific updates. This article introduces Spring's **User Destinations** mechanism, which enables sending messages to specific users without tracking session IDs manually.

We'll configure a custom user destination prefix, demonstrate how clients subscribe to generic destinations like `/user/queue/updates`, and show how Spring transparently translates these into unique, session-specific addresses. On the server side, we'll use `SimpMessagingTemplate.convertAndSendToUser()` to target messages to named users, enabling features like private messaging, personalized alerts, and user-specific state synchronization.

By the end of this article, you'll have a working private messaging system where the server can send messages directly to individual users, and clients can subscribe to their personal message queues without exposing internal session management details.

## Learning Goals

- Understand the difference between broadcast topics (`/topic/**`) and user-specific destinations (`/user/**`).
- Configure a custom user destination prefix in Spring's WebSocket configuration.
- Use `SimpMessagingTemplate.convertAndSendToUser()` to send messages to specific users.
- Build client-side subscriptions to user destinations that automatically resolve to session-specific queues.
- Implement a private messaging feature where users can send and receive direct messages.
- Grasp how Spring handles multiple sessions per user and session management behind the scenes.

## Why User Destinations Matter

Consider a typical web application: Alice logs in and opens the site in two browser tabs. Both tabs establish separate WebSocket sessions, but Alice is the same authenticated user in both. When Bob sends Alice a private message, we want **both** tabs to receive it, not just one. Similarly, if the server needs to notify Alice about a system event, it should reach all her active sessions.

Spring's User Destinations solve this elegantly:

1. **Generic Subscription**: The client subscribes to `/user/queue/messages` without knowing its session ID.
2. **Automatic Translation**: Spring translates this to a session-specific destination like `/queue/messages-user{alice}` internally.
3. **Multi-Session Delivery**: When you call `convertAndSendToUser("alice", ...)`, Spring finds all sessions for that user and delivers to each.
4. **Session Abstraction**: The application code never needs to track session IDs manually.

This abstraction keeps your code clean while providing powerful routing capabilities.

## Prerequisites and Project Setup

This article assumes you're comfortable with:

- Basic Spring Boot STOMP setup (from Articles 1-2)
- Authentication integration (Article 6 covered Spring Security with Principal propagation)
- Maven or Gradle project structure

If you haven't worked through the security article, you can still follow along—we'll include authentication setup, but the core user destination concepts apply even with anonymous sessions (though user names become session identifiers).

### Dependencies

Start with a Spring Boot 3.2+ project including:

- `spring-boot-starter-websocket`
- `spring-boot-starter-web`
- `spring-boot-starter-security` (for authenticated users)

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
</dependencies>
```

## Step 1: Configure WebSocket with User Destination Support

The foundation is the WebSocket configuration, where we set the user destination prefix. This prefix tells Spring which destinations should be translated into user-specific queues.

Create `src/main/java/com/example/userdestinations/config/WebSocketConfig.java`:

```java
package com.example.userdestinations.config;

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
        
        // User destination prefix - this is the key setting
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

**Critical points:**

- `setUserDestinationPrefix("/user")` enables user destination translation. When a client subscribes to `/user/queue/updates`, Spring internally maps this to `/queue/updates-user{username}` for that user's sessions.
- The broker supports both `/topic` (broadcast) and `/queue` (point-to-point) destinations. User destinations typically use `/queue` to ensure delivery semantics.
- The endpoint `/ws` is where clients connect. We allow all origins for simplicity; restrict this in production.

### Custom User Destination Prefix (Optional)

If you need a different prefix (e.g., `/secured/user` for organizational reasons), simply change:

```java
registry.setUserDestinationPrefix("/secured/user");
```

Clients would then subscribe to `/secured/user/queue/updates`. The prefix is arbitrary as long as it's consistent across client and server code.

## Step 2: Set Up Authentication (Minimal Security)

To demonstrate user-specific messaging, we need authenticated users. We'll create a simple security configuration with in-memory users, building on Article 6's approach.

Create `src/main/java/com/example/userdestinations/config/SecurityConfig.java`:

```java
package com.example.userdestinations.config;

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

This provides three test users (alice, bob, charlie) all with password "password". The WebSocket endpoint `/ws/**` is excluded from CSRF checks to allow the handshake to proceed.

## Step 3: Model Private Messages

We'll create simple message DTOs for sending and receiving private messages.

Create `src/main/java/com/example/userdestinations/model/PrivateMessage.java`:

```java
package com.example.userdestinations.model;

import java.time.Instant;

public class PrivateMessage {
    
    private String from;
    private String to;
    private String content;
    private Instant timestamp;
    
    public PrivateMessage() {
        this.timestamp = Instant.now();
    }
    
    public PrivateMessage(String from, String to, String content) {
        this.from = from;
        this.to = to;
        this.content = content;
        this.timestamp = Instant.now();
    }

    // Getters and setters
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

    public String getContent() {
        return content;
    }

    public void setContent(String content) {
        this.content = content;
    }

    public Instant getTimestamp() {
        return timestamp;
    }

    public void setTimestamp(Instant timestamp) {
        this.timestamp = timestamp;
    }
}
```

This payload carries sender, recipient, message content, and a timestamp. The server will enrich it with the authenticated sender's name to prevent impersonation.

## Step 4: Implement the Private Messaging Controller

The controller handles incoming private messages and uses `SimpMessagingTemplate.convertAndSendToUser()` to deliver them to the recipient.

Create `src/main/java/com/example/userdestinations/controller/PrivateMessageController.java`:

```java
package com.example.userdestinations.controller;

import com.example.userdestinations.model.PrivateMessage;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.messaging.handler.annotation.MessageMapping;
import org.springframework.messaging.handler.annotation.Payload;
import org.springframework.messaging.simp.SimpMessagingTemplate;
import org.springframework.stereotype.Controller;

import java.security.Principal;
import java.time.Instant;

@Controller
public class PrivateMessageController {

    private static final Logger log = LoggerFactory.getLogger(PrivateMessageController.class);
    
    private final SimpMessagingTemplate messagingTemplate;

    public PrivateMessageController(SimpMessagingTemplate messagingTemplate) {
        this.messagingTemplate = messagingTemplate;
    }

    @MessageMapping("/private.send")
    public void sendPrivateMessage(
            @Payload PrivateMessage message,
            Principal principal) {
        
        String sender = principal != null ? principal.getName() : "anonymous";
        
        // Override the 'from' field with the authenticated user to prevent spoofing
        message.setFrom(sender);
        message.setTimestamp(Instant.now());
        
        String recipient = message.getTo();
        if (recipient == null || recipient.isBlank()) {
            log.warn("Private message from {} has no recipient", sender);
            return;
        }
        
        log.info("Sending private message from {} to {}", sender, recipient);
        
        // Send to the recipient's user destination
        // Spring will deliver to all sessions for this user
        messagingTemplate.convertAndSendToUser(
            recipient,
            "/queue/private",
            message
        );
    }
}
```

**How `convertAndSendToUser()` works:**

1. **First parameter**: The username (e.g., "bob"). Spring looks up all active sessions for this user.
2. **Second parameter**: The destination path relative to the user prefix. `/queue/private` becomes `/user/{username}/queue/private`, which Spring then translates to session-specific queues.
3. **Third parameter**: The message payload to send.

If Bob has two browser tabs open, both will receive the message because Spring finds all of Bob's sessions and delivers to each.

### Sending Notifications to Sender

You might want to send a confirmation back to the sender. Update the method:

```java
@MessageMapping("/private.send")
public void sendPrivateMessage(
        @Payload PrivateMessage message,
        Principal principal) {
    
    String sender = principal != null ? principal.getName() : "anonymous";
    message.setFrom(sender);
    message.setTimestamp(Instant.now());
    
    String recipient = message.getTo();
    if (recipient == null || recipient.isBlank()) {
        log.warn("Private message from {} has no recipient", sender);
        return;
    }
    
    log.info("Sending private message from {} to {}", sender, recipient);
    
    // Send to recipient
    messagingTemplate.convertAndSendToUser(recipient, "/queue/private", message);
    
    // Also send a copy to the sender for their sent-messages view
    messagingTemplate.convertAndSendToUser(sender, "/queue/private", message);
}
```

This pattern is useful for building "sent messages" and "received messages" inboxes.

## Step 5: Add Server-Initiated Notifications (Optional)

Beyond user-to-user messaging, servers often need to send notifications. Create a service that demonstrates this:

Create `src/main/java/com/example/userdestinations/service/NotificationService.java`:

```java
package com.example.userdestinations.service;

import com.example.userdestinations.model.PrivateMessage;
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
    
    private int notificationCounter = 0;

    public NotificationService(SimpMessagingTemplate messagingTemplate) {
        this.messagingTemplate = messagingTemplate;
    }

    // Send a notification every 30 seconds to a random user
    @Scheduled(initialDelay = 10000, fixedRate = 30000)
    public void sendPeriodicNotification() {
        String recipient = sampleUsers.get(random.nextInt(sampleUsers.size()));
        notificationCounter++;
        
        PrivateMessage notification = new PrivateMessage(
            "System",
            recipient,
            "Notification #" + notificationCounter + ": You have a new system update!"
        );
        
        log.info("Sending system notification to {}", recipient);
        
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
package com.example.userdestinations;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.scheduling.annotation.EnableScheduling;

@SpringBootApplication
@EnableScheduling
public class UserDestinationsApplication {

    public static void main(String[] args) {
        SpringApplication.run(UserDestinationsApplication.class, args);
    }
}
```

This service sends periodic notifications to demonstrate server-initiated user messaging, independent of any user action.

## Step 6: Create the Browser Client

The client needs to:
1. Connect and authenticate (via form login).
2. Subscribe to `/user/queue/private` for private messages.
3. Subscribe to `/user/queue/notifications` for system notifications.
4. Provide a UI to send private messages and display incoming ones.

Create `src/main/java/com/example/userdestinations/controller/HomeController.java` to serve the HTML:

```java
package com.example.userdestinations.controller;

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

Add Thymeleaf dependency for templating:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-thymeleaf</artifactId>
</dependency>
```

Create `src/main/resources/templates/index.html`:

```html
<!DOCTYPE html>
<html lang="en" xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>Private Messaging with User Destinations</title>
    <style>
        body {
            font-family: system-ui, sans-serif;
            margin: 2rem;
            max-width: 900px;
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
        .message.system {
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
    </style>
</head>
<body>
    <header>
        <h1>Private Messaging Demo</h1>
        <div>
            <span>Signed in as: <strong th:text="${username}"></strong></span>
            <form action="/logout" method="post" style="display: inline; margin-left: 1rem;">
                <input type="hidden" th:name="${_csrf.parameterName}" th:value="${_csrf.token}">
                <button type="submit" style="background: #dc3545;">Logout</button>
            </form>
        </div>
    </header>

    <div style="margin-bottom: 1rem;">
        <button id="connect-btn">Connect</button>
        <button id="disconnect-btn" disabled>Disconnect</button>
        <span id="status" class="status disconnected">Disconnected</span>
    </div>

    <div class="container">
        <section>
            <h2>Send Private Message</h2>
            <form id="message-form">
                <input type="text" id="recipient" placeholder="Recipient username" required>
                <input type="text" id="message-input" placeholder="Message" required>
                <button type="submit">Send</button>
            </form>
        </section>

        <section>
            <h2>System Notifications</h2>
            <div id="notifications" class="messages"></div>
        </section>
    </div>

    <section>
        <h2>Private Messages</h2>
        <div id="messages" class="messages"></div>
    </section>

    <script src="https://cdn.jsdelivr.net/npm/sockjs-client@1/dist/sockjs.min.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/stompjs@2.3.3/lib/stomp.min.js"></script>
    <script>
        const username = /*[[${username}]]*/ 'anonymous';
        let stompClient = null;

        const connectBtn = document.getElementById('connect-btn');
        const disconnectBtn = document.getElementById('disconnect-btn');
        const statusEl = document.getElementById('status');
        const messagesEl = document.getElementById('messages');
        const notificationsEl = document.getElementById('notifications');
        const messageForm = document.getElementById('message-form');
        const recipientInput = document.getElementById('recipient');
        const messageInput = document.getElementById('message-input');

        function setConnected(connected) {
            connectBtn.disabled = connected;
            disconnectBtn.disabled = !connected;
            statusEl.textContent = connected ? 'Connected' : 'Disconnected';
            statusEl.className = `status ${connected ? 'connected' : 'disconnected'}`;
        }

        function connect() {
            const socket = new SockJS('/ws');
            stompClient = Stomp.over(socket);
            stompClient.debug = null; // Suppress console logs

            stompClient.connect({}, function(frame) {
                console.log('Connected: ' + frame);
                setConnected(true);

                // Subscribe to private messages
                // Spring automatically translates /user/queue/private to session-specific destination
                stompClient.subscribe('/user/queue/private', function(message) {
                    const msg = JSON.parse(message.body);
                    appendMessage(msg, messagesEl);
                });

                // Subscribe to notifications
                stompClient.subscribe('/user/queue/notifications', function(message) {
                    const notification = JSON.parse(message.body);
                    appendMessage(notification, notificationsEl, true);
                });

                console.log('Subscribed to /user/queue/private and /user/queue/notifications');
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

        function sendMessage(event) {
            event.preventDefault();
            const recipient = recipientInput.value.trim();
            const content = messageInput.value.trim();

            if (!recipient || !content || !stompClient || !stompClient.connected) {
                return;
            }

            const message = {
                to: recipient,
                content: content
            };

            stompClient.send('/app/private.send', {}, JSON.stringify(message));
            recipientInput.value = '';
            messageInput.value = '';
        }

        function appendMessage(msg, container, isSystem) {
            const messageDiv = document.createElement('div');
            messageDiv.className = `message ${isSystem ? 'system' : ''}`;

            const senderSpan = document.createElement('div');
            senderSpan.className = 'message-sender';
            senderSpan.textContent = `${msg.from} → ${msg.to}`;
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

            container.appendChild(messageDiv);
            container.scrollTop = container.scrollHeight;
        }

        connectBtn.addEventListener('click', connect);
        disconnectBtn.addEventListener('click', disconnect);
        messageForm.addEventListener('submit', sendMessage);
    </script>
</body>
</html>
```

**Key client-side behavior:**

1. **Subscription to `/user/queue/private`**: The client subscribes to this generic path. Spring automatically resolves it to the session-specific queue for the authenticated user.
2. **Sending via `/app/private.send`**: Messages are sent to the application destination, which routes to the controller.
3. **Multiple subscriptions**: We subscribe to both private messages and notifications to demonstrate different user destination queues.

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

4. **Send a private message:**
   - In Window 1 (alice), enter "bob" as recipient and send a message.
   - Observe that Window 2 (bob) receives it, but Window 3 (alice's second session) does not (unless you implement sender notifications as shown earlier).

5. **Test multiple sessions:**
   - If you enabled sender notifications, Window 3 (alice's second session) should also receive a copy.
   - This demonstrates that `convertAndSendToUser()` delivers to all sessions for a user.

6. **Wait for system notifications:**
   - Within 30 seconds, a random user should receive a system notification.
   - This shows server-initiated user messaging.

### Verification Points

- ✅ Private messages are delivered only to the intended recipient.
- ✅ Messages appear in the correct UI section.
- ✅ System notifications appear in the notifications section.
- ✅ Multiple sessions for the same user both receive messages (if sender notifications are enabled).

## Understanding the Translation Mechanism

Under the hood, Spring performs destination translation:

1. **Client subscribes to**: `/user/queue/private`
2. **Spring resolves username**: From the authenticated `Principal` in the WebSocket session
3. **Spring creates session-specific destination**: `/queue/private-user{alice}` (or similar internal format)
4. **When `convertAndSendToUser("alice", "/queue/private", msg)` is called**:
   - Spring looks up all sessions for user "alice"
   - For each session, it resolves the session-specific destination
   - It sends the message to each resolved destination

This abstraction means your application code doesn't need to know about session IDs, WebSocket session management, or routing details—Spring handles it all.

## Advanced Patterns

### 1. User-Specific Topics

You can also use topics with user destinations for broadcast-like behavior per user:

```java
messagingTemplate.convertAndSendToUser("alice", "/topic/updates", message);
```

Clients subscribe to `/user/topic/updates`. This is less common than queues but useful for per-user event streams.

### 2. Conditional Delivery Based on User State

You might want to check if a user is online before sending:

```java
@Autowired
private SimpUserRegistry userRegistry;

public void sendIfOnline(String username, String destination, Object payload) {
    SimpUser user = userRegistry.getUser(username);
    if (user != null && user.getSessions().size() > 0) {
        messagingTemplate.convertAndSendToUser(username, destination, payload);
    } else {
        // Queue for later or log
        log.info("User {} is offline, message not sent", username);
    }
}
```

### 3. Error Handling for Unknown Users

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
| Messages not received | Client subscribed to wrong destination | Ensure subscription uses `/user/queue/...` format |
| Messages received by wrong user | Username mismatch | Verify `convertAndSendToUser()` uses exact username from Principal |
| Multiple sessions not receiving | User destination not configured | Check `setUserDestinationPrefix()` in WebSocketConfig |
| Anonymous users not working | No Principal in session | Either require authentication or use session ID as identifier |

## Looking Ahead

User destinations unlock powerful real-time features:

- **Direct messaging systems**: Chat applications with private conversations
- **Personalized dashboards**: Stock tickers, news feeds per user
- **Notification systems**: Alerts, reminders, activity updates
- **Collaborative editing**: Per-user cursor positions and change notifications

In the next article, we'll explore presence tracking and user registries to build features like "users online" indicators and session management dashboards.

## Recap

You've implemented:

- ✅ User destination prefix configuration (`/user`)
- ✅ Private messaging with `convertAndSendToUser()`
- ✅ Client subscriptions to user-specific queues
- ✅ Server-initiated user notifications
- ✅ A working browser client for testing

The key insight is that user destinations abstract away session management, letting you think in terms of users rather than WebSocket sessions. This makes building personalized, real-time features straightforward and maintainable.

---

**Test your implementation**: Run the app, log in as different users, send private messages, and observe how Spring routes messages to the correct recipients across multiple sessions.
