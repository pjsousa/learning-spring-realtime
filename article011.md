## Deciphering STOMP Messaging: The Vectors of Real-Time Partitioning

Spring’s WebSocket support gives you raw access to socket connections, but it is the STOMP protocol that brings order to the chaos. Instead of juggling byte streams manually, STOMP lets you name logical channels, bind them to users, and react to subscriptions as structured events. This article dissects the six vectors that Spring leverages to partition and route real-time messages—**Topic**, **Queue**, **User**, **Subscription**, **Connection**, and **Session**—and shows you how to observe and harness each one in code.

We will work in the same toy-project spirit as the rest of the series: a single Spring Boot service paired with a plain HTML/JavaScript client. By the end you will:

- Instrument the handshake so every browser tab declares a username and becomes a distinct authenticated `Principal`.
- Observe how topics and queues split your message space into broadcast and point-to-point channels.
- Inspect subscription identifiers and session metadata emitted by Spring’s event system.
- Track the underlying WebSocket connection in tandem with the logical STOMP session.
- Build a diagnostics dashboard that lets you poke each vector by connecting multiple browser tabs.

> **Goal:** demystify how Spring maps a single WebSocket connection into a forest of logical destinations and how you can observe, debug, and leverage each branch.

---

### Project Setup and Dependencies

Start from a clean Spring Boot project (3.3.x or later) with the WebSocket starter. If you’re using Gradle, the `build.gradle` looks like this:

```groovy
plugins {
    id 'java'
    id 'org.springframework.boot' version '3.3.3'
    id 'io.spring.dependency-management' version '1.1.5'
}

group = 'com.acme.stompvectors'
version = '0.0.1-SNAPSHOT'
java {
    toolchain {
        languageVersion = JavaLanguageVersion.of(21)
    }
}

repositories {
    mavenCentral()
}

dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.boot:spring-boot-starter-websocket'
    implementation 'org.springframework.boot:spring-boot-starter-actuator'

    testImplementation 'org.springframework.boot:spring-boot-starter-test'
}

tasks.named('test') {
    useJUnitPlatform()
}
```

Create the project with Spring Initializr (`https://start.spring.io`) or by running:

```bash
spring init --dependencies=web,websocket,actuator --build=gradle stomp-vector-lab
```

Replace the generated `src/main` directory with the code snippets in this article or integrate them into your existing project from earlier articles.

---

### Wiring the STOMP Endpoint

Our toy service exposes a `/ws` endpoint that accepts STOMP over WebSocket or SockJS. In addition to the standard broker configuration, we attach two custom components:

1. A **handshake interceptor** that pulls the username from the query string and copies it into the WebSocket session attributes.
2. A **handshake handler** that crafts a `Principal` for anonymous demos so user destinations keep working.

```java
// src/main/java/com/acme/stompvectors/config/WebSocketConfig.java
package com.acme.stompvectors.config;

import org.springframework.context.annotation.Configuration;
import org.springframework.messaging.simp.config.MessageBrokerRegistry;
import org.springframework.web.socket.config.annotation.EnableWebSocketMessageBroker;
import org.springframework.web.socket.config.annotation.StompEndpointRegistry;
import org.springframework.web.socket.config.annotation.WebSocketMessageBrokerConfigurer;
import org.springframework.web.socket.config.annotation.WebSocketTransportRegistration;

@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {

    private final UsernameHandshakeInterceptor usernameHandshakeInterceptor;
    private final IdentityHandshakeHandler identityHandshakeHandler;
    private final WebSocketConnectionLogger connectionLogger;

    public WebSocketConfig(
            UsernameHandshakeInterceptor usernameHandshakeInterceptor,
            IdentityHandshakeHandler identityHandshakeHandler,
            WebSocketConnectionLogger connectionLogger
    ) {
        this.usernameHandshakeInterceptor = usernameHandshakeInterceptor;
        this.identityHandshakeHandler = identityHandshakeHandler;
        this.connectionLogger = connectionLogger;
    }

    @Override
    public void configureMessageBroker(MessageBrokerRegistry registry) {
        registry.enableSimpleBroker("/topic", "/queue", "/user");
        registry.setApplicationDestinationPrefixes("/app");
        registry.setUserDestinationPrefix("/user");
    }

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/ws")
                .addInterceptors(usernameHandshakeInterceptor)
                .setHandshakeHandler(identityHandshakeHandler)
                .setAllowedOriginPatterns("*");

        registry.addEndpoint("/ws")
                .addInterceptors(usernameHandshakeInterceptor)
                .setHandshakeHandler(identityHandshakeHandler)
                .setAllowedOriginPatterns("*")
                .withSockJS();
    }

    @Override
    public void configureWebSocketTransport(WebSocketTransportRegistration registry) {
        registry.addDecoratorFactory(connectionLogger);
    }
}
```

