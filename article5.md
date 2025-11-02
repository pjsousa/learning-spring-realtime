# Fast Startup: Using @SubscribeMapping for Initial Data Loading and Setup

## Introduction

In our chat application from Article 3, when a user joins, they see only new messages that arrive after their connection. But what if they want to see recent chat history? Or when building a dashboard, users expect to see current data immediately upon loading, not wait for the next scheduled update. Similarly, collaborative editing applications need to load the current document state when a user joins.

The `@SubscribeMapping` annotation solves this problem elegantly. Unlike `@MessageMapping` (which handles client SEND operations) or scheduled broadcasts (which push to all subscribers), `@SubscribeMapping` sends data directly to a specific client when they subscribe to a topic—exactly once, right when they need it.

In this article, we'll explore `@SubscribeMapping` in depth, understand when and why to use it, and build a practical example that loads initial application state. We'll enhance our chat application to include message history and create a dashboard that loads current data on connection.

## Understanding @SubscribeMapping

Let's clarify how `@SubscribeMapping` differs from other approaches:

**@MessageMapping**: Handles messages sent from clients (SEND frames). The return value is broadcast to all subscribers via the broker.

**Scheduled Broadcasting**: Server pushes messages to all subscribers at regular intervals.

**@SubscribeMapping**: Sends data directly to the requesting client when they subscribe (SUBSCRIBE frame). The return value goes directly to that client, not through the broker to all subscribers.

The key difference is the **message flow**:

- **@MessageMapping**: Client → Server → All Subscribers
- **Scheduled Broadcasting**: Server → All Subscribers
- **@SubscribeMapping**: Client Subscription → Server → That Specific Client Only

## When to Use @SubscribeMapping

Use `@SubscribeMapping` when:

1. **Loading Initial State**: User needs current application state when connecting
2. **One-Time Data Transfer**: Send data once without creating a long-term subscription
3. **Reducing Bandwidth**: Avoid broadcasting historical data to all clients
4. **Optimizing Startup**: Get users productive faster by loading essential data immediately

Don't use `@SubscribeMapping` for:
- Regular ongoing updates (use `@SendTo` or scheduled broadcasting)
- Data that changes frequently and should be pushed to all clients simultaneously
- Scenarios where all clients need the same data at the same time

## Project Setup

We'll build upon the chat application from Article 3 and add message history functionality. Ensure you have:
- Spring Boot 3.1.x or later
- `spring-boot-starter-websocket` dependency
- The chat application structure from Article 3

## Implementing Message History Service

First, let's create a service to store and retrieve chat message history.

### Step 1: Create Message History Service

Create `MessageHistoryService.java`:

```java
package com.example.stomptoyproject;

import org.springframework.stereotype.Service;

import java.util.ArrayList;
import java.util.Collections;
import java.util.List;

@Service
public class MessageHistoryService {

    private final List<ChatMessage> messageHistory = Collections.synchronizedList(new ArrayList<>());
    private static final int MAX_HISTORY_SIZE = 100;

    public void addMessage(ChatMessage message) {
        synchronized (messageHistory) {
            messageHistory.add(message);
            // Keep only the most recent messages
            if (messageHistory.size() > MAX_HISTORY_SIZE) {
                messageHistory.remove(0);
            }
        }
    }

    public List<ChatMessage> getRecentMessages(int count) {
        synchronized (messageHistory) {
            int size = messageHistory.size();
            int start = Math.max(0, size - count);
            return new ArrayList<>(messageHistory.subList(start, size));
        }
    }

    public List<ChatMessage> getAllMessages() {
        synchronized (messageHistory) {
            return new ArrayList<>(messageHistory);
        }
    }

    public void clearHistory() {
        synchronized (messageHistory) {
            messageHistory.clear();
        }
    }

    public int getHistorySize() {
        synchronized (messageHistory) {
            return messageHistory.size();
        }
    }
}
```

This service maintains a thread-safe list of recent messages. We use `Collections.synchronizedList()` to ensure thread safety when multiple clients connect simultaneously.

## Updating the Chat Controller

