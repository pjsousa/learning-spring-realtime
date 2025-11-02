# Mastering Two-Way Updates: The Classic STOMP Chat Room Case Study

## Series Overview

- Article 1 – Laying the Rails: Getting Started with Spring Boot WebSockets — establish the baseline STOMP chat skeleton, covering project bootstrapping, the messaging broker, and a single text broadcast.
- Article 2 – Event-Driven Thinking: Understanding STOMP Frames and Broker Routing — dive deep into the STOMP protocol, travel through inbound/outbound channels, and instrument frame logging.
- Article 3 – Mastering Two-Way Updates: The Classic STOMP Chat Room Case Study — build a polished public chat app with validation, timestamps, and a full browser client.
- Article 4 – Private Conversations: Direct Messaging with User Destinations — extend the chat to one-on-one messaging with authenticated principals.
- Article 5 – Presence Matters: User Registry, Typing Indicators, and Status Updates — track join/leave events, build presence dashboards, and surface typing notifications.
- Article 6 – Reliability Basics: Reconnect Strategies, Heartbeats, and Error Handling — harden the client connection logic and surface broker-level exceptions.
- Article 7 – Testing the Pipeline: Integration Tests for STOMP Endpoints — automate WebSocket interactions with `WebSocketStompClient` and Spring’s testing support.
- Article 8 – Scaling Out: External Message Brokers and Cluster Awareness — swap the simple broker for RabbitMQ, configure load-balancing, and discuss sticky sessions.
- Article 9 – Security Patterns: JWT Authentication and Channel Interceptors — protect endpoints, propagate identity, and enforce publish/subscribe rules.
- Article 10 – Beyond Chat: Building Real-Time Dashboards and Collaborative Widgets — showcase alternative STOMP-driven UIs, partial updates, and performance tuning.

## Why This Article Matters

You’ve already seen WebSocket basics and how STOMP frames travel through Spring’s messaging pipeline. Now we fulfil the classic promise of WebSockets: real-time, multi-user communication. The goal is a browser-based group chat where every participant receives updates instantly. Along the way we will:

- Craft a Spring Boot backend exposing a `/app/chat` `@MessageMapping` that validates inbound messages and rebroadcasts them on `/topic/public`.
- Wire up the STOMP broker configuration that routes frames client → `clientInboundChannel` → controller → `brokerChannel` → subscribed clients.
- Build a static HTML/JavaScript client that connects, joins the room, sends messages, and renders incoming events.
- Package everything so readers can clone, run `./mvnw spring-boot:run`, open the provided HTML file, and start chatting immediately—even with multiple browser tabs.

Expect to implement each file from scratch, see the message flow at every stage, and understand how the client and server collaborate for two-way updates.

## Prerequisites and Project Setup

Assumptions:

- Java 17+ installed.
- Maven or Gradle available (we’ll use Maven here).
- Familiarity with Spring Boot basics (covered in Article 1) and STOMP fundamentals (Article 2).

Create the project:

1. Use Spring Initializr or Maven archetype with dependencies: `spring-boot-starter-websocket`, `spring-boot-starter-web`, `spring-boot-starter-validation`, and optionally `spring-boot-starter-thymeleaf` if you prefer templating (we’ll stick to static HTML).
2. Structure:
   - `src/main/java/com/example/chat/` for application classes.
   - `src/main/resources/static/` for our HTML client.
   - `src/main/resources/application.yml` for broker tuning (optional for defaults).

Sample `pom.xml` dependencies section:

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
        <artifactId>spring-boot-starter-validation</artifactId>
    </dependency>
    <dependency>
        <groupId>org.webjars</groupId>
        <artifactId>stomp-websocket</artifactId>
        <version>2.3.3</version>
    </dependency>
    <dependency>
        <groupId>org.webjars</groupId>
        <artifactId>sockjs-client</artifactId>
        <version>1.5.1</version>
    </dependency>
