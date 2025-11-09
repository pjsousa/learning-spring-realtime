# Deciphering STOMP Messaging: The Vectors of Real-Time Partitioning

WebSockets give us a fast, bidirectional pipe; STOMP adds the language that lets us decide who should hear each message that flows through that pipe. Every frame carries headers that divide traffic by audience (topic or queue), listener (subscription), human identity (user), and the low-level channel (session and connection). Spring Boot’s messaging support reads those headers for you, but the abstraction can feel opaque until you watch the vectors in action.

In this eleventh installment we turn the microscope on those routing vectors. We will build a small “partitioning explorer” toy project: the server announces every session and subscription it sees, and the browser client sends messages through each vector so you can compare expectations with reality. The code stands alone; you can start from an empty folder, follow the snippets, and end with a working Spring Boot + plain HTML application that visualizes STOMP’s routing semantics.

## The Six Routing Vectors

STOMP’s core commands (`CONNECT`, `SUBSCRIBE`, `SEND`, `MESSAGE`) include headers that Spring translates into routing decisions. We will refer to those headers as vectors because they point messages toward specific targets.

| Vector | Header / Concept | What It Partitions | Spring Hook |
| --- | --- | --- | --- |
| Destination (`/topic/*`, `/queue/*`) | `destination` | Logical broadcast or point-to-point channel | `@SendTo`, `SimpMessagingTemplate.convertAndSend` |
| Subscription | `id` on `SUBSCRIBE` | Multiple listeners on one connection | `simpSubscriptionId`, `SessionSubscribeEvent` |
| User | Authenticated `Principal` | All tabs/devices owned by the same identity | `/user/**`, `convertAndSendToUser` |
| Session | `simpSessionId` | Application-level context for a single STOMP connection | `SessionConnectedEvent`, `simpSessionId` header |
| Connection | Handshake metadata | Physical WebSocket/TCP link | Handshake attributes, `SessionConnectEvent` |
| Queue vs Topic hint | Destination prefix | Delivery style (fan-out vs single consumer) | Broker configuration |

Our explorer surfaces each vector simultaneously so you can reason about combinations such as “send to user Alice, but only on session XYZ because that tab requested a private feed”.

## Minimal Project Setup

Generate a Gradle project with Java 21 using Spring Initializr (web, websocket, actuator, validation). From the command line:

```bash
curl https://start.spring.io/starter.zip \
  -d type=gradle-project \
  -d language=java \
  -d javaVersion=21 \
  -d dependencies=websocket,web,validation,actuator \
  -d name=partitioning-explorer \
  -o partitioning-explorer.zip
unzip partitioning-explorer.zip
cd partitioning-explorer
```

Add the dependencies `spring-boot-starter-websocket`, `spring-boot-starter-web`, `spring-boot-starter-validation`, and `spring-boot-starter-actuator` to your build file (the Initializr template already sets up the plugins and repositories). The generated `PartitioningExplorerApplication` class marked with `@SpringBootApplication` remains unchanged.

With the scaffolding ready, we can wire the WebSocket gateway, the tracker, and the messaging endpoints.

## Configuring the STOMP Gateway

We expose a SockJS-enabled endpoint and configure the simple broker to understand `/topic`, `/queue`, and a custom `/session-queue` prefix for session-specific replies. A small handshake handler synthesizes a `Principal` from the `?user=` query parameter so we can emulate authenticated users without involving Spring Security.

