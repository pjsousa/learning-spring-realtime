---
title: "High-Frequency State Sync: Implementing a Live Collaborative Mouse Cursor"
description: "Create a Spring Boot + STOMP service and plain HTML client that delivers low-latency collaborative cursor updates with session-scoped transient state."
series: "Implementing STOMP Toy Projects with Spring Boot and Plain HTML"
part: 8
---

# High-Frequency State Sync: Implementing a Live Collaborative Mouse Cursor

Real-time cursors give collaborative tools a tangible sense of presence, but they come with an unusual requirement: messages must flow continuously, and outdated positions instantly feel wrong. Unlike chat or document updates, cursor events are transient, high-frequency signals that must roundtrip quickly and never pile up. In this eighth article of our STOMP + Spring Boot toy project series, we combine two-way updates, presence tracking, and transient session state to build a fluid, low-latency cursor overlay.

By the end you will:

- Capture mouse movement in the browser, normalize the coordinates, and emit compact STOMP messages.
- Persist only the freshest cursor state per session using `StompHeaderAccessor` and WebSocket session attributes.
- Broadcast updates to all participants with minimal lag, including graceful teardown when peers disconnect.
- Throttle events on both client and server to prevent flooding while maintaining a responsive feel.

We continue to keep each article runnable on its own. Open a terminal, create a fresh project directory, and follow along—even if you have not read the earlier installments.

---

## 1. Why Cursor Sync Feels Different

Collaborative cursors amplify the sense of “live” collaboration. You can see where teammates’ attention rests, follow their clicks, and coordinate without audio. The catch: cursor updates are ephemeral. A message that arrives half a second late is worthless. The server must hold transient state—just enough to know each user’s most recent position—and rebroadcast immediately. There is no database to sync, no replay log to ship.

This project introduces three concepts beyond our earlier chat rooms and whiteboards:

1. **Normalized coordinates** — We send percentages, not pixels, so the same payload works across viewports.
2. **Session-scoped state** — WebSocket session attributes let us attach metadata (like “last update timestamp”) to each connection.
3. **Throttle loops** — Combining browser `requestAnimationFrame` with a server-side interceptor creates a soft ceiling of ~60 updates per second per client.

---

## 2. Project Setup

Generate a Spring Boot project using Spring Initializr. You only need the WebSocket starter; everything else is provided out of the box.

```bash
curl -s https://start.spring.io/starter.tgz \
  -d dependencies=websocket \
  -d name=live-mouse-cursor \
  -d groupId=com.example \
  -d artifactId=live-mouse-cursor \
  -d javaVersion=17 \
  | tar -xzvf -
cd live-mouse-cursor
```

If you prefer verifying the build file manually, ensure the WebSocket starter is present.

**`pom.xml`**

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-websocket</artifactId>
</dependency>
```

**`build.gradle`**

```groovy
implementation 'org.springframework.boot:spring-boot-starter-websocket'
```

Add a static assets directory for our plain HTML client.

```bash
mkdir -p src/main/resources/static
```

---

## 3. Architecture Overview

We aim for a lightweight pipeline:

1. Each browser opens a SockJS + STOMP connection to `/ws`.
2. Mouse movements trigger messages to the application destination `/app/cursor.update`.
3. The server reads the session id, clamps the coordinates, and stores a `CursorSnapshot` keyed by session id.
4. The server broadcasts the snapshot to `/topic/cursor/positions`.
5. The client subscribes to that topic, updates (or removes) DOM elements representing remote cursors, and remains resolution-independent because it multiplies normalized coordinates by its own viewport dimensions.

Because cursor updates expire instantly, we keep everything in memory—`ConcurrentHashMap` per JVM node—and rely on Spring’s messaging events to clean up when sessions end.

---

## 4. Data Contracts: Minimal and Fast

Create a DTO for inbound cursor updates. We send only the essentials: normalized coordinates `x`, `y`, and a `down` flag that indicates whether the mouse button is currently pressed.

**`src/main/java/com/example/livemousecursor/cursor/CursorUpdate.java`**

```java
package com.example.livemousecursor.cursor;

public class CursorUpdate {
    private double x;
    private double y;
    private boolean down;

    public CursorUpdate() {
    }

