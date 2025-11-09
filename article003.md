# Mastering Two-Way Updates: The Classic STOMP Chat Room Case Study

Two-way communication is where WebSocket and STOMP truly shine. In the previous articles we experimented with user-triggered requests and server-driven updates, but each interaction was still fundamentally one-directional. In this third installment we will build a multi-user chat room that demonstrates full bidirectional messaging: every browser sends messages through the Spring Boot application, which validates and broadcasts them to all subscribers in real time.

Chat applications are fantastic teaching tools because they expose every aspect of the STOMP pipeline: connection lifecycle, message routing, error handling, DOM updates, and even basic persistence if you choose to store history. We will keep the implementation simple enough to understand at a glance while still covering best practices you can reuse in more serious projects.

By the end of this article you will:

- Implement a `@MessageMapping` method that receives messages, enriches them, and distributes them using `@SendTo`.
- Add server-side validation to reject invalid payloads or sanitize incoming text.
- Maintain an in-memory message history to give new clients context when they join.
- Build a lightweight HTML client that supports nicknames, message entry, typing indicators, and connection controls.

Let’s dive in.

## Project Recap

We continue working inside the same Spring Boot project we created in Articles 1 and 2. Ensure you still have the WebSocket starter dependency and the `WebSocketConfig` class registering `/ws` as the STOMP endpoint, `/topic` as the broker prefix, and `/app` as the application prefix.

If you cloned the repository from earlier in the series, consider creating a new Git branch (e.g., `feature/chat-room`) so you can track the instructions cleanly. 

### Dependencies

One small change you'll need to do in the `pom.xml` file is to add `spring-boot-starter-validation` to the dependencies. This will come in handy during this article when we touch a bit on server-side input validation.

``` xml
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-validation</artifactId>
		</dependency>
```

Run `./mvnw spring-boot:run` once to make sure everything compiles.

## Modeling Chat Messages

First, define a payload class to represent the chat messages exchanged between client and server. We will store the sender nickname, the message content, a timestamp, and a generated message ID.

```java
package com.example.stompchat.chat;

import java.time.Instant;
import java.util.UUID;

public record ChatMessage(String id,
                          String sender,
                          String content,
                          Instant timestamp,
                          MessageType type) {

    public static ChatMessage userMessage(String sender, String content) {
        return new ChatMessage(UUID.randomUUID().toString(), sender, content, Instant.now(), MessageType.CHAT);
    }

    public static ChatMessage systemMessage(String content) {
        return new ChatMessage(UUID.randomUUID().toString(), "system", content, Instant.now(), MessageType.SYSTEM);
    }

    public enum MessageType {
        CHAT, SYSTEM
    }
}
```

We categorize messages as either user-generated chats or system notifications (for example, announcing when someone joins or leaves). This makes rendering easier on the client because we can style system messages differently.

Next, create a DTO to represent the payload the client sends. We do not want the browser to provide IDs or timestamps; the server supplies those fields to maintain integrity.

```java
package com.example.stompchat.chat;

public record ChatMessageRequest(String sender, String content) {}
```

## Managing Chat History

To keep the article focused we will store message history in memory using a thread-safe deque. When a new user connects we can send them the last 20 messages so they have context. In a production system you would persist history in a database or use a message replay mechanism, but the in-memory approach suits our toy project.

```java
package com.example.stompchat.chat;

import java.time.Instant;
import java.util.ArrayDeque;
import java.util.Deque;
import java.util.List;
import java.util.stream.Collectors;

import org.springframework.stereotype.Component;

@Component
public class ChatHistory {

    private static final int MAX_HISTORY = 50;
    private final Deque<ChatMessage> messages = new ArrayDeque<>();

    public synchronized void add(ChatMessage message) {
        if (messages.size() >= MAX_HISTORY) {
            messages.removeFirst();
        }
        messages.addLast(message);
    }

    public synchronized List<ChatMessage> snapshot(int limit) {
        return messages.stream()
                .skip(Math.max(0, messages.size() - limit))
                .collect(Collectors.toList());
    }

    public synchronized List<ChatMessage> all() {
        return List.copyOf(messages);
    }

    public synchronized void addSystemMessage(String content) {
        add(ChatMessage.systemMessage(content));
    }
}
```