```java
package com.example.partitioning.config;

import java.net.URLDecoder;
import java.nio.charset.StandardCharsets;
import java.security.Principal;
import java.util.Map;
import java.util.UUID;

import org.springframework.context.annotation.Configuration;
import org.springframework.http.server.ServerHttpRequest;
import org.springframework.http.server.ServerHttpResponse;
import org.springframework.lang.NonNull;
import org.springframework.messaging.simp.config.MessageBrokerRegistry;
import org.springframework.web.socket.WebSocketHandler;
import org.springframework.web.socket.config.annotation.EnableWebSocketMessageBroker;
import org.springframework.web.socket.config.annotation.StompEndpointRegistry;
import org.springframework.web.socket.config.annotation.WebSocketMessageBrokerConfigurer;
import org.springframework.web.socket.server.HandshakeInterceptor;
import org.springframework.web.socket.server.support.DefaultHandshakeHandler;

@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/ws-partition")
                .setHandshakeHandler(new QueryParamPrincipalHandler())
                .addInterceptors(new FingerprintInterceptor())
                .setAllowedOriginPatterns("*")
                .withSockJS();
    }

    @Override
    public void configureMessageBroker(MessageBrokerRegistry registry) {
        registry.setApplicationDestinationPrefixes("/app");
        registry.setUserDestinationPrefix("/user");
        registry.enableSimpleBroker("/topic", "/queue", "/session-queue");
    }

    private static class QueryParamPrincipalHandler extends DefaultHandshakeHandler {
        @Override
        protected Principal determineUser(ServerHttpRequest request, WebSocketHandler wsHandler, Map<String, Object> attributes) {
            var query = request.getURI().getQuery();
            if (query != null) {
                for (String pair : query.split("&")) {
                    var kv = pair.split("=", 2);
                    if (kv.length == 2 && kv[0].equals("user") && !kv[1].isBlank()) {
                        String decoded = URLDecoder.decode(kv[1], StandardCharsets.UTF_8);
                        return () -> decoded;
                    }
                }
            }
            return () -> "guest-" + UUID.randomUUID();
        }
    }

    private static class FingerprintInterceptor implements HandshakeInterceptor {
        @Override
        public boolean beforeHandshake(@NonNull ServerHttpRequest request, @NonNull ServerHttpResponse response,
                                       @NonNull WebSocketHandler wsHandler, @NonNull Map<String, Object> attributes) {
            attributes.put("connectionId", UUID.randomUUID().toString());
            attributes.put("ipAddress", request.getRemoteAddress() != null ? request.getRemoteAddress().toString() : "unknown");
            return true;
        }

        @Override
        public void afterHandshake(@NonNull ServerHttpRequest request, @NonNull ServerHttpResponse response,
                                   @NonNull WebSocketHandler wsHandler, Exception ex) {
            // nothing to do
        }
    }
}
```

This configuration gives us three core building blocks:

- `destination` prefixes map to broker routes (`/topic` fan-out, `/queue` single consumer, `/session-queue` ad-hoc).
- Every browser connection receives a generated `connectionId`, accessible from Spring events.
- `Principal` names come directly from the query string, allowing you to simulate multiple identities quickly.

## Tracking Sessions, Connections, and Subscriptions

Spring publishes lifecycle events whenever a client connects, subscribes, or disconnects. A lightweight tracker can listen to those events and retain a map keyed by session ID. The tracker also exposes a REST endpoint so the browser can poll and render what the server believes about each connection.

```java
package com.example.partitioning.observation;

import java.util.ArrayList;
import java.util.Collection;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

import org.springframework.context.event.EventListener;
import org.springframework.messaging.simp.SimpMessageHeaderAccessor;
import org.springframework.stereotype.Component;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.socket.messaging.SessionConnectEvent;
import org.springframework.web.socket.messaging.SessionDisconnectEvent;
import org.springframework.web.socket.messaging.SessionSubscribeEvent;
import org.springframework.web.socket.messaging.SessionUnsubscribeEvent;

@Component
public class VectorTracker {

    private final Map<String, String> connectionIds = new ConcurrentHashMap<>();
    private final Map<String, String> users = new ConcurrentHashMap<>();
    private final Map<String, Map<String, String>> subscriptions = new ConcurrentHashMap<>();

    @EventListener
    public void handleConnect(SessionConnectEvent event) {
        var accessor = SimpMessageHeaderAccessor.wrap(event.getMessage());
        var sessionId = accessor.getSessionId();
        if (sessionId == null) {
            return;
        }
        connectionIds.put(sessionId, attribute(accessor, "connectionId"));
        users.put(sessionId, accessor.getUser() != null ? accessor.getUser().getName() : "anonymous");
        subscriptions.computeIfAbsent(sessionId, id -> new ConcurrentHashMap<>());
    }

    @EventListener
    public void handleDisconnect(SessionDisconnectEvent event) {
        var sessionId = event.getSessionId();
        connectionIds.remove(sessionId);
        users.remove(sessionId);
        subscriptions.remove(sessionId);
    }

    @EventListener
    public void handleSubscribe(SessionSubscribeEvent event) {
        var accessor = SimpMessageHeaderAccessor.wrap(event.getMessage());
        subscriptions.computeIfAbsent(accessor.getSessionId(), id -> new ConcurrentHashMap<>())
                .put(accessor.getSubscriptionId(), accessor.getDestination());
    }

    @EventListener
    public void handleUnsubscribe(SessionUnsubscribeEvent event) {
        var accessor = SimpMessageHeaderAccessor.wrap(event.getMessage());
        var map = subscriptions.get(accessor.getSessionId());
        if (map != null) {
            map.remove(accessor.getSubscriptionId());
            if (map.isEmpty()) {
                subscriptions.remove(accessor.getSessionId());
            }
        }
    }

    public Collection<SessionView> currentState() {
        var result = new ArrayList<SessionView>(connectionIds.size());
        connectionIds.forEach((sessionId, connectionId) -> {
            var subs = subscriptions.getOrDefault(sessionId, Map.of());
            result.add(new SessionView(
                    sessionId,
                    connectionId,
                    users.getOrDefault(sessionId, "anonymous"),
                    Map.copyOf(subs)));
        });
        return result;
    }

    private static String attribute(SimpMessageHeaderAccessor accessor, String key) {
        var attrs = accessor.getSessionAttributes();
        var value = attrs != null ? attrs.get(key) : null;
        return value != null ? value.toString() : "unknown";
    }

    public record SessionView(
            String sessionId,
            String connectionId,
            String user,
            Map<String, String> subscriptions) { }
}

@RestController
class VectorsRestController {

    private final VectorTracker tracker;

    VectorsRestController(VectorTracker tracker) {
        this.tracker = tracker;
    }

    @GetMapping("/vectors/sessions")
    Collection<VectorTracker.SessionView> sessions() {
        return tracker.currentState();
    }
}
```

