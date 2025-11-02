# Fast Startup: Using @SubscribeMapping for Initial Data Loading and Setup

Real-time applications thrive on instant feedback. In the previous articles we leveraged STOMP topics to stream updates and power interactive chat. Yet, when a client connects it often needs an immediate snapshot of the current state—a list of active users, a recent message history, or the latest configuration. Spring’s `@SubscribeMapping` annotation is tailor-made for this scenario. It allows your application to return a one-time payload to the client that initiated a subscription before the broker begins forwarding broadcast messages.

In this fifth installment of our STOMP toy project series, we will deepen our chat application by delivering a rich initialization experience. Clients will receive a structured bundle of data immediately after subscribing: recent messages, a list of online participants, and personalized configuration. We will also introduce additional patterns for request/reply semantics and discuss how `@SubscribeMapping` compares to other alternatives like REST endpoints or custom application destinations.

By the end of this article you will:

- Understand the lifecycle of a subscription and how `@SubscribeMapping` hooks into it.
- Implement a multi-resource bootstrap payload served via `@SubscribeMapping`.
- Combine subscription initialization with live updates using STOMP destinations and user queues.
- Enhance the HTML client to process the initial payload and render the UI immediately.
- Explore testing strategies and pitfalls when mixing `@SubscribeMapping` with broadcast messaging.

Let’s unlock fast startup for our real-time toy projects.

## Revisiting the Chat Application State

Our chat application already stores message history in memory and tracks typing indicators. To support instant startup we need to track additional state: which users are currently online. We will capture this using `SimpUserRegistry`, a built-in Spring component that monitors sessions and subscriptions. We will also extend `ChatHistory` with a `List<ChatParticipant>` describing active users.

### Modeling the Initialization Payload

Define a DTO that encapsulates everything the client needs immediately after subscribing:

```java
package com.example.stompchat.bootstrap;

import java.util.List;

import com.example.stompchat.chat.ChatMessage;

public record ChatBootstrapPayload(List<ChatMessage> recentMessages,
                                   List<ChatParticipant> participants,
                                   ClientConfiguration configuration) {

    public record ChatParticipant(String username, String sessionId) {}

    public record ClientConfiguration(String theme, boolean notificationsEnabled) {}
}
```

This record bundles three pieces of information:

- The last 30 chat messages so the client can render history.
- The list of currently connected users.
- A fictional client configuration (for example, theme preference) to demonstrate how you can personalize data per user.

In a production system the configuration could come from a database. In our toy project we will generate a simple default based on the user’s nickname.

### Capturing Online Participants

Spring tracks active WebSocket sessions via the `SimpUserRegistry`. Inject it into a new component that converts the registry’s state into our DTO format:

```java
package com.example.stompchat.bootstrap;

import java.util.List;
import java.util.stream.Collectors;

import org.springframework.messaging.simp.user.SimpUserRegistry;
import org.springframework.stereotype.Component;

@Component
public class ParticipantTracker {

    private final SimpUserRegistry userRegistry;

    public ParticipantTracker(SimpUserRegistry userRegistry) {
        this.userRegistry = userRegistry;
    }

    public List<ChatBootstrapPayload.ChatParticipant> activeParticipants() {
        return userRegistry.getUsers().stream()
                .map(user -> new ChatBootstrapPayload.ChatParticipant(user.getName(),
                        user.getSessions().stream().findFirst().map(session -> session.getId()).orElse("unknown")))
                .collect(Collectors.toList());
    }
}
```

The registry organizes users by principal. Later in the series we will associate authenticated principals with STOMP sessions (Article 6). For now, the principal name defaults to the session ID. We extract the first session ID to demonstrate how you can attach metadata to each participant.

## Implementing the Subscription Handler

With our DTOs ready, create a controller that responds to subscription events. When a client subscribes to `/topic/chat.bootstrap`, the method will return the initial payload only to that client. Recall that broadcast messages sent through the broker (`/topic/chat`) still proceed normally; `@SubscribeMapping` is a supplementary hook.

```java
package com.example.stompchat.bootstrap;

import java.security.Principal;

import com.example.stompchat.chat.ChatHistory;

import org.springframework.messaging.handler.annotation.Header;
import org.springframework.messaging.handler.annotation.SubscribeMapping;
import org.springframework.messaging.simp.annotation.SendToUser;
import org.springframework.stereotype.Controller;

@Controller
public class BootstrapController {

    private final ChatHistory history;
    private final ParticipantTracker participantTracker;

    public BootstrapController(ChatHistory history, ParticipantTracker participantTracker) {
        this.history = history;
        this.participantTracker = participantTracker;
    }

    @SubscribeMapping("/topic/chat.bootstrap")
    @SendToUser("/queue/bootstrap")
    public ChatBootstrapPayload onBootstrap(@Header("simpSessionId") String sessionId,
                                            Principal principal) {
        var configuration = new ChatBootstrapPayload.ClientConfiguration(themeFor(principal), true);
        return new ChatBootstrapPayload(history.snapshot(30), participantTracker.activeParticipants(), configuration);
    }

    private String themeFor(Principal principal) {
        if (principal == null) {
            return "classic";
        }
        return principal.getName().hashCode() % 2 == 0 ? "dark" : "light";
    }
}
```

