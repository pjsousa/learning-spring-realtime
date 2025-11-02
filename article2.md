# Going Live: Implementing Real-Time Server Updates using @SendTo and Scheduling

In the first article of this series we built the simplest possible STOMP pipeline: a user-initiated message flowed from a browser to a Spring Boot controller and back out to subscribed clients. Real-time applications, however, often need to originate data on the server side—think market tickers, IoT sensor feeds, or collaborative dashboards that constantly refresh themselves. In this second installment we will teach our toy application to generate data on its own schedule and broadcast those updates to every connected browser without requiring user interaction.

We will walk through the following milestones:

- Add scheduling support to our Spring Boot project.
- Use `@SendTo` and `SimpMessagingTemplate` to publish server-generated payloads.
- Design a simple stock price simulation service that emits updates every few seconds.
- Update the plain HTML client to subscribe to a dedicated topic and render the streaming data.

By the end, you will have a live ticker demo where new messages appear automatically and in sync across every browser session.

## Revisiting the Project Setup

If you followed along with Article 1, you already have a Spring Boot project named something like `stomp-toy-01` with WebSocket support enabled and a basic ping-pong controller. We will evolve that project instead of starting from scratch. If you are jumping into the series here, consult the previous article to bootstrap the core configuration; you only need the `spring-boot-starter-websocket` dependency and a `WebSocketConfig` class with `@EnableWebSocketMessageBroker`.

To support scheduled tasks, add the Spring Boot scheduling starter. With Maven, open your `pom.xml` and include:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter</artifactId>
</dependency>
```

The “starter” dependency already includes scheduling support. If you want a more explicit dependency you can add `spring-context` and use `@EnableScheduling`, but the starter keeps your POM simple.

In our main application class (typically `StompToyApplication`), enable scheduling:

```java
@SpringBootApplication
@EnableScheduling
public class StompToyApplication {

    public static void main(String[] args) {
        SpringApplication.run(StompToyApplication.class, args);
    }
}
```

This annotation allows us to create beans with methods annotated by `@Scheduled`, which Spring executes according to cron or fixed interval expressions.

## Defining the Real-Time Domain: A Stock Ticker

We need an example that feels realistic but remains approachable. A stock ticker is the perfect fit: prices fluctuate frequently, the data is read-only for clients, and latency matters because users expect to see changes instantly.

Create a new package `com.example.stomplive.ticker` to organize the ticker components. We will define:

- A simple data class `StockPrice`.
- A `PriceGenerator` service that simulates price movements.
- A `TickerPublisher` component that broadcasts updates every two seconds.

### Modeling Stock Prices

Let’s keep the payload straightforward—each update contains a ticker symbol, the latest price, the absolute change, and the timestamp of the update.

```java
package com.example.stomplive.ticker;

import java.math.BigDecimal;
import java.time.Instant;

public record StockPrice(String symbol,
                         BigDecimal price,
                         BigDecimal change,
                         Instant lastUpdated) {
}
```

We use `BigDecimal` for accurate currency representation and `Instant` for the timestamp. The record syntax makes the class concise while still giving us a readable `toString()` for logging.

### Generating Fake Market Data

Next, implement a service that mutates prices slightly on every tick. We will maintain a map of tracked symbols and tweak each price using a simple random walk algorithm.

```java
package com.example.stomplive.ticker;

import java.math.BigDecimal;
import java.math.RoundingMode;
import java.security.SecureRandom;
import java.time.Instant;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

import org.springframework.stereotype.Component;

@Component
public class PriceGenerator {

    private final SecureRandom random = new SecureRandom();
    private final Map<String, BigDecimal> currentPrices = new ConcurrentHashMap<>();

    public PriceGenerator() {
        currentPrices.put("SPRING", BigDecimal.valueOf(125.43));
        currentPrices.put("JAVA", BigDecimal.valueOf(92.15));
        currentPrices.put("BOOT", BigDecimal.valueOf(55.87));
    }

    public StockPrice nextPrice(String symbol) {
        BigDecimal base = currentPrices.computeIfAbsent(symbol, key -> BigDecimal.valueOf(50));
        BigDecimal delta = BigDecimal.valueOf(random.nextGaussian() * 0.75)
                .setScale(2, RoundingMode.HALF_UP);
        BigDecimal updated = base.add(delta).max(BigDecimal.ONE);
        currentPrices.put(symbol, updated);
        return new StockPrice(symbol, updated, delta, Instant.now());
    }

