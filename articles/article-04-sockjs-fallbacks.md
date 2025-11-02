---
title: "Beyond Native WebSocket: Integrating SockJS for Cross-Browser Compatibility"
description: "Ensure your Spring Boot STOMP toy project thrives in legacy browsers and proxy-constrained environments by wrapping endpoints with SockJS fallbacks and shipping a runnable HTML client."
series: "Spring Boot STOMP Toy Projects"
episode: 4
word_count_target: "1500-2000"
---

# Beyond Native WebSocket: Integrating SockJS for Cross-Browser Compatibility

## Overview

WebSocket support is nearly ubiquitous today, yet production teams still encounter users stuck behind restrictive corporate proxies, aging browsers, or mobile networks that veto upgrade handshakes. SockJS extends your STOMP endpoints with a WebSocket-like API that gracefully degrades to alternative transports such as HTTP streaming or long polling. In this fourth entry of the Spring Boot STOMP toy-project series, you will retrofit the chat service from earlier episodes so that it keeps working even when the WebSocket upgrade fails. You will learn how Spring’s `.withSockJS()` flag wires everything together, how to fine-tune transport options, and how to ship an HTML/JavaScript client that relies on SockJS transparently.

## Series Context

This article follows **Article 3: Client-State Synchronization and Message Lifecycles**, where you built a simple STOMP-enabled chat playground with acknowledgement handling. While today’s topic builds directly on that service, the walkthrough is self-contained: you can start here as long as you have a basic Spring Boot project ready for WebSocket messaging.

By the end of this tutorial you will:

- Expose a SockJS-backed STOMP endpoint at `/ws-chat`.
- Deliver a Spring Boot service that still supports native WebSockets when available.
- Build an HTML client with `sockjs-client` + `stomp.js` that connects without browser tweaks.
- Verify fallback behavior with practical testing steps, including proxy simulations.

## Why SockJS Still Matters

Native WebSocket is elegant—one TCP connection, bidirectional frames, and minimal overhead. But real-life deployments contend with three recurring failure modes:

1. **Browser maturity**: Internet Explorer 9 and earlier never shipped WebSocket support. Embedded and kiosk-style environments still surface these old browsers.
2. **Intermediary policy**: Corporate proxies, enterprise firewalls, and some mobile carriers aggressively block the `Upgrade: websocket` handshake.
3. **Connection churn**: Network equipment sometimes terminates long-lived WebSocket connections prematurely, especially on flaky mobile links.

SockJS addresses these issues by offering a **WebSocket-like** interface that negotiates transports in priority order: native WebSocket, HTTP streaming transports (XHR streaming, EventSource), and finally heartbeat-enabled long polling. The client-side API mirrors the native WebSocket constructor (`new SockJS(url)`), so existing STOMP clients continue to function without modification. On the server Spring abstracts the complexity: calling `.withSockJS()` registers additional handshake endpoints, manages session emulation, and negotiates the best transport automatically.

## Project Structure

We will create the following structure (Maven build shown; Gradle instructions mirror it):

```
└── src
    ├── main
    │   ├── java/com/example/chat
    │   │   ├── ChatApplication.java
    │   │   ├── config/WebSocketConfig.java
    │   │   ├── controller/MessageController.java
    │   │   └── model/
    │   │       ├── ChatMessage.java
    │   │       └── ChatNotification.java
    │   └── resources/static
    │       ├── index.html
    │       ├── app.js
    │       └── styles.css (optional)
    └── test/java/com/example/chat/WebSocketIntegrationTest.java
```

`static/` holds the toy client so the entire playground ships in a single Spring Boot jar. Adjust package names to match your project conventions.

## Bootstrapping the Spring Boot Project