    public CursorUpdate(double x, double y, boolean down) {
        this.x = x;
        this.y = y;
        this.down = down;
    }

    public double getX() {
        return x;
    }

    public double getY() {
        return y;
    }

    public boolean isDown() {
        return down;
    }

    public void setX(double x) {
        this.x = x;
    }

    public void setY(double y) {
        this.y = y;
    }

    public void setDown(boolean down) {
        this.down = down;
    }
}
```

For outbound messages, include the session id so clients can map snapshots to DOM nodes.

**`src/main/java/com/example/livemousecursor/cursor/CursorSnapshot.java`**

```java
package com.example.livemousecursor.cursor;

public class CursorSnapshot {
    private String sessionId;
    private double x;
    private double y;
    private boolean down;

    public CursorSnapshot() {
    }

    public CursorSnapshot(String sessionId, double x, double y, boolean down) {
        this.sessionId = sessionId;
        this.x = x;
        this.y = y;
        this.down = down;
    }

    public String getSessionId() {
        return sessionId;
    }

    public double getX() {
        return x;
    }

    public double getY() {
        return y;
    }

    public boolean isDown() {
        return down;
    }

    public void setSessionId(String sessionId) {
        this.sessionId = sessionId;
    }

    public void setX(double x) {
        this.x = x;
    }

    public void setY(double y) {
        this.y = y;
    }

    public void setDown(boolean down) {
        this.down = down;
    }
}
```

Create a small helper to track active cursors.

**`src/main/java/com/example/livemousecursor/cursor/CursorRegistry.java`**

```java
package com.example.livemousecursor.cursor;

import java.util.Collection;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.ConcurrentMap;

import org.springframework.stereotype.Component;

@Component
public class CursorRegistry {

    private final ConcurrentMap<String, CursorSnapshot> cursors = new ConcurrentHashMap<>();

    public void upsert(CursorSnapshot snapshot) {
        cursors.put(snapshot.getSessionId(), snapshot);
    }

    public void remove(String sessionId) {
        cursors.remove(sessionId);
    }

    public Collection<CursorSnapshot> all() {
        return cursors.values();
    }

    public CursorSnapshot get(String sessionId) {
        return cursors.get(sessionId);
    }
}
```

We will also send removal messages when sessions end.

**`src/main/java/com/example/livemousecursor/cursor/CursorRemoval.java`**

```java
package com.example.livemousecursor.cursor;

public class CursorRemoval {
    private String sessionId;

    public CursorRemoval() {
    }

    public CursorRemoval(String sessionId) {
        this.sessionId = sessionId;
    }

    public String getSessionId() {
        return sessionId;
    }

    public void setSessionId(String sessionId) {
        this.sessionId = sessionId;
    }
}
```

---

## 5. WebSocket Configuration

Enable STOMP messaging and simple broker support via a familiar configuration class. At this stage we also prepare the inbound channel for a custom interceptor.

**`src/main/java/com/example/livemousecursor/config/WebSocketConfig.java`**

```java
package com.example.livemousecursor.config;

import org.springframework.context.annotation.Configuration;
import org.springframework.messaging.simp.config.ChannelRegistration;
import org.springframework.messaging.simp.config.MessageBrokerRegistry;
import org.springframework.web.socket.config.annotation.EnableWebSocketMessageBroker;
import org.springframework.web.socket.config.annotation.StompEndpointRegistry;
import org.springframework.web.socket.config.annotation.WebSocketMessageBrokerConfigurer;

@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {

    @Override
    public void configureMessageBroker(MessageBrokerRegistry registry) {
        registry.setApplicationDestinationPrefixes("/app");
        registry.enableSimpleBroker("/topic");
    }

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/ws")
                .setAllowedOriginPatterns("*")
                .withSockJS();
    }

    @Override
    public void configureClientInboundChannel(ChannelRegistration registration) {
        registration.interceptors(new InboundThrottleInterceptor());
    }
}
```

The interceptor protects the server from runaway clients. We will implement it shortly, after the controller.

---

## 6. Processing Cursor Updates with `StompHeaderAccessor`

The controller receives `/app/cursor.update` messages, clamps values, stores them in the registry, and broadcasts to all subscribers. We also stash the last-seen timestamp in the session attributes, which serves as metadata for throttling and cleanup.

**`src/main/java/com/example/livemousecursor/cursor/CursorController.java`**

```java
package com.example.livemousecursor.cursor;

