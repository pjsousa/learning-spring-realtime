# Automating STOMP API Specs: Leveraging Tools to Document Your Spring Message Services

*Estimated reading time: 25 minutes · Target word count: ~1,900*

## Overview

Throughout this series, we've built chat rooms, tickers, private messaging systems, and collaborative applications—all powered by STOMP over WebSocket. But as your real-time services grow, a critical question emerges: **How do other developers (or your future self) understand what destinations exist, what messages to send, and what responses to expect?**

Unlike REST APIs, which rely on widely accepted conventions like URLs and HTTP verbs, STOMP operates over a single persistent connection with messaging semantics defined entirely by your application. The meaning of a destination like `/app/chat.send` or `/topic/prices` is intentionally opaque in the STOMP specification—it's up to your server to define what these destinations mean, what payloads they accept, and where messages flow.

This article guides you through the importance of documenting asynchronous STOMP services and demonstrates how existing community tools can automatically generate API documentation (like AsyncAPI specifications) from your Spring configuration. By the end, you'll have integrated documentation generation into your build process, enabling better client integration and smoother team collaboration.

## Learning Goals

After completing this article, you will be able to:

- Understand why documenting asynchronous messaging services is more critical than documenting REST APIs.
- Identify the key elements of a STOMP service that must be documented: destinations, commands, payloads, and message flow.
- Integrate open-source tools that scan Spring annotations and generate AsyncAPI specifications automatically.
- Use generated documentation to guide client-side development and integration testing.
- Configure documentation generation as part of your build process for continuous API documentation.

## Why Documenting Asynchronous Messaging is Essential

Consider a typical REST API: a developer can inspect the URL (`/api/users/123`), HTTP method (`GET`), and response format to understand the contract. The semantics are largely self-evident from the URL structure and HTTP verbs.

STOMP services operate differently:

1. **Single Connection Architecture**: All communication happens over one WebSocket connection. Destinations like `/app/chat.send` are application-defined strings with no inherent meaning.
2. **Bidirectional Flow**: Messages flow both ways—clients send commands, servers broadcast responses—but the relationship between a SEND and the resulting MESSAGE frames isn't obvious without documentation.
3. **Opaque Destinations**: A destination like `/topic/prices` tells you nothing about what data it contains, how often it updates, or what format the payload uses.
4. **Message Semantics**: The STOMP protocol defines the frame structure (SEND, SUBSCRIBE, MESSAGE), but your application defines what those frames mean in context.

**The Analogy**: Documenting an asynchronous service is like giving a driver a detailed route map instead of just a street address. While the address (the WebSocket connection URL) gets them started, the map (the documentation) tells them exactly which turns to take (STOMP commands), what roads lead where (destinations), and what type of cargo they'll carry (message payloads) to ensure they arrive at the correct destination with the right goods. Without the map, they might arrive, but they won't know how to interact with the location or what to expect in return.

## Identifying Key Service Elements to Document

Before we automate documentation generation, let's identify what needs to be documented. A comprehensive STOMP API specification must cover:

### 1. STOMP Destinations

**Application Destinations** (prefixed by `/app`): These are routed to `@MessageMapping` methods in your controllers. Examples:
- `/app/chat.send` → routes to a controller method handling chat messages
- `/app/private.send` → routes to a private messaging handler
- `/app/prices/request-once` → routes to a one-off price request handler

**Broker Destinations** (e.g., `/topic` or `/queue`): These are used for broadcasting or point-to-point exchanges:
- `/topic/public` → broadcasts messages to all subscribers
- `/topic/prices` → broadcasts price updates to all subscribers
- `/queue/private` → delivers messages to a specific user's queue
- `/user/queue/notifications` → user-specific notification queue

### 2. Message Frames and Payloads

**SEND Command Payloads**: The structure of messages sent using the STOMP SEND command. This involves:
- The Java objects used (like `HelloMessage`, `PrivateMessage`, `PriceUpdate`)
- How Spring uses message converters to handle serialization (typically to JSON)
- Required vs. optional fields
- Validation rules

**MESSAGE Frame Payloads**: The structure of MESSAGE frames received by subscribers:
- Response objects (like `Greeting`, `PriceUpdate`)
- How these objects are serialized to JSON
- What triggers each type of message

