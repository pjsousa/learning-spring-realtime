## Securing Real-Time: Associating User Identity with STOMP Sessions

Real-time messaging is only as valuable as the trust we can place in each participant. When every STOMP exchange flows through the same broker endpoints, the only way to enforce permissions, privacy, and auditing is to know exactly who is on each session. Today we take our toy project into that territory: we will wire Spring Security into the WebSocket handshake, demonstrate how the authenticated `Principal` travels with every STOMP frame, and show how to tap into that identity in controllers, interceptors, and client logic. By the end you will have a secure chat-like application where every frame carries the user who sent it, plus a browser client that lets you experience the flow immediately.

### What We Are Building

- A Spring Boot 3.x application that exposes STOMP endpoints over WebSockets (with SockJS fallback).
- Conventional HTTP login backed by Spring Security’s session support (in-memory users for the exercise).
- A WebSocket/STOMP configuration that propagates the HTTP user into the WebSocket session.
- Server-side message handling that reads the `Principal` and stamps sender identity onto outgoing payloads.
- Presence tracking via Spring’s `SessionConnectEvent` and `SessionDisconnectEvent`.
- A plain HTML client (no build tools) that lets you log in, connect via STOMP, send messages, and see identities on each frame.

### Prerequisites and Recap

We assume you either followed the earlier installments or are comfortable with basic Spring Boot STOMP setups:

- JDK 17+
- Spring Boot CLI or Maven/Gradle wrapper
- Familiarity with STOMP endpoints (`/ws`), topics (`/topic/public`), and controllers annotated with `@MessageMapping`.

If you missed the previous article, the essential idea is that the browser connects to `/ws` using SockJS + STOMP, sends frames to `/app/...`, and subscribes to broker destinations like `/topic/...`. For today, the only new wrinkle is security identity.

---

### Step 1 – Create the Project Skeleton

Use Spring Initializr (or your build tool of choice) with these dependencies:

- Spring Web
- Spring WebSocket
- Spring Security
- Spring Messaging
- (Optional) Spring Boot DevTools for convenience

If you prefer Maven, your `pom.xml` dependencies look like:

```xml
<dependencies>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-websocket</artifactId>
  </dependency>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
  </dependency>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
  </dependency>
</dependencies>
```

With the project scaffolded, verify the basics with `./mvnw spring-boot:run` (or the Gradle equivalent) and ensure the app starts before proceeding.

---

### Step 2 – Understand the Authentication Flow

Before writing code, let’s clarify how Spring ties identity to WebSockets:

1. **HTTP Handshake:** The browser first performs a standard HTTP(s) handshake to upgrade to WebSocket. This handshake sits on the same domain/port as your application.
2. **Security Filter Chain:** Spring Security intercepts that handshake request. If there is an existing HTTP session with an authenticated user, the handshake succeeds with that identity; if not, the user remains anonymous (or the handshake can be rejected).
3. **Principal Propagation:** The `Principal` created by Spring Security is stored on the WebSocket session. Every STOMP frame sent over that session carries the principal reference.
4. **Principal in Controllers:** In `@MessageMapping` methods you can add a `Principal principal` parameter to retrieve that identity directly.

In short, the heavy lifting happens automatically once HTTP security is configured. Our job is to ensure we have an authenticated session beforehand and optionally add visibility/control around it.

---

### Step 3 – Configure Spring Security for HTTP Sessions

We will create a small security configuration that provides:

- Form login at `/login` with a default template.
- In-memory users (`alice`, `bob`, `charlie`) for the demo.
- A security filter chain that requires authentication for HTML pages and the WebSocket handshake.
- CSRF configuration that keeps WebSocket endpoints open while protecting POSTs (optional for the toy project but educational).

Create `SecurityConfig` under `src/main/java/com/example/stompsecurity/config/SecurityConfig.java`:

```java
package com.example.stompsecurity.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.Customizer;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.core.userdetails.User;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.provisioning.InMemoryUserDetailsManager;
import org.springframework.security.web.SecurityFilterChain;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.security.crypto.password.PasswordEncoder;

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

Key points:

- `csrf.ignoringRequestMatchers("/ws/**")` prevents CSRF from interfering with the WebSocket upgrade request.
- Every WebSocket handshake inherits the HTTP session, so requiring authentication on “any request” ensures the handshake cannot occur anonymously.
- In production you would integrate with LDAP, OAuth2, or your user database instead of in-memory, but the principle is identical.

---

### Step 4 – Wire the WebSocket/STOMP Configuration

Now we create a configuration class that registers STOMP endpoints, adds SockJS, and sets an interceptor that only logs—identity is already handled, but we want visibility.

Create `WebSocketConfig` under `src/main/java/com/example/stompsecurity/config/WebSocketConfig.java`:

```java
package com.example.stompsecurity.config;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.context.annotation.Configuration;
import org.springframework.messaging.simp.config.ChannelRegistration;
import org.springframework.messaging.simp.config.MessageBrokerRegistry;
import org.springframework.messaging.support.ChannelInterceptorAdapter;
import org.springframework.messaging.support.MessageHeaderAccessor;
import org.springframework.messaging.Message;
import org.springframework.messaging.support.ExecutorSubscribableChannel;
import org.springframework.web.socket.config.annotation.EnableWebSocketMessageBroker;
import org.springframework.web.socket.config.annotation.StompEndpointRegistry;
import org.springframework.web.socket.config.annotation.WebSocketMessageBrokerConfigurer;

import java.security.Principal;

@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {

  private static final Logger log = LoggerFactory.getLogger(WebSocketConfig.class);

  @Override
  public void registerStompEndpoints(StompEndpointRegistry registry) {
    registry.addEndpoint("/ws")
      .setAllowedOriginPatterns("*")
      .withSockJS();
  }

  @Override
  public void configureMessageBroker(MessageBrokerRegistry registry) {
    registry.enableSimpleBroker("/topic", "/queue");
    registry.setApplicationDestinationPrefixes("/app");
    registry.setUserDestinationPrefix("/user");
  }

  @Override
  public void configureClientInboundChannel(ChannelRegistration registration) {
    registration.interceptors(new ChannelInterceptorAdapter() {
      @Override
      public Message<?> preSend(Message<?> message, ExecutorSubscribableChannel channel) {
        Principal principal = message.getHeaders().get("simpUser", Principal.class);
        if (principal != null) {
          log.debug("Inbound frame from user {}", principal.getName());
        }
        return message;
      }
    });
  }
}
```

Notes:

- SockJS fallback ensures all browsers can follow along without SSL or reverse proxies.
- `setAllowedOriginPatterns("*")` keeps the demo simple; tighten it in production.
- The inbound channel interceptor demonstrates how to read `simpUser` from headers. You can enforce additional rules here (e.g., rejecting frames based on roles).
- `setUserDestinationPrefix("/user")` prepares us for per-user queues in later articles.

---

### Step 5 – Model the Messages with Identity Metadata

We will send a simple chat object containing the text and the server-stamped sender. Create `ChatMessage` and `ChatPayload` models:

`src/main/java/com/example/stompsecurity/model/ChatPayload.java`:

```java
package com.example.stompsecurity.model;

public record ChatPayload(String text) { }
```

`src/main/java/com/example/stompsecurity/model/ChatMessage.java`:

```java
package com.example.stompsecurity.model;

import java.time.Instant;

public record ChatMessage(
  String sender,
  String text,
  Instant timestamp
) { }
```

Clients will send only the raw text payload; the server will enrich it with the current user and timestamp.

---

### Step 6 – Implement the Controller that Reads `Principal`

Create `ChatController` under `src/main/java/com/example/stompsecurity/controller/ChatController.java`:

```java
package com.example.stompsecurity.controller;

import com.example.stompsecurity.model.ChatMessage;
import com.example.stompsecurity.model.ChatPayload;
import org.springframework.messaging.handler.annotation.MessageMapping;
import org.springframework.messaging.handler.annotation.Payload;
import org.springframework.messaging.simp.SimpMessagingTemplate;
import org.springframework.stereotype.Controller;

