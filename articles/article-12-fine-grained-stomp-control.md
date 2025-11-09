# Fine-Grained STOMP Control: Identifying Clients and Implementing Targeted Server-to-Client Messaging

*Estimated reading time: 22 minutes · Target word count: ~1,900*

## Overview

Articles 6 and 7 introduced authentication and user destinations. With those building blocks in place, we can now deliver truly personalized experiences: send a broadcast to everyone, whisper to a single user across all of their devices, or aim a message at one specific session. This article explains how Spring’s STOMP support keeps track of **who** is connected (the authenticated `Principal`) and **how** they are connected (the WebSocket session ID). We examine the routing channels that drive Spring’s messaging engine, show how to capture identity and connection metadata inside `@MessageMapping` methods, and demonstrate three server-to-client delivery patterns using `SimpMessagingTemplate` and `@SendToUser`.

You’ll finish with a runnable Spring Boot project and a plain HTML client that lets you test broadcasts, per-user messages, and per-session replies side by side.

## Learning Goals

- Understand the roles of `clientInboundChannel`, `clientOutboundChannel`, and `brokerChannel` in Spring’s STOMP pipeline.
- Capture the authenticated user (`Principal`) and the physical WebSocket session (`simpSessionId`) from inbound STOMP frames.
- Implement broadcasts with `convertAndSend("/topic/...")`.
- Target a logical user—across multiple active sessions—with `convertAndSendToUser()` and `@SendToUser`.
- Target a single session using `simpSessionId` plus `SimpUserRegistry`.
- Build a minimal HTML/JavaScript control panel that exercises all three messaging patterns.

## Why Identity and Session Context Matter

In a real system, the server must distinguish between *who* is sending a message and *which connection* delivered it:

- **Identity (`Principal`)** answers “Which user account does this message belong to?” Spring associates the `Principal` with the WebSocket session during the HTTP handshake (form login, OAuth2, etc.) and injects it into every STOMP message.
- **Session (`simpSessionId`)** answers “Which physical WebSocket/SockJS connection carried this frame?” A single user might be online from a phone and a laptop; you can choose to reach all of those sessions or only the one that triggered an action.

Ignoring this distinction leads to bugs like leaking private data to the wrong person or notifying a user’s inactive sessions. Understanding both identifiers unlocks fine-grained control.

## Mail Service Analogy

Think of your STOMP infrastructure as a corporate mailroom:

- A `/topic/announcements` broadcast behaves like the all-hands mailing list. Drop mail in the slot and everyone who opted in receives a copy.
- `/user/queue/alerts` behaves like your personal mail slot labeled with your employee ID. No matter how many desks you occupy, the sorter copies each message into every active slot for that ID.
- A session-specific reply is like handing a memo directly to the courier who just arrived at your desk—nobody else sees it, not even your other desks.

Spring’s messaging template and annotations provide the hooks to choose the correct “delivery lane” for each scenario.

## Prerequisites and Project Setup

### What You’ll Build

We will extend the secured STOMP application from Articles 6 and 7:

- Form login protects the WebSocket endpoint.
- Clients connect via `/ws` using SockJS + STOMP.
- An `IdentityAwareController` illustrates how to inspect inbound metadata.
- A `TargetedNotificationService` sends broadcast, per-user, and per-session updates.
- A simple HTML dashboard allows you to trigger each mode and verify where the messages land.

### Dependencies

Create or update `pom.xml` with the following dependencies (Spring Boot 3.2+):

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

If you are using Gradle, include the equivalent starters.

### Project Structure

```
src/main/java/com/example/targeted
 ├─ TargetedMessagingApplication.java
 ├─ config/
 │   ├─ WebSocketConfig.java
 │   └─ SecurityConfig.java
 ├─ controller/
 │   ├─ IdentityAwareController.java
 │   └─ AdminNotificationController.java
 ├─ model/
 │   ├─ BroadcastMessage.java
 │   └─ TargetedMessageRequest.java
 └─ service/
     └─ TargetedNotificationService.java
src/main/resources/templates/
 └─ dashboard.html
src/main/resources/static/
 └─ css/app.css (optional styling)
```

Feel free to adjust packages to match your naming conventions.

## Step 1: Inspect the Message Routing Pipeline

Before writing code, revisit Spring’s channels:

- **`clientInboundChannel`** receives frames sent from browsers. Interceptors here can log the `Principal`, session ID, and STOMP command before your `@MessageMapping` methods execute.
- **`clientOutboundChannel`** sends frames from your application back to clients. Interceptors can customize headers before they are pushed to the broker and down to the socket.
- **`brokerChannel`** links your application-side handlers with the broker (an in-memory simple broker, or an external relay such as RabbitMQ).

### Visualizing the Flow

1. A client sends `SEND` → `clientInboundChannel`.
2. Spring resolves the `Principal` and `simpSessionId`, then delivers to an `@MessageMapping`.
3. Your controller may call `SimpMessagingTemplate.convertAndSend*()` (or return a value annotated with `@SendTo` / `@SendToUser`).
4. The broker delivers the message to subscribers via `clientOutboundChannel`.

Knowing where to intercept the flow will help you capture identifiers and choose the right delivery method.

## Step 2: Configure WebSocket Infrastructure with Session Awareness

Create `WebSocketConfig.java`:

```java
package com.example.targeted.config;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.context.annotation.Configuration;
import org.springframework.messaging.Message;
import org.springframework.messaging.MessageChannel;
import org.springframework.messaging.simp.config.ChannelRegistration;
import org.springframework.messaging.simp.config.MessageBrokerRegistry;
import org.springframework.messaging.simp.stomp.StompCommand;
import org.springframework.messaging.simp.stomp.StompHeaderAccessor;
import org.springframework.messaging.support.ChannelInterceptor;
import org.springframework.web.socket.config.annotation.EnableWebSocketMessageBroker;
import org.springframework.web.socket.config.annotation.StompEndpointRegistry;
import org.springframework.web.socket.config.annotation.WebSocketMessageBrokerConfigurer;

import java.security.Principal;

@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {

    private static final Logger log = LoggerFactory.getLogger(WebSocketConfig.class);

    @Override
    public void configureMessageBroker(MessageBrokerRegistry registry) {
        registry.enableSimpleBroker("/topic", "/queue");
        registry.setApplicationDestinationPrefixes("/app");
        registry.setUserDestinationPrefix("/user");
    }

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/ws")
                .setAllowedOriginPatterns("*")
                .withSockJS();
    }

    @Override
    public void configureClientInboundChannel(ChannelRegistration registration) {
        registration.interceptors(logInboundFrames());
    }

    private ChannelInterceptor logInboundFrames() {
        return new ChannelInterceptor() {
            @Override
            public Message<?> preSend(Message<?> message, MessageChannel channel) {
                StompHeaderAccessor accessor = StompHeaderAccessor.wrap(message);

                if (StompCommand.SEND.equals(accessor.getCommand())
                        || StompCommand.SUBSCRIBE.equals(accessor.getCommand())) {

                    Principal user = accessor.getUser();
                    String sessionId = accessor.getSessionId();
                    log.debug("Inbound {} from user={} session={} destination={}",
                            accessor.getCommand(),
                            user != null ? user.getName() : "anonymous",
                            sessionId,
                            accessor.getDestination());
                }

                return message;
            }
        };
    }
}
```

Key highlights:

- We enable both `/topic` and `/queue` destinations so we can mix broadcast and point-to-point patterns.
- `setUserDestinationPrefix("/user")` enables the `/user/**` pattern explored in Article 7.
- The inbound channel interceptor logs the user and session identifiers for `SEND` and `SUBSCRIBE` frames, making it easy to observe routing behaviour while testing.

## Step 3: Secure the Handshake and Provide Test Accounts

Reuse the minimal security configuration from earlier articles or create `SecurityConfig.java`:

```java
package com.example.targeted.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.Customizer;
import org.springframework.security.config.annotation.method.configuration.EnableMethodSecurity;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.core.userdetails.User;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.security.provisioning.InMemoryUserDetailsManager;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
@EnableMethodSecurity
public class SecurityConfig {

    @Bean
    SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(authorize -> authorize
                .requestMatchers("/css/**", "/js/**", "/login", "/favicon.ico").permitAll()
                .anyRequest().authenticated()
            )
            .formLogin(Customizer.withDefaults())
            .logout(logout -> logout.logoutSuccessUrl("/login?logout").permitAll())
            .csrf(csrf -> csrf.ignoringRequestMatchers("/ws/**"));

        return http.build();
    }

    @Bean
    UserDetailsService users(PasswordEncoder passwordEncoder) {
        return new InMemoryUserDetailsManager(
            User.withUsername("alice").password(passwordEncoder.encode("password")).roles("USER").build(),
            User.withUsername("bob").password(passwordEncoder.encode("password")).roles("USER").build(),
            User.withUsername("charlie").password(passwordEncoder.encode("password")).roles("USER").build()
        );
    }

    @Bean
    PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

`@EnableMethodSecurity` turns on `@PreAuthorize` support for the REST controller later in the article. This setup ensures the `Principal` is populated for each authenticated session.

## Step 4: Capture Identity and Connection Metadata

Define DTOs under `model/`.

`BroadcastMessage.java`:

```java
package com.example.targeted.model;