import java.time.Instant;

import org.springframework.messaging.handler.annotation.Header;
import org.springframework.messaging.handler.annotation.MessageMapping;
import org.springframework.messaging.simp.SimpMessagingTemplate;
import org.springframework.messaging.simp.stomp.StompHeaderAccessor;
import org.springframework.stereotype.Controller;

@Controller
public class CursorController {

    private static final String SESSION_CURSOR_KEY = "cursor:lastTimestamp";

    private final CursorRegistry registry;
    private final SimpMessagingTemplate messagingTemplate;

    public CursorController(CursorRegistry registry, SimpMessagingTemplate messagingTemplate) {
        this.registry = registry;
        this.messagingTemplate = messagingTemplate;
    }

    @MessageMapping("/cursor.update")
    public void handleCursorUpdate(CursorUpdate update, StompHeaderAccessor accessor,
                                   @Header("simpSessionId") String sessionId) {

        double x = clamp(update.getX());
        double y = clamp(update.getY());
        boolean down = update.isDown();

        CursorSnapshot snapshot = new CursorSnapshot(sessionId, x, y, down);
        registry.upsert(snapshot);

        accessor.getSessionAttributes().put(SESSION_CURSOR_KEY, Instant.now().toEpochMilli());

        messagingTemplate.convertAndSend("/topic/cursor/positions", snapshot);
    }

    private double clamp(double value) {
        if (value < 0.0) {
            return 0.0;
        }
        if (value > 1.0) {
            return 1.0;
        }
        return value;
    }
}
```

Key callouts:

- `StompHeaderAccessor` offers direct access to the session attribute map.
- `@Header("simpSessionId")` extracts the session id without additional parsing.
- All outbound updates target a scoped topic (`/topic/cursor/positions`) you can reuse in other subscribers.

---

## 7. Removing Ghost Cursors

When a user disconnects, the cursor should disappear immediately. Listen for `SessionDisconnectEvent`, purge the registry entry, and broadcast a removal payload.

**`src/main/java/com/example/livemousecursor/cursor/CursorLifecycleListener.java`**

```java
package com.example.livemousecursor.cursor;

import org.springframework.context.event.EventListener;
import org.springframework.messaging.simp.SimpMessagingTemplate;
import org.springframework.messaging.simp.stomp.StompHeaderAccessor;
import org.springframework.stereotype.Component;
import org.springframework.web.socket.messaging.SessionDisconnectEvent;

@Component
public class CursorLifecycleListener {

    private final CursorRegistry registry;
    private final SimpMessagingTemplate messagingTemplate;

    public CursorLifecycleListener(CursorRegistry registry, SimpMessagingTemplate messagingTemplate) {
        this.registry = registry;
        this.messagingTemplate = messagingTemplate;
    }

    @EventListener
    public void handleDisconnect(SessionDisconnectEvent event) {
        StompHeaderAccessor accessor = StompHeaderAccessor.wrap(event.getMessage());
        String sessionId = accessor.getSessionId();
        if (sessionId != null) {
            registry.remove(sessionId);
            messagingTemplate.convertAndSend("/topic/cursor/positions",
                    new CursorRemoval(sessionId));
        }
    }
}
```

The client will look for messages that only contain a `sessionId` and interpret them as removal commands.

---

## 8. Throttling the Inbound Channel

High-speed mice and touchpads can easily emit hundreds of events per second. Combine your client-side `requestAnimationFrame` batching with server-side protection to keep throughput manageable. The interceptor below enforces a ~60 FPS ceiling.

**`src/main/java/com/example/livemousecursor/config/InboundThrottleInterceptor.java`**

```java
package com.example.livemousecursor.config;

import java.time.Instant;
import java.util.Map;

import org.springframework.messaging.Message;
import org.springframework.messaging.MessageChannel;
import org.springframework.messaging.simp.SimpMessageType;
import org.springframework.messaging.simp.stomp.StompCommand;
import org.springframework.messaging.support.ChannelInterceptor;
import org.springframework.messaging.support.MessageHeaderAccessor;

public class InboundThrottleInterceptor implements ChannelInterceptor {

