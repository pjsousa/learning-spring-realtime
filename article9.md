# Inside the Flow: Performance Tuning and Monitoring Spring’s Message Channels

Building real-time experiences is exhilarating, but sustaining them under load requires a deep understanding of Spring’s messaging internals. From inbound controller execution to outbound broker dispatch, every STOMP frame flows through configurable channels backed by thread pools and queues. Misconfigured dispatchers can bottleneck throughput, while missing observability makes diagnosing issues painful.

In this ninth article we will open the hood on Spring’s WebSocket message broker implementation. You will learn how to tune thread pools for inbound and outbound processing, configure transport-level limits, instrument the broker with metrics and logs, and profile live traffic. We will apply these techniques to our cursor streaming demo, but the lessons apply to any high-frequency STOMP application.

By the end of this article you will:

- Understand the roles of `clientInboundChannel`, `clientOutboundChannel`, and `brokerChannel`.
- Configure executor pools with optimal core sizes, queue capacities, and rejection policies.
- Adjust WebSocket transport settings such as `sendTimeLimit` and `messageSizeLimit`.
- Expose broker metrics using `WebSocketMessageBrokerStats` and Micrometer.
- Capture trace logs and sample message payloads for debugging.
- Develop load-testing strategies using tools like Gatling or custom scripts.

Let’s make our toy project production-ready.

## Recap: Message Flow in Spring WebSocket

When a STOMP frame arrives, Spring routes it through several channels:

1. **clientInboundChannel**: Receives messages from connected clients. Here, Spring resolves destination mappings (`@MessageMapping`) and invokes controller methods.
2. **brokerChannel**: Handles messages destined for the simple broker or broker relay. This channel broadcasts to subscribed clients.
3. **clientOutboundChannel**: Delivers messages from the broker back to clients.

Each channel uses an executor with configurable thread pools. By default, Spring Boot sets conservative values (core size 1, max size 1, queue capacity 100 for simple broker). High-frequency workloads quickly saturate these defaults, causing messages to queue or be dropped.

## Configuring Channel Executors

Create a configuration class to tune the executors:

```java
package com.example.stompchat.config;

import org.springframework.context.annotation.Configuration;
import org.springframework.messaging.simp.config.ChannelRegistration;
import org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor;
import org.springframework.web.socket.config.annotation.WebSocketMessageBrokerConfigurer;

@Configuration
public class BrokerPerformanceConfig implements WebSocketMessageBrokerConfigurer {

    @Override
    public void configureClientInboundChannel(ChannelRegistration registration) {
        registration.taskExecutor(pool("ws-inbound", 4, 16, 2000));
    }

    @Override
    public void configureClientOutboundChannel(ChannelRegistration registration) {
        registration.taskExecutor(pool("ws-outbound", 4, 16, 2000));
    }

    private ThreadPoolTaskExecutor pool(String prefix, int core, int max, int queueCapacity) {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setThreadNamePrefix(prefix + "-");
        executor.setCorePoolSize(core);
        executor.setMaxPoolSize(max);
        executor.setQueueCapacity(queueCapacity);
        executor.setAllowCoreThreadTimeOut(true);
        executor.setKeepAliveSeconds(60);
        executor.initialize();
        return executor;
    }
}
```

Here we provision up to 16 threads for both inbound and outbound channels with a queue capacity of 2000 messages. Adjust these numbers based on CPU cores and expected load. If messages exceed the queue capacity, Spring logs warnings and may drop frames depending on the rejection policy (default: caller runs). Monitor queue depths to avoid overload.

For the broker channel, use `configureMessageBroker` to specify `setApplicationDestinationPrefixes` and optionally configure the simple broker executor:

```java
registry.enableSimpleBroker("/topic", "/queue")
        .setTaskScheduler(heartbeatScheduler())
        .setThreadPoolExecutor(new ConcurrentTaskExecutor(pool("ws-broker", 4, 8, 1000)));
```