import java.time.Instant;

public class BroadcastMessage {

    private String type;
    private String content;
    private String from;
    private Instant timestamp = Instant.now();

    public BroadcastMessage() {
    }

    public BroadcastMessage(String type, String content, String from) {
        this.type = type;
        this.content = content;
        this.from = from;
        this.timestamp = Instant.now();
    }

    public String getType() {
        return type;
    }

    public void setType(String type) {
        this.type = type;
    }

    public String getContent() {
        return content;
    }

    public void setContent(String content) {
        this.content = content;
    }

    public String getFrom() {
        return from;
    }

    public void setFrom(String from) {
        this.from = from;
    }

    public Instant getTimestamp() {
        return timestamp;
    }

    public void setTimestamp(Instant timestamp) {
        this.timestamp = timestamp;
    }
}
```

`TargetedMessageRequest.java`:

```java
package com.example.targeted.model;

public class TargetedMessageRequest {

    private String targetUsername;
    private String targetSessionId;
    private String content;

    public String getTargetUsername() {
        return targetUsername;
    }

    public void setTargetUsername(String targetUsername) {
        this.targetUsername = targetUsername;
    }

    public String getTargetSessionId() {
        return targetSessionId;
    }

    public void setTargetSessionId(String targetSessionId) {
        this.targetSessionId = targetSessionId;
    }

    public String getContent() {
        return content;
    }

    public void setContent(String content) {
        this.content = content;
    }
}
```

Next, implement `IdentityAwareController.java`:

```java
package com.example.targeted.controller;

import com.example.targeted.model.BroadcastMessage;
import com.example.targeted.model.TargetedMessageRequest;
import com.example.targeted.service.TargetedNotificationService;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.messaging.handler.annotation.Header;
import org.springframework.messaging.handler.annotation.MessageMapping;
import org.springframework.messaging.handler.annotation.Payload;
import org.springframework.messaging.handler.annotation.SendToUser;
import org.springframework.stereotype.Controller;

import java.security.Principal;

@Controller
public class IdentityAwareController {

    private static final Logger log = LoggerFactory.getLogger(IdentityAwareController.class);

    private final TargetedNotificationService notificationService;

    public IdentityAwareController(TargetedNotificationService notificationService) {
        this.notificationService = notificationService;
    }

    @MessageMapping("/messages.broadcast")
    public void handleBroadcast(@Payload TargetedMessageRequest request,
                                Principal principal,
                                @Header("simpSessionId") String sessionId) {

        String sender = principal != null ? principal.getName() : "anonymous";
        log.info("Broadcast requested by {} on session {}", sender, sessionId);

        notificationService.sendBroadcast(
            new BroadcastMessage("BROADCAST", request.getContent(), sender)
        );
    }

    @MessageMapping("/messages.to-user")
    public void handleDirectToUser(@Payload TargetedMessageRequest request,
                                   Principal principal,
                                   @Header("simpSessionId") String sessionId) {

        String sender = principal != null ? principal.getName() : "anonymous";
        log.info("User-targeted message from {} (session {}) to {}",
                sender, sessionId, request.getTargetUsername());

        notificationService.sendToUser(
            request.getTargetUsername(),
            new BroadcastMessage("USER", request.getContent(), sender)
        );
    }