Key details:

- The method listens for subscriptions to `/topic/chat.bootstrap` but responds via `@SendToUser("/queue/bootstrap")`. This ensures the payload is delivered privately to the subscriber rather than broadcast to everyone.
- We accept the `simpSessionId` header and `Principal` for logging or personalization. Later we will use the principal to enforce access control (Article 6).
- `ChatHistory.snapshot(30)` provides the recent messages we already store.

This design separates broadcast topics (`/topic/chat`) from bootstrap channels (`/topic/chat.bootstrap`). The client subscribes to both: one for initial data, one for streaming updates.

### Why Use @SendToUser with @SubscribeMapping?

Without `@SendToUser`, the return value would be dispatched to `/topic/chat.bootstrap` and broadcast to all subscribers. That defeats the purpose of a private initialization payload. Combining `@SubscribeMapping` with `@SendToUser` routes the payload through the user-specific queue `/user/queue/bootstrap`, guaranteeing that only the requester receives it.

## Wiring the Client Bootstrap Flow

Update `chat.html` (with SockJS support from Article 4) to subscribe to the new bootstrap destination immediately after connecting:

```javascript
function subscribeAll() {
    const bootstrapSubscription = stompClient.subscribe('/topic/chat.bootstrap', () => {
        // placeholder; the payload is delivered to /user/queue/bootstrap
    });

    stompClient.subscribe('/user/queue/bootstrap', message => {
        const payload = JSON.parse(message.body);
        renderBootstrap(payload);
    });

    stompClient.subscribe('/topic/chat', frame => {
        const payload = JSON.parse(frame.body);
        Array.isArray(payload) ? payload.forEach(appendMessage) : appendMessage(payload);
    });

    // existing subscriptions (typing notifications, error queue)
}

function renderBootstrap(payload) {
    messagesDiv.innerHTML = '';
    payload.recentMessages.forEach(appendMessage);
    renderParticipants(payload.participants);
    applyConfiguration(payload.configuration);
}

function renderParticipants(participants) {
    usersList.innerHTML = participants.map(p => `<li>${p.username}</li>`).join('');
}

function applyConfiguration(config) {
    document.body.dataset.theme = config.theme;
    transportLabel.textContent += ` • Theme: ${config.theme}`;
}
```

Important points:

- The client subscribes to `/topic/chat.bootstrap` purely to trigger the server-side handler. The actual payload arrives on `/user/queue/bootstrap`.
- We populate the message panel and participant list based on the returned data.
- `applyConfiguration` showcases how you might apply user-specific settings (for example, toggling a CSS theme). For the demo we add `data-theme` to `<body>` for future styling.

### Ensuring Idempotency

`@SubscribeMapping` fires whenever the client subscribes. If the client repeatedly subscribes to `/topic/chat.bootstrap`, it will receive multiple payloads. To avoid duplicate processing, you can use the subscription callback to immediately unsubscribe after receiving the first payload:

```javascript
let bootstrapSub;

bootstrapSub = stompClient.subscribe('/topic/chat.bootstrap', () => {
    // trigger only once
    if (bootstrapSub) {
        bootstrapSub.unsubscribe();
        bootstrapSub = null;
    }
});
```

This pattern ensures the server method runs only when absolutely necessary, reducing load.

## Comparing Alternatives to @SubscribeMapping

You might wonder why not simply call a REST endpoint to fetch initialization data. Both approaches are valid. Here are the trade-offs:

- **REST endpoint:** Simple, cacheable, and works even for clients without WebSocket capability. However, it requires an additional HTTP roundtrip and separate authentication logic.
- **`@SubscribeMapping`:** Aligned with the STOMP lifecycle, no extra HTTP request, and integrates naturally with STOMP authentication (the same principal is used). The downside is that it only works after the WebSocket/SockJS connection is established.
- **Application destination request (`@MessageMapping` + `@SendToUser`):** The client could `SEND` a message to `/app/bootstrap` and wait for a reply via `/user/queue/bootstrap`. This pattern resembles RPC. It is flexible but requires the client to implement correlation logic to match responses with requests.

`@SubscribeMapping` is the most declarative option when the data is closely tied to a subscription. For our chat app, it makes sense: the moment you join the chat topic, you receive the context you need.

## Testing the Bootstrap Flow

