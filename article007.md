# Differentiate Clients: Sending Private Messages using User Destinations (/user/)

After introducing authentication in Article 6, we can finally treat users as unique identities. The next logical step is to deliver personalized updates—private chat messages, targeted notifications, or user-specific task lists. Spring’s STOMP support provides a powerful abstraction for this: user destinations. When a client subscribes to `/user/queue/updates`, the framework automatically routes messages to the session associated with that user without requiring you to track session IDs manually.

In this seventh article we will extend the secured chat project to support private messaging. Two users will be able to send direct messages to each other, and the server will guarantee that only the intended recipient receives the content. Along the way we will examine how user destinations work under the hood, discuss scaling considerations, and update the HTML client with UI for targeting specific users.

By the end of this article you will:

- Configure a custom user destination prefix and understand the broker mappings.
- Use `SimpMessagingTemplate.convertAndSendToUser` to deliver private messages.
- Manage online user presence to drive UI elements like “send to” dropdowns.
- Update the client to subscribe to `/user/queue/messages` and `/user/queue/errors` for personalized streams.
- Incorporate server-side validation to prevent unauthorized private messaging.

Let’s personalize our toy project.

## Revisiting the Security Setup

Ensure you completed Article 6’s authentication steps. We rely on logged-in users (`alice`, `bob`, etc.) so the server can identify recipients by username. Our `SecurityConfig` already enables form login and restricts `/app/chat` to authenticated users. We will build on top of that configuration without altering it significantly.

## Understanding User Destinations

Spring’s messaging infrastructure has a user destination prefix (default `/user/`). When a client subscribes to `/user/queue/messages`, Spring translates this to an internal destination such as `/queue/messages-user123` where `user123` represents the session identifier. When the server invokes `convertAndSendToUser("alice", "/queue/messages", payload)`, Spring resolves all active sessions for user `alice` and forwards the message to those internal destinations.

This abstraction simplifies private messaging because you do not need to manage session IDs yourself. It also supports multiple concurrent sessions per user (browser tabs, mobile devices) automatically.

## Configuring the User Destination Prefix

Although the default prefix is `/user/`, we will explicitly configure it in `WebSocketConfig` to highlight the option and align with our HTML client. Ensure the configuration includes `registry.setUserDestinationPrefix("/user")` (already present if you followed Article 6). If not, update the `configureMessageBroker` method:

```java
@Override
public void configureMessageBroker(MessageBrokerRegistry registry) {
    registry.enableSimpleBroker("/topic", "/queue")
            .setHeartbeatValue(new long[]{10000, 10000});
    registry.setApplicationDestinationPrefixes("/app");
    registry.setUserDestinationPrefix("/user");
}
```

The broker now knows it should translate destinations starting with `/user/` into user-specific queues.

## Creating the Private Message DTO

Define `DirectMessage` and `DirectMessageRequest` records for server responses and client requests respectively:

```java
package com.example.stompchat.privatechat;

import java.time.Instant;

public record DirectMessage(String id,
                            String sender,
                            String recipient,
                            String content,
                            Instant timestamp) {}

public record DirectMessageRequest(String recipient, String content) {}
```

We rely on the authenticated principal for the sender identity. The client only needs to specify the recipient and message text.

## Building the Private Messaging Controller

Create `PrivateChatController` under `com.example.stompchat.privatechat`:

```java
package com.example.stompchat.privatechat;

import java.security.Principal;
import java.time.Instant;
import java.util.UUID;

import jakarta.validation.Valid;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.Size;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.messaging.handler.annotation.MessageExceptionHandler;
import org.springframework.messaging.handler.annotation.MessageMapping;
import org.springframework.messaging.simp.SimpMessagingTemplate;
import org.springframework.messaging.simp.annotation.SendToUser;
import org.springframework.stereotype.Controller;

@Controller
public class PrivateChatController {

    private static final Logger log = LoggerFactory.getLogger(PrivateChatController.class);

    private final SimpMessagingTemplate messagingTemplate;

    public record DirectMessagePayload(@NotBlank @Size(max = 40) String recipient,
                                       @NotBlank @Size(max = 500) String content) {}

    public PrivateChatController(SimpMessagingTemplate messagingTemplate) {
        this.messagingTemplate = messagingTemplate;
    }

    @MessageMapping("/direct")
    @SendToUser("/queue/direct")
    public DirectMessage sendDirectMessage(@Valid DirectMessagePayload request, Principal principal) {
        if (principal == null) {
            throw new IllegalStateException("User must be authenticated to send private messages");
        }

        if (principal.getName().equals(request.recipient())) {
            throw new IllegalArgumentException("Cannot send a direct message to yourself");
        }

        DirectMessage message = new DirectMessage(
                UUID.randomUUID().toString(),
                principal.getName(),
                request.recipient(),
                request.content().trim(),
                Instant.now()
        );

        messagingTemplate.convertAndSendToUser(request.recipient(), "/queue/direct", message);
        log.info("{} sent a direct message to {}", principal.getName(), request.recipient());
        return message;
    }

    @MessageExceptionHandler
    @SendToUser("/queue/errors")
    public String handleMessagingErrors(Exception ex) {
        return ex.getMessage();
    }
}
```

