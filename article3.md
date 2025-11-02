# Mastering Two-Way Updates: The Classic STOMP Chat Room Case Study

## Introduction

In our previous articles, we've explored one-way communication patterns: clients sending messages to the server (Article 1) and servers broadcasting updates to clients (Article 2). Real-world applications, however, require true bidirectional communication where clients can both send and receive messages in a coordinated flow.

In this article, we'll build a classic multi-user chat application—a perfect use case for WebSocket technology. Chat applications exemplify the power of real-time, two-way communication: when one user sends a message, it must immediately appear on all other users' screens, including the sender's own display for confirmation.

By implementing this chat room, you'll master the complete STOMP message flow: client SEND operations, server-side message processing with validation, and automatic broadcasting to all subscribers. This pattern is fundamental to collaborative applications, gaming, customer support systems, and any scenario requiring real-time group interaction.

## Understanding the Chat Application Architecture

A chat application demonstrates several key WebSocket concepts:

**Message Flow**: When a user types a message and clicks "Send":
1. Client sends a STOMP SEND frame to `/app/chat`
2. Server receives it via `clientInboundChannel`
3. `@MessageMapping` routes it to the controller method
4. Controller processes and validates the message
5. `@SendTo` broadcasts it to `/topic/public`
6. All subscribed clients receive it via `clientOutboundChannel`

**Session Management**: Each WebSocket connection represents a user session. Spring automatically associates WebSocket sessions with connections, allowing you to track active users.

**Message Broadcasting**: All clients subscribe to the same topic (`/topic/public`), creating a broadcast channel where every message reaches everyone.

**State Management**: While our simple chat won't maintain message history (we'll add that in later articles), we'll track active users and notify others when someone joins or leaves.

## Project Setup

We'll build upon the foundation from Articles 1 and 2. Ensure you have:
- Spring Boot 3.1.x or later
- `spring-boot-starter-websocket` dependency
- `WebSocketConfig` with `/topic` broker enabled
- Basic understanding of `@MessageMapping` and `@SendTo`

## Creating the Message Models

Let's start by defining the data structures for our chat messages.

### Step 1: Create the ChatMessage Model

Create `ChatMessage.java`:

```java
package com.example.stomptoyproject;

import java.time.LocalDateTime;

public class ChatMessage {
    private MessageType type;
    private String content;
    private String sender;
    private LocalDateTime timestamp;

    public enum MessageType {
        CHAT,    // Regular chat message
        JOIN,    // User joined
        LEAVE    // User left
    }

    public ChatMessage() {
        this.timestamp = LocalDateTime.now();
    }

    public ChatMessage(MessageType type, String content, String sender) {
        this.type = type;
        this.content = content;
        this.sender = sender;
        this.timestamp = LocalDateTime.now();
    }

    // Getters and setters
    public MessageType getType() {
        return type;
    }

    public void setType(MessageType type) {
        this.type = type;
    }

    public String getContent() {
        return content;
    }

    public void setContent(String content) {
        this.content = content;
    }

    public String getSender() {
        return sender;
    }

    public void setSender(String sender) {
        this.sender = sender;
    }

    public LocalDateTime getTimestamp() {
        return timestamp;
    }

    public void setTimestamp(LocalDateTime timestamp) {
        this.timestamp = timestamp;
    }
}
```

This model supports different message types, allowing us to distinguish between regular chat messages and system notifications (join/leave events).

## Implementing the Chat Controller

Now let's create the controller that handles incoming chat messages and broadcasts them to all subscribers.

### Step 2: Create the Chat Controller

Create `ChatController.java`:

```java
package com.example.stomptoyproject;

import org.springframework.messaging.handler.annotation.MessageMapping;
import org.springframework.messaging.handler.annotation.Payload;
import org.springframework.messaging.handler.annotation.SendTo;
import org.springframework.messaging.simp.SimpMessageHeaderAccessor;
import org.springframework.stereotype.Controller;

@Controller
public class ChatController {

    @MessageMapping("/chat.sendMessage")
    @SendTo("/topic/public")
    public ChatMessage sendMessage(@Payload ChatMessage chatMessage) {
        // The message is already constructed by the client
        // We can add validation, logging, or processing here
        System.out.println("Received message from " + chatMessage.getSender() + 
                          ": " + chatMessage.getContent());
        return chatMessage;
    }

    @MessageMapping("/chat.addUser")
    @SendTo("/topic/public")
    public ChatMessage addUser(@Payload ChatMessage chatMessage,
                               SimpMessageHeaderAccessor headerAccessor) {
        // Add username in websocket session
        headerAccessor.getSessionAttributes().put("username", chatMessage.getSender());
        
        System.out.println("User " + chatMessage.getSender() + " joined the chat");
        return chatMessage;
    }
}
```

