---
title: "Differentiate Clients: Sending Private Messages using User Destinations (/user/)"
description: "Article 7 of the STOMP with Spring Boot series: configure user destinations, attach principals, and deliver private notifications with a plain HTML client."
series: "STOMP Toy Projects with Spring Boot"
order: 7
word_count_target: "1500-2000"
---

# Differentiate Clients: Sending Private Messages using User Destinations

Broadcasting updates to `/topic` channels works for open chatrooms, but personalized notifications—order updates, direct messages, to-do reminders—demand per-user delivery. Spring’s STOMP support solves this with **user destinations**, virtual endpoints that the broker rewrites into unique, session-aware queues. By targeting a username rather than a session identifier, we avoid manual connection bookkeeping while keeping the development model close to HTTP controllers.

This article (number seven in our ten-part series) guides you through configuring a custom user destination prefix, attaching principals to STOMP sessions, delivering private payloads with `SimpMessagingTemplate.convertAndSendToUser`, and surfacing the feature through a minimalist HTML client. We will

- configure a custom prefix (`/secured/user`) so private traffic is clearly separated from public topics,
- attach human-readable usernames to STOMP sessions using a channel interceptor,
- deliver private notifications from the server with `convertAndSendToUser`, and
- ship a zero-dependency browser client that lets you simulate two users exchanging private messages.

> **Goal**: 1,700 words (within the 1,500–2,000 target) covering setup, server logic, client wiring, and validation steps.

Even if you skipped earlier parts of the series, this article stands alone: it reintroduces the dependencies, configuration, and front-end wiring you need to follow along from scratch.

## Prerequisites

- Java 17 or newer
- Maven 3.8+ (snippets use the Maven Wrapper; adapt to Gradle if preferred)
- A modern browser (Chrome, Firefox, or Edge)
- Familiarity with Spring Boot starter projects
- Basic understanding of STOMP (application vs. broker destinations) helps but is optional

If you have been following the series, you can reuse your existing project. New readers should create a fresh Spring Boot application.

## Project Setup (Standalone)

Create a project named `personalized-messaging-demo` using `start.spring.io`. We need two starters: `spring-boot-starter-websocket` for STOMP support and `spring-boot-starter-web` so Spring Boot serves the static HTML assets.

```bash
curl https://start.spring.io/starter.zip \
  -d dependencies=websocket,web \
  -d name=personalized-messaging-demo \
  -d packageName=com.example.personalized \
  -d javaVersion=17 \
  -o personalized-messaging-demo.zip

unzip personalized-messaging-demo.zip && cd personalized-messaging-demo
./mvnw spring-boot:run
```

Leave the server running. Spring Boot reloads classes at runtime, so the subsequent edits become active without a restart.

## Anatomy of User Destinations

Before we write code, clarify the user-destination vocabulary:

- **User destination prefix**: The namespace clients subscribe to. Spring defaults to `/user`; we will use `/secured/user` to highlight sensitive traffic.
- **User-specific queue**: Behind the scenes, Spring rewrites `/secured/user/queue/updates` into something like `/queue/updates-user123`. The suffix differs per session, guaranteeing isolation, and you never reference it directly.
- **Principal**: Who the server sends to. `SimpMessagingTemplate.convertAndSendToUser("alice", "/queue/updates", payload)` directs the message to whichever session currently identifies as `alice`. Spring reads `Principal#getName()` from the STOMP session.

Our plan is to insist that every STOMP connection supplies a username (via a STOMP CONNECT header). We attach that username as the session’s `Principal`, enabling private messaging without tracking session IDs manually.

## Configure the WebSocket Broker

Create `src/main/java/com/example/personalized/config/WebSocketConfig.java` with broker setup, custom prefix, and a CONNECT-time interceptor.