Breakdown:

- Clients send requests to `/app/direct` (thanks to `@MessageMapping("/direct")`).
- The controller validates recipient and content, prevents self-targeting, and constructs a `DirectMessage` with the authenticated principal as sender.
- `convertAndSendToUser` delivers the message to the recipient’s queue `/user/{username}/queue/direct`.
- The method also returns the message via `@SendToUser("/queue/direct")`, so the sender receives a confirmation copy (useful for rendering their own message bubble immediately).
- Errors are sent to `/user/queue/errors`, the same destination we already use for validation errors.

## Surfacing Online Users to the Client

The UI needs to present a list of potential recipients. We already built `ParticipantTracker` in Article 5 to snapshot active sessions. Let’s enhance it with a service that exposes the list of connected usernames. Add a controller under `com.example.stompchat.presence`:

```java
package com.example.stompchat.presence;

import java.util.List;

import org.springframework.messaging.handler.annotation.MessageMapping;
import org.springframework.messaging.simp.annotation.SendToUser;
import org.springframework.stereotype.Controller;

import com.example.stompchat.bootstrap.ParticipantTracker;

@Controller
public class PresenceController {

    private final ParticipantTracker participantTracker;

    public PresenceController(ParticipantTracker participantTracker) {
        this.participantTracker = participantTracker;
    }

    @MessageMapping("/presence")
    @SendToUser("/queue/presence")
    public List<String> listParticipants() {
        return participantTracker.activeParticipants().stream()
                .map(ParticipantTracker.ChatParticipant::username)
                .sorted()
                .toList();
    }
}
```

Clients can send an empty message to `/app/presence` to retrieve the current user list. Alternatively, we could reuse `@SubscribeMapping`, but this explicit command allows refreshing on demand.

## Updating the HTML Client for Private Messaging

Enhance the chat UI with a recipient selector and a private message panel. Inside `chat.html`, add the following markup within the chat section:

```html
<section id="private" class="card" hidden>
    <h2>Direct Message</h2>
    <label for="recipient">Send to</label>
    <select id="recipient"></select>
    <textarea id="direct-message" rows="3"></textarea>
    <button id="send-direct">Send Private Message</button>
    <div id="direct-log"></div>
</section>
```

Update the script:

```javascript
const recipientSelect = document.getElementById('recipient');
const directMessageInput = document.getElementById('direct-message');
const sendDirectButton = document.getElementById('send-direct');
const directLog = document.getElementById('direct-log');
const privateSection = document.getElementById('private');

function renderPresence(list) {
    const options = list.map(user => `<option value="${user}">${user}</option>`).join('');
    recipientSelect.innerHTML = options;
    privateSection.hidden = list.length === 0;
}

function appendDirectMessage(message, direction) {
    const entry = document.createElement('div');
    entry.className = `dm ${direction}`;
    entry.textContent = `${new Date(message.timestamp).toLocaleTimeString()} · ${message.sender}: ${message.content}`;
    directLog.prepend(entry);
}

function subscribeAll() {
    // ... existing subscriptions ...

    stompClient.subscribe('/user/queue/direct', frame => {
        const message = JSON.parse(frame.body);
        const direction = message.sender === nicknameInput.value ? 'outgoing' : 'incoming';
        appendDirectMessage(message, direction);
    });

    stompClient.subscribe('/user/queue/presence', frame => {
        const list = JSON.parse(frame.body);
        renderPresence(list.filter(name => name !== nicknameInput.value));
    });
}

sendDirectButton.addEventListener('click', () => {
    const recipient = recipientSelect.value;
    const content = directMessageInput.value.trim();
    if (!recipient || !content) {
        errorsDiv.textContent = 'Select a recipient and enter a message.';
        return;
    }

    stompClient.publish({
        destination: '/app/direct',
        body: JSON.stringify({ recipient, content })
    });
    directMessageInput.value = '';
});

function requestPresence() {
    stompClient.publish({ destination: '/app/presence', body: '{}' });
}

stompClient.onConnect = () => {
    // existing setup
    subscribeAll();
    requestPresence();
    setInterval(requestPresence, 15000);
};
```

