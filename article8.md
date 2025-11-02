# High-Frequency State Sync: Implementing a Live Collaborative Mouse Cursor

## Introduction

In our previous articles, we've built chat applications, stock tickers, and private messaging systems. These handle relatively low-frequency updates (messages every few seconds or when users type). But real-time collaboration applications like Google Docs, Figma, or collaborative whiteboards require a different approach: they need to sync state at very high frequencies with minimal latency.

Mouse cursor tracking is an excellent example of this challenge. When multiple users collaborate on a document, each user's mouse cursor position needs to be broadcast to all other users many times per second. This creates a flood of messages that must be processed efficiently without overwhelming the server or clients.

In this article, we'll build a collaborative mouse cursor tracker that demonstrates high-frequency real-time state synchronization. We'll learn how to optimize message payloads, throttle client sends, handle transient state efficiently, and maintain smooth animations despite network latency.

## Understanding the Challenge

High-frequency updates present several challenges:

**Message Volume**: Mouse movements can generate 50-100 events per second per user. With 10 users, that's 500-1000 messages per second flowing through the system.

**Latency Requirements**: Cursor positions should feel immediate. Even 100ms delay makes the experience feel sluggish.

**Payload Optimization**: Each message should contain only essential data. Sending unnecessary information wastes bandwidth and processing time.

**Client-Side Throttling**: Clients shouldn't send every mouse movement event. We need to intelligently throttle while maintaining smooth visuals.

**State Management**: Cursor positions are transient—they don't need persistence, but they need efficient in-memory handling.

## Project Setup

We'll build a new application focused on cursor tracking. Ensure you have:
- Spring Boot 3.1.x or later
- `spring-boot-starter-websocket` dependency
- Basic WebSocket configuration from Article 1

## Creating the Cursor Position Model

Let's create a minimal model for cursor positions.

### Step 1: Create Cursor Position Model

Create `CursorPosition.java`:

```java
package com.example.stomptoyproject;

import java.time.Instant;

public class CursorPosition {
    private String userId;
    private String userName;
    private double x;
    private double y;
    private long timestamp;

    public CursorPosition() {
        this.timestamp = Instant.now().toEpochMilli();
    }

    public CursorPosition(String userId, String userName, double x, double y) {
        this.userId = userId;
        this.userName = userName;
        this.x = x;
        this.y = y;
        this.timestamp = Instant.now().toEpochMilli();
    }

    // Getters and setters
    public String getUserId() {
        return userId;
    }

    public void setUserId(String userId) {
        this.userId = userId;
    }

    public String getUserName() {
        return userName;
    }

    public void setUserName(String userName) {
        this.userName = userName;
    }

    public double getX() {
        return x;
    }

    public void setX(double x) {
        this.x = x;
    }

    public double getY() {
        return y;
    }

    public void setY(double y) {
        this.y = y;
    }

    public long getTimestamp() {
        return timestamp;
    }

    public void setTimestamp(long timestamp) {
        this.timestamp = timestamp;
    }
}
```

Notice how minimal this is—just essential coordinates and identity. We're not sending colors, shapes, or other metadata with every update.

## Implementing the Cursor Controller

The controller needs to handle high-frequency updates efficiently.

### Step 2: Create Cursor Controller

Create `CursorController.java`:

```java
package com.example.stomptoyproject;

import org.springframework.messaging.handler.annotation.MessageMapping;
import org.springframework.messaging.handler.annotation.Payload;
import org.springframework.messaging.handler.annotation.SendTo;
import org.springframework.messaging.simp.SimpMessageHeaderAccessor;
import org.springframework.stereotype.Controller;

import java.security.Principal;
import java.util.Map;

@Controller
public class CursorController {

    @MessageMapping("/cursor.move")
    @SendTo("/topic/cursors")
    public CursorPosition updateCursor(@Payload CursorPosition position,
                                       Principal principal,
                                       SimpMessageHeaderAccessor headerAccessor) {
        // Get user identity
        String userId = principal != null ? principal.getName() : "anonymous";
        String userName = userId;
        
        // Try to get username from session if available
        Map<String, Object> sessionAttributes = headerAccessor.getSessionAttributes();
        if (sessionAttributes != null && sessionAttributes.containsKey("username")) {
            userName = (String) sessionAttributes.get("username");
        }

        // Set user information
        position.setUserId(userId);
        position.setUserName(userName != null ? userName : userId);
        
        // Update timestamp
        position.setTimestamp(System.currentTimeMillis());

        // Broadcast to all clients
        // In a high-frequency scenario, you might want to throttle on server side too
        return position;
    }

    @MessageMapping("/cursor.join")
    @SendTo("/topic/cursors")
    public CursorPosition joinSession(@Payload CursorPosition position,
                                     Principal principal,
                                     SimpMessageHeaderAccessor headerAccessor) {
        String userId = principal != null ? principal.getName() : "anonymous-" + System.currentTimeMillis();
        String userName = userId;
        
        position.setUserId(userId);
        position.setUserName(userName);
        position.setTimestamp(System.currentTimeMillis());
        
        // Store in session
        headerAccessor.getSessionAttributes().put("userId", userId);
        headerAccessor.getSessionAttributes().put("username", userName);
        
        return position;
    }
}
```

## Optimizing with Message Throttling

For high-frequency updates, we should throttle on the server side to prevent one slow client from overwhelming others.

### Step 3: Create Throttling Interceptor

Create `ThrottlingInterceptor.java`:

```java
package com.example.stomptoyproject;

import org.springframework.messaging.Message;
import org.springframework.messaging.MessageChannel;
import org.springframework.messaging.simp.stomp.StompCommand;
import org.springframework.messaging.simp.stomp.StompHeaderAccessor;
import org.springframework.messaging.support.ChannelInterceptor;
import org.springframework.messaging.support.MessageHeaderAccessor;

import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

@Component
public class ThrottlingInterceptor implements ChannelInterceptor {

    private static final long THROTTLE_INTERVAL_MS = 50; // Max one message per 50ms per user
    private final Map<String, Long> lastMessageTime = new ConcurrentHashMap<>();

    @Override
    public Message<?> preSend(Message<?> message, MessageChannel channel) {
        StompHeaderAccessor accessor = MessageHeaderAccessor.getAccessor(message, StompHeaderAccessor.class);
        
        if (accessor != null && accessor.getCommand() == StompCommand.SEND) {
            String userId = accessor.getUser() != null ? accessor.getUser().getName() : "anonymous";
            String destination = accessor.getDestination();
            
            // Only throttle cursor updates
            if (destination != null && destination.startsWith("/app/cursor")) {
                String key = userId + ":" + destination;
                long now = System.currentTimeMillis();
                Long lastTime = lastMessageTime.get(key);
                
                if (lastTime != null && (now - lastTime) < THROTTLE_INTERVAL_MS) {
                    // Throttle: drop this message
                    return null;
                }
                
                lastMessageTime.put(key, now);
            }
        }
        
        return message;
    }
}
```

Register the interceptor:

```java
@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {
    
    @Autowired
    private ThrottlingInterceptor throttlingInterceptor;

    @Override
    public void configureClientInboundChannel(ChannelRegistration registration) {
        registration.interceptors(throttlingInterceptor);
    }

    // ... rest of configuration ...
}
```

## Creating the Collaborative Cursor Client

The client needs to throttle sends and handle smooth rendering.

### Step 4: Create Cursor Tracking Client

Create `collaborative-cursor.html` in `src/main/resources/static/`:

```html
<!DOCTYPE html>
<html>
<head>
    <title>Collaborative Cursor Tracker</title>
    <script src="https://cdn.jsdelivr.net/npm/sockjs-client@1/dist/sockjs.min.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/@stomp/stompjs@7/bundles/stomp.umd.min.js"></script>
    <style>
        body {
            margin: 0;
            padding: 0;
            overflow: hidden;
            background: #f0f0f0;
            font-family: Arial, sans-serif;
        }
        #canvas {
            width: 100vw;
            height: 100vh;
            cursor: crosshair;
        }
        .cursor {
            position: absolute;
            width: 20px;
            height: 20px;
            pointer-events: none;
            z-index: 1000;
            transition: transform 0.1s linear;
        }
        .cursor::before {
            content: '';
            position: absolute;
            width: 0;
            height: 0;
            border-left: 5px solid transparent;
            border-right: 5px solid transparent;
            border-top: 10px solid;
            top: 0;
            left: 50%;
            transform: translateX(-50%);
        }
        .cursor-label {
            position: absolute;
            top: 15px;
            left: 50%;
            transform: translateX(-50%);
            background: rgba(0,0,0,0.7);
            color: white;
            padding: 2px 6px;
            border-radius: 3px;
            font-size: 10px;
            white-space: nowrap;
        }
        .info-panel {
            position: absolute;
            top: 10px;
            left: 10px;
            background: white;
            padding: 10px;
            border-radius: 5px;
            box-shadow: 0 2px 5px rgba(0,0,0,0.2);
        }
        .login-panel {
            position: fixed;
            top: 50%;
            left: 50%;
            transform: translate(-50%, -50%);
            background: white;
            padding: 30px;
            border-radius: 10px;
            box-shadow: 0 5px 15px rgba(0,0,0,0.3);
            text-align: center;
        }
        .login-panel input {
            padding: 10px;
            margin: 10px;
            width: 200px;
            font-size: 16px;
        }
        .login-panel button {
            padding: 10px 20px;
            font-size: 16px;
            background: #007bff;
            color: white;
            border: none;
            border-radius: 5px;
            cursor: pointer;
        }
    </style>
</head>
<body>
    <div id="loginPanel" class="login-panel">
        <h2>Join Collaborative Session</h2>
        <input type="text" id="usernameInput" placeholder="Enter your name" />
        <br>
        <button onclick="join()">Join</button>
    </div>

    <div id="infoPanel" class="info-panel" style="display:none;">
        <div>Your name: <strong id="yourName"></strong></div>
        <div>Active users: <strong id="userCount">0</strong></div>
    </div>

    <canvas id="canvas"></canvas>

    <script>
        let stompClient = null;
        let currentUser = null;
        let cursors = new Map(); // userId -> cursor element
        let lastSendTime = 0;
        const THROTTLE_MS = 50; // Send max once per 50ms
        let animationFrameId = null;

        // Generate color for user
        function getUserColor(userId) {
            let hash = 0;
            for (let i = 0; i < userId.length; i++) {
                hash = userId.charCodeAt(i) + ((hash << 5) - hash);
            }
            const hue = hash % 360;
            return `hsl(${hue}, 70%, 50%)`;
        }

        function join() {
            const username = document.getElementById('usernameInput').value.trim();
            if (!username) {
                alert('Please enter your name');
                return;
            }

            currentUser = username;
            document.getElementById('yourName').textContent = username;
            document.getElementById('loginPanel').style.display = 'none';
            document.getElementById('infoPanel').style.display = 'block';

            const socket = new SockJS('/gs-guide-websocket');
            stompClient = StompJs.Stomp.over(socket);

            stompClient.connect({}, function(frame) {
                console.log('Connected');

                // Subscribe to cursor updates
                stompClient.subscribe('/topic/cursors', function(message) {
                    const position = JSON.parse(message.body);
                    updateCursor(position);
                });

                // Send join message
                stompClient.send('/app/cursor.join', {}, JSON.stringify({
                    userId: currentUser,
                    userName: currentUser,
                    x: 0,
                    y: 0
                }));

                // Set up mouse tracking
                setupMouseTracking();
            });
        }

        function setupMouseTracking() {
            const canvas = document.getElementById('canvas');
            const ctx = canvas.getContext('2d');
            
            // Set canvas size
            function resizeCanvas() {
                canvas.width = window.innerWidth;
                canvas.height = window.innerHeight;
            }
            resizeCanvas();
            window.addEventListener('resize', resizeCanvas);

            let mouseX = 0, mouseY = 0;

            canvas.addEventListener('mousemove', function(e) {
                mouseX = e.clientX;
                mouseY = e.clientY;

                // Throttle sends
                const now = Date.now();
                if (now - lastSendTime < THROTTLE_MS) {
                    return;
                }
                lastSendTime = now;

                // Send cursor position
                if (stompClient) {
                    stompClient.send('/app/cursor.move', {}, JSON.stringify({
                        userId: currentUser,
                        userName: currentUser,
                        x: mouseX,
                        y: mouseY
                    }));
                }

                // Update own cursor immediately (optimistic update)
                updateLocalCursor(currentUser, mouseX, mouseY);
            });

            // Smooth animation loop
            function animate() {
                // Update cursor positions smoothly
                cursors.forEach((cursorEl, userId) => {
                    // Smooth interpolation happens via CSS transitions
                });
                animationFrameId = requestAnimationFrame(animate);
            }
            animate();
        }

        function updateCursor(position) {
            let cursorEl = cursors.get(position.userId);

            if (!cursorEl) {
                // Create new cursor element
                cursorEl = document.createElement('div');
                cursorEl.className = 'cursor';
                cursorEl.id = 'cursor-' + position.userId;
                
                const color = getUserColor(position.userId);
                cursorEl.style.borderTopColor = color;
                
                const label = document.createElement('div');
                label.className = 'cursor-label';
                label.textContent = position.userName || position.userId;
                label.style.backgroundColor = color;
                cursorEl.appendChild(label);
                
                document.body.appendChild(cursorEl);
                cursors.set(position.userId, cursorEl);

                // Update user count
                updateUserCount();
            }

            // Update position with smooth transition
            // Use transform for better performance
            cursorEl.style.transform = `translate(${position.x - 10}px, ${position.y - 10}px)`;
            cursorEl.style.left = '0';
            cursorEl.style.top = '0';
        }

        function updateLocalCursor(userId, x, y) {
            const cursorEl = cursors.get(userId);
            if (cursorEl) {
                cursorEl.style.transform = `translate(${x - 10}px, ${y - 10}px)`;
            }
        }

        function updateUserCount() {
            document.getElementById('userCount').textContent = cursors.size;
        }

        // Handle disconnect
        window.addEventListener('beforeunload', function() {
            if (stompClient) {
                stompClient.disconnect();
            }
            if (animationFrameId) {
                cancelAnimationFrame(animationFrameId);
            }
        });
    </script>
</body>
</html>
```