### 3. Message Handling Logic

**Controller Methods**: How annotated controller methods, utilizing `@MessageMapping` and `@SendTo`, determine message flow:
- Which destinations map to which methods
- What processing occurs (validation, transformation, business logic)
- Where responses are sent (`@SendTo` destinations)
- User-specific routing (`@SendToUser` or `SimpMessagingTemplate.convertAndSendToUser()`)

### 4. Connection and Subscription Patterns

**Connection Requirements**: 
- WebSocket endpoint URL
- Authentication requirements
- SockJS fallback configuration

**Subscription Patterns**:
- Which destinations clients should subscribe to
- When subscriptions should be established (on connect, on demand)
- Expected message frequency and volume

## Project Setup: Building a Documentable STOMP Service

We'll create a sample service that demonstrates all the elements we need to document. This builds on concepts from previous articles but focuses on creating a well-structured, documentable API.

### Generate the Project

```bash
curl https://start.spring.io/starter.zip \
  -d dependencies=websocket,web,validation \
  -d name=stomp-api-docs \
  -d javaVersion=21 \
  -d type=maven-project \
  -d packageName=com.example.stompapidocs \
  -o stomp-api-docs.zip

unzip stomp-api-docs.zip
cd stomp-api-docs
```

### Dependencies for Documentation Generation

We'll use the `springwolf` library, which automatically scans Spring WebSocket annotations and generates AsyncAPI specifications. Add these dependencies to your `pom.xml`:

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
    
    <!-- Springwolf for AsyncAPI documentation generation -->
    <dependency>
        <groupId>io.github.springwolf</groupId>
        <artifactId>springwolf-core</artifactId>
        <version>1.0.0</version>
    </dependency>
    <dependency>
        <groupId>io.github.springwolf</groupId>
        <artifactId>springwolf-stomp</artifactId>
        <version>1.0.0</version>
    </dependency>
    <dependency>
        <groupId>io.github.springwolf</groupId>
        <artifactId>springwolf-ui</artifactId>
        <version>1.0.0</version>
    </dependency>
</dependencies>
```

**Note**: Springwolf versions may vary. Check the [Springwolf GitHub repository](https://github.com/springwolf/springwolf-core) for the latest version. If Springwolf isn't available or you prefer a different approach, we'll also show how to manually create AsyncAPI specs.

### Basic WebSocket Configuration

Create the standard STOMP configuration:

**`src/main/java/com/example/stompapidocs/config/WebSocketConfig.java`**

```java
package com.example.stompapidocs.config;

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
        registry.enableSimpleBroker("/topic", "/queue");
        registry.setApplicationDestinationPrefixes("/app");
        registry.setUserDestinationPrefix("/user");
    }

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/ws")
                .setAllowedOriginPatterns("*")
                .withSockJS();
    }
}
```

This configuration enables:
- Simple broker for `/topic` and `/queue` destinations
- Application destination prefix `/app` for client-to-server messages
- User destination prefix `/user` for user-specific messaging
- STOMP endpoint at `/ws` with SockJS fallback

## Building a Sample Service to Document

Let's create a service that demonstrates multiple messaging patterns we'll document:

### Message Payload Classes

**`src/main/java/com/example/stompapidocs/messages/ChatMessage.java`**

```java
package com.example.stompapidocs.messages;

import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.Size;

public class ChatMessage {
    
    @NotBlank(message = "Sender name is required")
    @Size(max = 50, message = "Sender name must be less than 50 characters")
    private String sender;
    
    @NotBlank(message = "Message content is required")
    @Size(max = 500, message = "Message must be less than 500 characters")
    private String content;
    
    public ChatMessage() {
    }
    
    public ChatMessage(String sender, String content) {
        this.sender = sender;
        this.content = content;
    }

    // Getters and setters
    public String getSender() {
        return sender;
    }

    public void setSender(String sender) {
        this.sender = sender;
    }

    public String getContent() {
        return content;
    }

    public void setContent(String content) {
        this.content = content;
    }
}
```

**`src/main/java/com/example/stompapidocs/messages/ChatResponse.java`**

```java
package com.example.stompapidocs.messages;

import java.time.Instant;

public class ChatResponse {
    private String sender;
    private String content;
    private Instant timestamp;
    
