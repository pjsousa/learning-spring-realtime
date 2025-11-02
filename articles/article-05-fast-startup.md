# Fast Startup: Using @SubscribeMapping for Initial Data Loading and Setup

*Estimated reading time: 18 minutes · Target word count: ~1,800*

## Overview

In the previous articles, we've seen how `@MessageMapping` handles client-initiated messages and how scheduled tasks can push updates to all subscribers. But there's a common scenario we haven't addressed: **what happens when a client first connects and needs immediate, one-time data?**

Consider these use cases:
- A chat application where new users want to see the last 20 messages when they join
- A dashboard that needs current statistics as soon as it loads
- A collaborative editor requiring the current document state upon connection
- A configuration endpoint that loads app settings synchronously

These scenarios share a key requirement: the client needs data **immediately upon subscribing**, but only **that specific client** should receive it—not all subscribers via a broadcast. This is exactly where `@SubscribeMapping` shines.

In this fifth installment of our Spring Boot STOMP series, you'll learn how `@SubscribeMapping` differs from `@MessageMapping`, understand its message flow through Spring's `clientOutboundChannel`, and build practical examples including a chat application with message history and a real-time dashboard that loads initial state.

## Series Context

This article builds upon **Article 3: The Classic STOMP Chat Room**, where we implemented two-way messaging. While you can follow along with just the basics from Articles 1–3, having a working chat application will make the examples more tangible.

By the end of this tutorial, you will:

- Understand when and why to use `@SubscribeMapping` versus `@MessageMapping`
- Implement message history loading in a chat application
- Build a dashboard that initializes with current statistics
- Handle initial data correctly in browser clients
- Recognize the difference between broker-based broadcasting and direct client responses

## Why @SubscribeMapping Exists

Let's compare the three primary message handling patterns in Spring STOMP:

| Pattern | Trigger | Flow | Use Case |
|---------|---------|------|----------|
| `@MessageMapping` | Client SEND frame | Client → `clientInboundChannel` → Controller → `brokerChannel` → All Subscribers | Ongoing two-way communication |
| Scheduled Broadcasting | Timer/Event | Server → `brokerChannel` → All Subscribers | Regular updates to everyone |
| `@SubscribeMapping` | Client SUBSCRIBE frame | Client Subscription → Controller → `clientOutboundChannel` → That Client Only | One-time initial data load |

**The Critical Difference**: When a method annotated with `@SubscribeMapping` returns a value, Spring sends it directly to the subscribing client via the `clientOutboundChannel`, bypassing the broker entirely. This is perfect for initialization data that only the new subscriber needs.

**When to use `@SubscribeMapping`:**
- Loading application state when a client first connects
- Fetching configuration or settings for a specific client
- Retrieving historical data that doesn't need to be broadcast
- Optimizing bandwidth by avoiding unnecessary broadcasts

**When NOT to use `@SubscribeMapping`:**
- Data that should be pushed to all clients simultaneously
- Regular ongoing updates (use scheduled broadcasting or `@SendTo`)
- Scenarios requiring the same data at the exact same moment for all subscribers

## Project Setup

We'll enhance the chat application from Article 3. If you're starting fresh, create a Spring Boot 3.3+ project with:

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
</dependencies>
```

Generate via [start.spring.io](https://start.spring.io) or use Maven/Gradle with these dependencies.

## Step 1: Building the Message History Service

First, we need a service to store and retrieve chat message history. This will be used both for storing new messages and providing history to new subscribers.

Create `src/main/java/com/example/chat/service/MessageHistoryService.java`:

```java
package com.example.chat.service;

import com.example.chat.model.ChatMessage;
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
            // Keep only the most recent messages to prevent memory issues
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

    public int getHistorySize() {
        synchronized (messageHistory) {
            return messageHistory.size();
        }
    }

    public void clearHistory() {
        synchronized (messageHistory) {
            messageHistory.clear();
        }
    }
}
```

**Key points:**
- `Collections.synchronizedList` ensures thread safety when multiple clients connect simultaneously
- We limit history to 100 messages to prevent unbounded memory growth
- Methods return defensive copies to prevent external modification

## Step 2: Enhanced Chat Message Model

Ensure your `ChatMessage` model (from Article 3) includes a timestamp. Here's a reference if needed:

```java
package com.example.chat.model;

