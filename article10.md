# Scaling to Millions: Implementing the STOMP Broker Relay for Enterprise Applications

## Introduction

Throughout this series, we've used Spring's simple in-memory message broker. It's perfect for learning, development, and small-scale deployments. However, as your application grows to handle thousands or millions of concurrent connections, the in-memory broker becomes a bottleneck. It can't scale beyond a single application instance, has no persistence, and lacks the reliability features needed for production.

The solution is the **STOMP Broker Relay**: Spring's mechanism for connecting to an external, dedicated message broker via STOMP protocol. Popular choices include RabbitMQ and ActiveMQ, both of which support STOMP natively and provide clustering, persistence, high availability, and enterprise-grade reliability.

In this final article, we'll replace the simple broker with a RabbitMQ broker relay, configure it for production use, and understand how this enables horizontal scaling across multiple Spring Boot application instances. This is the architecture that powers applications serving millions of real-time connections.

## Understanding the Broker Relay Architecture

When using a broker relay:

**Application Instances**: Multiple Spring Boot instances can run behind a load balancer. Each instance connects to the same external broker.

**Message Flow**: 
1. Client sends message to Instance A
2. Instance A processes it and sends to broker
3. Broker routes message to all subscribers
4. Instance B (with a client subscribed) receives message from broker
5. Instance B delivers to its client

**Key Benefits**:
- **Horizontal Scaling**: Add more application instances as needed
- **High Availability**: Broker can be clustered for failover
- **Persistence**: Messages can be persisted if broker supports it
- **Reliability**: Enterprise brokers handle millions of messages efficiently
- **Cross-Language**: Other services can publish to the broker directly

## Project Setup

We'll configure RabbitMQ as our external broker. You'll need:

1. RabbitMQ installed and running (or use Docker: `docker run -d -p 5672:5672 -p 15672:15672 rabbitmq:3-management`)
2. RabbitMQ STOMP plugin enabled (usually enabled by default)
3. Spring Boot application from previous articles

### Step 1: Add RabbitMQ Dependencies

Update your `pom.xml`:

```xml
<dependencies>
    <!-- Existing dependencies -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-websocket</artifactId>
    </dependency>
    
    <!-- Add RabbitMQ (optional, for direct access if needed) -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-amqp</artifactId>
    </dependency>
</dependencies>
```

Note: The AMQP dependency is optional if you're only using STOMP. Spring can connect to RabbitMQ via STOMP without it.

### Step 2: Configure RabbitMQ Connection

Create `application.yml`:

```yaml
spring:
  rabbitmq:
    host: localhost
    port: 5672
    username: guest
    password: guest
    virtual-host: /

# Custom broker configuration
stomp:
  broker:
    relay:
      host: localhost
      port: 61613  # RabbitMQ STOMP port
      client-login: guest
      client-passcode: guest
      system-login: guest
      system-passcode: guest
```

Or use `application.properties`:

```properties
spring.rabbitmq.host=localhost
spring.rabbitmq.port=5672
spring.rabbitmq.username=guest
spring.rabbitmq.password=guest

stomp.broker.relay.host=localhost
stomp.broker.relay.port=61613
stomp.broker.relay.client-login=guest
stomp.broker.relay.client-passcode=guest
stomp.broker.relay.system-login=guest
stomp.broker.relay.system-passcode=guest
```

## Configuring the Broker Relay

Now let's update our WebSocket configuration to use the broker relay instead of the simple broker.

### Step 3: Update WebSocket Configuration

Update `WebSocketConfig.java`:

```java
package com.example.stomptoyproject;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.messaging.simp.config.MessageBrokerRegistry;
import org.springframework.web.socket.config.annotation.EnableWebSocketMessageBroker;
import org.springframework.web.socket.config.annotation.StompEndpointRegistry;
import org.springframework.web.socket.config.annotation.WebSocketMessageBrokerConfigurer;

@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {

    @Value("${stomp.broker.relay.host:localhost}")
    private String relayHost;

    @Value("${stomp.broker.relay.port:61613}")
    private int relayPort;

    @Value("${stomp.broker.relay.client-login:guest}")
    private String clientLogin;

    @Value("${stomp.broker.relay.client-passcode:guest}")
    private String clientPasscode;

    @Value("${stomp.broker.relay.system-login:guest}")
    private String systemLogin;

    @Value("${stomp.broker.relay.system-passcode:guest}")
    private String systemPasscode;

    @Override
    public void configureMessageBroker(MessageBrokerRegistry config) {
        // Replace enableSimpleBroker with enableStompBrokerRelay
        config.enableStompBrokerRelay("/topic", "/queue")
              .setRelayHost(relayHost)
              .setRelayPort(relayPort)
              .setClientLogin(clientLogin)
              .setClientPasscode(clientPasscode)
              .setSystemLogin(systemLogin)
              .setSystemPasscode(systemPasscode)
              .setVirtualHost("/")
              .setSystemHeartbeatSendInterval(10000)  // Send heartbeat every 10s
              .setSystemHeartbeatReceiveInterval(10000); // Expect heartbeat every 10s
              
        config.setApplicationDestinationPrefixes("/app");
        config.setUserDestinationPrefix("/user");
    }

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/gs-guide-websocket")
                .setAllowedOriginPatterns("*")
                .withSockJS();
    }
}
```

Key configuration options:

**enableStompBrokerRelay()**: Replaces `enableSimpleBroker()`. Tells Spring to connect to an external STOMP broker instead of using in-memory broker.

**setRelayHost/setRelayPort()**: Broker connection details.

**setClientLogin/setClientPasscode()**: Credentials for client connections (messages from Spring to broker).

