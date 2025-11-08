# The STOMP Startup: Connecting Spring Boot and Plain HTML for Simple Real-Time Messaging

> curated by @psousa - Nov 2025

Real-time messaging is no longer a luxury feature reserved for massive social platforms. Even the most modest internal dashboards or toy projects can benefit from instantly pushing information between a server and a browser without forcing the user to refresh the page. In this opening article of our STOMP toy project series, we will set the foundation for everything that follows: creating a Spring Boot application that speaks STOMP over WebSocket, pairing it with a minimalist HTML client, and watching the two exchange messages in real time.

We will keep this journey deliberately hands-on. You will create a brand-new project, add only the dependencies you truly need, wire up the required configuration classes, and build your first controller that can receive and broadcast messages. On the front end, you will write a simple HTML page with a sprinkling of vanilla JavaScript that opens a STOMP connection, sends data, and displays the server’s responses in the browser.

By the end of this article you will have:

- A running Spring Boot application exposing a STOMP endpoint.
- A clear understanding of how STOMP frames map to Spring annotations like `@MessageMapping`.
- A test client you can open in a browser to immediately verify the “ping-pong” message flow.

Throughout the series we will reuse this skeleton, layering richer features on top. For now, let’s focus on understanding the moving pieces and getting the simplest STOMP pipeline working end to end.

## Prerequisites and Tools

This tutorial assumes you are comfortable with basic Spring Boot development and have Java 17+ installed. You should also have Maven (or Gradle) available, although we will use Maven commands in our examples. On the client side we rely purely on HTML, CSS, and vanilla JavaScript—no modern build steps required.

You will also need:

- A terminal or IDE capable of running Spring Boot (IntelliJ IDEA, VS Code with the Java extension, or the Spring CLI).
- A modern browser (Chrome, Firefox, Edge, Safari) to host the simple test page.
- Optional: `curl` or an equivalent tool to make quick HTTP requests and check the health of the service.

If you are brand new to WebSockets, do not worry. STOMP (Simple Text Oriented Messaging Protocol) gives us a structured frame format that makes exchanging messages intuitive. Spring’s messaging support takes care of most of the boilerplate, leaving us to focus on application logic.

## Understanding WebSocket and STOMP in Spring Boot

Before we write any code, let’s clarify the relationships between WebSocket, STOMP, and Spring’s programming model.

- **WebSocket** is a network protocol that upgrades an HTTP connection to a persistent, full-duplex socket. Once the handshake is complete, both server and client can send data at any time without the overhead of additional HTTP requests.
- **STOMP** rides on top of WebSocket (or other transports) and defines an interoperable frame format and common verbs (CONNECT, SUBSCRIBE, SEND, etc.). This makes it much easier to build messaging applications that understand concepts like topics and queues.
- **Spring Messaging** provides an abstraction layer that maps STOMP destinations to annotated controller methods. You declare methods with `@MessageMapping` (for application destinations) or `@SendTo` (for broker destinations), and Spring handles the translation between STOMP frames and your Java objects.

In our minimal “ping-pong” application we will configure a `/ws` endpoint that browsers can connect to via WebSocket. Messages sent to `/app/ping` will be routed to our controller, and responses will be broadcast to anyone subscribed to `/topic/pong`. The in-memory simple broker included in `spring-messaging` will handle routing to subscribers.

## Bootstrapping the Project

Let’s start from scratch with a new Spring Boot project. If you prefer to use the Spring Initializr web UI, choose Java, Maven, Spring Boot 3.x, and add the **Spring Websocket** dependency. Alternatively, you can generate the project from the command line:

```bash
curl https://start.spring.io/starter.zip \
  -d dependencies=websocket \
  -d language=java \
  -d platformVersion=3.3.4 \
  -d javaVersion=17 \
  -d type=maven-project \
  -d baseDir=stomp-toy-01 \
  -o stomp-toy-01.zip
```

Unzip the archive and open the project in your IDE of choice.