import java.time.Instant;

public class ChatMessage {

    public enum MessageType {
        CHAT, JOIN, LEAVE
    }

    private MessageType type = MessageType.CHAT;
    private String sender;
    private String content;
    private Instant timestamp = Instant.now();

    // Default constructor
    public ChatMessage() {}

    // Getters and setters
    public MessageType getType() {
        return type;
    }

    public void setType(MessageType type) {
        this.type = type;
    }

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

## Step 3: Updating the Chat Controller with @SubscribeMapping

Now we'll modify the chat controller to:
1. Store messages in history when they arrive
2. Provide history via `@SubscribeMapping` when clients subscribe

Update or create `src/main/java/com/example/chat/controller/ChatController.java`:

```java
package com.example.chat.controller;

import com.example.chat.model.ChatMessage;
import com.example.chat.service.MessageHistoryService;
import org.springframework.messaging.handler.annotation.MessageMapping;
import org.springframework.messaging.handler.annotation.Payload;
import org.springframework.messaging.handler.annotation.SendTo;
import org.springframework.messaging.handler.annotation.SubscribeMapping;
import org.springframework.messaging.simp.SimpMessageHeaderAccessor;
import org.springframework.stereotype.Controller;
import org.springframework.util.StringUtils;

import java.time.Instant;
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
        
        chatMessage.setTimestamp(Instant.now());
        chatMessage.setType(ChatMessage.MessageType.CHAT);
        
        // Store in history before broadcasting
        historyService.addMessage(chatMessage);
        
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
        chatMessage.setTimestamp(Instant.now());
        
        // Store join message in history
        historyService.addMessage(chatMessage);
        
        return chatMessage;
    }

    @SubscribeMapping("/topic/public")
    public List<ChatMessage> getInitialMessages() {
        // Return recent messages (last 20) to the subscribing client only
        List<ChatMessage> recentMessages = historyService.getRecentMessages(20);
        System.out.println("Sending " + recentMessages.size() + " messages to new subscriber");
        return recentMessages;
    }
}
```

**Understanding the `@SubscribeMapping` method:**

1. **Method signature**: `@SubscribeMapping("/topic/public")` means this method is invoked when a client subscribes to `/topic/public`

2. **Return value flow**: The returned `List<ChatMessage>` is:
   - Serialized to JSON automatically
   - Sent directly to the subscribing client via `clientOutboundChannel`
   - **NOT** broadcast to other subscribers
   - Delivered **before** the subscription is confirmed

3. **Timing**: This happens synchronously during the SUBSCRIBE frame processing, so the client receives initial data immediately

## Understanding the Subscription Flow

Let's trace exactly what happens when a client subscribes:

```
1. Client sends SUBSCRIBE frame:
   SUBSCRIBE
   id:sub-0
   destination:/topic/public
   
2. Spring receives subscription request:
   └─> Checks for @SubscribeMapping matching /topic/public
   └─> Calls getInitialMessages() method
   └─> Return value goes to clientOutboundChannel
   └─> Sent directly to requesting client (bypasses broker)
   
3. Subscription confirmed:
   └─> Client receives initial data
   └─> Subscription is established
   └─> Client now receives regular broadcasts via broker
   
4. Future messages:
   └─> @MessageMapping methods with @SendTo("/topic/public")
   └─> Messages go through brokerChannel
   └─> All subscribers (including this client) receive them
```

**Important**: The initial data from `@SubscribeMapping` arrives as a **regular MESSAGE frame** on the same subscription destination. The client receives it in the same callback, but it will be an array instead of a single object.

## Step 4: Building the Enhanced Chat Client

Now we need a client that can handle both:
1. Initial message history (array) from `@SubscribeMapping`
2. Regular messages (single objects) from `@SendTo` broadcasts

Create `src/main/resources/static/chat-with-history.html`:

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Chat with Message History</title>
    <style>
        * {
            box-sizing: border-box;
            margin: 0;
            padding: 0;
        }

        body {
            font-family: system-ui, -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif;
            background: #f5f5f5;
            padding: 2rem;
        }

        .chat-container {
            max-width: 800px;
            margin: 0 auto;
            background: white;
            border-radius: 12px;
            box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
            overflow: hidden;
        }

