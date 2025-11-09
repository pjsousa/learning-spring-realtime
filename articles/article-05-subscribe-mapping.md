# Fast Startup: Using @SubscribeMapping for Initial Data Loading and Setup

*Estimated reading time: 20 minutes · Target word count: ~1,800*

## Overview

While `@MessageMapping` handles client-initiated commands that often broadcast responses to all subscribers, this article introduces the specialized `@SubscribeMapping` annotation. We explore a scenario where a client requires an immediate, one-time data payload upon subscribing, such as loading an application's initial configuration, recent message history, or current application state. Unlike `@MessageMapping`, which defaults to broadcasting via the broker, the return value of an `@SubscribeMapping` method is sent directly back to the requesting client via the `clientOutboundChannel`. This is ideal for optimizing application startup, allowing the client to request data immediately upon connection without establishing an unnecessary long-term subscription for that specific payload.

By the end of this article, you'll understand when and how to use `@SubscribeMapping` to streamline client initialization, reduce latency, and eliminate unnecessary broadcast overhead for one-time data transfers.

## Learning Goals

- Understand the difference between `@MessageMapping` and `@SubscribeMapping` and when to use each.
- Implement `@SubscribeMapping` handlers that return data directly to subscribing clients.
- Recognize use cases where request/reply on subscription is more efficient than broadcast patterns.
- Build a Spring Boot service that loads initial state (configuration, history, current data) when clients subscribe.
- Create a browser client that receives and processes subscription-time payloads.
- Grasp how Spring routes subscription responses through the `clientOutboundChannel` instead of the broker.

## Why @SubscribeMapping Exists

Consider a typical real-time application startup flow:

1. **Client connects** to the WebSocket endpoint.
2. **Client subscribes** to `/topic/live-updates` to receive ongoing broadcasts.
3. **Client needs initial data**: recent messages, current game state, user preferences, etc.

Without `@SubscribeMapping`, you might:
- Send a `SEND` frame to `/app/loadInitialData`, which broadcasts the result (wasteful if only one client needs it).
- Use a separate REST endpoint before connecting (adds complexity and round trips).
- Subscribe to a separate topic and unsubscribe after the first message (awkward and error-prone).

`@SubscribeMapping` solves this elegantly: when a client subscribes to a destination mapped to an `@SubscribeMapping` method, Spring:
1. Executes the method immediately.
2. Sends the return value **directly to that specific client** via `clientOutboundChannel`.
3. Does not broadcast the response through the broker.
4. Completes the subscription normally (the client remains subscribed for future messages if needed).

This creates a clean request/reply pattern tied to the subscription event itself.

## Prerequisites and Project Setup

This article builds on the foundational STOMP concepts from Articles 1–4. If you're starting fresh, ensure you have:

- Java 17+ installed.
- Maven 3.9+ or Gradle 8+.
- A Spring Boot 3.2+ project with WebSocket support.

### Generate the Project

Create a new Spring Boot project or extend an existing one. For Maven users:

```bash
curl https://start.spring.io/starter.zip \
  -d dependencies=websocket \
  -d name=subscribe-mapping-demo \
  -d javaVersion=21 \
  -d type=maven-project \
  -d packageName=com.example.subscribemapping \
  -o subscribe-mapping-demo.zip

unzip subscribe-mapping-demo.zip
cd subscribe-mapping-demo
```

Your `pom.xml` should include:

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

## Step 1: Configure the WebSocket Broker

We'll use the standard STOMP configuration, identical to previous articles but tuned for our subscription-based use case.

Create `src/main/java/com/example/subscribemapping/config/WebSocketConfig.java`:

```java
package com.example.subscribemapping.config;

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
- Simple broker for `/topic` and `/queue` destinations.
- Application destination prefix `/app` for client-to-server messages.
- WebSocket endpoint at `/ws` with SockJS fallback support.

## Step 2: Define Initialization Data Models

We'll demonstrate `@SubscribeMapping` with a practical scenario: a chat application that loads recent message history when a client subscribes, plus an application configuration payload.

Create `src/main/java/com/example/subscribemapping/model/ChatMessage.java`:

```java
package com.example.subscribemapping.model;

