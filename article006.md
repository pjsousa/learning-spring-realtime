# Securing Real-Time: Associating User Identity with STOMP Sessions

Up to this point, our STOMP toy projects have allowed any browser to connect and interact freely. For prototypes that may be acceptable, but most real-world applications must authenticate users, enforce authorization, and trace actions back to specific identities. WebSocket connections share the underlying HTTP infrastructure, which means we can leverage Spring Security to protect our STOMP endpoints just like REST APIs.

In this sixth article we introduce authentication and identity to our chat application. We will configure Spring Security for form-based login, ensure the authenticated `Principal` travels with every STOMP message, and teach the client how to log in before establishing a WebSocket session. Along the way we will demystify how the HTTP handshake and WebSocket session tie together and explore advanced topics like channel interceptors for access control.

By the end of this article you will:

- Protect STOMP endpoints using Spring Security’s HTTP configuration.
- Understand how the HTTP handshake principal becomes the STOMP session principal.
- Enforce destination-level authorization with `SimpMessageType` matchers.
- Access `Principal` information inside `@MessageMapping` methods to stamp user identity on chat messages.
- Update the HTML client with a simple login form and handle authenticated sessions via `fetch` and cookies.

Let’s secure our toy chat project.

## Adding Spring Security Dependencies

Open your build file and include the Spring Security starter. For Maven:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```

If you use Gradle:

```groovy
implementation 'org.springframework.boot:spring-boot-starter-security'
```

Spring Boot automatically configures a basic security setup when this dependency is present. Restarting the application will now prompt for a username and password on every HTTP request. We will craft a custom security configuration tailored to our chat experience.

## Creating a Basic User Store

For simplicity we will define an in-memory user store with two sample users: `alice` and `bob`. Create a configuration class under `com.example.stompchat.security`:

```java
package com.example.stompchat.security;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.Customizer;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.core.userdetails.User;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.security.provisioning.InMemoryUserDetailsManager;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public UserDetailsService users(PasswordEncoder passwordEncoder) {
        return new InMemoryUserDetailsManager(
                User.withUsername("alice").password(passwordEncoder.encode("password")).roles("USER").build(),
                User.withUsername("bob").password(passwordEncoder.encode("password")).roles("USER").build()
        );
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
                .authorizeHttpRequests(authorize -> authorize
                        .requestMatchers("/css/**", "/js/**", "/login", "/sockjs/**", "/ws/**").permitAll()
                        .anyRequest().authenticated()
                )
                .formLogin(Customizer.withDefaults())
                .logout(logout -> logout.logoutSuccessUrl("/login?logout"))
                .csrf(csrf -> csrf.ignoringRequestMatchers("/ws/**", "/sockjs/**"));

        return http.build();
    }
}
```

Highlights:

- We register two in-memory users with BCrypt-hashed passwords.
- Static resources (`/css`, `/js`) and WebSocket endpoints remain publicly accessible so the client can download assets and begin the handshake. You may choose to require authentication even for the handshake; we will discuss that shortly.
- Form login is enabled with default settings. Spring Security generates a login page at `/login` and handles session cookies automatically.
- CSRF protection is disabled for `/ws/**` and `/sockjs/**`. WebSocket handshakes are not compatible with CSRF tokens, so we need to opt out for these paths. REST endpoints remain protected by CSRF tokens.

## Understanding the Handshake Principal

When a client connects to `/ws` or `/sockjs`, Spring performs the handshake via HTTP. If the client has an authenticated session (for example, after logging in), Spring Security attaches the `Principal` to the handshake and stores it in the WebSocket session. Every subsequent STOMP frame processed through `@MessageMapping` includes that principal.

To verify this, modify the chat controller to log the principal name:

```java
@MessageMapping("/chat")
@SendTo("/topic/chat")
public ChatMessage handleChat(@Payload @Valid TypedRequest request, Principal principal) {
    String user = principal != null ? principal.getName() : request.sender();
    ChatMessage message = ChatMessage.userMessage(user, sanitize(request.content()));
    history.add(message);
    log.info("{} posted a message", user);
    return message;
}
```

Even if the client attempts to spoof the sender field, the server trusts the authenticated principal. This is essential for preventing impersonation. The UI can still set a display nickname, but the server records the actual identity.

## Restricting STOMP Destinations

Spring Security’s messaging integration allows fine-grained access control on STOMP destinations. Add a `SecurityConfig` bean for the message broker by extending `AbstractSecurityWebSocketMessageBrokerConfigurer`:

```java
package com.example.stompchat.security;

import org.springframework.context.annotation.Configuration;
import org.springframework.messaging.simp.config.ChannelRegistration;
import org.springframework.security.config.annotation.web.messaging.MessageSecurityMetadataSourceRegistry;
import org.springframework.security.config.annotation.web.socket.AbstractSecurityWebSocketMessageBrokerConfigurer;

@Configuration
public class WebSocketSecurityConfig extends AbstractSecurityWebSocketMessageBrokerConfigurer {

    @Override
    protected void configureInbound(MessageSecurityMetadataSourceRegistry messages) {
        messages
                .simpTypeMatchers(SimpMessageType.CONNECT, SimpMessageType.DISCONNECT).permitAll()
                .simpDestMatchers("/app/chat", "/app/typing").hasRole("USER")
                .simpDestMatchers("/topic/**", "/queue/**").permitAll()
                .anyMessage().denyAll();
    }

    @Override
    public void configureClientInboundChannel(ChannelRegistration registration) {
        registration.interceptors(new PrincipalPropagationInterceptor());
    }

    @Override
    protected boolean sameOriginDisabled() {
        return true; // allow cross-origin SockJS for local demos
    }
}
```

Here we permit anyone to connect or disconnect but require role `USER` for sending messages to `/app/chat` and `/app/typing`. Broadcast destinations remain open so the broker can deliver messages. We also register a custom channel interceptor (`PrincipalPropagationInterceptor`) which we will define to log user actions or enforce additional rules.

### Principal Propagation Interceptor

The interceptor inspects incoming messages and attaches metadata. Create the class:

```java
package com.example.stompchat.security;

import java.security.Principal;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.messaging.Message;
import org.springframework.messaging.MessageChannel;
import org.springframework.messaging.simp.stomp.StompHeaderAccessor;
import org.springframework.messaging.support.ChannelInterceptor;

public class PrincipalPropagationInterceptor implements ChannelInterceptor {

    private static final Logger log = LoggerFactory.getLogger(PrincipalPropagationInterceptor.class);

    @Override
    public Message<?> preSend(Message<?> message, MessageChannel channel) {
        StompHeaderAccessor accessor = StompHeaderAccessor.wrap(message);
        Principal principal = accessor.getUser();
        if (principal == null) {
            log.warn("Anonymous message attempted: {}", accessor.getCommand());
        } else {
            log.debug("User {} sent {} to {}", principal.getName(), accessor.getCommand(), accessor.getDestination());
        }
        return message;
    }
}
```

This interceptor is optional but demonstrates how you can observe and veto messages before they reach controllers. You could check custom headers, rate-limit users, or reject messages based on business rules.

## Updating the Browser Client with Authentication

The HTML client must authenticate via HTTP before establishing the WebSocket connection. Because we enabled form login, the simplest approach is to embed a login form on the chat page and submit it using `fetch` to `/login`. Once the server sets the session cookie, the STOMP handshake automatically uses it.

Update `chat.html` to include a login panel:

```html
<section id="auth" class="card">
    <h2>Sign In</h2>
    <form id="login-form">
        <label>Username <input name="username" value="alice" required /></label>
        <label>Password <input type="password" name="password" value="password" required /></label>
        <button type="submit">Login</button>
    </form>
    <div id="login-error"></div>
</section>

<section id="chat" hidden>
    <!-- existing chat UI -->
</section>
```

Add the script logic:

```javascript
const loginForm = document.getElementById('login-form');
const loginError = document.getElementById('login-error');
const authSection = document.getElementById('auth');
const chatSection = document.getElementById('chat');

loginForm.addEventListener('submit', async event => {
    event.preventDefault();
    const formData = new FormData(loginForm);
    const response = await fetch('/login', {
        method: 'POST',
        body: formData,
        headers: { 'X-Requested-With': 'XMLHttpRequest' }
    });

    if (response.ok) {
        authSection.hidden = true;
        chatSection.hidden = false;
        renderStatus('Authenticated. Click Connect to join.');
    } else {
        loginError.textContent = 'Login failed. Check credentials';
    }
});
```

The fetch request posts credentials to `/login`. Spring Security handles authentication and establishes a session cookie. For cross-site requests consider `SameSite` cookie policies; locally, the default is fine.

### Handling Logout

Add a logout button that calls `/logout` via POST and resets the UI:

```html
<button id="logout" hidden>Logout</button>
```

```javascript
const logoutBtn = document.getElementById('logout');

logoutBtn.addEventListener('click', async () => {
    await fetch('/logout', { method: 'POST' });
    authSection.hidden = false;
    chatSection.hidden = true;
    logoutBtn.hidden = true;
    renderStatus('Logged out');
    if (stompClient) {
        stompClient.deactivate();
    }
});

loginForm.addEventListener('submit', async event => {
    // ...existing logic...
    if (response.ok) {
        logoutBtn.hidden = false;
    }
});
```

## Verifying Secure Messaging

Restart the application, open `http://localhost:8080/chat.html`, and attempt to connect without logging in. The server denies access, and you should see warnings `Anonymous message attempted`. After logging in as `alice`, click “Connect” and send a message; the server logs now include `alice posted a message`. Inspect the WebSocket frames to confirm that the STOMP `CONNECT` frame still contains the default headers, but the server associates the session with `alice` automatically.

### Testing Unauthorized Access

Try modifying the client code to publish to `/app/chat` before logging in. The server should return an ERROR frame and close the connection. You can also use curl to test the handshake:

```bash
curl -i http://localhost:8080/ws
```

Without authentication, the handshake returns `403 Forbidden`. After logging in via the browser and copying the session cookie, you can replay the handshake using curl with `-H 'Cookie: JSESSIONID=...'` to confirm access is granted.

## Channel-Level Authorization Strategies

The configuration we used is straightforward, but Spring Security supports more advanced scenarios:

- **Role-based authorization on subscriptions:** Require that only certain roles can subscribe to sensitive destinations using `.simpSubscribeDestMatchers("/topic/admin/**").hasRole("ADMIN")`.
- **Method-level security:** Add `@PreAuthorize` annotations directly on `@MessageMapping` methods to enforce business rules. For example, `@PreAuthorize("principal.name == #request.sender")` ensures users only post on their behalf.
- **Handshake interceptors:** Implement `HttpSessionHandshakeInterceptor` or custom interceptors to perform logic during the HTTP handshake, such as rejecting certain user agents or injecting additional attributes into the WebSocket session.

We encourage exploring these options as your toy project evolves.

## Testing Authentication with Integration Tests

Extend your integration tests to authenticate before connecting:

```java
@Test
void authenticatedUserCanSendMessages() throws Exception {
    WebClient webClient = WebClient.create("http://localhost:" + port);

    // login to capture session cookie
    ResponseEntity<String> login = webClient.post()
            .uri("/login")
            .contentType(MediaType.APPLICATION_FORM_URLENCODED)
            .body(BodyInserters.fromFormData("username", "alice").with("password", "password"))
            .retrieve()
            .toEntity(String.class)
            .block();

    String cookie = login.getHeaders().getFirst(HttpHeaders.SET_COOKIE);

    StompHeaders headers = new StompHeaders();
    headers.add("login-cookie", cookie);

    WebSocketHttpHeaders handshakeHeaders = new WebSocketHttpHeaders();
    handshakeHeaders.add(HttpHeaders.COOKIE, cookie);

    StompSession session = stompClient.connect(String.format("ws://localhost:%d/ws", port), handshakeHeaders, headers, new StompSessionHandlerAdapter() {}).get(1, TimeUnit.SECONDS);

    // proceed with assertions as before
}
```

This test demonstrates how to authenticate via HTTP and reuse the session cookie for the STOMP connection.

## Troubleshooting Authentication Issues

Common pitfalls include:

- **Handshake rejected with 403:** Ensure the `/ws` or `/sockjs` endpoints are permitted in the HTTP security configuration. If you require authentication for the handshake, the client must log in first and include the session cookie.
- **Principal is null inside controllers:** Verify that the WebSocket connection shares the same domain as the login request; otherwise, the cookie may not be sent. Also ensure `WebSocketSecurityConfig` extends `AbstractSecurityWebSocketMessageBrokerConfigurer` to integrate security.
- **CSRF errors on login:** When posting to `/login` via fetch, Spring Security expects a CSRF token. Include the token from the login page or disable CSRF for `/login` in development. The default login form includes the token automatically; our fetch request should include credentials from the generated login page or use `http.csrf().disable()` cautiously.
- **Session fixation on reconnect:** By default, Spring Security protects against session fixation. If your client reconnects frequently, ensure you handle the updated session cookie after login.

## Summary and Next Steps

We have fortified our STOMP toy project with authentication and user identity. Spring Security seamlessly integrates with WebSocket handshakes, allowing `Principal` information to flow into every `@MessageMapping` method. We can now trust that chat messages originate from authenticated users and enforce access control on specific destinations.

In the next article we build on this foundation to implement private messaging using user destinations and `convertAndSendToUser`. Before moving forward, confirm that you can:

- Log in through the chat UI and establish a STOMP connection as an authenticated user.
- Observe the `Principal` name inside controllers and interceptors.
- Prevent unauthenticated users from sending messages.

Once you are comfortable with these flows, you are ready to target specific users with personalized notifications in Article 7.