</dependencies>
```

`stomp-websocket` and `sockjs-client` WebJars make it trivial to serve the JS libraries without hitting a CDN, keeping the article standalone.

Initialize the application:

```java
package com.example.chat;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class ClassicChatApplication {

    public static void main(String[] args) {
        SpringApplication.run(ClassicChatApplication.class, args);
    }
}
```

## Configuring the STOMP Message Broker

The broker configuration ensures STOMP destinations behave predictably. We use `@EnableWebSocketMessageBroker` to register the STOMP endpoint and configure the broker.

```java
package com.example.chat.config;

import org.springframework.context.annotation.Configuration;
import org.springframework.messaging.simp.config.MessageBrokerRegistry;
import org.springframework.web.socket.config.annotation.*;

@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/ws")
                .setAllowedOriginPatterns("*")
                .withSockJS();
    }

    @Override
    public void configureMessageBroker(MessageBrokerRegistry registry) {
        registry.setApplicationDestinationPrefixes("/app");
        registry.enableSimpleBroker("/topic");
    }
}
```

Key choices:

- Endpoint `/ws` is where clients perform the SockJS/STOMP handshake.
- `setAllowedOriginPatterns("*")` simplifies local testing across tabs; later we’ll tighten CORS.
- Application destinations start with `/app`; anything sent to `/app/chat` goes to our `@MessageMapping`.
- The simple broker handles destinations under `/topic`, broadcasting to subscribers.

## Defining Message Payloads

We need a payload format for chat messages. Each message includes the sender, content, timestamp, and type (e.g., `CHAT`, `JOIN`, `LEAVE`) for simple system events.

```java
package com.example.chat.model;

import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.Size;

import java.time.Instant;

public class ChatMessage {

    public enum MessageType {
        CHAT,
        JOIN,
        LEAVE
    }

    private MessageType type = MessageType.CHAT;

    @NotBlank
    @Size(max = 40)
    private String sender;

    @NotBlank
    @Size(max = 500)
    private String content;

    private Instant timestamp = Instant.now();

    // getters and setters omitted for brevity
}
```

Notes:

- Bean validation ensures empty messages or overly long inputs are rejected before broadcasting.
- `Instant` timestamp is generated server-side so all clients see consistent ordering.

If you prefer Lombok, you can use `@Data`, but keeping dependencies minimal makes the article more accessible.

## Creating the Chat Controller

The controller handles inbound messages via `@MessageMapping`. It validates the payload, applies server-side augmentation (timestamp, sanitized sender), and returns a message to broadcast.

```java
package com.example.chat.controller;

import com.example.chat.model.ChatMessage;
import jakarta.validation.Valid;
import org.springframework.messaging.handler.annotation.MessageMapping;
import org.springframework.messaging.handler.annotation.Payload;
import org.springframework.messaging.handler.annotation.SendTo;
import org.springframework.stereotype.Controller;
import org.springframework.validation.annotation.Validated;

import java.time.Instant;

@Controller
@Validated
public class ChatController {

    @MessageMapping("/chat")
    @SendTo("/topic/public")
    public ChatMessage broadcast(@Payload @Valid ChatMessage message) {
        message.setTimestamp(Instant.now());
        if (message.getType() == null) {
            message.setType(ChatMessage.MessageType.CHAT);
        }
        message.setSender(stripDangerous(message.getSender()));
        message.setContent(stripDangerous(message.getContent()));
        return message;
    }

    private String stripDangerous(String value) {
        return value == null ? null : value.replaceAll("<", "&lt;")
                                           .replaceAll(">", "&gt;");
    }
}
```

Breakdown:

- `@MessageMapping("/chat")` handles `SEND` frames sent to `/app/chat`.
- `@SendTo("/topic/public")` broadcasts the return value to subscribers of `/topic/public`.
- Validation ensures the `@Payload` respects constraints; invalid messages trigger a STOMP `ERROR` frame.
- Sanitization prevents basic HTML injection since the client will render raw text.

## Tracking Session Events

To broadcast JOIN/LEAVE notifications, listen for session connect and disconnect events.

```java
package com.example.chat.events;