Your `pom.xml` should contain the websocket starter dependency, which in turn brings in `spring-messaging` and the embedded Tomcat server:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-websocket</artifactId>
</dependency>
```

Because we are building a toy project, we do not need a relational database or JPA. The in-memory broker that ships with Spring Messaging will be sufficient for now.

If you prefer Gradle, the equivalent dependency looks like this:

```groovy
implementation 'org.springframework.boot:spring-boot-starter-websocket'
```

Run `./mvnw spring-boot:run` (or `./gradlew bootRun`) once to make sure the project builds and starts. You should see the standard Spring Boot banner and the message that the application started on port 8080.

## Structuring the Application

Because this is a minimal application, we will keep everything under a single package: `com.example.stompstartup`. Inside `src/main/java`, create the following classes:

- `WebSocketConfig` to register the STOMP endpoint and configure the broker.
- `PingPongController` to handle incoming messages.
- Optionally, `PingMessage` and `PongMessage` records for cleaner payload mapping.

Let’s start with the configuration class.

```java
package com.example.stompstartup;

import org.springframework.context.annotation.Configuration;
import org.springframework.messaging.simp.config.MessageBrokerRegistry;
import org.springframework.web.socket.config.annotation.EnableWebSocketMessageBroker;
import org.springframework.web.socket.config.annotation.StompEndpointRegistry;
import org.springframework.web.socket.config.annotation.WebSocketMessageBrokerConfigurer;

@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {

    @Override
    public void configureMessageBroker(MessageBrokerRegistry registry) {
        registry.enableSimpleBroker("/topic");
        registry.setApplicationDestinationPrefixes("/app");
    }

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/ws")
                .setAllowedOriginPatterns("*");
    }
}
```

The `@EnableWebSocketMessageBroker` annotation activates Spring’s STOMP message handling. We tell Spring that any destination starting with `/topic` should be handled by the simple broker (which will broadcast to subscribers), and any destination starting with `/app` should be routed to our controller methods. Finally, we expose an endpoint at `/ws` that browsers can connect to. For now we allow all origins to keep testing easy; later in the series we will replace this with a proper CORS policy.

### Creating the Message Payloads

Although STOMP frames are textual, Spring can automatically serialize and deserialize JSON payloads for us. We will define two simple records to represent the messages traveling between client and server:

```java
package com.example.stompstartup;

public record PingMessage(String clientId, String content) {}
```

```java
package com.example.stompstartup;

public record PongMessage(String serverId, String content, long timestamp) {}
```

These records give us strongly typed fields and keep the code readable. The `PingMessage` carries the client ID and raw content the user typed. The server responds with a `PongMessage` containing its own identifier and a timestamp we can display on the page.

### Writing the Controller

Next we will author the heart of the application: the controller that receives `PingMessage` payloads and broadcasts `PongMessage` responses.

```java
package com.example.stompstartup;

import java.time.Instant;
import java.util.UUID;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.messaging.handler.annotation.MessageMapping;
import org.springframework.messaging.handler.annotation.Payload;
import org.springframework.messaging.handler.annotation.SendTo;
import org.springframework.stereotype.Controller;

@Controller
public class PingPongController {

    private static final Logger log = LoggerFactory.getLogger(PingPongController.class);
    private final String serverId = UUID.randomUUID().toString();