    public ChatResponse() {
        this.timestamp = Instant.now();
    }
    
    public ChatResponse(String sender, String content) {
        this.sender = sender;
        this.content = content;
        this.timestamp = Instant.now();
    }

    // Getters and setters
    public String getSender() {
        return sender;
    }

    public void setSender(String sender) {
        this.sender = sender;
    }

    public String getContent() {
        return content;
    }

    public void setContent(String content) {
        this.content = content;
    }

    public Instant getTimestamp() {
        return timestamp;
    }

    public void setTimestamp(Instant timestamp) {
        this.timestamp = timestamp;
    }
}
```

**`src/main/java/com/example/stompapidocs/messages/PriceUpdate.java`**

```java
package com.example.stompapidocs.messages;

import java.math.BigDecimal;
import java.time.Instant;

public class PriceUpdate {
    private String symbol;
    private BigDecimal price;
    private Instant timestamp;
    
    public PriceUpdate() {
        this.timestamp = Instant.now();
    }
    
    public PriceUpdate(String symbol, BigDecimal price) {
        this.symbol = symbol;
        this.price = price;
        this.timestamp = Instant.now();
    }

    // Getters and setters
    public String getSymbol() {
        return symbol;
    }

    public void setSymbol(String symbol) {
        this.symbol = symbol;
    }

    public BigDecimal getPrice() {
        return price;
    }

    public void setPrice(BigDecimal price) {
        this.price = price;
    }

    public Instant getTimestamp() {
        return timestamp;
    }

    public void setTimestamp(Instant timestamp) {
        this.timestamp = timestamp;
    }
}
```

### Controller with Documented Endpoints

**`src/main/java/com/example/stompapidocs/controller/ChatController.java`**

```java
package com.example.stompapidocs.controller;

import com.example.stompapidocs.messages.ChatMessage;
import com.example.stompapidocs.messages.ChatResponse;
import jakarta.validation.Valid;
import org.springframework.messaging.handler.annotation.MessageMapping;
import org.springframework.messaging.handler.annotation.Payload;
import org.springframework.messaging.handler.annotation.SendTo;
import org.springframework.stereotype.Controller;

@Controller
public class ChatController {

    /**
     * Handles chat messages sent to /app/chat.send
     * Broadcasts the message to all subscribers of /topic/public
     * 
     * @param message The chat message payload
     * @return A chat response that will be sent to /topic/public
     */
    @MessageMapping("/chat.send")
    @SendTo("/topic/public")
    public ChatResponse sendMessage(@Payload @Valid ChatMessage message) {
        return new ChatResponse(message.getSender(), message.getContent());
    }
}
```

**`src/main/java/com/example/stompapidocs/controller/PriceController.java`**

```java
package com.example.stompapidocs.controller;

import com.example.stompapidocs.messages.PriceUpdate;
import org.springframework.messaging.handler.annotation.MessageMapping;
import org.springframework.messaging.handler.annotation.Payload;
import org.springframework.messaging.handler.annotation.SendTo;
import org.springframework.stereotype.Controller;

@Controller
public class PriceController {

    /**
     * Handles one-off price requests sent to /app/prices/request
     * Returns the current price for the requested symbol
     * 
     * @param symbol The stock symbol to query
     * @return A price update sent to /topic/prices
     */
    @MessageMapping("/prices/request")
    @SendTo("/topic/prices")
    public PriceUpdate requestPrice(@Payload String symbol) {
        // Simulate price lookup
        java.math.BigDecimal price = java.math.BigDecimal.valueOf(
            100.0 + Math.random() * 50.0
        ).setScale(2, java.math.RoundingMode.HALF_UP);
        
        return new PriceUpdate(symbol.toUpperCase(), price);
    }
}
```

### Scheduled Price Feed Service

**`src/main/java/com/example/stompapidocs/service/PriceFeedService.java`**

```java
package com.example.stompapidocs.service;

import com.example.stompapidocs.messages.PriceUpdate;
import org.springframework.messaging.simp.SimpMessagingTemplate;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Service;

import java.math.BigDecimal;
import java.math.RoundingMode;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

@Service
public class PriceFeedService {

    private final SimpMessagingTemplate messagingTemplate;
    private final Map<String, BigDecimal> prices = new ConcurrentHashMap<>();
    