import com.example.chat.model.ChatMessage;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.context.event.EventListener;
import org.springframework.messaging.simp.SimpMessagingTemplate;
import org.springframework.messaging.simp.stomp.StompHeaderAccessor;
import org.springframework.messaging.support.GenericMessage;
import org.springframework.stereotype.Component;
import org.springframework.web.socket.messaging.SessionConnectEvent;
import org.springframework.web.socket.messaging.SessionDisconnectEvent;

import java.time.Instant;
import java.util.Optional;

@Component
public class WebSocketEventListener {

    private static final Logger log = LoggerFactory.getLogger(WebSocketEventListener.class);

    private final SimpMessagingTemplate messagingTemplate;

    public WebSocketEventListener(SimpMessagingTemplate messagingTemplate) {
        this.messagingTemplate = messagingTemplate;
    }

    @EventListener
    public void handleWebSocketConnectListener(SessionConnectEvent event) {
        log.info("New WebSocket connection: {}", event.getUser());
    }

    @EventListener
    public void handleWebSocketDisconnectListener(SessionDisconnectEvent event) {
        StompHeaderAccessor accessor = StompHeaderAccessor.wrap(event.getMessage());
        String username = Optional.ofNullable(accessor.getSessionAttributes())
                                  .map(map -> map.get("username"))
                                  .map(Object::toString)
                                  .orElse("Anonymous");

        ChatMessage leaveMessage = new ChatMessage();
        leaveMessage.setType(ChatMessage.MessageType.LEAVE);
        leaveMessage.setSender(username);
        leaveMessage.setContent(username + " left the chat");
        leaveMessage.setTimestamp(Instant.now());

        messagingTemplate.convertAndSend("/topic/public", leaveMessage);
    }
}
```

For a basic public chat where no login is required, we can store the username in the session attributes during the first message (the client will send a JOIN frame). This listener ensures we emit a LEAVE message felt by everyone.

## Understanding the Message Flow

Let’s map the travel path of messages:

1. **Client SEND frame**: The browser sends a `SEND` to `/app/chat` via STOMP when the user submits a message.
2. **clientInboundChannel**: Spring’s `clientInboundChannel` interceptors (if any) receive the frame. In this simple example, it routes straight to our controller.
3. **@Controller processing**: `ChatController.broadcast` validates, sanitizes, stamps the timestamp, and returns the `ChatMessage`.
4. **brokerChannel**: The return value is converted to a STOMP message and passed to `brokerChannel`, where the simple broker fans it out to `/topic/public`.
5. **Subscribed clients**: Every client that previously issued a `SUBSCRIBE` to `/topic/public` receives a `MESSAGE` frame carrying the JSON payload.

Capturing logs helps you visualize the pipeline. Add to `application.yml`:

```yaml
logging:
  level:
    org.springframework.messaging: DEBUG
    org.springframework.web.socket: DEBUG
