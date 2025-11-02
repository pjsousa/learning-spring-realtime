# Securing Real-Time: Associating User Identity with STOMP Sessions

## Introduction

Up to this point, our applications have been open to anyone who can connect. While this is fine for simple demonstrations and public chat rooms, real-world applications need to know who their users are. User identity enables features like private messaging, personalized notifications, access control, and audit trails.

Spring Security integrates seamlessly with WebSocket and STOMP, allowing you to authenticate users during the initial HTTP handshake and then associate that authentication with the WebSocket session. Once authenticated, the user's identity (Principal) is available in every message handler, enabling you to implement user-specific features and security controls.

In this article, we'll explore WebSocket authentication, integrate Spring Security with our STOMP setup, and learn how to access user identity in message handlers. We'll build a foundation that enables user-specific messaging (which we'll implement in Article 7) and access control for different destinations.

## Understanding WebSocket Authentication

WebSocket authentication works differently from traditional HTTP authentication:

**HTTP Handshake**: WebSocket connections start with an HTTP upgrade request. This is where authentication happens—you can use session-based authentication, JWT tokens, or HTTP Basic/Digest authentication at this stage.

**Session Association**: Once authenticated, Spring automatically associates the authenticated user (Principal) with the WebSocket session. This Principal is then available in all subsequent STOMP message handlers.

**No Re-authentication**: After the initial handshake, WebSocket connections don't re-authenticate. The session maintains the user's identity throughout its lifetime.

**Security Considerations**: 
- WebSocket connections are long-lived, so proper session management is crucial
- Consider implementing token refresh mechanisms for long-running connections
- Validate user permissions on each message, not just at connection time

## Project Setup

We'll add Spring Security to our project and configure it to work with WebSocket.

### Step 1: Add Spring Security Dependency

Update your `pom.xml`:

```xml
<dependencies>
    <!-- Existing dependencies -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-websocket</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    
    <!-- Add Spring Security -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-security</artifactId>
    </dependency>
</dependencies>
```

### Step 2: Create Security Configuration

Create `SecurityConfig.java`:

```java
package com.example.stomptoyproject;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.annotation.web.configurers.AbstractHttpConfigurer;
import org.springframework.security.core.userdetails.User;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.security.provisioning.InMemoryUserDetailsManager;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .csrf(AbstractHttpConfigurer::disable) // Disable CSRF for WebSocket (or configure properly)
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/gs-guide-websocket/**", "/index.html", "/chat.html", 
                               "/stock-ticker.html", "/css/**", "/js/**").permitAll()
                .anyRequest().authenticated()
            )
            .httpBasic(); // Use HTTP Basic authentication
        
        return http.build();
    }

    @Bean
    public UserDetailsService userDetailsService() {
        UserDetails user1 = User.builder()
            .username("alice")
            .password(passwordEncoder().encode("password"))
            .roles("USER")
            .build();
        
        UserDetails user2 = User.builder()
            .username("bob")
            .password(passwordEncoder().encode("password"))
            .roles("USER")
            .build();
        
        UserDetails admin = User.builder()
            .username("admin")
            .password(passwordEncoder().encode("admin"))
            .roles("USER", "ADMIN")
            .build();
        
        return new InMemoryUserDetailsManager(user1, user2, admin);
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

This configuration:
- Creates in-memory users for testing (alice, bob, admin)
- Uses HTTP Basic authentication
- Allows access to static resources and WebSocket endpoints
- Disables CSRF (for simplicity; in production, configure it properly for WebSocket)

**Important**: In production, you'd use a proper authentication mechanism (JWT, OAuth2, database-backed users, etc.). This in-memory setup is for demonstration.

## Configuring WebSocket Security

Now we need to configure WebSocket to accept authenticated connections and pass the Principal to message handlers.

### Step 3: Update WebSocket Configuration

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
    }

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/gs-guide-websocket")
                .setAllowedOriginPatterns("*")
                .withSockJS();
        
        // No explicit security configuration needed here
        // Spring Security automatically secures WebSocket endpoints
        // The Principal from the HTTP handshake is automatically associated
    }
}
```

Spring Security automatically:
- Secures the WebSocket endpoint
- Extracts the Principal from the HTTP handshake
- Associates it with the WebSocket session
- Makes it available in message handlers

## Accessing User Identity in Controllers

Now let's update our chat controller to access and use the authenticated user's identity.

### Step 4: Enhanced Chat Controller with Authentication

Update `ChatController.java`:

```java
package com.example.stomptoyproject;

import org.springframework.messaging.handler.annotation.MessageMapping;
import org.springframework.messaging.handler.annotation.Payload;
import org.springframework.messaging.handler.annotation.SendTo;
import org.springframework.messaging.simp.SimpMessageHeaderAccessor;
import org.springframework.security.core.Authentication;
import org.springframework.stereotype.Controller;
import org.springframework.util.StringUtils;

import java.security.Principal;

@Controller
public class ChatController {

    @MessageMapping("/chat.sendMessage")
    @SendTo("/topic/public")
    public ChatMessage sendMessage(@Payload ChatMessage chatMessage,
                                   Principal principal,
                                   SimpMessageHeaderAccessor headerAccessor) {
        
        // Get authenticated username
        String username = principal != null ? principal.getName() : "Anonymous";
        
        // Override any client-provided username with authenticated username
        chatMessage.setSender(username);
        
        if (!StringUtils.hasText(chatMessage.getContent())) {
            return null;
        }
        
        chatMessage.setTimestamp(java.time.LocalDateTime.now());
        chatMessage.setType(ChatMessage.MessageType.CHAT);
        
        System.out.println("[" + chatMessage.getTimestamp() + "] " + 
                          username + ": " + chatMessage.getContent());
        
        return chatMessage;
    }

    @MessageMapping("/chat.addUser")
    @SendTo("/topic/public")
    public ChatMessage addUser(@Payload ChatMessage chatMessage,
                               Principal principal,
                               SimpMessageHeaderAccessor headerAccessor) {
        
        // Use authenticated username
        String username = principal != null ? principal.getName() : "Anonymous";
        
        // Store in session for later use
        headerAccessor.getSessionAttributes().put("username", username);
        headerAccessor.getSessionAttributes().put("principal", principal);
        
        chatMessage.setSender(username);
        chatMessage.setType(ChatMessage.MessageType.JOIN);
        chatMessage.setContent(username + " joined!");
        chatMessage.setTimestamp(java.time.LocalDateTime.now());
        
        System.out.println("User " + username + " joined the chat");
        return chatMessage;
    }

    @MessageMapping("/chat.getUserInfo")
    @SendTo("/user/queue/info")
    public UserInfo getUserInfo(Principal principal) {
        UserInfo info = new UserInfo();
        info.setUsername(principal != null ? principal.getName() : "Anonymous");
        info.setAuthenticated(principal != null);
        
        if (principal instanceof Authentication) {
            Authentication auth = (Authentication) principal;
            info.setRoles(auth.getAuthorities().stream()
                .map(a -> a.getAuthority())
                .toList());
        }
        
        return info;
    }
}
```

Key points:

**Principal Parameter**: Spring automatically injects the `Principal` (authenticated user) into message handler methods. You can add it as a parameter to any `@MessageMapping` method.

**Security Enforcement**: By using `principal.getName()`, we ensure messages are always attributed to the authenticated user, not any client-provided username.

**UserInfo Model**: Create this simple model:

```java
package com.example.stomptoyproject;

import java.util.List;

public class UserInfo {
    private String username;
    private boolean authenticated;
    private List<String> roles;

    // Getters and setters
    public String getUsername() {
        return username;
    }

    public void setUsername(String username) {
        this.username = username;
    }

    public boolean isAuthenticated() {
        return authenticated;
    }

    public void setAuthenticated(boolean authenticated) {
        this.authenticated = authenticated;
    }

    public List<String> roles() {
        return roles;
    }

    public void setRoles(List<String> roles) {
        this.roles = roles;
    }
}
```

## Implementing Access Control

Let's create a method that demonstrates access control based on user roles:

### Step 5: Create Admin-Only Controller

Create `AdminController.java`:

```java
package com.example.stomptoyproject;

import org.springframework.messaging.handler.annotation.MessageMapping;
import org.springframework.messaging.handler.annotation.SendTo;
import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.security.core.Authentication;
import org.springframework.security.core.GrantedAuthority;
import org.springframework.stereotype.Controller;

import java.security.Principal;
import java.util.stream.Collectors;

@Controller
public class AdminController {

    @MessageMapping("/admin.broadcast")
    @SendTo("/topic/admin-announcements")
    @PreAuthorize("hasRole('ADMIN')")
    public AdminMessage broadcastMessage(AdminMessage message, Principal principal) {
        message.setSender(principal.getName());
        message.setTimestamp(java.time.LocalDateTime.now());
        System.out.println("Admin " + principal.getName() + " sent: " + message.getContent());
        return message;
    }

    @MessageMapping("/admin.getStats")
    @PreAuthorize("hasRole('ADMIN')")
    public AdminStats getStats(Principal principal) {
        AdminStats stats = new AdminStats();
        stats.setMessage("Admin-only statistics");
        stats.setRequestedBy(principal.getName());
        
        // Add your admin-only statistics here
        return stats;
    }
}
```

Create the models:

```java
package com.example.stomptoyproject;

import java.time.LocalDateTime;

public class AdminMessage {
    private String sender;
    private String content;
    private LocalDateTime timestamp;

    // Getters and setters...
}

public class AdminStats {
    private String message;
    private String requestedBy;

    // Getters and setters...
}
```

**@PreAuthorize**: This annotation checks user permissions before allowing the method to execute. If the user doesn't have the required role, Spring throws an exception.

## Enabling Method Security

To use `@PreAuthorize`, enable method security:

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity  // Add this
public class SecurityConfig {
    // ... existing code ...
}
```

## Creating an Authenticated Chat Client

Now let's create a client that handles HTTP Basic authentication.

### Step 6: Create Authenticated Client

Create `authenticated-chat.html` in `src/main/resources/static/`:

```html
<!DOCTYPE html>
<html>
<head>
    <title>Authenticated Chat</title>
    <script src="https://cdn.jsdelivr.net/npm/sockjs-client@1/dist/sockjs.min.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/@stomp/stompjs@7/bundles/stomp.umd.min.js"></script>
    <style>
        /* Similar styles to previous chat examples */
        body {
            font-family: Arial, sans-serif;
            max-width: 800px;
            margin: 50px auto;
            padding: 20px;
        }
        .login-section {
            border: 1px solid #ddd;
            padding: 30px;
            border-radius: 5px;
            text-align: center;
        }
        .login-section input {
            padding: 10px;
            margin: 5px;
            width: 200px;
        }
        .user-info {
            background: #e7f3ff;
            padding: 10px;
            border-radius: 5px;
            margin: 10px 0;
        }
    </style>
</head>
<body>
    <div id="loginSection" class="login-section">
        <h2>Login</h2>
        <p>Try: alice/password, bob/password, or admin/admin</p>
        <input type="text" id="username" placeholder="Username" />
        <input type="password" id="password" placeholder="Password" />
        <br>
        <button onclick="login()">Login</button>
    </div>

    <div id="chatSection" style="display:none;">
        <div class="user-info">
            Logged in as: <strong id="displayUsername"></strong>
        </div>
        <div id="messages"></div>
        <input type="text" id="messageInput" placeholder="Type message..." />
        <button onclick="sendMessage()">Send</button>
        <button onclick="getUserInfo()">Get My Info</button>
    </div>

    <script>
        let stompClient = null;
        let authenticatedUsername = null;
        let credentials = null;

        function login() {
            const username = document.getElementById('username').value;
            const password = document.getElementById('password').value;

            if (!username || !password) {
                alert('Please enter username and password');
                return;
            }

            // Store credentials for WebSocket connection
            credentials = btoa(username + ':' + password);

            // Test HTTP authentication first
            fetch('/gs-guide-websocket/info', {
                headers: {
                    'Authorization': 'Basic ' + credentials
                }
            }).then(response => {
                if (response.ok) {
                    authenticatedUsername = username;
                    connectWebSocket();
                } else {
                    alert('Authentication failed');
                }
            }).catch(error => {
                alert('Error: ' + error);
            });
        }

        function connectWebSocket() {
            const socket = new SockJS('/gs-guide-websocket');
            
            // Configure headers for authentication
            socket.onopen = function() {
                // Connection opened, but authentication happens at HTTP level
            };

            stompClient = StompJs.Stomp.over(socket);
            
            // Configure connection headers with credentials
            const headers = {};
            if (credentials) {
                headers['Authorization'] = 'Basic ' + credentials;
            }

            stompClient.connect(headers, function(frame) {
                document.getElementById('loginSection').style.display = 'none';
                document.getElementById('chatSection').style.display = 'block';
                document.getElementById('displayUsername').textContent = authenticatedUsername;

                // Subscribe to public messages
                stompClient.subscribe('/topic/public', function(message) {
                    const chatMessage = JSON.parse(message.body);
                    displayMessage(chatMessage);
                });

                // Subscribe to user-specific queue
                stompClient.subscribe('/user/queue/info', function(message) {
                    const userInfo = JSON.parse(message.body);
                    alert('Your info: ' + JSON.stringify(userInfo, null, 2));
                });

                // Notify server of join
                stompClient.send('/app/chat.addUser', {}, JSON.stringify({
                    type: 'JOIN',
                    sender: authenticatedUsername,
                    content: ''
                }));
            }, function(error) {
                alert('WebSocket connection failed: ' + error);
            });
        }

        function sendMessage() {
            const messageInput = document.getElementById('messageInput');
            const content = messageInput.value.trim();

            if (!content || !stompClient) {
                return;
            }

            stompClient.send('/app/chat.sendMessage', {}, JSON.stringify({
                type: 'CHAT',
                sender: authenticatedUsername,
                content: content
            }));

            messageInput.value = '';
        }

        function getUserInfo() {
            if (stompClient) {
                stompClient.send('/app/chat.getUserInfo', {}, '');
            }
        }

        function displayMessage(chatMessage) {
            const messagesDiv = document.getElementById('messages');
            const messageDiv = document.createElement('div');
            messageDiv.innerHTML = `<strong>${chatMessage.sender}:</strong> ${chatMessage.content}`;
            messagesDiv.appendChild(messageDiv);
        }
    </script>
</body>
</html>
```

**Important Note**: STOMP.js doesn't directly support passing credentials in connect headers the way shown above. For HTTP Basic authentication, you typically need to:

1. Authenticate via a regular HTTP endpoint first
2. Use session-based authentication (cookies)
3. Or implement custom authentication headers (requires server-side configuration)

For simplicity in this example, we'll use a session-based approach.

## Using Session-Based Authentication

A better approach for WebSocket is to authenticate via a regular HTTP endpoint first, then use the session:

### Step 7: Create Login Endpoint

Create `AuthController.java`:

```java
package com.example.stomptoyproject;

import jakarta.servlet.http.HttpSession;
import org.springframework.http.ResponseEntity;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.Authentication;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.security.web.context.HttpSessionSecurityContextRepository;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RestController;

import java.util.HashMap;
import java.util.Map;

@RestController
public class AuthController {

    @PostMapping("/api/login")
    public ResponseEntity<Map<String, String>> login(@RequestBody LoginRequest request,
                                                      HttpSession session) {
        // Spring Security handles authentication automatically
        // If we reach here, authentication was successful
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        
        Map<String, String> response = new HashMap<>();
        response.put("status", "success");
        response.put("username", auth.getName());
        
        return ResponseEntity.ok(response);
    }

    @PostMapping("/api/logout")
    public ResponseEntity<Map<String, String>> logout(HttpSession session) {
        session.invalidate();
        Map<String, String> response = new HashMap<>();
        response.put("status", "logged out");
        return ResponseEntity.ok(response);
    }
}

class LoginRequest {
    private String username;
    private String password;

    // Getters and setters...
}
```

## JWT Token Authentication (Alternative)

For production applications, JWT tokens are often preferred. Here's a simplified example:

```java
@MessageMapping("/chat.sendMessage")
@SendTo("/topic/public")
public ChatMessage sendMessage(@Payload ChatMessage chatMessage,
                               @Header("Authorization") String token) {
    // Validate JWT token and extract user info
    String username = jwtService.extractUsername(token);
    // ... rest of logic
}
```

This requires custom interceptor configuration, which is beyond the scope of this article but is a common production pattern.

## Testing Authentication

1. Start your Spring Boot application
2. Open `http://localhost:8080/authenticated-chat.html`
3. Try logging in with:
   - Username: `alice`, Password: `password`
   - Username: `bob`, Password: `password`
   - Username: `admin`, Password: `admin`
4. Verify that messages show the authenticated username
5. Try accessing admin endpoints (if you implemented them)

## Complete Test Client

Here's a minimal authenticated test client:

```html
<!DOCTYPE html>
<html>
<head>
    <title>Auth Test</title>
    <script src="https://cdn.jsdelivr.net/npm/sockjs-client@1/dist/sockjs.min.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/@stomp/stompjs@7/bundles/stomp.umd.min.js"></script>
</head>
<body>
    <input type="text" id="user" placeholder="Username" />
    <input type="password" id="pass" placeholder="Password" />
    <button onclick="connect()">Connect</button>
    <div id="output"></div>

    <script>
        function connect() {
            const user = document.getElementById('user').value;
            const pass = document.getElementById('pass').value;
            
            // Create authenticated fetch request first
            fetch('/gs-guide-websocket/info', {
                headers: {
                    'Authorization': 'Basic ' + btoa(user + ':' + pass)
                }
            }).then(() => {
                const socket = new SockJS('/gs-guide-websocket');
                const client = StompJs.Stomp.over(socket);
                
                client.connect({
                    // For session-based auth, credentials are in cookies
                }, () => {
                    log('Connected as: ' + user);
                    client.subscribe('/topic/public', (msg) => {
                        const chat = JSON.parse(msg.body);
                        log(chat.sender + ': ' + chat.content);
                    });
                });
            }).catch(err => log('Auth failed: ' + err));
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
1. Integrated Spring Security with WebSocket
2. Learned how WebSocket authentication works via HTTP handshake
3. Accessed user identity (Principal) in message handlers
4. Implemented role-based access control
5. Created authenticated clients

## Next Steps

You now have a secure foundation where users are authenticated and their identity is available throughout the WebSocket session. This enables user-specific features.

In our next article, we'll leverage this authentication to implement user destinations (`/user/queue/*`), allowing you to send private messages to specific users. This is where the authentication work pays off—we can target messages to specific authenticated users without managing session IDs manually.
