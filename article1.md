# The STOMP Startup: Connecting Spring Boot and Plain HTML for Simple Real-Time Messaging

## Introduction

Welcome to the first article in our comprehensive series on implementing STOMP (Simple Text Oriented Messaging Protocol) over WebSocket using Spring Boot and plain HTML clients. In this foundational article, we'll establish the groundwork for real-time, bidirectional communication between a Spring Boot backend and browser-based clients.

WebSocket technology enables full-duplex communication channels over a single TCP connection, eliminating the overhead of HTTP request-response cycles. However, raw WebSocket is quite low-level. STOMP provides a higher-level messaging protocol that operates on top of WebSocket, offering familiar messaging concepts like destinations, subscriptions, and message types.

By the end of this article, you'll have a working Spring Boot application with a STOMP endpoint that can exchange simple messages with a basic HTML client. We'll build a "ping-pong" demonstration system that validates your setup and demonstrates the core communication flow.

## Understanding the Core Concepts

Before diving into implementation, let's clarify a few key concepts:

**WebSocket**: A communication protocol that provides full-duplex communication channels over a single TCP connection. Unlike HTTP, which follows a request-response pattern, WebSocket allows both client and server to initiate data transfer at any time.

**STOMP**: An application-level messaging protocol that operates over WebSocket. STOMP defines a frame-based protocol with commands like CONNECT, SEND, SUBSCRIBE, and DISCONNECT. It provides a structured way to handle messaging without dealing with raw WebSocket frames.

**Message Broker**: In Spring's WebSocket architecture, a message broker routes messages between clients. The simple in-memory broker (which we'll use initially) handles destination-based routing within a single application instance.

**Application Prefixes**: Spring uses prefixes to distinguish between messages sent to application controllers (`/app`) and messages sent directly to the broker (`/topic`, `/queue`).

## Project Setup

Let's start by creating a new Spring Boot project. You can use Spring Initializr (start.spring.io) or create it manually. For this series, we'll assume you're using Maven as the build tool.

### Step 1: Create the Spring Boot Project

Create a new Spring Boot project with the following dependencies:
- Spring Web
- Spring WebSocket
- Spring Messaging

If you're using Spring Initializr, select:
- Project: Maven
- Language: Java
- Spring Boot: 3.1.x or later
- Dependencies: Web, WebSocket

### Step 2: Maven Dependencies

Your `pom.xml` should include these dependencies:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema/instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 
         https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.1.5</version>
        <relativePath/>
    </parent>
    <groupId>com.example</groupId>
    <artifactId>stomp-toy-project</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>stomp-toy-project</name>
    <description>STOMP over WebSocket with Spring Boot</description>
    <properties>
        <java.version>17</java.version>
    </properties>
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
    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```

The `spring-boot-starter-websocket` dependency includes both `spring-websocket` and `spring-messaging`, which are essential for STOMP support.

## Configuring the WebSocket Message Broker

Now, let's create the configuration class that enables WebSocket message broker functionality. This is the heart of our STOMP setup.

### Step 3: Create the WebSocket Configuration

Create a new Java class called `WebSocketConfig.java` in your main package:

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
        // Enable a simple in-memory message broker to carry messages
        // back to the client on destinations prefixed with "/topic"
        config.enableSimpleBroker("/topic");
        
        // Messages bound for methods annotated with @MessageMapping
        // should be prefixed with "/app"
        config.setApplicationDestinationPrefixes("/app");
    }

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        // Register "/gs-guide-websocket" endpoint, enabling SockJS fallback options
        // so that alternate transports may be used if WebSocket is not available
        registry.addEndpoint("/gs-guide-websocket")
                .setAllowedOriginPatterns("*")
                .withSockJS();
    }
}
```

Let's break down what this configuration does:

**@EnableWebSocketMessageBroker**: This annotation enables WebSocket message handling, backed by a message broker. It imports the necessary Spring configuration to make WebSocket and STOMP work.