    @MessageMapping("/ping")
    @SendTo("/topic/pong")
    public PongMessage handlePing(@Payload PingMessage ping) {
        log.info("Received ping from {}: {}", ping.clientId(), ping.content());
        return new PongMessage(serverId, "PONG: " + ping.content(), Instant.now().toEpochMilli());
    }
}
```

Here are the key annotations:

- `@MessageMapping("/ping")` maps to the application destination `/app/ping`. When a client sends a STOMP SEND frame to `/app/ping`, this method is invoked.
- `@SendTo("/topic/pong")` tells Spring to broadcast the returned `PongMessage` to all subscribers of the `/topic/pong` destination. Under the hood, Spring converts the object to JSON and wraps it in a STOMP MESSAGE frame.

We also log incoming messages to the console for easier debugging.

## Building the Minimal HTML Client

With the server ready, let’s craft a simple client you can open directly from the filesystem. Create `src/main/resources/static/index.html` with the following contents:

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>STOMP Ping-Pong</title>
    <style>
        body { font-family: system-ui, sans-serif; margin: 2rem; }
        #log { border: 1px solid #ccc; padding: 1rem; min-height: 10rem; background: #f8f9fb; }
        .pong { margin-bottom: 0.75rem; }
        label { display: block; margin-top: 1rem; }
        input, button { padding: 0.5rem; }
    </style>
</head>
<body>
<h1>STOMP Ping-Pong Demo</h1>

<div>
    <label for="clientId">Client ID</label>
    <input id="clientId" value="client-" placeholder="Enter a unique client id" />
</div>

<div>
    <label for="message">Message</label>
    <input id="message" value="Hello server" />
    <button id="send">Send Ping</button>
</div>

<section>
    <h2>Connection</h2>
    <button id="connect">Connect</button>
    <button id="disconnect" disabled>Disconnect</button>
</section>

<section>
    <h2>Server Responses</h2>
    <div id="log"></div>
</section>

<script type="module">
    import { Client } from 'https://cdn.jsdelivr.net/npm/@stomp/stompjs@7.0.0/esm6/index.min.js';

    const connectButton = document.getElementById('connect');
    const disconnectButton = document.getElementById('disconnect');
    const sendButton = document.getElementById('send');
    const log = document.getElementById('log');
    const messageInput = document.getElementById('message');
    const clientIdInput = document.getElementById('clientId');

    let stompClient;

    function appendLog(entry) {
        const div = document.createElement('div');
        div.className = 'pong';
        div.textContent = entry;
        log.prepend(div);
    }

    connectButton.addEventListener('click', () => {
        if (stompClient && stompClient.connected) {
            appendLog('Already connected.');
            return;
        }

        stompClient = new Client({
            brokerURL: `ws://${window.location.host}/ws`,
            reconnectDelay: 5000,
            debug: str => console.log(str)
        });

        stompClient.onConnect = () => {
            appendLog('Connected to STOMP broker.');
            stompClient.subscribe('/topic/pong', message => {
                const body = JSON.parse(message.body);
                appendLog(`Server ${body.serverId} replied at ${new Date(body.timestamp).toLocaleTimeString()}: ${body.content}`);
            });
            connectButton.disabled = true;
            disconnectButton.disabled = false;
        };

        stompClient.onStompError = frame => {
            appendLog(`Broker error: ${frame.headers['message']}`);
        };

        stompClient.onWebSocketError = event => {
            appendLog(`WebSocket error: ${event}`);
        };

        stompClient.activate();
    });

    disconnectButton.addEventListener('click', () => {
        if (!stompClient || !stompClient.connected) {
            appendLog('Not connected.');
            return;
        }
        stompClient.deactivate();
        appendLog('Disconnected.');
        connectButton.disabled = false;
        disconnectButton.disabled = true;
    });

    sendButton.addEventListener('click', () => {
        if (!stompClient || !stompClient.connected) {
            appendLog('Connect first before sending.');
            return;
        }
        const payload = {
            clientId: clientIdInput.value || `client-${crypto.randomUUID().slice(0, 8)}`,
            content: messageInput.value
        };
        stompClient.publish({ destination: '/app/ping', body: JSON.stringify(payload) });
        appendLog(`Sent ping: ${payload.content}`);
    });

    clientIdInput.value += crypto.randomUUID().slice(0, 8);
