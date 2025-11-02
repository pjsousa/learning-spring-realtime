# Inside the Flow: Performance Tuning and Monitoring Spring's Message Channels

*Estimated reading time: 25 minutes · Target word count: ~1,800*

## Overview

By now you've built chat rooms, tickers, and collaborative cursors—all riding on Spring's STOMP infrastructure. But what happens when you scale from ten users to ten thousand? How do you diagnose why messages seem sluggish under load? This article lifts the hood on Spring's internal messaging architecture to reveal the channels, thread pools, and buffers that determine your application's performance characteristics.

We'll explore:

- The anatomy of `clientInboundChannel` and `clientOutboundChannel`, the asynchronous pipelines that process every STOMP frame.
- Thread pool configuration strategies for CPU-bound vs. I/O-bound workloads.
- Queue sizing to handle traffic spikes without dropping messages or exhausting memory.
- WebSocket transport tuning (`sendTimeLimit`, buffer sizes) to prevent slow clients from bottlenecking the entire system.
- Runtime observability using `WebSocketMessageBrokerStats` to identify bottlenecks and debug production issues.

By the end, you'll have a comprehensive dashboard that visualizes channel metrics, connection counts, and message throughput. You'll understand how to tune Spring's messaging infrastructure for your specific use case, whether that's high-frequency trading data, collaborative editing, or IoT sensor streams.

## Learning Goals

After completing this article, you will be able to:

- Configure custom thread pools for `clientInboundChannel` and `clientOutboundChannel` based on workload characteristics.
- Adjust queue capacities to balance memory usage and message buffering during peak loads.
- Set WebSocket transport timeouts and buffer limits to protect against slow or misbehaving clients.
- Expose `WebSocketMessageBrokerStats` via REST endpoints or actuator to monitor system health in production.
- Diagnose performance issues by analyzing channel saturation, queue depths, and message processing times.

## Why Message Channel Performance Matters

Spring's WebSocket messaging operates on an asynchronous, channel-based architecture. Every STOMP frame—whether inbound (SEND, SUBSCRIBE) or outbound (MESSAGE)—flows through dedicated channels backed by thread pools and queues. Understanding this pipeline is crucial because:

1. **Bottlenecks hide in channels**: A saturated `clientInboundChannel` can delay all incoming messages, even if your controller logic is fast.
2. **Memory pressure from queues**: Unbounded queues risk out-of-memory errors during traffic spikes. Bounded queues may drop messages if not sized correctly.
3. **Thread starvation**: Default thread pools may be too small for high-frequency applications, causing contention and latency.
4. **Slow client impact**: A single slow client can block outbound message delivery to all subscribers if transport limits aren't configured.

This article provides the tools to measure, tune, and monitor these moving parts.

## Prerequisites and Project Setup

We'll extend a minimal STOMP application to demonstrate performance characteristics. If you're following the series, you can build on any previous article's codebase. For maximum clarity, we'll start fresh with a focused example.

### Generate the Project

Use Spring Initializr to create a new project:

```bash
curl https://start.spring.io/starter.zip \
  -d dependencies=websocket,web,actuator \
  -d name=channel-performance \
  -d javaVersion=21 \
  -d type=maven-project \
  -d packageName=com.example.channelperformance \
  -o channel-performance.zip

unzip channel-performance.zip
cd channel-performance
```

The `actuator` dependency enables health checks and custom metrics endpoints we'll add later.

### Verify Dependencies