    public Iterable<String> trackedSymbols() {
        return currentPrices.keySet();
    }
}
```

The `SecureRandom` provides good variability. We guard against negative prices by clamping to at least 1.00. This component is stateless from Spring’s perspective, but it stores the last known price per symbol in a thread-safe map so that each subsequent update builds on the previous value.

### Broadcasting with SimpMessagingTemplate

To push updates to clients, we have two options: use `@SendTo` on a `@MessageMapping` method, or inject `SimpMessagingTemplate` and call `convertAndSend`. In this article we will demonstrate both. The scheduled publisher will inject the template so it can broadcast without being triggered by an incoming message.

```java
package com.example.stomplive.ticker;

import java.time.Duration;
import java.util.List;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.messaging.simp.SimpMessagingTemplate;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Component;

@Component
public class TickerPublisher {

    private static final Logger log = LoggerFactory.getLogger(TickerPublisher.class);
    private final SimpMessagingTemplate messagingTemplate;
    private final PriceGenerator priceGenerator;
    private final List<String> symbols = List.of("SPRING", "JAVA", "BOOT");

    public TickerPublisher(SimpMessagingTemplate messagingTemplate, PriceGenerator priceGenerator) {
        this.messagingTemplate = messagingTemplate;
        this.priceGenerator = priceGenerator;
    }

    @Scheduled(initialDelay = 2000, fixedRate = 2000)
    public void publishPrices() {
        symbols.forEach(symbol -> {
            StockPrice price = priceGenerator.nextPrice(symbol);
            messagingTemplate.convertAndSend("/topic/prices", price);
            log.debug("Published {}", price);
        });
    }
}
```

Every two seconds the `publishPrices` method iterates through the symbol list, generates the next price, and calls `convertAndSend`. The destination `/topic/prices` matches the prefix we configured earlier with `enableSimpleBroker`. As soon as the template sends the message, any subscribed clients receive it instantly.

The `initialDelay` ensures the first batch of updates is sent after the application has fully started, giving browsers time to connect.

## Adding a REST Endpoint for Health Checks (Optional)

While not required for STOMP functionality, exposing a simple HTTP endpoint can help confirm that the application is running. Create a controller under `com.example.stomplive.web`:

```java
package com.example.stomplive.web;

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class HealthController {

    @GetMapping("/health")
    public String health() {
        return "OK";
    }
}
```

A quick `curl http://localhost:8080/health` confirms the app is reachable even if you are troubleshooting client connections.

## Creating a Server-Side “Command” Endpoint

Although the ticker runs automatically, providing a way to trigger a manual refresh can be helpful for demos. We will create a `TickerController` with an application destination `/app/request-prices` that returns a batch on demand.

```java
package com.example.stomplive.ticker;

import java.util.stream.StreamSupport;

import org.springframework.messaging.handler.annotation.MessageMapping;
import org.springframework.messaging.handler.annotation.SendTo;
import org.springframework.stereotype.Controller;

@Controller
public class TickerController {

    private final PriceGenerator priceGenerator;

    public TickerController(PriceGenerator priceGenerator) {
        this.priceGenerator = priceGenerator;
    }

    @MessageMapping("/request-prices")
    @SendTo("/topic/prices")
    public Iterable<StockPrice> handlePriceRequest() {
        return StreamSupport.stream(priceGenerator.trackedSymbols().spliterator(), false)
                .map(priceGenerator::nextPrice)
                .toList();
    }
}
```

Whenever a client sends a STOMP SEND frame to `/app/request-prices`, the controller fetches the latest price for each tracked symbol and returns a collection. The `@SendTo` annotation broadcasts the list to `/topic/prices`, which means the listening clients must be ready to handle either a single `StockPrice` or an array. We address this in the client logic.

## Enhancing the HTML Client for Streaming Data