    private static final long MIN_INTERVAL_MS = 16; // ~60 fps

    @Override
    public Message<?> preSend(Message<?> message, MessageChannel channel) {
        SimpMessageType messageType = (SimpMessageType) message.getHeaders()
                .get(MessageHeaderAccessor.MESSAGE_TYPE_HEADER);

        if (messageType != SimpMessageType.MESSAGE) {
            return message;
        }

        StompCommand command = (StompCommand) message.getHeaders().get("stompCommand");
        if (command != StompCommand.SEND) {
            return message;
        }

        MessageHeaderAccessor accessor = MessageHeaderAccessor.getAccessor(message, MessageHeaderAccessor.class);
        if (accessor == null) {
            return message;
        }

        @SuppressWarnings("unchecked")
        Map<String, Object> sessionAttributes = (Map<String, Object>) accessor.getSessionAttributes();
        if (sessionAttributes == null) {
            return message;
        }

        Object lastTimestamp = sessionAttributes.get("cursor:lastTimestamp");
        long now = Instant.now().toEpochMilli();

        if (lastTimestamp instanceof Long lastSent) {
            if (now - lastSent < MIN_INTERVAL_MS) {
                return null; // drop the frame
            }
        }

        sessionAttributes.put("cursor:lastTimestamp", now);
        return message;
    }
}
```

Returning `null` stops the message before it reaches your controller, effectively rate limiting each connection while keeping the code stateless.

---

## 9. Building the HTML Client

To keep the focus on messaging, we use a single HTML file with embedded JavaScript. It connects via SockJS, subscribes to cursor updates, and draws remote cursors as absolutely positioned circles.

**`src/main/resources/static/index.html`**

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Live Cursor Playground</title>
    <style>
        body {
            font-family: system-ui, sans-serif;
            margin: 0;
            height: 100vh;
            overflow: hidden;
            background: #101820;
            color: #f8f8f8;
            display: flex;
            flex-direction: column;
        }
        header {
            padding: 1rem;
            background: #182a3a;
            box-shadow: 0 2px 8px rgba(0, 0, 0, 0.4);
        }
        main {
            position: relative;
            flex: 1;
        }
        #canvas-layer {
            width: 100%;
            height: 100%;
            cursor: crosshair;
        }
        .cursor {
            position: absolute;
            width: 18px;
            height: 18px;
            margin-top: -9px;
            margin-left: -9px;
            border-radius: 50%;
            border: 2px solid rgba(255, 255, 255, 0.9);
            background: rgba(255, 255, 255, 0.12);
            pointer-events: none;
            transition: transform 0.08s linear;
        }
        .cursor.down {
            background: rgba(52, 211, 153, 0.4);
            border-color: rgba(52, 211, 153, 0.9);
        }
    </style>
</head>
<body>
<header>
    <h1>Live Cursor Playground</h1>
    <p>Move your mouse to see real-time positions. Click and hold to highlight your cursor.</p>
</header>
<main id="canvas-layer"></main>

<script src="https://cdn.jsdelivr.net/npm/sockjs-client@1.6.1/dist/sockjs.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/stompjs@2.3.3/lib/stomp.min.js"></script>
<script>
    const layer = document.getElementById('canvas-layer');

    const cursors = new Map();
    let stompClient = null;

    const state = {
        width: layer.clientWidth,
        height: layer.clientHeight,
        mouse: { x: 0, y: 0, down: false },
        pendingSend: false,
    };

    function resize() {
        state.width = layer.clientWidth;
        state.height = layer.clientHeight;
    }
    window.addEventListener('resize', resize);

    function ensureCursor(sessionId) {
        let node = cursors.get(sessionId);
        if (!node) {
            node = document.createElement('div');
            node.className = 'cursor';
            node.dataset.sessionId = sessionId;
            layer.appendChild(node);
            cursors.set(sessionId, node);
        }
        return node;
    }

    function removeCursor(sessionId) {
        const node = cursors.get(sessionId);
        if (node) {
            node.remove();
            cursors.delete(sessionId);
        }
    }

    function applySnapshot({ sessionId, x, y, down }) {
        const node = ensureCursor(sessionId);
        const left = x * state.width;
        const top = y * state.height;
        node.style.transform = `translate(${left}px, ${top}px)`;
        node.classList.toggle('down', !!down);
    }

    function onMouseMove(event) {
        state.mouse.x = event.clientX / state.width;
        state.mouse.y = event.clientY / state.height;
        scheduleSend();
    }

    function onMouseDown() {
        state.mouse.down = true;
        scheduleSend(true);
    }

    function onMouseUp() {
        state.mouse.down = false;
        scheduleSend(true);
    }

    function scheduleSend(force) {
        if (!stompClient || !stompClient.connected) {
            return;
        }
        if (force) {
            sendCursor();
            return;
        }
        if (state.pendingSend) {
            return;
        }
        state.pendingSend = true;
        requestAnimationFrame(() => {
            state.pendingSend = false;
            sendCursor();
        });
    }

    function sendCursor() {
        stompClient.send('/app/cursor.update', {}, JSON.stringify({
            x: clamp(state.mouse.x),
            y: clamp(state.mouse.y),
            down: state.mouse.down
        }));
    }

    function clamp(value) {
        if (value < 0) return 0;
        if (value > 1) return 1;
        return value;
    }

    function connect() {
        const socket = new SockJS('/ws');
        stompClient = Stomp.over(socket);
        stompClient.reconnect_delay = 5000;
        stompClient.debug = null;

        stompClient.connect({}, frame => {
            console.log('Connected:', frame);
            stompClient.subscribe('/topic/cursor/positions', payload => {
                const body = JSON.parse(payload.body);
                if (body.sessionId && typeof body.x === 'number') {
                    applySnapshot(body);
                } else if (body.sessionId) {
                    removeCursor(body.sessionId);
                }
            });
        }, error => {
            console.error('STOMP error', error);
        });
    }

    layer.addEventListener('mousemove', onMouseMove);
    layer.addEventListener('mousedown', onMouseDown);
    layer.addEventListener('mouseup', onMouseUp);
    layer.addEventListener('mouseleave', onMouseUp);

    resize();
    connect();
</script>
</body>
</html>
```