## Optimizations Explained

**Client-Side Throttling**: We only send cursor positions every 50ms, even if the mouse moves more frequently. This reduces message volume by 80-90% while maintaining smooth visuals.

**Optimistic Updates**: We update our own cursor immediately without waiting for server confirmation. This eliminates perceived latency for the local user.

**CSS Transitions**: Using CSS `transform` with transitions provides smooth animations without JavaScript interpolation, leveraging GPU acceleration.

**RequestAnimationFrame**: We use `requestAnimationFrame` for efficient animation loops that sync with the browser's refresh rate.

## Handling Disconnections

We should clean up cursors when users disconnect.

### Step 5: Track and Clean Up Disconnections

Update `CursorController.java`:

```java
@EventListener
public void handleSessionDisconnect(SessionDisconnectEvent event) {
    StompHeaderAccessor headerAccessor = StompHeaderAccessor.wrap(event.getMessage());
    String userId = headerAccessor.getUser() != null 
        ? headerAccessor.getUser().getName() 
        : (String) headerAccessor.getSessionAttributes().get("userId");
    
    if (userId != null) {
        // Send removal message
        CursorPosition removeMsg = new CursorPosition();
        removeMsg.setUserId(userId);
        removeMsg.setX(-1); // Signal for removal
        removeMsg.setY(-1);
        
        messagingTemplate.convertAndSend("/topic/cursors", removeMsg);
    }
}
```

