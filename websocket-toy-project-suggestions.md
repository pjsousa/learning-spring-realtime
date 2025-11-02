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

---

# 10 Casual Game & Puzzle Projects
## Collaborative and Competitive WebSocket Games

These projects focus on simple, engaging games and puzzles that leverage WebSocket for real-time multiplayer experiences. Each can be implemented with Spring Boot + STOMP and provides an interactive, fun way to learn WebSocket patterns.

---

## **Simple Game Projects** (Beginner-Friendly)

### 11. **Collaborative Drawing & Guessing Game**
**Title:** "Draw This: Building a Pictionary-Style Game with WebSocket"

**Overview:**
A classic party game where one player draws (or describes) a word, and others guess in real time. The first correct guess wins points. Players take turns drawing. Perfect for demonstrating turn-based state management and event-driven gameplay.

**Mode:** Competitive (within collaborative rounds)

**Key Concepts:**
- Turn rotation and role assignment (drawer vs guessers)
- Broadcasting drawing strokes to all players
- Real-time guess submission and validation
- Score tracking and round management
- Timer synchronization for drawing rounds

**Implementation Focus:**
- Game room state (current drawer, word, timer, scores)
- Drawing event broadcasting (similar to whiteboard)
- Guess validation and winner detection
- Using `@Scheduled` for round timers
- Client-side drawing canvas or text description interface

---

### 12. **Word Scramble Race**
**Title:** "Unscramble the Words: Real-Time Competitive Word Puzzle"

**Overview:**
Players receive the same scrambled word simultaneously and race to be the first to unscramble it. Points awarded for speed. Multiple rounds with increasing difficulty. Shows real-time competition with server-side validation.

**Mode:** Competitive

**Key Concepts:**
- Broadcasting scrambled words to all players at once
- Fastest-response detection and validation
- Score tracking with time bonuses
- Round progression and difficulty levels
- Leaderboard updates in real time

**Implementation Focus:**
- Word scrambling algorithm and word list management
- Answer validation and timing logic
- Broadcasting winners and score updates
- Client-side timer display and input handling
- Using `@SendTo` for round start signals

---

### 13. **Collaborative Memory Matching Game**
**Title:** "Find the Pairs: Multi-Player Memory Game Collaboration"

**Overview:**
Multiple players work together to flip cards and find matching pairs. All players see the same board state. When someone flips a card, everyone sees it. First pair to find all matches wins as a team. Alternatively, competitive mode where players compete to find the most pairs.

**Mode:** Collaborative (with optional competitive scoring)

**Key Concepts:**
- Card state synchronization (flipped, matched, hidden)
- Broadcasting card flip events
- Match detection and pair tracking
- Game completion detection
- Board state sync on player join

**Implementation Focus:**
- Card grid generation and shuffling
- Flip event DTOs and state management
- Match detection logic
- Using `@SubscribeMapping` to send board state to new players
- Client-side card grid UI with flip animations

---

### 14. **Quick Math Challenge**
**Title:** "Math Race: Real-Time Arithmetic Competition"

**Overview:**
Players see math problems (e.g., "12 + 7 = ?") and race to submit the correct answer first. Points awarded based on speed and difficulty. Server generates problems and validates answers in real time.

**Mode:** Competitive

**Key Concepts:**
- Server-generated problems with configurable difficulty
- Answer submission and validation
- Leaderboard with response times
- Round-based progression (fixed time or problems)

**Implementation Focus:**
- Problem generation (addition, subtraction, multiplication, division)
- Answer validation and timing
- Broadcasting problems via `SimpMessagingTemplate`
- Score calculation (points for speed + difficulty)
- Client-side problem display and input form

---

### 15. **Story Chain Game**
**Title:** "Building Stories Together: Collaborative Storytelling Game"

**Overview:**
Players take turns adding one sentence to a shared story. Each player sees the story so far and adds the next line. The story evolves collaboratively. Optional: time limits per turn, story themes, or word count constraints.

**Mode:** Collaborative

**Key Concepts:**
- Turn rotation with time tracking
- Story state accumulation and broadcasting
- Sentence validation (length, content rules)
- Story history and replay

**Implementation Focus:**
- Turn management and session-to-turn mapping
- Story DTO with appended sentences
- Broadcasting story updates on each turn
- Client-side story display with turn indicators
- Optional: story export/sharing functionality

---

## **Intermediate Game Projects** (Moderate Complexity)

### 16. **Typing Speed Race**
**Title:** "Type Faster: Real-Time Typing Competition"

**Overview:**
Players receive the same text passage and race to type it correctly. Progress bars and typing speeds update in real time for all participants. First to finish wins, with speed/accuracy scoring.

**Mode:** Competitive

**Key Concepts:**
- Text passage distribution to all players
- Real-time progress updates (characters typed, WPM)
- Typing accuracy validation
- Leaderboard with speed and accuracy metrics
- Timer synchronization