    public PriceFeedService(SimpMessagingTemplate messagingTemplate) {
        this.messagingTemplate = messagingTemplate;
        // Initialize some sample prices
        prices.put("ACME", BigDecimal.valueOf(120.50));
        prices.put("GLOBO", BigDecimal.valueOf(98.25));
        prices.put("MEGACORP", BigDecimal.valueOf(75.80));
    }

    /**
     * Publishes price updates every 5 seconds to /topic/prices
     * This is a server-initiated broadcast, not triggered by client messages
     */
    @Scheduled(initialDelay = 2000, fixedRate = 5000)
    public void publishPriceUpdates() {
        prices.forEach((symbol, currentPrice) -> {
            // Simulate price movement
            double change = (Math.random() - 0.5) * 2.0;
            BigDecimal newPrice = currentPrice.add(BigDecimal.valueOf(change))
                .setScale(2, RoundingMode.HALF_UP);
            prices.put(symbol, newPrice);
            
            PriceUpdate update = new PriceUpdate(symbol, newPrice);
            messagingTemplate.convertAndSend("/topic/prices", update);
        });
    }
}
```

Enable scheduling in the main application class:

**`src/main/java/com/example/stompapidocs/StompApiDocsApplication.java`**

```java
package com.example.stompapidocs;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.scheduling.annotation.EnableScheduling;

@SpringBootApplication
@EnableScheduling
public class StompApiDocsApplication {

    public static void main(String[] args) {
        SpringApplication.run(StompApiDocsApplication.class, args);
    }
}
```

## Integrating Documentation Tools

### Option 1: Using Springwolf (Automatic Generation)

Springwolf scans your Spring configuration and annotations to automatically generate AsyncAPI specifications. Configure it in `application.yml`:

**`src/main/resources/application.yml`**

```yaml
springwolf:
  enabled: true
  scanner:
    asyncapi:
      info:
        title: STOMP Chat and Price Service
        version: 1.0.0
        description: A sample STOMP service demonstrating chat and price updates
      servers:
        production:
          url: ws://localhost:8080/ws
          protocol: ws
          description: WebSocket server endpoint
```

Springwolf will automatically:
- Scan `@MessageMapping` annotations to identify application destinations
- Scan `@SendTo` annotations to identify broker destinations
- Extract payload types from method parameters and return types
- Generate AsyncAPI specification at `/springwolf/docs` endpoint
- Provide a UI at `/springwolf-ui` for browsing the documentation

### Option 2: Manual AsyncAPI Specification

If you prefer manual control or Springwolf isn't suitable, you can create an AsyncAPI specification manually. AsyncAPI is a specification format (similar to OpenAPI for REST) designed for asynchronous messaging.

Create `src/main/resources/static/asyncapi.yaml`:

```yaml
asyncapi: '2.6.0'
info:
  title: STOMP Chat and Price Service
  version: 1.0.0
  description: |
    A sample STOMP service demonstrating real-time chat and price updates.
    Clients connect via WebSocket to /ws and use STOMP protocol for messaging.

servers:
  production:
    url: ws://localhost:8080/ws
    protocol: ws
    description: WebSocket server endpoint with SockJS fallback

channels:
  /app/chat.send:
    publish:
      operationId: sendChatMessage
      summary: Send a chat message
      message:
        $ref: '#/components/messages/ChatMessage'
  
  /topic/public:
    subscribe:
      operationId: receiveChatMessages
      summary: Subscribe to public chat messages
      message:
        $ref: '#/components/messages/ChatResponse'
  
  /app/prices/request:
    publish:
      operationId: requestPrice
      summary: Request a one-off price update
      message:
        payload:
          type: string
          example: "ACME"
  
  /topic/prices:
    subscribe:
      operationId: receivePriceUpdates
      summary: Subscribe to price updates
      message:
        $ref: '#/components/messages/PriceUpdate'

