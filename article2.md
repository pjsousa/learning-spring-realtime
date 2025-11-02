# Going Live: Implementing Real-Time Server Updates using @SendTo and Scheduling

## Introduction

In our previous article, we established the foundation for STOMP over WebSocket communication, demonstrating how clients can send messages to the server and receive responses. However, real-time applications often need the server to push updates to clients proactively, without waiting for client requests.

In this article, we'll explore server-initiated messaging—a crucial pattern for applications like stock tickers, live dashboards, sensor monitoring, and notification systems. We'll learn how Spring Boot enables any component in your application to broadcast messages to subscribed clients using Spring's messaging infrastructure.

By the end of this article, you'll have implemented a live data feed system that continuously publishes updates from the server to all connected clients. We'll build a simulated stock price ticker that demonstrates real-time server-to-client communication using Spring's `@Scheduled` annotation and the `SimpMessagingTemplate`.

## Understanding Server-to-Client Broadcasting

Before diving into implementation, let's understand the key concepts:

**Publish-Subscribe Pattern**: This is the messaging pattern we're using. The server publishes messages to a topic (like `/topic/prices`), and all clients subscribed to that topic receive the messages automatically. This is ideal for broadcasting information that multiple clients need simultaneously.

**SimpMessagingTemplate**: Spring's messaging template provides a programmatic way to send messages to STOMP destinations. Unlike `@SendTo` (which works with method return values), `SimpMessagingTemplate` allows you to send messages from anywhere in your application—services, schedulers, event handlers, etc.

**@Scheduled**: Spring's scheduling annotation enables periodic execution of methods. We'll use this to simulate a continuous data stream that publishes updates at regular intervals.

**Broker Channel**: When messages are sent to topics, they flow through Spring's internal `brokerChannel`, which routes them to all subscribed clients via the `clientOutboundChannel`.

## Project Setup

We'll build upon the project from Article 1. If you're starting fresh, ensure you have:
- Spring Boot 3.1.x or later
- `spring-boot-starter-websocket` dependency
- The `WebSocketConfig` class from Article 1

We'll also add scheduling support to our project.

### Step 1: Enable Scheduling

Update your main application class to enable scheduling:

```java
package com.example.stomptoyproject;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.scheduling.annotation.EnableScheduling;

@SpringBootApplication
@EnableScheduling
public class StompToyProjectApplication {

    public static void main(String[] args) {
        SpringApplication.run(StompToyProjectApplication.class, args);
    }
}
```

The `@EnableScheduling` annotation activates Spring's scheduled task execution capability, allowing us to use `@Scheduled` annotations in our components.

## Creating a Data Model

Let's create a simple model to represent our stock price data.

### Step 2: Create the StockPrice Model

Create a new class `StockPrice.java`:

```java
package com.example.stomptoyproject;

import java.math.BigDecimal;
import java.math.RoundingMode;
import java.time.LocalDateTime;

public class StockPrice {
    private String symbol;
    private BigDecimal price;
    private BigDecimal change;
    private LocalDateTime timestamp;

    public StockPrice() {
    }

    public StockPrice(String symbol, BigDecimal price, BigDecimal change) {
        this.symbol = symbol;
        this.price = price;
        this.change = change;
        this.timestamp = LocalDateTime.now();
    }

    // Getters and setters
    public String getSymbol() {
        return symbol;
    }

    public void setSymbol(String symbol) {
        this.symbol = symbol;
    }

    public BigDecimal getPrice() {
        return price;
    }

    public void setPrice(BigDecimal price) {
        this.price = price;
    }

    public BigDecimal getChange() {
        return change;
    }

    public void setChange(BigDecimal change) {
        this.change = change;
    }

    public LocalDateTime getTimestamp() {
        return timestamp;
    }

    public void setTimestamp(LocalDateTime timestamp) {
        this.timestamp = timestamp;
    }

    @Override
    public String toString() {
        return String.format("StockPrice{symbol='%s', price=%.2f, change=%.2f}", 
            symbol, price.doubleValue(), change.doubleValue());
    }
}
```

