# 10 WebSocket Toy Projects for Blog Articles
## From Simple to Advanced

These projects complement your existing STOMP series and showcase diverse real-time use cases. Each can be implemented with Spring Boot + STOMP and a minimalist HTML/JavaScript client.

---

## **Simple Projects** (Beginner-Friendly)

### 1. **Live Visitor Counter**
**Title:** "Counting Connections: Building a Real-Time Visitor Counter with WebSocket"

**Overview:**
A simple project that displays the number of active WebSocket connections in real time. Perfect for demonstrating basic connection tracking and server-to-client broadcasting. When a user opens the page, the counter increments. When they close it, it decrements. Multiple tabs show synchronized updates.

**Key Concepts:**
- `SessionConnectEvent` and `SessionDisconnectEvent` listeners
- Broadcasting connection count changes via `SimpMessagingTemplate`
- Simple client-side subscription to a single topic
- Demonstrates stateless broadcasting patterns

**Implementation Focus:**
- Tracking active sessions using `SimpUserRegistry` or a custom registry
- Broadcasting count updates on connect/disconnect
- Handling edge cases (page refresh, network drops)

---

### 2. **Real-Time Polling/Voting System**
**Title:** "Instant Results: Creating a Live Voting System with STOMP"

**Overview:**
Users vote on a question (e.g., "Best programming language?"), and results update in real time as votes come in. Shows aggregating data from multiple clients and broadcasting aggregated results.

**Key Concepts:**
- Collecting votes via `@MessageMapping`
- Server-side vote aggregation and storage
- Broadcasting updated results to all subscribers
- Preventing duplicate votes (optional: session-based tracking)

**Implementation Focus:**
- Vote DTOs and result aggregation logic
- Using `@SendTo` for broadcasting poll results
- Client-side visualization of vote percentages (progress bars, charts)

---

### 3. **Live Notification Center**
**Title:** "Stay Informed: Building a Real-Time Notification System"

**Overview:**
A notification hub where the server sends notifications (e.g., "New message", "System update") to connected clients. Demonstrates one-to-many messaging with optional priority levels and notification grouping.

**Key Concepts:**
- Server-originated notifications via scheduled tasks or events
- Notification DTOs with types (INFO, WARNING, ERROR)
- Client-side notification stacking and auto-dismissal
- Optional: user-specific notifications using `/user/queue/notifications`

**Implementation Focus:**
- `SimpMessagingTemplate` for broadcasting notifications
- Notification history and replay for reconnecting clients
- Client-side toast/notification UI components

---

## **Intermediate Projects** (Moderate Complexity)

### 4. **Collaborative Whiteboard**
**Title:** "Drawing Together: Building a Real-Time Collaborative Canvas"

**Overview:**
Multiple users draw on a shared canvas simultaneously. Each stroke is broadcast to all participants, creating a Google Jamboard-like experience. Great for demonstrating coordinate synchronization and handling high-frequency updates efficiently.

**Key Concepts:**
- Broadcasting mouse/pointer coordinates and drawing events
- Efficient message batching for smooth drawing
- Canvas state synchronization on join
- Handling different drawing tools (pen, shapes, colors)

**Implementation Focus:**
- Drawing event DTOs (start point, end point, tool, color)
- Throttling client-side updates to prevent message flooding
- Reconstructing canvas state for late joiners using `@SubscribeMapping`
- Client-side Canvas API integration

---

### 5. **Real-Time Multiplayer Game (Tic-Tac-Toe)**
**Title:** "Game On: Creating a Multiplayer Game with WebSocket State Sync"

**Overview:**
A two-player Tic-Tac-Toe game where moves are synchronized in real time. Players take turns, and both see the board update instantly. Introduces game state management and turn-based logic.

**Key Concepts:**
- Game state management (board state, current player, game status)
- Move validation and broadcast
- Session pairing (matchmaking two players)
- Win condition checking and broadcasting game results

**Implementation Focus:**
- Game room abstraction (session-to-room mapping)
- `@MessageMapping` for move submission
- Broadcasting board updates to room participants
- Client-side game logic and UI updates

---

### 6. **Live Auction/Bidding System**
**Title:** "Going Once, Going Twice: Real-Time Bidding with WebSocket"

**Overview:**
An auction platform where users place bids on items. The current highest bid and bidder update in real time for all participants. Shows time-sensitive updates and bid validation.

**Key Concepts:**
- Bid validation (must exceed current bid)
- Broadcasting highest bid updates
- Timer/countdown synchronization
- Bid history tracking

**Implementation Focus:**
- Auction state management (current bid, bidder, time remaining)
- Using `@Scheduled` for countdown updates
- Bid submission and validation logic
- Client-side bid form and real-time price display

---

