# Article 9: Inside the Flow – Performance Tuning and Monitoring Spring’s Message Channels

Scaling high-frequency STOMP applications on Spring Boot means learning what happens after a frame leaves the browser. This installment tunes the `clientInboundChannel` and `clientOutboundChannel`, adjusts WebSocket transport limits, and exposes runtime metrics with `WebSocketMessageBrokerStats`. By the end, you will hammer your service via a plain HTML client, observe executor pressure in real time, and understand how to respond before users notice.

The article assumes you already have a simple STOMP setup from earlier entries: a Spring Boot 3.3+ application (Java 17+), a simple broker routing `/app` destinations to `/topic` subscribers, and a SockJS/STOMP browser client. If you are jumping in fresh, scaffold the project first:

```bash
spring init --dependencies=websocket,web,messaging,actuator --build=maven --java-version=17 stomp-performance
cd stomp-performance
./mvnw spring-boot:run
```

> **Test drive checklist**
> 1. Start the app with `./mvnw spring-boot:run`.
> 2. Open `http://localhost:8080` and connect the browser client.
> 3. Trigger burst messages to stress the broker.
> 4. Inspect metrics at `/actuator/ws-stats` or the browser dashboard.
> 5. Adjust executor sizes, limits, and queues until latency stays predictable.

---

## Understanding Spring’s Internal Channels

Spring’s annotation-driven WebSocket support is built on a series of message channels backed by executors:

- `clientInboundChannel` delivers frames from the transport layer into your annotated controllers.
- `clientOutboundChannel` handles outbound frames from the broker, STOMP endpoints, or your `SimpMessagingTemplate` calls back to the transport.

By default these are `ExecutorSubscribableChannel` instances with a single core thread, unbounded queue, and `CallerRunsPolicy`. Those settings are intentionally conservative so that newcomers rarely hit thread starvation, but they hide overload until queue latency balloons or the garbage collector screams.

Before changing anything, turn on trace logging to understand current behavior:

```yaml
# src/main/resources/application.yml
logging:
  level:
    org.springframework.messaging.simp.stomp: DEBUG
    org.springframework.messaging.simp.broker.SimpleBrokerMessageHandler: TRACE
```

Restart the application and observe the executor statistics around connection spikes. Knowing your baseline makes it easier to verify improvements later.

---

## Designing Dedicated Channel Executors

For CPU-bound work (message validation, enrichment, persistence), give the inbound channel a fixed-size pool roughly equal to available processors and a finite queue. For outbound traffic, treat delivery as I/O-bound, keeping slightly more threads and a larger queue to tolerate bursty clients.

Create `BrokerChannelConfig` to override both executors and tune WebSocket transport properties:

```java
// src/main/java/com/example/stomp/config/BrokerChannelConfig.java
package com.example.stomp.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.messaging.simp.config.ChannelRegistration;
import org.springframework.messaging.simp.config.MessageBrokerRegistry;
import org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor;
import org.springframework.web.socket.config.annotation.EnableWebSocketMessageBroker;
import org.springframework.web.socket.config.annotation.StompEndpointRegistry;
import org.springframework.web.socket.config.annotation.WebSocketMessageBrokerConfigurer;
import org.springframework.web.socket.config.annotation.WebSocketTransportRegistration;

@Configuration
@EnableWebSocketMessageBroker
public class BrokerChannelConfig implements WebSocketMessageBrokerConfigurer {

    @Bean(name = "clientInboundChannelExecutor")
    public ThreadPoolTaskExecutor clientInboundExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        int cores = Runtime.getRuntime().availableProcessors();
        executor.setThreadNamePrefix("ws-inbound-");
        executor.setCorePoolSize(cores);
        executor.setMaxPoolSize(cores * 2);
        executor.setQueueCapacity(500);
        executor.setAllowCoreThreadTimeOut(true);
        executor.setKeepAliveSeconds(30);
        executor.initialize();
        return executor;
    }

    @Bean(name = "clientOutboundChannelExecutor")
    public ThreadPoolTaskExecutor clientOutboundExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setThreadNamePrefix("ws-outbound-");
        executor.setCorePoolSize(4);
        executor.setMaxPoolSize(8);
        executor.setQueueCapacity(1000);
        executor.setAllowCoreThreadTimeOut(true);
        executor.setKeepAliveSeconds(60);
        executor.initialize();
        return executor;
    }

    @Bean
    public ThreadPoolTaskExecutor heartBeatScheduler() {
        ThreadPoolTaskExecutor scheduler = new ThreadPoolTaskExecutor();
        scheduler.setThreadNamePrefix("ws-heartbeat-");
        scheduler.setCorePoolSize(1);
        scheduler.setMaxPoolSize(1);
        scheduler.initialize();
        return scheduler;
    }

    @Override
    public void configureMessageBroker(MessageBrokerRegistry registry) {
        registry.setApplicationDestinationPrefixes("/app");
        registry.enableSimpleBroker("/topic").setTaskScheduler(heartBeatScheduler());
    }

    @Override
    public void configureClientInboundChannel(ChannelRegistration registration) {
        registration.taskExecutor(clientInboundExecutor());
    }

    @Override
    public void configureClientOutboundChannel(ChannelRegistration registration) {
        registration.taskExecutor(clientOutboundExecutor());
    }

    @Override
    public void configureWebSocketTransport(WebSocketTransportRegistration registry) {
        registry
            .setSendTimeLimit(15_000)          // 15 seconds max to push a frame
            .setSendBufferSizeLimit(512 * 1024)
            .setTimeToFirstMessage(10_000);
    }

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/ws").setAllowedOriginPatterns("*").withSockJS();
    }
}
```