We provide methods to add new messages, fetch a snapshot of the most recent entries, and append system notifications. The `synchronized` keyword keeps operations atomic—adequate for a toy app with modest traffic.

## Creating the Chat Controller

Now we write the controller that accepts incoming chat messages and broadcasts them to all subscribers. This is the core of the article.

```java
package com.example.stompchat.chat;

import jakarta.validation.Valid;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.Size;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.messaging.handler.annotation.MessageMapping;
import org.springframework.messaging.handler.annotation.Payload;
import org.springframework.messaging.handler.annotation.SendTo;
import org.springframework.messaging.simp.SimpMessagingTemplate;
import org.springframework.stereotype.Controller;

@Controller
public class ChatController {

    private static final Logger log = LoggerFactory.getLogger(ChatController.class);

    private final ChatHistory history;
    private final SimpMessagingTemplate messagingTemplate;

    public record TypedRequest(@NotBlank @Size(max = 40) String sender,
                               @NotBlank @Size(max = 500) String content) {}

    public record TypingRequest(@NotBlank @Size(max = 40) String sender) {}

    public ChatController(ChatHistory history, SimpMessagingTemplate messagingTemplate) {
        this.history = history;
        this.messagingTemplate = messagingTemplate;
    }

    @MessageMapping("/chat")
    @SendTo("/topic/chat")
    public ChatMessage handleChat(@Payload @Valid TypedRequest request) {
        ChatMessage message = ChatMessage.userMessage(request.sender().trim(), sanitize(request.content()));
        history.add(message);
        log.info("Broadcasting message from {}", message.sender());
        return message;
    }

    @MessageMapping("/typing")
    public void handleTyping(@Payload @Valid TypingRequest request) {
        messagingTemplate.convertAndSend("/topic/chat.typing", request.sender());
    }

    private String sanitize(String content) {
        return content.replaceAll("\\s+", " ").trim();
    }
}```

Key points:

- We use Bean Validation annotations to ensure the sender and content are non-empty and within reasonable length limits. If validation fails, Spring will raise a `MethodArgumentNotValidException` that we can handle globally (discussed shortly).
- The `@SendTo("/topic/chat")` annotation automatically broadcasts the resulting `ChatMessage` to all subscribers. The return value is serialized as JSON and wrapped in a STOMP MESSAGE frame.
- Typing indicators use a separate destination `/topic/chat.typing`. We do not need a return value; the method sends a message directly through `SimpMessagingTemplate` so every listener sees who is currently typing.

### Handling Validation Errors Gracefully

When validation fails we want the client to receive a meaningful error rather than silently failing. Add a controller advice:

```java
package com.example.stompchat.chat;

import java.util.Map;

import org.springframework.messaging.handler.annotation.MessageExceptionHandler;
import org.springframework.messaging.simp.annotation.SendToUser;
import org.springframework.stereotype.Controller;

@Controller
public class ChatExceptionHandler {

    @MessageExceptionHandler
    @SendToUser("/queue/errors")
    public Map<String, String> handleValidation(Exception exception) {
        return Map.of("error", exception.getMessage());
    }
}
```

`@SendToUser` routes the response to the specific client that triggered the error using the STOMP user queue. We will subscribe to `/user/queue/errors` on the client side so that each browser sees its own validation messages.

## Serving Initial History with @SubscribeMapping

We have not yet introduced `@SubscribeMapping` formally—that’s the focus of Article 5—but even now we can preview how it helps. When a client first subscribes to `/topic/chat`, we can send them the latest conversation history immediately:

```java
package com.example.stompchat.chat;

import java.util.List;

import org.springframework.messaging.simp.annotation.SubscribeMapping;
import org.springframework.stereotype.Controller;

@Controller
public class ChatHistoryController {

    private final ChatHistory history;

    public ChatHistoryController(ChatHistory history) {
        this.history = history;
    }

    @SubscribeMapping("/topic/chat")
    public List<ChatMessage> historySnapshot() {
        return history.snapshot(20);
    }
}
```

`@SubscribeMapping` sends the returned payload only to the subscribing client, giving them context before live messages start flowing. We keep the limit at 20 to avoid overwhelming the UI.