**configureMessageBroker()**: This method configures the message broker:
- `enableSimpleBroker("/topic")`: Creates an in-memory message broker that routes messages to destinations prefixed with `/topic`. Clients subscribe to topics like `/topic/greetings`.
- `setApplicationDestinationPrefixes("/app")`: Messages sent from clients to the server that target controller methods annotated with `@MessageMapping` should be prefixed with `/app`. For example, sending to `/app/hello` will route to a method annotated with `@MessageMapping("/hello")`.

**registerStompEndpoints()**: This method registers the STOMP endpoint:
- `addEndpoint("/gs-guide-websocket")`: Creates the WebSocket endpoint that clients will connect to.
- `setAllowedOriginPatterns("*")`: Allows connections from any origin (useful for development; in production, specify actual origins).
- `withSockJS()`: Enables SockJS fallback options for browsers that don't support WebSocket natively. We'll explore SockJS in more detail in Article 4.

## Creating a Simple Message Controller

Now let's create a controller that handles messages from clients and can send responses.

### Step 4: Create the Message Controller

Create a new class called `MessageController.java`:

```java
package com.example.stomptoyproject;

import org.springframework.messaging.handler.annotation.MessageMapping;
import org.springframework.messaging.handler.annotation.SendTo;
import org.springframework.stereotype.Controller;

@Controller
public class MessageController {

    @MessageMapping("/ping")
    @SendTo("/topic/pong")
    public String ping(String message) {
        System.out.println("Received: " + message);
        return "Pong: " + message;
    }
}
```

This controller demonstrates the basic STOMP message flow:

**@Controller**: Marks this class as a Spring controller that can handle WebSocket messages.

**@MessageMapping("/ping")**: Maps incoming messages sent to `/app/ping` (remember the `/app` prefix from configuration) to this method. The method receives the message payload.

**@SendTo("/topic/pong")**: After processing, the return value is automatically sent to all clients subscribed to `/topic/pong`.

The flow works like this:
1. Client sends a message to `/app/ping`
2. Spring routes it to the `ping()` method
3. The method processes the message and returns a response
4. Spring automatically broadcasts the response to `/topic/pong`
5. All subscribed clients receive the message

## Understanding the Message Flow Architecture

Spring's WebSocket messaging architecture uses several internal channels:

1. **clientInboundChannel**: Handles messages coming from WebSocket clients
2. **brokerChannel**: Routes messages to the message broker
3. **clientOutboundChannel**: Handles messages going to WebSocket clients

When a client sends a message to `/app/ping`:
- It enters through `clientInboundChannel`
- Spring routes it to the `@MessageMapping` method based on the destination
- The method's return value goes to `brokerChannel`
- The broker routes it to `clientOutboundChannel`
- Finally, it's delivered to subscribed clients

This architecture provides a clean separation of concerns and enables advanced features like security, message transformation, and custom routing.

## Creating the HTML Test Client

Now let's create a simple HTML client that connects to our STOMP endpoint and demonstrates the ping-pong functionality.

### Step 5: Create the HTML Client

Create a file called `index.html` in `src/main/resources/static/`:

```html
<!DOCTYPE html>
<html>
<head>
    <title>STOMP Ping-Pong Client</title>
    <script src="https://cdn.jsdelivr.net/npm/sockjs-client@1/dist/sockjs.min.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/@stomp/stompjs@7/bundles/stomp.umd.min.js"></script>
    <style>
        body {
            font-family: Arial, sans-serif;
            max-width: 800px;
            margin: 50px auto;
            padding: 20px;
        }
        .container {
            border: 1px solid #ccc;
            padding: 20px;
            border-radius: 5px;
        }
        button {
            background-color: #4CAF50;
            color: white;
            padding: 10px 20px;
            border: none;
            border-radius: 4px;
            cursor: pointer;
            margin: 5px;
        }
        button:hover {
            background-color: #45a049;
        }
        button:disabled {
            background-color: #cccccc;
            cursor: not-allowed;
        }
        #messages {
            height: 300px;
            overflow-y: auto;
            border: 1px solid #ddd;
            padding: 10px;
            margin-top: 20px;
            background-color: #f9f9f9;
        }
        .message {
            margin: 5px 0;
            padding: 5px;
        }
        .sent {
            color: #0066cc;
        }
        .received {
            color: #009900;
        }
        input {
            padding: 8px;
            margin: 5px;
            width: 300px;
        }
    </style>
</head>
<body>
    <div class="container">
        <h1>STOMP Ping-Pong Demo</h1>
        <div>
            <button id="connect" onclick="connect()">Connect</button>
            <button id="disconnect" onclick="disconnect()" disabled>Disconnect</button>
        </div>
        <div style="margin-top: 20px;">
            <input type="text" id="messageInput" placeholder="Enter message to ping..." />
            <button id="sendBtn" onclick="sendPing()" disabled>Send Ping</button>
        </div>
        <div id="status">Not connected</div>
        <div id="messages"></div>
    </div>

    <script>
        let stompClient = null;
        let connected = false;

        function setConnected(value) {
            connected = value;
            document.getElementById('connect').disabled = value;
            document.getElementById('disconnect').disabled = !value;
            document.getElementById('sendBtn').disabled = !value;
            document.getElementById('status').textContent = value ? 'Connected' : 'Disconnected';
        }

        function connect() {
            // Create a new SockJS instance pointing to our STOMP endpoint
            const socket = new SockJS('/gs-guide-websocket');
            
            // Create a STOMP client over the SockJS connection
            stompClient = StompJs.Stomp.over(socket);
            
            // Disable debug logging (set to true to see STOMP frames)
            stompClient.debug = function(str) {
                console.log(str);
            };

            // Connect to the server
            stompClient.connect({}, function(frame) {
                setConnected(true);
                addMessage('Connected: ' + frame, 'received');
                
                // Subscribe to the /topic/pong destination
                stompClient.subscribe('/topic/pong', function(message) {
                    addMessage('Received: ' + message.body, 'received');
                });
            });
        }

        function disconnect() {
            if (stompClient !== null) {
                stompClient.disconnect();
            }
            setConnected(false);
            addMessage('Disconnected', 'received');
        }

        function sendPing() {
            const messageInput = document.getElementById('messageInput');
            const message = messageInput.value || 'Ping!';
            
            if (stompClient !== null && connected) {
                // Send message to /app/ping
                stompClient.send('/app/ping', {}, message);
                addMessage('Sent: ' + message, 'sent');
                messageInput.value = '';
            }
        }

        function addMessage(message, type) {
            const messagesDiv = document.getElementById('messages');
            const messageElement = document.createElement('div');
            messageElement.className = 'message ' + type;
            messageElement.textContent = new Date().toLocaleTimeString() + ' - ' + message;
            messagesDiv.appendChild(messageElement);
            messagesDiv.scrollTop = messagesDiv.scrollHeight;
        }

        // Allow Enter key to send message
        document.getElementById('messageInput').addEventListener('keypress', function(e) {
            if (e.key === 'Enter' && connected) {
                sendPing();
            }
        });
    </script>
</body>
</html>
```

## Understanding the Client Code

Let's break down the key parts of the client:

**SockJS and STOMP Libraries**: We're using CDN-hosted libraries:
- SockJS: Provides WebSocket-like functionality with fallback transports
- STOMP.js: A JavaScript STOMP client library

**Connection Flow**:
1. Create a SockJS connection to `/gs-guide-websocket`
2. Wrap it with a STOMP client using `StompJs.Stomp.over(socket)`
3. Call `connect()` to establish the connection
4. In the connection callback, subscribe to `/topic/pong`