Now let's update the chat controller to store messages and provide history via `@SubscribeMapping`.

### Step 2: Enhanced Chat Controller with History

Update `ChatController.java`:

```java
package com.example.stomptoyproject;

import org.springframework.messaging.handler.annotation.MessageMapping;
import org.springframework.messaging.handler.annotation.Payload;
import org.springframework.messaging.handler.annotation.SendTo;
import org.springframework.messaging.handler.annotation.SubscribeMapping;
import org.springframework.messaging.simp.SimpMessageHeaderAccessor;
import org.springframework.stereotype.Controller;
import org.springframework.util.StringUtils;

import java.util.List;

@Controller
public class ChatController {

    private final MessageHistoryService historyService;

    public ChatController(MessageHistoryService historyService) {
        this.historyService = historyService;
    }

    @MessageMapping("/chat.sendMessage")
    @SendTo("/topic/public")
    public ChatMessage sendMessage(@Payload ChatMessage chatMessage,
                                   SimpMessageHeaderAccessor headerAccessor) {
        String username = (String) headerAccessor.getSessionAttributes().get("username");
        
        if (!StringUtils.hasText(chatMessage.getContent())) {
            return null;
        }
        
        if (username != null) {
            chatMessage.setSender(username);
        }
        
        chatMessage.setTimestamp(java.time.LocalDateTime.now());
        chatMessage.setType(ChatMessage.MessageType.CHAT);
        
        // Store in history
        historyService.addMessage(chatMessage);
        
        System.out.println("[" + chatMessage.getTimestamp() + "] " + 
                          chatMessage.getSender() + ": " + chatMessage.getContent());
        
        return chatMessage;
    }

    @MessageMapping("/chat.addUser")
    @SendTo("/topic/public")
    public ChatMessage addUser(@Payload ChatMessage chatMessage,
                               SimpMessageHeaderAccessor headerAccessor) {
        String username = chatMessage.getSender();
        if (!StringUtils.hasText(username)) {
            username = "Anonymous";
        }
        
        username = username.trim();
        if (username.length() > 20) {
            username = username.substring(0, 20);
        }
        
        headerAccessor.getSessionAttributes().put("username", username);
        chatMessage.setSender(username);
        chatMessage.setType(ChatMessage.MessageType.JOIN);
        chatMessage.setContent(username + " joined!");
        chatMessage.setTimestamp(java.time.LocalDateTime.now());
        
        // Store join message in history
        historyService.addMessage(chatMessage);
        
        System.out.println("User " + username + " joined the chat");
        return chatMessage;
    }

    @SubscribeMapping("/topic/public")
    public List<ChatMessage> getInitialMessages() {
        // Return recent messages (last 20) to the subscribing client
        List<ChatMessage> recentMessages = historyService.getRecentMessages(20);
        System.out.println("Sending " + recentMessages.size() + " messages to new subscriber");
        return recentMessages;
    }
}
```

The key addition is the `@SubscribeMapping` method:

**@SubscribeMapping("/topic/public")**: When a client subscribes to `/topic/public`, this method is called **before** the subscription is confirmed. The return value is sent directly to that client only.

**Important**: The method must return data synchronously. Spring converts the return value to JSON and sends it immediately to the subscribing client via `clientOutboundChannel`, bypassing the broker.

## Understanding the Subscribe Flow

Let's trace what happens when a client subscribes:

1. **Client sends SUBSCRIBE frame**: `SUBSCRIBE\nid:sub-0\ndestination:/topic/public\n\n\0`

2. **Spring receives subscription**:
   - Checks for `@SubscribeMapping` methods matching the destination
   - If found, calls the method immediately
   - Method return value goes to `clientOutboundChannel`
   - Sent directly to the requesting client (not through broker)

3. **Subscription confirmed**:
   - Client receives the initial data
   - Subscription is confirmed
   - Client can now receive regular broadcasts

4. **Future broadcasts**:
   - Messages sent via `@SendTo("/topic/public")` reach this client
   - The client receives both the initial data and ongoing updates

## Alternative: Returning Single Object vs List

You can return any serializable object:

```java
@SubscribeMapping("/topic/public")
public ChatMessageHistory getInitialMessages() {
    return new ChatMessageHistory(
        historyService.getRecentMessages(20),
        historyService.getHistorySize()
    );
}
```

Or even a simple wrapper:

```java
public class MessageHistoryResponse {
    private List<ChatMessage> messages;
    private int totalMessages;
    private LocalDateTime serverTime;

    // Constructors, getters, setters...
}
```

Spring automatically serializes complex objects to JSON.

## Creating a Dashboard with Initial Data

Let's build another example: a dashboard that loads current statistics on subscription.

### Step 3: Create Dashboard Data Model

Create `DashboardStats.java`:

```java
package com.example.stomptoyproject;

import java.time.LocalDateTime;

public class DashboardStats {
    private int activeUsers;
    private int totalMessages;
    private int messagesToday;
    private LocalDateTime lastUpdate;
    private double averageMessagesPerHour;

    public DashboardStats() {
        this.lastUpdate = LocalDateTime.now();
    }

    // Getters and setters
    public int getActiveUsers() {
        return activeUsers;
    }

    public void setActiveUsers(int activeUsers) {
        this.activeUsers = activeUsers;
    }

    public int getTotalMessages() {
        return totalMessages;
    }

    public void setTotalMessages(int totalMessages) {
        this.totalMessages = totalMessages;
    }

    public int getMessagesToday() {
        return messagesToday;
    }

    public void setMessagesToday(int messagesToday) {
        this.messagesToday = messagesToday;
    }

    public LocalDateTime getLastUpdate() {
        return lastUpdate;
    }

    public void setLastUpdate(LocalDateTime lastUpdate) {
        this.lastUpdate = lastUpdate;
    }

    public double getAverageMessagesPerHour() {
        return averageMessagesPerHour;
    }

    public void setAverageMessagesPerHour(double averageMessagesPerHour) {
        this.averageMessagesPerHour = averageMessagesPerHour;
    }
}
```

### Step 4: Create Dashboard Controller

Create `DashboardController.java`:

```java
package com.example.stomptoyproject;

import org.springframework.messaging.handler.annotation.SubscribeMapping;
import org.springframework.messaging.simp.SimpMessagingTemplate;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Controller;

import java.time.LocalDate;
import java.time.LocalDateTime;
import java.util.List;

@Controller
public class DashboardController {

    private final MessageHistoryService historyService;
    private final ChatService chatService;
    private final SimpMessagingTemplate messagingTemplate;

    public DashboardController(MessageHistoryService historyService,
                              ChatService chatService,
                              SimpMessagingTemplate messagingTemplate) {
        this.historyService = historyService;
        this.chatService = chatService;
        this.messagingTemplate = messagingTemplate;
    }

    @SubscribeMapping("/topic/dashboard")
    public DashboardStats getInitialDashboardStats() {
        DashboardStats stats = new DashboardStats();
        
        stats.setActiveUsers(chatService.getCurrentUserCount());
        stats.setTotalMessages(historyService.getHistorySize());
        
        // Calculate messages today
        List<ChatMessage> allMessages = historyService.getAllMessages();
        long messagesToday = allMessages.stream()
            .filter(msg -> msg.getTimestamp().toLocalDate().equals(LocalDate.now()))
            .count();
        stats.setMessagesToday((int) messagesToday);
        
        // Calculate average messages per hour (last 24 hours)
        LocalDateTime dayAgo = LocalDateTime.now().minusHours(24);
        long messagesLast24h = allMessages.stream()
            .filter(msg -> msg.getTimestamp().isAfter(dayAgo))
            .count();
        stats.setAverageMessagesPerHour(messagesLast24h / 24.0);
        
        stats.setLastUpdate(LocalDateTime.now());
        
        System.out.println("Sending initial dashboard stats to subscriber");
        return stats;
    }

    @Scheduled(fixedRate = 5000) // Every 5 seconds
    public void broadcastDashboardUpdates() {
        DashboardStats stats = new DashboardStats();
        
        stats.setActiveUsers(chatService.getCurrentUserCount());
        stats.setTotalMessages(historyService.getHistorySize());
        
        List<ChatMessage> allMessages = historyService.getAllMessages();
        long messagesToday = allMessages.stream()
            .filter(msg -> msg.getTimestamp().toLocalDate().equals(LocalDate.now()))
            .count();
        stats.setMessagesToday((int) messagesToday);
        
        LocalDateTime dayAgo = LocalDateTime.now().minusHours(24);
        long messagesLast24h = allMessages.stream()
            .filter(msg -> msg.getTimestamp().isAfter(dayAgo))
            .count();
        stats.setAverageMessagesPerHour(messagesLast24h / 24.0);
        
        stats.setLastUpdate(LocalDateTime.now());
        
        // Broadcast to all subscribers
        messagingTemplate.convertAndSend("/topic/dashboard", stats);
    }
}
```