If you are starting fresh, generate a Spring Boot 3.3.x project via [start.spring.io](https://start.spring.io) with the following settings:

- **Project**: Maven
- **Language**: Java (17 or newer)
- **Dependencies**: `Spring Web`, `Spring WebSocket`, optional `Spring Boot DevTools`

Your `pom.xml` should include at least:

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-dependencies</artifactId>
            <version>3.3.2</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>

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
        <artifactId>spring-boot-starter-actuator</artifactId>
        <optional>true</optional>
    </dependency>
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <optional>true</optional>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

Run `./mvnw spring-boot:run` once to make sure the base application starts cleanly.

## Configuring STOMP with SockJS

In Article 3 you registered a STOMP endpoint without SockJS support. Update the configuration to add `.withSockJS()` and tune the fallbacks.

`WebSocketConfig.java`:

```java
package com.example.chat.config;

import org.springframework.context.annotation.Configuration;
import org.springframework.messaging.simp.config.MessageBrokerRegistry;
import org.springframework.scheduling.TaskScheduler;
import org.springframework.web.socket.config.annotation.EnableWebSocketMessageBroker;
import org.springframework.web.socket.config.annotation.StompEndpointRegistry;
import org.springframework.web.socket.config.annotation.WebSocketMessageBrokerConfigurer;

@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {

    private final TaskScheduler messageBrokerTaskScheduler;

    public WebSocketConfig(TaskScheduler messageBrokerTaskScheduler) {
        this.messageBrokerTaskScheduler = messageBrokerTaskScheduler;
    }

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry
            .addEndpoint("/ws-chat")
            .setAllowedOriginPatterns("http://localhost:*", "https://*.example.com")
            .withSockJS()
            .setClientLibraryUrl("https://cdn.jsdelivr.net/npm/sockjs-client@1.6.1/dist/sockjs.min.js")
            .setSupressCors(false)
            .setSessionCookieNeeded(false)
            .setHeartbeatTime(25_000)
            .setDisconnectDelay(5_000)
            .setStreamBytesLimit(512 * 1024);
    }

    @Override
    public void configureMessageBroker(MessageBrokerRegistry registry) {
        registry.enableSimpleBroker("/topic", "/queue").setTaskScheduler(messageBrokerTaskScheduler);
        registry.setApplicationDestinationPrefixes("/app");
    }
}
```

**Key takeaways**:

- `addEndpoint("/ws-chat")` defines the handshake entry point. SockJS augments it with auxiliary paths such as `/ws-chat/info` and `/ws-chat/{server-id}/{session-id}/...` for fallback transports.
- `.withSockJS()` wires SockJS-specific infrastructure. `setClientLibraryUrl` instructs browsers where to fetch the client library when they request `/ws-chat/sockjs.min.js`.
- `setSupressCors(false)` allows SockJS XHR fallbacks to operate cross-origin. Note the historic typo in the method name.
- `setSessionCookieNeeded(false)` keeps the connection stateless, valuable for CDN caching or strictly RESTful setups.
- `setHeartbeatTime` and `setDisconnectDelay` tune heartbeats and disconnect grace periods so proxies keep the pseudo-connection alive.

Feel free to narrow the `setAllowedOriginPatterns` entries for production deployments, replacing the wildcard with your actual domain list.

### Advanced SockJS Tuning

Spring exposes additional knobs when you need to take fine-grained control:

```java
registry
    .addEndpoint("/ws-chat")
    .withSockJS()
    .setWebSocketEnabled(true)
    .setSessionCookieNeeded(false)
    .setStreamBytesLimit(1024 * 1024)
    .setDisconnectDelay(10_000);
```

- **`setWebSocketEnabled(false)`** forces fallbacks even for modern browsers, handy when emulating constrained network environments during testing.
- **`setStreamBytesLimit`** raises the allowed bytes per streaming frame; bump this if you send larger payloads over XHR streaming.
- **`setDisconnectDelay`** controls how long the server keeps fallback sessions around waiting for the next poll request. Lower values free resources faster, higher values offer more resilience to temporary network drops.

## Message Models and Controller Logic

Keep chat semantics simple: a `ChatMessage` echoes user text, while a `ChatNotification` highlights system events such as joins.

`ChatMessage.java`:

```java
package com.example.chat.model;

public record ChatMessage(String from, String text) {}
```

`ChatNotification.java`:

```java
package com.example.chat.model;

import java.time.Instant;

public record ChatNotification(String message, Instant timestamp) {}
```

`MessageController.java`:

```java
package com.example.chat.controller;

import com.example.chat.model.ChatMessage;
import com.example.chat.model.ChatNotification;
import java.time.Instant;
import org.springframework.messaging.handler.annotation.MessageMapping;
import org.springframework.messaging.handler.annotation.SendTo;
import org.springframework.messaging.simp.SimpMessagingTemplate;
import org.springframework.stereotype.Controller;

@Controller
public class MessageController {

    private final SimpMessagingTemplate messagingTemplate;

    public MessageController(SimpMessagingTemplate messagingTemplate) {
        this.messagingTemplate = messagingTemplate;
    }

    @MessageMapping("/chat.send")
    @SendTo("/topic/chat")
    public ChatMessage handleChat(ChatMessage incoming) {
        return incoming;
    }

    @MessageMapping("/chat.join")
    public void announceJoin(ChatMessage joiningUser) {
        var notice = new ChatNotification(
            joiningUser.from() + " joined the room",
            Instant.now()
        );
        messagingTemplate.convertAndSend("/topic/system", notice);
    }
}
```

The STOMP broker handles message fan-out across transports—SockJS fallbacks require zero changes to your method signatures or destinations.

## Serving a SockJS-Enabled Client

Place static assets under `src/main/resources/static`. Spring Boot will serve them from `/` automatically.

`index.html`:

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>SockJS Chat Playground</title>
    <link rel="stylesheet" href="styles.css">
</head>
<body>
<div id="app">
    <header>
        <h1>Beyond Native WebSocket</h1>
        <p>Test SockJS fallbacks by opening multiple browsers or throttling the network.</p>
    </header>

    <section id="connection">
        <label>
            Display Name:
            <input type="text" id="username" placeholder="Ada Lovelace">
        </label>
        <button id="connect">Connect</button>
        <button id="disconnect" disabled>Disconnect</button>
        <span id="status" class="status disconnected">Disconnected</span>
        <label class="transport">
            Active transport:
            <span id="transport">—</span>
        </label>
    </section>

    <section id="chat" class="hidden">
        <ul id="system"></ul>
        <ul id="messages"></ul>
        <form id="message-form">
            <input type="text" id="message" placeholder="Say something…" required>
            <button type="submit">Send</button>
        </form>
    </section>
</div>

<script src="https://cdn.jsdelivr.net/npm/sockjs-client@1.6.1/dist/sockjs.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/stompjs@2.3.3/lib/stomp.min.js"></script>
<script src="app.js"></script>
</body>
</html>
```

`app.js`:

```javascript
(() => {
    const connectBtn = document.querySelector('#connect');
    const disconnectBtn = document.querySelector('#disconnect');
    const usernameInput = document.querySelector('#username');
    const statusEl = document.querySelector('#status');
    const transportEl = document.querySelector('#transport');
    const chatSection = document.querySelector('#chat');
    const systemList = document.querySelector('#system');
    const messageList = document.querySelector('#messages');
    const form = document.querySelector('#message-form');
    const messageInput = document.querySelector('#message');

    let stompClient = null;
    let transport = null;

    const setConnected = (connected) => {
        connectBtn.disabled = connected;
        disconnectBtn.disabled = !connected;
        statusEl.textContent = connected ? 'Connected' : 'Disconnected';
        statusEl.className = `status ${connected ? 'connected' : 'disconnected'}`;
        chatSection.classList.toggle('hidden', !connected);
    };

    const connect = () => {
        const name = usernameInput.value.trim();
        if (!name) {
            alert('Pick a display name first.');
            return;
        }

        const socket = new SockJS('/ws-chat');
        transport = () => socket._transport?.transportName || 'pending…';

        stompClient = Stomp.over(socket);
        stompClient.debug = (msg) => console.debug(`[STOMP] ${msg}`);

        stompClient.connect({}, (frame) => {
            void frame;
            setConnected(true);
            transportEl.textContent = transport();
            stompClient.subscribe('/topic/chat', (payload) => {
                const body = JSON.parse(payload.body);
                appendMessage(body.from, body.text);
            });
            stompClient.subscribe('/topic/system', (payload) => {
                const body = JSON.parse(payload.body);
                appendSystem(body.message, body.timestamp);
            });

            stompClient.send('/app/chat.join', {}, JSON.stringify({ from: name }));
        }, (error) => {
            console.error(error);
            setConnected(false);
            transportEl.textContent = 'error';
        });
    };

    const disconnect = () => {
        if (stompClient) {
            stompClient.disconnect(() => {
                setConnected(false);
                transportEl.textContent = '—';
            });
        }
    };

    const appendMessage = (from, text) => {
        const li = document.createElement('li');
        li.innerHTML = `<strong>${from}:</strong> ${text}`;
        messageList.appendChild(li);
    };

    const appendSystem = (message, timestamp) => {
        const li = document.createElement('li');
        li.textContent = `[${new Date(timestamp).toLocaleTimeString()}] ${message}`;
        systemList.appendChild(li);
    };

    form.addEventListener('submit', (event) => {
        event.preventDefault();
        const name = usernameInput.value.trim();
        const text = messageInput.value.trim();
        if (stompClient && text) {
            stompClient.send('/app/chat.send', {}, JSON.stringify({ from: name, text }));
            messageInput.value = '';
            messageInput.focus();
        }
    });

    connectBtn.addEventListener('click', connect);
    disconnectBtn.addEventListener('click', disconnect);
})();
```

Optional `styles.css` (trim to taste):

```css
body {
    font-family: system-ui, -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif;
    margin: 0;
    padding: 2rem;
    background: #f8fafc;
    color: #0f172a;
}

#app {
    max-width: 720px;
    margin: 0 auto;
    background: #fff;
    padding: 2rem;
    border-radius: 16px;
    box-shadow: 0 10px 30px rgba(15, 23, 42, 0.12);
}