For simplicity, Spring will automatically serialize this to JSON when sending it to clients. In production, you might want to use `@JsonFormat` annotations for better control over serialization.

## Implementing the Price Service

Now let's create a service that generates simulated stock prices and uses `SimpMessagingTemplate` to broadcast them.

### Step 3: Create the Stock Price Service

Create a new class `StockPriceService.java`:

```java
package com.example.stomptoyproject;

import org.springframework.messaging.simp.SimpMessagingTemplate;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Service;

import java.math.BigDecimal;
import java.math.RoundingMode;
import java.util.HashMap;
import java.util.Map;
import java.util.Random;

@Service
public class StockPriceService {

    private final SimpMessagingTemplate messagingTemplate;
    private final Map<String, BigDecimal> currentPrices;
    private final Random random;

    public StockPriceService(SimpMessagingTemplate messagingTemplate) {
        this.messagingTemplate = messagingTemplate;
        this.currentPrices = new HashMap<>();
        this.random = new Random();
        
        // Initialize with some stock symbols
        currentPrices.put("AAPL", new BigDecimal("150.00"));
        currentPrices.put("GOOGL", new BigDecimal("2800.00"));
        currentPrices.put("MSFT", new BigDecimal("380.00"));
        currentPrices.put("AMZN", new BigDecimal("3200.00"));
        currentPrices.put("TSLA", new BigDecimal("250.00"));
    }

    @Scheduled(fixedRate = 2000) // Execute every 2 seconds
    public void publishPriceUpdates() {
        // Update prices for all stocks
        for (Map.Entry<String, BigDecimal> entry : currentPrices.entrySet()) {
            String symbol = entry.getKey();
            BigDecimal currentPrice = entry.getValue();
            
            // Generate a random price change (-2% to +2%)
            double changePercent = (random.nextDouble() - 0.5) * 4.0; // -2% to +2%
            BigDecimal change = currentPrice.multiply(
                BigDecimal.valueOf(changePercent / 100.0)
            ).setScale(2, RoundingMode.HALF_UP);
            
            BigDecimal newPrice = currentPrice.add(change).setScale(2, RoundingMode.HALF_UP);
            
            // Update the stored price
            currentPrices.put(symbol, newPrice);
            
            // Create and broadcast the price update
            StockPrice stockPrice = new StockPrice(symbol, newPrice, change);
            
            // Send to all clients subscribed to /topic/prices
            messagingTemplate.convertAndSend("/topic/prices", stockPrice);
        }
    }
    
    // Method to get all current prices (useful for initial data loading)
    public Map<String, BigDecimal> getAllCurrentPrices() {
        return new HashMap<>(currentPrices);
    }
}
```

Let's break down the key components:

**@Service**: Marks this class as a Spring service component, making it eligible for dependency injection.

**SimpMessagingTemplate Injection**: Spring automatically provides a configured `SimpMessagingTemplate` bean. This template is the gateway for sending messages to STOMP destinations from anywhere in your application.

**@Scheduled(fixedRate = 2000)**: This annotation schedules the method to run every 2000 milliseconds (2 seconds). Other scheduling options include:
- `fixedDelay = 2000`: Wait 2 seconds after the previous execution completes
- `cron = "0 * * * * ?"`: Use a cron expression for more complex scheduling

**convertAndSend()**: This method converts the Java object to JSON (using Jackson by default) and sends it to the specified destination (`/topic/prices`). All clients subscribed to this topic will receive the message.

## Understanding Message Broadcasting

When `messagingTemplate.convertAndSend("/topic/prices", stockPrice)` is called:

1. Spring converts the `StockPrice` object to JSON using Jackson
2. The message is sent to the `brokerChannel`
3. The in-memory broker routes it to all clients subscribed to `/topic/prices`
4. Each subscribed client receives the message via their WebSocket connection

This happens asynchronously, so the service method doesn't block waiting for clients to receive the message. Spring handles the message routing and delivery automatically.

## Creating an Alternative Approach with @SendTo