This controller demonstrates both patterns:
- `@SubscribeMapping`: Sends initial stats when a client subscribes
- Scheduled broadcasting: Updates all subscribers every 5 seconds

## Updating ChatService

Make sure `ChatService` from Article 3 has the `getCurrentUserCount()` method:

```java
@Service
public class ChatService {
    // ... existing code ...
    
    public int getCurrentUserCount() {
        return userCount.get();
    }
}
```

## Creating Enhanced Chat Client with History

Let's update the chat client to handle initial message history.

### Step 5: Enhanced Chat Client

Create `chat-with-history.html` in `src/main/resources/static/`:

```html
<!DOCTYPE html>
<html>
<head>
    <title>Chat with History</title>
    <script src="https://cdn.jsdelivr.net/npm/sockjs-client@1/dist/sockjs.min.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/@stomp/stompjs@7/bundles/stomp.umd.min.js"></script>
    <style>
        /* Similar styles to Article 3, but add history loading indicator */
        .history-loading {
            text-align: center;
            padding: 20px;
            color: #666;
            font-style: italic;
        }
        .history-separator {
            border-top: 2px dashed #ccc;
            margin: 20px 0;
            text-align: center;
            color: #999;
            font-size: 12px;
        }
    </style>
</head>
<body>
    <!-- Similar structure to Article 3 chat.html -->
    <div class="chat-container">
        <div class="chat-header">
            <h1>💬 Chat with History</h1>
        </div>

        <div class="join-section" id="joinSection">
            <input type="text" id="usernameInput" placeholder="Your username..." />
            <button onclick="joinChat()">Join Chat</button>
        </div>

        <div class="messages-container" id="messagesContainer">
            <div class="history-loading" id="historyLoading" style="display:none;">
                Loading message history...
            </div>
        </div>

        <div class="input-section" id="inputSection" style="display:none;">
            <input type="text" id="messageInput" placeholder="Type your message..." />
            <button onclick="sendMessage()">Send</button>
        </div>
    </div>

    <script>
        let stompClient = null;
        let currentUsername = null;
        let historyLoaded = false;

        function joinChat() {
            const username = document.getElementById('usernameInput').value.trim();
            if (!username) {
                alert('Please enter a username');
                return;
            }

            currentUsername = username;
            const socket = new SockJS('/gs-guide-websocket');
            stompClient = StompJs.Stomp.over(socket);

            stompClient.connect({}, function(frame) {
                document.getElementById('joinSection').style.display = 'none';
                document.getElementById('inputSection').style.display = 'block';
                document.getElementById('historyLoading').style.display = 'block';

                // Subscribe to public messages
                // The @SubscribeMapping method will be called automatically
                // and the return value will be received here
                stompClient.subscribe('/topic/public', function(message) {
                    const chatMessage = JSON.parse(message.body);
                    
                    // Check if this is the initial history (array of messages)
                    if (Array.isArray(chatMessage)) {
                        // Handle initial message history
                        handleInitialHistory(chatMessage);
                        document.getElementById('historyLoading').style.display = 'none';
                        historyLoaded = true;
                    } else {
                        // Handle regular message
                        displayMessage(chatMessage);
                    }
                });

                // Notify server that user joined
                const joinMessage = {
                    type: 'JOIN',
                    sender: username,
                    content: ''
                };
                stompClient.send('/app/chat.addUser', {}, JSON.stringify(joinMessage));
            });
        }

        function handleInitialHistory(messages) {
            const messagesContainer = document.getElementById('messagesContainer');
            const historyLoading = document.getElementById('historyLoading');
            historyLoading.style.display = 'none';

            if (messages.length === 0) {
                const noHistory = document.createElement('div');
                noHistory.className = 'message system';
                noHistory.innerHTML = '<div class="message-content">No previous messages</div>';
                messagesContainer.appendChild(noHistory);
                return;
            }

            // Display history messages
            messages.forEach(function(msg) {
                displayMessage(msg, true); // true indicates it's from history
            });

            // Add separator
            const separator = document.createElement('div');
            separator.className = 'history-separator';
            separator.textContent = '--- New messages ---';
            messagesContainer.appendChild(separator);
        }

        function displayMessage(chatMessage, isHistory) {
            const messagesContainer = document.getElementById('messagesContainer');
            const messageDiv = document.createElement('div');
            messageDiv.className = 'message';

            const isOwnMessage = chatMessage.sender === currentUsername;
            const isSystemMessage = chatMessage.type === 'JOIN' || chatMessage.type === 'LEAVE';

            if (isSystemMessage) {
                messageDiv.className += ' system';
            } else if (isOwnMessage) {
                messageDiv.className += ' own';
            } else {
                messageDiv.className += ' other';
            }

            if (isHistory) {
                messageDiv.style.opacity = '0.7'; // Make history messages slightly faded
            }

            const timestamp = chatMessage.timestamp 
                ? new Date(chatMessage.timestamp).toLocaleTimeString()
                : new Date().toLocaleTimeString();

            messageDiv.innerHTML = `
                <div class="message-header">${chatMessage.sender} • ${timestamp}</div>
                <div class="message-content">${escapeHtml(chatMessage.content)}</div>
            `;

            messagesContainer.appendChild(messageDiv);
            
            if (!isHistory) {
                messagesContainer.scrollTop = messagesContainer.scrollHeight;
            }
        }

        function sendMessage() {
            const messageInput = document.getElementById('messageInput');
            const content = messageInput.value.trim();

            if (!content || !stompClient) {
                return;
            }

            const chatMessage = {
                type: 'CHAT',
                sender: currentUsername,
                content: content
            };

            stompClient.send('/app/chat.sendMessage', {}, JSON.stringify(chatMessage));
            messageInput.value = '';
        }

        function escapeHtml(text) {
            const map = {
                '&': '&amp;',
                '<': '&lt;',
                '>': '&gt;',
                '"': '&quot;',
                "'": '&#039;'
            };
            return text.replace(/[&<>"']/g, m => map[m]);
        }

        document.getElementById('messageInput').addEventListener('keypress', function(e) {
            if (e.key === 'Enter') {
                sendMessage();
            }
        });
    </script>
</body>
</html>
```