import java.time.Instant;

public class ChatMessage {

  private String sender;
  private String content;
  private Instant timestamp;

  public ChatMessage() {
  }

  public ChatMessage(String sender, String content, Instant timestamp) {
    this.sender = sender;
    this.content = content;
    this.timestamp = timestamp;
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

Create `src/main/java/com/example/subscribemapping/model/MessageHistory.java`:

```java
package com.example.subscribemapping.model;

import java.util.List;

public class MessageHistory {

  private List<ChatMessage> messages;
  private int totalCount;
  private String message;

  public MessageHistory() {
  }

  public MessageHistory(List<ChatMessage> messages, int totalCount, String message) {
    this.messages = messages;
    this.totalCount = totalCount;
    this.message = message;
  }

  // Getters and setters
  public List<ChatMessage> getMessages() {
    return messages;
  }

  public void setMessages(List<ChatMessage> messages) {
    this.messages = messages;
  }

  public int getTotalCount() {
    return totalCount;
  }

  public void setTotalCount(int totalCount) {
    this.totalCount = totalCount;
  }

  public String getMessage() {
    return message;
  }

  public void setMessage(String message) {
    this.message = message;
  }
}
```

Create `src/main/java/com/example/subscribemapping/model/AppConfig.java`:

```java
package com.example.subscribemapping.model;

public class AppConfig {

  private String appName;
  private String version;
  private int maxMessageLength;
  private boolean featuresEnabled;

  public AppConfig() {
  }

  public AppConfig(String appName, String version, int maxMessageLength, boolean featuresEnabled) {
    this.appName = appName;
    this.version = version;
    this.maxMessageLength = maxMessageLength;
    this.featuresEnabled = featuresEnabled;
  }

  // Getters and setters
  public String getAppName() {
    return appName;
  }

  public void setAppName(String appName) {
    this.appName = appName;
  }

  public String getVersion() {
    return version;
  }

  public void setVersion(String version) {
    this.version = version;
  }

  public int getMaxMessageLength() {
    return maxMessageLength;
  }

  public void setMaxMessageLength(int maxMessageLength) {
    this.maxMessageLength = maxMessageLength;
  }

  public boolean isFeaturesEnabled() {
    return featuresEnabled;
  }

  public void setFeaturesEnabled(boolean featuresEnabled) {
    this.featuresEnabled = featuresEnabled;
  }
}
```

## Step 3: Create a Simple Message Store

To simulate a real application with historical data, we'll maintain an in-memory message store. In production, this would be backed by a database.

Create `src/main/java/com/example/subscribemapping/service/MessageStore.java`:

```java
package com.example.subscribemapping.service;

import com.example.subscribemapping.model.ChatMessage;
import org.springframework.stereotype.Service;

import java.time.Instant;
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.ConcurrentLinkedQueue;

@Service
public class MessageStore {

  private final ConcurrentLinkedQueue<ChatMessage> messages = new ConcurrentLinkedQueue<>();

  public MessageStore() {
    // Initialize with some sample messages
    messages.add(new ChatMessage("System", "Welcome to the chat!", Instant.now().minusSeconds(3600)));
    messages.add(new ChatMessage("Alice", "Hello everyone!", Instant.now().minusSeconds(1800)));
    messages.add(new ChatMessage("Bob", "Great to be here!", Instant.now().minusSeconds(1200)));
    messages.add(new ChatMessage("Charlie", "This is a test message.", Instant.now().minusSeconds(600)));
  }

  public void addMessage(ChatMessage message) {
    messages.add(message);
    // Keep only last 100 messages in memory
    while (messages.size() > 100) {
      messages.poll();
    }
  }

  public List<ChatMessage> getRecentMessages(int limit) {
    List<ChatMessage> recent = new ArrayList<>();
    ChatMessage[] allMessages = messages.toArray(new ChatMessage[0]);
    
    int startIndex = Math.max(0, allMessages.length - limit);
    for (int i = startIndex; i < allMessages.length; i++) {
      recent.add(allMessages[i]);
    }
    
    return recent;
  }

  public int getTotalCount() {
    return messages.size();
  }
}
```

## Step 4: Implement @SubscribeMapping Handlers

Now we'll create controllers that use `@SubscribeMapping` to return initial data directly to subscribing clients.

Create `src/main/java/com/example/subscribemapping/controller/InitializationController.java`:

```java
package com.example.subscribemapping.controller;

import com.example.subscribemapping.model.AppConfig;
import com.example.subscribemapping.model.MessageHistory;
import com.example.subscribemapping.service.MessageStore;
import org.springframework.messaging.handler.annotation.SubscribeMapping;
import org.springframework.stereotype.Controller;

@Controller
public class InitializationController {

  private final MessageStore messageStore;

  public InitializationController(MessageStore messageStore) {
    this.messageStore = messageStore;
  }

  @SubscribeMapping("/topic/history")
  public MessageHistory loadMessageHistory() {
    // This method is called when a client subscribes to /topic/history
    // The return value is sent ONLY to that subscribing client
    List<ChatMessage> recent = messageStore.getRecentMessages(20);
    
    return new MessageHistory(
        recent,
        messageStore.getTotalCount(),
        "Loaded " + recent.size() + " recent messages"
    );
  }

  @SubscribeMapping("/topic/config")
  public AppConfig loadAppConfig() {
    // Return application configuration directly to the subscribing client
    return new AppConfig(
        "STOMP Chat Demo",
        "1.0.0",
        500,
        true
    );
  }
}
```

**Key observations:**

1. **`@SubscribeMapping("/topic/history")`**: When a client subscribes to `/topic/history`, this method executes. The return value is sent directly to that client—not broadcast.
2. **Direct reply**: The response travels via `clientOutboundChannel` instead of `brokerChannel`, ensuring it reaches only the subscribing client.
3. **No @SendTo needed**: Unlike `@MessageMapping`, `@SubscribeMapping` doesn't require `@SendTo` because it doesn't use the broker for delivery.

### Combining SubscribeMapping with Ongoing Subscriptions

A common pattern is to subscribe to both an initialization destination and a live updates destination:

- Client subscribes to `/topic/history` → receives initial history via `@SubscribeMapping`.
- Client subscribes to `/topic/live-messages` → receives ongoing broadcasts via normal broker subscriptions.

We'll add a regular `@MessageMapping` for live messages:

Create `src/main/java/com/example/subscribemapping/controller/ChatController.java`:

```java
package com.example.subscribemapping.controller;

import com.example.subscribemapping.model.ChatMessage;
import com.example.subscribemapping.service.MessageStore;
import org.springframework.messaging.handler.annotation.MessageMapping;
import org.springframework.messaging.handler.annotation.Payload;
import org.springframework.messaging.handler.annotation.SendTo;
import org.springframework.stereotype.Controller;

import java.time.Instant;

@Controller
public class ChatController {

  private final MessageStore messageStore;

  public ChatController(MessageStore messageStore) {
    this.messageStore = messageStore;
  }

  @MessageMapping("/chat.send")
  @SendTo("/topic/live-messages")
  public ChatMessage sendMessage(@Payload ChatMessage message) {
    // Set timestamp server-side for consistency
    message.setTimestamp(Instant.now());
    
    // Store the message
    messageStore.addMessage(message);
    
    // Broadcast to all subscribers via @SendTo
    return message;
  }
}
```

## Step 5: Understanding the Message Flow

Let's trace what happens when a client subscribes:

**Scenario: Client subscribes to `/topic/history`**

1. **Client sends SUBSCRIBE frame**: `SUBSCRIBE id:sub-1 destination:/topic/history`
2. **Spring routes to `@SubscribeMapping`**: Finds the handler method `loadMessageHistory()`.
3. **Method executes**: Retrieves recent messages from the store.
4. **Response routing**: Instead of going to `brokerChannel`, the return value is sent via `clientOutboundChannel` directly to the client that issued the subscription.
5. **Client receives**: The client gets a `MESSAGE` frame with the history payload, addressed specifically to subscription `sub-1`.
6. **Subscription remains active**: The client stays subscribed to `/topic/history`; if something else publishes there, the client will receive those messages too.

**Contrast with `@MessageMapping`:**

- `@MessageMapping` + `@SendTo`: Client sends a `SEND` frame → method executes → return value goes to `brokerChannel` → broker broadcasts to all subscribers.
- `@SubscribeMapping`: Client sends a `SUBSCRIBE` frame → method executes → return value goes to `clientOutboundChannel` → sent directly to the requesting client only.

## Step 6: Build the Browser Client

Create a client that demonstrates both subscription-time initialization and ongoing live updates.

Create `src/main/resources/static/index.html`:

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>@SubscribeMapping Demo</title>
  <style>
    body {
      font-family: system-ui, sans-serif;
      margin: 2rem;
      max-width: 1000px;
      margin-left: auto;
      margin-right: auto;
    }
    header {
      margin-bottom: 2rem;
      padding-bottom: 1rem;
      border-bottom: 2px solid #e0e0e0;
    }
    .status {
      padding: 0.5rem 1rem;
      border-radius: 4px;
      font-weight: 600;
      display: inline-block;
      margin-left: 1rem;
    }
    .status.connected {
      background: #d4edda;
      color: #155724;
    }
    .status.disconnected {
      background: #f8d7da;
      color: #721c24;
    }
    .container {
      display: grid;
      grid-template-columns: 1fr 1fr;
      gap: 2rem;
      margin-bottom: 2rem;
    }
    section {
      border: 1px solid #ddd;
      border-radius: 8px;
      padding: 1rem;
      background: #fafafa;
    }
    h2 {
      margin-top: 0;
      font-size: 1.25rem;
      color: #333;
    }
    .messages {
      height: 300px;
      overflow-y: auto;
      border: 1px solid #eee;
      padding: 0.5rem;
      margin-bottom: 1rem;
      background: white;
      border-radius: 4px;
    }
    .message {
      margin-bottom: 0.75rem;
      padding: 0.5rem;
      background: #f8f9fa;
      border-radius: 4px;
      border-left: 3px solid #007bff;
    }
    .message.history {
      border-left-color: #28a745;
      font-style: italic;
      opacity: 0.8;
    }
    .message-sender {
      font-weight: 600;
      color: #007bff;
      margin-right: 0.5rem;
    }
    .message-content {
      margin-top: 0.25rem;
    }
    .message-timestamp {
      font-size: 0.75rem;
      color: #666;
      margin-top: 0.25rem;
    }
    .config {
      background: #fff3cd;
      border: 1px solid #ffc107;
      border-radius: 4px;
      padding: 1rem;
      margin-bottom: 1rem;
    }
    .config-item {
      margin: 0.5rem 0;
    }
    .config-label {
      font-weight: 600;
      color: #856404;
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
      font-weight: 600;
    }
    button:disabled {
      background: #ccc;
      cursor: not-allowed;
    }
    button.secondary {
      background: #6c757d;
    }
    .info {
      background: #d1ecf1;
      border: 1px solid #bee5eb;
      border-radius: 4px;
      padding: 1rem;
      margin-bottom: 1rem;
      color: #0c5460;
    }
  </style>
</head>
<body>
  <header>
    <h1>@SubscribeMapping Demo: Fast Initialization</h1>
    <div>
      <button id="connect-btn">Connect</button>
      <button id="disconnect-btn" disabled>Disconnect</button>
      <span id="status" class="status disconnected">Disconnected</span>
    </div>
  </header>

  <div class="info">
    <strong>How it works:</strong> When you connect, the client subscribes to <code>/topic/history</code> 
    and <code>/topic/config</code>. These subscriptions trigger <code>@SubscribeMapping</code> methods 
    that send initial data directly to this client—no broadcast, no REST calls needed!
  </div>

  <div class="container">
    <section>
      <h2>Application Configuration</h2>
      <div id="config" class="config">
        <p><em>Configuration will load when you subscribe...</em></p>
      </div>
    </section>

    <section>
      <h2>Message History</h2>
      <div id="history-info"></div>
      <div id="history" class="messages"></div>
    </section>
  </div>

  <section>
    <h2>Live Chat Messages</h2>
    <div id="live-messages" class="messages"></div>
    <form id="message-form">
      <input type="text" id="sender-input" placeholder="Your name" required>
      <input type="text" id="message-input" placeholder="Type a message..." required>
      <button type="submit">Send</button>
    </form>
  </section>

  <script src="https://cdn.jsdelivr.net/npm/sockjs-client@1/dist/sockjs.min.js"></script>
  <script src="https://cdn.jsdelivr.net/npm/stompjs@2.3.3/lib/stomp.min.js"></script>
  <script>
    let stompClient = null;
    let historySubscription = null;
    let configSubscription = null;
    let liveSubscription = null;

    const connectBtn = document.getElementById('connect-btn');
    const disconnectBtn = document.getElementById('disconnect-btn');
    const statusEl = document.getElementById('status');
    const configEl = document.getElementById('config');
    const historyEl = document.getElementById('history');
    const historyInfoEl = document.getElementById('history-info');
    const liveMessagesEl = document.getElementById('live-messages');
    const messageForm = document.getElementById('message-form');
    const senderInput = document.getElementById('sender-input');
    const messageInput = document.getElementById('message-input');

    function setConnected(connected) {
      connectBtn.disabled = connected;
      disconnectBtn.disabled = !connected;
      statusEl.textContent = connected ? 'Connected' : 'Disconnected';
      statusEl.className = `status ${connected ? 'connected' : 'disconnected'}`;
    }

    function connect() {
      const socket = new SockJS('/ws');
      stompClient = Stomp.over(socket);
      stompClient.debug = null; // Suppress console logs

      stompClient.connect({}, function(frame) {
        console.log('Connected: ' + frame);
        setConnected(true);

        // Subscribe to history - @SubscribeMapping will trigger
        historySubscription = stompClient.subscribe('/topic/history', function(message) {
          const history = JSON.parse(message.body);
          console.log('Received history via @SubscribeMapping:', history);
          
          historyInfoEl.innerHTML = `<p><strong>${history.message}</strong> (Total: ${history.totalCount} messages)</p>`;
          
          // Render historical messages
          historyEl.innerHTML = '';
          if (history.messages && history.messages.length > 0) {
            history.messages.forEach(msg => {
              appendMessage(msg, historyEl, true);
            });
          } else {
            historyEl.innerHTML = '<p><em>No message history available.</em></p>';
          }
        });

        // Subscribe to config - @SubscribeMapping will trigger
        configSubscription = stompClient.subscribe('/topic/config', function(message) {
          const config = JSON.parse(message.body);
          console.log('Received config via @SubscribeMapping:', config);
          
          configEl.innerHTML = `
            <div class="config-item">
              <span class="config-label">App Name:</span> ${config.appName}
            </div>
            <div class="config-item">
              <span class="config-label">Version:</span> ${config.version}
            </div>
            <div class="config-item">
              <span class="config-label">Max Message Length:</span> ${config.maxMessageLength} characters
            </div>
            <div class="config-item">
              <span class="config-label">Features Enabled:</span> ${config.featuresEnabled ? 'Yes' : 'No'}
            </div>
          `;
        });

        // Subscribe to live messages - normal broker subscription
        liveSubscription = stompClient.subscribe('/topic/live-messages', function(message) {
          const msg = JSON.parse(message.body);
          console.log('Received live message:', msg);
          appendMessage(msg, liveMessagesEl, false);
        });

        console.log('Subscribed to /topic/history, /topic/config, and /topic/live-messages');
      }, function(error) {
        console.error('STOMP error: ' + error);
        setConnected(false);
      });
    }

    function disconnect() {
      if (stompClient !== null) {
        if (historySubscription) {
          historySubscription.unsubscribe();
          historySubscription = null;
        }
        if (configSubscription) {
          configSubscription.unsubscribe();
          configSubscription = null;
        }
        if (liveSubscription) {
          liveSubscription.unsubscribe();
          liveSubscription = null;
        }
        
        stompClient.disconnect(function() {
          setConnected(false);
          console.log('Disconnected');
          
          // Clear UI
          configEl.innerHTML = '<p><em>Configuration will load when you subscribe...</em></p>';
          historyEl.innerHTML = '';
          historyInfoEl.innerHTML = '';
          liveMessagesEl.innerHTML = '';
        });
      }
    }

    function sendMessage(event) {
      event.preventDefault();
      const sender = senderInput.value.trim();
      const content = messageInput.value.trim();

      if (!sender || !content || !stompClient || !stompClient.connected) {
        return;
      }

      const message = {
        sender: sender,
        content: content
      };

      stompClient.send('/app/chat.send', {}, JSON.stringify(message));
      messageInput.value = '';
      messageInput.focus();
    }

    function appendMessage(msg, container, isHistory) {
      const messageDiv = document.createElement('div');
      messageDiv.className = `message ${isHistory ? 'history' : ''}`;

      const senderSpan = document.createElement('span');
      senderSpan.className = 'message-sender';
      senderSpan.textContent = msg.sender + ':';
      messageDiv.appendChild(senderSpan);

      const contentSpan = document.createElement('span');
      contentSpan.className = 'message-content';
      contentSpan.textContent = msg.content;
      messageDiv.appendChild(contentSpan);

      if (msg.timestamp) {
        const timestampDiv = document.createElement('div');
        timestampDiv.className = 'message-timestamp';
        timestampDiv.textContent = new Date(msg.timestamp).toLocaleString();
        messageDiv.appendChild(timestampDiv);
      }

      container.appendChild(messageDiv);
      container.scrollTop = container.scrollHeight;
    }

    connectBtn.addEventListener('click', connect);
    disconnectBtn.addEventListener('click', disconnect);
    messageForm.addEventListener('submit', sendMessage);
  </script>
</body>
</html>
```

**Client behavior highlights:**

1. **Subscription triggers initialization**: When the client subscribes to `/topic/history` and `/topic/config`, the `@SubscribeMapping` methods execute and send data directly back.
2. **Direct replies**: The client receives the history and config immediately without any `SEND` frames or REST calls.
3. **Ongoing subscriptions**: After initialization, the client remains subscribed and can receive future messages on those destinations if published.
4. **Separate live feed**: The client also subscribes to `/topic/live-messages` for ongoing broadcasts, demonstrating the pattern of combining initialization with live updates.

## Step 7: Running and Testing

### Start the Application

```bash
./mvnw spring-boot:run
```

### Test the Implementation

1. **Open the browser**: Navigate to `http://localhost:8080/`
2. **Click "Connect"**: Watch the browser console—you should see:
   - Connection established
   - History subscription created
   - Config subscription created
   - Live messages subscription created
   - History payload received (from `@SubscribeMapping`)
   - Config payload received (from `@SubscribeMapping`)
3. **Observe the UI**: 
   - The "Application Configuration" section populates immediately.
   - The "Message History" section shows the pre-loaded messages.
   - Both happen without any explicit `SEND` frames from the client.
4. **Send a live message**: Enter your name and a message, then click "Send". The message appears in the "Live Chat Messages" section and is broadcast to all connected clients (including other browser tabs).
5. **Open multiple tabs**: Connect in multiple tabs. Each tab receives its own initialization data via `@SubscribeMapping`, but live messages are broadcast to all tabs.

### Verification Points

- ✅ History loads immediately upon subscription (no client `SEND` required).
- ✅ Config loads immediately upon subscription.
- ✅ Initial data goes only to the subscribing client (not broadcast).
- ✅ Live messages broadcast to all subscribers as expected.
- ✅ Each new connection gets its own initialization payload.

## Understanding the Differences: @MessageMapping vs @SubscribeMapping

| Aspect | @MessageMapping | @SubscribeMapping |
|--------|----------------|-------------------|
| **Trigger** | Client sends a `SEND` frame | Client sends a `SUBSCRIBE` frame |
| **Response routing** | Goes to `brokerChannel` (broadcast by default) | Goes to `clientOutboundChannel` (direct reply) |
| **Use case** | Commands, actions, broadcasts | Initialization, one-time data loading |
| **@SendTo needed?** | Yes, for broadcasting | No, reply is automatic |
| **Receiving clients** | All subscribers to the destination | Only the subscribing client |
| **Common pattern** | Request → Process → Broadcast | Subscribe → Receive initial state |

## Advanced Patterns

### Conditional Initialization

You might want to customize the initialization payload based on the client:

```java
@SubscribeMapping("/topic/history")
public MessageHistory loadMessageHistory(SimpMessageHeaderAccessor headerAccessor) {
  String sessionId = headerAccessor.getSessionId();
  // Customize based on session or user
  Principal user = headerAccessor.getUser();
  int limit = (user != null && user.getName().equals("admin")) ? 100 : 20;
  
  return new MessageHistory(
      messageStore.getRecentMessages(limit),
      messageStore.getTotalCount(),
      "Loaded " + limit + " recent messages"
  );
}
```

### Error Handling

If initialization fails, you can throw an exception, which Spring converts to an ERROR frame:

```java
@SubscribeMapping("/topic/history")
public MessageHistory loadMessageHistory() {
  try {
    return new MessageHistory(/* ... */);
  } catch (Exception e) {
    throw new MessagingException("Failed to load history", e);
  }
}
```

The client should handle ERROR frames in its subscription callback.

### Combining with @SendTo

While unusual, you can combine `@SubscribeMapping` with `@SendTo` if you want both a direct reply and a broadcast:

```java
@SubscribeMapping("/topic/announce")
@SendTo("/topic/announcements")
public String announceSubscribe() {
  return "A new client has subscribed!";
}
```

This sends the message both to the subscribing client and broadcasts it to all subscribers of `/topic/announcements`. Use this pattern sparingly—it's usually clearer to separate initialization from broadcasts.

## Troubleshooting

| Issue | Likely Cause | Solution |
|-------|--------------|----------|
| No initialization data received | Method not mapped to subscription destination | Verify `@SubscribeMapping` path matches client subscription |
| Data broadcast to all clients | Using `@SendTo` with `@SubscribeMapping` | Remove `@SendTo`; `@SubscribeMapping` replies directly |
| Subscription works but no reply | Method returns `null` or `void` | Ensure method returns a value that can be serialized |
| Multiple replies received | Multiple subscriptions to same destination | Ensure client subscribes only once, or unsubscribe before resubscribing |

## Looking Ahead

`@SubscribeMapping` is perfect for:
- **Application configuration**: Feature flags, limits, settings
- **Recent history**: Chat history, activity logs, recent events
- **Current state**: Game board state, document contents, dashboard data
- **User-specific initialization**: User preferences, recent notifications

In future articles, we'll combine this with user destinations (`/user/queue/...`) for personalized initialization, and explore how to cache initialization data for performance at scale.

## Recap

You've implemented:

- ✅ `@SubscribeMapping` handlers for request/reply on subscription
- ✅ Direct client replies via `clientOutboundChannel`
- ✅ Initial data loading without REST endpoints or broadcast overhead
- ✅ Combined initialization subscriptions with ongoing live subscriptions
- ✅ A complete browser client demonstrating the pattern

The key insight is that `@SubscribeMapping` optimizes client startup by tying initialization to the subscription event itself, creating a clean, efficient pattern for loading initial state in real-time applications.

---

**Test your implementation**: Run the app, connect the client, and observe how history and configuration load immediately upon subscription—no extra round trips or unnecessary broadcasts required!