**setSystemLogin/setSystemPasscode()**: Credentials for system connections (Spring's own management connection).

**setSystemHeartbeatSendInterval/ReceiveInterval()**: Heartbeat configuration to detect connection failures.

**setVirtualHost()**: RabbitMQ virtual host (usually "/").

## Understanding Broker Relay Connections

When using a broker relay, Spring establishes two connections to the broker:

**System Connection**: Spring's management connection used for:
- Destination management
- Subscription management  
- Health checks

**Client Connections**: Separate connections for each application instance's message delivery. These are pooled and reused.

## Enabling RabbitMQ STOMP Plugin

Ensure RabbitMQ has STOMP enabled:

```bash
# If using Docker
docker exec -it <container-id> rabbitmq-plugins enable rabbitmq_stomp

# If installed locally
rabbitmq-plugins enable rabbitmq_stomp
```

Verify STOMP is listening:

```bash
netstat -an | grep 61613
# or
lsof -i :61613
```

RabbitMQ should be listening on port 61613 for STOMP connections.

## Testing the Broker Relay

Let's verify the connection works.

### Step 4: Create Connection Test

Update any controller to log broker status:

```java
@Controller
public class BrokerTestController {

    @Autowired
    private SimpMessagingTemplate messagingTemplate;

    @MessageMapping("/test.broker")
    @SendTo("/topic/test-response")
    public String testBroker(String message) {
        System.out.println("Message received: " + message);
        System.out.println("Sending to broker relay...");
        return "Echo: " + message + " (via broker relay)";
    }
}
```

Test with a simple client - if messages flow correctly, the broker relay is working!

## Configuring for Production

Production configurations need additional considerations.

### Step 5: Production Configuration

Create `application-prod.yml`:

```yaml
spring:
  rabbitmq:
    host: ${RABBITMQ_HOST:rabbitmq.example.com}
    port: ${RABBITMQ_PORT:5672}
    username: ${RABBITMQ_USERNAME:app_user}
    password: ${RABBITMQ_PASSWORD:secure_password}
    virtual-host: ${RABBITMQ_VHOST:/prod}
    connection-timeout: 10000
    requested-heartbeat: 30

stomp:
  broker:
    relay:
      host: ${RABBITMQ_HOST:rabbitmq.example.com}
      port: ${STOMP_PORT:61613}
      client-login: ${RABBITMQ_USERNAME:app_user}
      client-passcode: ${RABBITMQ_PASSWORD:secure_password}
      system-login: ${RABBITMQ_SYSTEM_USER:system_user}
      system-passcode: ${RABBITMQ_SYSTEM_PASSWORD:system_password}
      virtual-host: ${RABBITMQ_VHOST:/prod}
      system-heartbeat-send-interval: 10000
      system-heartbeat-receive-interval: 10000
```

Use environment variables for sensitive credentials in production!

## Connection Resilience

Add reconnection logic and error handling.

### Step 6: Add Connection Resilience

Update `WebSocketConfig.java`:

```java
@Override
public void configureMessageBroker(MessageBrokerRegistry config) {
    config.enableStompBrokerRelay("/topic", "/queue")
          .setRelayHost(relayHost)
          .setRelayPort(relayPort)
          .setClientLogin(clientLogin)
          .setClientPasscode(clientPasscode)
          .setSystemLogin(systemLogin)
          .setSystemPasscode(systemPasscode)
          .setVirtualHost("/")
          .setSystemHeartbeatSendInterval(10000)
          .setSystemHeartbeatReceiveInterval(10000)
          .setAutoStartup(true)  // Start automatically
          .setClientHeartbeatReceiveInterval(0)  // Disable client heartbeat checks
          .setClientHeartbeatSendInterval(0);    // Let broker handle it
          
    // ... rest of configuration
}
```

Spring automatically handles reconnection, but you can add custom logic:

```java
@Component
public class BrokerConnectionListener {

    private static final Logger logger = LoggerFactory.getLogger(BrokerConnectionListener.class);

    @EventListener
    public void handleBrokerAvailabilityEvent(BrokerAvailabilityEvent event) {
        if (event.isBrokerAvailable()) {
            logger.info("STOMP broker relay is now available");
        } else {
            logger.warn("STOMP broker relay is not available");
            // Implement your reconnection/alerting logic here
        }
    }
}
```

## Scaling Across Multiple Instances

The key benefit: multiple application instances can share the same broker.

### Step 7: Load Balancer Configuration

Deploy multiple instances behind a load balancer:

```
                    Load Balancer
                         |
        +----------------+----------------+
        |                |                |
   Instance A       Instance B       Instance C
        |                |                |
        +----------------+----------------+
                         |
                    RabbitMQ Broker
                         |
              (Single source of truth)
```

Clients can connect to any instance via the load balancer. Messages sent through any instance are delivered to clients connected to any other instance.

## Monitoring Broker Connections

Monitor broker health in production.

### Step 8: Create Broker Health Check

Create `BrokerHealthIndicator.java`:

```java
package com.example.stomptoyproject;

import org.springframework.boot.actuate.health.Health;
import org.springframework.boot.actuate.health.HealthIndicator;
import org.springframework.messaging.simp.stomp.StompBrokerRelayMessageHandler;
import org.springframework.stereotype.Component;

@Component
public class BrokerHealthIndicator implements HealthIndicator {

    private final StompBrokerRelayMessageHandler brokerRelay;

    public BrokerHealthIndicator(StompBrokerRelayMessageHandler brokerRelay) {
        this.brokerRelay = brokerRelay;
    }

    @Override
    public Health health() {
        boolean isRelayAvailable = brokerRelay.isRelayAvailable();
        
        if (isRelayAvailable) {
            return Health.up()
                    .withDetail("broker", "Connected")
                    .withDetail("relayHost", brokerRelay.getRelayHost())
                    .build();
        } else {
            return Health.down()
                    .withDetail("broker", "Disconnected")
                    .withDetail("error", "Cannot connect to STOMP broker")
                    .build();
        }
    }
}
```

Enable Actuator for health checks:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

Access health: `GET /actuator/health`

## Using ActiveMQ Instead of RabbitMQ

ActiveMQ also supports STOMP and can be used similarly:

### Step 9: ActiveMQ Configuration (Alternative)

```java
@Override
public void configureMessageBroker(MessageBrokerRegistry config) {
    config.enableStompBrokerRelay("/topic", "/queue")
          .setRelayHost("localhost")
          .setRelayPort(61613)  // ActiveMQ STOMP port
          .setClientLogin("admin")
          .setClientPasscode("admin")
          .setSystemLogin("system")
          .setSystemPasscode("manager")
          .setSystemHeartbeatSendInterval(10000)
          .setSystemHeartbeatReceiveInterval(10000);
}
```

ActiveMQ setup:

```bash
# Download and run ActiveMQ
# ActiveMQ STOMP is enabled by default on port 61613
```

## Advanced: Direct Broker Publishing

You can publish directly to the broker from non-WebSocket code:

```java
@Service
public class ExternalPublisherService {

    @Autowired
    private RabbitTemplate rabbitTemplate;  // If using RabbitMQ

    public void publishToTopic(String topic, Object message) {
        rabbitTemplate.convertAndSend("/exchange/amq.topic", topic, message);
    }
}
```

Or use STOMP directly from other services:

```java
@Service
public class StompPublisherService {

    public void publishViaStomp(String destination, Object payload) {
        // Create STOMP client connection
        // Send message directly to broker
        // Close connection
    }
}
```

## Performance Tuning for Broker Relay

Tune connection pooling and timeouts:

```java
@Override
public void configureMessageBroker(MessageBrokerRegistry config) {
    config.enableStompBrokerRelay("/topic", "/queue")
          // ... connection settings ...
          .setSystemHeartbeatSendInterval(5000)      // More frequent heartbeats
          .setSystemHeartbeatReceiveInterval(5000)
          .setClientHeartbeatReceiveInterval(20000)  // Client timeout
          .setClientHeartbeatSendInterval(10000);
}
```

## Complete Migration Checklist

When migrating from simple broker to relay:

- [ ] Install and configure external broker (RabbitMQ/ActiveMQ)
- [ ] Update `WebSocketConfig` to use `enableStompBrokerRelay()`
- [ ] Configure broker credentials and connection details
- [ ] Test connection with single instance
- [ ] Deploy multiple instances
- [ ] Configure load balancer
- [ ] Test cross-instance message delivery
- [ ] Set up monitoring and alerts
- [ ] Configure backup/redundancy for broker
- [ ] Document connection strings for operations team

## Complete Test Client

Test that messages flow through the broker relay:

```html
<!DOCTYPE html>
<html>
<head>
    <title>Broker Relay Test</title>
    <script src="https://cdn.jsdelivr.net/npm/sockjs-client@1/dist/sockjs.min.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/@stomp/stompjs@7/bundles/stomp.umd.min.js"></script>
</head>
<body>
    <h1>Broker Relay Test</h1>
    <button onclick="connect()">Connect</button>
    <button onclick="send()">Send Test Message</button>
    <div id="output"></div>

    <script>
        let client = null;

        function connect() {
            const socket = new SockJS('http://localhost:8080/gs-guide-websocket');
            client = StompJs.Stomp.over(socket);
            
            client.connect({}, () => {
                log('✅ Connected via broker relay');
                
                client.subscribe('/topic/test-response', (msg) => {
                    log('📨 Received: ' + msg.body);
                });
            }, (error) => {
                log('❌ Connection failed: ' + error);
            });
        }

        function send() {
            if (client) {
                client.send('/app/test.broker', {}, 'Hello from broker relay!');
                log('📤 Sent test message');
            }
        }

        function log(msg) {
            const output = document.getElementById('output');
            output.innerHTML += '<p>' + new Date().toLocaleTimeString() + ' - ' + msg + '</p>';
        }
    </script>
</body>
</html>
```

## What We've Accomplished

In this final article, we've:
1. Replaced the simple in-memory broker with an external broker relay
2. Configured RabbitMQ as our STOMP broker
3. Set up production-ready configuration with environment variables
4. Understood how broker relay enables horizontal scaling
5. Added monitoring and health checks
6. Learned to scale across multiple application instances

## Series Summary

Over these 10 articles, we've covered:

1. **Foundation**: Basic STOMP endpoint setup
2. **Server Push**: Real-time server-to-client updates
3. **Two-Way Communication**: Complete chat application
4. **Compatibility**: SockJS fallbacks for cross-browser support
5. **Initial Data**: @SubscribeMapping for startup optimization
6. **Security**: Authentication and user identity
7. **Private Messaging**: User destinations for targeted delivery
8. **High Frequency**: Optimizing for cursor tracking and similar use cases
9. **Performance**: Tuning channels and monitoring
10. **Enterprise Scale**: External broker relay for millions of connections

You now have a comprehensive understanding of building production-ready, scalable real-time applications with Spring Boot and STOMP. These patterns and techniques power some of the world's largest real-time web applications.

Thank you for following this series, and happy building!