Your `pom.xml` should include:

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
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
</dependencies>
```

For Gradle:

```kotlin
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-websocket")
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-actuator")
}
```

### Basic Configuration

Create the standard WebSocket configuration:

**`src/main/java/com/example/channelperformance/config/WebSocketConfig.java`**

```java
package com.example.channelperformance.config;

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
        registry.setApplicationDestinationPrefixes("/app");
        registry.enableSimpleBroker("/topic");
    }

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/ws")
                .setAllowedOriginPatterns("*")
                .withSockJS();
    }
}
```

We'll enhance this configuration progressively as we introduce tuning options.

## Understanding the Message Channel Architecture

Spring's STOMP support relies on four core channels:

| Channel | Purpose | Direction |
|---------|---------|-----------|
| `clientInboundChannel` | Delivers inbound STOMP frames (CONNECT, SEND, SUBSCRIBE, DISCONNECT) from clients to application handlers. | Client → Application |
| `clientOutboundChannel` | Delivers outbound STOMP frames (MESSAGE, CONNECTED, ERROR) from the broker to clients. | Application → Client |
| `brokerChannel` | Internal channel that carries messages from application code (via `@SendTo` or `SimpMessagingTemplate`) to the broker for distribution. | Application → Broker |
| `brokerSubscriptionsChannel` | Manages subscription state (subscribe/unsubscribe events). Rarely tuned directly. | Internal |

By default, Spring configures these channels with:

- **Thread pools**: Executor-based channels with core pool sizes of 1 (inbound) and 1 (outbound), scaling up as needed.
- **Queues**: Unbounded `LinkedBlockingQueue` implementations that grow until memory is exhausted.

This works fine for low-traffic scenarios, but breaks down under load. Let's customize these settings.

## Step 1: Configuring Thread Pools for clientInboundChannel

The `clientInboundChannel` handles message parsing, routing to controllers, and validation. Its thread pool size depends on whether your handlers are CPU-bound (heavy computation) or I/O-bound (database calls, external APIs).

### CPU-Bound Workload Configuration

If your `@MessageMapping` methods perform CPU-intensive work (encryption, image processing, complex calculations), you want thread pools sized close to your CPU core count:

```java
package com.example.channelperformance.config;

import org.springframework.context.annotation.Configuration;
import org.springframework.messaging.simp.config.ChannelRegistration;
import org.springframework.messaging.simp.config.MessageBrokerRegistry;
import org.springframework.web.socket.config.annotation.EnableWebSocketMessageBroker;
import org.springframework.web.socket.config.annotation.StompEndpointRegistry;
import org.springframework.web.socket.config.annotation.WebSocketMessageBrokerConfigurer;

import java.util.concurrent.Executors;
import java.util.concurrent.ThreadPoolExecutor;

@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {

    @Override
    public void configureMessageBroker(MessageBrokerRegistry registry) {
        registry.setApplicationDestinationPrefixes("/app");
        registry.enableSimpleBroker("/topic");
    }

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/ws")
                .setAllowedOriginPatterns("*")
                .withSockJS();
    }

    @Override
    public void configureClientInboundChannel(ChannelRegistration registration) {
        // For CPU-bound workloads: match thread count to CPU cores
        int corePoolSize = Runtime.getRuntime().availableProcessors();
        int maxPoolSize = corePoolSize * 2;
        
        ThreadPoolExecutor executor = new ThreadPoolExecutor(
            corePoolSize,
            maxPoolSize,
            60L,
            java.util.concurrent.TimeUnit.SECONDS,
            new java.util.concurrent.LinkedBlockingQueue<>(1000), // Bounded queue
            new java.util.concurrent.ThreadFactory() {
                private int counter = 0;
                @Override
                public Thread newThread(Runnable r) {
                    Thread t = new Thread(r, "clientInbound-" + (counter++));
                    t.setDaemon(false);
                    return t;
                }
            },
            new ThreadPoolExecutor.CallerRunsPolicy() // Block sender if queue is full
        );
        
        registration.taskExecutor(executor);
    }
}
```

Key decisions:

- **Core pool size = CPU cores**: Maximizes CPU utilization for parallel processing.
- **Max pool size = cores × 2**: Allows temporary bursts while preventing over-subscription.
- **Bounded queue (1000)**: Prevents unbounded memory growth; adjust based on expected message rate.
- **CallerRunsPolicy**: When the queue is full, the calling thread (the WebSocket I/O thread) handles the message directly, backpressuring the client connection.

### I/O-Bound Workload Configuration

If handlers perform database queries, HTTP calls, or other blocking I/O, you want larger thread pools to keep CPUs busy while threads wait for I/O:

```java
@Override
public void configureClientInboundChannel(ChannelRegistration registration) {
    // For I/O-bound workloads: larger thread pools to hide I/O latency
    ThreadPoolExecutor executor = new ThreadPoolExecutor(
        10,                    // Higher core pool size
        50,                    // Much higher max for I/O wait
        60L,
        java.util.concurrent.TimeUnit.SECONDS,
        new java.util.concurrent.LinkedBlockingQueue<>(5000),
        new java.util.concurrent.ThreadFactory() {
            private int counter = 0;
            @Override
            public Thread newThread(Runnable r) {
                Thread t = new Thread(r, "clientInbound-io-" + (counter++));
                t.setDaemon(false);
                return t;
            }
        },
        new ThreadPoolExecutor.CallerRunsPolicy()
    );
    
    registration.taskExecutor(executor);
}
```

The larger thread count compensates for threads blocked waiting on database connections, REST API responses, or file system operations.

## Step 2: Tuning clientOutboundChannel

The `clientOutboundChannel` delivers messages from the broker to clients. This channel is typically I/O-bound because it writes to WebSocket connections. However, it can also become CPU-bound if you're transforming large payloads or applying compression.

```java
@Override
public void configureClientOutboundChannel(ChannelRegistration registration) {
    ThreadPoolExecutor executor = new ThreadPoolExecutor(
        10,                    // Sufficient for I/O-bound writes
        20,                    // Allow bursts during broadcast storms
        60L,
        java.util.concurrent.TimeUnit.SECONDS,
        new java.util.concurrent.LinkedBlockingQueue<>(2000),
        new java.util.concurrent.ThreadFactory() {
            private int counter = 0;
            @Override
            public Thread newThread(Runnable r) {
                Thread t = new Thread(r, "clientOutbound-" + (counter++));
                t.setDaemon(false);
                return t;
            }
        },
        new ThreadPoolExecutor.CallerRunsPolicy()
    );
    
    registration.taskExecutor(executor);
}
```

**Critical consideration**: If `clientOutboundChannel` queues fill up, messages accumulate in memory for all subscribed clients. During a broadcast to 1000 clients, a full queue means 1000 copies of the message waiting to be sent. Keep outbound queues smaller or use backpressure mechanisms to limit broadcaster throughput.

## Step 3: Setting Queue Capacities

Queue sizing is a balancing act:

- **Too small**: Messages are rejected or threads block unnecessarily during short spikes.
- **Too large**: Memory usage grows unbounded; messages experience long delays before processing.

A reasonable starting point:

```java
// Inbound: 1000-5000 messages
new java.util.concurrent.LinkedBlockingQueue<>(1000)