        .chat-header {
            background: #2563eb;
            color: white;
            padding: 1.5rem;
            text-align: center;
        }

        .chat-header h1 {
            font-size: 1.5rem;
            font-weight: 600;
        }

        .join-section {
            padding: 2rem;
            text-align: center;
            border-bottom: 1px solid #e5e7eb;
        }

        .join-section input {
            padding: 0.75rem 1rem;
            font-size: 1rem;
            border: 2px solid #d1d5db;
            border-radius: 8px;
            width: 300px;
            margin-right: 0.5rem;
        }

        .join-section button {
            padding: 0.75rem 1.5rem;
            font-size: 1rem;
            background: #2563eb;
            color: white;
            border: none;
            border-radius: 8px;
            cursor: pointer;
            font-weight: 600;
        }

        .join-section button:hover {
            background: #1d4ed8;
        }

        .messages-container {
            height: 400px;
            overflow-y: auto;
            padding: 1rem;
            background: #fafafa;
        }

        .history-loading {
            text-align: center;
            padding: 1rem;
            color: #6b7280;
            font-style: italic;
        }

        .history-separator {
            border-top: 2px dashed #d1d5db;
            margin: 1rem 0;
            text-align: center;
            color: #9ca3af;
            font-size: 0.875rem;
            padding: 0.5rem;
        }

        .message {
            margin-bottom: 1rem;
            padding: 0.75rem;
            border-radius: 8px;
            background: white;
            box-shadow: 0 1px 2px rgba(0, 0, 0, 0.05);
        }

        .message.system {
            background: #fef3c7;
            border-left: 4px solid #f59e0b;
        }

        .message.own {
            background: #dbeafe;
            border-left: 4px solid #2563eb;
        }

        .message.history {
            opacity: 0.7;
        }

        .message-header {
            font-size: 0.875rem;
            color: #6b7280;
            margin-bottom: 0.25rem;
            font-weight: 600;
        }

        .message-content {
            color: #111827;
            line-height: 1.5;
        }

        .input-section {
            padding: 1rem;
            border-top: 1px solid #e5e7eb;
            display: flex;
            gap: 0.5rem;
        }

        .input-section input {
            flex: 1;
            padding: 0.75rem;
            font-size: 1rem;
            border: 2px solid #d1d5db;
            border-radius: 8px;
        }

        .input-section button {
            padding: 0.75rem 1.5rem;
            font-size: 1rem;
            background: #2563eb;
            color: white;
            border: none;
            border-radius: 8px;
            cursor: pointer;
            font-weight: 600;
        }

        .input-section button:hover {
            background: #1d4ed8;
        }

        .input-section button:disabled {
            background: #9ca3af;
            cursor: not-allowed;
        }
    </style>
</head>
<body>
    <div class="chat-container">
        <div class="chat-header">
            <h1>💬 Chat with Message History</h1>
            <p style="font-size: 0.875rem; margin-top: 0.5rem;">Join the conversation and see recent messages</p>
        </div>

        <div class="join-section" id="joinSection">
            <input type="text" id="usernameInput" placeholder="Enter your username..." autocomplete="username">
            <button onclick="joinChat()">Join Chat</button>
        </div>

        <div class="messages-container" id="messagesContainer">
            <div class="history-loading" id="historyLoading" style="display: none;">
                Loading message history...
            </div>
        </div>

        <div class="input-section" id="inputSection" style="display: none;">
            <input type="text" id="messageInput" placeholder="Type your message..." autocomplete="off">
            <button id="sendButton" onclick="sendMessage()">Send</button>
        </div>
    </div>

    <script src="https://cdn.jsdelivr.net/npm/sockjs-client@1/dist/sockjs.min.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/@stomp/stompjs@7/bundles/stomp.umd.min.js"></script>
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
            const socket = new SockJS('/ws');
            stompClient = StompJs.Stomp.over(socket);
            
            // Disable STOMP debugging for cleaner console
            stompClient.debug = () => {};