import java.security.Principal;
import java.time.Instant;

@Controller
public class ChatController {

  private final SimpMessagingTemplate messagingTemplate;

  public ChatController(SimpMessagingTemplate messagingTemplate) {
    this.messagingTemplate = messagingTemplate;
  }

  @MessageMapping("/chat")
  public void handleChat(
      @Payload ChatPayload payload,
      Principal principal
  ) {
    String sender = principal != null ? principal.getName() : "anonymous";
    ChatMessage message = new ChatMessage(sender, payload.text(), Instant.now());
    messagingTemplate.convertAndSend("/topic/public", message);
  }
}
```

Highlights:

- The method signature includes `Principal principal`. Spring resolves it automatically from the WebSocket session that sent the frame.
- We publish to `/topic/public`, which all clients subscribe to.
- The controller enforces identity stamping by ignoring any sender field the client might try to sneak in.

You could also add role checks (`principal instanceof UsernamePasswordAuthenticationToken` etc.) or propagate more metadata (roles, avatars). The crucial step is that we always trust the server-provided principal, not anything in the incoming payload.

---

### Step 7 – Track Presence with Application Events

To reinforce that the user identity is part of the session lifecycle, listen to `SessionConnectEvent` and `SessionDisconnectEvent`.

Create `PresenceEventListener` under `src/main/java/com/example/stompsecurity/listener/PresenceEventListener.java`:

```java
package com.example.stompsecurity.listener;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.context.event.EventListener;
import org.springframework.messaging.simp.stomp.StompHeaderAccessor;
import org.springframework.stereotype.Component;
import org.springframework.web.socket.messaging.SessionConnectEvent;
import org.springframework.web.socket.messaging.SessionDisconnectEvent;

import java.security.Principal;

@Component
public class PresenceEventListener {

  private static final Logger log = LoggerFactory.getLogger(PresenceEventListener.class);

  @EventListener
  public void handleSessionConnected(SessionConnectEvent event) {
    StompHeaderAccessor accessor = StompHeaderAccessor.wrap(event.getMessage());
    Principal principal = accessor.getUser();
    log.info("WebSocket connected: session={}, user={}", accessor.getSessionId(), principal != null ? principal.getName() : "unknown");
  }

  @EventListener
  public void handleSessionDisconnected(SessionDisconnectEvent event) {
    StompHeaderAccessor accessor = StompHeaderAccessor.wrap(event.getMessage());
    Principal principal = accessor.getUser();
    log.info("WebSocket disconnected: session={}, user={}", accessor.getSessionId(), principal != null ? principal.getName() : "unknown");
  }
}
```

Logging is sufficient for the toy project, but you could store presence in a repository or notify clients via `/topic` updates. The identity is accessible through `accessor.getUser()`.

---

### Step 8 – Expose the Home Page and Static Assets

We need a controller that serves the main HTML page. Create `HomeController` under `src/main/java/com/example/stompsecurity/controller/HomeController.java`:

```java
package com.example.stompsecurity.controller;

import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.GetMapping;
import java.security.Principal;
import org.springframework.ui.Model;

@Controller
public class HomeController {

  @GetMapping("/")
  public String index(Principal principal, Model model) {
    model.addAttribute("username", principal != null ? principal.getName() : "anonymous");
    return "index";
  }
}
```

Add Thymeleaf (or basic Mustache) to render the username. If using Thymeleaf, include the dependency:

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-thymeleaf</artifactId>
</dependency>
```

Alternatively, you can expose a static HTML page with placeholders and set the username from JavaScript by hitting `/user`. To keep it simple, we’ll add Thymeleaf and render the user server-side.

Create `src/main/resources/templates/index.html`:

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>Secure STOMP Chat</title>
  <link rel="stylesheet" href="/css/main.css">