**Implementation Focus:**
- Passage selection and broadcasting
- Progress tracking (typed characters, mistakes)
- WPM (words per minute) calculation
- Broadcasting leaderboard updates via `/topic/leaderboard`
- Client-side typing area with live progress visualization

---

### 17. **Live Trivia Quiz**
**Title:** "Know It All: Building a Real-Time Trivia Game"

**Overview:**
A quiz where questions appear one at a time, and players submit answers. Results update in real time after each question. Shows correct answers, score updates, and leaderboard. Optional: multiple choice or open-ended questions.

**Mode:** Competitive

**Key Concepts:**
- Question distribution with timer
- Answer collection and validation
- Result broadcasting (correct answer, player scores)
- Round progression and final scoring
- Question bank management

**Implementation Focus:**
- Question DTOs with answers and options
- Timer management for question display
- Answer aggregation and validation
- Score calculation and leaderboard updates
- Client-side quiz UI with question/answer forms

---

### 18. **Collaborative Escape Room Puzzle**
**Title:** "Escape Together: Multi-Player Puzzle Solving"

**Overview:**
Players work together to solve a series of puzzles (clues, riddles, pattern matching). Each player can contribute clues or solutions. Progress is shared in real time. Team wins when all puzzles are solved.

**Mode:** Collaborative

**Key Concepts:**
- Puzzle state management (solved, in-progress, locked)
- Clue/solution sharing between players
- Progress tracking and puzzle unlocking
- Hint system (optional server-provided hints)
- Team completion detection

**Implementation Focus:**
- Puzzle DTOs with solutions and hints
- State synchronization across players
- Solution validation logic
- Progress broadcasting via `/topic/puzzles/progress`
- Client-side puzzle UI with clue display and input

---

### 19. **Number Guessing Game (Hot/Cold)**
**Title:** "Hot or Cold: Real-Time Number Guessing with Feedback"

**Overview:**
One player picks a number, others guess. Server responds with "hot" (close) or "cold" (far) feedback. All players see guesses and feedback in real time. First to guess correctly wins. Alternatively: collaborative mode where players share hints.

**Mode:** Competitive (or collaborative with shared hints)

**Key Concepts:**
- Secret number storage (game state)
- Guess validation and distance calculation
- Hot/cold feedback generation
- Broadcasting guesses and feedback to all players
- Win condition detection

**Implementation Focus:**
- Number range configuration
- Distance calculation algorithm
- Feedback message generation
- Game state (target number, current guesses)
- Client-side guess input and feedback display

---

### 20. **Word Association Chain**
**Title:** "Connect the Words: Collaborative Word Association Game"

**Overview:**
Players take turns adding words that associate with the previous word. Each word must relate to the last one. The chain grows collaboratively. Optional: time limits, word restrictions (no repeats, must start with last letter), or competitive scoring (longest chain wins).

**Mode:** Collaborative (with optional competitive elements)

**Key Concepts:**
- Turn-based word submission
- Word validation (association rules, restrictions)
- Chain state accumulation
- Turn rotation and timing
- Chain history and replay

**Implementation Focus:**
- Word chain DTO and state management
- Association validation (can be simple or use APIs)
- Turn management and time tracking
- Broadcasting chain updates
- Client-side chain visualization with player attribution

---

## **Suggested Order by Complexity for Game Projects**

1. **Number Guessing Game** (Simplest - basic guess/feedback loop)
2. **Quick Math Challenge** (Simple problem generation and validation)
3. **Collaborative Memory Matching** (Card state synchronization)
4. **Word Scramble Race** (Fast validation and timing)
5. **Story Chain Game** (Turn-based collaboration)
6. **Typing Speed Race** (Real-time progress tracking)
7. **Live Trivia Quiz** (Question flow and scoring)
8. **Word Association Chain** (Association validation)
9. **Collaborative Drawing & Guessing** (Combines drawing + guessing mechanics)
10. **Collaborative Escape Room** (Most complex - multiple puzzles and state management)

---

## **Game-Specific Implementation Patterns**

### **Turn Management**
- Session-to-player mapping
- Turn rotation logic with timeouts
- Handling player disconnects during turns

### **Scoring Systems**
- Real-time score updates via `/topic/scores`
- Leaderboard aggregation and sorting
- Point calculation based on speed, accuracy, or difficulty

### **Game State Synchronization**
- Initial state via `@SubscribeMapping` for late joiners
- State updates broadcasted after each action
- Game completion detection and cleanup

### **Timer Synchronization**
- Server-side timers using `@Scheduled`
- Broadcasting time remaining to all players
- Handling timer expiration and round transitions

### **Validation Patterns**
- Server-side validation for all player actions
- Broadcasting validation results
- Preventing invalid or duplicate submissions

---

## **Testing Game Projects**

Each game should support:
- Multiple browser tabs/windows for multi-player testing
- Graceful handling of player disconnections
- Reconnection with game state recovery (optional)
- Clear visual feedback for all game events
- Immediate feedback for player actions

These game projects provide engaging, interactive ways to learn WebSocket patterns while building something fun and playable.