`SessionView` now captures three of the six vectors: the session ID, the recorded connection fingerprint, and the active subscription IDs with their destinations. The browser will poll `/vectors/sessions` and show these values alongside the frames it receives.

## Driving Messages Across the Vectors

The messaging controller demonstrates how each vector is used in day-to-day STOMP development. We broadcast announcements, enqueue support tickets, send user-targeted whispers, and reply to the current session. A dedicated mapping also echoes back the subscription ID so you can see which listener caught the response.

```java
package com.example.partitioning.messaging;

import java.security.Principal;
import java.time.Instant;

import jakarta.validation.Valid;
import jakarta.validation.constraints.NotBlank;

import org.springframework.messaging.handler.annotation.Header;
import org.springframework.messaging.handler.annotation.MessageMapping;
import org.springframework.messaging.simp.SimpMessagingTemplate;
import org.springframework.messaging.simp.annotation.SendToUser;
import org.springframework.stereotype.Controller;

@Controller
public class PartitioningController {

    private final SimpMessagingTemplate messagingTemplate;

    public PartitioningController(SimpMessagingTemplate messagingTemplate) {
        this.messagingTemplate = messagingTemplate;
    }

    @MessageMapping("/announce")
    public Announcement announce(@Valid Announcement input, Principal principal) {
        String sender = resolveUser(principal, input.from());
        var payload = new Announcement(sender, input.body(), Instant.now());
        return payload;
    }

    @MessageMapping("/support")
    public void support(@Valid SupportTicket ticket, Principal principal) {
        String sender = resolveUser(principal, ticket.from());
        var payload = new SupportTicket(sender, ticket.topic(), ticket.details(), Instant.now());
        messagingTemplate.convertAndSend("/queue/support", payload);
    }

    @MessageMapping("/whisper")
    public void whisper(@Valid Whisper whisper, Principal principal) {
        String sender = resolveUser(principal, whisper.from());
        var payload = new Whisper(sender, whisper.to(), whisper.body(), Instant.now());
        messagingTemplate.convertAndSendToUser(whisper.to(), "/queue/direct", payload);
    }

    @MessageMapping("/session/ping")
    public void pingSession(@Valid SessionPing ping,
                            @Header("simpSessionId") String sessionId,
                            Principal principal) {
        var payload = new SessionPing(ping.label() + " -> " + sessionId, ping.payload(), Instant.now());
        messagingTemplate.convertAndSend("/session-queue/" + sessionId, payload);
    }

    @MessageMapping("/subscription/echo")
    @SendToUser("/queue/subscription-echo")
    public String echoSubscription(@Header("simpSubscriptionId") String subscriptionId) {
        return "You subscribed with id: " + subscriptionId;
    }

    private static String resolveUser(Principal principal, String fallback) {
        if (fallback != null && !fallback.isBlank()) {
            return fallback;
        }
        return principal != null ? principal.getName() : "anonymous";
    }

    public record Announcement(String from, @NotBlank String body, Instant at) { }
    public record SupportTicket(String from, @NotBlank String topic, String details, Instant at) { }
    public record Whisper(String from, @NotBlank String to, @NotBlank String body, Instant at) { }
    public record SessionPing(String label, String payload, Instant at) { }
}
```