The custom components live alongside the configuration:

```java
// src/main/java/com/acme/stompvectors/config/UsernameHandshakeInterceptor.java
package com.acme.stompvectors.config;

import jakarta.servlet.http.HttpServletRequest;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.http.server.ServerHttpRequest;
import org.springframework.http.server.ServerHttpResponse;
import org.springframework.stereotype.Component;
import org.springframework.util.StringUtils;
import org.springframework.web.socket.WebSocketHandler;
import org.springframework.web.socket.server.HandshakeInterceptor;

import java.util.UUID;
import java.util.Map;

@Component
public class UsernameHandshakeInterceptor implements HandshakeInterceptor {

    private static final Logger log = LoggerFactory.getLogger(UsernameHandshakeInterceptor.class);

    @Override
    public boolean beforeHandshake(ServerHttpRequest request, ServerHttpResponse response,
                                   WebSocketHandler wsHandler, Map<String, Object> attributes) {
        if (request instanceof HttpServletRequest servletRequest) {
            String username = servletRequest.getParameter("username");
            if (!StringUtils.hasText(username)) {
                username = "guest-" + System.currentTimeMillis();
            }
            String connectionId = UUID.randomUUID().toString();
            attributes.put("username", username);
            attributes.put("connectionId", connectionId);
            log.info("Handshake starting for username '{}', connectionId={}, remote {}",
                    username, connectionId, servletRequest.getRemoteAddr());
        }
        return true;
    }

    @Override
    public void afterHandshake(ServerHttpRequest request, ServerHttpResponse response,
                               WebSocketHandler wsHandler, Exception exception) {
        // no-op
    }
}
```

```java
// src/main/java/com/acme/stompvectors/config/IdentityHandshakeHandler.java
package com.acme.stompvectors.config;

import org.springframework.stereotype.Component;
import org.springframework.web.socket.WebSocketSession;
import org.springframework.web.socket.server.support.DefaultHandshakeHandler;

import java.security.Principal;
import java.util.Map;
import java.util.UUID;

@Component
public class IdentityHandshakeHandler extends DefaultHandshakeHandler {

    @Override
    protected Principal determineUser(WebSocketSession session, Map<String, Object> attributes) {
        String username = (String) attributes.getOrDefault("username", "guest-" + UUID.randomUUID());
        return () -> username;
    }
}
```

We inject a simple `ConnectionCatalog` (introduced later) so the decorator can persist a mapping between physical sockets and logical STOMP sessions.