// Outbound: 500-2000 messages (multiply by expected subscriber count)
new java.util.concurrent.LinkedBlockingQueue<>(2000)
```

Monitor queue depths in production and adjust based on:

- Average message processing time
- Peak traffic patterns
- Available heap memory
- Acceptable latency requirements

We'll add monitoring later to observe actual queue behavior.

## Step 4: Configuring WebSocket Transport Properties

WebSocket connections can suffer from slow clients (mobile networks, CPU-bound browsers) that can't keep up with message delivery. Spring allows you to protect the server from these scenarios.

### sendTimeLimit

The maximum time (in milliseconds) allowed to send a message to a client before timing out. Default is `10000` (10 seconds). If a client is too slow, the connection may be closed.

```java
@Override
public void configureWebSocketTransport(WebSocketTransportRegistration registration) {
    registration.setSendTimeLimit(5000)      // 5 seconds max send time
                 .setSendBufferSizeLimit(512 * 1024)  // 512 KB buffer per connection
                 .setMessageSizeLimit(128 * 1024);    // 128 KB max message size
}
```

### sendBufferSizeLimit

The maximum size (in bytes) of the outbound buffer per WebSocket connection. When the buffer fills (because the client isn't reading fast enough), sending blocks until space becomes available. Default is `512 * 1024` (512 KB).

### messageSizeLimit

The maximum allowed size for an inbound STOMP message. Messages exceeding this limit are rejected. Default is `64 * 1024` (64 KB).

Add this method to `WebSocketConfig`:

```java
@Override
public void configureWebSocketTransport(WebSocketTransportRegistration registration) {
    registration.setSendTimeLimit(5000)
                 .setSendBufferSizeLimit(512 * 1024)
                 .setMessageSizeLimit(128 * 1024);
}
```

These limits prevent a single slow or malicious client from consuming excessive server resources.

## Step 5: Creating a Test Application

To observe channel performance, we need a workload generator. Let's build a simple message broadcaster that sends periodic updates:

**`src/main/java/com/example/channelperformance/service/PerformanceTestService.java`**

```java
package com.example.channelperformance.service;

