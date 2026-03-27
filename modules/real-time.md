# Module: Real-Time Communication

Patterns for push-based and bidirectional communication between clients and servers. Real-time systems are architecturally different from request/response: connections are long-lived, state must be managed across disconnections, and message ordering matters.

**Prerequisite knowledge:** [03-architecture](../03-architecture.md) (event-driven patterns, async, background jobs), [07-error-handling](../07-error-handling.md) (retries, graceful degradation), [08-performance](../08-performance.md) (connection management, memory).

---

## Transport Decision Framework

| Transport | Direction | Complexity | Browser support | Reconnection | Best for |
|-----------|-----------|------------|-----------------|--------------|----------|
| **Short polling** | Client pulls | Low | Universal | N/A (new request each time) | Simple status checks, low-frequency updates |
| **Long polling** | Client pulls, server holds | Medium | Universal | Client retries | Moderate update frequency, no WebSocket support |
| **Server-Sent Events (SSE)** | Server pushes | Low | All modern browsers | Built-in (auto-reconnect) | Server-to-client streams, progress updates |
| **WebSocket** | Bidirectional | High | All modern browsers | Manual (must implement) | Chat, collaboration, gaming, bidirectional data |

### How to Choose

- **Do you only need server-to-client updates?** Use SSE. It's simpler than WebSocket and has built-in reconnection.
- **Do you need client-to-server real-time messages too?** Use WebSocket.
- **Is update frequency low (every 30+ seconds)?** Short polling may be sufficient and dramatically simpler.
- **Do you need to support very old browsers or restrictive proxies?** Long polling works everywhere.

**Default recommendation:** Start with SSE for server-push use cases. Only upgrade to WebSocket when you need bidirectional communication. Short polling is perfectly acceptable for low-frequency updates.

---

## Connection Lifecycle

All long-lived connections share common lifecycle concerns.

### Establishing a Connection

- **Authentication:** Validate the user before accepting the connection. For WebSocket, authenticate during the HTTP upgrade handshake (via cookie or token in query string). For SSE, the initial HTTP request carries normal auth headers.
- **Connection limits:** Set a maximum number of concurrent connections per user and globally. Reject new connections beyond the limit with a clear error.
- **Connection registry:** Maintain a server-side map of active connections (user ID to connection object). This enables targeted messaging ("send this update to user X").

### Heartbeats and Keepalive

Long-lived connections can silently die (network changes, NAT timeouts, server-side GC).

- **Server heartbeat:** Send a lightweight ping message every 15-30 seconds. If the client doesn't respond within a timeout (e.g., 2 missed pings), consider the connection dead and clean up.
- **Client heartbeat:** The client sends periodic pings. If no pong is received, reconnect.
- **Why both sides:** Network failures can be asymmetric. The server may think the connection is alive while the client has already lost it, or vice versa.

### Detecting Disconnection

- **Clean disconnect:** Client sends a close frame (WebSocket) or the request ends (SSE). Server cleans up immediately.
- **Unclean disconnect:** Network drops, browser crashes, device loses connectivity. Detected by heartbeat timeout.
- **Cleanup on disconnect:** Remove the connection from the registry, update presence status, release any resources held for this connection.

### Reconnection with Backoff

When a connection drops, the client should reconnect automatically.

- Start with a short delay (1 second)
- Increase exponentially on repeated failures (1s, 2s, 4s, 8s, up to 30-60s max)
- Add random jitter to prevent thundering herd (all clients reconnecting simultaneously after a server restart)
- On successful reconnect, reset the backoff to the initial delay
- SSE has reconnection built-in via the `retry` field. WebSocket requires manual implementation.

### Connection State Management

Track the connection state on the client:

```
DISCONNECTED → CONNECTING → CONNECTED → DISCONNECTED
                   ↑                         |
                   └─────── RECONNECTING ←───┘
```