```java
// src/main/java/com/acme/stompvectors/config/WebSocketConnectionLogger.java
package com.acme.stompvectors.config;

import com.acme.stompvectors.monitor.ConnectionCatalog;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Component;
import org.springframework.web.socket.WebSocketHandler;
import org.springframework.web.socket.WebSocketSession;
import org.springframework.web.socket.handler.WebSocketHandlerDecorator;
import org.springframework.web.socket.handler.WebSocketHandlerDecoratorFactory;

@Component
public class WebSocketConnectionLogger implements WebSocketHandlerDecoratorFactory {

    private static final Logger log = LoggerFactory.getLogger(WebSocketConnectionLogger.class);
    private final ConnectionCatalog connectionCatalog;

    public WebSocketConnectionLogger(ConnectionCatalog connectionCatalog) {
        this.connectionCatalog = connectionCatalog;
    }

    @Override
    public WebSocketHandler decorate(WebSocketHandler handler) {
        return new WebSocketHandlerDecorator(handler) {
            @Override
            public void afterConnectionEstablished(WebSocketSession session) throws Exception {
                String connectionId = (String) session.getAttributes().get("connectionId");
                if (connectionId == null) {
                    connectionId = session.getId();
                    session.getAttributes().put("connectionId", connectionId);
                }
                connectionCatalog.register(connectionId, session.getId());
                log.info("WebSocket connection {} established (connectionId={}, principal={}, remote={})",
                        session.getId(), connectionId, session.getPrincipal(), session.getRemoteAddress());
                super.afterConnectionEstablished(session);
            }

            @Override
            public void afterConnectionClosed(WebSocketSession session, org.springframework.web.socket.CloseStatus closeStatus) throws Exception {
                String connectionId = (String) session.getAttributes().get("connectionId");
                if (connectionId != null) {
                    connectionCatalog.unregister(connectionId);
                }
                log.info("WebSocket connection {} closed with status {}", session.getId(), closeStatus);
                super.afterConnectionClosed(session, closeStatus);
            }
        };
    }
}
```

These hooks surface the **Connection** vector (the physical WebSocket) and help tie it to the logical **User** and **Session** vectors.

---

### Modeling the Message Payloads

To keep the snippets concise, we use a single DTO that records who sent what, where it should go, and optional metadata:

```java
// src/main/java/com/acme/stompvectors/model/DispatchRequest.java
package com.acme.stompvectors.model;

public record DispatchRequest(
        String message,
        String targetUsername,
        String subscriptionId
) { }
```

We also define the envelope that the server will broadcast back to clients. This isn’t mandatory—STOMP happily delivers plain strings—but including the metadata makes the UI easier to debug.

```java
// src/main/java/com/acme/stompvectors/model/DispatchEnvelope.java
package com.acme.stompvectors.model;

import java.time.Instant;

public record DispatchEnvelope(
        String channelType,
        String destination,
        String payload,
        String fromUser,
        String simpSessionId,
        String connectionId,
        String subscriptionId,
        Instant timestamp
) { }
```

---

### Routing Controller: Exercising Each Vector

`PartitioningController` exposes STOMP destinations under `/app/**` (thanks to our application prefix). Each handler demonstrates a particular vector:

- `/app/topic.broadcast` forwards to `/topic/announcements` (classic publish/subscribe).
- `/app/queue.dispatch` forwards to `/queue/work-items` (point-to-point semantics).
- `/app/user.direct` hits `/user/{username}/queue/inbox` (user-scoped routing).
- `/app/session.echo` uses the caller’s session and subscription id to bounce a message back.