    @MessageMapping("/messages.to-session")
    @SendToUser(destinations = "/queue/replies", broadcast = false)
    public BroadcastMessage handleDirectToSession(@Payload TargetedMessageRequest request,
                                                  Principal principal,
                                                  @Header("simpSessionId") String sessionId) {

        String sender = principal != null ? principal.getName() : "anonymous";
        log.info("Session-targeted message from {} (session {}) to session {}",
                sender, sessionId, request.getTargetSessionId());

        notificationService.sendToSession(
            request.getTargetSessionId(),
            new BroadcastMessage("SESSION", request.getContent(), sender)
        );

        // Respond only to the caller's session
        return new BroadcastMessage(
            "ACK",
            "Delivered to session " + request.getTargetSessionId(),
            "server"
        );
    }
}
```

Important details:

- The `Principal` parameter gives you the authenticated username.
- `@Header("simpSessionId")` exposes the physical connection ID.
- `@SendToUser(... broadcast = false)` ensures the acknowledgement goes only to the session that invoked the mapping, even if the same user has several active sessions.

## Step 5: Implement the Targeted Notification Service

Create `TargetedNotificationService.java`:

```java
package com.example.targeted.service;

import com.example.targeted.model.BroadcastMessage;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.messaging.simp.SimpMessageHeaderAccessor;
import org.springframework.messaging.simp.SimpMessagingTemplate;
import org.springframework.messaging.simp.user.SimpSession;
import org.springframework.messaging.simp.user.SimpUser;
import org.springframework.messaging.simp.user.SimpUserRegistry;
import org.springframework.stereotype.Service;

import java.util.Map;
import java.util.Optional;

@Service
public class TargetedNotificationService {

    private static final Logger log = LoggerFactory.getLogger(TargetedNotificationService.class);

    private final SimpMessagingTemplate messagingTemplate;
    private final SimpUserRegistry userRegistry;

    public TargetedNotificationService(SimpMessagingTemplate messagingTemplate,
                                       SimpUserRegistry userRegistry) {
        this.messagingTemplate = messagingTemplate;
        this.userRegistry = userRegistry;
    }

    public void sendBroadcast(BroadcastMessage message) {
        log.info("Sending broadcast to /topic/announcements");
        messagingTemplate.convertAndSend("/topic/announcements", message);
    }

    public void sendToUser(String username, BroadcastMessage message) {
        if (username == null || username.isBlank()) {
            log.warn("Cannot send to empty username");
            return;
        }

        log.info("Sending message to user {}", username);
        messagingTemplate.convertAndSendToUser(
            username,
            "/queue/personal",
            message
        );
    }

    public void sendToSession(String sessionId, BroadcastMessage message) {
        if (sessionId == null || sessionId.isBlank()) {
            log.warn("Cannot send to empty sessionId");
            return;
        }

        Optional<SimpUser> targetUser = userRegistry.getUsers().stream()
            .filter(user -> user.getSessions().stream()
                .anyMatch(session -> sessionId.equals(session.getId())))
            .findFirst();

        if (targetUser.isEmpty()) {
            log.warn("Session {} not found. Message dropped.", sessionId);
            return;
        }

        SimpUser user = targetUser.get();
        SimpSession session = user.getSessions().stream()
            .filter(s -> sessionId.equals(s.getId()))
            .findFirst()
            .orElseThrow();

        log.info("Sending message to user {} on session {}", user.getName(), sessionId);

        messagingTemplate.convertAndSendToUser(
            user.getName(),
            "/queue/session-specific",
            message,
            headersForSession(session.getId())
        );
    }

    private Map<String, Object> headersForSession(String sessionId) {
        SimpMessageHeaderAccessor headerAccessor = SimpMessageHeaderAccessor.create();
        headerAccessor.setSessionId(sessionId);
        headerAccessor.setLeaveMutable(true);
        return headerAccessor.getMessageHeaders();
    }
}
```

Notes:

- The simple broker stores per-session destination prefixes. When you call `convertAndSendToUser`, Spring expands `/user/{username}/queue/...` into a session-specific destination such as `/queue/personal-user123`.
- For session-specific delivery, we use `SimpUserRegistry` to locate the user that owns the session ID, then pass that ID in a custom header created with `SimpMessageHeaderAccessor`. Spring’s user destination resolver inspects the header and routes the payload to just the matching session.
- If the session ID is not active, we log a warning instead of throwing an exception.

## Step 6: Allow Admins to Trigger Messages Outside STOMP

While most of this article focuses on STOMP handlers, many systems also expose REST endpoints for back-office tools. Add `AdminNotificationController.java`:

```java
package com.example.targeted.controller;