To verify that the bootstrap payload works as expected, write an integration test using `WebSocketStompClient`. This test subscribes to `/topic/chat.bootstrap`, waits for the `/user/queue/bootstrap` response, and asserts that the payload is populated.

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class BootstrapControllerTest {

    @LocalServerPort
    int port;

    @Autowired
    WebSocketStompClient stompClient;

    @Test
    void receivesBootstrapPayloadOnSubscribe() throws Exception {
        StompSession session = stompClient.connect(String.format("ws://localhost:%d/ws", port), new StompSessionHandlerAdapter() {}).get(1, TimeUnit.SECONDS);

        CompletableFuture<ChatBootstrapPayload> future = new CompletableFuture<>();

        session.subscribe("/user/queue/bootstrap", new StompFrameHandler() {
            @Override
            public Type getPayloadType(StompHeaders headers) {
                return ChatBootstrapPayload.class;
            }

            @Override
            public void handleFrame(StompHeaders headers, Object payload) {
                future.complete((ChatBootstrapPayload) payload);
            }
        });

        session.subscribe("/topic/chat.bootstrap", new StompFrameHandler() {
            @Override
            public Type getPayloadType(StompHeaders headers) {
                return Object.class;
            }

            @Override
            public void handleFrame(StompHeaders headers, Object payload) {
                // no-op
            }
        });

        ChatBootstrapPayload payload = future.get(2, TimeUnit.SECONDS);

        assertThat(payload.recentMessages()).isNotNull();
        assertThat(payload.participants()).isNotNull();
    }
}
```

This test ensures the bootstrap logic is wired correctly and that the user-specific queue receives the response.

## Enhancing the User Experience

With the bootstrap payload in place, the client can render useful information immediately upon connection. Consider these enhancements to polish the experience further:

- **Skeleton screens:** Show placeholders while waiting for the bootstrap payload. When the payload arrives, replace them with actual content.
- **Latency measurement:** Record the timestamp when the client subscribes and compare it to when the payload arrives. Display the “bootstrap latency” in the UI to monitor startup performance.
- **Progressive hydration:** If the bootstrap payload is large, split it into smaller parts (for example, send configuration first, then history) so the UI can become partially interactive sooner.

## Handling Large Payloads and Pagination

`@SubscribeMapping` responses travel through the same STOMP channel as regular messages. Keep payload sizes manageable to avoid blocking the connection. If you need to send thousands of records, consider one of these strategies:

- Paginate the history: include a token in the payload that clients can use to request additional pages via `@MessageMapping`.
- Compress data: send compact JSON structures or encode data as binary (Spring supports binary STOMP frames).
- Delay heavy data: send essential configuration first, then stream the rest once the UI is ready.

Our toy project keeps things light, but be mindful when adapting these patterns to production.

## Troubleshooting @SubscribeMapping

Common issues and remedies:

- **Handler never invoked:** Ensure the client subscribes to the exact destination (`/topic/chat.bootstrap`) *after* the STOMP connection is established. Subscriptions attempted before `client.activate()` completes will be ignored.
- **Payload broadcast to everyone:** Verify you combined `@SubscribeMapping` with `@SendToUser`. Without `@SendToUser`, the payload is routed to the broker destination and every subscriber receives it.
- **`Principal` is null:** In unauthenticated setups, `Principal` may be absent. Later in Article 6 we will integrate Spring Security so that the principal contains the authenticated username.
- **Serialization errors:** The bootstrap payload uses nested records. Ensure Jackson can handle them (Spring Boot 3 does by default). If you encounter errors, annotate the record with `@JsonIgnoreProperties(ignoreUnknown = true)` or ensure getters follow standard naming conventions.

## Summary and Next Steps

`@SubscribeMapping` is a powerful tool for delivering initial data to clients seamlessly within the STOMP lifecycle. By implementing a bootstrap payload, we improved the chat experience dramatically: the message history appears instantly, the list of participants is accurate from the start, and we laid the groundwork for personalized configuration.

In the next article we pivot toward securing our STOMP endpoints. We will integrate Spring Security, authenticate users, and associate principals with WebSocket sessions so that private messaging (Articles 7 and 8) becomes possible. Before moving on, make sure you can:

- Subscribe to `/topic/chat.bootstrap` from the client and receive the `/user/queue/bootstrap` payload.
- Render history and participant lists immediately on connection.
- Confirm via logs or tests that `@SubscribeMapping` triggers precisely when clients subscribe.

Consider experimenting with additional bootstrap scenarios:

- Send application feature flags or dynamic configuration so the client can enable/disable UI elements.
- Fetch external data (for example, call an HTTP API inside the `@SubscribeMapping` method) and measure the impact on startup latency.
- Cache the bootstrap payload per user to avoid recomputing expensive data for frequent reconnections.

Mastering `@SubscribeMapping` sets the stage for more sophisticated real-time experiences. With instant context in place, we are ready to tackle authentication and user identity in the next installment.