Key additions:

- The client subscribes to `/user/queue/direct` to receive both incoming messages and confirmation of outgoing ones.
- We keep the recipient list up-to-date by hitting `/app/presence` initially and every 15 seconds. You can trigger the fetch on demand (for example, when opening the private message panel) to reduce traffic.
- When rendering the presence list we exclude the current user to avoid self-targeting (the server already enforces this rule).

### Styling and UX

Add CSS to differentiate incoming and outgoing direct messages:

```css
#private { margin-top: 1rem; }
#direct-log { margin-top: 1rem; max-height: 15rem; overflow-y: auto; background: #1f2933; color: #f5f7fa; padding: 1rem; border-radius: 4px; }
.dm.outgoing { text-align: right; color: #8ae2ff; }
.dm.incoming { text-align: left; color: #ffd166; }
```

Feel free to refine the styling to match your brand.

## Validating Private Messaging End-to-End

Run the application, log in as `alice` in one browser window and `bob` in another. Connect both clients. The recipient dropdown should list the other user. Compose a private message from `alice` to `bob`; both the sender and recipient see a log entry in their Direct Message panel. Verify that public chat messages continue to function as before.

### Testing Unauthorized Access

Attempt to send a direct message without logging in. The server responds with an error sent to `/user/queue/errors`. If you intercept the message on the client, you can display the error to the user. Also test sending to an invalid recipient; by default, `convertAndSendToUser` drops messages if the user has no active sessions. You can detect this by inspecting the returned count from `messagingTemplate.convertAndSendToUser`. If the message fails to deliver, consider storing it for later or informing the sender.

## How User Destinations Work Internally

Under the hood, Spring maintains a `UserDestinationMessageHandler` that listens for `/user/**` messages. When it receives a message targeting `/user/alice/queue/direct`, it consults the `SimpUserRegistry` to find all sessions for `alice` and rewrites the destination to the session-specific queue (for example, `/queue/direct-user0wj`). The broker then delivers the message as usual. This means user destinations are compatible with the simple broker, broker relays (Article 10), and even multicast setups.

One important implication is that user destinations rely on the user’s `Principal.getName()`. If you customize authentication (JWT tokens, OAuth2), ensure the principal name is consistent and unique per user. For anonymous access you can assign a temporary principal using a handshake interceptor.

## Troubleshooting User Destinations

Common challenges:

- **Messages not delivered:** Verify users are authenticated and that the recipient spelled the username exactly. `convertAndSendToUser` uses case-sensitive matching.
- **Multiple tabs receiving duplicates:** This is expected—each session receives the message. If you want only the most recent session to receive it, you must implement custom logic to track active sessions and choose which one to target.
- **User list outdated:** Our presence updates run every 15 seconds. For immediate updates, listen to `SessionConnectEvent` and `SessionDisconnectEvent` in a `@EventListener` and broadcast presence changes to a topic.
- **Performance concerns:** Each call to `convertAndSendToUser` involves looking up the user’s sessions. With thousands of users, consider caching session lookups or using a broker relay (RabbitMQ, ActiveMQ) that scales user destination resolution.

## Extending the Feature Set

With user destinations working, consider implementing:

- **Notification badges:** Count unread direct messages per conversation and display badges in the UI.
- **Threaded conversations:** Group messages by recipient and store them in a database for persistence.
- **Typing indicators in DMs:** Reuse the typing channel to show when a specific user is composing a private message.
- **Access control lists:** Restrict who can message whom (for example, only within the same team or friend list).

These upgrades demonstrate the flexibility of Spring’s messaging architecture when combined with user identity.

## Summary and Next Steps

We have successfully personalized our STOMP toy project by leveraging user destinations. The server now delivers private messages via `/user/queue/direct`, and the client can manage direct conversations alongside the public chat. This pattern applies to any personalized data, from notifications to per-user dashboards.

In Article 8 we will tackle high-frequency collaboration scenarios by streaming cursor positions. Before moving on, make sure you can:

- Send authenticated private messages between users and observe the correct delivery.
- Update the presence list to reflect connected users.
- Handle errors gracefully when messaging fails.

Armed with user destinations, we are ready to explore more demanding real-time features that push the messaging infrastructure to its limits.