import org.springframework.messaging.simp.SimpMessagingTemplate;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Service;

import java.time.Instant;

@Service
public class PerformanceTestService {

    private final SimpMessagingTemplate messagingTemplate;
    private long messageCounter = 0;

    public PerformanceTestService(SimpMessagingTemplate messagingTemplate) {
        this.messagingTemplate = messagingTemplate;
    }

    @Scheduled(fixedRate = 100) // 10 messages per second
    public void broadcastTestMessage() {
        messageCounter++;
        TestMessage message = new TestMessage(
            messageCounter,
            "Test message #" + messageCounter,
            Instant.now()
        );
        messagingTemplate.convertAndSend("/topic/test", message);
    }

    public static class TestMessage {
        private long id;
        private String content;
        private Instant timestamp;

        public TestMessage(long id, String content, Instant timestamp) {
            this.id = id;
            this.content = content;
            this.timestamp = timestamp;
        }

        // Getters and setters
        public long getId() { return id; }
        public void setId(long id) { this.id = id; }
        public String getContent() { return content; }
        public void setContent(String content) { this.content = content; }
        public Instant getTimestamp() { return timestamp; }
        public void setTimestamp(Instant timestamp) { this.timestamp = timestamp; }
    }
}
```

Enable scheduling in your main application class:

**`src/main/java/com/example/channelperformance/ChannelPerformanceApplication.java`**

```java
package com.example.channelperformance;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.scheduling.annotation.EnableScheduling;

@SpringBootApplication
@EnableScheduling
public class ChannelPerformanceApplication {

    public static void main(String[] args) {
        SpringApplication.run(ChannelPerformanceApplication.class, args);
    }
}
```

## Step 6: Exposing WebSocketMessageBrokerStats

Spring provides a `WebSocketMessageBrokerStats` bean that collects runtime metrics about connections, message counts, and channel statistics. We'll expose this via a REST endpoint for easy monitoring:

**`src/main/java/com/example/channelperformance/monitoring/StatsController.java`**

```java
package com.example.channelperformance.monitoring;

import org.springframework.messaging.simp.config.MessageBrokerStats;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import java.util.HashMap;
import java.util.Map;

@RestController
@RequestMapping("/api/stats")
public class StatsController {

    private final MessageBrokerStats brokerStats;

    public StatsController(MessageBrokerStats brokerStats) {
        this.brokerStats = brokerStats;
    }

    @GetMapping
    public Map<String, Object> getStats() {
        Map<String, Object> stats = new HashMap<>();
        
        // Connection metrics
        stats.put("stompBrokerRelay", brokerStats.getStompBrokerRelayStats());
        stats.put("webSocketSessionStats", brokerStats.getWebSocketSessionStats());
        
        // Inbound channel metrics
        if (brokerStats.getClientInboundChannelStats() != null) {
            Map<String, Object> inbound = new HashMap<>();
            inbound.put("executor", brokerStats.getClientInboundChannelStats().getExecutorStats());
            inbound.put("queueSize", brokerStats.getClientInboundChannelStats().getQueueSize());
            stats.put("clientInboundChannel", inbound);
        }
        
        // Outbound channel metrics
        if (brokerStats.getClientOutboundChannelStats() != null) {
            Map<String, Object> outbound = new HashMap<>();
            outbound.put("executor", brokerStats.getClientOutboundChannelStats().getExecutorStats());
            outbound.put("queueSize", brokerStats.getClientOutboundChannelStats().getQueueSize());
            stats.put("clientOutboundChannel", outbound);
        }
        
        // Overall broker stats
        stats.put("stompBrokerRelayStats", brokerStats.getStompBrokerRelayStats());
        
        return stats;
    }

    @GetMapping("/summary")
    public String getStatsSummary() {
        return brokerStats.toString();
    }
}
```

The `toString()` method on `MessageBrokerStats` returns a human-readable summary perfect for logging or simple dashboards.

### Enabling Actuator Metrics

Expose actuator endpoints for production monitoring:

**`src/main/resources/application.properties`**

```properties
# Actuator configuration
management.endpoints.web.exposure.include=health,metrics,info
management.endpoint.health.show-details=when-authorized