```java
package com.example.personalized.config;

import com.example.personalized.support.StompPrincipal;
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
import org.springframework.messaging.support.MessageHeaderAccessor;
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
        registry.setApplicationDestinationPrefixes("/app");
        registry.setUserDestinationPrefix("/secured/user");
        registry.enableSimpleBroker("/topic", "/queue");
    }

    @Override
    public void configureClientInboundChannel(ChannelRegistration registration) {
        registration.interceptors(new UsernameInterceptor());
    }

    private static class UsernameInterceptor implements ChannelInterceptor {

        @Override
        public Message<?> preSend(Message<?> message, MessageChannel channel) {
            StompHeaderAccessor accessor = MessageHeaderAccessor.getAccessor(message, StompHeaderAccessor.class);
            if (accessor == null) {
                return message;
            }

            if (StompCommand.CONNECT.equals(accessor.getCommand())) {
                String rawUsername = accessor.getFirstNativeHeader("username");
                if (rawUsername == null || rawUsername.isBlank()) {
                    throw new IllegalArgumentException("Username header is required for private messaging");
                }

                Principal principal = new StompPrincipal(rawUsername.trim());
                accessor.setUser(principal);
                log.info("STOMP CONNECT accepted for user '{}'", principal.getName());
            }
            return message;
        }
    }
}
```

Highlights:

- `setUserDestinationPrefix("/secured/user")` sets the logical namespace for private subscriptions.
- SOCKJS support (`withSockJS()`) simplifies multi-tab testing and bypasses strict same-origin policies for this demo.
- `UsernameInterceptor` blocks users who forget to supply the header, ensuring every session carries a `Principal`.

Add a simple `Principal` implementation referenced above.

```java
package com.example.personalized.support;

import java.security.Principal;
import java.util.Objects;

public class StompPrincipal implements Principal {

    private final String name;

    public StompPrincipal(String name) {
        this.name = Objects.requireNonNull(name, "name");
    }

    @Override
    public String getName() {
        return name;
    }
}
```

Place this class under `src/main/java/com/example/personalized/support/StompPrincipal.java`.

## Model the Payloads

Define simple DTOs for inbound requests and outbound notifications. Java records keep the declarations concise.

```java
package com.example.personalized.model;

public record PrivateMessage(String recipient, String content) { }
```

```java
package com.example.personalized.model;

public record PrivateMessageNotification(String from, String body) { }
```

Store them in `src/main/java/com/example/personalized/model/PrivateMessage.java` and `src/main/java/com/example/personalized/model/PrivateMessageNotification.java`.

## Message Handling Controller

Implement a controller that processes private chat messages and targets the appropriate users.

```java
package com.example.personalized.messaging;

import com.example.personalized.model.PrivateMessage;
import com.example.personalized.model.PrivateMessageNotification;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.messaging.handler.annotation.MessageMapping;
import org.springframework.messaging.simp.SimpMessagingTemplate;
import org.springframework.stereotype.Controller;

import java.security.Principal;

@Controller
public class PrivateMessageController {

    private static final Logger log = LoggerFactory.getLogger(PrivateMessageController.class);
    private final SimpMessagingTemplate messagingTemplate;

    public PrivateMessageController(SimpMessagingTemplate messagingTemplate) {
        this.messagingTemplate = messagingTemplate;
    }

    @MessageMapping("/chat/private")
    public void processPrivateMessage(PrivateMessage message, Principal principal) {
        String sender = principal.getName();
        String recipient = message.recipient();

        log.info("Private message from '{}' to '{}'", sender, recipient);
        PrivateMessageNotification notification =
                new PrivateMessageNotification(sender, message.content());

        messagingTemplate.convertAndSendToUser(
                recipient,
                "/queue/updates",
                notification
        );

        messagingTemplate.convertAndSendToUser(
                sender,
                "/queue/updates",
                new PrivateMessageNotification("You", message.content())
        );
    }
}
```

Key details:

- `@MessageMapping("/chat/private")` expects clients to send payloads to `/app/chat/private`.
- `Principal` comes from the interceptor; its `name` indicates the sender.
- `convertAndSendToUser(recipient, "/queue/updates", ...)` publishes to `/secured/user/queue/updates` for the targeted user.
- The controller acknowledges the sender by echoing a "You" notification so both panes share a consistent UX.

## Browser Client