Key choices:

- **Inbound pool**: bounded queue forces backpressure once 500 frames pile up, exposing overload quickly.
- **Outbound pool**: more threads and queue capacity so slow clients do not starve faster ones immediately.
- **Heartbeat scheduler**: isolated executor prevents broker pings from competing with application work.
- **Transport limits**: `setSendTimeLimit` ejects slow clients, `setSendBufferSizeLimit` protects heap, `setTimeToFirstMessage` guards handshake stragglers.

Consider setting a custom rejection handler if you prefer logging or metrics on queue overflow:

```java
executor.setRejectedExecutionHandler((runnable, exec) ->
        log.warn("Inbound executor saturated: {}", exec));
```

Switching to `AbortPolicy` makes overload visible to clients via STOMP error frames, prompting reconnect/backoff logic on the browser side.

---

## Observability with WebSocketMessageBrokerStats

Spring exposes channel activity through `WebSocketMessageBrokerStats`. Wire it into an actuator-friendly controller:

```java
// src/main/java/com/example/stomp/metrics/BrokerMetricsController.java
package com.example.stomp.metrics;

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.socket.config.WebSocketMessageBrokerStats;

import java.time.Instant;
import java.util.Map;

@RestController
public class BrokerMetricsController {

    private final WebSocketMessageBrokerStats stats;

    public BrokerMetricsController(WebSocketMessageBrokerStats stats) {
        this.stats = stats;
    }

    @GetMapping("/actuator/ws-stats")
    public Map<String, Object> brokerStats() {
        return Map.of(
                "timestamp", Instant.now(),
                "inboundExecutor", stats.getClientInboundExecutorStatsInfo(),
                "outboundExecutor", stats.getClientOutboundExecutorStatsInfo(),
                "sockJsSession", stats.getSockJsSessionStatsInfo(),
                "stompBrokerRelay", stats.getStompBrokerRelayStatsInfo(),
                "simpMessagingTemplate", stats.getSimpMessagingTemplateStatsInfo()
        );
    }
}
```

Apply `@EnableScheduling` and a simple logger if you want periodic snapshots without polling:

```java
// src/main/java/com/example/stomp/metrics/ChannelLoadLogger.java
package com.example.stomp.metrics;

import lombok.extern.slf4j.Slf4j;
import org.springframework.scheduling.annotation.EnableScheduling;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Component;
import org.springframework.web.socket.config.WebSocketMessageBrokerStats;

@Slf4j
@Component
@EnableScheduling
public class ChannelLoadLogger {

    private final WebSocketMessageBrokerStats stats;

    public ChannelLoadLogger(WebSocketMessageBrokerStats stats) {
        this.stats = stats;
    }

    @Scheduled(fixedRate = 5_000)
    public void logStats() {
        log.info("Inbound executor: {}", stats.getClientInboundExecutorStatsInfo());
        log.info("Outbound executor: {}", stats.getClientOutboundExecutorStatsInfo());
    }
}
```

If you are already shipping metrics to Prometheus or another backend, register gauges via Micrometer:

```java
// src/main/java/com/example/stomp/metrics/WebSocketMetricsConfiguration.java
package com.example.stomp.metrics;

import io.micrometer.core.instrument.MeterRegistry;
import jakarta.annotation.PostConstruct;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.socket.config.WebSocketMessageBrokerStats;

@Configuration
public class WebSocketMetricsConfiguration {

    private final WebSocketMessageBrokerStats stats;
    private final MeterRegistry meterRegistry;

    public WebSocketMetricsConfiguration(WebSocketMessageBrokerStats stats,
                                         MeterRegistry meterRegistry) {
        this.stats = stats;
        this.meterRegistry = meterRegistry;
    }

    @PostConstruct
    public void bindGauges() {
        meterRegistry.gauge("stomp.client.inbound.queue.size", stats,
                s -> s.getClientInboundExecutorStats().getQueueSize());
        meterRegistry.gauge("stomp.client.outbound.queue.size", stats,
                s -> s.getClientOutboundExecutorStats().getQueueSize());
        meterRegistry.gauge("stomp.client.inbound.active.count", stats,
                s -> s.getClientInboundExecutorStats().getActiveCount());
    }
}
```

Expose `/actuator/prometheus` or your registry of choice, then graph queue depth versus application latency to understand pressure points.

---

## A Load-Testing Browser Client