## Updating WebSocket Configuration (Optional Heartbeats)

For chat applications with long-lived connections, heartbeats help detect hung connections. You can enable outbound heartbeats in your `WebSocketConfig`:

```java
@Override
public void configureMessageBroker(MessageBrokerRegistry registry) {
    registry.enableSimpleBroker("/topic", "/queue").setHeartbeatValue(new long[]{10000, 10000});
    registry.setApplicationDestinationPrefixes("/app");
}
```

The `setHeartbeatValue` call enables heartbeats on both directions (server-to-client and client-to-server) at 10-second intervals. The browser STOMP client needs matching settings, which we will configure shortly.

## Crafting the Chat Room HTML Client

Create `src/main/resources/static/chat.html` with the following markup and script:

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>STOMP Chat Room</title>
    <style>
        body { font-family: system-ui, sans-serif; margin: 0; background: #f4f5f7; }
        header { background: #0052cc; color: white; padding: 1rem 2rem; display: flex; justify-content: space-between; align-items: center; }
        main { display: grid; grid-template-columns: 2fr 1fr; gap: 1rem; padding: 1rem 2rem; }
        #messages { background: white; border-radius: 6px; padding: 1rem; height: 60vh; overflow-y: auto; display: flex; flex-direction: column-reverse; }
        .message { margin-bottom: 1rem; padding-bottom: 0.5rem; border-bottom: 1px solid #e1e4e8; }
        .message.system { color: #5e6c84; font-style: italic; }
        .meta { font-size: 0.85rem; color: #5e6c84; }
        #users { background: white; border-radius: 6px; padding: 1rem; height: 60vh; overflow-y: auto; }
        #status { margin-top: 0.5rem; font-weight: 600; }
        form { display: flex; gap: 0.5rem; margin-top: 1rem; }
        input, button, textarea { font-family: inherit; font-size: 1rem; }
        textarea { flex: 1; resize: vertical; min-height: 4rem; padding: 0.5rem; }
        #typing { font-size: 0.9rem; color: #5e6c84; margin-top: 0.5rem; min-height: 1.5rem; }
        #errors { color: #c53030; margin-top: 0.75rem; white-space: pre-wrap; }
    </style>
</head>
<body>
<header>
    <div>
        <h1>Spring STOMP Chat</h1>
        <div id="status">Disconnected</div>
    </div>
    <div>
        <label for="nickname">Nickname</label>
        <input id="nickname" value="guest" />
        <button id="connect">Connect</button>
        <button id="disconnect" disabled>Disconnect</button>
    </div>
</header>

<main>
    <section>
        <div id="messages"></div>
        <div id="typing"></div>
        <div id="errors"></div>
        <form id="chat-form">
            <textarea id="message" placeholder="Type a message..." maxlength="500"></textarea>
            <button type="submit">Send</button>
        </form>
    </section>
    <aside>
        <h2>Online</h2>
        <ul id="users"></ul>
    </aside>
</main>

<script type="module">
    import { Client } from 'https://cdn.jsdelivr.net/npm/@stomp/stompjs@7.0.0/esm6/index.min.js';

    const connectBtn = document.getElementById('connect');
    const disconnectBtn = document.getElementById('disconnect');
    const status = document.getElementById('status');
    const nicknameInput = document.getElementById('nickname');
    const form = document.getElementById('chat-form');
    const messageInput = document.getElementById('message');
    const messagesDiv = document.getElementById('messages');
    const typingDiv = document.getElementById('typing');
    const usersList = document.getElementById('users');
    const errorsDiv = document.getElementById('errors');

    let stompClient;
    let typingTimeout;
    const typingUsers = new Map();

    const renderStatus = text => { status.textContent = text; };

    function appendMessage({ sender, content, timestamp, type }) {
        const container = document.createElement('div');
        container.className = `message ${type.toLowerCase()}`;
        const meta = document.createElement('div');
        meta.className = 'meta';
        meta.textContent = type === 'SYSTEM'
            ? `${new Date(timestamp).toLocaleTimeString()}`
            : `${sender} • ${new Date(timestamp).toLocaleTimeString()}`;
        const body = document.createElement('div');
        body.textContent = content;
        container.append(body, meta);
        messagesDiv.prepend(container);
    }

    function updateTypingIndicator() {
        const active = Array.from(typingUsers.keys());
        if (!active.length) {
            typingDiv.textContent = '';
            return;
        }
        const names = active.slice(0, 3).join(', ');
        typingDiv.textContent = active.length > 3
            ? `${names}, and others are typing...`
            : `${names} ${active.length === 1 ? 'is' : 'are'} typing...`;
    }

    function notifyTyping() {
        if (!stompClient || !stompClient.connected) return;
        stompClient.publish({ destination: '/app/typing', body: JSON.stringify({ sender: nicknameInput.value, content: '' }) });
        clearTimeout(typingTimeout);
        typingTimeout = setTimeout(() => typingUsers.delete(nicknameInput.value), 4000);
    }

    function subscribeToTopics() {
        stompClient.subscribe('/topic/chat', frame => {
            const payload = JSON.parse(frame.body);
            if (Array.isArray(payload)) {
                payload.forEach(appendMessage);
            } else {
                appendMessage(payload);
            }
        });

        stompClient.subscribe('/topic/chat.typing', frame => {
            const user = frame.body;
            if (user === nicknameInput.value) {
                return;
            }
            typingUsers.set(user, Date.now());
            updateTypingIndicator();
            setTimeout(() => {
                typingUsers.delete(user);
                updateTypingIndicator();
            }, 4000);
        });

        stompClient.subscribe('/user/queue/errors', frame => {
            const body = JSON.parse(frame.body);
            errorsDiv.textContent = body.error;
            setTimeout(() => errorsDiv.textContent = '', 4000);
        });
    }

    connectBtn.addEventListener('click', () => {
        if (stompClient && stompClient.connected) {
            return;
        }

        stompClient = new Client({
            brokerURL: `ws://${window.location.host}/ws`,
            reconnectDelay: 5000,
            heartbeatIncoming: 10000,
            heartbeatOutgoing: 10000,
            debug: str => console.debug(str)
        });

        stompClient.onConnect = () => {
            renderStatus(`Connected as ${nicknameInput.value}`);
            connectBtn.disabled = true;
            disconnectBtn.disabled = false;
            nicknameInput.disabled = true;
            subscribeToTopics();
            errorsDiv.textContent = '';
            stompClient.publish({ destination: '/app/chat', body: JSON.stringify({ sender: 'system', content: `${nicknameInput.value} joined` }) });
        };

        stompClient.onStompError = frame => {
            errorsDiv.textContent = frame.headers['message'];
        };

        stompClient.onWebSocketClose = () => {
            renderStatus('Disconnected');
            connectBtn.disabled = false;
            disconnectBtn.disabled = true;
            nicknameInput.disabled = false;
        };

        stompClient.activate();
    });

    disconnectBtn.addEventListener('click', () => {
        if (!stompClient) return;
        stompClient.deactivate();
        renderStatus('Disconnected');
        connectBtn.disabled = false;
        disconnectBtn.disabled = true;
        nicknameInput.disabled = false;
    });

    form.addEventListener('submit', event => {
        event.preventDefault();
        if (!stompClient || !stompClient.connected) {
            errorsDiv.textContent = 'Connect before sending.';
            return;
        }
        const content = messageInput.value.trim();
        if (!content) {
            errorsDiv.textContent = 'Message cannot be empty.';
            return;
        }
        stompClient.publish({
            destination: '/app/chat',
            body: JSON.stringify({ sender: nicknameInput.value, content })
        });
        messageInput.value = '';
        notifyTyping();
    });

    messageInput.addEventListener('input', () => {
        notifyTyping();
    });
</script>
</body>
</html>
```

The client:

- Connects and subscribes to three destinations: `/topic/chat` for messages, `/topic/chat.typing` for typing indicators, and `/user/queue/errors` for personal error messages.
- Displays initial history sent by the `@SubscribeMapping` controller and appends new messages as they arrive.
- Publishes a message whenever the user submits the form. We trim whitespace and enforce the same length restriction as the server to minimize validation errors.
- Uses a simple timeout mechanism to show typing indicators for four seconds after a typing notification is received.

Feel free to tweak the styling or the UI layout. Because this is a toy project, we intentionally avoid frameworks or build tools. Everything runs in a single HTML file you can open directly from the Spring Boot static resources.

## Running the Chat Demo

Start or restart the Spring Boot application:

```bash
./mvnw spring-boot:run
```

Open `http://localhost:8080/chat.html` in two browser windows, set different nicknames, and click “Connect” in each. You should see system messages announcing that each user joined, followed by any messages you exchange. Typing in one window shows the typing indicator in the other. If you try to submit an empty message, the server rejects it and the client displays the error from `/user/queue/errors`.

Explore the application by opening additional windows, disconnecting, and reconnecting. Every action flows through the STOMP pipeline:

1. Client sends a `SEND` frame to `/app/chat` with the message content.
2. Spring routes the payload to `ChatController.handleChat`.
3. The controller validates, sanitizes, enriches, and returns a `ChatMessage`.
4. `@SendTo` broadcasts the result via `/topic/chat`.
5. Clients receive the `MESSAGE` frame and render it immediately.

## Testing the Message Flow with Integration Tests

For extra confidence you can write a Spring Boot test that leverages `spring-messaging`’s `StompSession` support. The following snippet demonstrates how to connect to the WebSocket endpoint during a test and assert that messages echo as expected:

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@AutoConfigureWebSocket
class ChatControllerTest {

    @LocalServerPort
    private int port;

    @Autowired
    private WebSocketStompClient stompClient;

    @Test
    void broadcastChatMessage() throws Exception {
        StompSession session = stompClient.connect(String.format("ws://localhost:%d/ws", port), new StompSessionHandlerAdapter() {}).get(1, TimeUnit.SECONDS);

        CompletableFuture<ChatMessage> future = new CompletableFuture<>();
        session.subscribe("/topic/chat", new StompFrameHandler() {
            @Override
            public Type getPayloadType(StompHeaders headers) {
                return ChatMessage.class;
            }

            @Override
            public void handleFrame(StompHeaders headers, Object payload) {
                future.complete((ChatMessage) payload);
            }
        });

        session.send("/app/chat", new ChatController.TypedRequest("tester", "hello"));
        ChatMessage response = future.get(2, TimeUnit.SECONDS);

        assertThat(response.sender()).isEqualTo("tester");
        assertThat(response.content()).isEqualTo("hello");
    }
}
```

This test uses Spring’s `WebSocketStompClient` and asserts that the broadcast message matches expectations. While we will not run the test in this article, including it in the codebase helps reinforce the message flow.

## Troubleshooting the Chat Application

Common issues include:

- **Validation errors not reaching the client:** Ensure you subscribed to `/user/queue/errors` after connecting. User destinations are resolved per-session; subscriptions must happen after the STOMP connection is established.
- **Typing indicators never appear:** Confirm that the `handleTyping` method publishes to `/topic/chat.typing` and that the client subscribes to the same destination. Also verify that you call `notifyTyping()` in the `input` event handler.
- **History not loading:** The `@SubscribeMapping` method only fires on the initial subscription. Make sure the client subscribes to `/topic/chat` after the connection establishes; subscribing before STOMP activation results in missed history.
- **Duplicate system messages on connect:** Our simple client publishes a system message announcing the user’s arrival. In a more polished implementation you would move this logic server-side, possibly triggered by a session connect event. For now, the manual approach keeps the demo explicit.

## Next Steps

You now have a fully functional multiparty chat room powered by Spring Boot, STOMP, and a plain HTML client. This exercise cements the concepts of `@MessageMapping`, `@SendTo`, validation, and client subscriptions. It also previews user-specific messaging via `/user` queues and `@SubscribeMapping`, topics we will explore in more depth later in the series.

In the next article we will broaden compatibility by integrating SockJS fallbacks so your chat application works even in environments that lack native WebSocket support. Before you move on, consider experimenting with the chat project:

- Add emoji support or markdown rendering (sanitize input carefully!).
- Display avatars or random colors per user.
- Store chat history to disk or integrate with a database for persistence.
- Implement buttons to filter system messages or search through the message list.

Every enhancement builds your intuition for real-time messaging. Keep this base project handy—we will continue to refine it as we progress through the series.

