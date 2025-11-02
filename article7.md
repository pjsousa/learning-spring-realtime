# Differentiate Clients: Sending Private Messages using User Destinations (/user/)

## Introduction

In our previous articles, we've implemented public broadcasting where messages go to all subscribers. But real applications need targeted messaging: sending notifications to specific users, delivering private messages in chat applications, or providing personalized data updates.

Spring's User Destinations feature solves this elegantly. Instead of manually tracking which WebSocket session belongs to which user, you can use a generic destination like `/user/queue/notifications`, and Spring automatically translates it to a user-specific, session-specific address. On the server side, you simply call `SimpMessagingTemplate.convertAndSendToUser(username, destination, payload)`, and Spring handles the routing.

In this article, we'll implement a private messaging system where users can send direct messages to each other. We'll configure user destinations, learn how Spring translates generic user destinations to session-specific ones, and build a complete private messaging application.

## Understanding User Destinations

User destinations work through a translation mechanism:

**Client Subscription**: A client subscribes to `/user/queue/private-messages`. This looks like a broadcast destination, but Spring recognizes the `/user/` prefix.

**Server Translation**: Spring translates `/user/queue/private-messages` to a session-specific destination like `/queue/private-messages-user123` for each user's session.

**Server Sending**: When the server calls `convertAndSendToUser("alice", "/queue/private-messages", message)`, Spring:
1. Finds all sessions for user "alice"
2. Translates the destination to the session-specific address
3. Delivers the message only to alice's sessions

**Multiple Sessions**: If a user has multiple active sessions (e.g., multiple browser tabs), they all receive the message. This is often desirable for notifications.

## Project Setup

We'll build upon the authenticated chat application from Article 6. Ensure you have:
- Spring Security configured with user authentication
- `SimpMessagingTemplate` available (automatically provided by Spring)
- Understanding of Principal injection from Article 6

## Configuring User Destinations

First, we need to configure the user destination prefix in our WebSocket configuration.

### Step 1: Update WebSocket Configuration

Update `WebSocketConfig.java`:

```java
package com.example.stomptoyproject;

import org.springframework.context.annotation.Configuration;
import org.springframework.messaging.simp.config.MessageBrokerRegistry;
import org.springframework.web.socket.config.annotation.EnableWebSocketMessageBroker;
import org.springframework.web.socket.config.annotation.StompEndpointRegistry;
import org.springframework.web.socket.config.annotation.WebSocketMessageBrokerConfigurer;

@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {

    @Override
    public void configureMessageBroker(MessageBrokerRegistry config) {
        config.enableSimpleBroker("/topic", "/queue");
        config.setApplicationDestinationPrefixes("/app");
        
        // Configure user destination prefix
        // This tells Spring that destinations starting with /user/
        // should be translated to user-specific destinations
        config.setUserDestinationPrefix("/user");
    }

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/gs-guide-websocket")
                .setAllowedOriginPatterns("*")
                .withSockJS();
    }
}
```

The key addition is `config.setUserDestinationPrefix("/user")`. This tells Spring:
- Destinations starting with `/user/` are user-specific
- Spring should translate them to session-specific addresses
- The broker should handle these translated destinations

## Creating Private Message Models

Let's create models for private messaging.

### Step 2: Create Private Message Model

Create `PrivateMessage.java`:

```java
package com.example.stomptoyproject;

import java.time.LocalDateTime;

public class PrivateMessage {
    private String from;
    private String to;
    private String content;
    private LocalDateTime timestamp;
    private boolean read;

    public PrivateMessage() {
        this.timestamp = LocalDateTime.now();
        this.read = false;
    }

    public PrivateMessage(String from, String to, String content) {
        this.from = from;
        this.to = to;
        this.content = content;
        this.timestamp = LocalDateTime.now();
        this.read = false;
    }

    // Getters and setters
    public String getFrom() {
        return from;
    }

    public void setFrom(String from) {
        this.from = from;
    }

    public String getTo() {
        return to;
    }

    public void setTo(String to) {
        this.to = to;
    }

    public String getContent() {
        return content;
    }

    public void setContent(String content) {
        this.content = content;
    }

    public LocalDateTime getTimestamp() {
        return timestamp;
    }

    public void setTimestamp(LocalDateTime timestamp) {
        this.timestamp = timestamp;
    }

    public boolean isRead() {
        return read;
    }

    public void setRead(boolean read) {
        this.read = read;
    }
}
```

## Implementing Private Message Controller

Now let's create a controller that handles private messaging using user destinations.