While `SimpMessagingTemplate` gives us flexibility to send messages from anywhere, we can also use `@SendTo` in a scheduled method. Let's explore both approaches:

### Step 4: Alternative Controller-Based Approach

You could also create a scheduled method in a controller:

```java
package com.example.stomptoyproject;

import org.springframework.messaging.handler.annotation.SendTo;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Controller;

import java.math.BigDecimal;
import java.time.LocalDateTime;

@Controller
public class ScheduledBroadcastController {

    private int counter = 0;

    @Scheduled(fixedRate = 3000) // Every 3 seconds
    @SendTo("/topic/announcements")
    public String broadcastAnnouncement() {
        counter++;
        return String.format("Server announcement #%d at %s", 
            counter, LocalDateTime.now());
    }
}
```

However, note that `@SendTo` with `@Scheduled` requires the method to return a value, and the scheduling infrastructure needs to be aware of the messaging context. For more complex scenarios, `SimpMessagingTemplate` is generally preferred.

## Configuring the WebSocket (Review)

Make sure your `WebSocketConfig` is set up correctly (from Article 1):

```java
@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {

    @Override
    public void configureMessageBroker(MessageBrokerRegistry config) {
        config.enableSimpleBroker("/topic");
        config.setApplicationDestinationPrefixes("/app");
    }

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/gs-guide-websocket")
                .setAllowedOriginPatterns("*")
                .withSockJS();
    }
}
```

This configuration enables:
- Topics prefixed with `/topic` for broadcasting
- Application destinations prefixed with `/app` for client-to-server messages

## Creating the HTML Client

Now let's create an HTML client that subscribes to the price updates and displays them in real-time.

### Step 5: Create the Stock Ticker Client

Create `stock-ticker.html` in `src/main/resources/static/`:

```html
<!DOCTYPE html>
<html>
<head>
    <title>Live Stock Price Ticker</title>
    <script src="https://cdn.jsdelivr.net/npm/sockjs-client@1/dist/sockjs.min.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/@stomp/stompjs@7/bundles/stomp.umd.min.js"></script>
    <style>
        body {
            font-family: Arial, sans-serif;
            max-width: 1000px;
            margin: 20px auto;
            padding: 20px;
            background-color: #f5f5f5;
        }
        .container {
            background: white;
            padding: 30px;
            border-radius: 8px;
            box-shadow: 0 2px 4px rgba(0,0,0,0.1);
        }
        h1 {
            color: #333;
            text-align: center;
        }
        .status {
            padding: 10px;
            margin: 10px 0;
            border-radius: 4px;
            text-align: center;
            font-weight: bold;
        }
        .status.connected {
            background-color: #d4edda;
            color: #155724;
        }
        .status.disconnected {
            background-color: #f8d7da;
            color: #721c24;
        }
        button {
            background-color: #007bff;
            color: white;
            padding: 10px 20px;
            border: none;
            border-radius: 4px;
            cursor: pointer;
            margin: 5px;
            font-size: 16px;
        }
        button:hover {
            background-color: #0056b3;
        }
        button:disabled {
            background-color: #cccccc;
            cursor: not-allowed;
        }
        #stockTable {
            width: 100%;
            border-collapse: collapse;
            margin-top: 20px;
        }
        #stockTable th {
            background-color: #007bff;
            color: white;
            padding: 12px;
            text-align: left;
        }
        #stockTable td {
            padding: 10px;
            border-bottom: 1px solid #ddd;
        }
        #stockTable tr:hover {
            background-color: #f5f5f5;
        }
        .price {
            font-weight: bold;
            font-family: 'Courier New', monospace;
        }
        .positive {
            color: #28a745;
        }
        .negative {
            color: #dc3545;
        }
        .update-indicator {
            display: inline-block;
            width: 8px;
            height: 8px;
            border-radius: 50%;
            background-color: #ffc107;
            margin-left: 5px;
            animation: pulse 1s infinite;
        }
        @keyframes pulse {
            0%, 100% { opacity: 1; }
            50% { opacity: 0.5; }
        }
    </style>
</head>
<body>
    <div class="container">
        <h1>📈 Live Stock Price Ticker</h1>
        <div id="status" class="status disconnected">Disconnected</div>
        <div style="text-align: center; margin: 20px 0;">
            <button id="connectBtn" onclick="connect()">Connect</button>
            <button id="disconnectBtn" onclick="disconnect()" disabled>Disconnect</button>
        </div>
        <table id="stockTable">
            <thead>
                <tr>
                    <th>Symbol</th>
                    <th>Price</th>
                    <th>Change</th>
                    <th>Last Update</th>
                </tr>
            </thead>
            <tbody id="stockBody">
            </tbody>
        </table>
    </div>

    <script>
        let stompClient = null;
        let stockData = {};

        function setConnected(value) {
            const statusDiv = document.getElementById('status');
            statusDiv.className = value ? 'status connected' : 'status disconnected';
            statusDiv.textContent = value ? '🟢 Connected - Receiving live updates' : '🔴 Disconnected';
            document.getElementById('connectBtn').disabled = value;
            document.getElementById('disconnectBtn').disabled = !value;
        }

        function connect() {
            const socket = new SockJS('/gs-guide-websocket');
            stompClient = StompJs.Stomp.over(socket);
            
            stompClient.debug = function(str) {
                console.log(str);
            };

            stompClient.connect({}, function(frame) {
                setConnected(true);
                console.log('Connected: ' + frame);

                // Subscribe to the price updates topic
                stompClient.subscribe('/topic/prices', function(message) {
                    const stockPrice = JSON.parse(message.body);
                    updateStockPrice(stockPrice);
                });
            }, function(error) {
                console.error('Connection error:', error);
                setConnected(false);
            });
        }

        function disconnect() {
            if (stompClient !== null) {
                stompClient.disconnect();
            }
            setConnected(false);
            stockData = {};
            document.getElementById('stockBody').innerHTML = '';
        }

        function updateStockPrice(stockPrice) {
            // Store the stock data
            stockData[stockPrice.symbol] = stockPrice;

            // Find or create the table row
            let row = document.getElementById('row-' + stockPrice.symbol);
            
            if (!row) {
                // Create new row
                const tbody = document.getElementById('stockBody');
                row = document.createElement('tr');
                row.id = 'row-' + stockPrice.symbol;
                tbody.appendChild(row);
            }

            // Format the change value
            const changeValue = parseFloat(stockPrice.change).toFixed(2);
            const changeClass = changeValue >= 0 ? 'positive' : 'negative';
            const changeSign = changeValue >= 0 ? '+' : '';

            // Format timestamp
            const timestamp = new Date().toLocaleTimeString();

            // Update row content
            row.innerHTML = `
                <td><strong>${stockPrice.symbol}</strong></td>
                <td class="price">$${parseFloat(stockPrice.price).toFixed(2)}</td>
                <td class="${changeClass}">${changeSign}${changeValue}</td>
                <td>${timestamp}<span class="update-indicator"></span></td>
            `;

            // Add highlight animation
            row.style.backgroundColor = '#fff3cd';
            setTimeout(() => {
                row.style.backgroundColor = '';
            }, 500);
        }

        // Auto-connect on page load (optional)
        // window.addEventListener('load', connect);
    </script>
</body>
</html>
```

## Understanding the Client Implementation

The client demonstrates several important concepts:

**Automatic Updates**: Once connected and subscribed, the client receives updates automatically without sending any requests. This is the essence of server-pushed real-time updates.

**JSON Parsing**: STOMP messages contain JSON payloads. We parse them using `JSON.parse(message.body)` to get JavaScript objects.

**Dynamic UI Updates**: The client maintains a map of stock symbols and updates the table dynamically as new prices arrive.

**Visual Feedback**: The update indicator and highlight animation provide visual feedback when prices change, making it clear that real-time updates are occurring.

## Advanced: Filtering and Selective Broadcasting

Sometimes you might want to send different updates to different groups of clients. Here's an example:

```java
@Service
public class SelectiveBroadcastService {
    
    private final SimpMessagingTemplate messagingTemplate;
    
    public SelectiveBroadcastService(SimpMessagingTemplate messagingTemplate) {
        this.messagingTemplate = messagingTemplate;
    }
    
    @Scheduled(fixedRate = 1000)
    public void broadcastToMultipleTopics() {
        // Send to tech stocks topic
        messagingTemplate.convertAndSend("/topic/stocks/tech", 
            new StockPrice("AAPL", new BigDecimal("150.00"), new BigDecimal("1.50")));
        
        // Send to energy stocks topic
        messagingTemplate.convertAndSend("/topic/stocks/energy", 
            new StockPrice("XOM", new BigDecimal("80.00"), new BigDecimal("-0.50")));
        
        // Send to all stocks topic
        messagingTemplate.convertAndSend("/topic/stocks/all", 
            new StockPrice("MSFT", new BigDecimal("380.00"), new BigDecimal("2.00")));
    }
}
```

Clients can subscribe to specific topics based on their needs, reducing unnecessary data transfer.

## Testing the Implementation

1. Start your Spring Boot application.
2. Open `http://localhost:8080/stock-ticker.html` in your browser.
3. Click "Connect" to establish the WebSocket connection.
4. Watch as stock prices update automatically every 2 seconds!
5. Open the same page in multiple browser tabs to see that all clients receive the same updates simultaneously.

## Performance Considerations

**Message Frequency**: Be mindful of how often you broadcast updates. Too frequent updates can:
- Overwhelm clients with processing
- Consume excessive bandwidth
- Put unnecessary load on the message broker

**Message Size**: Keep message payloads small. If you need to send large amounts of data, consider sending only deltas (changes) rather than full state.

**Client Buffering**: STOMP.js and browsers handle message queuing, but if clients process messages slowly, they might fall behind. Consider implementing message acknowledgment or selective subscriptions.

## Complete Test Client

Here's a minimal test client you can use to verify the service:

```html
<!DOCTYPE html>
<html>
<head>
    <title>Stock Ticker Test</title>
    <script src="https://cdn.jsdelivr.net/npm/sockjs-client@1/dist/sockjs.min.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/@stomp/stompjs@7/bundles/stomp.umd.min.js"></script>
</head>
<body>
    <h1>Stock Price Updates</h1>
    <button onclick="connect()">Connect</button>
    <button onclick="disconnect()">Disconnect</button>
    <div id="output"></div>

    <script>
        let client = null;
        
        function connect() {
            const socket = new SockJS('http://localhost:8080/gs-guide-websocket');
            client = StompJs.Stomp.over(socket);
            client.connect({}, () => {
                log('Connected!');
                client.subscribe('/topic/prices', (msg) => {
                    const stock = JSON.parse(msg.body);
                    log(`${stock.symbol}: $${stock.price} (${stock.change >= 0 ? '+' : ''}${stock.change})`);
                });
            });
        }
        
        function disconnect() {
            if (client) client.disconnect();
            log('Disconnected');
        }
        
        function log(msg) {
            const output = document.getElementById('output');
            output.innerHTML += '<p>' + new Date().toLocaleTimeString() + ' - ' + msg + '</p>';
            output.scrollTop = output.scrollHeight;
        }
    </script>
</body>
</html>
```

## What We've Accomplished

In this article, we've:
1. Enabled Spring scheduling to run periodic tasks
2. Created a service that generates and broadcasts stock price updates
3. Used `SimpMessagingTemplate` to send messages programmatically
4. Built a real-time stock ticker client that displays live updates
5. Demonstrated the publish-subscribe pattern in action

## Next Steps

You now understand how to push server-generated updates to clients in real-time! This pattern is fundamental for dashboards, monitoring systems, and live data feeds.

In our next article, we'll explore two-way communication more deeply by building a classic chat application where clients can send messages that are broadcast to all other connected clients. This will demonstrate how to combine client-to-server messaging (from Article 1) with server-to-client broadcasting (from this article) to create interactive, collaborative applications.