- Show connection status to the user (subtle indicator: green dot for connected, yellow for reconnecting, red for disconnected)
- Buffer user actions while reconnecting (don't lose typed messages)
- Replay buffered actions on reconnect (see Offline-First section)

---

## Server-Sent Events (SSE)

### When to Use

SSE is ideal when the server needs to push updates to the client, but the client doesn't need to send messages back over the same connection. Common use cases: live feeds, progress updates, notifications, dashboard metrics.

### Implementation Patterns

**Server side:**
- Set response headers: `Content-Type: text/event-stream`, `Cache-Control: no-cache`, `Connection: keep-alive`
- Send events as text lines: `data: {"type": "article_added", "id": 42}\n\n`
- Use named events (`event: notification\ndata: ...\n\n`) to let clients filter by type
- Include an `id` field with each event for resumption after reconnection
- Send a comment line (`: keepalive\n\n`) periodically to prevent proxy timeouts

**Client side:**
- Use `EventSource` API for basic usage (auto-reconnect, event parsing built-in)
- Use `fetch` with `ReadableStream` for more control (custom headers, POST body)
- Handle `onopen`, `onmessage`, and `onerror` events
- On error: EventSource reconnects automatically. Show a "reconnecting" indicator to the user.

### Limitations

- **One direction only:** Client can't send messages over the SSE connection. Use separate HTTP requests for client-to-server communication.
- **Connection limit:** Browsers limit SSE connections per domain (typically 6 for HTTP/1.1). Use HTTP/2 to avoid this.
- **No binary data:** SSE is text-only. Encode binary data as base64 if needed (but consider WebSocket for binary-heavy use cases).
- **Proxy issues:** Some proxies buffer SSE responses. The keepalive comments help, but some environments (corporate proxies) may still break SSE.

### Resumption After Reconnect

SSE supports automatic resumption:
- Include an `id` field with each event
- On reconnect, the browser sends `Last-Event-ID` header
- The server resumes from that event ID, replaying any missed events
- Store recent events server-side (in memory or database) to enable replay

> **CyberPulse example:** SSE is used for streaming LLM generation progress to the frontend. Each chunk of generated text is sent as an SSE event. The client appends text as it arrives, creating a typewriter effect.

---

## WebSocket Patterns

### When to Use

WebSocket is necessary when both client and server need to send messages to each other over a persistent connection. Common use cases: chat, collaborative editing, multiplayer games, real-time dashboards with user interactions.

### Message Protocol Design

Define a structured message format for all WebSocket communication:

```
{
  "type": "chat_message",        // message type (action/event name)
  "payload": { ... },            // message data
  "id": "msg_abc123",           // unique message ID (for dedup, acknowledgment)
  "timestamp": "2024-01-15T..."  // when the message was created
}
```

**Why a protocol:** Raw WebSocket messages are just bytes. Without a protocol, you end up with ad-hoc parsing and type checking scattered everywhere. A consistent message shape makes the system predictable and debuggable.

**In practice:**
- Define all message types in a shared schema (both client and server reference it)
- Validate incoming messages against the schema before processing
- Unknown message types should be logged and ignored (not crash the handler)

### Authentication on Connect

- **Option 1: Token in query string** -- `ws://host/ws?token=xxx`. Simple but tokens appear in server logs and browser history.
- **Option 2: Cookie** -- If the user has a session cookie, it's sent automatically during the WebSocket handshake. Simplest for same-origin web apps.
- **Option 3: First message** -- Connect without auth, then send an authentication message as the first frame. Reject and close the connection if auth fails within a timeout.

**Recommendation:** Cookie for same-origin web apps. Token in query string for cross-origin or mobile clients. Always validate before allowing any other messages.

### Ping/Pong

WebSocket has built-in ping/pong frames at the protocol level:
- Server sends a ping frame periodically (every 30 seconds)
- Client responds with a pong frame automatically (handled by the WebSocket library)
- If no pong is received within a timeout, close the connection

For application-level heartbeats (detecting application-layer freezes, not just network connectivity), implement custom ping/pong messages in your protocol.

---

## Pub/Sub Patterns

### Channels and Rooms

Group connections by topic so messages are delivered only to interested clients.

```
Channels:
  "articles:new"     → clients subscribed to new article notifications
  "dashboard:stats"  → clients viewing the dashboard
  "user:42"          → messages for a specific user
```

**In practice:**
- Maintain a map of channel name to list of connections
- When a client subscribes: add their connection to the channel's list
- When a message is published to a channel: iterate the list and send to each connection
- When a client disconnects: remove from all channels
- Use channel naming conventions (namespace:topic) for organization

### Broadcasting

Sending the same message to many clients simultaneously.

- **Broadcast to all:** System-wide announcements, maintenance notifications
- **Broadcast to channel:** Topic-specific updates (new article in a category)
- **Broadcast to user:** Multi-device notifications (user logged in on phone and laptop)

**Performance consideration:** Broadcasting to thousands of connections is CPU-bound (serializing the message once, then writing to each connection). For very large fan-outs, consider a message queue that distributes the work across multiple server processes.

---

## Presence & Status

### Tracking Who's Online

- When a user connects: mark them as "online" in a shared data structure (database or cache)
- When a user disconnects: mark them as "offline" after a grace period (30-60 seconds, to handle brief reconnections)
- Expose presence data via an API or push it to interested clients

### Activity Detection

- Track last activity timestamp per user (last message sent, last interaction)
- Distinguish between "online" (connected) and "active" (recently interacted)
- Show graduated status: active, idle (connected but inactive for 5+ minutes), offline

### Typing Indicators

- Client sends a "typing" event when the user starts typing
- Client sends a "stopped typing" event after 3-5 seconds of inactivity
- Server broadcasts typing status to other participants in the conversation
- Don't send typing events on every keystroke. Debounce to once per second maximum.

---

## Offline-First & Sync

### Queueing Actions While Offline

When the connection drops, don't discard user actions.

- Queue outgoing messages in a local buffer (in-memory or localStorage)
- Show a visual indicator that the user is offline and actions are queued
- When the connection is restored, replay queued actions in order
- Handle conflicts that may arise from replaying stale actions (see Conflict Resolution below)

### Optimistic UI Updates

Update the UI immediately when the user acts, before the server confirms.

- Show the change locally (e.g., message appears in chat immediately)
- Send the action to the server
- On server confirmation: do nothing (UI already reflects the change)
- On server rejection: revert the local change and show an error

**Why:** Waiting for server confirmation on every action makes the UI feel sluggish. Optimistic updates make the app feel instant. The trade-off is handling the (rare) case where the server rejects the action.

---

## Conflict Resolution

When multiple clients modify the same data, conflicts arise. The strategy depends on your data and use case.

### Last-Write-Wins (LWW)

The most recent update overwrites previous ones.

- Simple to implement (compare timestamps)
- Acceptable when conflicts are rare and data loss from overwrites is tolerable
- Dangerous for collaborative editing (one user's changes silently disappear)

### Operational Transforms (OT)

Transform each user's operations based on other users' concurrent operations.

- Designed for collaborative text editing (Google Docs uses this)
- Complex to implement correctly
- Requires a central server to order operations
- Only consider this for real-time collaborative text editing

### CRDTs (Conflict-Free Replicated Data Types)

Data structures designed so concurrent operations always converge to the same state.

- No central server needed for conflict resolution
- Different CRDT types for different data: counters, sets, registers, sequences
- More constrained than OT (not all operations can be expressed)
- Best for: collaborative applications, offline-first with multi-device sync

### Choosing a Strategy

- **Settings, configuration:** Last-write-wins. Conflicts are rare and the latest value is usually correct.
- **Counters, votes:** Use a CRDT counter (or server-side increment to avoid conflicts entirely).
- **Collaborative text:** OT or a text-oriented CRDT. Don't attempt to build this yourself unless you're specifically building a collaborative editor.
- **Lists, collections:** Depends on merge semantics. Adding items is easy (union). Removing items is harder (what if one user edits an item another user deleted?).

---

## Backpressure & Rate Limiting

### Client Sending Too Fast

- Set a maximum message rate per client (e.g., 10 messages per second)
- Drop or queue excess messages
- Notify the client when they're being throttled

### Server Broadcasting to Slow Consumers

If a client can't keep up with the message rate (slow network, overwhelmed device):
- Buffer messages up to a limit
- If the buffer overflows, disconnect the slow client (they can reconnect and catch up)
- Never let a slow consumer block delivery to other clients

### Message Coalescing

For high-frequency updates (real-time metrics, cursor positions):
- Don't send every individual update
- Batch updates and send at intervals (e.g., position updates every 50ms)
- Send the latest state, not every intermediate state

---

## Testing Real-Time Systems

### Connection Lifecycle Tests

- Test connect, disconnect, reconnect flows
- Test authentication failures during connection
- Test behavior when the server restarts (do clients reconnect?)
- Test maximum connection limits

### Message Delivery Tests

- Test that published messages reach all subscribers
- Test message ordering (messages arrive in the order sent)
- Test that unsubscribed clients don't receive messages
- Test message delivery after reconnection (are missed messages replayed?)

### Failure Mode Tests

- Simulate network partitions (client connected but messages not flowing)
- Test behavior during server overload (connection limits, message backlog)
- Test graceful degradation (what does the UI show when real-time is unavailable?)
- Test reconnection with backoff (verify exponential increase and jitter)

### Load Testing

- Simulate many concurrent connections (hundreds to thousands)
- Measure message latency under load (time from publish to receipt)
- Monitor server resource usage (memory per connection, CPU for message fanout)
- Identify the connection count at which performance degrades

---

## Related Documents

- [03-architecture](../03-architecture.md) -- Event-driven patterns, async, background jobs
- [07-error-handling](../07-error-handling.md) -- Retry strategies, circuit breakers for real-time connections
- [08-performance](../08-performance.md) -- Connection management, memory, rate limiting
- [modules/ai-llm-integration](ai-llm-integration.md) -- SSE streaming for LLM responses