Implementation notes:

- The client keeps a `Map` of session id to DOM node, enabling quick updates and removals.
- `requestAnimationFrame` smooths out send bursts, aligning with display refresh rates.
- We normalize coordinates at send time so every viewport can rehydrate to pixel space using its own dimensions.

---

## 10. Running the Project Locally

Launch the application using your preferred build tool.

```bash
./mvnw spring-boot:run
```

Open `http://localhost:8080` in two or more browser windows. As you move the mouse in one window, every other window should show a white circle tracking the motion. Clicking and holding your mouse button turns your circle green to signal intent.

If you want to inspect the STOMP frames, temporarily set `stompClient.debug = console.log;` in the HTML client and watch the browser console scroll.

---

## 11. Performance and Scaling Considerations

- **Throttle tuning** — The interceptor’s `MIN_INTERVAL_MS` default of 16 ms targets ~60 FPS. Raising it to 32 ms halves the event frequency (31 FPS) and may be appropriate for larger rooms.
- **Batch broadcasts** — In larger deployments, you can aggregate the entire registry into a single message every 100 ms rather than sending a new snapshot for each move. Clients would render the batch by iterating over the list of cursors.
- **Idle cleanup** — Schedule a `@Scheduled` task that reads the `cursor:lastTimestamp` session attribute and removes entries that have not updated in 10 seconds. This protects against rare cases where disconnect events do not fire.
- **Horizontal scaling** — Because the registry is in-memory, each node manages only its connected sessions. If you run multiple instances, ensure the front-end uses sticky sessions or front them with a STOMP broker relay (Article 9’s focus).

---

## 12. Where to Go Next

You now have a fluid collaborative cursor overlay that sits on top of any shared workspace. Enhancements to try:

- Attach user initials to each cursor, using a `/app/presence.identify` endpoint from earlier articles.
- Highlight selections or drag rectangles by extending the DTO with `selection` coordinates.
- Debounce reconnection events and display fading animations when cursors disappear.

In the next article, we step into a bigger arena: scaling out with broker relays and horizontally distributed nodes. We will examine how our transient state strategy adapts when messages hop across infrastructure boundaries.

Happy hacking—and enjoy watching the cursors dance across the screen in perfect sync.