- Returning an object from `@MessageMapping("/announce")` sends it to `/topic/announce`, illustrating the topic vector (fan-out).
- `convertAndSend("/queue/support", ...)` exercises the queue vector (one consumer per message).
- `convertAndSendToUser` targets the authenticated user. Open two windows as the same user to see both sessions receive the whisper.
- `simpSessionId` lets us address a specific session via the custom `/session-queue/{id}` channel.
- The subscription echo shows the broker attaching `subscription:id` to every `MESSAGE` frame it delivers.

## Browser Explorer Client

Save the client as `src/main/resources/static/partitioning-explorer.html`. It connects through SockJS, subscribes with explicit IDs, and renders both the STOMP headers and the server-side tracker data.

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>STOMP Partitioning Explorer</title>
  <style>
    body{font-family:system-ui,sans-serif;margin:1.5rem;background:#f8fafc;color:#0f172a;}
    fieldset{border:1px solid #cbd5f5;border-radius:6px;padding:1rem;margin-bottom:1rem;background:#fff;}
    legend{font-weight:700;}
    label{display:block;margin-top:.5rem;}
    input,textarea{width:100%;padding:.45rem;border:1px solid #94a3b8;border-radius:4px;}
    button{padding:.4rem .9rem;border:none;border-radius:4px;background:#2563eb;color:#fff;font-weight:600;margin-right:.5rem;cursor:pointer;}
    pre{background:#0f172a;color:#cbd5f5;padding:.75rem;border-radius:6px;overflow-x:auto;}
  </style>
</head>
<body>
  <h1>STOMP Partitioning Explorer</h1>
  <p>Attach <code>?user=alice</code> or <code>?user=bob</code> to the URL to simulate authenticated principals. Open two tabs to observe the vectors side by side.</p>

  <fieldset>
    <legend>Connection</legend>
    <label>User override</label>
    <input id="userName" placeholder="Defaults to query string or random guest" />
    <label>Endpoint</label>
    <input id="brokerUrl" value="/ws-partition" />
    <div style="margin-top:0.75rem;">
      <button id="connectBtn">Connect</button>
      <button id="disconnectBtn">Disconnect</button>
    </div>
    <p id="connectionInfo"></p>
  </fieldset>

  <fieldset>
    <legend>Send Frames</legend>
    <label>Broadcast body → `/app/announce`</label>
    <input id="announceBody" />
    <label>Support topic/details → `/app/support`</label>
    <input id="supportTopic" placeholder="Topic" />
    <textarea id="supportDetails" rows="3" placeholder="Details"></textarea>
    <label>Whisper to user → `/app/whisper`</label>
    <input id="whisperTo" placeholder="Recipient user" />
    <input id="whisperBody" placeholder="Message" />
    <label>Session ping payload → `/app/session/ping`</label>
    <input id="pingLabel" placeholder="Label" />
    <input id="pingPayload" placeholder="Payload" />
    <div style="margin-top:0.75rem;">
      <button id="broadcastBtn">Broadcast</button>
      <button id="supportBtn">Queue Ticket</button>
      <button id="whisperBtn">Whisper</button>
      <button id="pingBtn">Ping Session</button>
    </div>
  </fieldset>

  <fieldset>
    <legend>Frames Received</legend>
    <pre id="messageLog"></pre>
  </fieldset>

  <fieldset>
    <legend>Server Registry `/vectors/sessions`</legend>
    <pre id="registryLog"></pre>
  </fieldset>

  <script src="https://cdn.jsdelivr.net/npm/sockjs-client@1/dist/sockjs.min.js"></script>
  <script src="https://cdn.jsdelivr.net/npm/stompjs@2.3.3/lib/stomp.min.js"></script>
  <script>
    let stompClient=null,sessionId=null,pollHandle=null;
    const log=document.getElementById('messageLog');
    const registryLog=document.getElementById('registryLog');
    const info=document.getElementById('connectionInfo');

    document.getElementById('connectBtn').addEventListener('click',connect);
    document.getElementById('disconnectBtn').addEventListener('click',disconnect);
    document.getElementById('broadcastBtn').addEventListener('click',sendBroadcast);
    document.getElementById('supportBtn').addEventListener('click',sendSupport);
    document.getElementById('whisperBtn').addEventListener('click',sendWhisper);
    document.getElementById('pingBtn').addEventListener('click',sendPing);

    function connect(){
      const params=new URLSearchParams(window.location.search);
      const userInput=document.getElementById('userName').value.trim();
      const user=userInput||params.get('user')||`guest-${Math.random().toString(36).slice(2,6)}`;
      const endpoint=document.getElementById('brokerUrl').value;
      const socket=new SockJS(`${endpoint}?user=${encodeURIComponent(user)}`);
      stompClient=Stomp.over(socket);
      stompClient.heartbeat.incoming=10000;
      stompClient.heartbeat.outgoing=10000;
      stompClient.connect({},frame=>{
        sessionId=frame.headers['session'];
        info.textContent=`Connected as ${user} (session ${sessionId})`;
        subscribeAll();
        startPolling();
      },error=>info.textContent=`Connection error: ${error.body||error}`);
    }

    function subscribeAll(){
      stompClient.subscribe('/topic/announce',f=>appendFrame('topic',f),{id:'topic-announce'});
      stompClient.subscribe('/queue/support',f=>appendFrame('queue',f),{id:'queue-support'});
      stompClient.subscribe('/user/queue/direct',f=>appendFrame('user',f),{id:'user-direct'});
      stompClient.subscribe(`/session-queue/${sessionId}`,f=>appendFrame('session',f),{id:`session-${sessionId}`});
      stompClient.subscribe('/user/queue/subscription-echo',f=>appendFrame('echo',f),{id:'subscription-echo'});
      stompClient.send('/app/subscription/echo',{},JSON.stringify({}));
    }

    function appendFrame(label,frame){
      const headers=JSON.stringify(frame.headers,null,2);
      const payload=frame.body?JSON.stringify(JSON.parse(frame.body),null,2):null;
      log.textContent=`[${new Date().toLocaleTimeString()}] ${label}\nheaders: ${headers}\npayload: ${payload}\n\n`+log.textContent;
    }

    const sendJson=(destination,body)=>stompClient.send(destination,{},JSON.stringify(body));

    function sendBroadcast(){sendJson('/app/announce',{body:document.getElementById('announceBody').value});}
    function sendSupport(){sendJson('/app/support',{topic:document.getElementById('supportTopic').value,details:document.getElementById('supportDetails').value});}
    function sendWhisper(){sendJson('/app/whisper',{to:document.getElementById('whisperTo').value,body:document.getElementById('whisperBody').value});}
    function sendPing(){sendJson('/app/session/ping',{label:document.getElementById('pingLabel').value,payload:document.getElementById('pingPayload').value});}

    function startPolling(){
      pollHandle=setInterval(async()=>{
        const response=await fetch('/vectors/sessions');
        registryLog.textContent=JSON.stringify(await response.json(),null,2);
      },2000);
    }

    function disconnect(){
      if(stompClient){stompClient.disconnect(()=>info.textContent='Disconnected.');}
      stompClient=null;sessionId=null;
      if(pollHandle){clearInterval(pollHandle);pollHandle=null;}
    }
  </script>
</body>
</html>
```

## Try It Out

1. Start the server with `./gradlew bootRun`.
2. Open `http://localhost:8080/partitioning-explorer.html?user=alice` in one tab and `?user=bob` in another, then click **Connect** in each.
3. From Alice’s tab, send a broadcast, a whisper to `bob`, and a session ping. Bob’s tab shows how topic, user, and session vectors behave differently.
4. Submit several support tickets from Bob; note that only one tab logs each queue delivery. Watch the subscription echo responses and `/vectors/sessions` output to confirm the broker’s `id` headers and the active session list.

## Debugging with the Vector Data

- **Destination alignment:** When someone misses an update, compare the subscribed destination in the registry with the endpoint your controller publishes to. Mismatches jump out immediately.
- **Identity and churn:** Anonymous users or a rapid churn of session IDs hint at authentication or heartbeat issues. Fix the login handshake or adjust heartbeats before chasing phantom bugs.
- **Subscription hygiene:** The registry lists every subscription ID. Use it to spot duplicate listeners, confirm queue fan-out behaviour, and justify why only one consumer receives `/queue/*` messages.

## Conclusion

The six STOMP vectors give you surgical control over how real-time messages flow: topics for broadcasting, queues for work distribution, user destinations for identity, sessions for per-tab workflows, connections for low-level diagnostics, and subscription IDs for listener bookkeeping. The partitioning explorer couples Spring’s event stream with a simple HTML dashboard so you can watch those axes interact in real time. Keep it handy when designing new messaging features or debugging tricky routing issues—the clarity it provides will save hours of guesswork and turn STOMP’s headers into a trusted ally.