### Step 3: Create Private Message Controller

Create `PrivateMessageController.java`:

```java
package com.example.stomptoyproject;

import org.springframework.messaging.handler.annotation.MessageMapping;
import org.springframework.messaging.handler.annotation.Payload;
import org.springframework.messaging.simp.SimpMessagingTemplate;
import org.springframework.messaging.simp.annotation.SendToUser;
import org.springframework.security.core.Authentication;
import org.springframework.stereotype.Controller;
import org.springframework.util.StringUtils;

import java.security.Principal;

@Controller
public class PrivateMessageController {

    private final SimpMessagingTemplate messagingTemplate;

    public PrivateMessageController(SimpMessagingTemplate messagingTemplate) {
        this.messagingTemplate = messagingTemplate;
    }

    @MessageMapping("/private.send")
    public void sendPrivateMessage(@Payload PrivateMessage privateMessage,
                                    Principal principal) {
        String sender = principal != null ? principal.getName() : "Anonymous";
        privateMessage.setFrom(sender);
        privateMessage.setTimestamp(java.time.LocalDateTime.now());

        // Validate message
        if (!StringUtils.hasText(privateMessage.getTo()) || 
            !StringUtils.hasText(privateMessage.getContent())) {
            // Could send error back to sender
            return;
        }

        // Validate sender is not sending to themselves
        if (sender.equals(privateMessage.getTo())) {
            // Could send error back
            return;
        }

        System.out.println("Private message from " + sender + " to " + 
                          privateMessage.getTo() + ": " + privateMessage.getContent());

        // Send to the recipient using user destination
        // Spring will automatically translate /user/queue/private-messages
        // to the recipient's session-specific destination
        messagingTemplate.convertAndSendToUser(
            privateMessage.getTo(),           // Target username
            "/queue/private-messages",         // Destination (relative to /user prefix)
            privateMessage                     // Message payload
        );

        // Optionally, send confirmation to sender
        PrivateMessage confirmation = new PrivateMessage();
        confirmation.setFrom("System");
        confirmation.setTo(sender);
        confirmation.setContent("Message delivered to " + privateMessage.getTo());
        confirmation.setTimestamp(java.time.LocalDateTime.now());

        messagingTemplate.convertAndSendToUser(
            sender,
            "/queue/private-messages",
            confirmation
        );
    }

    @MessageMapping("/private.getOnlineUsers")
    @SendToUser("/queue/online-users")
    public OnlineUsersList getOnlineUsers(Principal principal) {
        // In a real application, you'd query active sessions
        // For now, return a simple response
        OnlineUsersList list = new OnlineUsersList();
        list.setMessage("Online users list");
        // You would populate this from a session registry
        return list;
    }
}
```

Key concepts:

**convertAndSendToUser()**: This method takes:
- Username: The target user's name (must match the Principal name)
- Destination: A destination relative to `/user` (so `/queue/private-messages` becomes `/user/{username}/queue/private-messages`)
- Payload: The message object (automatically converted to JSON)

**@SendToUser**: Alternative annotation that works similarly to `@SendTo` but sends to a specific user. The method return value is sent to the requesting user.

**Multiple Recipients**: To send to multiple users, call `convertAndSendToUser()` multiple times.

Create the `OnlineUsersList` model:

```java
package com.example.stomptoyproject;

import java.util.List;

public class OnlineUsersList {
    private List<String> users;
    private String message;

    // Getters and setters...
}
```

## Tracking Active Users for Private Messaging

To enable users to see who's online for private messaging, we need to track active users.

### Step 4: Create User Session Registry

Create `UserSessionRegistry.java`:

```java
package com.example.stomptoyproject;

import org.springframework.context.event.EventListener;
import org.springframework.messaging.simp.stomp.StompHeaderAccessor;
import org.springframework.stereotype.Component;
import org.springframework.web.socket.messaging.SessionConnectedEvent;
import org.springframework.web.socket.messaging.SessionDisconnectEvent;

import java.util.Set;
import java.util.concurrent.ConcurrentHashMap;

@Component
public class UserSessionRegistry {

    private final Set<String> onlineUsers = ConcurrentHashMap.newKeySet();

    @EventListener
    public void handleSessionConnected(SessionConnectedEvent event) {
        StompHeaderAccessor headerAccessor = StompHeaderAccessor.wrap(event.getMessage());
        String username = headerAccessor.getUser() != null 
            ? headerAccessor.getUser().getName() 
            : null;
        
        if (username != null) {
            onlineUsers.add(username);
            System.out.println("User " + username + " connected. Online users: " + onlineUsers.size());
        }
    }

    @EventListener
    public void handleSessionDisconnected(SessionDisconnectEvent event) {
        StompHeaderAccessor headerAccessor = StompHeaderAccessor.wrap(event.getMessage());
        String username = headerAccessor.getUser() != null 
            ? headerAccessor.getUser().getName() 
            : null;
        
        if (username != null) {
            onlineUsers.remove(username);
            System.out.println("User " + username + " disconnected. Online users: " + onlineUsers.size());
        }
    }

    public Set<String> getOnlineUsers() {
        return Set.copyOf(onlineUsers);
    }

    public boolean isUserOnline(String username) {
        return onlineUsers.contains(username);
    }
}
```