### 7. **Collaborative Todo List**
**Title:** "Team Tasks: Real-Time Collaboration on Shared Lists"

**Overview:**
Multiple users can add, complete, and delete todos from a shared list. All changes appear instantly for everyone. Demonstrates CRUD operations over WebSocket and optimistic UI updates.

**Key Concepts:**
- Todo CRUD operations (Create, Read, Update, Delete)
- Synchronizing list state across clients
- Conflict resolution (handling simultaneous edits)
- User attribution (who added/completed what)

**Implementation Focus:**
- Todo DTOs and state management
- Broadcasting todos on create/update/delete
- Using `@SubscribeMapping` to send full list on connection
- Client-side list manipulation and conflict handling

---

## **Advanced Projects** (Complex Real-World Patterns)

### 8. **Real-Time Code Collaboration (Simple Pair Programming)**
**Title:** "Pair Programming Over WebSocket: Building a Collaborative Code Editor"

**Overview:**
Multiple users edit code simultaneously, with each user's cursor position and changes synced in real time. Like Google Docs for code. Requires sophisticated text synchronization and operational transform concepts (or simplified CRDTs).

**Key Concepts:**
- Text synchronization (insert, delete, replace operations)
- Cursor position tracking and broadcasting
- Handling concurrent edits without conflicts
- User presence (who's editing where)

**Implementation Focus:**
- Text operation DTOs (position, content, operation type)
- Server-side text state management or integration with existing libraries
- Client-side CodeMirror/Monaco editor integration
- Operational Transform basics or simplified conflict resolution

---

### 9. **Live System Monitoring Dashboard**
**Title:** "Watching the System: Real-Time Metrics Dashboard with WebSocket Aggregation"

**Overview:**
A dashboard displaying system metrics (CPU, memory, network) updated in real time. Multiple dashboard views can subscribe to different metric streams. Shows aggregating data from multiple sources and selective subscriptions.

**Key Concepts:**
- Metric collection via `@Scheduled` tasks or actuator endpoints
- Topic-based subscriptions (`/topic/metrics/cpu`, `/topic/metrics/memory`)
- Metric aggregation and broadcasting
- Client-side charting libraries (Chart.js, D3.js)

**Implementation Focus:**
- Metric DTOs and periodic publishing
- Selective subscription patterns
- Dashboard UI with multiple chart types
- Performance considerations for high-frequency metric updates

---

### 10. **Distributed Real-Time System (Microservices + WebSocket Gateway)**
**Title:** "Across Services: Building a Distributed Real-Time Architecture"

**Overview:**
An advanced project where multiple Spring Boot services communicate via WebSocket. One service aggregates events from others and broadcasts to clients. Demonstrates microservices patterns with WebSocket, service-to-service messaging, and WebSocket as a gateway pattern.

**Key Concepts:**
- WebSocket gateway service aggregating events from backend services
- Service-to-service communication (HTTP, RabbitMQ, or direct WebSocket connections)
- Event aggregation and transformation
- Scaling WebSocket connections across services

**Implementation Focus:**
- Gateway service pattern
- Backend services publishing events
- Message routing and transformation
- Client connection management across distributed services
- Integration with external message brokers (RabbitMQ, Kafka) for service communication

---

## **Suggested Order by Complexity**

1. **Live Visitor Counter** (Simplest - introduces connection tracking)
2. **Real-Time Polling System** (Simple aggregation)
3. **Live Notification Center** (One-way server push with structure)
4. **Collaborative Todo List** (Two-way CRUD operations)
5. **Real-Time Multiplayer Game** (State management and validation)
6. **Live Auction System** (Time-sensitive updates)
7. **Collaborative Whiteboard** (High-frequency coordinate sync)
8. **Real-Time Code Collaboration** (Complex text synchronization)
9. **Live Monitoring Dashboard** (Data aggregation and visualization)
10. **Distributed Real-Time System** (Most complex - architecture patterns)

---

## **Cross-Cutting Themes to Cover**

- **State Management:** Server-side vs client-side state
- **Conflict Resolution:** Handling concurrent updates
- **Performance:** Message throttling and batching
- **Reliability:** Reconnection strategies and state recovery
- **Scalability:** Patterns for handling many concurrent connections
- **Testing:** Integration testing for WebSocket flows
- **Security:** Authorization and validation in real-time contexts

---

## **Implementation Notes**

Each article should include:
- Spring Boot project setup with WebSocket dependencies
- STOMP configuration (`@EnableWebSocketMessageBroker`)
- Server-side message handlers and business logic
- Complete HTML/JavaScript client (standalone, no frameworks required)
- Instructions for testing (multiple browser tabs/windows)
- Optional enhancements for readers to try

These projects complement your existing series by exploring different domains and patterns while building on the same foundational STOMP concepts.