</head>
<body>
  <header>
    <h1>Secure STOMP Chat</h1>
    <p id="user-info">Signed in as <span th:text="${username}"></span></p>
    <form action="/logout" method="post">
      <input type="hidden" th:name="${_csrf.parameterName}" th:value="${_csrf.token}">
      <button type="submit">Logout</button>
    </form>
  </header>

  <section id="connection">
    <button id="connect-btn">Connect</button>
    <button id="disconnect-btn" disabled>Disconnect</button>
    <span id="connection-status">Disconnected</span>
  </section>

  <section id="chat">
    <div id="messages"></div>
    <form id="chat-form">
      <input type="text" id="chat-input" placeholder="Type message..." autocomplete="off">
      <button type="submit">Send</button>
    </form>
  </section>

  <script src="https://cdn.jsdelivr.net/npm/sockjs-client@1/dist/sockjs.min.js"></script>
  <script src="https://cdn.jsdelivr.net/npm/stompjs@2.3.3/lib/stomp.min.js"></script>
  <script src="/js/app.js"></script>
</body>
</html>
```

Add a simple stylesheet at `src/main/resources/static/css/main.css`:

```css
body {
  font-family: system-ui, sans-serif;
  margin: 2rem auto;
  max-width: 720px;
  color: #222;
}

header {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

#messages {
  border: 1px solid #ccc;
  border-radius: 6px;
  padding: 1rem;
  height: 320px;
  overflow-y: auto;
  margin-bottom: 1rem;
  background: #fafafa;
}

.message-row {
  margin-bottom: 0.75rem;
}

.message-sender {
  font-weight: 600;
  margin-right: 0.5rem;
  color: #0066cc;
}

.message-meta {
  font-size: 0.8rem;
  color: #666;
}
```

---

### Step 9 – Build the Browser STOMP Client

The JavaScript client handles connecting, subscribing, sending messages, and surfacing identity from the server responses. Create `src/main/resources/static/js/app.js`:

```javascript
let stompClient = null;

function connect() {
  const socket = new SockJS('/ws');
  stompClient = Stomp.over(socket);

  stompClient.connect({}, frame => {
    document.querySelector('#connection-status').textContent = 'Connected';
    document.querySelector('#connect-btn').disabled = true;
    document.querySelector('#disconnect-btn').disabled = false;
    console.log('Connected: ' + frame);

    stompClient.subscribe('/topic/public', message => {
      const body = JSON.parse(message.body);
      renderMessage(body);
    });
  }, error => {
    console.error('STOMP error', error);
    document.querySelector('#connection-status').textContent = 'Disconnected (error)';
  });
}

function disconnect() {
  if (stompClient !== null) {
    stompClient.disconnect(() => {
      document.querySelector('#connection-status').textContent = 'Disconnected';
      document.querySelector('#connect-btn').disabled = false;
      document.querySelector('#disconnect-btn').disabled = true;
    });
  }
}

function sendMessage(event) {
  event.preventDefault();
  const input = document.querySelector('#chat-input');
  const text = input.value.trim();
  if (!text || !stompClient || !stompClient.connected) {
    return;
  }
  stompClient.send('/app/chat', {}, JSON.stringify({ text }));
  input.value = '';
}

function renderMessage(message) {
  const container = document.querySelector('#messages');
  const row = document.createElement('div');
  row.classList.add('message-row');

  const header = document.createElement('div');
  const senderSpan = document.createElement('span');
  senderSpan.classList.add('message-sender');
  senderSpan.textContent = message.sender;

  const metaSpan = document.createElement('span');
  metaSpan.classList.add('message-meta');
  const timestamp = new Date(message.timestamp);
  metaSpan.textContent = ` • ${timestamp.toLocaleTimeString()}`;

  header.appendChild(senderSpan);
  header.appendChild(metaSpan);

  const textParagraph = document.createElement('p');
  textParagraph.textContent = message.text;

  row.appendChild(header);
  row.appendChild(textParagraph);
  container.appendChild(row);
  container.scrollTop = container.scrollHeight;
}