Update `PrivateMessageController` to use this:

```java
private final UserSessionRegistry userRegistry;

public PrivateMessageController(SimpMessagingTemplate messagingTemplate,
                                UserSessionRegistry userRegistry) {
    this.messagingTemplate = messagingTemplate;
    this.userRegistry = userRegistry;
}

@MessageMapping("/private.send")
public void sendPrivateMessage(@Payload PrivateMessage privateMessage,
                                Principal principal) {
    // ... existing code ...
    
    // Check if recipient is online
    if (!userRegistry.isUserOnline(privateMessage.getTo())) {
        // Send error to sender
        PrivateMessage error = new PrivateMessage();
        error.setFrom("System");
        error.setTo(principal.getName());
        error.setContent("User " + privateMessage.getTo() + " is not online");
        messagingTemplate.convertAndSendToUser(
            principal.getName(),
            "/queue/private-messages",
            error
        );
        return;
    }
    
    // ... rest of send logic ...
}
```

## Broadcasting Online Users List

Let's broadcast the online users list periodically:

### Step 5: Broadcast Online Users

Update `ChatService.java` or create a new component:

```java
@Service
public class OnlineUsersBroadcaster {

    private final SimpMessagingTemplate messagingTemplate;
    private final UserSessionRegistry userRegistry;

    public OnlineUsersBroadcaster(SimpMessagingTemplate messagingTemplate,
                                  UserSessionRegistry userRegistry) {
        this.messagingTemplate = messagingTemplate;
        this.userRegistry = userRegistry;
    }

    @Scheduled(fixedRate = 5000) // Every 5 seconds
    public void broadcastOnlineUsers() {
        Set<String> users = userRegistry.getOnlineUsers();
        
        // Broadcast to all users via /topic
        messagingTemplate.convertAndSend("/topic/online-users", users);
        
        // Or send individually to each user's queue
        // for more privacy
        users.forEach(username -> {
            messagingTemplate.convertAndSendToUser(
                username,
                "/queue/online-users",
                users
            );
        });
    }
}
```

## Creating Private Messaging Client

Now let's build a client that supports both public chat and private messaging.

### Step 6: Create Private Messaging Client

Create `private-chat.html` in `src/main/resources/static/`:

```html
<!DOCTYPE html>
<html>
<head>
    <title>Private Chat</title>
    <script src="https://cdn.jsdelivr.net/npm/sockjs-client@1/dist/sockjs.min.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/@stomp/stompjs@7/bundles/stomp.umd.min.js"></script>
    <style>
        body {
            font-family: Arial, sans-serif;
            margin: 0;
            padding: 20px;
            display: flex;
            height: 100vh;
        }
        .sidebar {
            width: 250px;
            border-right: 1px solid #ddd;
            padding: 20px;
            overflow-y: auto;
        }
        .main-chat {
            flex: 1;
            display: flex;
            flex-direction: column;
            padding: 20px;
        }
        .chat-area {
            flex: 1;
            overflow-y: auto;
            border: 1px solid #ddd;
            padding: 10px;
            margin-bottom: 10px;
        }
        .message {
            margin: 10px 0;
            padding: 8px;
            border-radius: 5px;
        }
        .message.private {
            background-color: #fff3cd;
            border-left: 4px solid #ffc107;
        }
        .message.public {
            background-color: #e7f3ff;
        }
        .user-list {
            list-style: none;
            padding: 0;
        }
        .user-list li {
            padding: 8px;
            margin: 5px 0;
            cursor: pointer;
            border-radius: 4px;
        }
        .user-list li:hover {
            background-color: #f0f0f0;
        }
        .user-list li.active {
            background-color: #007bff;
            color: white;
        }
        .input-area {
            display: flex;
            gap: 10px;
        }
        .input-area input {
            flex: 1;
            padding: 10px;
        }
        .input-area button {
            padding: 10px 20px;
        }
        .tab {
            padding: 10px;
            cursor: pointer;
            border: 1px solid #ddd;
            margin-right: 5px;
            display: inline-block;
        }
        .tab.active {
            background-color: #007bff;
            color: white;
        }
    </style>
</head>
<body>
    <div class="sidebar">
        <h3>Online Users</h3>
        <ul class="user-list" id="userList"></ul>
    </div>

    <div class="main-chat">
        <div>
            <span class="tab active" onclick="switchTab('public')">Public Chat</span>
            <span class="tab" onclick="switchTab('private')" id="privateTab">Private Messages</span>
        </div>

        <div class="chat-area" id="chatArea"></div>

        <div class="input-area">
            <input type="text" id="messageInput" placeholder="Type message..." />
            <input type="text" id="recipientInput" placeholder="Recipient (for private)" style="display:none;" />
            <button onclick="sendMessage()">Send</button>
        </div>
    </div>

    <script>
        let stompClient = null;
        let currentUsername = null;
        let currentTab = 'public';
        let selectedRecipient = null;

        function connect(username) {
            currentUsername = username;
            const socket = new SockJS('/gs-guide-websocket');
            stompClient = StompJs.Stomp.over(socket);

            stompClient.connect({}, function(frame) {
                console.log('Connected: ' + frame);

                // Subscribe to public messages
                stompClient.subscribe('/topic/public', function(message) {
                    const msg = JSON.parse(message.body);
                    displayMessage(msg, 'public');
                });

                // Subscribe to private messages
                // Note: /user/queue/private-messages will be translated
                // to a session-specific destination
                stompClient.subscribe('/user/queue/private-messages', function(message) {
                    const msg = JSON.parse(message.body);
                    displayMessage(msg, 'private');
                });

                // Subscribe to online users updates
                stompClient.subscribe('/topic/online-users', function(message) {
                    const users = JSON.parse(message.body);
                    updateUserList(users);
                });

                // Subscribe to user-specific online users list
                stompClient.subscribe('/user/queue/online-users', function(message) {
                    const users = JSON.parse(message.body);
                    updateUserList(users);
                });

                // Notify server of join
                stompClient.send('/app/chat.addUser', {}, JSON.stringify({
                    type: 'JOIN',
                    sender: currentUsername,
                    content: ''
                }));
            });
        }

        function switchTab(tab) {
            currentTab = tab;
            document.querySelectorAll('.tab').forEach(t => t.classList.remove('active'));
            
            if (tab === 'public') {
                document.getElementById('privateTab').classList.remove('active');
                document.querySelector('.tab').classList.add('active');
                document.getElementById('recipientInput').style.display = 'none';
                document.getElementById('messageInput').placeholder = 'Type message...';
            } else {
                document.getElementById('privateTab').classList.add('active');
                document.querySelector('.tab').classList.remove('active');
                document.getElementById('recipientInput').style.display = 'block';
                document.getElementById('messageInput').placeholder = 'Type private message...';
            }
        }

        function sendMessage() {
            const messageInput = document.getElementById('messageInput');
            const content = messageInput.value.trim();

            if (!content || !stompClient) {
                return;
            }

            if (currentTab === 'private') {
                const recipientInput = document.getElementById('recipientInput');
                const recipient = recipientInput.value.trim();

                if (!recipient) {
                    alert('Please enter a recipient');
                    return;
                }

                // Send private message
                stompClient.send('/app/private.send', {}, JSON.stringify({
                    from: currentUsername,
                    to: recipient,
                    content: content
                }));

                recipientInput.value = '';
            } else {
                // Send public message
                stompClient.send('/app/chat.sendMessage', {}, JSON.stringify({
                    type: 'CHAT',
                    sender: currentUsername,
                    content: content
                }));
            }

            messageInput.value = '';
        }

        function displayMessage(msg, type) {
            const chatArea = document.getElementById('chatArea');
            const messageDiv = document.createElement('div');
            messageDiv.className = 'message ' + type;

            let displayText = '';
            if (type === 'private') {
                displayText = `<strong>From ${msg.from}:</strong> ${msg.content}`;
            } else {
                displayText = `<strong>${msg.sender}:</strong> ${msg.content}`;
            }

            messageDiv.innerHTML = displayText;
            chatArea.appendChild(messageDiv);
            chatArea.scrollTop = chatArea.scrollHeight;
        }

        function updateUserList(users) {
            const userList = document.getElementById('userList');
            userList.innerHTML = '';

            users.forEach(function(username) {
                if (username === currentUsername) {
                    return; // Don't show self
                }

                const li = document.createElement('li');
                li.textContent = username;
                li.onclick = function() {
                    // Select this user for private messaging
                    selectedRecipient = username;
                    document.querySelectorAll('.user-list li').forEach(l => 
                        l.classList.remove('active'));
                    li.classList.add('active');
                    document.getElementById('recipientInput').value = username;
                    switchTab('private');
                };
                userList.appendChild(li);
            });
        }

        // For demo purposes, prompt for username
        // In production, this would come from authentication
        const username = prompt('Enter your username:');
        if (username) {
            connect(username);
        }
    </script>
</body>
</html>
```