Give your readers an immediate way to stress-test the broker by upgrading the static HTML client. Place the following file at `src/main/resources/static/index.html`:

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8" />
    <title>STOMP Performance Playground</title>
    <style>
        body { font-family: system-ui, sans-serif; margin: 2rem; }
        section { margin-top: 1.5rem; }
        textarea { width: 100%; min-height: 120px; }
        pre { background: #111; color: #0ff; padding: 1rem; height: 220px; overflow: auto; }
        label { margin-right: 1rem; }
    </style>
</head>
<body>
<h1>STOMP Performance Playground</h1>
<p>Connect, send bursts, and watch broker metrics refresh every 5 seconds.</p>

<div>
    <button id="connect">Connect</button>
    <button id="disconnect" disabled>Disconnect</button>
</div>

<div>Status: <span id="status">disconnected</span></div>

<section>
    <label>Burst size <input type="number" id="burstSize" value="50" /></label>
    <label>Interval (ms) <input type="number" id="interval" value="500" /></label>
    <button id="startBurst" disabled>Start burst</button>
    <button id="stopBurst" disabled>Stop burst</button>
</section>

<section>
    <h2>Manual Send</h2>
    <textarea id="message" placeholder="Enter message payload"></textarea>
    <button id="send" disabled>Send Once</button>
</section>

<section>
    <h2>Inbound Feed</h2>
    <pre id="feed"></pre>
</section>

<section>
    <h2>Broker Stats</h2>
    <pre id="stats"></pre>
</section>

<script src="https://cdn.jsdelivr.net/npm/sockjs-client@1/dist/sockjs.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/stompjs@2.3.3/lib/stomp.min.js"></script>
<script>
    let stompClient = null;
    let burstHandle = null;

    const statusEl = document.getElementById('status');
    const feedEl = document.getElementById('feed');
    const statsEl = document.getElementById('stats');
    const messageEl = document.getElementById('message');

    function appendFeed(text) {
        feedEl.textContent = `${new Date().toISOString()} → ${text}\n` + feedEl.textContent;
    }

    function fetchStats() {
        fetch('/actuator/ws-stats')
            .then(res => res.json())
            .then(data => statsEl.textContent = JSON.stringify(data, null, 2))
            .catch(err => console.error('Stats fetch failed', err));
    }

    setInterval(() => {
        if (stompClient) {
            fetchStats();
        }
    }, 5000);

    document.getElementById('connect').addEventListener('click', () => {
        const socket = new SockJS('/ws');
        stompClient = Stomp.over(socket);
        stompClient.debug = (msg) => console.log(msg);
        stompClient.connect({}, () => {
            statusEl.textContent = 'connected';
            document.getElementById('disconnect').disabled = false;
            document.getElementById('connect').disabled = true;
            document.getElementById('send').disabled = false;
            document.getElementById('startBurst').disabled = false;

            stompClient.subscribe('/topic/updates', message => {
                appendFeed(message.body);
            });

            fetchStats();
        }, error => {
            statusEl.textContent = `error: ${error}`;
        });
    });

    document.getElementById('disconnect').addEventListener('click', () => {
        if (stompClient) {
            stompClient.disconnect(() => {
                statusEl.textContent = 'disconnected';
            });
            stompClient = null;
        }

        document.getElementById('disconnect').disabled = true;
        document.getElementById('connect').disabled = false;
        document.getElementById('send').disabled = true;
        document.getElementById('startBurst').disabled = true;
        document.getElementById('stopBurst').disabled = true;
        clearInterval(burstHandle);
    });

    document.getElementById('send').addEventListener('click', () => {
        stompClient.send('/app/broadcast', {}, JSON.stringify({
            message: messageEl.value || 'hello'
        }));
    });

    document.getElementById('startBurst').addEventListener('click', () => {
        const burstSize = parseInt(document.getElementById('burstSize').value, 10);
        const interval = parseInt(document.getElementById('interval').value, 10);

        burstHandle = setInterval(() => {
            for (let i = 0; i < burstSize; i++) {
                stompClient.send('/app/broadcast', {}, JSON.stringify({
                    message: `Burst payload ${Math.random().toString(36).slice(2, 8)}`
                }));
            }
        }, interval);

        document.getElementById('stopBurst').disabled = false;
    });

    document.getElementById('stopBurst').addEventListener('click', () => {
        clearInterval(burstHandle);
        document.getElementById('stopBurst').disabled = true;
    });
</script>
</body>
</html>
```

This small control panel supports single-message sends, configurable bursts, and a live view of executor stats.

---

## Echo Controller for Testing

Ensure the backend echoes messages back to `/topic/updates` so every connected browser sees the load they generate:

```java
// src/main/java/com/example/stomp/api/BroadcastController.java
package com.example.stomp.api;

import org.springframework.messaging.handler.annotation.MessageMapping;
import org.springframework.messaging.handler.annotation.Payload;
import org.springframework.messaging.simp.SimpMessagingTemplate;
import org.springframework.stereotype.Controller;

import java.time.Instant;
import java.util.Map;

@Controller
public class BroadcastController {

    private final SimpMessagingTemplate messagingTemplate;

    public BroadcastController(SimpMessagingTemplate messagingTemplate) {
        this.messagingTemplate = messagingTemplate;
    }

    @MessageMapping("/broadcast")
    public void broadcast(@Payload Map<String, Object> payload) {
        payload.put("serverTimestamp", Instant.now().toString());
        messagingTemplate.convertAndSend("/topic/updates", payload);
    }
}
```

---

## Running Experiments

1. **Baseline** – Launch the app and connect two browser tabs. Send a few manual messages and verify everything lands in the feed.
2. **Burst traffic** – Increase burst size to 200, lower interval to 100 ms, and watch queue size, pool size, and active counts rise. Tune executor limits to keep queues short.
3. **Slow client** – Open the page in a separate machine or throttle network (Chrome DevTools → Network → Slow 3G). Observe `sendTimeLimit` kicking in once the outbound buffer exceeds the limit.
4. **Metrics correlation** – Collect application latency, GC pauses, and WebSocket stats to correlate spikes with backlog.
5. **Fail-fast strategy** – Experiment with smaller queue capacities. Use STOMP error frames to notify clients to back off when the queue fills.

---

## Troubleshooting Patterns

- **Inbound queue grows while active threads stay low** → CPU-bound handlers; consider reducing message work, increasing cores, or offloading to a distributed queue.
- **Outbound queue grows, clients disconnect** → slow receivers; segment them into dedicated topics, lower burst size, or enable acknowledgements (`ack=client-individual`).
- **SockJS session count balloons** → sticky session problems in load balancers; ensure `JSESSIONID` affinity or switch to WebSocket-only endpoints when possible.
- **Frequent send limit violations** → revisit `setSendTimeLimit` and `setSendBufferSizeLimit`, or implement client-side rate limiting.

---

## Next Steps

- Feed `WebSocketMessageBrokerStats` into Grafana dashboards to spot regressions after deploys.
- Add integration tests using `WebSocketStompClient` to simulate burst traffic in CI.
- For production, consider offloading message routing to RabbitMQ or ActiveMQ and keeping the tunable settings described here per node.
- Up next in the series: production packaging, deployment topology, synthetic monitoring, and long-term health checks.

You now have the tooling to observe and tune Spring’s messaging internals before they become a bottleneck. Keep iteration tight—observe, tweak, measure—and your STOMP workloads will stay fast even under heavy fire.

