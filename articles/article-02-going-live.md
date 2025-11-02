# Going Live: Implementing Real-Time Server Updates using `@SendTo` and Scheduling

## Why Real-Time Server Push Matters

In Article 1 we built a simple request/response style STOMP demo, but it still relied on clients pinging the server. Many real-world use cases—live scoreboards, telemetry dashboards, auction tickers, IoT readings—demand that the server publish fresh data proactively. This article shows how Spring’s messaging architecture lets any bean broadcast to subscribed clients, no browser interaction required. We will:

- Keep the tooling lightweight: Spring Boot, Spring Messaging/WebSocket, plain HTML/JS client (no frameworks).
- Configure the broker so that the server can push to `/topic/prices`.
- Generate a continuous stream of fake stock quotes via `@Scheduled`.
- Demonstrate both the declarative `@SendTo` annotation and the imperative `SimpMessagingTemplate`.
- Deliver an HTML client that a reader can open immediately to watch the feed.

These 1500–2000 words assume only basic familiarity with Java and Maven/Gradle. If you skipped Article 1, follow the setup below and you’ll be caught up.

## Project Setup Recap (or Start Fresh)

We’ll extend the same Gradle/Maven project scaffolding from the first installment. If you already have it, verify you’re on Spring Boot 3.2+ with Java 17; otherwise create a new project:

1. Use Spring Initializr (<https://start.spring.io/>) or your IDE wizard.
2. Choose:
   - Project: Maven (or Gradle if you prefer).
   - Language: Java.
   - Spring Boot: 3.2.x.
   - Packaging: Jar.
   - Dependencies: `Spring Web`, `Spring Websocket`, `Spring Boot Actuator` (optional for future observability), `Spring Boot DevTools` (optional), and `Lombok` if you like it.
3. Group: `com.example`.
4. Artifact: `stomp-toy` (or similar).

If you lean on Maven, your `pom.xml` dependencies should look like:

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
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter</artifactId>
    </dependency>
    <!-- Optional helpers -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-devtools</artifactId>
        <scope>runtime</scope>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
</dependencies>
```

Gradle users: include `implementation 'org.springframework.boot:spring-boot-starter-websocket'` and friends.

Next, enable scheduling in `src/main/java/com/example/stomptoy/StompToyApplication.java`:

```java
package com.example.stomptoy;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.scheduling.annotation.EnableScheduling;

@SpringBootApplication
@EnableScheduling
public class StompToyApplication {

    public static void main(String[] args) {
        SpringApplication.run(StompToyApplication.class, args);
    }
}
```

`@EnableScheduling` tells Spring to scan for `@Scheduled` methods we’ll add shortly.

Finally, set server port if you want something other than 8080. In `src/main/resources/application.properties`:

```
server.port=8080
spring.main.banner-mode=off
```

We now have a runnable Spring Boot app.

## WebSocket/STOMP Configuration Refresher

Everything a STOMP application does rides on the broker configuration. Create `src/main/java/com/example/stomptoy/config/WebSocketConfig.java`:

```java
package com.example.stomptoy.config;

import org.springframework.context.annotation.Configuration;
import org.springframework.messaging.simp.config.MessageBrokerRegistry;
import org.springframework.web.socket.config.annotation.EnableWebSocketMessageBroker;
import org.springframework.web.socket.config.annotation.StompEndpointRegistry;
import org.springframework.web.socket.config.annotation.WebSocketMessageBrokerConfigurer;

@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {

    @Override
    public void configureMessageBroker(MessageBrokerRegistry registry) {
        registry.enableSimpleBroker("/topic");
        registry.setApplicationDestinationPrefixes("/app");
    }

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/ws-ticker")
                .setAllowedOriginPatterns("*")
                .withSockJS();
    }
}
```

Key points:

- `/app/**` destinations route into `@MessageMapping` methods.
- `/topic/**` destinations are broadcast by the simple broker to all subscribers.
- The `/ws-ticker` endpoint is what our browser client will connect to. SockJS ensures compatibility if native WebSocket is blocked.

This config remains almost identical across articles, providing readers with consistency.

## Modeling a Price Update

Create a compact DTO to send over STOMP. Java 17 records keep it tidy:

```java
package com.example.stomptoy.ticker;

import java.math.BigDecimal;
import java.time.Instant;

public record PriceUpdate(
        String symbol,
        BigDecimal price,
        Instant timestamp
) {}
```

We can optionally add `toString()` or builder logic, but the record suffices. STOMP will serialize it as JSON thanks to Jackson on the classpath.

## Declarative Broadcasting with `@SendTo`

`@SendTo` works with methods that handle messages from clients. Even though this article focuses on server-only events, including a manual trigger endpoint clarifies how `@SendTo` relates to the general publish-subscribe flow.

Create `src/main/java/com/example/stomptoy/ticker/TickerController.java`:

```java
package com.example.stomptoy.ticker;

import java.math.BigDecimal;
import java.time.Instant;
import java.util.concurrent.ThreadLocalRandom;

import org.springframework.messaging.handler.annotation.MessageMapping;
import org.springframework.messaging.handler.annotation.Payload;
import org.springframework.messaging.handler.annotation.SendTo;
import org.springframework.stereotype.Controller;

@Controller
public class TickerController {

    @MessageMapping("/prices/request-once")
    @SendTo("/topic/prices")
    public PriceUpdate requestOneOff(@Payload String symbol) {
        BigDecimal price = randomPrice();
        return new PriceUpdate(symbol.toUpperCase(), price, Instant.now());
    }

    private BigDecimal randomPrice() {
        double base = ThreadLocalRandom.current().nextDouble(90.0, 130.0);
        return BigDecimal.valueOf(Math.round(base * 100.0) / 100.0);
    }
}
```

Here’s what happens:

1. A client sends a STOMP message to `/app/prices/request-once`.
2. The method executes, creates a `PriceUpdate`, and returns it.
3. `@SendTo("/topic/prices")` instructs Spring to take the returned payload and send it to the broker topic, which in turn fans it out to all subscribers.

This is an “echo by the server” flow, bridging Article 1 behavior with today’s emphasis.

## Imperative Broadcasting with `SimpMessagingTemplate`

For truly server-originated events—scheduled jobs, domain events, integration callbacks—we inject `SimpMessagingTemplate` and call `convertAndSend`. Create `src/main/java/com/example/stomptoy/ticker/PriceFeedService.java`:

```java
package com.example.stomptoy.ticker;

import java.math.BigDecimal;
import java.time.Instant;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.ThreadLocalRandom;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.messaging.simp.SimpMessagingTemplate;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Service;

@Service
public class PriceFeedService {

    private static final Logger log = LoggerFactory.getLogger(PriceFeedService.class);
    private static final String DESTINATION = "/topic/prices";

    private final SimpMessagingTemplate messagingTemplate;
    private final Map<String, BigDecimal> lastPrices = new ConcurrentHashMap<>();

    public PriceFeedService(SimpMessagingTemplate messagingTemplate) {
        this.messagingTemplate = messagingTemplate;
        lastPrices.put("ACME", BigDecimal.valueOf(120.12));
        lastPrices.put("GLOBO", BigDecimal.valueOf(98.45));
        lastPrices.put("MEGACORP", BigDecimal.valueOf(75.30));
    }

    @Scheduled(initialDelay = 1000, fixedRate = 2000)
    public void publishTicks() {
        lastPrices.keySet().forEach(symbol -> {
            PriceUpdate update = nextUpdate(symbol);
            messagingTemplate.convertAndSend(DESTINATION, update);
            log.debug("Broadcast {} -> {}", symbol, update.price());
        });
    }

    private PriceUpdate nextUpdate(String symbol) {
        BigDecimal current = lastPrices.get(symbol);
        double delta = ThreadLocalRandom.current().nextDouble(-1.5, 1.5);
        BigDecimal updated = current.add(BigDecimal.valueOf(delta)).setScale(2, BigDecimal.ROUND_HALF_UP);
        lastPrices.put(symbol, updated);
        return new PriceUpdate(symbol, updated, Instant.now());
    }
}
```

Highlights:

- `@Scheduled(initialDelay = 1000, fixedRate = 2000)` waits a second after startup and then emits every two seconds.
- We keep state for a few sample symbols and mutate them slightly to mimic a live feed.
- `SimpMessagingTemplate.convertAndSend("/topic/prices", update)` injects the payload into the broker channel—no client involvement required.
- Logging for each broadcast helps debugging and shows activity in the console.

If you prefer more realistic randomness, swap in `BigDecimal` math tuned to your domain.

## Optional: Broadcasting from Outside Beans

Because `SimpMessagingTemplate` is a regular Spring bean, you can inject it anywhere:

```java
@Component
public class InventoryWatcher {

    private final SimpMessagingTemplate messagingTemplate;

    public InventoryWatcher(SimpMessagingTemplate messagingTemplate) {
        this.messagingTemplate = messagingTemplate;
    }

    @EventListener(InventoryLowEvent.class)
    public void handleLowInventory(InventoryLowEvent event) {
        PriceUpdate update = new PriceUpdate(event.productSku(), event.suggestedPrice(), Instant.now());
        messagingTemplate.convertAndSend("/topic/prices", update);
    }
}
```

This pattern will reappear in later articles when we integrate REST controllers, database triggers, and third-party APIs.

## Crafting a Browser Client

Readers need an immediate smoke test. Create `src/main/resources/static/index.html`:

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Live Price Ticker</title>
    <style>
        body { font-family: system-ui, sans-serif; margin: 2rem; }
        #status { margin-bottom: 1rem; }
        table { border-collapse: collapse; width: 100%; max-width: 480px; }
        th, td { border: 1px solid #ddd; padding: 0.5rem; text-align: left; }
        #feed { font-variant-numeric: tabular-nums; }
        .up { color: #1b7c1b; }
        .down { color: #a82323; }
    </style>
</head>
<body>
<h1>Live Price Ticker</h1>
<p id="status">Connecting…</p>

<table>
    <thead>
    <tr>
        <th>Symbol</th>
        <th>Price</th>
        <th>Last Updated</th>
    </tr>
    </thead>
    <tbody id="feed"></tbody>
</table>

<section>
    <h2>Request a One-Off Price</h2>
    <input id="symbolInput" type="text" placeholder="Symbol (e.g. ACME)">
    <button id="requestBtn">Request</button>
</section>

<script src="https://cdn.jsdelivr.net/npm/sockjs-client@1/dist/sockjs.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/stompjs@2.3.3/lib/stomp.min.js"></script>
<script>
    const feed = document.getElementById('feed');
    const status = document.getElementById('status');
    const symbolInput = document.getElementById('symbolInput');
    const requestBtn = document.getElementById('requestBtn');

    const socket = new SockJS('/ws-ticker');
    const stompClient = Stomp.over(socket);
    stompClient.debug = null;

    const prices = new Map();

    stompClient.connect({}, frame => {
        status.textContent = 'Connected: ' + frame;
        stompClient.subscribe('/topic/prices', message => {
            const update = JSON.parse(message.body);
            prices.set(update.symbol, update);
            render();
        });
    }, error => {
        status.textContent = 'Connection error: ' + error;
    });

    requestBtn.addEventListener('click', () => {
        const symbol = symbolInput.value.trim();
        if (!symbol) {
            alert('Enter a symbol');
            return;
        }
        stompClient.send('/app/prices/request-once', {}, symbol);
        symbolInput.value = '';
    });

    function render() {
        feed.innerHTML = '';
        Array.from(prices.values())
            .sort((a, b) => a.symbol.localeCompare(b.symbol))
            .forEach(update => {
                const existing = prices.get(update.symbol);
                const direction = existing && existing.price <= update.price ? 'up' : 'down';

                const row = document.createElement('tr');

                const symbolCell = document.createElement('td');
                symbolCell.textContent = update.symbol;
                row.appendChild(symbolCell);

                const priceCell = document.createElement('td');
                priceCell.textContent = Number(update.price).toFixed(2);
                priceCell.className = direction;
                row.appendChild(priceCell);

                const timestampCell = document.createElement('td');
                timestampCell.textContent = new Date(update.timestamp).toLocaleTimeString();
                row.appendChild(timestampCell);

                feed.appendChild(row);
            });
    }
</script>
</body>
</html>
```

What the client does:

- Opens a SockJS connection to `/ws-ticker`.
- Subscribes to `/topic/prices` and renders each update.
- Provides the “request once” button to demonstrate the `@SendTo` path.
- Uses vanilla DOM manipulation to keep dependencies nonexistent.

## Running the Demo

1. Start the application: `./mvnw spring-boot:run` or `./gradlew bootRun`.
2. Watch the console. After a second you’ll see logs like `Broadcast ACME -> 120.73`.
3. Open `http://localhost:8080/` in one or more browsers. Within two seconds, the table populates and updates.
4. Click “Request” with `ACME` or any other symbol to see a one-off request flow (the new symbol will appear instantly for everyone).
5. Open the page in a second tab to appreciate fan-out behavior—every broadcast updates all tabs simultaneously.

If nothing appears:

- Check the browser console for WebSocket errors (CORS, port mismatch).
- Confirm the schedule is firing (`@EnableScheduling` is present and the `@Scheduled` method is in a managed bean).
- Ensure the endpoint path in Java (`/ws-ticker`) matches the client SockJS URL.
- If on a corporate network, SockJS might fall back to XHR; keep the endpoint accessible over HTTP.

## Understanding the Message Flow

This architecture hinges on three channels:

- **Client → Application channel (`/app`)**: STOMP frames from the browser that target `@MessageMapping` methods.
- **Application → Broker channel**: When the controller returns a value or when `SimpMessagingTemplate` is used, the payload enters the broker channel.
- **Broker → Client channel (`/topic`)**: The simple broker dispatches messages to any subscriber on that destination.

`@SendTo` and `SimpMessagingTemplate` both push into the broker channel; the difference is where they execute. `@SendTo` suits controller methods that already have a client interaction, whereas `SimpMessagingTemplate` unlocks push from scheduled jobs, asynchronous listeners, or even batch processes that run with no open WebSocket sessions.

## Extending the Example

Suggestions for readers:

- Add more symbols or jitter rates to simulate different markets.
- Persist snapshots every minute so reconnecting clients can fetch the last known price history.
- Throttle updates to specific clients or implement rate limiting via `SimpMessageSendingOperations.convertAndSendToUser`.
- Expose a REST endpoint that toggles the scheduler’s speed.
- Enhance visuals with Chart.js or D3 once the data stream is familiar.

## Wrapping Up

We just crossed a key milestone: the server now owns the cadence of communication. By combining `@SendTo` and `SimpMessagingTemplate`, we grant every Spring bean the power to publish to STOMP destinations. The HTML client stays simple yet proves the wiring is correct.

In Article 3 we will shift the lens toward delivering messages to specific users. We’ll explore `@SendToUser`, user registries, and private channels so you can implement notification trays or direct messages. Until then, experiment with the ticker—perhaps plug in real API data—and keep the broker humming.