.status {
    margin-left: 1rem;
    font-weight: 600;
}

.status.connected {
    color: #16a34a;
}

.status.disconnected {
    color: #dc2626;
}

.hidden {
    display: none;
}

ul {
    list-style: none;
    padding: 0;
}

ul#system {
    color: #334155;
    font-size: 0.9rem;
    margin-bottom: 1rem;
}

form {
    display: flex;
    gap: 0.5rem;
}

input[type="text"] {
    flex: 1;
    padding: 0.75rem;
    border-radius: 12px;
    border: 1px solid #cbd5f5;
}

button {
    padding: 0.75rem 1.25rem;
    border: none;
    border-radius: 12px;
    background: #2563eb;
    color: #fff;
    font-weight: 600;
    cursor: pointer;
}

button:disabled {
    background: #94a3b8;
    cursor: not-allowed;
}
```

The UI shows the negotiated transport (`websocket`, `xhr-streaming`, `xhr-polling`, etc.) so you can confirm fallback behavior at a glance.

## Testing the Fallbacks

Run the service:

```bash
./mvnw spring-boot:run
```

Open `http://localhost:8080/` in two browsers and confirm chat messages flow. Then explore fallback scenarios:

1. **Disable WebSocket in DevTools**: In Chrome, open DevTools → Network → “More tools > Network conditions”. Set a legacy user agent such as “Internet Explorer 9” to push SockJS toward XHR transports.
2. **Throttle the network**: Pick “Slow 3G” in the Network throttling dropdown to stress long polling behavior.
3. **Strip upgrade headers**: Use `mitmproxy`, `Fiddler`, or `Charles` to remove the `Upgrade` header from the handshake. SockJS should renegotiate to HTTP-based transport.
4. **Inspect UI diagnostics**: The “Active transport” label should switch from `websocket` to `xhr-streaming` or `xhr-polling` when the fallback engages.
5. **Check server logs**: Spring logs lines like `TransportHandlingSockJsService : Using "xhr_streaming" transport` when fallbacks fire.
6. **Validate `/ws-chat/info`**: `curl http://localhost:8080/ws-chat/info` returns JSON describing available SockJS transports; ensure it is reachable.