**Sending Messages**: `stompClient.send('/app/ping', {}, message)` sends a message to the server:
- First parameter: The destination (`/app/ping`)
- Second parameter: Headers (empty object in this case)
- Third parameter: The message body (a string)

**Receiving Messages**: The subscription callback handles incoming messages from `/topic/pong`.

## Running the Application

### Step 6: Configure Static Resource Serving

Since we're using plain HTML, Spring Boot will automatically serve files from `src/main/resources/static/`. Make sure your `application.properties` or `application.yml` doesn't override this behavior.

Create or update `src/main/resources/application.properties`:

```properties
spring.application.name=stomp-toy-project
server.port=8080
```

### Step 7: Run and Test

1. Start your Spring Boot application (using your IDE's run configuration or `mvn spring-boot:run`).
2. Open your browser and navigate to `http://localhost:8080/index.html`.
3. Click "Connect" to establish the WebSocket connection.
4. Enter a message in the input field and click "Send Ping" (or press Enter).
5. You should see:
   - Your sent message in blue
   - The server's response ("Pong: [your message]") in green

## Troubleshooting Common Issues

**Connection Refused**: Ensure your Spring Boot application is running on port 8080.

**404 on HTML Page**: Make sure `index.html` is in `src/main/resources/static/`.

**STOMP Frame Errors**: Check the browser console for detailed error messages. Ensure you're using compatible versions of SockJS and STOMP.js.

**CORS Issues**: In development, `setAllowedOriginPatterns("*")` handles this. In production, specify actual origins.

## What We've Accomplished

In this article, we've:
1. Set up a Spring Boot project with WebSocket and STOMP dependencies
2. Configured the WebSocket message broker with application and topic prefixes
3. Created a simple message controller that handles client messages
4. Built a functional HTML client that connects and exchanges messages
5. Demonstrated the basic ping-pong message flow

## Next Steps

You now have a working foundation for STOMP over WebSocket! The ping-pong demonstration validates that:
- The WebSocket connection is established correctly
- Messages flow from client to server
- Messages are broadcast from server to subscribed clients

In the next article, we'll explore server-initiated messaging using scheduled tasks to push real-time updates to clients without requiring client requests. This will demonstrate the power of the publish-subscribe model and how Spring makes it easy to broadcast server-generated data to all connected clients.

## Complete Test Client

For convenience, here's a standalone HTML file you can use to test the service. Save it as `test-client.html` and open it directly in your browser (pointing to your running Spring Boot app):

```html
<!DOCTYPE html>
<html>
<head>
    <title>STOMP Test Client</title>
    <script src="https://cdn.jsdelivr.net/npm/sockjs-client@1/dist/sockjs.min.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/@stomp/stompjs@7/bundles/stomp.umd.min.js"></script>
</head>
<body>
    <h1>STOMP Ping-Pong Test</h1>
    <button onclick="connect()">Connect</button>
    <button onclick="disconnect()">Disconnect</button>
    <button onclick="sendPing()">Send Ping</button>
    <div id="output"></div>

    <script>
        let client = null;
        
        function connect() {
            const socket = new SockJS('http://localhost:8080/gs-guide-websocket');
            client = StompJs.Stomp.over(socket);
            client.connect({}, () => {
                log('Connected!');
                client.subscribe('/topic/pong', (msg) => {
                    log('Received: ' + msg.body);
                });
            });
        }
        
        function disconnect() {
            if (client) client.disconnect();
            log('Disconnected');
        }
        
        function sendPing() {
            if (client) {
                client.send('/app/ping', {}, 'Hello from client!');
                log('Sent ping');
            }
        }
        
        function log(msg) {
            document.getElementById('output').innerHTML += '<p>' + msg + '</p>';
        }
    </script>
</body>
</html>
```

Congratulations! You've successfully implemented your first STOMP endpoint and verified two-way communication between Spring Boot and a browser client. This foundation will support all the advanced features we'll explore in the upcoming articles.