This code uses a custom executor for the simple broker, ensuring it handles fan-out efficiently. If you use an external broker relay (Article 10), tuning the simple broker matters less, but the inbound/outbound executors remain crucial.

## Adjusting Transport-Level Limits

Beyond channel executors, Spring provides transport settings that control per-connection behavior. Override `configureWebSocketTransport` in your configuration:

```java
@Override
public void configureWebSocketTransport(WebSocketTransportRegistration registration) {
    registration
            .setSendTimeLimit(20_000)        // 20 seconds to send a message before closing the session
            .setSendBufferSizeLimit(1_048_576) // 1 MB buffer per session
            .setMessageSizeLimit(128 * 1024);   // drop messages larger than 128 KB
}
```

These defaults buffer moderate bursts without overwhelming the server. For high-frequency cursor updates, message size is small, so the key is to allow enough buffer for brief spikes.

## Instrumenting with WebSocketMessageBrokerStats

Spring automatically registers a `WebSocketMessageBrokerStats` bean that exposes insightful metrics, including executor pool sizes, queue depths, and session counts. To view them, inject the bean and expose an actuator endpoint or log periodically.

### Logging Every 30 Seconds

```java
@Component
public class BrokerMetricsLogger {

    private static final Logger log = LoggerFactory.getLogger(BrokerMetricsLogger.class);
    private final WebSocketMessageBrokerStats stats;

    public BrokerMetricsLogger(WebSocketMessageBrokerStats stats) {
        this.stats = stats;
    }

    @Scheduled(fixedRate = 30_000)
    public void logStats() {
        log.info("Broker metrics:\n{}", stats); // stats.toString() formats all metrics
    }
}
```

The log output includes information like `clientInboundExecutorActiveCount`, `clientOutboundExecutorQueueSize`, and `currentWebSocketSessionCount`. Use these metrics to detect saturation before it impacts users.

### Exposing via Actuator

Add the following dependencies and configuration:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

```properties
management.endpoints.web.exposure.include=health,info,metrics,threaddump
```

Then register a custom metrics binder:

```java
@Component
public class BrokerMetricsBinder implements MeterBinder {

    private final WebSocketMessageBrokerStats stats;

    public BrokerMetricsBinder(WebSocketMessageBrokerStats stats) {
        this.stats = stats;
    }

    @Override
    public void bindTo(MeterRegistry registry) {
        registry.gauge("stomp.sessions", stats, s -> s.getWebSocketSessionStatsInfo().getCurrentSessions());
        registry.gauge("stomp.inbound.queue.size", stats,
                s -> s.getClientInboundExecutorStatsInfo().getQueueSize());
        registry.gauge("stomp.outbound.queue.size", stats,
                s -> s.getClientOutboundExecutorStatsInfo().getQueueSize());
    }
}
```

Now you can scrape metrics using Prometheus or visualize them in Grafana.

## Instrumenting Message Flow with Logging

For debugging, enable trace logging on Spring’s messaging packages:

```properties
logging.level.org.springframework.web.socket.messaging=DEBUG
logging.level.org.springframework.messaging.simp.stomp=TRACE
```

This reveals detailed logs for every STOMP frame. Use caution in production—logs can grow rapidly. For targeted tracing, implement a `ChannelInterceptor` that logs specific destinations or messages exceeding a threshold.

Example selective logger:

```java
public class CursorLoggingInterceptor implements ChannelInterceptor {

    private static final Logger log = LoggerFactory.getLogger(CursorLoggingInterceptor.class);

    @Override
    public Message<?> preSend(Message<?> message, MessageChannel channel) {
        StompHeaderAccessor accessor = StompHeaderAccessor.wrap(message);
        if ("/app/cursor".equals(accessor.getDestination())) {
            log.trace("Cursor frame payload: {}", new String((byte[]) message.getPayload(), StandardCharsets.UTF_8));
        }
        return message;
    }
}
```

Attach this interceptor only during debugging to avoid performance penalties.

## Load Testing Your STOMP Endpoints