**Important Note**: When using `@SubscribeMapping`, the return value is sent as a **regular message** to the subscription destination. The client receives it through the same subscription callback. However, since we're returning a `List`, the client needs to detect if the received message is an array (initial history) or a single object (regular message).

## Alternative: Dedicated History Endpoint

A cleaner approach is to use a separate destination for history:

```java
@SubscribeMapping("/topic/history")
public List<ChatMessage> getChatHistory() {
    return historyService.getRecentMessages(20);
}
```

Then clients subscribe to both:
```javascript
// Subscribe to history (one-time)
stompClient.subscribe('/topic/history', function(message) {
    const history = JSON.parse(message.body);
    displayHistory(history);
});

// Subscribe to new messages (ongoing)
stompClient.subscribe('/topic/public', function(message) {
    const msg = JSON.parse(message.body);
    displayMessage(msg);
});
```

## Creating Dashboard Client

Create `dashboard.html`:

```html
<!DOCTYPE html>
<html>
<head>
    <title>Chat Dashboard</title>
    <script src="https://cdn.jsdelivr.net/npm/sockjs-client@1/dist/sockjs.min.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/@stomp/stompjs@7/bundles/stomp.umd.min.js"></script>
    <style>
        body {
            font-family: Arial, sans-serif;
            max-width: 800px;
            margin: 50px auto;
            padding: 20px;
        }
        .stat-card {
            border: 1px solid #ddd;
            padding: 20px;
            margin: 10px;
            border-radius: 5px;
            display: inline-block;
            min-width: 150px;
            text-align: center;
        }
        .stat-value {
            font-size: 32px;
            font-weight: bold;
            color: #007bff;
        }
        .stat-label {
            color: #666;
            margin-top: 5px;
        }
    </style>
</head>
<body>
    <h1>Chat Dashboard</h1>
    <button onclick="connect()">Connect</button>
    <div id="stats"></div>

    <script>
        let client = null;

        function connect() {
            const socket = new SockJS('/gs-guide-websocket');
            client = StompJs.Stomp.over(socket);
            
            client.connect({}, () => {
                // Subscribe to dashboard updates
                // Initial data will come via @SubscribeMapping
                client.subscribe('/topic/dashboard', (message) => {
                    const stats = JSON.parse(message.body);
                    displayStats(stats);
                });
            });
        }

        function displayStats(stats) {
            const container = document.getElementById('stats');
            container.innerHTML = `
                <div class="stat-card">
                    <div class="stat-value">${stats.activeUsers}</div>
                    <div class="stat-label">Active Users</div>
                </div>
                <div class="stat-card">
                    <div class="stat-value">${stats.totalMessages}</div>
                    <div class="stat-label">Total Messages</div>
                </div>
                <div class="stat-card">
                    <div class="stat-value">${stats.messagesToday}</div>
                    <div class="stat-label">Messages Today</div>
                </div>
                <div class="stat-card">
                    <div class="stat-value">${stats.averageMessagesPerHour.toFixed(1)}</div>
                    <div class="stat-label">Avg/Hour</div>
                </div>
            `;
        }
    </script>
</body>
</html>
```