## Integration Testing (Optional)

Adding a WebSocket STOMP integration test keeps regression coverage high:

```java
package com.example.chat;

import static org.assertj.core.api.Assertions.assertThat;

import java.lang.reflect.Type;
import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.BlockingQueue;
import java.util.concurrent.TimeUnit;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.messaging.converter.MappingJackson2MessageConverter;
import org.springframework.messaging.simp.stomp.StompFrameHandler;
import org.springframework.messaging.simp.stomp.StompHeaders;
import org.springframework.messaging.simp.stomp.StompSession;
import org.springframework.messaging.simp.stomp.StompSessionHandlerAdapter;
import org.springframework.util.concurrent.ListenableFuture;
import org.springframework.web.socket.client.standard.StandardWebSocketClient;
import org.springframework.web.socket.messaging.WebSocketStompClient;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class WebSocketIntegrationTest {

    @Value("${local.server.port}")
    private int port;

    @Test
    void chatEndpointDeliversMessages() throws Exception {
        var stompClient = new WebSocketStompClient(new StandardWebSocketClient());
        stompClient.setMessageConverter(new MappingJackson2MessageConverter());

        ListenableFuture<StompSession> future = stompClient.connectAsync(
                "ws://localhost:" + port + "/ws-chat",
                new StompSessionHandlerAdapter() {}
        );
        StompSession session = future.get(3, TimeUnit.SECONDS);

        BlockingQueue<String> messages = new ArrayBlockingQueue<>(1);
        session.subscribe("/topic/chat", new StompFrameHandler() {
            @Override
            public Type getPayloadType(StompHeaders headers) {
                return String.class;
            }

            @Override
            public void handleFrame(StompHeaders headers, Object payload) {
                messages.offer(payload.toString());
            }
        });

        session.send("/app/chat.send", "{\"from\":\"test\",\"text\":\"hi\"}");
        assertThat(messages.poll(2, TimeUnit.SECONDS)).contains("hi");
    }
}
```