```java
// src/main/java/com/acme/stompvectors/web/PartitioningController.java
package com.acme.stompvectors.web;

import com.acme.stompvectors.model.DispatchEnvelope;
import com.acme.stompvectors.model.DispatchRequest;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.messaging.handler.annotation.MessageMapping;
import org.springframework.messaging.handler.annotation.Payload;
import org.springframework.messaging.simp.SimpMessageHeaderAccessor;
import org.springframework.messaging.simp.SimpMessagingTemplate;
import org.springframework.stereotype.Controller;

import java.security.Principal;
import java.time.Instant;
import java.util.Optional;

@Controller
public class PartitioningController {

    private static final Logger log = LoggerFactory.getLogger(PartitioningController.class);
    private final SimpMessagingTemplate messagingTemplate;

    public PartitioningController(SimpMessagingTemplate messagingTemplate) {
        this.messagingTemplate = messagingTemplate;
    }

    @MessageMapping("/topic.broadcast")
    public void broadcastToTopic(@Payload DispatchRequest request,
                                 Principal principal,
                                 SimpMessageHeaderAccessor headers) {
        var envelope = buildEnvelope("topic", "/topic/announcements", request, principal, headers);
        log.info("Broadcasting to topic: {}", envelope);
        messagingTemplate.convertAndSend("/topic/announcements", envelope);
    }

    @MessageMapping("/queue.dispatch")
    public void dispatchToQueue(@Payload DispatchRequest request,
                                Principal principal,
                                SimpMessageHeaderAccessor headers) {
        var envelope = buildEnvelope("queue", "/queue/work-items", request, principal, headers);
        log.info("Dispatching to queue: {}", envelope);
        messagingTemplate.convertAndSend("/queue/work-items", envelope);
    }

    @MessageMapping("/user.direct")
    public void directToUser(@Payload DispatchRequest request,
                              Principal principal,
                              SimpMessageHeaderAccessor headers) {
        String destination = "/queue/inbox";
        var envelope = buildEnvelope("user", "/user/" + request.targetUsername() + destination, request, principal, headers);
        log.info("Routing to user {}: {}", request.targetUsername(), envelope);
        messagingTemplate.convertAndSendToUser(
                request.targetUsername(),
                destination,
                envelope
        );
    }

    @MessageMapping("/session.echo")
    public void echoToSubscription(@Payload DispatchRequest request,
                                   Principal principal,
                                   SimpMessageHeaderAccessor headers) {
        String subscriptionId = request.subscriptionId();
        String destination = Optional.ofNullable(headers.getDestination())
                .orElseGet(() -> "/user/" + principal.getName() + "/queue/session");
        var envelope = new DispatchEnvelope(
                "subscription",
                destination,
                request.message(),
                principal.getName(),
                headers.getSessionId(),
                headers.getSessionAttributes() == null ? null : (String) headers.getSessionAttributes().get("connectionId"),
                subscriptionId,
                Instant.now()
        );
        log.info("Echoing back to subscription {} on destination {}", subscriptionId, destination);
        messagingTemplate.convertAndSend(destination, envelope);
    }

    private DispatchEnvelope buildEnvelope(String channelType,
                                           String destination,
                                           DispatchRequest request,
                                           Principal principal,
                                           SimpMessageHeaderAccessor headers) {
        String sessionId = headers.getSessionId();
        String connectionId = headers.getSessionAttributes() == null ? null :
                (String) headers.getSessionAttributes().get("connectionId");
        return new DispatchEnvelope(
                channelType,
                destination,
                request.message(),
                principal.getName(),
                sessionId,
                connectionId,
                request.subscriptionId(),
                Instant.now()
        );
    }
}
```

Notice the final method: STOMP automatically copies the client’s `subscription` header into `SimpMessageHeaderAccessor#getSubscriptionId()` for incoming messages, allowing you to make subscription-aware responses.

---

### Tracking Sessions, Subscriptions, and Users

Spring publishes rich application events during the life-cycle of a STOMP connection. Listening to these events exposes the Session and Subscription vectors explicitly. The listener below keeps a simple registry you can inspect via log statements or actuator endpoints.