components:
  messages:
    ChatMessage:
      payload:
        type: object
        properties:
          sender:
            type: string
            maxLength: 50
            description: The sender's name
          content:
            type: string
            maxLength: 500
            description: The message content
        required:
          - sender
          - content
        examples:
          - sender: "Alice"
            content: "Hello, everyone!"
    
    ChatResponse:
      payload:
        type: object
        properties:
          sender:
            type: string
            description: The sender's name
          content:
            type: string
            description: The message content
          timestamp:
            type: string
            format: date-time
            description: When the message was sent
        required:
          - sender
          - content
          - timestamp
    
    PriceUpdate:
      payload:
        type: object
        properties:
          symbol:
            type: string
            description: Stock symbol (e.g., ACME, GLOBO)
          price:
            type: number
            format: decimal
            description: Current price
          timestamp:
            type: string
            format: date-time
            description: When the price was recorded
        required:
          - symbol
          - price
          - timestamp
```

### Exposing Documentation via REST Endpoint

Create a controller to serve the AsyncAPI specification:

**`src/main/java/com/example/stompapidocs/controller/DocsController.java`**

```java
package com.example.stompapidocs.controller;

import org.springframework.core.io.ClassPathResource;
import org.springframework.core.io.Resource;
import org.springframework.http.MediaType;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

import java.io.IOException;
import java.nio.charset.StandardCharsets;

@RestController
public class DocsController {