# Logging for debugging
logging.level.org.springframework.messaging=INFO
logging.level.org.springframework.web.socket=INFO
```

## Step 7: Building a Monitoring Dashboard Client

Create an HTML dashboard that visualizes channel metrics:

**`src/main/resources/static/dashboard.html`**

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>WebSocket Channel Performance Dashboard</title>
    <style>
        body {
            font-family: system-ui, sans-serif;
            margin: 2rem;
            background: #f5f5f5;
        }
        .container {
            max-width: 1200px;
            margin: 0 auto;
        }
        .card {
            background: white;
            border-radius: 8px;
            padding: 1.5rem;
            margin-bottom: 1.5rem;
            box-shadow: 0 2px 4px rgba(0,0,0,0.1);
        }
        h1, h2 {
            color: #333;
        }
        .metric {
            display: flex;
            justify-content: space-between;
            padding: 0.5rem 0;
            border-bottom: 1px solid #eee;
        }
        .metric-label {
            font-weight: 500;
            color: #666;
        }
        .metric-value {
            font-family: monospace;
            color: #2196F3;
        }
        .warning {
            color: #FF9800;
        }
        .error {
            color: #F44336;
        }
        button {
            background: #2196F3;
            color: white;
            border: none;
            padding: 0.75rem 1.5rem;
            border-radius: 4px;
            cursor: pointer;
            font-size: 1rem;
        }
        button:hover {
            background: #1976D2;
        }
        #status {
            margin-bottom: 1rem;
            padding: 0.75rem;
            background: #E3F2FD;
            border-radius: 4px;
        }
    </style>
</head>
<body>
    <div class="container">
        <h1>WebSocket Channel Performance Dashboard</h1>
        
        <div id="status">Disconnected</div>
        
        <div class="card">
            <h2>Connection Status</h2>
            <div id="connectionStats"></div>
        </div>
        
        <div class="card">
            <h2>Client Inbound Channel</h2>
            <div id="inboundStats"></div>
        </div>
        
        <div class="card">
            <h2>Client Outbound Channel</h2>
            <div id="outboundStats"></div>
        </div>
        
        <div class="card">
            <h2>Message Feed</h2>
            <div id="messageFeed" style="max-height: 300px; overflow-y: auto; font-family: monospace; font-size: 0.9rem;"></div>
        </div>
        
        <button onclick="connect()">Connect to Test Feed</button>
        <button onclick="disconnect()">Disconnect</button>
    </div>

    <script src="https://cdn.jsdelivr.net/npm/sockjs-client@1/dist/sockjs.min.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/stompjs@2.3.3/lib/stomp.min.js"></script>
    <script>
        let stompClient = null;
        let statsInterval = null;

        function connect() {
            const socket = new SockJS('/ws');
            stompClient = Stomp.over(socket);
            stompClient.debug = null;
            
            stompClient.connect({}, () => {
                document.getElementById('status').textContent = 'Connected';
                document.getElementById('status').style.background = '#C8E6C9';
                
                // Subscribe to test messages
                stompClient.subscribe('/topic/test', (message) => {
                    const data = JSON.parse(message.body);
                    const feed = document.getElementById('messageFeed');
                    const entry = document.createElement('div');
                    entry.textContent = `[${new Date(data.timestamp).toLocaleTimeString()}] ${data.content} (ID: ${data.id})`;
                    feed.insertBefore(entry, feed.firstChild);
                    if (feed.children.length > 50) {
                        feed.removeChild(feed.lastChild);
                    }
                });
                
                // Start polling stats
                statsInterval = setInterval(updateStats, 2000);
                updateStats();
            }, (error) => {
                document.getElementById('status').textContent = 'Connection Error: ' + error;
                document.getElementById('status').style.background = '#FFCDD2';
            });
        }

        function disconnect() {
            if (stompClient) {
                stompClient.disconnect();
                stompClient = null;
            }
            if (statsInterval) {
                clearInterval(statsInterval);
                statsInterval = null;
            }
            document.getElementById('status').textContent = 'Disconnected';
            document.getElementById('status').style.background = '#E3F2FD';
        }

        function updateStats() {
            fetch('/api/stats')
                .then(response => response.json())
                .then(data => {
                    renderConnectionStats(data);
                    renderInboundStats(data);
                    renderOutboundStats(data);
                })
                .catch(error => {
                    console.error('Failed to fetch stats:', error);
                });
        }

        function renderConnectionStats(data) {
            const container = document.getElementById('connectionStats');
            const wsStats = data.webSocketSessionStats || {};
            container.innerHTML = `
                <div class="metric">
                    <span class="metric-label">WebSocket Sessions</span>
                    <span class="metric-value">${wsStats.current || 0}</span>
                </div>
            `;
        }

        function renderInboundStats(data) {
            const container = document.getElementById('inboundStats');
            const inbound = data.clientInboundChannel || {};
            const executor = inbound.executor || {};
            const queueSize = inbound.queueSize || 0;
            
            const queueClass = queueSize > 500 ? 'error' : queueSize > 200 ? 'warning' : '';
            
            container.innerHTML = `
                <div class="metric">
                    <span class="metric-label">Queue Size</span>
                    <span class="metric-value ${queueClass}">${queueSize}</span>
                </div>
                <div class="metric">
                    <span class="metric-label">Active Threads</span>
                    <span class="metric-value">${executor.activeCount || 0}</span>
                </div>
                <div class="metric">
                    <span class="metric-label">Pool Size</span>
                    <span class="metric-value">${executor.poolSize || 0}</span>
                </div>
            `;
        }

        function renderOutboundStats(data) {
            const container = document.getElementById('outboundStats');
            const outbound = data.clientOutboundChannel || {};
            const executor = outbound.executor || {};
            const queueSize = outbound.queueSize || 0;
            
            const queueClass = queueSize > 1000 ? 'error' : queueSize > 500 ? 'warning' : '';
            
            container.innerHTML = `
                <div class="metric">
                    <span class="metric-label">Queue Size</span>
                    <span class="metric-value ${queueClass}">${queueSize}</span>
                </div>
                <div class="metric">
                    <span class="metric-label">Active Threads</span>
                    <span class="metric-value">${executor.activeCount || 0}</span>
                </div>
                <div class="metric">
                    <span class="metric-label">Pool Size</span>
                    <span class="metric-value">${executor.poolSize || 0}</span>
                </div>
            `;
        }

        // Auto-connect on load
        window.addEventListener('load', () => {
            connect();
        });
    </script>
</body>
</html>
```