Let's break down the key annotations and concepts:

**@MessageMapping("/chat.sendMessage")**: Maps incoming messages sent to `/app/chat.sendMessage` to this method. Remember, the `/app` prefix comes from our WebSocket configuration.

**@Payload**: This annotation explicitly marks the method parameter as the message payload. While not strictly required for simple types, it's a best practice that makes your intent clear and helps with more complex scenarios.

**@SendTo("/topic/public")**: Automatically broadcasts the method's return value to all clients subscribed to `/topic/public`. Spring converts the Java object to JSON automatically.

**SimpMessageHeaderAccessor**: This utility class provides access to STOMP message headers and session attributes. We use it to store the username in the WebSocket session, which allows us to track which user sent which message.

**Session Attributes**: The `headerAccessor.getSessionAttributes()` map persists data for the duration of the WebSocket session. This is perfect for storing user-specific information like usernames.

## Understanding the Message Flow in Detail

Let's trace a complete message journey:

1. **User types "Hello everyone!" and clicks Send**
   - JavaScript constructs a `ChatMessage` object
   - Calls `stompClient.send('/app/chat.sendMessage', {}, JSON.stringify(chatMessage))`

2. **Client sends STOMP SEND frame**
   - WebSocket connection transmits the frame
   - Spring receives it via the WebSocket connection

3. **Spring routing**
   - Message enters `clientInboundChannel`
   - Destination matcher identifies `/app/chat.sendMessage`
   - Routes to `sendMessage()` method via `@MessageMapping`

4. **Controller processing**
   - Method receives the `ChatMessage` object (auto-deserialized from JSON)
   - Method can validate, log, or modify the message
   - Returns the (possibly modified) `ChatMessage`

5. **Broadcasting**
   - Return value is sent to `brokerChannel`
   - Broker routes to `/topic/public`
   - All subscribed clients receive the message via `clientOutboundChannel`

6. **Client reception**
   - Each client's subscription callback fires
   - JavaScript parses the JSON and updates the UI

This entire flow happens asynchronously and efficiently, enabling real-time interaction with minimal latency.

## Creating a More Advanced Controller with Validation

Let's enhance our controller with better validation and error handling:

### Step 3: Enhanced Chat Controller with Validation

Update `ChatController.java`:

```java
package com.example.stomptoyproject;

import org.springframework.messaging.handler.annotation.MessageMapping;
import org.springframework.messaging.handler.annotation.Payload;
import org.springframework.messaging.handler.annotation.SendTo;
import org.springframework.messaging.simp.SimpMessageHeaderAccessor;
import org.springframework.stereotype.Controller;
import org.springframework.util.StringUtils;

@Controller
public class ChatController {

    @MessageMapping("/chat.sendMessage")
    @SendTo("/topic/public")
    public ChatMessage sendMessage(@Payload ChatMessage chatMessage,
                                   SimpMessageHeaderAccessor headerAccessor) {
        // Get username from session (set during join)
        String username = (String) headerAccessor.getSessionAttributes().get("username");
        
        // Validate message
        if (!StringUtils.hasText(chatMessage.getContent())) {
            // Could send error message to specific user here
            return null; // Won't broadcast null
        }
        
        // Ensure sender matches session username (security)
        if (username != null) {
            chatMessage.setSender(username);
        }
        
        // Set timestamp
        chatMessage.setTimestamp(java.time.LocalDateTime.now());
        
        // Log for server-side tracking
        System.out.println("[" + chatMessage.getTimestamp() + "] " + 
                          chatMessage.getSender() + ": " + chatMessage.getContent());
        
        return chatMessage;
    }

    @MessageMapping("/chat.addUser")
    @SendTo("/topic/public")
    public ChatMessage addUser(@Payload ChatMessage chatMessage,
                               SimpMessageHeaderAccessor headerAccessor) {
        // Validate username
        String username = chatMessage.getSender();
        if (!StringUtils.hasText(username)) {
            username = "Anonymous";
        }
        
        // Trim and sanitize username
        username = username.trim();
        if (username.length() > 20) {
            username = username.substring(0, 20);
        }
        
        // Store username in session
        headerAccessor.getSessionAttributes().put("username", username);
        
        // Update message with sanitized username
        chatMessage.setSender(username);
        chatMessage.setType(ChatMessage.MessageType.JOIN);
        chatMessage.setContent(username + " joined!");
        chatMessage.setTimestamp(java.time.LocalDateTime.now());
        
        System.out.println("User " + username + " joined the chat");
        
        return chatMessage;
    }
}
```

