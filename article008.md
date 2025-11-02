# High-Frequency State Sync: Implementing a Live Collaborative Mouse Cursor

Direct messaging is powerful, but some collaborative scenarios demand even faster feedback. Imagine a design whiteboard where you see colleagues’ cursors move in real time, or a multiplayer game where every player’s position updates dozens of times per second. STOMP and Spring’s messaging infrastructure can handle these high-frequency updates when used thoughtfully.

In this eighth article we will build a live mouse cursor demo that streams pointer positions from each connected client to all others. The challenge is to balance responsiveness with network efficiency: we must throttle updates, manage transient state, and ensure slow clients do not degrade the experience for everyone else. We will also explore WebSocket session storage for per-user state and discuss strategies for handling bursty workloads.

By the end of this article you will:

- Implement a STOMP endpoint for cursor updates using lightweight JSON payloads.
- Store session metadata (color, display name) using `HandshakeInterceptor` and WebSocket session attributes.
- Broadcast updates efficiently with `SimpMessagingTemplate` and targeted topics.
- Apply client-side throttling (debounce) to reduce update volume without sacrificing smoothness.
- Monitor performance and tune broker settings for high-frequency traffic.

Let’s bring our toy project to life with moving cursors.

## Architecture Overview

The live cursor feature involves several components:

1. When a user connects, we assign them a unique color and store it in the WebSocket session.
2. The client captures pointer movements and publishes STOMP messages to `/app/cursor` at a throttled rate (for example, every 50 milliseconds).
3. The server receives updates, enriches them with session metadata (color, display name), and broadcasts to `/topic/cursor`.
4. All clients subscribe to `/topic/cursor` and render markers for each user.

We will reuse the authentication setup from Article 6 so each cursor can be labeled with the authenticated username.

## Enriching Session Metadata with a Handshake Interceptor

To avoid sending redundant data with every message, we store per-user attributes during the handshake. Create `CursorHandshakeInterceptor`:

```java
package com.example.stompchat.cursor;

import java.security.Principal;
import java.util.Map;
import java.util.concurrent.ThreadLocalRandom;

import org.springframework.http.server.ServerHttpRequest;
import org.springframework.http.server.ServerHttpResponse;
import org.springframework.stereotype.Component;
import org.springframework.web.socket.WebSocketHandler;
import org.springframework.web.socket.server.HandshakeInterceptor;

@Component
public class CursorHandshakeInterceptor implements HandshakeInterceptor {

    @Override
    public boolean beforeHandshake(ServerHttpRequest request, ServerHttpResponse response,
                                   WebSocketHandler wsHandler, Map<String, Object> attributes) {
        Principal principal = request.getPrincipal();
        String username = principal != null ? principal.getName() : "guest";
        attributes.put("cursorUsername", username);
        attributes.put("cursorColor", randomColor());
        return true;
    }

    @Override
    public void afterHandshake(ServerHttpRequest request, ServerHttpResponse response,
                               WebSocketHandler wsHandler, Exception exception) {
        // no-op
    }

    private String randomColor() {
        ThreadLocalRandom random = ThreadLocalRandom.current();
        int r = random.nextInt(50, 220);
        int g = random.nextInt(50, 220);
        int b = random.nextInt(50, 220);
        return String.format("rgb(%d, %d, %d)", r, g, b);
    }
}
```

Register this interceptor on the SockJS endpoint in `WebSocketConfig`:

```java
@Autowired
private CursorHandshakeInterceptor cursorHandshakeInterceptor;

@Override
public void registerStompEndpoints(StompEndpointRegistry registry) {
    registry.addEndpoint("/sockjs")
            .setAllowedOriginPatterns("*")
            .addInterceptors(cursorHandshakeInterceptor)
            .withSockJS();

    registry.addEndpoint("/ws")
            .setAllowedOriginPatterns("*")
            .addInterceptors(cursorHandshakeInterceptor);
}
```

Now every WebSocket session has `cursorUsername` and `cursorColor` attributes accessible via `SimpMessageHeaderAccessor`.

## Defining Cursor Payloads

The client will send minimal payloads: x and y positions as percentages of the viewport. The server will enrich the message with metadata before broadcasting. Define `CursorUpdate` and `CursorBroadcast` records:

```java
package com.example.stompchat.cursor;

public record CursorUpdate(double xPercent, double yPercent) {}

public record CursorBroadcast(String username, String color, double xPercent, double yPercent, long timestamp) {}
```

## Implementing the Cursor Controller

Create `CursorController`:

```java
package com.example.stompchat.cursor;

import java.time.Instant;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.messaging.handler.annotation.MessageMapping;
import org.springframework.messaging.handler.annotation.Payload;
import org.springframework.messaging.handler.annotation.SendTo;
import org.springframework.messaging.simp.SimpMessageHeaderAccessor;
import org.springframework.stereotype.Controller;

@Controller
public class CursorController {

    private static final Logger log = LoggerFactory.getLogger(CursorController.class);

    @MessageMapping("/cursor")
    @SendTo("/topic/cursor")
    public CursorBroadcast broadcast(@Payload CursorUpdate update, SimpMessageHeaderAccessor headers) {
        double x = clamp(update.xPercent(), 0, 100);
        double y = clamp(update.yPercent(), 0, 100);

        String username = (String) headers.getSessionAttributes().getOrDefault("cursorUsername", "guest");
        String color = (String) headers.getSessionAttributes().getOrDefault("cursorColor", "rgba(0,0,0,0.5)");

        CursorBroadcast broadcast = new CursorBroadcast(username, color, x, y, Instant.now().toEpochMilli());
        log.debug("Cursor update from {}: ({}, {})", username, x, y);
        return broadcast;
    }

    private double clamp(double value, double min, double max) {
        return Math.max(min, Math.min(max, value));
    }
}
```

This controller is intentionally lightweight: it clamps values to avoid off-screen positions, retrieves session metadata, and broadcasts to `/topic/cursor`. The simple broker handles fan-out to all subscribers.

## Updating Security Rules

Ensure the messaging security configuration permits authenticated users to send cursor updates:

```java
messages
    .simpDestMatchers("/app/chat", "/app/typing", "/app/direct", "/app/presence", "/app/cursor").hasRole("USER")
    .simpSubscribeDestMatchers("/topic/cursor").permitAll();
```

Depending on your requirements, you might allow read-only access for anonymous users (`permitAll` on subscribe) while restricting publishers to authenticated users.

## Building the Cursor Client

Add a new HTML section to `chat.html` for the collaborative board:

```html
<section id="board" class="card">
    <h2>Live Cursor Board</h2>
    <div id="canvas">
        <div id="my-cursor"></div>
    </div>
    <div id="cursor-hint">Move your mouse over the board to share your position</div>
</section>
```

Add CSS in `chat.css`:

```css
#board { margin-top: 1rem; }
#canvas { position: relative; height: 300px; border: 1px solid #d0d4d9; border-radius: 6px; background: repeating-linear-gradient(45deg, #f7f9fc, #f7f9fc 10px, #eef2f7 10px, #eef2f7 20px); overflow: hidden; }
.cursor { position: absolute; width: 14px; height: 14px; border-radius: 50%; transform: translate(-50%, -50%); pointer-events: none; }
#my-cursor { width: 18px; height: 18px; border: 2px solid #3949ab; background: rgba(57,73,171,0.2); }
.cursor-label { position: absolute; top: 16px; left: 16px; background: rgba(0,0,0,0.6); color: #fff; padding: 2px 6px; border-radius: 4px; font-size: 0.75rem; }
```

Now extend the script to wire up cursor tracking:

```javascript
const canvas = document.getElementById('canvas');
const myCursor = document.getElementById('my-cursor');
const cursors = new Map();

function updateMyCursor(event) {
    const rect = canvas.getBoundingClientRect();
    const x = ((event.clientX - rect.left) / rect.width) * 100;
    const y = ((event.clientY - rect.top) / rect.height) * 100;

    myCursor.style.left = `${x}%`;
    myCursor.style.top = `${y}%`;

    publishCursor(x, y);
}

const publishCursor = throttle((x, y) => {
    if (!stompClient || !stompClient.connected) return;
    stompClient.publish({
        destination: '/app/cursor',
        body: JSON.stringify({ xPercent: x, yPercent: y })
    });
}, 50);

function throttle(fn, delay) {
    let last = 0;
    return (...args) => {
        const now = Date.now();
        if (now - last >= delay) {
            last = now;
            fn(...args);
        }
    };
}

canvas.addEventListener('pointermove', updateMyCursor);

function renderRemoteCursor({ username, color, xPercent, yPercent, timestamp }) {
    const existing = cursors.get(username) || createCursor(username, color);
    existing.element.style.left = `${xPercent}%`;
    existing.element.style.top = `${yPercent}%`;
    existing.label.textContent = username;
    existing.lastSeen = Date.now();
}

function createCursor(username, color) {
    const dot = document.createElement('div');
    dot.className = 'cursor';
    dot.style.background = color;

    const label = document.createElement('div');
    label.className = 'cursor-label';
    label.textContent = username;

    const wrapper = document.createElement('div');
    wrapper.style.position = 'absolute';
    wrapper.append(dot, label);
    canvas.append(wrapper);

    const entry = { element: wrapper, label, lastSeen: Date.now() };
    cursors.set(username, entry);
    return entry;
}

function purgeStaleCursors() {
    const now = Date.now();
    for (const [username, entry] of cursors.entries()) {
        if (now - entry.lastSeen > 5000) {
            entry.element.remove();
            cursors.delete(username);
        }
    }
}

setInterval(purgeStaleCursors, 2000);

function subscribeAll() {
    // existing subscriptions...
    stompClient.subscribe('/topic/cursor', frame => {
        const payload = JSON.parse(frame.body);
        if (payload.username === nicknameInput.value) {
            return; // already rendered as my cursor
        }
        renderRemoteCursor(payload);
    });
}
```