```java
// src/main/java/com/acme/stompvectors/monitor/StompEventListener.java
package com.acme.stompvectors.monitor;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.context.event.EventListener;
import org.springframework.messaging.simp.SimpMessageHeaderAccessor;
import org.springframework.messaging.simp.user.SimpUserRegistry;
import org.springframework.messaging.simp.stomp.StompHeaderAccessor;
import org.springframework.stereotype.Component;
import org.springframework.web.socket.messaging.SessionConnectEvent;
import org.springframework.web.socket.messaging.SessionConnectedEvent;
import org.springframework.web.socket.messaging.SessionDisconnectEvent;
import org.springframework.web.socket.messaging.SessionSubscribeEvent;
import org.springframework.web.socket.messaging.SessionUnsubscribeEvent;

@Component
public class StompEventListener {

    private static final Logger log = LoggerFactory.getLogger(StompEventListener.class);
    private final SimpUserRegistry userRegistry;

    public StompEventListener(SimpUserRegistry userRegistry) {
        this.userRegistry = userRegistry;
    }

    @EventListener
    public void handleSessionConnect(SessionConnectEvent event) {
        StompHeaderAccessor accessor = StompHeaderAccessor.wrap(event.getMessage());
        log.info("Session connect: session={}, user={}, headers={}",
                accessor.getSessionId(), accessor.getUser(), accessor.toNativeHeaderMap());
    }

    @EventListener
    public void handleSessionConnected(SessionConnectedEvent event) {
        StompHeaderAccessor accessor = StompHeaderAccessor.wrap(event.getMessage());
        log.info("Session connected: session={}, simpUser={}, subscriptionsActive={}",
                accessor.getSessionId(),
                accessor.getUser(),
                userRegistry.findSubscriptions(s -> true).size());
    }

    @EventListener
    public void handleSessionSubscribe(SessionSubscribeEvent event) {
        SimpMessageHeaderAccessor accessor = SimpMessageHeaderAccessor.wrap(event.getMessage());
        log.info("Session subscribe: session={}, subscription={}, destination={}",
                accessor.getSessionId(), accessor.getSubscriptionId(), accessor.getDestination());
    }

    @EventListener
    public void handleSessionUnsubscribe(SessionUnsubscribeEvent event) {
        SimpMessageHeaderAccessor accessor = SimpMessageHeaderAccessor.wrap(event.getMessage());
        log.info("Session unsubscribe: session={}, subscription={}",
                accessor.getSessionId(), accessor.getSubscriptionId());
    }

    @EventListener
    public void handleSessionDisconnect(SessionDisconnectEvent event) {
        StompHeaderAccessor accessor = StompHeaderAccessor.wrap(event.getMessage());
        log.info("Session disconnect: session={}, user={}, closeStatus={}",
                accessor.getSessionId(), accessor.getUser(), accessor.getCloseStatus());
    }
}
```

Opening two tabs with different usernames immediately demonstrates how a single authenticated **User** can hold multiple **Sessions** (one per tab), each potentially carrying multiple **Subscriptions**. The `SimpUserRegistry` also gives you programmatic access to the live registry for dashboards or admin endpoints.

---

### Persisting Connection Metadata

Although STOMP shields you from TCP-level state, Spring exposes the underlying connection through `WebSocketSession`. We already log connection events, but you can also store the relationship between connection and session for ongoing diagnostics.

```java
// src/main/java/com/acme/stompvectors/monitor/ConnectionCatalog.java
package com.acme.stompvectors.monitor;

import org.springframework.stereotype.Component;

import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

@Component
public class ConnectionCatalog {

    private final Map<String, String> connectionToSession = new ConcurrentHashMap<>();

    public void register(String connectionId, String sessionId) {
        connectionToSession.put(connectionId, sessionId);
    }

    public void unregister(String connectionId) {
        connectionToSession.remove(connectionId);
    }

    public Map<String, String> snapshot() {
        return Map.copyOf(connectionToSession);
    }
}
```

The decorator can hook into this catalog:

```java
// src/main/java/com/acme/stompvectors/config/WebSocketConnectionLogger.java
// ... inside afterConnectionEstablished
String connectionId = (String) session.getAttributes().get("connectionId");
connectionCatalog.register(connectionId, session.getId());

// ... inside afterConnectionClosed
connectionCatalog.unregister(connectionId);
```

With this tiny registry in place you can answer questions like “Which physical socket is carrying this principal’s subscription?” The **Connection** vector maps to the WebSocket session ID, whereas the **Session** vector refers to `simpSessionId` inside Spring’s messaging layer; tracking both makes the distinction concrete.

---

### HTML Client Playground

Save the following file under `src/main/resources/static/partitioning.html`. It connects to the broker, lets you subscribe to topics or queues, and sends messages along each vector. Open several tabs with different usernames to see the routing differences in real time.

