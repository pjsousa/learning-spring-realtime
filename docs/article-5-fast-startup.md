# STOMP Toy Projects Blog Series Plan

## Series Plan

- Article 1 – Hello STOMP: scaffold Spring Boot WebSocket project, enable simple broker, deliver minimal HTML client.
- Article 2 – Broadcasting Messages: add chat-style `@MessageMapping`, introduce topics, build multi-client HTML demo.
- Article 3 – User-Specific Queues: explore `convertAndSendToUser`, refine client routing, add login-free session identity.
- Article 4 – DTOs and Validation: formalize message payloads, add validation/exception handling, upgrade UI messaging.
- Article 5 – Fast Startup with `@SubscribeMapping`: serve initial data snapshot on subscription, wire request/reply client flow.
- Article 6 – Persistence Layer: back protocol with JPA/H2, preload fixtures, sync client on data changes.
- Article 7 – Stateful Workflows: manage multi-step processes, store per-session context, extend UI for workflow state.
- Article 8 – Error Recovery: configure `@MessageExceptionHandler`, retry logic, client feedback patterns.
- Article 9 – Scaling Out: switch to external broker (RabbitMQ/ActiveMQ), discuss clustering, adjust client settings.
- Article 10 – Packaging & Deployment: bundle assets, secure endpoints, document testing and deployment checklist.

## Article 5 Outline

- **Hook**: articulate startup pain point; why immediate data matters for responsive clients.
- **Recap & Prereqs**: what readers should already have (service skeleton, basic HTML client, `/app` commands).
- **Concept Primer**: contrast `@MessageMapping` vs `@SubscribeMapping`; explain direct reply via `clientOutboundChannel`.
- **Domain Scenario**: define toy app data (e.g., lobby settings/history) that must load once on subscribe.
- **Server Setup**: add Spring component with `@SubscribeMapping` method returning DTO snapshot; highlight serialization.
- **Data Source**: create simple in-memory repository (map/list) providing deterministic initial payload.
- **Client Workflow**: in HTML/JS, connect SockJS/Stomp, subscribe to initialization endpoint, render payload immediately.
- **Follow-on Command**: demonstrate chaining—after initial data, subscribe to live updates topic, send command.
- **Testing the Flow**: run app, open two browser tabs, inspect network frames, verify one-time response vs broadcasts.
- **Troubleshooting**: cover common pitfalls (no subscription prefix, returning void, broker relaying).
- **Wrap-Up & Next Steps**: summarize benefits, tease persistence integration in Article 6.