Key improvements:
- **Validation**: Checks for empty messages and invalid usernames
- **Security**: Uses session-stored username instead of trusting client-provided values
- **Sanitization**: Trims and limits username length
- **Logging**: Comprehensive server-side logging for debugging and monitoring

## Tracking Active Users

Let's add functionality to track and broadcast the number of active users:

### Step 4: Create a User Tracking Service

Create `ChatService.java`:

```java
package com.example.stomptoyproject;

import org.springframework.context.event.EventListener;
import org.springframework.messaging.simp.SimpMessagingTemplate;
import org.springframework.messaging.simp.stomp.StompHeaderAccessor;
import org.springframework.stereotype.Service;
import org.springframework.web.socket.messaging.SessionConnectedEvent;
import org.springframework.web.socket.messaging.SessionDisconnectEvent;

import java.util.concurrent.atomic.AtomicInteger;

@Service
public class ChatService {

    private final SimpMessagingTemplate messagingTemplate;
    private final AtomicInteger userCount = new AtomicInteger(0);

    public ChatService(SimpMessagingTemplate messagingTemplate) {
        this.messagingTemplate = messagingTemplate;
    }

    @EventListener
    public void handleWebSocketConnectListener(SessionConnectedEvent event) {
        int count = userCount.incrementAndGet();
        broadcastUserCount(count);
        System.out.println("User connected. Total users: " + count);
    }

    @EventListener
    public void handleWebSocketDisconnectListener(SessionDisconnectEvent event) {
        StompHeaderAccessor headerAccessor = StompHeaderAccessor.wrap(event.getMessage());
        String username = (String) headerAccessor.getSessionAttributes().get("username");
        
        int count = userCount.decrementAndGet();
        broadcastUserCount(count);
        
        if (username != null) {
            // Broadcast leave message
            ChatMessage chatMessage = new ChatMessage();
            chatMessage.setType(ChatMessage.MessageType.LEAVE);
            chatMessage.setSender(username);
            chatMessage.setContent(username + " left the chat");
            messagingTemplate.convertAndSend("/topic/public", chatMessage);
        }
        
        System.out.println("User disconnected. Total users: " + count);
    }

    private void broadcastUserCount(int count) {
        messagingTemplate.convertAndSend("/topic/userCount", count);
    }

    public int getCurrentUserCount() {
        return userCount.get();
    }
}
```

This service uses Spring's WebSocket event listeners:

**@EventListener**: Marks methods that handle Spring application events.

**SessionConnectedEvent**: Fired when a WebSocket session is established. We increment our user counter here.

**SessionDisconnectEvent**: Fired when a session ends. We decrement the counter and optionally broadcast a "user left" message.

**AtomicInteger**: Thread-safe counter for tracking users across concurrent connections.

## Creating the Chat HTML Client

Now let's build a comprehensive chat client with a modern UI.

### Step 5: Create the Chat Client

Create `chat.html` in `src/main/resources/static/`:

```html
<!DOCTYPE html>
<html>
<head>
    <title>STOMP Chat Room</title>
    <script src="https://cdn.jsdelivr.net/npm/sockjs-client@1/dist/sockjs.min.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/@stomp/stompjs@7/bundles/stomp.umd.min.js"></script>
    <style>
        * {
            margin: 0;
            padding: 0;
            box-sizing: border-box;
        }

        body {
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            height: 100vh;
            display: flex;
            justify-content: center;
            align-items: center;
        }

        .chat-container {
            width: 90%;
            max-width: 800px;
            height: 90vh;
            background: white;
            border-radius: 10px;
            box-shadow: 0 10px 40px rgba(0,0,0,0.2);
            display: flex;
            flex-direction: column;
            overflow: hidden;
        }

        .chat-header {
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            color: white;
            padding: 20px;
            display: flex;
            justify-content: space-between;
            align-items: center;
        }

        .chat-header h1 {
            font-size: 24px;
        }

        .user-count {
            background: rgba(255,255,255,0.2);
            padding: 5px 15px;
            border-radius: 20px;
            font-size: 14px;
        }

        .join-section {
            padding: 30px;
            text-align: center;
            background: #f8f9fa;
        }

        .join-section input {
            padding: 12px;
            font-size: 16px;
            border: 2px solid #ddd;
            border-radius: 5px;
            width: 300px;
            margin-right: 10px;
        }

        .join-section button {
            padding: 12px 30px;
            font-size: 16px;
            background: #667eea;
            color: white;
            border: none;
            border-radius: 5px;
            cursor: pointer;
        }

        .join-section button:hover {
            background: #5568d3;
        }

        .messages-container {
            flex: 1;
            overflow-y: auto;
            padding: 20px;
            background: #f8f9fa;
        }

        .message {
            margin-bottom: 15px;
            animation: fadeIn 0.3s;
        }

        @keyframes fadeIn {
            from { opacity: 0; transform: translateY(10px); }
            to { opacity: 1; transform: translateY(0); }
        }

        .message-header {
            font-size: 12px;
            color: #666;
            margin-bottom: 5px;
        }

        .message-content {
            display: inline-block;
            padding: 10px 15px;
            border-radius: 18px;
            max-width: 70%;
            word-wrap: break-word;
        }

        .message.own .message-content {
            background: #667eea;
            color: white;
            margin-left: auto;
            display: block;
        }

        .message.other .message-content {
            background: white;
            color: #333;
            border: 1px solid #ddd;
        }

        .message.system .message-content {
            background: #fff3cd;
            color: #856404;
            border: 1px solid #ffc107;
            text-align: center;
            width: 100%;
            max-width: 100%;
            font-style: italic;
        }

        .input-section {
            padding: 20px;
            background: white;
            border-top: 1px solid #ddd;
            display: none;
        }

        .input-section.active {
            display: block;
        }

        .input-container {
            display: flex;
            gap: 10px;
        }

        .input-container input {
            flex: 1;
            padding: 12px;
            font-size: 16px;
            border: 2px solid #ddd;
            border-radius: 5px;
        }

        .input-container button {
            padding: 12px 30px;
            font-size: 16px;
            background: #667eea;
            color: white;
            border: none;
            border-radius: 5px;
            cursor: pointer;
        }

        .input-container button:hover {
            background: #5568d3;
        }

        .input-container button:disabled {
            background: #ccc;
            cursor: not-allowed;
        }
    </style>
</head>
<body>
    <div class="chat-container">
        <div class="chat-header">
            <h1>💬 STOMP Chat Room</h1>
            <div class="user-count">Users: <span id="userCount">0</span></div>
        </div>

        <div class="join-section" id="joinSection">
            <h2>Enter your username to join</h2>
            <input type="text" id="usernameInput" placeholder="Your username..." maxlength="20" />
            <button onclick="joinChat()">Join Chat</button>
        </div>

        <div class="messages-container" id="messagesContainer"></div>

        <div class="input-section" id="inputSection">
            <div class="input-container">
                <input type="text" id="messageInput" placeholder="Type your message..." />
                <button id="sendButton" onclick="sendMessage()">Send</button>
            </div>
        </div>
    </div>

    <script>
        let stompClient = null;
        let currentUsername = null;

        function joinChat() {
            const usernameInput = document.getElementById('usernameInput');
            const username = usernameInput.value.trim();

            if (!username) {
                alert('Please enter a username');
                return;
            }

            currentUsername = username;

            // Connect to WebSocket
            const socket = new SockJS('/gs-guide-websocket');
            stompClient = StompJs.Stomp.over(socket);

            stompClient.debug = function(str) {
                console.log(str);
            };

            stompClient.connect({}, function(frame) {
                console.log('Connected: ' + frame);

                // Hide join section, show chat
                document.getElementById('joinSection').style.display = 'none';
                document.getElementById('inputSection').classList.add('active');

                // Subscribe to public messages
                stompClient.subscribe('/topic/public', function(message) {
                    const chatMessage = JSON.parse(message.body);
                    displayMessage(chatMessage);
                });

                // Subscribe to user count updates
                stompClient.subscribe('/topic/userCount', function(message) {
                    const count = parseInt(message.body);
                    document.getElementById('userCount').textContent = count;
                });

                // Notify server that user joined
                const joinMessage = {
                    type: 'JOIN',
                    sender: username,
                    content: ''
                };
                stompClient.send('/app/chat.addUser', {}, JSON.stringify(joinMessage));
            }, function(error) {
                console.error('Connection error:', error);
                alert('Failed to connect to chat server');
            });
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

        function displayMessage(chatMessage) {
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

            const timestamp = new Date().toLocaleTimeString();
            messageDiv.innerHTML = `
                <div class="message-header">${chatMessage.sender} • ${timestamp}</div>
                <div class="message-content">${escapeHtml(chatMessage.content)}</div>
            `;

            messagesContainer.appendChild(messageDiv);
            messagesContainer.scrollTop = messagesContainer.scrollHeight;
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

        // Allow Enter key to send message
        document.getElementById('messageInput').addEventListener('keypress', function(e) {
            if (e.key === 'Enter') {
                sendMessage();
            }
        });

        document.getElementById('usernameInput').addEventListener('keypress', function(e) {
            if (e.key === 'Enter') {
                joinChat();
            }
        });
    </script>
</body>
</html>
```

