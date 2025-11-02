# Scaling to Millions: Implementing the STOMP Broker Relay for Enterprise Applications

Our toy project has evolved from a simple ping-pong demo into a feature-rich application with authentication, private messaging, and high-frequency cursor updates. The simple in-memory broker served us well during development, but it has limits: it runs in a single JVM, lacks persistence, and cannot span multiple application instances. For enterprise deployments that demand horizontal scalability and reliability, Spring’s STOMP broker relay bridges the gap by forwarding messages to an external broker—typically RabbitMQ or ActiveMQ—that can handle millions of destinations and clients.

In this final article we will reconfigure our Spring Boot application to use a STOMP broker relay backed by RabbitMQ. We will explore the architectural implications, deployment considerations, and testing strategies for large-scale messaging. Even if your production environment uses a different broker (ActiveMQ Artemis, Apollo, or a cloud-managed service), the principles remain the same.

By the end of this article you will:

- Understand how the broker relay differs from the simple broker.
- Spin up RabbitMQ locally with STOMP support (Docker or manual installation).
- Configure Spring Boot to connect to the broker relay using credentials and virtual hosts.
- Map application destinations, user queues, and heartbeat settings for optimal performance.
- Verify end-to-end messaging across multiple application instances.
- Consider clustering, failover, and monitoring practices for enterprise deployments.

Let’s take our STOMP application to production scale.

## Broker Relay vs. Simple Broker

The simple broker is an in-memory implementation within Spring that handles subscriptions and broadcasts. It is easy to set up but limited to a single JVM. The broker relay, on the other hand, forwards STOMP frames to an external message broker that implements the STOMP protocol. The broker handles routing, persistence (if configured), and distribution across clustered nodes. Spring’s role becomes that of a relay: it accepts STOMP frames, adapts them, and sends them over TCP to the broker.

Key advantages of the broker relay:

- **Horizontal scaling:** Multiple Spring Boot instances can connect to the same broker, allowing clients connected to different application servers to exchange messages seamlessly.
- **Performance:** Brokers like RabbitMQ are optimized for high-throughput messaging with durable queues and efficient routing algorithms.
- **Persistence:** Configure durable queues or topics so messages survive application restarts or client disconnects.
- **Advanced features:** Use broker-specific features like dead-lettering, TTLs, and routing keys.

The trade-off is increased infrastructure complexity: you must provision, secure, and monitor the external broker.

## Setting Up RabbitMQ with STOMP

We will use RabbitMQ, which supports STOMP via a plugin. The broker listens on port 61613 by default.

### Using Docker

Run the following command to start RabbitMQ with the STOMP plugin enabled:

```bash
docker run -d --name rabbitmq-stomp \
  -p 5672:5672 \
  -p 15672:15672 \
  -p 61613:61613 \
  rabbitmq:3.13-management
```

This container exposes AMQP (5672), the management UI (15672), and STOMP (61613). The management console is available at `http://localhost:15672` (default credentials: `guest/guest`). The STOMP plugin is enabled by default in the management image.

### Verifying STOMP Connectivity

Use `openssl` or `telnet` to check the STOMP port:

```bash
openssl s_client -connect localhost:61613
```

You should see a successful TCP connection. Press `Ctrl+C` to exit.

## Configuring Spring Boot for the Broker Relay

Replace the simple broker configuration in `WebSocketConfig` with the broker relay:

```java
@Override
public void configureMessageBroker(MessageBrokerRegistry registry) {
    registry.setApplicationDestinationPrefixes("/app");
    registry.setUserDestinationPrefix("/user");

    registry.enableStompBrokerRelay("/topic", "/queue")
            .setRelayHost("localhost")
            .setRelayPort(61613)
            .setClientLogin("guest")
            .setClientPasscode("guest")
            .setSystemLogin("guest")
            .setSystemPasscode("guest")
            .setVirtualHost("/")
            .setUserRegistryBroadcast("/topic/registry")
            .setUserDestinationBroadcast("/topic/unresolved-user-destination");
}
```

Explanation:

- `enableStompBrokerRelay` registers the relay with destination prefixes `/topic` and `/queue`. Clients continue to subscribe using the same destinations.
- `setRelayHost` and `setRelayPort` point to the RabbitMQ STOMP endpoint.
- `clientLogin`/`clientPasscode` are the credentials the application uses when forwarding client frames. `systemLogin`/`systemPasscode` are used for internal broker communications (e.g., broadcasting registry updates). For production, create dedicated credentials instead of using the default `guest`.
- `setVirtualHost` selects the RabbitMQ virtual host.
- `setUserRegistryBroadcast` and `setUserDestinationBroadcast` enable cluster-wide awareness of user sessions. The relay uses these destinations to synchronize subscriptions across Spring Boot instances.

Remove any simple broker executor configuration—RabbitMQ handles distribution now. The channel executor tuning from Article 9 still applies to the inbound/outbound channels within each application instance.

## Handling Heartbeats and Network Stability

External brokers rely on heartbeats to detect dead connections. Configure matching heartbeat intervals on both the application and client:

```java
registry.enableStompBrokerRelay("/topic", "/queue")
        .setSystemHeartbeatSendInterval(10_000)
        .setSystemHeartbeatReceiveInterval(10_000);
```

On the client side, set `heartbeatIncoming`/`heartbeatOutgoing` to 10,000 milliseconds to stay aligned.

If the network is unreliable, increase the intervals or enable automatic reconnect in the client (`reconnectDelay` already covers this). Watch Spring logs for warnings like `STOMP broker relay reconnect attempt failed`—they indicate connectivity issues.

## Creating Dedicated Broker Users

In RabbitMQ, create two users: one for clients (`stomp-app`) and one for the system connection (`stomp-system`). Assign them to a virtual host for the application, and restrict permissions to only the necessary exchanges and queues.

```bash
docker exec -it rabbitmq-stomp rabbitmqctl add_vhost /stomp-demo
docker exec -it rabbitmq-stomp rabbitmqctl add_user stomp-app supersecret
docker exec -it rabbitmq-stomp rabbitmqctl set_permissions -p /stomp-demo stomp-app ".*" ".*" ".*"
docker exec -it rabbitmq-stomp rabbitmqctl add_user stomp-system supersecret
docker exec -it rabbitmq-stomp rabbitmqctl set_permissions -p /stomp-demo stomp-system ".*" ".*" ".*"
```

Update Spring’s configuration accordingly:

```java
registry.enableStompBrokerRelay("/topic", "/queue")
        .setVirtualHost("/stomp-demo")
        .setClientLogin("stomp-app")
        .setClientPasscode("supersecret")
        .setSystemLogin("stomp-system")
        .setSystemPasscode("supersecret");
```

## Testing Across Multiple Application Instances

Start two instances of the Spring Boot application (different ports) and connect clients to each. Because both instances share the same broker, messages should route between them seamlessly. Test the following scenarios:

- A user connected to instance A sends a chat message; a user on instance B receives it instantly.
- Private messaging and user destinations continue to work because the broker relay handles per-user queues.
- Cursor updates broadcast across instances without duplication.

Check RabbitMQ’s management UI to observe STOMP connections, queues, and message rates. The `stomp-subscription` queues correspond to individual user sessions.

## Persisting Messages and Durable Subscriptions

RabbitMQ supports durable queues and persistent messages. To enable them for STOMP:

- Use `/queue/` destinations for point-to-point messaging; RabbitMQ creates durable queues by default if you prefix the destination with `/queue/` and the client subscribes with header `durable:true`.
- For topics (`/topic/`), RabbitMQ uses transient exchanges. If you require persistence for broadcast messages, consider using AMQP directly or configure RabbitMQ to delegate to durable exchanges via policy.

Remember that STOMP is a text protocol layered on top of AMQP features. Review the broker’s STOMP plug-in documentation for exact behaviors.

## Security Considerations

When deploying the broker relay in production:

- **Use TLS:** Configure RabbitMQ’s STOMP listener with TLS certificates. Update Spring’s relay configuration with `setRelayPort(61614)` and enable TLS using a `TcpClient` customization (available via `setTcpClient` in Spring 6).
- **Firewall rules:** Allow STOMP connections only from your application servers, not from end-user devices.
- **Authentication:** Use strong credentials and limit permissions to the necessary virtual host.
- **Rate limiting:** RabbitMQ can enforce per-connection or per-vhost limits to protect against abuse.

## Monitoring and Alerting

Leverage RabbitMQ’s metrics or Prometheus exporter to monitor:

- Connection count and per-connection message rate.
- Queue depths and consumer counts.
- Message redeliveries and dead-letter queues.
- Heartbeat drops and connection churn.

Integrate these metrics with your existing observability stack. Combining broker metrics with the application-level statistics from Article 9 provides a complete view of the system.

## Failure Handling and Reconnection Strategy

When the broker becomes unavailable, Spring attempts to reconnect automatically. During outages, clients receive error frames or experience connection drops. Implement client-side retry logic (already covered via `reconnectDelay`). On the server, monitor `ApplicationEvent`s such as `BrokerAvailabilityEvent` to adjust behavior:

```java
@Component
public class BrokerAvailabilityListener {

    private static final Logger log = LoggerFactory.getLogger(BrokerAvailabilityListener.class);

    @EventListener
    public void onBrokerEvent(BrokerAvailabilityEvent event) {
        log.warn("Broker availability changed: {}", event.isBrokerAvailable());
    }
}
```

You might disable certain features or queue messages locally until the broker returns.

## Deployment Topology

For robust production deployments:

- Run RabbitMQ as a clustered service (3+ nodes) with mirrored queues for high availability. RabbitMQ’s quorum queues or classic mirrored queues provide resilience against node failures.
- Deploy multiple Spring Boot instances behind a load balancer. They all connect to the broker using the relay configuration.
- Ensure sticky sessions at the HTTP load balancer are not required because STOMP connections are stateful; the WebSocket handshake connection stays with the chosen application instance, but any instance can publish/subscribe through the broker.

## Migration Strategy from Simple Broker

Transitioning from a simple broker to a relay can be gradual:

1. **Abstract destinations:** Keep destination prefixes (`/topic`, `/queue`, `/user`) consistent so clients require no change.
2. **Deploy RabbitMQ:** Run the broker in staging and reconfigure the application to use `enableStompBrokerRelay`.
3. **Smoke test:** Verify all features (chat, private messaging, cursor updates) work in staging.
4. **Load test:** Repeat the load tests from Article 9 with the external broker.
5. **Roll out gradually:** Deploy the new configuration to production, monitoring broker metrics closely.

If issues arise, you can temporarily revert to the simple broker by switching the configuration, though messages will no longer cross application instances.

## Conclusion and Series Wrap-Up

Congratulations! You have completed the STOMP toy project series, evolving from a basic Spring Boot WebSocket endpoint to an enterprise-ready real-time application backed by an external broker. Along the way you learned how to:

- Configure Spring WebSocket/STOMP endpoints and build minimal HTML clients.
- Broadcast server-side updates, implement chat, integrate SockJS fallbacks, and deliver bootstrap data via `@SubscribeMapping`.
- Enforce security, personalize messaging with user destinations, and stream high-frequency state updates.
- Tune messaging performance, monitor internal channels, and scale horizontally with a broker relay.

The techniques you now possess apply to many domains: collaborative tools, live dashboards, multiplayer games, IoT control panels, and more. Keep experimenting—add persistence, integrate with message-driven microservices, or package the front-end in a modern framework.

### Next Steps

- Explore RabbitMQ’s advanced features: topic exchanges, routing keys, and dead-letter queues. Map STOMP destinations to those constructs carefully.
- Evaluate other brokers, such as ActiveMQ Artemis, Apache Apollo, or cloud services like AWS MQ and Google Pub/Sub with STOMP gateways.
- Investigate protocol alternatives like MQTT or RSocket if your use case demands binary transports or reactive streams.
- Share your learnings: write blog posts, present talks, or contribute sample projects to the community.

Thank you for following this 10-part journey. Happy coding, and may your real-time applications scale smoothly! 🎉