```html
<!-- src/main/resources/static/partitioning.html -->
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>STOMP Partitioning Lab</title>
    <style>
        body { font-family: system-ui, sans-serif; margin: 2rem; }
        fieldset { margin-bottom: 1.5rem; }
        label { display: block; margin-bottom: 0.3rem; }
        textarea { width: 100%; height: 120px; }
        code { background: #f6f8fa; padding: 0.2rem 0.4rem; border-radius: 4px; }
        .log { font-family: ui-monospace, SFMono-Regular, SFMono-Regular, Menlo, monospace; background: #111; color: #0f0; padding: 1rem; height: 260px; overflow-y: auto; }
        .inline { display: flex; gap: 1rem; align-items: center; }
    </style>
</head>
<body>
<h1>STOMP Partitioning Playground</h1>
<p>
    Open multiple tabs and connect with different usernames to explore how topics, queues, users, and subscriptions isolate traffic.
    Each incoming message appears in the log with metadata extracted from the server’s <code>DispatchEnvelope</code>.
</p>

<fieldset>
    <legend>Connection</legend>
    <div class="inline">
        <label>WebSocket URL:
            <input id="ws-url" value="ws://localhost:8080/ws?username=alice" size="40"/>
        </label>
        <button id="connect-btn">Connect</button>
        <button id="disconnect-btn" disabled>Disconnect</button>
    </div>
    <div>Socket state: <span id="socket-state">DISCONNECTED</span></div>
</fieldset>

<fieldset>
    <legend>Subscriptions</legend>
    <div class="inline">
        <label>Destination:
            <input id="subscription-destination" value="/topic/announcements" size="32"/>
        </label>
        <label>Subscription ID:
            <input id="subscription-id" value="sub-1" size="10"/>
        </label>
        <button id="subscribe-btn" disabled>Subscribe</button>
        <button id="unsubscribe-btn" disabled>Unsubscribe</button>
    </div>
</fieldset>

<fieldset>
    <legend>Send Message</legend>
    <label>Message text:
        <input id="message-text" value="Hello from the playground! 🎯" size="60"/>
    </label>
    <div class="inline">
        <button data-destination="/app/topic.broadcast">Send to Topic</button>
        <button data-destination="/app/queue.dispatch">Send to Queue</button>
        <button data-destination="/app/user.direct">Send to User</button>
        <button data-destination="/app/session.echo">Echo to Subscription</button>
    </div>
    <label>Target username (for /app/user.direct):
        <input id="target-username" value="bob" size="20"/>
    </label>
</fieldset>

<fieldset>
    <legend>Message Log</legend>
    <div id="log" class="log"></div>
</fieldset>

<script src="https://cdn.jsdelivr.net/npm/sockjs-client@1.6.1/dist/sockjs.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/@stomp/stompjs@7.0.0/bundles/stomp.umd.min.js"></script>
<script>
    const logEl = document.getElementById('log');
    let client;
    let connected = false;

    function appendLog(message) {
        const now = new Date().toISOString();
        logEl.innerText += `[${now}] ${message}\n`;
        logEl.scrollTop = logEl.scrollHeight;
    }

    document.getElementById('connect-btn').addEventListener('click', () => {
        const url = document.getElementById('ws-url').value;
        client = new StompJs.Client({
            webSocketFactory: () => new SockJS(url),
            reconnectDelay: 5000,
            heartbeatIncoming: 10000,
            heartbeatOutgoing: 10000,
            debug: (msg) => appendLog(msg)
        });

        client.onConnect = (frame) => {
            connected = true;
            document.getElementById('socket-state').innerText = 'CONNECTED';
            document.getElementById('connect-btn').disabled = true;
            document.getElementById('disconnect-btn').disabled = false;
            document.getElementById('subscribe-btn').disabled = false;
            appendLog(`Connected with session ${frame.headers['session']}`);
        };

        client.onDisconnect = () => {
            connected = false;
            document.getElementById('socket-state').innerText = 'DISCONNECTED';
            document.getElementById('connect-btn').disabled = false;
            document.getElementById('disconnect-btn').disabled = true;
            document.getElementById('subscribe-btn').disabled = true;
            document.getElementById('unsubscribe-btn').disabled = true;
            appendLog('Socket disconnected');
        };

        client.onStompError = (frame) => {
            appendLog(`Broker error: ${frame.headers['message']} - ${frame.body}`);
        };

        client.activate();
    });

    document.getElementById('disconnect-btn').addEventListener('click', () => {
        if (client && connected) {
            client.deactivate();
        }
    });

    document.getElementById('subscribe-btn').addEventListener('click', () => {
        if (!client || !connected) {
            alert('Connect first!');
            return;
        }
        const destination = document.getElementById('subscription-destination').value;
        const subscriptionId = document.getElementById('subscription-id').value;
        client.subscribe(destination, (message) => {
            appendLog(`Received from ${destination}: ${message.body}`);
        }, { id: subscriptionId });
        document.getElementById('unsubscribe-btn').disabled = false;
        appendLog(`Subscribed to ${destination} with id ${subscriptionId}`);
    });

    document.getElementById('unsubscribe-btn').addEventListener('click', () => {
        const subscriptionId = document.getElementById('subscription-id').value;
        if (client && client.active) {
            client.unsubscribe(subscriptionId);
            appendLog(`Unsubscribed ${subscriptionId}`);
        }
    });

    document.querySelectorAll('[data-destination]').forEach(button => {
        button.addEventListener('click', () => {
            if (!client || !connected) {
                alert('Connect first!');
                return;
            }
            const destination = button.dataset.destination;
            const message = document.getElementById('message-text').value;
            const target = document.getElementById('target-username').value;
            client.publish({
                destination,
                body: JSON.stringify({
                    message,
                    targetUsername: target,
                    subscriptionId: document.getElementById('subscription-id').value
                })
            });
            appendLog(`Sent to ${destination}`);
        });
    });
</script>
</body>
</html>
```