## Understanding the Client Implementation

**Connection Flow**:
1. User enters username and clicks "Join Chat"
2. Client establishes WebSocket connection
3. Subscribes to `/topic/public` for messages
4. Subscribes to `/topic/userCount` for user count updates
5. Sends a JOIN message to `/app/chat.addUser`

**Message Sending**:
- User types message and clicks "Send" (or presses Enter)
- Client constructs a `ChatMessage` object
- Sends it to `/app/chat.sendMessage` as JSON
- Server broadcasts it to all subscribers
- Client receives it via subscription and displays it

**UI Styling**:
- Own messages appear on the right (blue)
- Other users' messages on the left (white)
- System messages (join/leave) are centered and styled differently
- HTML escaping prevents XSS attacks

## Testing the Chat Application

1. Start your Spring Boot application
2. Open `http://localhost:8080/chat.html` in multiple browser tabs/windows
3. Enter different usernames in each tab
4. Send messages and observe them appearing in all tabs in real-time
5. Close a tab and see the "user left" notification in other tabs

## Advanced Features: Message History (Optional)

To add message history, you could modify the controller:

```java
@Service
public class MessageHistoryService {
    private final List<ChatMessage> messageHistory = new ArrayList<>();

    public void addMessage(ChatMessage message) {
        synchronized (messageHistory) {
            messageHistory.add(message);
            // Keep only last 100 messages
            if (messageHistory.size() > 100) {
                messageHistory.remove(0);
            }
        }
    }

    public List<ChatMessage> getRecentMessages(int count) {
        synchronized (messageHistory) {
            int start = Math.max(0, messageHistory.size() - count);
            return new ArrayList<>(messageHistory.subList(start, messageHistory.size()));
        }
    }
}
```

Then use `@SubscribeMapping` (covered in Article 5) to send history to new users when they join.

## Complete Test Client

Here's a minimal test client:

```html
<!DOCTYPE html>
<html>
<head>
    <title>Chat Test</title>
    <script src="https://cdn.jsdelivr.net/npm/sockjs-client@1/dist/sockjs.min.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/@stomp/stompjs@7/bundles/stomp.umd.min.js"></script>
</head>
<body>
    <input type="text" id="username" placeholder="Username" />
    <button onclick="join()">Join</button>
    <input type="text" id="message" placeholder="Message" />
    <button onclick="send()">Send</button>
    <div id="messages"></div>

    <script>
        let client = null;
        let username = null;

        function join() {
            username = document.getElementById('username').value;
            const socket = new SockJS('http://localhost:8080/gs-guide-websocket');
            client = StompJs.Stomp.over(socket);
            client.connect({}, () => {
                client.subscribe('/topic/public', (msg) => {
                    const chat = JSON.parse(msg.body);
                    log(chat.sender + ': ' + chat.content);
                });
                client.send('/app/chat.addUser', {}, JSON.stringify({
                    type: 'JOIN',
                    sender: username,
                    content: ''
                }));
            });
        }

        function send() {
            if (client && username) {
                client.send('/app/chat.sendMessage', {}, JSON.stringify({
                    type: 'CHAT',
                    sender: username,
                    content: document.getElementById('message').value
                }));
            }
        }

        function log(msg) {
            document.getElementById('messages').innerHTML += '<p>' + msg + '</p>';
        }
    </script>
</body>
</html>
```

## What We've Accomplished

In this article, we've:
1. Built a complete multi-user chat application
2. Implemented two-way communication with client sending and server broadcasting
3. Added user tracking and session management
4. Created a modern, responsive chat UI
5. Demonstrated the complete STOMP message flow from client to server to all clients

## Next Steps

You now have a working chat application! This pattern—receiving messages from clients and broadcasting to all subscribers—is the foundation for many collaborative applications.

In our next article, we'll explore SockJS fallbacks, which ensure your WebSocket applications work across all browsers and network configurations, including environments with restrictive proxies that block WebSocket connections.