```

Run the app, send a message, and watch the logs trace the frame’s path through the inbound/outbound channels.

## Building the Browser Client

Place `index.html` under `src/main/resources/static/`. The client must:

- Connect to `/ws` via SockJS + STOMP.
- Prompt for a username.
- Send a JOIN message after connecting.
- Subscribe to `/topic/public`.
- Render incoming chat messages with simple CSS.
- Provide an input to send new messages.

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Classic STOMP Chat</title>
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/milligram/1.4.1/milligram.min.css">
    <style>
        body { margin: 2rem; }
        #chat { height: 400px; overflow-y: auto; border: 1px solid #ccc; padding: 1rem; }
        .system { color: #888; font-style: italic; }
        .message { margin-bottom: 0.5rem; }
        .sender { font-weight: bold; margin-right: 0.5rem; }
        #status { margin-bottom: 1rem; }
    </style>
</head>
<body>
    <h1>Classic STOMP Chat</h1>
    <section id="status"></section>
    <section id="chat"></section>

    <form id="messageForm">
        <label for="message">Message</label>
        <input type="text" id="message" autocomplete="off" required>
        <button type="submit">Send</button>
    </form>

    <script src="/webjars/sockjs-client/1.5.1/sockjs.min.js"></script>
    <script src="/webjars/stomp-websocket/2.3.3/stomp.min.js"></script>
    <script>
        let stompClient = null;
        let username = null;

        const chat = document.getElementById('chat');
        const status = document.getElementById('status');
        const messageForm = document.getElementById('messageForm');
        const messageInput = document.getElementById('message');

        function connect() {
            username = prompt('Enter a chat nickname:') || 'Anonymous';
            status.textContent = 'Connecting as ' + username + '...';

            const socket = new SockJS('/ws');
            stompClient = Stomp.over(socket);
            stompClient.debug = null; // silence console chatter

            stompClient.connect({}, onConnected, onError);
        }

        function onConnected() {
            status.textContent = 'Connected as ' + username;
            stompClient.subscribe('/topic/public', onMessageReceived);

            const joinMessage = {
                sender: username,
                content: username + ' joined the chat',
                type: 'JOIN'
            };
            stompClient.send('/app/chat', {}, JSON.stringify(joinMessage));
        }

        function onError(error) {
            status.textContent = 'Connection error. Retry or refresh the page.';
            console.error('STOMP error', error);
        }

        function onMessageReceived(payload) {
            const message = JSON.parse(payload.body);
            const messageElement = document.createElement('div');
            messageElement.classList.add('message');

            const meta = document.createElement('span');
            meta.classList.add('sender');
            meta.textContent = formatSender(message);
            messageElement.appendChild(meta);

            const text = document.createElement('span');
            text.innerHTML = sanitize(message.content);
            messageElement.appendChild(text);

            if (message.type && message.type !== 'CHAT') {
                messageElement.classList.add('system');
            }

            chat.appendChild(messageElement);
            chat.scrollTop = chat.scrollHeight;
        }

        function formatSender(message) {
            const time = message.timestamp ? new Date(message.timestamp).toLocaleTimeString() : '';
            return '[' + time + '] ' + (message.sender || 'System');
        }

        function sanitize(value) {
            const div = document.createElement('div');
            div.innerText = value;
            return div.innerHTML;
        }

        messageForm.addEventListener('submit', function (event) {
            event.preventDefault();
            const messageContent = messageInput.value.trim();
            if (!messageContent) return;

            const chatMessage = {
                sender: username,
                content: messageContent,
                type: 'CHAT'
            };

            stompClient.send('/app/chat', {}, JSON.stringify(chatMessage));
            messageInput.value = '';
        });

        window.addEventListener('beforeunload', () => {
            if (stompClient) {
                const leaveMessage = {
                    sender: username,
                    content: username + ' left the chat',
                    type: 'LEAVE'
                };
                stompClient.send('/app/chat', {}, JSON.stringify(leaveMessage));
                stompClient.disconnect();
            }
        });

        connect();
    </script>
</body>
</html>
```

Highlights:

- The client loads SockJS and STOMP from WebJars (no external network dependency).
- `onConnected` sends an initial JOIN message that the server treats like any chat message; it appears to everyone, including the sender.
- `onMessageReceived` formats timestamps and differentiates system vs. user messages via CSS.
- `beforeunload` sends a LEAVE message, complementing the server-side disconnect listener for robustness.

## Making JOIN and LEAVE Messages Work

Currently, JOIN/LEAVE rely on the client sending a specific `type`. Some refinements:

- When a JOIN message arrives, store the username in the session so the disconnect listener can reference it. Add this snippet in `ChatController.broadcast` before returning:

```java
import org.springframework.messaging.handler.annotation.Header;
import org.springframework.messaging.simp.stomp.StompHeaderAccessor;

@MessageMapping("/chat")
@SendTo("/topic/public")
public ChatMessage broadcast(@Payload @Valid ChatMessage message,
                             @Header("simpSessionId") String sessionId,
                             StompHeaderAccessor headerAccessor) {
    message.setTimestamp(Instant.now());
    if (message.getType() == ChatMessage.MessageType.JOIN) {
        headerAccessor.getSessionAttributes().put("username", message.getSender());
    }
    if (message.getType() == null) {
        message.setType(ChatMessage.MessageType.CHAT);
    }
    message.setSender(stripDangerous(message.getSender()));
    message.setContent(stripDangerous(message.getContent()));
    return message;
}
```