## Understanding the Client Subscription

The critical line is:

```javascript
stompClient.subscribe('/user/queue/private-messages', function(message) {
    // Handle private messages
});
```

When the client subscribes to `/user/queue/private-messages`:
1. Spring recognizes the `/user/` prefix
2. Spring translates it to a session-specific destination like `/queue/private-messages-{session-id}`
3. The broker routes messages sent via `convertAndSendToUser()` to this translated destination
4. Only this client receives messages sent to their username

## Testing Private Messaging

1. Start your Spring Boot application
2. Open `http://localhost:8080/private-chat.html` in multiple browser tabs
3. Enter different usernames in each tab
4. Switch to "Private Messages" tab in one tab
5. Select a user from the list and send a private message
6. Verify the message appears only in the recipient's tab

## Advanced: Message Delivery Confirmation

You can implement delivery confirmations:

```java
@MessageMapping("/private.send")
public void sendPrivateMessage(@Payload PrivateMessage privateMessage,
                                Principal principal) {
    // ... send message ...
    
    // Send read receipt request
    messagingTemplate.convertAndSendToUser(
        privateMessage.getTo(),
        "/queue/private-messages",
        new ReadReceiptRequest(messageId)
    );
}
```

## Complete Test Client

Here's a minimal test client for private messaging:

```html
<!DOCTYPE html>
<html>
<head>
    <title>Private Message Test</title>
    <script src="https://cdn.jsdelivr.net/npm/sockjs-client@1/dist/sockjs.min.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/@stomp/stompjs@7/bundles/stomp.umd.min.js"></script>
</head>
<body>
    <h1>Private Message Test</h1>
    <input type="text" id="username" placeholder="Your username" />
    <button onclick="connect()">Connect</button>
    <br><br>
    <input type="text" id="recipient" placeholder="Recipient username" />
    <input type="text" id="message" placeholder="Message" />
    <button onclick="sendPrivate()">Send Private</button>
    <div id="output"></div>

    <script>
        let client = null;
        let username = null;

        function connect() {
            username = document.getElementById('username').value;
            const socket = new SockJS('http://localhost:8080/gs-guide-websocket');
            client = StompJs.Stomp.over(socket);
            
            client.connect({}, () => {
                log('Connected as: ' + username);
                
                // Subscribe to private messages
                client.subscribe('/user/queue/private-messages', (msg) => {
                    const pm = JSON.parse(msg.body);
                    log(`📨 Private from ${pm.from}: ${pm.content}`);
                });
            });
        }

        function sendPrivate() {
            if (client && username) {
                const recipient = document.getElementById('recipient').value;
                const message = document.getElementById('message').value;
                
                client.send('/app/private.send', {}, JSON.stringify({
                    from: username,
                    to: recipient,
                    content: message
                }));
                
                log(`Sent to ${recipient}: ${message}`);
            }
        }

        function log(msg) {
            document.getElementById('output').innerHTML += '<p>' + msg + '</p>';
        }
    </script>
</body>
</html>
```

## What We've Accomplished

In this article, we've:
1. Configured user destinations in Spring Boot
2. Learned how Spring translates user destinations to session-specific addresses
3. Implemented private messaging using `convertAndSendToUser()`
4. Built a complete private messaging client
5. Tracked online users for messaging

## Next Steps

You now have the ability to send targeted messages to specific users! This pattern is essential for notifications, private messaging, and personalized updates.

In our next article, we'll tackle a challenging real-time scenario: implementing a collaborative mouse cursor tracker. This requires high-frequency updates, efficient message processing, and managing transient state—perfect for demonstrating the power of STOMP for real-time collaboration.