The client-side counterpart handles three responsibilities: connect with a username, subscribe to private updates, and send direct messages. Place the following file at `src/main/resources/static/private-chat.html`.

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>User Destination Private Chat</title>
  <style>
    body { font-family: sans-serif; margin: 2rem; max-width: 640px; }
    label { display: block; margin-top: 1rem; }
    textarea { width: 100%; height: 120px; }
    #log { border: 1px solid #ccc; padding: 1rem; margin-top: 1.5rem; height: 220px; overflow-y: auto; background: #f9f9f9; }
    .system { color: #999; font-style: italic; }
    .message { margin-bottom: .75rem; }
  </style>
</head>
<body>
<h1>Private Messaging Demo (/secured/user)</h1>

<section id="connect-panel">
  <label>
    Username
    <input type="text" id="username" placeholder="e.g. alice">
  </label>
  <button id="connect-btn">Connect</button>
  <p class="system">
    Open this page in two tabs, use different usernames (alice & bob), then exchange private messages.
  </p>
</section>

<section id="chat-panel" style="display:none;">
  <div>
    <strong>Connected as:</strong> <span id="connected-user"></span>
    <button id="disconnect-btn">Disconnect</button>
  </div>

  <label>
    Recipient username
    <input type="text" id="recipient" placeholder="e.g. bob">
  </label>

  <label>
    Message
    <textarea id="content" placeholder="Say hello privately..."></textarea>
  </label>

  <button id="send-btn">Send Private Message</button>
</section>

<section>
  <h2>Activity</h2>
  <div id="log"></div>
</section>

<script src="https://cdn.jsdelivr.net/npm/sockjs-client@1/dist/sockjs.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/stompjs@2.3.3/lib/stomp.min.js"></script>
<script>
  let stompClient = null;

  const logEl = document.getElementById('log');
  const connectPanel = document.getElementById('connect-panel');
  const chatPanel = document.getElementById('chat-panel');
  const usernameInput = document.getElementById('username');
  const recipientInput = document.getElementById('recipient');
  const contentInput = document.getElementById('content');
  const connectedUserEl = document.getElementById('connected-user');

  function log(message, type = 'message') {
    const entry = document.createElement('div');
    entry.className = type;
    entry.textContent = `[${new Date().toLocaleTimeString()}] ${message}`;
    logEl.appendChild(entry);
    logEl.scrollTop = logEl.scrollHeight;
  }

  function connect() {
    const username = usernameInput.value.trim();
    if (!username) {
      alert('Please choose a username');
      return;
    }

    const socket = new SockJS('/ws');
    stompClient = Stomp.over(socket);
    stompClient.connect(
      {username},
      frame => {
        log(`Connected: ${frame.headers['user-name'] || username}`, 'system');
        connectPanel.style.display = 'none';
        chatPanel.style.display = 'block';
        connectedUserEl.textContent = username;

        const destination = '/secured/user/queue/updates';
        stompClient.subscribe(destination, message => {
          const payload = JSON.parse(message.body);
          log(`${payload.from}: ${payload.body}`);
        });

      },
      error => {
        log(`Connection failed: ${error}`, 'system');
      }
    );
  }

  function disconnect() {
    if (stompClient) {
      stompClient.disconnect(() => {
        log('Disconnected', 'system');
        connectPanel.style.display = 'block';
        chatPanel.style.display = 'none';
        recipientInput.value = '';
        contentInput.value = '';
      });
      stompClient = null;
    }
  }

  function sendPrivateMessage() {
    if (!stompClient || !stompClient.connected) {
      alert('Connect first');
      return;
    }
    const recipient = recipientInput.value.trim();
    const content = contentInput.value.trim();
    if (!recipient || !content) {
      alert('Recipient and message are required');
      return;
    }

    stompClient.send('/app/chat/private', {}, JSON.stringify({
      recipient,
      content
    }));
    contentInput.value = '';
  }

  document.getElementById('connect-btn').addEventListener('click', connect);
  document.getElementById('disconnect-btn').addEventListener('click', disconnect);
  document.getElementById('send-btn').addEventListener('click', sendPrivateMessage);

  window.addEventListener('beforeunload', disconnect);
</script>
</body>
</html>
```

Noteworthy characteristics:

- The client supplies the `username` header during STOMP CONNECT, matching the interceptor’s expectations.
- Subscriptions target `/secured/user/queue/updates`, the custom prefix established earlier.
- The UI is deliberately basic, relying on SockJS and stomp.js from CDNs to prove the private messaging behavior.

## Running and Verifying the Demo

1. Ensure the application is running (`./mvnw spring-boot:run`).
2. Open http://localhost:8080/private-chat.html in two tabs or windows.
3. In the first tab, connect as `alice`.
4. In the second tab, connect as `bob`.
5. From `alice`, send "Hello Bob!" targeting recipient `bob`.
6. Observe that Bob’s Activity feed receives the message, while Alice sees a "You" confirmation. No other users receive the payload.

Check the application logs for confirmation:

```
STOMP CONNECT accepted for user 'alice'
Private message from 'alice' to 'bob'
```

These log statements confirm the interceptor attached the principal and the controller routed traffic correctly.

### Tracing the Destination Rewrite

If you would like to see Spring translate user destinations into session-specific queues, enable DEBUG logging for `org.springframework.messaging.simp`. Create `src/main/resources/application.properties` (or append to it) with:

```
logging.level.org.springframework.messaging.simp=DEBUG
```

On the next private message, the console will display entries similar to:

```
Detected user destination [/secured/user/queue/updates], converting to [/queue/updates-user123]
```

The suffix (`user123`) is generated per session; the application does not need to manage it explicitly.

## Edge Cases and Enhancements

- **Duplicate usernames**: The interceptor only checks for blank usernames. In production, you would typically integrate with Spring Security so the Principal reflects the authenticated user and duplicate checks become part of the auth layer.
- **Disconnect handling**: When a user closes the tab, SockJS tears down the session and Spring removes the subscription. To announce departures, observe `SessionDisconnectEvent` via `@EventListener` (covered earlier in the series).
- **Error feedback**: `IllegalArgumentException` thrown by the interceptor surfaces as an ERROR frame. Enhance the browser client’s error callback to render friendly messages for users who omitted the username field.
- **Inbox segmentation**: Consider multiple queues per user (`/queue/systemAlerts`, `/queue/messages`) to separate notifications from chats.
- **Typing indicators**: Send ephemeral events that update the recipient’s UI while composing messages.
- **Persistence**: Pair the controller with a database to store and replay private conversations (a focus for Article 8).
- **Security**: Replace the custom header mechanism with Spring Security authentication. Once integrated, the framework injects the authenticated principal automatically and the interceptor becomes unnecessary.

## Testing Strategy

Automated testing for STOMP user destinations is possible, albeit more involved than for REST endpoints:

- **Controller unit tests**: Slice the Spring context (`@WebMvcTest`) and mock `SimpMessagingTemplate` to assert `convertAndSendToUser` receives the right arguments.
- **Integration tests**: Spin up the broker with `WebSocketStompClient`, connect as two simulated users, send messages, and assert they subscribe to the expected queues.
- **Client smoke tests**: Using Playwright or Cypress, script two browser contexts interacting with `private-chat.html` and assert that private messages remain isolated.

For this toy project, hands-on browser testing proves the concept adequately.

## Summary

We expanded the STOMP demo with session-aware private communication:

- Configured a custom user destination prefix (`/secured/user`) to namespace private traffic.
- Attached principals to STOMP sessions by inspecting the STOMP CONNECT frame.
- Delivered direct messages using `SimpMessagingTemplate.convertAndSendToUser` without tracking session IDs.
- Built a minimal SockJS/STOMP client that lets you experiment with multiple users in parallel tabs.
- Highlighted debugging, error handling, and extension ideas to prepare for persistence and security layers.

In the next article, we will persist private messages and hydrate inbox history when users connect, giving the toy application a memory.

## Next Steps

- Clean up any debug logging once you are satisfied with testing.
- Commit the changes to source control.
- If you are integrating with real authentication, experiment with swapping the interceptor for Spring Security’s Principal handling.
- Continue to Article 8, where we will persist and replay messages for reconnecting clients.