The `throttle` helper ensures we publish at most one update every 50 milliseconds (~20 frames per second). Adjust the interval based on performance needs. We also clean up cursors that have not been updated in five seconds, preventing stale markers from lingering if a user disconnects abruptly.

### Pointer Events and Touch Support

We used the `pointermove` event instead of `mousemove` so touch and pen inputs work automatically. On touch devices, consider throttling more aggressively (e.g., 100 ms) to reduce bandwidth.

## Monitoring Performance

High-frequency updates can saturate the simple broker if not managed carefully. Monitor the following metrics:

- **Application logs:** With `log.debug` enabled, you can observe the rate of incoming cursor updates.
- **Heap usage:** Frequent allocations can stress the garbage collector. Using compact payloads minimizes memory churn.
- **Network traffic:** Throttle on both client and server if necessary. Consider compression if updates become large.

If you notice backpressure warnings (e.g., `Execution of session send failed`), tune the broker settings in `WebSocketConfig`:

```java
@Override
public void configureWebSocketTransport(WebSocketTransportRegistration registration) {
    registration.setSendTimeLimit(15_000)
            .setSendBufferSizeLimit(512 * 1024)
            .setMessageSizeLimit(64 * 1024);
}
```

The defaults are often sufficient, but tuning ensures slow clients do not cause disconnections.

## Handling Burst Control Server-Side

If clients misbehave and send too many updates, you can add a server-side interceptor to drop excessive messages. Extend `ChannelInterceptor` to implement rate limiting based on timestamps stored in `SimpMessageHeaderAccessor`. For example:

```java
public class CursorRateLimiter implements ChannelInterceptor {

    private final Map<String, Long> lastSent = new ConcurrentHashMap<>();

    @Override
    public Message<?> preSend(Message<?> message, MessageChannel channel) {
        StompHeaderAccessor accessor = StompHeaderAccessor.wrap(message);
        if (SimpMessageType.MESSAGE.equals(accessor.getMessageType()) && "/app/cursor".equals(accessor.getDestination())) {
            String sessionId = accessor.getSessionId();
            long now = System.currentTimeMillis();
            long last = lastSent.getOrDefault(sessionId, 0L);
            if (now - last < 30) {
                return null; // drop message
            }
            lastSent.put(sessionId, now);
        }
        return message;
    }
}
```

Register this interceptor on the inbound channel if you need server-side throttling.

## Testing the Live Cursor Experience

Restart the application and open two browser windows. Move the mouse within the board; your cursor appears as a highlighted circle, while other users’ cursors show colored dots with labels. Try opening the page on a mobile device: the pointer events should translate touch movements smoothly. Adjust the throttle interval to strike the right balance between responsiveness and network load.

### Debugging Tips

- Inspect the `/sockjs` network requests. You should see a steady stream of STOMP `SEND` frames with small payloads.
- If cursors freeze, check the browser console for WebSocket errors or throttling warnings.
- Temporarily disable `throttle` to simulate worst-case traffic and observe how the server handles it.

## Advanced Enhancements

Once the basic cursor sharing works, consider:

- **Z-index ordering:** Highlight the cursor belonging to the user currently speaking (if combined with audio chat) by adjusting CSS z-index.
- **Trail effects:** Store recent positions and animate a fading trail for each cursor using `<canvas>` or SVG.
- **Collision detection:** Demonstrate collaborative features like “snap to grid” by validating positions server-side before broadcasting.
- **State partitioning:** For large boards, partition users into “rooms” or use dynamic topics like `/topic/cursor.roomA` to reduce broadcast load.

These enhancements showcase STOMP’s versatility for interactive experiences.

## Summary and Next Steps

We have implemented a high-frequency collaboration feature using Spring Boot and STOMP. By carefully throttling updates, storing session metadata, and tuning the broker, we delivered smooth cursor streaming across clients. This pattern applies to many real-time scenarios: collaborative drawing, syntax highlighters, or sensor dashboards.

In Article 9 we will examine the internal message channels, thread pools, and observability tools that help sustain these demanding workloads. Before moving on, ensure you can:

- Stream cursor updates without noticeable lag among multiple clients.
- Monitor and adjust throttling to keep bandwidth under control.
- Understand how session attributes and interceptors enrich STOMP messages.

With high-frequency state sync mastered, we are ready to dive deeper into performance tuning.