## Complete Test Client

Here's a minimal test for `@SubscribeMapping`:

```html
<!DOCTYPE html>
<html>
<head>
    <title>SubscribeMapping Test</title>
    <script src="https://cdn.jsdelivr.net/npm/sockjs-client@1/dist/sockjs.min.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/@stomp/stompjs@7/bundles/stomp.umd.min.js"></script>
</head>
<body>
    <h1>SubscribeMapping Test</h1>
    <button onclick="test()">Subscribe and Get Initial Data</button>
    <div id="output"></div>

    <script>
        function test() {
            const socket = new SockJS('http://localhost:8080/gs-guide-websocket');
            const client = StompJs.Stomp.over(socket);
            
            client.connect({}, () => {
                log('Connected!');
                
                // Subscribe - @SubscribeMapping will be called automatically
                client.subscribe('/topic/public', (msg) => {
                    const data = JSON.parse(msg.body);
                    
                    if (Array.isArray(data)) {
                        log(`✅ Received initial data: ${data.length} messages`);
                        data.forEach(m => log(`  - ${m.sender}: ${m.content}`));
                    } else {
                        log(`📨 Regular message: ${data.sender}: ${data.content}`);
                    }
                });
            });
        }
        
        function log(msg) {
            const output = document.getElementById('output');
            output.innerHTML += '<p>' + new Date().toLocaleTimeString() + ' - ' + msg + '</p>';
        }
    </script>
</body>
</html>
```

## What We've Accomplished

In this article, we've:
1. Understood the difference between `@SubscribeMapping` and other messaging patterns
2. Implemented message history loading using `@SubscribeMapping`
3. Created a dashboard that loads initial statistics
4. Learned when to use `@SubscribeMapping` vs other approaches
5. Built clients that handle initial data correctly

## Next Steps

You now know how to provide initial data to clients when they subscribe, enabling faster application startup and better user experience.

In our next article, we'll explore security and authentication, learning how to identify users and restrict access to specific destinations. This is crucial for building production-ready applications that differentiate between clients and protect sensitive data.