    @GetMapping(value = "/api/docs/asyncapi", produces = MediaType.APPLICATION_JSON_VALUE)
    public ResponseEntity<String> getAsyncApiSpec() throws IOException {
        Resource resource = new ClassPathResource("static/asyncapi.yaml");
        String content = new String(resource.getInputStream().readAllBytes(), StandardCharsets.UTF_8);
        return ResponseEntity.ok(content);
    }
}
```

## Client Integration via Generated Documentation

The generated documentation provides a clear contract for client developers. Let's build a client that follows the documented API:

**`src/main/resources/static/index.html`**

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>STOMP API Documentation Demo</title>
    <style>
        body {
            font-family: system-ui, sans-serif;
            margin: 2rem;
            max-width: 1200px;
            margin-left: auto;
            margin-right: auto;
        }
        .container {
            display: grid;
            grid-template-columns: 1fr 1fr;
            gap: 2rem;
            margin-top: 2rem;
        }
        section {
            border: 1px solid #ddd;
            border-radius: 8px;
            padding: 1rem;
        }
        h2 {
            margin-top: 0;
        }
        .status {
            padding: 0.5rem 1rem;
            border-radius: 4px;
            display: inline-block;
            margin-bottom: 1rem;
        }
        .status.connected {
            background: #d4edda;
            color: #155724;
        }
        .status.disconnected {
            background: #f8d7da;
            color: #721c24;
        }
        .messages {
            height: 300px;
            overflow-y: auto;
            border: 1px solid #eee;
            padding: 0.5rem;
            margin-bottom: 1rem;
            background: #fafafa;
        }
        .message {
            margin-bottom: 0.5rem;
            padding: 0.5rem;
            background: white;
            border-radius: 4px;
            border-left: 3px solid #007bff;
        }
        .message-price {
            border-left-color: #28a745;
        }
        form {
            display: flex;
            gap: 0.5rem;
        }
        input[type="text"] {
            flex: 1;
            padding: 0.5rem;
            border: 1px solid #ddd;
            border-radius: 4px;
        }
        button {
            padding: 0.5rem 1rem;
            background: #007bff;
            color: white;
            border: none;
            border-radius: 4px;
            cursor: pointer;
        }
        button:disabled {
            background: #ccc;
            cursor: not-allowed;
        }
        .docs-link {
            margin-top: 1rem;
            padding: 1rem;
            background: #e7f3ff;
            border-radius: 4px;
        }
        .docs-link a {
            color: #0066cc;
            text-decoration: none;
            font-weight: 600;
        }
    </style>
</head>
<body>
    <h1>STOMP API Documentation Demo</h1>
    <p>This client demonstrates integration using the documented STOMP API.</p>
    
    <div>
        <button id="connect-btn">Connect</button>
        <button id="disconnect-btn" disabled>Disconnect</button>
        <span id="status" class="status disconnected">Disconnected</span>
    </div>

    <div class="docs-link">
        <strong>📚 API Documentation:</strong>
        <a href="/api/docs/asyncapi" target="_blank">View AsyncAPI Specification (JSON)</a> |
        <a href="/springwolf-ui" target="_blank">View Springwolf UI</a> (if Springwolf is enabled)
    </div>

    <div class="container">
        <section>
            <h2>Chat Messages</h2>
            <p><small>Destination: <code>/app/chat.send</code> → <code>/topic/public</code></small></p>
            <form id="chat-form">
                <input type="text" id="sender" placeholder="Your name" required>
                <input type="text" id="message" placeholder="Message" required>
                <button type="submit">Send</button>
            </form>
            <div id="chat-messages" class="messages"></div>
        </section>

        <section>
            <h2>Price Updates</h2>
            <p><small>Subscribe to: <code>/topic/prices</code></small></p>
            <form id="price-form">
                <input type="text" id="symbol" placeholder="Symbol (e.g., ACME)" required>
                <button type="submit">Request Price</button>
            </form>
            <div id="price-updates" class="messages"></div>
        </section>
    </div>

    <script src="https://cdn.jsdelivr.net/npm/sockjs-client@1/dist/sockjs.min.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/stompjs@2.3.3/lib/stomp.min.js"></script>
    <script>
        let stompClient = null;
        const connectBtn = document.getElementById('connect-btn');
        const disconnectBtn = document.getElementById('disconnect-btn');
        const statusEl = document.getElementById('status');
        const chatMessagesEl = document.getElementById('chat-messages');
        const priceUpdatesEl = document.getElementById('price-updates');
        const chatForm = document.getElementById('chat-form');
        const priceForm = document.getElementById('price-form');

        function setConnected(connected) {
            connectBtn.disabled = connected;
            disconnectBtn.disabled = !connected;
            statusEl.textContent = connected ? 'Connected' : 'Disconnected';
            statusEl.className = `status ${connected ? 'connected' : 'disconnected'}`;
        }

        function connect() {
            // Connect to the documented endpoint: /ws
            const socket = new SockJS('/ws');
            stompClient = Stomp.over(socket);
            stompClient.debug = null;

            stompClient.connect({}, function(frame) {
                console.log('Connected: ' + frame);
                setConnected(true);

                // Subscribe to documented destinations
                // /topic/public - receives chat messages
                stompClient.subscribe('/topic/public', function(message) {
                    const response = JSON.parse(message.body);
                    appendChatMessage(response);
                });

                // /topic/prices - receives price updates
                stompClient.subscribe('/topic/prices', function(message) {
                    const update = JSON.parse(message.body);
                    appendPriceUpdate(update);
                });

                console.log('Subscribed to /topic/public and /topic/prices');
            }, function(error) {
                console.error('STOMP error: ' + error);
                setConnected(false);
            });
        }

        function disconnect() {
            if (stompClient !== null) {
                stompClient.disconnect(function() {
                    setConnected(false);
                    console.log('Disconnected');
                });
            }
        }

        function sendChatMessage(event) {
            event.preventDefault();
            const sender = document.getElementById('sender').value.trim();
            const content = document.getElementById('message').value.trim();

            if (!sender || !content || !stompClient || !stompClient.connected) {
                return;
            }

            // Send to documented destination: /app/chat.send
            // Payload matches ChatMessage schema from documentation
            const message = {
                sender: sender,
                content: content
            };

            stompClient.send('/app/chat.send', {}, JSON.stringify(message));
            document.getElementById('message').value = '';
        }

        function requestPrice(event) {
            event.preventDefault();
            const symbol = document.getElementById('symbol').value.trim();

            if (!symbol || !stompClient || !stompClient.connected) {
                return;
            }

            // Send to documented destination: /app/prices/request
            // Payload is a simple string (symbol)
            stompClient.send('/app/prices/request', {}, symbol);
            document.getElementById('symbol').value = '';
        }

        function appendChatMessage(response) {
            const div = document.createElement('div');
            div.className = 'message';
            div.innerHTML = `
                <strong>${escapeHtml(response.sender)}</strong>: ${escapeHtml(response.content)}
                <br><small>${new Date(response.timestamp).toLocaleString()}</small>
            `;
            chatMessagesEl.appendChild(div);
            chatMessagesEl.scrollTop = chatMessagesEl.scrollHeight;
        }

        function appendPriceUpdate(update) {
            const div = document.createElement('div');
            div.className = 'message message-price';
            div.innerHTML = `
                <strong>${escapeHtml(update.symbol)}</strong>: $${Number(update.price).toFixed(2)}
                <br><small>${new Date(update.timestamp).toLocaleString()}</small>
            `;
            priceUpdatesEl.appendChild(div);
            priceUpdatesEl.scrollTop = priceUpdatesEl.scrollHeight;
        }

        function escapeHtml(text) {
            const div = document.createElement('div');
            div.textContent = text;
            return div.innerHTML;
        }

        connectBtn.addEventListener('click', connect);
        disconnectBtn.addEventListener('click', disconnect);
        chatForm.addEventListener('submit', sendChatMessage);
        priceForm.addEventListener('submit', requestPrice);
    </script>
</body>
</html>
```