Although the test uses native WebSockets, SockJS fallbacks remain compatible. If you need to exercise fallback transports explicitly, consider using a headless browser with network throttling scripts or integration suites like Selenium.

## Deployment Considerations

`.withSockJS()` is only part of production hardening. Keep these additional ideas in mind:

- **Reverse proxies**: SockJS fallbacks issue more HTTP requests; configure sticky sessions or route sessions via an external broker to avoid state mismatches.
- **CORS**: When serving the client from another origin, align `setAllowedOriginPatterns` or register a dedicated `CorsRegistry` bean; SockJS XHR fallbacks obey the same CORS policy as any REST endpoint.
- **Compression**: Some proxies compress streaming responses, which can delay frame flushes. Disable HTTP compression if messages arrive in bursts rather than streaming.
- **Thread pools**: Long polling increases request concurrency. Monitor `server.tomcat.max-threads` (or Reactor Netty equivalents) and adjust based on load tests.
- **Custom `TaskScheduler`**: For high throughput, provide a dedicated scheduler to handle SockJS heartbeats so they do not starve other tasks.

## Clean Shutdown and Presence Tracking

Fallback transports introduce additional session lifecycle nuances. Consider subscribing to application events for analytics or cleanup:

```java
package com.example.chat.monitoring;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.context.ApplicationListener;
import org.springframework.messaging.simp.stomp.StompHeaderAccessor;
import org.springframework.web.socket.messaging.AbstractSubProtocolEvent;
import org.springframework.web.socket.messaging.SessionConnectEvent;
import org.springframework.web.socket.messaging.SessionDisconnectEvent;
import org.springframework.stereotype.Component;

@Component
public class PresenceEventListener implements ApplicationListener<AbstractSubProtocolEvent> {

    private static final Logger log = LoggerFactory.getLogger(PresenceEventListener.class);

    @Override
    public void onApplicationEvent(AbstractSubProtocolEvent event) {
        StompHeaderAccessor accessor = StompHeaderAccessor.wrap(event.getMessage());
        if (event instanceof SessionConnectEvent) {
            log.info("Client connected: {}", accessor.getSessionId());
        } else if (event instanceof SessionDisconnectEvent disconnect) {
            log.info("Client disconnected: {} status={} reason={}", accessor.getSessionId(), disconnect.getCloseStatus(), disconnect.getCloseStatus().getReason());
        }
    }
}
```

Spring publishes `SessionConnectEvent` and `SessionDisconnectEvent` regardless of the underlying transport, letting you track presence or clean up resources as clients arrive and depart.

## Next Steps

You now have a SockJS-capable STOMP endpoint with a browser client that remains functional even when native WebSockets are blocked. Run a regression check:

```bash
./mvnw test
./mvnw spring-boot:run
```

Open two browsers—ideally one legacy or locked-down environment—and confirm that messages keep flowing while the **Active transport** indicator shows fallbacks in action. SockJS ensures this toy project is universally accessible, demonstrating a pragmatic pattern you can carry into production-grade systems.

In **Article 5** we will secure the STOMP channel, adding authentication layers and CSRF protection so you can safely expose the toy service beyond localhost.