            stompClient.connect({}, function(frame) {
                console.log('Connected: ' + frame);
                
                document.getElementById('joinSection').style.display = 'none';
                document.getElementById('inputSection').style.display = 'flex';
                document.getElementById('historyLoading').style.display = 'block';
                document.getElementById('sendButton').disabled = false;

                // Subscribe to public messages
                // When subscription happens, @SubscribeMapping will be called
                // and the return value will arrive as the first message
                stompClient.subscribe('/topic/public', function(message) {
                    const data = JSON.parse(message.body);
                    
                    // Check if this is initial history (array) or regular message (object)
                    if (Array.isArray(data)) {
                        // This is the initial history from @SubscribeMapping
                        handleInitialHistory(data);
                        document.getElementById('historyLoading').style.display = 'none';
                        historyLoaded = true;
                    } else {
                        // This is a regular message from @SendTo
                        displayMessage(data);
                    }
                });

                // Notify server that user joined
                const joinMessage = {
                    type: 'JOIN',
                    sender: username,
                    content: ''
                };
                stompClient.send('/app/chat.addUser', {}, JSON.stringify(joinMessage));
            }, function(error) {
                console.error('STOMP error: ' + error);
                alert('Connection failed. Please try again.');
            });
        }

        function handleInitialHistory(messages) {
            const messagesContainer = document.getElementById('messagesContainer');
            const historyLoading = document.getElementById('historyLoading');
            historyLoading.style.display = 'none';

            if (messages.length === 0) {
                const noHistory = document.createElement('div');
                noHistory.className = 'message system';
                noHistory.innerHTML = '<div class="message-content">No previous messages. Be the first to chat!</div>';
                messagesContainer.appendChild(noHistory);
                return;
            }

            // Display history messages (faded style)
            messages.forEach(function(msg) {
                displayMessage(msg, true); // true indicates it's from history
            });

            // Add separator between history and new messages
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
            }

            if (isHistory) {
                messageDiv.className += ' history';
            }

            const timestamp = chatMessage.timestamp 
                ? new Date(chatMessage.timestamp).toLocaleTimeString()
                : new Date().toLocaleTimeString();

            messageDiv.innerHTML = `
                <div class="message-header">${escapeHtml(chatMessage.sender || 'System')} • ${timestamp}</div>
                <div class="message-content">${escapeHtml(chatMessage.content || '')}</div>
            `;

            messagesContainer.appendChild(messageDiv);
            
            // Auto-scroll only for new messages (not history)
            if (!isHistory) {
                messagesContainer.scrollTop = messagesContainer.scrollHeight;
            }
        }

        function sendMessage() {
            const messageInput = document.getElementById('messageInput');
            const content = messageInput.value.trim();

            if (!content || !stompClient || !stompClient.connected) {
                return;
            }

            const chatMessage = {
                type: 'CHAT',
                sender: currentUsername,
                content: content
            };

            stompClient.send('/app/chat.sendMessage', {}, JSON.stringify(chatMessage));
            messageInput.value = '';
            messageInput.focus();
        }

        function escapeHtml(text) {
            if (!text) return '';
            const map = {
                '&': '&amp;',
                '<': '&lt;',
                '>': '&gt;',
                '"': '&quot;',
                "'": '&#039;'
            };
            return text.toString().replace(/[&<>"']/g, m => map[m]);
        }

        // Allow Enter key to send message
        document.getElementById('messageInput').addEventListener('keypress', function(e) {
            if (e.key === 'Enter') {
                sendMessage();
            }
        });

        // Allow Enter key to join
        document.getElementById('usernameInput').addEventListener('keypress', function(e) {
            if (e.key === 'Enter') {
                joinChat();
            }
        });
    </script>
</body>
</html>
```

**Client-side key points:**

1. **Subscription handling**: When we call `stompClient.subscribe('/topic/public', ...)`, the server's `@SubscribeMapping` method is triggered automatically

2. **Array vs object detection**: The first message will be an array (history), subsequent messages will be objects. We check with `Array.isArray(data)`

3. **Visual distinction**: History messages are displayed with reduced opacity to differentiate them from new messages

## Step 5: Alternative Approach - Separate History Destination

Instead of mixing history with regular messages, you can use a dedicated destination. This makes client handling cleaner:

```java
@SubscribeMapping("/topic/history")
public List<ChatMessage> getChatHistory() {
    return historyService.getRecentMessages(20);
}
```

Then clients subscribe to both destinations separately:

```javascript
// Subscribe to history (one-time initial data)
stompClient.subscribe('/topic/history', function(message) {
    const history = JSON.parse(message.body);
    displayHistory(history);
});

// Subscribe to new messages (ongoing broadcasts)
stompClient.subscribe('/topic/public', function(message) {
    const msg = JSON.parse(message.body);
    displayMessage(msg);
});
```

This approach is cleaner because:
- The client doesn't need array/object type checking
- History and live messages are clearly separated
- Multiple clients can have different history requirements

## Step 6: Building a Dashboard Example

Let's create another practical example: a dashboard that loads initial statistics when a client subscribes, then receives periodic updates.

Create `src/main/java/com/example/chat/model/DashboardStats.java`:

```java
package com.example.chat.model;

import java.time.Instant;

public class DashboardStats {
    private int activeUsers;
    private int totalMessages;
    private int messagesToday;
    private Instant lastUpdate;
    private double averageMessagesPerHour;

    public DashboardStats() {
        this.lastUpdate = Instant.now();
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

    public Instant getLastUpdate() {
        return lastUpdate;
    }

    public void setLastUpdate(Instant lastUpdate) {
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

Create `src/main/java/com/example/chat/controller/DashboardController.java`:

```java
package com.example.chat.controller;

import com.example.chat.model.ChatMessage;
import com.example.chat.model.DashboardStats;
import com.example.chat.service.MessageHistoryService;
import org.springframework.messaging.handler.annotation.SubscribeMapping;
import org.springframework.messaging.simp.SimpMessagingTemplate;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Controller;

import java.time.Instant;
import java.time.LocalDate;
import java.util.List;

@Controller
public class DashboardController {

    private final MessageHistoryService historyService;
    private final SimpMessagingTemplate messagingTemplate;
    private int activeUserCount = 0; // Simplified; in production, track actual sessions

    public DashboardController(MessageHistoryService historyService,
                              SimpMessagingTemplate messagingTemplate) {
        this.historyService = historyService;
        this.messagingTemplate = messagingTemplate;
    }

    @SubscribeMapping("/topic/dashboard")
    public DashboardStats getInitialDashboardStats() {
        DashboardStats stats = calculateStats();
        System.out.println("Sending initial dashboard stats to subscriber");
        return stats;
    }

    @Scheduled(fixedRate = 5000) // Every 5 seconds
    public void broadcastDashboardUpdates() {
        DashboardStats stats = calculateStats();
        // Broadcast to all subscribers (not just the one who subscribed)
        messagingTemplate.convertAndSend("/topic/dashboard", stats);
    }

    private DashboardStats calculateStats() {
        DashboardStats stats = new DashboardStats();
        
        stats.setActiveUsers(activeUserCount);
        stats.setTotalMessages(historyService.getHistorySize());
        
        // Calculate messages today
        List<ChatMessage> allMessages = historyService.getAllMessages();
        long messagesToday = allMessages.stream()
            .filter(msg -> {
                if (msg.getTimestamp() == null) return false;
                return msg.getTimestamp().atZone(java.time.ZoneId.systemDefault())
                          .toLocalDate().equals(LocalDate.now());
            })
            .count();
        stats.setMessagesToday((int) messagesToday);
        
        // Calculate average messages per hour (last 24 hours)
        Instant dayAgo = Instant.now().minusSeconds(86400); // 24 hours
        long messagesLast24h = allMessages.stream()
            .filter(msg -> msg.getTimestamp() != null && msg.getTimestamp().isAfter(dayAgo))
            .count();
        stats.setAverageMessagesPerHour(messagesLast24h / 24.0);
        
        stats.setLastUpdate(Instant.now());
        
        return stats;
    }

    // These would be called by session event listeners in a real app
    public void incrementUserCount() {
        activeUserCount++;
    }

    public void decrementUserCount() {
        activeUserCount = Math.max(0, activeUserCount - 1);
    }
}
```

This demonstrates the **hybrid pattern**:
- `@SubscribeMapping`: Provides initial stats when a client subscribes
- `@Scheduled` + `SimpMessagingTemplate`: Pushes updates to all subscribers every 5 seconds

## Testing the Implementation

1. **Start the application:**
   ```bash
   ./mvnw spring-boot:run
   ```

2. **Test message history:**
   - Open `http://localhost:8080/chat-with-history.html`
   - Enter a username and join
   - Send a few messages
   - Open a second browser tab and join with a different username
   - The second tab should see the recent messages (history) before new ones arrive

3. **Verify the flow:**
   - Check browser console logs to see when initial history arrives
   - Watch server logs for "Sending X messages to new subscriber"
   - Confirm that history messages appear faded (with `.history` class)

4. **Test multiple subscriptions:**
   - Open multiple tabs and confirm each receives its own history
   - Verify that new messages broadcast to all tabs correctly

## Troubleshooting

| Issue | Cause | Solution |
|-------|-------|----------|
| History appears as regular messages | Client not detecting array vs object | Check `Array.isArray()` logic in subscription callback |
| No history received | `@SubscribeMapping` method not called | Verify destination matches exactly (`/topic/public`) |
| History received but duplicated | Client displaying history and regular messages incorrectly | Ensure array is handled separately before subscription confirmation |
| Only first client gets history | Misunderstanding of `@SubscribeMapping` behavior | Each subscriber gets their own initial data - this is expected |

## Recap and Next Steps

You've now mastered `@SubscribeMapping` and understand:

- **When to use it**: Initial data loading, one-time transfers, optimizing startup
- **How it works**: Direct delivery via `clientOutboundChannel`, bypassing the broker
- **Implementation patterns**: Single destination vs. separate history destination
- **Client handling**: Detecting arrays vs. objects, visual differentiation

This technique is essential for building responsive real-time applications where users need context immediately upon connection.

**Coming up in Article 6**: We'll explore security and authentication, learning how to identify users and restrict access to specific destinations. This sets the foundation for user-specific messaging and protected channels.

---

## Appendix: Complete Test Client

Here's a minimal standalone test page to verify `@SubscribeMapping` behavior:

```html
<!DOCTYPE html>
<html>
<head>
    <title>@SubscribeMapping Test</title>
    <style>
        body { font-family: monospace; padding: 2rem; }
        .output { background: #f5f5f5; padding: 1rem; margin-top: 1rem; border-radius: 4px; }
        .message { margin: 0.5rem 0; padding: 0.5rem; background: white; border-left: 3px solid #2563eb; }
        button { padding: 0.75rem 1.5rem; background: #2563eb; color: white; border: none; border-radius: 4px; cursor: pointer; }
    </style>
</head>
<body>
    <h1>@SubscribeMapping Test</h1>
    <button onclick="testSubscribe()">Subscribe and Get Initial Data</button>
    <div id="output" class="output"></div>

    <script src="https://cdn.jsdelivr.net/npm/sockjs-client@1/dist/sockjs.min.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/@stomp/stompjs@7/bundles/stomp.umd.min.js"></script>
    <script>
        function testSubscribe() {
            const socket = new SockJS('/ws');
            const client = StompJs.Stomp.over(socket);
            client.debug = () => {};
            
            const output = document.getElementById('output');
            
            client.connect({}, () => {
                log('✅ Connected!');
                
                // Subscribe - @SubscribeMapping will be called automatically
                client.subscribe('/topic/public', (msg) => {
                    const data = JSON.parse(msg.body);
                    
                    if (Array.isArray(data)) {
                        log(`📦 Received initial history: ${data.length} messages`);
                        data.forEach((m, i) => {
                            log(`  ${i + 1}. [${m.sender}] ${m.content}`);
                        });
                    } else {
                        log(`📨 Regular message: [${data.sender}] ${data.content}`);
                    }
                });
                
                log('👂 Subscribed to /topic/public');
                log('⏳ Waiting for initial data from @SubscribeMapping...');
            }, (error) => {
                log('❌ Connection error: ' + error);
            });
        }
        
        function log(msg) {
            const output = document.getElementById('output');
            const div = document.createElement('div');
            div.className = 'message';
            div.textContent = new Date().toLocaleTimeString() + ' - ' + msg;
            output.appendChild(div);
            output.scrollTop = output.scrollHeight;
        }
    </script>
</body>
</html>
```

Save this as `src/main/resources/static/test-subscribe.html` and access it at `http://localhost:8080/test-subscribe.html` to verify `@SubscribeMapping` behavior independently.