## Testing the Documentation

1. **Start the application:**
   ```bash
   ./mvnw spring-boot:run
   ```

2. **Access the documentation:**
   - Manual AsyncAPI spec: `http://localhost:8080/api/docs/asyncapi`
   - Springwolf UI (if enabled): `http://localhost:8080/springwolf-ui`
   - Client demo: `http://localhost:8080/`

3. **Test the client:**
   - Connect to the WebSocket
   - Send chat messages and observe they appear in the chat section
   - Request prices and observe updates in the price section
   - Open multiple browser tabs to see broadcast behavior

4. **Verify documentation accuracy:**
   - Compare the documented destinations with actual controller mappings
   - Verify payload schemas match the Java classes
   - Test edge cases (empty messages, invalid symbols) to see if validation matches documentation

## Integrating Documentation into Your Build Process

### Maven Plugin for AsyncAPI Generation

You can integrate AsyncAPI generation into your Maven build:

```xml
<plugin>
    <groupId>com.asyncapi</groupId>
    <artifactId>asyncapi-generator-maven-plugin</artifactId>
    <version>1.0.0</version>
    <configuration>
        <specFile>${project.basedir}/src/main/resources/static/asyncapi.yaml</specFile>
        <outputDir>${project.build.directory}/asyncapi</outputDir>
        <generator>html</generator>
    </configuration>
    <executions>
        <execution>
            <phase>generate-resources</phase>
            <goals>
                <goal>generate</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

This generates HTML documentation during the build process, which can be deployed alongside your application.

## Best Practices for STOMP API Documentation

1. **Keep Documentation in Sync**: Update documentation when you change destinations, payloads, or message flow. Consider automated checks in CI/CD.

2. **Document Message Semantics**: Don't just list destinations—explain what triggers messages, expected frequency, and business logic.

3. **Include Examples**: Provide working examples of message payloads and client code snippets.

4. **Version Your API**: Use versioning in your AsyncAPI spec to track breaking changes.

5. **Document Error Handling**: Specify what happens when validation fails, destinations don't exist, or connections drop.

6. **Security Documentation**: Document authentication requirements, authorization rules, and secure connection practices.

## Troubleshooting

| Issue | Likely Cause | Solution |
|-------|--------------|----------|
| Documentation not generated | Springwolf not configured correctly | Check dependencies and configuration in `application.yml` |
| Missing destinations in docs | Annotations not scanned | Ensure `@MessageMapping` methods are in `@Controller` classes |
| Payload schemas incorrect | Type information not available | Ensure DTOs have proper getters/setters and Jackson annotations |
| Client can't connect | Endpoint mismatch | Verify WebSocket endpoint in documentation matches `registerStompEndpoints` |

## Looking Ahead

Documentation is an ongoing process. As you add features:
- Update AsyncAPI specifications
- Add new message types
- Document complex workflows
- Provide integration guides for client developers

Consider integrating documentation generation into your CI/CD pipeline to ensure it stays current with code changes.

## Recap

You've learned:
- ✅ Why documenting asynchronous STOMP services is critical
- ✅ Key elements to document: destinations, payloads, message flow
- ✅ How to integrate automatic documentation generation tools
- ✅ How to create manual AsyncAPI specifications
- ✅ How generated documentation guides client development

The key insight is that documentation for asynchronous services isn't optional—it's essential for integration, maintenance, and team collaboration. Automated tools can help, but the most important step is starting to document your API and keeping it current.

---

**Next Steps**: Use the generated documentation to onboard new team members, create integration tests, and build client libraries. Consider setting up automated documentation generation in your CI/CD pipeline to ensure it stays synchronized with your codebase.