Next, upgrade the browser client to render a live table of stock prices. Replace the content of `src/main/resources/static/index.html` with the following more sophisticated UI:

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Spring STOMP Live Ticker</title>
    <style>
        :root {
            color-scheme: light dark;
            font-family: 'Segoe UI', system-ui, sans-serif;
        }
        body { margin: 2rem; }
        h1 { margin-bottom: 0.5rem; }
        #status { margin-bottom: 1.5rem; font-weight: 600; }
        table { border-collapse: collapse; width: 100%; max-width: 40rem; }
        th, td { border: 1px solid #ccc; padding: 0.5rem 0.75rem; text-align: right; }
        th { background: #f0f3f5; text-align: left; }
        tr.up td.change { color: #1f9d55; }
        tr.down td.change { color: #e3342f; }
        button { margin-right: 0.5rem; padding: 0.5rem 0.75rem; }
        #log { margin-top: 2rem; font-family: 'Fira Mono', monospace; white-space: pre-wrap; }
    </style>
</head>
<body>
<h1>Spring STOMP Live Ticker</h1>
<div id="status">Disconnected</div>
<div>
    <button id="connect">Connect</button>
    <button id="disconnect" disabled>Disconnect</button>
    <button id="refresh" disabled>Request Snapshot</button>
</div>

<table>
    <thead>
    <tr>
        <th>Symbol</th>
        <th>Price</th>
        <th>Change</th>
        <th>Updated</th>
    </tr>
    </thead>
    <tbody id="prices">
    </tbody>
</table>

<section id="log"></section>

<script type="module">
    import { Client } from 'https://cdn.jsdelivr.net/npm/@stomp/stompjs@7.0.0/esm6/index.min.js';

    const connectBtn = document.getElementById('connect');
    const disconnectBtn = document.getElementById('disconnect');
    const refreshBtn = document.getElementById('refresh');
    const status = document.getElementById('status');
    const pricesTbody = document.getElementById('prices');
    const logSection = document.getElementById('log');

    const formatCurrency = value => Number(value).toFixed(2);
    const formatChange = change => (change >= 0 ? `+${Number(change).toFixed(2)}` : Number(change).toFixed(2));

    let stompClient;
    const lastPrices = new Map();

    function appendLog(message) {
        const time = new Date().toLocaleTimeString();
        logSection.textContent = `[${time}] ${message}\n` + logSection.textContent;
    }

    function renderTable() {
        const rows = Array.from(lastPrices.values()).sort((a, b) => a.symbol.localeCompare(b.symbol));
        pricesTbody.innerHTML = rows.map(row => {
            const trend = row.change > 0 ? 'up' : row.change < 0 ? 'down' : '';
            return `<tr class="${trend}">
                <td>${row.symbol}</td>
                <td>$${formatCurrency(row.price)}</td>
                <td class="change">${formatChange(row.change)}</td>
                <td>${new Date(row.lastUpdated).toLocaleTimeString()}</td>
            </tr>`;
        }).join('');
    }

    function upsertPrice(payload) {
        if (Array.isArray(payload)) {
            payload.forEach(upsertPrice);
            return;
        }
        const row = {
            symbol: payload.symbol,
            price: payload.price,
            change: payload.change,
            lastUpdated: payload.lastUpdated
        };
        lastPrices.set(row.symbol, row);
        renderTable();
    }

    connectBtn.addEventListener('click', () => {
        if (stompClient && stompClient.connected) {
            appendLog('Already connected.');
            return;
        }

        stompClient = new Client({
            brokerURL: `ws://${window.location.host}/ws`,
            reconnectDelay: 4000,
            heartbeatIncoming: 10000,
            heartbeatOutgoing: 10000,
            debug: str => console.debug(str)
        });

        stompClient.onConnect = () => {
            status.textContent = 'Connected';
            status.style.color = '#1f9d55';
            disconnectBtn.disabled = false;
            refreshBtn.disabled = false;
            connectBtn.disabled = true;
            appendLog('Subscribed to /topic/prices');
            stompClient.subscribe('/topic/prices', message => {
                const body = JSON.parse(message.body);
                upsertPrice(body);
            });
        };

        stompClient.onStompError = frame => {
            appendLog(`Broker reported error: ${frame.headers['message']}`);
        };

        stompClient.onWebSocketClose = () => {
            status.textContent = 'Disconnected';
            status.style.color = '#e3342f';
            connectBtn.disabled = false;
            disconnectBtn.disabled = true;
            refreshBtn.disabled = true;
        };

        stompClient.activate();
    });

    disconnectBtn.addEventListener('click', () => {
        if (!stompClient) {
            return;
        }
        stompClient.deactivate();
        appendLog('Manual disconnect triggered.');
    });

    refreshBtn.addEventListener('click', () => {
        if (!stompClient || !stompClient.connected) {
            appendLog('Connect first.');
            return;
        }
        stompClient.publish({
            destination: '/app/request-prices',
            body: JSON.stringify({ reason: 'manual-refresh' })
        });
        appendLog('Requested price snapshot.');
    });
</script>
</body>
</html>
```

The UI now displays a price table and includes a “Request Snapshot” button that triggers the `@MessageMapping` method we created. The `upsertPrice` function gracefully handles both single `StockPrice` objects and arrays returned by the manual snapshot. We also introduced heartbeats to keep the connection healthy.

## Running and Verifying the Live Updates

Start the application:

```bash
./mvnw spring-boot:run
```

Open `http://localhost:8080/index.html` in a browser. When you click “Connect,” the status indicator turns green and the table gradually fills with the first batch of prices as the scheduled job fires. Leave the tab open and you will see updates streaming every two seconds, with the `Change` column flashing red or green depending on the delta. Clicking “Request Snapshot” triggers an immediate burst of updates.

To validate that the messages are truly server-driven, open the browser’s developer tools, select the Network tab, and inspect the `/ws` WebSocket connection. You will observe `MESSAGE` frames arriving at fixed intervals even when you are not interacting with the page. Because the updates originate inside the application, there are no corresponding outbound `SEND` frames.

For additional validation, start multiple browser windows or connect from different devices on the same network. Every subscribed client receives the same stream simultaneously, demonstrating the publish-subscribe model powered by `enableSimpleBroker`.

## Understanding Topic Semantics and Delivery Guarantees

It is worth reflecting on what Spring’s simple broker provides in this scenario:

- Messages are broadcast in-memory to all connected clients; there is no persistence or replay. If a client disconnects, it misses any updates sent during that downtime.
- Delivery is best-effort. If a client is slow to process messages, the simple broker may eventually drop frames to avoid exhausting buffers. In later articles we will configure limits and, eventually, connect to an external broker for stronger guarantees.
- The simple broker does not support durable subscriptions. We will tackle more advanced messaging patterns when we reach the enterprise scaling article.

Despite these limitations, the simple broker is ideal for toy projects and tutorials because it eliminates infrastructure dependencies. You can focus on application logic and graduate to broker relays when you need more robustness.

## Troubleshooting the Live Ticker

Here are common hiccups and how to resolve them:

- **`@Scheduled` methods never fire:** Ensure you added `@EnableScheduling` to your application class. Also verify that the method is declared within a Spring-managed bean (annotated with `@Component` or similar).
- **Clients see duplicate rows:** The HTML client’s `upsertPrice` method must key prices by symbol. If you accidentally created new DOM rows without replacing existing ones, refine the logic to update entries by symbol.
- **JSON serialization issues:** Spring Boot 3 uses Jackson 2.15+. Ensure your payload classes (records) have getters corresponding to the expected JSON fields. If you rely on Java 17 records, Jackson supports them out of the box.
- **Scheduling too aggressive:** If you reduce the `fixedRate` to a few milliseconds, your CPU usage will spike. We will discuss performance tuning in Article 9, but for now keep the interval reasonable and monitor the console logs to confirm that the publisher keeps up.

## Where We Go from Here

We have transformed our application from a purely reactive echo service into an active broadcaster that pushes changing state to subscribers. This pattern underpins many real-world use cases, from operational dashboards to multiplayer game state replication.

In the next article we will build on this foundation to implement a classic two-way chat room, reinforcing your understanding of STOMP frame flow and preparing us for features like SockJS fallbacks and user-specific messaging. Before moving forward, make sure you can:

- Start the Spring Boot application and watch live price updates populate the HTML table.
- Trigger manual snapshots and verify they broadcast to every connected client.
- Inspect WebSocket frames to confirm that `convertAndSend` emits messages without inbound triggers.

Take a moment to experiment. Try adding more symbols, randomizing update intervals, or styling the UI to highlight significant positive or negative swings. Each small enhancement deepens your familiarity with Spring’s messaging tools—and that experience will pay off as the scenarios grow more sophisticated.