document.addEventListener('DOMContentLoaded', () => {
  document.querySelector('#connect-btn').addEventListener('click', connect);
  document.querySelector('#disconnect-btn').addEventListener('click', disconnect);
  document.querySelector('#chat-form').addEventListener('submit', sendMessage);
});
```

This client intentionally does **not** send the username—it relies entirely on the server identity. When multiple browser sessions (with different logins) connect, the UI shows each message labeled with the authenticated user.

---

### Step 10 – Walk Through the End-to-End Flow

1. **Start the app:** `./mvnw spring-boot:run`.
2. **Navigate to `http://localhost:8080`**. You will be redirected to `/login`.
3. **Log in** using one of the in-memory users (e.g., `alice` / `password`). Spring creates an HTTP session with that identity.
4. **Landing page** shows “Signed in as alice.” Click “Connect” to establish the WebSocket session.
5. **Send messages:** Type a message. The server receives `/app/chat`, obtains the `Principal`, and broadcasts to `/topic/public` as:

```json
{
  "sender": "alice",
  "text": "Hello from secured STOMP!",
  "timestamp": "2025-11-02T15:10:45.123456Z"
}
```

6. **Open another browser (or incognito)**, log in as `bob`, connect, and send messages. Each message should show the correct sender identity. Observe the logs to see session connect/disconnect events annotated with user names.
7. **Logout** to invalidate the HTTP session. Attempting to reconnect without logging in should redirect you back to the login form.

---

### Enhancing Identity Enforcement (Optional Experiments)

- **Reject Anonymous Frames:** Update `SecurityFilterChain` to permit `/ws` only for authenticated sessions; currently `.anyRequest().authenticated()` already enforces this, but you can go further by adding a handshake handler that denies `Principal` null.
- **Inject Custom User Details:** If you enrich `UserDetails` with more metadata (roles, profile data), you can publish it in headers or message payloads.
- **Token-Based Auth:** Replace form login with JWTs by extracting tokens in a custom `HandshakeInterceptor`. The handshake can validate the token and populate a principal object.
- **Access Control in Controller:** Add method security annotations (`@PreAuthorize("hasRole('ADMIN')")`) or manual checks against `principal` to restrict dests.

---

### Testing the Identity Propagation

To make sure the principle holds under pressure, try the following:

1. **MitM Attempt:** Modify the JavaScript to send a fake `sender` field (not used) and confirm the server still stamps the correct user.
2. **Multiple Sessions per User:** Login as `alice` in two tabs, connect in both, send messages. Both sessions share the same `Principal` name but each has distinct WebSocket session IDs (tracked in event logs).
3. **Disconnect Handling:** Close one tab; the server logs a disconnect event with both session ID and username, demonstrating how you could update presence rosters.
4. **Role-Specific Behavior:** Update the in-memory users to include an `ADMIN` role. Add a new `@MessageMapping("/moderate")` method that only admins can trigger by checking `principal`. Confirm unauthorized users get an error frame (Spring will throw an `AccessDeniedException` that results in an `ERROR` STOMP frame).

---

### Wrap-Up and Looking Ahead

We now have a working, secure STOMP application where identity is a first-class citizen:

- HTTP authentication establishes the user.
- WebSocket handshakes inherit the session and `Principal`.
- Each STOMP frame is delivered to your controller with the user automatically attached.
- Browser clients need not (and must not) supply their own identity; the server is the source of truth.
- Presence listeners and interceptors give you hooks for auditing, metrics, and security policies.

In the next article we will leverage the `Principal` to implement user-specific destinations (`/user/queue/...`) and private messaging—exactly the scenarios made possible by today’s groundwork.

**Next Steps**

- Experiment with token-based authentication (JWT Bearer) by plugging into the handshake.
- Persist chat history with sender metadata to a database to prepare for richer features.
- Add integration tests using `WebSocketStompClient` to automate identity verification across connections.

Keep this project running; we will extend it in Article 7 to route messages directly to specific users and enforce fine-grained authorization.