import com.example.targeted.model.BroadcastMessage;
import com.example.targeted.model.TargetedMessageRequest;
import com.example.targeted.service.TargetedNotificationService;
import org.springframework.http.ResponseEntity;
import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("/api/admin/notifications")
public class AdminNotificationController {

    private final TargetedNotificationService notificationService;

    public AdminNotificationController(TargetedNotificationService notificationService) {
        this.notificationService = notificationService;
    }

    @PostMapping("/broadcast")
    @PreAuthorize("hasRole('USER')")
    public ResponseEntity<Void> broadcast(@RequestBody TargetedMessageRequest request) {
        notificationService.sendBroadcast(
            new BroadcastMessage("BROADCAST", request.getContent(), "rest-admin")
        );
        return ResponseEntity.accepted().build();
    }

    @PostMapping("/user")
    @PreAuthorize("hasRole('USER')")
    public ResponseEntity<Void> sendToUser(@RequestBody TargetedMessageRequest request) {
        notificationService.sendToUser(
            request.getTargetUsername(),
            new BroadcastMessage("USER", request.getContent(), "rest-admin")
        );
        return ResponseEntity.accepted().build();
    }
}
```

`@PreAuthorize` ensures only authenticated users trigger these endpoints; adjust roles as needed.

## Step 7: Build the Browser Client

Create `templates/dashboard.html`:

```html
<!DOCTYPE html>
<html lang="en" xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>Targeted STOMP Messaging Dashboard</title>
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <style>
        body { font-family: system-ui, sans-serif; margin: 2rem auto; max-width: 960px; }
        header { display: flex; justify-content: space-between; align-items: center; margin-bottom: 1.5rem; }
        .status { padding: 0.4rem 0.75rem; border-radius: 4px; font-weight: 600; }
        .status.connected { background: #d4edda; color: #155724; }
        .status.disconnected { background: #f8d7da; color: #721c24; }
        section { border: 1px solid #ddd; border-radius: 8px; padding: 1rem; margin-bottom: 1.5rem; }
        h2 { margin-top: 0; }
        form { display: grid; gap: 0.75rem; margin-top: 0.5rem; }
        label { font-weight: 600; }
        input, textarea { padding: 0.5rem; border: 1px solid #ccc; border-radius: 4px; }
        button { padding: 0.6rem 1rem; border: none; border-radius: 4px; background: #007bff; color: white; cursor: pointer; }
        button:disabled { background: #999; cursor: not-allowed; }
        .log { height: 260px; overflow-y: auto; background: #f9f9f9; border-radius: 6px; border: 1px solid #eee; padding: 0.75rem; }
        .log-entry { margin-bottom: 0.75rem; border-left: 4px solid #007bff; padding-left: 0.75rem; }
        .log-entry.session { border-left-color: #6f42c1; }
        .log-entry.user { border-left-color: #17a2b8; }
        .log-entry.system { border-left-color: #ffc107; }
        .log-entry .meta { font-size: 0.8rem; color: #666; }
    </style>
</head>
<body>
<header>
    <div>
        <h1>Targeted STOMP Messaging Demo</h1>
        <p>Signed in as <strong th:text="${#authentication.name}">anonymous</strong></p>
    </div>
    <div>
        <button id="connect-btn">Connect</button>
        <button id="disconnect-btn" disabled>Disconnect</button>
        <span id="status" class="status disconnected">Disconnected</span>
    </div>
</header>

<section>
    <h2>Active Session Details</h2>
    <p id="session-info">Session ID: <em>unknown until connected</em></p>
</section>

<section>
    <h2>Broadcast to Everyone</h2>
    <form id="broadcast-form">
        <label for="broadcast-content">Message</label>
        <input id="broadcast-content" placeholder="What should everyone hear?" required>
        <button type="submit">Send Broadcast</button>
    </form>
</section>

<section>
    <h2>Message a User</h2>
    <form id="user-form">
        <label for="user-target">Recipient Username</label>
        <input id="user-target" placeholder="e.g., alice" required>
        <label for="user-content">Message</label>
        <input id="user-content" placeholder="Message content" required>
        <button type="submit">Send to User</button>
    </form>
</section>

<section>
    <h2>Message a Specific Session</h2>
    <form id="session-form">
        <label for="session-target">Target Session ID</label>
        <input id="session-target" placeholder="Paste a session ID" required>
        <label for="session-content">Message</label>
        <input id="session-content" placeholder="Only one tab should receive this" required>
        <button type="submit">Send to Session</button>
    </form>
</section>

<section>
    <h2>Inbound Message Log</h2>
    <div id="log" class="log"></div>
</section>

<script src="https://cdn.jsdelivr.net/npm/sockjs-client@1/dist/sockjs.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/stompjs@2.3.3/lib/stomp.min.js"></script>
<script>
    let stompClient = null;
    let currentSessionId = null;

    const statusEl = document.getElementById('status');
    const connectBtn = document.getElementById('connect-btn');
    const disconnectBtn = document.getElementById('disconnect-btn');
    const logEl = document.getElementById('log');
    const sessionInfoEl = document.getElementById('session-info');

    function setConnected(connected) {
        connectBtn.disabled = connected;
        disconnectBtn.disabled = !connected;
        statusEl.textContent = connected ? 'Connected' : 'Disconnected';
        statusEl.className = 'status ' + (connected ? 'connected' : 'disconnected');
    }

    function appendLog(entry, cssClass) {
        const div = document.createElement('div');
        div.className = 'log-entry ' + (cssClass || '');
        div.innerHTML = `
            <div class="meta">${new Date().toLocaleTimeString()} · ${entry.type}</div>
            <div><strong>From:</strong> ${entry.from}</div>
            <div><strong>Content:</strong> ${entry.content}</div>
        `;
        logEl.appendChild(div);
        logEl.scrollTop = logEl.scrollHeight;
    }

    function connect() {
        const socket = new SockJS('/ws');
        stompClient = Stomp.over(socket);
        stompClient.debug = null;

        stompClient.connect({}, frame => {
            setConnected(true);
            currentSessionId = frame.headers['session'];
            sessionInfoEl.textContent = 'Session ID: ' + currentSessionId;

            stompClient.subscribe('/topic/announcements', payload => {
                appendLog(JSON.parse(payload.body), 'system');
            });

            stompClient.subscribe('/user/queue/personal', payload => {
                appendLog(JSON.parse(payload.body), 'user');
            });

            stompClient.subscribe('/user/queue/session-specific', payload => {
                appendLog(JSON.parse(payload.body), 'session');
            });

            stompClient.subscribe('/user/queue/replies', payload => {
                appendLog(JSON.parse(payload.body), 'session');
            });
        }, error => {
            console.error('STOMP error', error);
            setConnected(false);
        });
    }

    function disconnect() {
        if (stompClient) {
            stompClient.disconnect(() => setConnected(false));
        }
    }

    function send(destination, message) {
        if (!stompClient || !stompClient.connected) {
            alert('Connect first!');
            return;
        }
        stompClient.send(destination, {}, JSON.stringify(message));
    }

    document.getElementById('broadcast-form').addEventListener('submit', event => {
        event.preventDefault();
        send('/app/messages.broadcast', { content: document.getElementById('broadcast-content').value });
        event.target.reset();
    });

    document.getElementById('user-form').addEventListener('submit', event => {
        event.preventDefault();
        send('/app/messages.to-user', {
            targetUsername: document.getElementById('user-target').value,
            content: document.getElementById('user-content').value
        });
        event.target.reset();
    });

    document.getElementById('session-form').addEventListener('submit', event => {
        event.preventDefault();
        send('/app/messages.to-session', {
            targetSessionId: document.getElementById('session-target').value,
            content: document.getElementById('session-content').value
        });
    });

    connectBtn.addEventListener('click', connect);
    disconnectBtn.addEventListener('click', disconnect);
</script>
</body>
</html>
```

The dashboard displays your current STOMP session ID (from the CONNECTED frame header) so you can copy it into another tab to test session-targeted messages.

## Step 8: Wire Up the Application Entry Point

Create `TargetedMessagingApplication.java` to enable scheduling (if you keep automated notifications) and launch the app:

```java
package com.example.targeted;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class TargetedMessagingApplication {

    public static void main(String[] args) {
        SpringApplication.run(TargetedMessagingApplication.class, args);
    }
}
```

Add a simple MVC controller to serve the dashboard:

```java
package com.example.targeted.controller;

import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.GetMapping;

@Controller
public class DashboardController {

    @GetMapping("/")
    public String dashboard() {
        return "dashboard";
    }
}
```

## Step 9: Test the Three Delivery Patterns

### Start the Application

```bash
./mvnw spring-boot:run
```

### Open Multiple Tabs

1. Navigate to `http://localhost:8080/` and log in as `alice` (password `password`).
2. Open a second tab and log in as `bob`.
3. Optional: Open a third tab as `alice` again to simulate multiple sessions for the same user.

Click “Connect” in each tab to establish the STOMP session. The dashboard shows the session ID once connected.

### Try Broadcast

- In Alice’s tab, send a broadcast. All connected tabs (including Bob’s) receive it via `/topic/announcements`.

### Try User-Targeted Messaging

- From Bob’s tab, send a message to `alice`.
- Both tabs where Alice is signed in receive the message via `/user/queue/personal`.
- Bob does not receive his own message unless you add logic to echo it back.

### Try Session-Targeted Messaging

- Copy the session ID from Alice’s first tab.
- In Alice’s second tab, paste that ID into the “Message a Specific Session” form.
- Only the tab whose session ID you entered receives the message (via `/user/queue/session-specific`).
- The sender receives an acknowledgement in `/user/queue/replies` thanks to `@SendToUser`.

### Observe the Logs

The inbound channel interceptor logs each `SEND` and `SUBSCRIBE` with the user and session ID, verifying that your routing decisions align with expectations.

## Behind the Scenes: Expanding User Destinations

When you call `convertAndSendToUser("alice", "/queue/personal", payload)`:

1. Spring looks up all sessions registered for user “alice” using `SimpUserRegistry`.
2. For each session, it expands the destination into a unique address, such as `/queue/personal-user123`.
3. The simple broker delivers the message to each resolved destination.

`@SendToUser` follows the same rules, using the current `Principal`. Adding `broadcast = false` restricts delivery to the session that initiated the method—essential when sending confidential acknowledgements or errors.

## Advanced Techniques and Safeguards

- **Presence checks:** Inject `SimpUserRegistry` and verify `userRegistry.getUser(username)` before sending. Fall back to storing notifications if the user is offline.
- **Session metadata:** Use `SimpMessageHeaderAccessor` to store custom headers (e.g., device type) in a `ChannelInterceptor`, then act on them when routing.
- **Error reporting:** Add a `/user/queue/errors` destination and send structured errors when a target username or session ID is invalid. This keeps clients informed without exposing implementation details.
- **Broker relay compatibility:** If you upgrade to an external broker (Article 10 topic), the same `convertAndSendToUser` API works; the relay resolves destinations on the broker side instead of the simple broker. Session IDs remain meaningful because Spring still tracks them on the application tier.

## Troubleshooting

| Symptom | Likely Cause | Fix |
|---------|--------------|-----|
| Messages do not arrive in `/user/queue/personal` | Missing user destination prefix | Ensure `registry.setUserDestinationPrefix("/user")` exists in `WebSocketConfig`. |
| Only one of a user’s tabs receives a message | Session registry missing | Confirm the user is authenticated in both tabs and `convertAndSendToUser` is used instead of raw `convertAndSend`. |
| Session-targeted message delivers nowhere | Session ID outdated | Refresh the tab to obtain a new session ID; verify the server still sees it via logs or `SimpUserRegistry`. |
| `Principal` is `null` inside `@MessageMapping` | Unsecured handshake | Confirm the user is authenticated before opening the WebSocket and that `/ws/**` is covered by Spring Security. |
| `IllegalArgumentException` when sending to session | Destination prefix mismatch | Use the `SimpSession.getDestinationPrefix()` when constructing the destination, as shown in `sendToSession`. |

## Recap

- We inspected Spring’s STOMP routing pipeline and learned where identity and session information flow through the system.
- We captured `Principal` and `simpSessionId` in `@MessageMapping` methods to understand who sent each message.
- We implemented broadcast, per-user, and per-session delivery using `SimpMessagingTemplate` and `@SendToUser`.
- We built an HTML dashboard that exercises all three delivery modes and exposes the current session ID for testing.
- We explored safeguards like presence checks, error channels, and broker relay compatibility to keep messaging reliable at scale.

With these tools, you can design secure, fine-grained messaging topologies that treat broadcasts, user-specific alerts, and session-scoped replies as first-class citizens. In upcoming articles we’ll build on this foundation to add rich presence tracking and administrative tooling for observing every active STOMP session in real time.