Before deploying to production, simulate load to ensure your tuning works. Here are two approaches:

### 1. Using Gatling with WebSocket Support

Gatling’s WebSocket DSL can connect to `/sockjs` or `/ws` endpoints. Example scenario (Scala):

```scala
class CursorSimulation extends Simulation {
  val httpProtocol = http.baseUrl("http://localhost:8080")

  val feeder = Iterator.continually(Map("x" -> Random.nextDouble(), "y" -> Random.nextDouble()))

  val scn = scenario("Cursor Flow")
    .forever {
      exec(ws("Connect").connect("/sockjs/websocket"))
      .feed(feeder)
      .exec(ws("Send cursor")
        .sendText(session => s"{""xPercent"":${session("x").as[Double]},""yPercent"":${session("y").as[Double]}}"))
      .pause(0.05)
    }

  setUp(scn.inject(rampUsers(200).during(30.seconds))).protocols(httpProtocol)
}
```

This script stresses the cursor endpoint with 200 simulated users. Monitor CPU usage, queue depths, and network throughput.

### 2. Custom JMH or Reactor-Based Load Generator

For finer control, write a Reactor-based client that opens numerous STOMP sessions and publishes messages. Use `ReactorNettyWebSocketClient` and `StompClientSupport` to craft asynchronous simulations tailored to your environment.

## Handling Backpressure and Slow Clients

If some clients cannot keep up with outbound traffic (perhaps due to slow connections), the server may close their sessions to protect overall health. Monitor logs for warnings like `Sending heartbeat to client took longer than...` or `Execution of session send failed`. You can mitigate this by:

- Lowering `setSendTimeLimit` to detect slow clients quickly.
- Introducing application-level backpressure (for example, dropping less important updates when the outbound queue grows).
- Grouping high-frequency updates into batches rather than sending every event individually.

For the cursor demo, you might send combined updates every 100ms containing multiple positions instead of dozens of individual events.

## Profiling Server-Side Hotspots

When tuning thread pools and buffers isn’t enough, profile the application to find hot paths:

- Use Java Flight Recorder or Async Profiler to capture CPU hotspots while running load tests.
- Pay attention to JSON serialization costs—Jackson can become a bottleneck. Consider switching to a faster serializer (e.g., `jackson-afterburner`, `Kryo`, or JSON-B) or precomputing JSON strings.
- Ensure logging at INFO or DEBUG level is not flooding disk; for high-frequency endpoints, log only aggregated metrics.

## Observability Dashboards

Create dashboards showing:

- Active WebSocket session count over time.
- Inbound/outbound executor queue size and latency.
- STOMP error frames per minute.
- Heartbeat failures.

Use Micrometer + Prometheus + Grafana to chart these metrics. Alerts can notify you when queue sizes exceed thresholds or session counts drop unexpectedly.

## Checklist for Performance Readiness

Before shipping, run through this checklist:

- [ ] Channel executors sized for expected peak concurrency.
- [ ] Transport limits configured (send buffer, message size, heartbeat intervals).
- [ ] Metrics captured via `WebSocketMessageBrokerStats` and exposed through Actuator/Micrometer.
- [ ] Load-tested with realistic publish rates.
- [ ] Logging tuned (no excessive per-message logs in production).
- [ ] Backpressure strategy in place for slow clients.

Completing this checklist ensures your STOMP application can handle real-world load gracefully.

## Summary and Next Steps

We have demystified Spring’s messaging internals and equipped our toy project with production-ready tuning and observability. By sizing executor pools, adjusting transport limits, and instrumenting the broker, you can scale real-time features confidently.

In the final article we will connect our application to an external STOMP broker relay (RabbitMQ/ActiveMQ) to support massive scale and clustering. Before proceeding, make sure you can:

- Monitor queue sizes and executor activity under load.
- Adjust configuration to eliminate bottlenecks.
- Interpret `WebSocketMessageBrokerStats` output.

With performance under control, we are ready to take the next step toward enterprise scaling.