Key observations:

- Subscribing multiple tabs to `/topic/announcements` reveals broadcast behaviour: every subscription receives the message, each with its own `subscriptionId`.
- Subscribing one tab to `/queue/work-items` shows point-to-point delivery. Because we’re using the simple broker, queue semantics round-robin messages between subscriptions.
- Publishing to `/app/user.direct` with `targetUsername=bob` only publishes to Bob’s `/user/queue/inbox`, regardless of how many sessions (tabs) Bob has open. Each session receives the message once.
- Hitting `/app/session.echo` with different subscription IDs demonstrates how a single client can multiplex distinct logical channels over the same session.

---

### Understanding the Six Vectors in Context

With the playground running, we can ground the abstract terminology in actual runtime behaviour.

**1. Destination (Topic/Queue)**  
Destinations are the high-level address space. Prefixes communicate intent:

- `/topic/**` → publish/subscribe. The broker delivers each message to every active subscription.
- `/queue/**` → point-to-point. A message is consumed by one subscription; others wait for their turn.

Switch the client’s subscription between `/topic/announcements` and `/queue/work-items` to see how the same message payload either fans out or load-balances.

**2. Subscription**  
Every `SUBSCRIBE` frame carries an `id` header, unique within the connection. Our HTML form lets you assign `sub-1`, `sub-2`, etc. When a `MESSAGE` frame returns, Spring includes this id in the header so the client knows which callback to trigger. In the server log, `SessionSubscribeEvent` prints both the destination and subscription id, letting you trace which callback maps to which channel.

**3. User**  
`Principal` ties multiple sessions back to the same authenticated identity. Because we fabricate the principal during the handshake, `/user/queue/**` destinations “follow” the username even if they reconnect from another browser tab or device. Test this by connecting as `alice` in two tabs: send a direct message from a `bob` tab (`targetUsername=alice`) and watch both `alice` tabs receive the payload.