This dashboard:

- Connects to the test message feed to generate load.
- Polls `/api/stats` every 2 seconds to display channel metrics.
- Highlights queue sizes that exceed warning thresholds (yellow) or error thresholds (red).
- Shows real-time message reception in the feed panel.

## Step 8: Creating a Load Test Client

To stress-test the channels, create a client that sends rapid messages:

**`src/main/resources/static/loadtest.html`**

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Load Test Client</title>
    <style>
        body { font-family: system-ui, sans-serif; margin: 2rem; }
        .controls { margin-bottom: 2rem; }
        input[type="number"] { width: 100px; padding: 0.5rem; }
        button { padding: 0.75rem 1.5rem; margin: 0.5rem; cursor: pointer; }
        #status { margin-top: 1rem; padding: 1rem; background: #f0f0f0; border-radius: 4px; }
    </style>
</head>
<body>
    <h1>STOMP Load Test Client</h1>
    
    <div class="controls">
        <label>Messages per second: <input type="number" id="rate" value="10" min="1" max="100"></label><br>
        <label>Total messages: <input type="number" id="total" value="100" min="1" max="10000"></label><br>
        <button onclick="startLoadTest()">Start Load Test</button>
        <button onclick="stopLoadTest()">Stop</button>
    </div>
    
    <div id="status">Ready</div>

    <script src="https://cdn.jsdelivr.net/npm/sockjs-client@1/dist/sockjs.min.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/stompjs@2.3.3/lib/stomp.min.js"></script>
    <script>
        let stompClient = null;
        let loadTestInterval = null;
        let sentCount = 0;
        let receivedCount = 0;
        const totalMessages = 100;

        function connect() {
            const socket = new SockJS('/ws');
            stompClient = Stomp.over(socket);
            stompClient.debug = null;
            
            stompClient.connect({}, () => {
                updateStatus('Connected');
                stompClient.subscribe('/topic/test', () => {
                    receivedCount++;
                    updateStatus(`Sent: ${sentCount}, Received: ${receivedCount}`);
                });
            }, (error) => {
                updateStatus('Error: ' + error);
            });
        }

        function startLoadTest() {
            if (!stompClient || !stompClient.connected) {
                connect();
                return;
            }
            
            const rate = parseInt(document.getElementById('rate').value);
            const total = parseInt(document.getElementById('total').value);
            sentCount = 0;
            receivedCount = 0;
            
            const interval = 1000 / rate;
            let count = 0;
            
            loadTestInterval = setInterval(() => {
                if (count >= total) {
                    stopLoadTest();
                    return;
                }
                
                stompClient.send('/app/test', {}, JSON.stringify({
                    message: `Load test message ${++count}`,
                    timestamp: Date.now()
                }));
                sentCount++;
                updateStatus(`Sent: ${sentCount}, Received: ${receivedCount}`);
            }, interval);
        }

        function stopLoadTest() {
            if (loadTestInterval) {
                clearInterval(loadTestInterval);
                loadTestInterval = null;
            }
            updateStatus(`Load test complete. Sent: ${sentCount}, Received: ${receivedCount}`);
        }

        function updateStatus(message) {
            document.getElementById('status').textContent = message;
        }

        // Add a controller handler for /app/test if needed
        // For now, we'll just use the scheduled broadcaster
    </script>
