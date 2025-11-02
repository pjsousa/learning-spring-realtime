## STOMP Toy Project Series Plan

- **Article 1** – The STOMP Startup: configure Spring Boot, enable the broker, wire a “ping-pong” exchange, demo with plain HTML.
- **Article 2** – Channeling Conversations: add multiple destinations, namespaces, and routing logic for one-to-many chat rooms.
- **Article 3** – User-Specific Messaging: explore `/user` queues, session tracking, and private notifications.
- **Article 4** – State Synchronization: broadcast shared state updates for collaborative counters, lists, and toggles.
- **Article 5** – Reliable Delivery: add heartbeat tuning, reconnection logic, and message acknowledgements.
- **Article 6** – Validation & Security: secure endpoints with Spring Security, CSRF headers, and input validation.
- **Article 7** – Scaling Out: swap the simple broker for a broker relay (RabbitMQ/ActiveMQ) and tune performance.
- **Article 8** – Testing Strategies: unit, integration, and browser-automation testing for STOMP flows.
- **Article 9** – Observability: instrument metrics, tracing, and logging for live WebSocket systems.
- **Article 10** – Production Hardening: deployment checklists, load testing, and graceful degradation patterns.

---

# The STOMP Startup: Connecting Spring Boot and Plain HTML for Simple Real-Time Messaging

*Estimated reading time: 20 minutes · Target word count: ~1,800*

## Overview

This opening article launches a 10-part journey that incrementally builds a collection of toy projects showcasing Spring Boot, STOMP, and plain HTML/JavaScript clients. We begin by answering two foundational questions:

1. **What problems do WebSockets and STOMP solve together?**
2. **How do we stand up a minimal Spring Boot service and browser client that can exchange messages immediately?**

By the end, you will have a working “ping-pong” system. A simple HTML page opens a STOMP connection to a Spring Boot application, sends a “hello” message, receives a greeting back, and renders the response. This skeleton becomes the backbone for every subsequent article, letting us layer richer behaviour—multi-room chat, private messaging, reliability, security, scaling, testing, and production-readiness.

## Learning Goals

- Understand the relationship between WebSocket and STOMP, and why Spring abstracts them with a brokered messaging model.
- Scaffold a Spring Boot 3.2+ application that enables `spring-websocket` and `spring-messaging`.
- Configure a STOMP endpoint (`/gs-guide-websocket`), application destination prefix (`/app`), and in-memory simple broker (`/topic`).
- Implement a `@MessageMapping` handler that echoes greetings and forwards them to subscribed clients.
- Build a minimal plain HTML/JS client that connects, subscribes, sends JSON payloads, and updates the DOM.
- Troubleshoot common first-day issues: missing dependencies, mismatched destinations, and cross-origin restrictions.

## Why Real-Time Messaging Needs STOMP

WebSockets upgrade an HTTP connection into a persistent, bidirectional pipe. That pipe, however, speaks in raw frames with no message semantics. You could invent your own protocol, but doing so introduces needless risk: how do you describe message destinations, error frames, heartbeat intervals, or acknowledgements?

STOMP—the **S**imple **T**ext **O**riented **M**essaging **P**rotocol—solves this by defining a minimal command set (`CONNECT`, `SEND`, `SUBSCRIBE`, `MESSAGE`, `DISCONNECT`, etc.). Each frame carries headers (destinations, receipt IDs, content type) and optional payloads. On the server side, Spring’s messaging support maps STOMP frames to annotated methods, similar to how Spring MVC maps HTTP requests.

In short:

- **WebSocket transports the bytes.**
- **STOMP defines the message format and routing semantics.**
- **Spring abstracts both through a broker architecture**: clients publish to `/app/...`, application code processes the message, and the broker fan-outs to subscribers on `/topic/...` or `/queue/...` destinations.

Understanding this separation is essential. We will repeatedly rely on the same concepts in later installments to introduce routing, acknowledgements, security, and scaling.

## Prerequisites and Tooling

Make sure your environment resembles the following:

- **Java**: 17 or 21. Spring Boot 3.x requires Java 17+, but using 21 ensures long-term support.
- **Build Tool**: Maven 3.9+ or Gradle 8+. This article assumes Maven; Gradle snippets are included where helpful.
- **Spring Boot Initializer**: either via the web UI (<https://start.spring.io/>) or the command-line `curl` example below.
- **IDE/Text Editor**: IntelliJ IDEA, VS Code, or your favorite code editor.
- **Browser**: any modern browser with ES6 support (Chrome, Firefox, Edge, Safari). The client relies on standard DOM APIs plus SockJS and STOMP.js.

Optional but recommended:

- `httpie` or `curl` for quick sanity checks on the HTTP layer.
- Browser DevTools to monitor network frames and console logs.
- Familiarity with basic Spring MVC concepts (controllers, dependency injection) and JSON serialization.

## Generate the Project Skeleton

We start with the leanest possible Spring Boot project. Run the following commands from your development directory:

```bash
curl https://start.spring.io/starter.zip \
  -d dependencies=websocket \
  -d name=stomp-startup \
  -d javaVersion=21 \
  -d type=maven-project \
  -d packageName=com.example.stompstartup \
  -o stomp-startup.zip

unzip stomp-startup.zip
cd stomp-startup
```

The `websocket` starter transitively brings in `spring-messaging`, which provides STOMP abstractions. Inspect the `pom.xml` to confirm the dependencies and note the Java version target.

### Maven Dependency Snapshot

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
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
  </dependency>
</dependencies>
```

For Gradle (Kotlin DSL), dependencies would look like:

```kotlin
dependencies {
  implementation("org.springframework.boot:spring-boot-starter-websocket")
  implementation("org.springframework.boot:spring-boot-starter-web")
  testImplementation("org.springframework.boot:spring-boot-starter-test")
}
```

At this stage, running `./mvnw spring-boot:run` should launch a simple “Hello World” application serving the default actuator endpoints.

## Understand Spring’s WebSocket Architecture

Before writing any code, map the components you are about to configure:

| Component | Purpose |
|-----------|---------|
| `@EnableWebSocketMessageBroker` | Enables broker-backed STOMP messaging support, registers message channels, configures endpoints. |
| `registerStompEndpoints` | Exposes one or more endpoints that clients use when initiating the WebSocket (or SockJS) handshake. |
| `configureMessageBroker` | Configures destination prefixes. `enableSimpleBroker` activates Spring’s in-memory broker for `/topic/**` and `/queue/**`. `setApplicationDestinationPrefixes` maps application-level destinations like `/app/**`. |
| `@MessageMapping` | Marks controller methods that receive messages. Each method’s mapping corresponds to a destination under the application prefix. |
| `@SendTo` or `SimpMessagingTemplate` | Defines where responses get published. Subscribers listen on these destinations. |

Keep these moving pieces in mind; they come together in the configuration and controller below.

## Step 1: Enable WebSocket Message Broker Support

Create `src/main/java/com/example/stompstartup/config/WebSocketConfig.java`:

```java
package com.example.stompstartup.config;

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
    registry.addEndpoint("/gs-guide-websocket")
            .setAllowedOriginPatterns("*")
            .withSockJS();
  }
}
```

**Key decisions explained:**

- `enableSimpleBroker("/topic")` spins up Spring’s lightweight, in-memory broker. It’s perfect for toy projects and small demos. In Article 7 we swap it for a full broker relay.
- `setApplicationDestinationPrefixes("/app")` ensures client messages sent to `/app/...` are routed to `@MessageMapping` methods. Changing the prefix later only requires updating a single configuration and the client endpoints.
- `addEndpoint("/gs-guide-websocket")` defines where clients connect. We allow all origins for simplicity; in production, restrict this to trusted hosts. SockJS adds fallback transports (XHR streaming, long polling) for older browsers. Future articles will demonstrate toggling or removing SockJS depending on deployment targets.

## Step 2: Create Message Payload Classes

Data Transfer Objects (DTOs) describe the payloads passing between client and server. Place them under `com.example.stompstartup.messages`.

`HelloMessage.java`:

```java
package com.example.stompstartup.messages;

public class HelloMessage {

  private String name;

  public HelloMessage() {
  }

  public HelloMessage(String name) {
    this.name = name;
  }

  public String getName() {
    return name;
  }

  public void setName(String name) {
    this.name = name;
  }
}
```

`Greeting.java`:

```java
package com.example.stompstartup.messages;

public class Greeting {

  private String content;

  public Greeting() {
  }

  public Greeting(String content) {
    this.content = content;
  }

  public String getContent() {
    return content;
  }

  public void setContent(String content) {
    this.content = content;
  }
}
```

We include default constructors because Jackson (the JSON serializer) relies on them. In later articles, you will see how validation and record types change this structure.

## Step 3: Wire a STOMP Controller

Create `src/main/java/com/example/stompstartup/controller/GreetingController.java`:

```java
package com.example.stompstartup.controller;

import com.example.stompstartup.messages.Greeting;
import com.example.stompstartup.messages.HelloMessage;
import org.springframework.messaging.handler.annotation.MessageMapping;
import org.springframework.messaging.handler.annotation.Payload;
import org.springframework.messaging.handler.annotation.SendTo;
import org.springframework.stereotype.Controller;

@Controller
public class GreetingController {

  @MessageMapping("/hello")
  @SendTo("/topic/greetings")
  public Greeting greet(@Payload HelloMessage message) {
    String name = message.getName();
    String safeName = (name == null || name.isBlank()) ? "Anonymous" : name.trim();
    return new Greeting("Hello, " + safeName + "! 👋");
  }
}
```

Flow summary:

1. The browser sends a STOMP frame to `/app/hello` with a JSON body such as `{ "name": "Avery" }`.
2. Spring deserializes that payload into `HelloMessage` and passes it to `greet`.
3. The method builds a `Greeting` response, which is forwarded to `/topic/greetings` thanks to the `@SendTo` annotation.
4. Any client subscribed to `/topic/greetings` receives the message, even if multiple tabs are open.

You can replace `@SendTo` with `SimpMessagingTemplate` for advanced routing (e.g., user-specific queues), which we explore next articles.

## Step 4: Build a Browser Test Client

Spring Boot automatically serves static resources from `src/main/resources/static`. Create `index.html` in that directory:

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <title>STOMP Startup Ping-Pong</title>
  <style>
    body { font-family: system-ui, sans-serif; margin: 2rem; }
    #conversation { margin-top: 1.5rem; border: 1px solid #ccc; padding: 1rem; min-height: 150px; }
    .message { margin-bottom: 0.75rem; }
    .message span { display: inline-block; padding: 0.25rem 0.5rem; background: #f3f4f6; border-radius: 4px; }
    form { margin-top: 1rem; display: flex; gap: 0.75rem; }
    input[type="text"] { flex: 1; padding: 0.5rem; font-size: 1rem; }
    button { padding: 0.5rem 1.25rem; font-size: 1rem; cursor: pointer; }
    #status { margin-top: 0.5rem; font-style: italic; color: #4b5563; }
  </style>
</head>
<body>
  <h1>The STOMP Startup</h1>
  <p>Step 1: Connect the client, then send your name to get a greeting back via STOMP.</p>

  <div>
    <button id="connect">Connect</button>
    <button id="disconnect" disabled>Disconnect</button>
    <span id="status">Disconnected</span>
  </div>

  <form id="helloForm">
    <input type="text" id="name" placeholder="Your name" autocomplete="given-name" />
    <button type="submit">Send</button>
  </form>

  <div id="conversation"></div>

  <script src="https://cdn.jsdelivr.net/npm/sockjs-client@1/dist/sockjs.min.js"></script>
  <script src="https://cdn.jsdelivr.net/npm/stompjs@2.3.3/lib/stomp.min.js"></script>
  <script>
    let stompClient = null;

    function setConnected(connected) {
      document.getElementById("connect").disabled = connected;
      document.getElementById("disconnect").disabled = !connected;
      document.getElementById("status").textContent = connected ? "Connected" : "Disconnected";
    }

    function connect() {
      const socket = new SockJS("/gs-guide-websocket");
      stompClient = Stomp.over(socket);
      stompClient.connect({}, frame => {
        console.log("Connected: ", frame);
        setConnected(true);
        stompClient.subscribe("/topic/greetings", greeting => {
          showGreeting(JSON.parse(greeting.body).content);
        });
      }, error => {
        setConnected(false);
        console.error("Connection closed:", error);
      });
    }

    function disconnect() {
      if (stompClient !== null) {
        stompClient.disconnect(() => setConnected(false));
        stompClient = null;
      }
    }

    function sendName() {
      if (!stompClient || !stompClient.connected) {
        alert("Connect first!");
        return;
      }
      const name = document.getElementById("name").value;
      stompClient.send("/app/hello", {}, JSON.stringify({ name }));
      document.getElementById("name").value = "";
    }

    function showGreeting(message) {
      const container = document.getElementById("conversation");
      const p = document.createElement("p");
      p.classList.add("message");
      p.innerHTML = `<span>${message}</span>`;
      container.appendChild(p);
      container.scrollTop = container.scrollHeight;
    }

    document.getElementById("connect").addEventListener("click", event => {
      event.preventDefault();
      connect();
    });

    document.getElementById("disconnect").addEventListener("click", event => {
      event.preventDefault();
      disconnect();
    });

    document.getElementById("helloForm").addEventListener("submit", event => {
      event.preventDefault();
      sendName();
    });
  </script>
</body>
</html>
```

### Client Walkthrough

- **Transport Initialization**: `const socket = new SockJS("/gs-guide-websocket")` reuses the server-side endpoint. SockJS automatically downgrades the transport if native WebSocket is unavailable.
- **STOMP Session**: `Stomp.over(socket)` wraps the transport. Calling `connect` sends `CONNECT`. Spring replies with `CONNECTED` if the handshake succeeds.
- **Subscriptions**: `stompClient.subscribe("/topic/greetings", ...)` registers interest in broadcast messages.
- **Sending Messages**: The submit handler posts JSON to `/app/hello`, which maps to the controller method.
- **DOM Updates**: `showGreeting` appends the returned message to the conversation log, letting you open multiple tabs to observe broadcast behaviour.

The external CDN links simplify setup. If you prefer local assets, download them into `src/main/resources/static/js` and reference with relative paths.

## Step 5: Run and Verify

Start the Spring Boot application:

```bash
./mvnw spring-boot:run
```

Open `http://localhost:8080/` in your browser and follow these steps:

1. Click **Connect**. The status indicator flips to “Connected,” and the browser console logs the STOMP `CONNECTED` frame.
2. Enter a name and press **Send**. The request appears in the server logs, and the greeting is rendered in the conversation area.
3. Open a second tab or another browser window. Repeat the process and observe that both clients receive the greeting, proving the topic broadcast.

If you prefer automated verification, set the WebSocket logging level to `TRACE` in `src/main/resources/application.properties`:

```properties
logging.level.org.springframework.messaging=TRACE
logging.level.org.springframework.web.socket=TRACE
```

With these settings, Spring emits detailed logs of STOMP frames, subscription events, and message dispatch, which helps debug future enhancements.

## Troubleshooting Guide

| Symptom | Likely Cause | Fix |
|---------|--------------|-----|
| Browser console shows `404` or `Connection refused` when connecting | Endpoint mismatch or app not running | Confirm `registerStompEndpoints` path matches client SockJS path; ensure Spring Boot app is running on port 8080. |
| No greeting appears, but the client stays connected | Prefix mismatch | Verify the client sends to `/app/hello` and the controller uses `@MessageMapping("/hello")`. |
| `Failed to load sockjs.min.js` | CDN blocked or offline | Download assets locally or ensure network access to `cdn.jsdelivr.net`. |
| CORS errors in browser console | Client served from another origin | Replace `.setAllowedOriginPatterns("*")` with explicit origins or configure CORS in the controller. |
| JSON parsing errors or null values | Missing getters/setters or property names don’t line up | Ensure DTOs have getters/setters and the client uses matching JSON keys. |

During development, it often helps to test the HTTP upgrade path using `wscat` or browser DevTools. Seeing the raw frames clarifies whether issues are on the transport or application layer.

## Stretch Experiments

Ready to go beyond the basic flow? Try these incremental enhancements while keeping the core architecture untouched:

- **Input validation**: Reject empty or overly long names, responding with an error frame using `@SendToUser` for targeted feedback.
- **Custom headers**: Add metadata (timestamps, browser info) to the outgoing `Greeting` payload.
- **Client UX polish**: Disable the send button until the connection is established, show connection latency, or animate new messages.
- **Logging**: Inject `SimpMessageHeaderAccessor` into the controller to inspect session IDs and track connections.

Each tweak deepens your understanding of how STOMP frames map to Spring abstractions.

## Looking Ahead

This foundation supports everything else in the series:

- A working Spring Boot application with WebSocket/STOMP enabled.
- A reproducible build and runtime environment.
- A plain HTML/JS client ready to be expanded into multi-room chat, user-specific messaging, and robust error handling.

**Next up: “Channeling Conversations”**—we’ll extend this setup to support multiple chat rooms, introduce destination patterns, and manage subscriptions for groups of clients.

## Appendix: Complete File Tree

After following the steps above, your project structure should resemble the following (generated files omitted for brevity):

```text
stomp-startup/
├── mvnw
├── pom.xml
├── src
│   ├── main
│   │   ├── java
│   │   │   └── com
│   │   │       └── example
│   │   │           └── stompstartup
│   │   │               ├── StompStartupApplication.java
│   │   │               ├── config
│   │   │               │   └── WebSocketConfig.java
│   │   │               ├── controller
│   │   │               │   └── GreetingController.java
│   │   │               └── messages
│   │   │                   ├── Greeting.java
│   │   │                   └── HelloMessage.java
│   │   └── resources
│   │       ├── application.properties
│   │       └── static
│   │           └── index.html
└── src/test/java (default Spring Boot test skeleton)
```

> **Tip:** Commit this baseline before moving to Article 2. Having a clean, working checkpoint makes it easier to isolate issues when experimenting with new features.

## Suggested Git Workflow

Although this series primarily targets learning, version control hygiene keeps experiments manageable:

1. Initialize git (if not already): `git init` and commit the unmodified starter.
2. Create a feature branch, e.g., `feature/article-01-stomp-startup`.
3. Commit each major step—configuration, controller, client—to follow your learning journey.
4. Tag the repo (`git tag article-01`) once the article example works. Future articles can branch from this tag.

This practice provides a time machine: if you break something in Article 5, you can fast-forward or rewind at will.