However, injecting `StompHeaderAccessor` directly requires a `@MessageExceptionHandler` or custom argument resolver. Simpler approach: use `SimpMessageHeaderAccessor` inside the method:

```java
SimpMessageHeaderAccessor accessor = SimpMessageHeaderAccessor.create();
```

But the cleanest solution is to modify the method signature:

```java
import org.springframework.messaging.handler.annotation.Header;
import org.springframework.messaging.simp.SimpMessageHeaderAccessor;

@MessageMapping("/chat")
@SendTo("/topic/public")
public ChatMessage broadcast(@Payload @Valid ChatMessage message,
                             SimpMessageHeaderAccessor headerAccessor) {
    message.setTimestamp(Instant.now());
    if (message.getType() == ChatMessage.MessageType.JOIN) {
        headerAccessor.getSessionAttributes().put("username", message.getSender());
    }
    if (message.getType() == null) {
        message.setType(ChatMessage.MessageType.CHAT);
    }
    message.setSender(stripDangerous(message.getSender()));
    message.setContent(stripDangerous(message.getContent()));
    return message;
}
```

`SimpMessageHeaderAccessor` gives access to session attributes, ensuring the disconnect listener finds the username even if the client closes the tab without sending a LEAVE frame.

## Validating Inputs and Handling Errors

If validation fails, Spring triggers a `MethodArgumentNotValidException`. By default, the STOMP session receives an `ERROR` frame, and the client logs it but continues. For a better user experience:

- Implement a `@MessageExceptionHandler` to route validation errors to a dedicated destination (e.g., `/queue/errors/{sessionId}`).
- On the client, subscribe to that queue and display inline error messages.

Example handler:

```java
import org.springframework.messaging.handler.annotation.MessageExceptionHandler;
import org.springframework.messaging.simp.annotation.SendToUser;

@Controller
public class ChatController {

    // ... broadcast code

    @MessageExceptionHandler
    @SendToUser("/queue/errors")
    public String handleValidationExceptions(Exception ex) {
        return ex.getMessage();
    }
}
```

Then, the client subscribes to `/user/queue/errors` after connecting and alerts the user. This keeps the article at manageable complexity while hinting at robust patterns.

## Testing the Application

Manual testing:

1. Run the app: `./mvnw spring-boot:run`.
2. Open two browser tabs at `http://localhost:8080/index.html`.
3. Enter different nicknames and exchange messages.
4. Observe that JOIN/LEAVE notifications appear in both tabs.
5. Watch server logs to see STOMP frame routing.

Automated smoke test (optional but recommended for Article 7):

- Use Spring’s `WebSocketStompClient` in a JUnit test to connect to `/ws`, send a message, and assert that a `CHAT` payload arrives on `/topic/public`.
- Validates that controller mapping and broker config are functioning.

## Extending the Chat (Optional Enhancements)

Encourage experimentation:

- Persist chat history using Spring Data and broadcast the last N messages on connect.
- Add a simple list of active users by maintaining a `Map<sessionId, username>` and broadcasting updates.
- Introduce typing indicators by sending `type: TYPING` messages throttled at the client.
- Secure the room using HTTP authentication and propagate the principal through STOMP (preview of Articles 4 and 9).

## Recap and Next Steps

You now own a fully functional STOMP-powered chat application:

- Messages travel from the client `SEND` frame through Spring’s messaging pipeline to all subscribers without polling.
- Both server and client enforce basic validation and sanitization, keeping the experience trustworthy.
- The static HTML client demonstrates how little code is needed to harness WebSockets when STOMP handles the protocol details.

In Article 4 we’ll build on this foundation to send private messages, requiring us to route frames to user-specific destinations and tie in authentication-aware features. Before moving on, ensure you’re comfortable with:

- The difference between application and broker destinations.
- Injecting `SimpMessageHeaderAccessor` to work with STOMP session attributes.
- Responding to `SessionDisconnectEvent` to keep the UI informed.

Run the app, invite a colleague, and enjoy your new real-time chat room.