</body>
</html>
```

## Running and Observing

1. **Start the application**:
   ```bash
   ./mvnw spring-boot:run
   ```

2. **Open the dashboard**: Navigate to `http://localhost:8080/dashboard.html`. You should see:
   - Connection status
   - Real-time channel queue sizes
   - Thread pool statistics
   - Incoming test messages

3. **Open multiple browser tabs** to the dashboard to simulate multiple clients. Watch how queue sizes change as more subscribers connect.

4. **Check the stats API directly**: Visit `http://localhost:8080/api/stats` to see raw JSON metrics.

5. **Run the load test**: Open `http://localhost:8080/loadtest.html` in another tab and send bursts of messages. Observe queue depths in the dashboard.

## Interpreting Metrics

Key metrics to monitor:

- **Queue Size**: Should stay well below capacity. Consistently high values indicate the channel can't keep up.
- **Active Threads**: Should match your workload. For CPU-bound tasks, active threads near pool size suggests saturation.
- **Pool Size**: Should grow from core to max during bursts, then shrink back during idle periods.
- **WebSocket Sessions**: Tracks active connections. Sudden drops may indicate connection issues or timeouts.

## Tuning Recommendations

Based on your observations:

1. **If inbound queues consistently fill**: Increase thread pool size (for I/O-bound) or optimize message handlers (for CPU-bound).
2. **If outbound queues grow**: Reduce broadcast frequency, implement message batching, or increase outbound thread pool.
3. **If connections drop**: Check `sendTimeLimit` and `sendBufferSizeLimit`—they may be too restrictive for your client network conditions.
4. **If memory grows**: Lower queue capacities and implement backpressure at the application level.

## Next Steps

You now have:

- Custom thread pools tuned for your workload characteristics.
- Bounded queues to prevent memory issues.
- WebSocket transport limits to protect against slow clients.
- Runtime monitoring to diagnose performance bottlenecks.

In the final article of this series, we'll explore horizontal scaling with external message brokers (RabbitMQ, ActiveMQ) to handle millions of connections across multiple application instances. The channel tuning techniques from this article apply equally to broker relay configurations.

## Troubleshooting

**Issue**: Stats endpoint returns empty or null values.

**Solution**: Ensure `MessageBrokerStats` is properly injected. If using a simple broker (not a relay), some stats may be limited. Check Spring Boot version compatibility.

**Issue**: Queue sizes don't update in the dashboard.

**Solution**: Verify the executor configuration was applied. Default executors may not expose all metrics. Use custom thread pool executors as shown in the configuration examples.

**Issue**: Messages are delayed even with empty queues.

**Solution**: Check for slow message handlers. Use profiling tools to identify bottlenecks in `@MessageMapping` methods or `SimpMessagingTemplate` calls.

---

*Happy tuning! Monitor your channels, adjust thread pools, and watch your real-time applications scale smoothly.*