</script>
</body>
</html>
```

This client uses the modern ESM build of `@stomp/stompjs`, loaded directly from the jsDelivr CDN. When you click “Connect,” it opens a WebSocket connection to `/ws`, negotiates the STOMP protocol, and subscribes to `/topic/pong`. The “Send Ping” button publishes a JSON payload to `/app/ping`, which the server controller consumes.

We prepend log entries to keep the newest messages at the top. CSS is minimal yet pleasant enough for quick demos. Because we operate a toy project, copying this file to any static hosting service or serving it from the Spring Boot `static` directory is fine.

## Running the Application

Start the server:

```bash
./mvnw spring-boot:run
```

Open your browser and navigate to `http://localhost:8080/index.html`. (Spring Boot automatically serves static resources under `/static`.) Click “Connect” and watch the log area display `Connected to STOMP broker.` Enter a message—“Hello from the browser!”—and press “Send Ping.” Within milliseconds you should see a response message like `Server 9f54... replied at 10:22:13 AM: PONG: Hello from the browser!`

If you want to test with multiple clients, open the page in multiple tabs or different browsers. Each will receive the broadcast message because they are all subscribed to `/topic/pong`. Change the `message` or `clientId` fields to observe how the server responds.

### Verifying the STOMP Frames

For extra confidence, open your browser’s developer tools, switch to the Network tab, and filter by “WS” for WebSocket connections. Clicking on the `/ws` request reveals the raw STOMP frames. You will see a CONNECT frame followed by the SUBSCRIBE frame when the client joins `/topic/pong`. Each published message appears as a SEND frame, and the server replies with MESSAGE frames. This visibility is invaluable when debugging future features.

## Troubleshooting Common Issues

Even a small application can run into roadblocks. Here are common problems and their fixes:

- **CORS or Origin Errors:** If you host the HTML page on a different domain or port, browsers may block the WebSocket connection. Use `setAllowedOriginPatterns("http://localhost:8080")` with the correct host or tighten the configuration once you deploy.
- **404 on `/ws`:** Ensure the WebSocket endpoint is correct (`registry.addEndpoint("/ws")`). If you renamed the path, update the client’s `brokerURL` accordingly.
- **JSON Mapping Errors:** Spring uses Jackson to map JSON payloads to Java records. If field names differ or a field is missing, you will see conversion errors in the logs. Check the browser console to make sure you are sending the expected JSON structure.
- **STOMP Client Library Mismatch:** Different versions of `@stomp/stompjs` have slightly different import paths. Using the ESM build as shown keeps things straightforward. If you prefer a bundled version, include the UMD script with a `<script src="...">` tag and use the global `StompJs.Client` constructor.

## Expanding the Toy Project

You now have a solid core that can evolve in many directions. Here are a few optional enhancements you can already experiment with:

- Add a heartbeat interval by configuring the `Client` options (`heartbeatIncoming` / `heartbeatOutgoing`) and updating the server configuration with `registry.enableSimpleBroker("/topic").setHeartbeatValue(...)`.
- Introduce message validation in the controller. For example, reject empty messages or enforce a maximum length.
- Display connection status more prominently in the client, such as changing colors or disabling inputs when disconnected.
- Persist messages in memory so new clients can fetch the latest conversation history upon connection (a concept we will revisit with `@SubscribeMapping`).

These are not essential for our first milestone, but they tease topics we will tackle in later articles.

## Looking Ahead

In this first installment we focused on wiring the basic pieces: a Spring Boot application configured for STOMP messaging and a browser client that can talk to it. The code is intentionally lightweight and approachable. In the next articles we will keep layering complexity: scheduled server pushes, chat rooms, SockJS fallbacks, authentication, user destinations, and eventually scaling to external brokers.

Before moving on, make sure you are comfortable with the following checklist:

- You can start and stop the Spring Boot application without errors.
- The browser client connects, subscribes, sends messages, and receives broadcasts.
- You can open multiple client windows and watch broadcast behavior.
- You understand the flow: client SEND to `/app/ping` -> controller -> broker -> clients subscribed to `/topic/pong`.

Mastering this foundation ensures every new feature builds on solid ground. Keep the project handy—you will reuse it throughout the series—and feel free to experiment with playful messages or UI adjustments. When you are ready, we will move on to server-driven updates that keep clients synced in real time without user interaction.