Update the client to handle removals:

```javascript
stompClient.subscribe('/topic/cursors', function(message) {
    const position = JSON.parse(message.body);
    
    // Check for removal signal
    if (position.x < 0 || position.y < 0) {
        removeCursor(position.userId);
    } else {
        updateCursor(position);
    }
});

function removeCursor(userId) {
    const cursorEl = cursors.get(userId);
    if (cursorEl) {
        cursorEl.remove();
        cursors.delete(userId);
        updateUserCount();
    }
}
```

## Performance Monitoring

For high-frequency applications, monitoring is crucial.

### Step 6: Add Performance Metrics

Create a simple metrics component:

```java
@Component
public class CursorMetrics {
    private final AtomicLong messagesReceived = new AtomicLong(0);
    private final AtomicLong messagesSent = new AtomicLong(0);

    public void incrementReceived() {
        messagesReceived.incrementAndGet();
    }

    public void incrementSent() {
        messagesSent.incrementAndGet();
    }

    @Scheduled(fixedRate = 5000)
    public void logMetrics() {
        long received = messagesReceived.getAndSet(0);
        long sent = messagesSent.getAndSet(0);
        System.out.println(String.format(
            "Cursor metrics - Received: %d/sec, Sent: %d/sec",
            received / 5, sent / 5
        ));
    }
}
```

## Complete Test Client

Here's a minimal test client:

```html
<!DOCTYPE html>
<html>
<head>
    <title>Cursor Test</title>
    <script src="https://cdn.jsdelivr.net/npm/sockjs-client@1/dist/sockjs.min.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/@stomp/stompjs@7/bundles/stomp.umd.min.js"></script>
    <style>
        body { margin: 0; padding: 0; overflow: hidden; }
        #canvas { width: 100vw; height: 100vh; background: #f0f0f0; }
    </style>
</head>
<body>
    <canvas id="canvas"></canvas>
    <script>
        const username = prompt('Your name:');
        const socket = new SockJS('http://localhost:8080/gs-guide-websocket');
        const client = StompJs.Stomp.over(socket);
        const cursors = {};
        let lastSend = 0;

        client.connect({}, () => {
            client.subscribe('/topic/cursors', (msg) => {
                const pos = JSON.parse(msg.body);
                if (!cursors[pos.userId]) {
                    const el = document.createElement('div');
                    el.style.position = 'absolute';
                    el.style.width = '10px';
                    el.style.height = '10px';
                    el.style.background = 'red';
                    el.style.borderRadius = '50%';
                    document.body.appendChild(el);
                    cursors[pos.userId] = el;
                }
                cursors[pos.userId].style.left = pos.x + 'px';
                cursors[pos.userId].style.top = pos.y + 'px';
            });

            document.getElementById('canvas').addEventListener('mousemove', (e) => {
                if (Date.now() - lastSend > 50) {
                    client.send('/app/cursor.move', {}, JSON.stringify({
                        userId: username,
                        x: e.clientX,
                        y: e.clientY
                    }));
                    lastSend = Date.now();
                }
            });
        });
    </script>
</body>
</html>
```

## What We've Accomplished

In this article, we've:
1. Built a high-frequency cursor tracking system
2. Implemented client-side and server-side throttling
3. Optimized message payloads for minimal size
4. Created smooth visual rendering with CSS transforms
5. Handled user disconnections gracefully
6. Learned performance optimization techniques for real-time applications

## Next Steps

You've mastered high-frequency real-time updates! This pattern applies to collaborative editing, live gaming, and any scenario requiring frequent state synchronization.

In our next article, we'll dive deeper into performance tuning and monitoring, exploring Spring's internal message channels, thread pool configuration, and tools for diagnosing performance bottlenecks in production WebSocket applications.