**4. Session**  
STOMP sessions (`simpSessionId`) track logical context on top of the WebSocket connection. They back features like presence, heartbeats, and subscription scoping. When you open two tabs as the same username, the user has multiple sessions; closing one tab fires `SessionDisconnectEvent` without affecting the other. The controller’s log statements include the session id so you can correlate inbound and outbound frames.

**5. Connection**  
This is the physical WebSocket (or SockJS fallback). The decorator logs connection lifecycle events, including the remote IP and the `Principal`. If your infrastructure load balances across several Spring Boot instances, connections terminate at different nodes but sessions roam freely across the cluster via an external broker (Article 10’s topic). Locally, a session rides on exactly one connection.

**6. Subscription (again) & Acknowledgements**  
Subscriptions also control acknowledgement modes (`ack:auto`, `ack:client`) and message flow. Extend the client to pass headers like `{ ack: 'client-individual' }` and handle `client.ack` or `client.nack` to see how back-pressure operates. Spring surfaces these headers in the `SimpMessageHeaderAccessor`, so your `@MessageMapping` methods can adapt behaviour per subscription.

---

### Test Flow: Bringing It All Together

1. Run the service:
   ```bash
   ./gradlew bootRun
   ```
2. Open three browser windows:
   - Window A: `http://localhost:8080/partitioning.html`, connect as `alice`.
   - Window B: same URL, change the connection field to `ws://localhost:8080/ws?username=bob`.
   - Window C: connect as `support`.
3. In each window, subscribe to `/topic/announcements` using distinct subscription ids (`topic-alice`, etc.).
4. Send a topic message (`Send to Topic`). All windows receive a log entry with `channelType = topic`.
5. Subscribe Bob and Support to `/queue/work-items` with ids `queue-bob` and `queue-support`. Send a queue message: watch the log bounce between them as you send multiple messages.
6. From Alice, send a user-direct message targeting `bob`. Only Bob’s window logs it, but both of Bob’s subscriptions (`topic` and `queue`) remain intact.
7. Subscribe Alice to `/user/queue/inbox` with id `inbox-alice`. From Bob’s window, target `alice`: both of Alice’s sessions receive the message, proving that user destinations fan out per session.
8. Press “Echo to Subscription” from Alice. The response includes the `subscriptionId` you provided, letting each tab verify which logical path triggered the echo.
9. Disconnect Bob’s window: the log prints `Session disconnect` and `WebSocket connection closed`. Alice remains unaffected.

These exercises anchor each vector in observed behaviour, not just theory.

---

### Extending the Experiment

Curious where to go next? Try these enhancements:

- **Expose REST diagnostics:** Create a `@RestController` that returns `connectionCatalog.snapshot()` and `userRegistry.getUsers()` so you can drive an admin console overlay.
- **Per-subscription throttling:** Maintain a `Map<subscriptionId, RateLimiter>` to showcase how subscription ids gate throttling or quota enforcement.
- **Session affinity:** Store game lobbies or chat rooms in a `Map<simpSessionId, LobbyId>` and watch Spring clean entries automatically on disconnect.
- **Broker relay metrics:** Combine this article’s instrumentation with Article 10’s broker relay to correlate connection churn with external broker metrics.

---

### Conclusion

The STOMP protocol layers a rich address space on top of plain WebSockets. By naming destinations, scoping them to users, and tagging every subscription and session, Spring lets you build complex real-time features without wrestling with raw sockets.

In this lab you:

- Configured handshake components so each connection yielded a meaningful `Principal`.
- Observed how topics, queues, and user destinations slice the message namespace.
- Captured subscription ids and session metadata through Spring’s event system.
- Built a browser playground that exercises each partitioning vector, exposing the routing logic visually.

Once these vectors click, designing collaboration features becomes far less mysterious. You can describe requirements (“Send to every collaborator watching the board” or “Alert only the author”) purely in terms of destinations, subscriptions, and users, letting Spring and STOMP do the routing heavy lifting. Keep the playground handy—it’s the perfect tool to explain STOMP concepts to teammates or to debug complex production scenarios.

