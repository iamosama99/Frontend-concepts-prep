# Real-time Communication: WebSockets, SSE & Long Polling

## Quick Reference

| Technique | Direction | Protocol | Reconnect | Best fit |
|---|---|---|---|---|
| Long Polling | Server→Client (request-per-message) | HTTP | Manual | Legacy systems, firewall-restricted environments |
| Server-Sent Events (SSE) | Server→Client only | HTTP | Automatic | Notifications, feeds, live dashboards |
| WebSockets | Bidirectional | WS / WSS | Manual | Chat, collaborative editing, live gaming |

## What Is This?

Standard HTTP is request-response: the client asks, the server answers, the connection closes (or is reused but idle). There's no way for the server to spontaneously send data to the client. Real-time use cases — chat messages arriving, stock prices updating, collaborative cursors moving — need a model where the server can push data without the client polling for it.

Three techniques exist to achieve this, with different trade-offs in protocol complexity, browser support, and capability:

1. **Long Polling** — emulates push with a clever request pattern over plain HTTP
2. **Server-Sent Events (SSE)** — a unidirectional HTTP streaming protocol built into browsers
3. **WebSockets** — a full-duplex protocol upgrade for bidirectional real-time communication

> **Check yourself:** All three techniques let the server send data to the client. What's the fundamental difference between long polling and SSE in terms of the underlying HTTP connection?

## Long Polling

Long polling is the oldest technique and requires no special browser API — it's just HTTP used cleverly.

**How it works:**
1. Client sends a request: `GET /updates`
2. Server holds the request open — doesn't respond immediately
3. When data is available, server responds with the data
4. Client immediately sends a new request
5. Loop

```javascript
async function longPoll() {
  while (true) {
    try {
      const response = await fetch('/api/events?lastId=' + lastEventId);
      const data = await response.json();
      handleEvent(data);
      lastEventId = data.id;
    } catch (err) {
      // Connection error — wait before retrying
      await sleep(1000);
    }
    // Immediately re-request — no gap between events
  }
}
```

**Characteristics:**
- Works with any HTTP infrastructure — proxies, firewalls, CDNs
- Each "push" requires a full HTTP round-trip (headers, TCP overhead)
- Server holds connections open during idle periods — resource cost
- Reconnection logic is entirely manual — you write it
- No native browser API — just fetch/XHR

**When to use:** When WebSockets or SSE are blocked by firewalls or load balancers, in legacy environments, or when the infrastructure doesn't support persistent connections. Most modern systems have moved past this.

## Server-Sent Events (SSE)

SSE is a browser-native API for receiving a continuous stream of events from the server over a single, long-lived HTTP connection. It's unidirectional: server → client only.

**How it works:**
- Client opens an `EventSource` connection
- Server responds with `Content-Type: text/event-stream` and keeps the connection open
- Server writes event data to the response body incrementally
- Browser fires events for each received message
- If the connection drops, the browser automatically reconnects

**Wire format** (plain text):
```
data: {"type":"update","user":"alice","text":"hello"}\n\n

event: notification\n
data: {"count":5}\n\n

id: 42\n
data: {"type":"heartbeat"}\n\n
```

Each event is separated by a blank line. The `id` field enables automatic resume — the browser sends `Last-Event-ID` header on reconnect, so the server can replay missed events.

```javascript
// Client
const es = new EventSource('/api/stream');

es.onmessage = (event) => {
  const data = JSON.parse(event.data);
  updateUI(data);
};

// Named events
es.addEventListener('notification', (event) => {
  showNotification(JSON.parse(event.data));
});

es.onerror = () => {
  // EventSource automatically reconnects — this fires during the retry gap
  console.log('Reconnecting...');
};
```

```javascript
// Server (Node.js/Express)
app.get('/api/stream', (req, res) => {
  res.setHeader('Content-Type', 'text/event-stream');
  res.setHeader('Cache-Control', 'no-cache');
  res.setHeader('Connection', 'keep-alive');

  const sendEvent = (data) => {
    res.write(`data: ${JSON.stringify(data)}\n\n`);
  };

  // Push events as they happen
  const interval = setInterval(() => {
    sendEvent({ time: Date.now() });
  }, 1000);

  req.on('close', () => clearInterval(interval));
});
```

**Characteristics:**
- Native browser reconnection with exponential backoff
- `Last-Event-ID` enables replay of missed events on reconnect
- Plain text, human-readable wire format
- Standard HTTP — works through proxies and firewalls
- No binary support (text only without base64 encoding)
- HTTP/1.1 has a browser limit of ~6 connections per domain — SSE consumes one permanently. HTTP/2 multiplexing eliminates this limit.

**When to use:** Notifications, live feeds, activity streams, dashboards with one-way server push. If the client doesn't need to send data in the same channel (data sent separately via `fetch`), SSE is often simpler than WebSockets.

> **Check yourself:** SSE automatically reconnects when the connection drops. What information does the browser send on reconnect that allows the server to resume the event stream?

## WebSockets

WebSockets provide a full-duplex, persistent connection between browser and server. Either side can send messages at any time after the initial handshake.

**How it works:**
1. Client initiates an HTTP upgrade request
2. Server responds with `101 Switching Protocols`
3. The TCP connection is now a WebSocket connection — both sides can send frames
4. Messages flow in either direction until either side closes

```http
GET /chat HTTP/1.1
Host: example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13
```

```javascript
// Client
const ws = new WebSocket('wss://example.com/ws');

ws.onopen = () => {
  ws.send(JSON.stringify({ type: 'join', room: 'general' }));
};

ws.onmessage = (event) => {
  const message = JSON.parse(event.data);
  renderMessage(message);
};

ws.onclose = (event) => {
  console.log('Closed:', event.code, event.reason);
  // No automatic reconnect — must implement manually
};

ws.onerror = (err) => {
  console.error('WebSocket error:', err);
};

// Send from client to server at any time
function sendMessage(text) {
  if (ws.readyState === WebSocket.OPEN) {
    ws.send(JSON.stringify({ type: 'message', text }));
  }
}
```

**Characteristics:**
- True bidirectional — client and server both send without waiting
- Supports binary frames (ArrayBuffer, Blob) — can send images, audio, compressed data
- Very low overhead per message after handshake — no HTTP headers on each frame
- No automatic reconnection — must be implemented manually with backoff
- Stateful — load balancers must support sticky sessions or a pub/sub layer (Redis) for multi-server deployments

**Reconnection pattern:**
```javascript
function createWebSocket(url) {
  let ws;
  let reconnectDelay = 1000;

  function connect() {
    ws = new WebSocket(url);

    ws.onopen = () => {
      reconnectDelay = 1000;  // reset backoff on successful connection
    };

    ws.onclose = () => {
      setTimeout(connect, reconnectDelay);
      reconnectDelay = Math.min(reconnectDelay * 2, 30000);  // exponential backoff, max 30s
    };
  }

  connect();
  return { send: (data) => ws?.send(data) };
}
```

**When to use:** Chat applications, collaborative editing (Google Docs model), multiplayer games, live cursors, any use case requiring the client to send real-time data to the server at low latency.

## Comparison

| | Long Polling | SSE | WebSockets |
|---|---|---|---|
| Bidirectional | No (client must re-request) | No | Yes |
| Auto-reconnect | No | Yes | No (manual) |
| Binary support | Via base64 | Via base64 | Yes (native) |
| Header overhead | Per message | Only on connect | Only on handshake |
| Proxy support | Excellent | Good | Variable |
| HTTP/2 compatible | Yes | Yes (multiplexed) | Requires upgrade |
| Complexity | Low | Low | Medium |
| State on server | No (each request independent) | Per-connection state | Per-connection state |

## WebSocket Scaling: The Statefulness Problem

WebSockets are stateful — each connected client is a persistent open connection on a specific server. In a horizontal scaling setup with multiple servers, a client connected to server A can't receive messages intended for them if those messages arrive at server B.

Solutions:
- **Sticky sessions:** Load balancer routes a client's requests to the same server always. Single point of failure if that server goes down.
- **Pub/sub layer (Redis, Kafka):** Each server publishes to a shared channel. All servers subscribe. When server B needs to send a message to a client connected to server A, it publishes to the channel — server A sees it and delivers to the client.
- **Managed services (Pusher, Ably, Supabase Realtime):** Outsource the WebSocket infrastructure entirely. Your backend emits events via an HTTP API; the service manages WebSocket connections to clients.

SSE avoids this problem only if the server can rehydrate the client's state from the `Last-Event-ID` — the reconnect can go to any server, as long as that server can resume the correct stream.

## Gotchas

**WebSocket connections don't survive page navigation.** Unlike HTTP which is stateless, a WebSocket connection is tied to the browser tab. Navigation kills it. For SPAs, this is usually fine — the app controls navigation. For multi-page apps, reconnection on every page load is a real cost.

**SSE is blocked if you're using HTTP/2 with a proxy that doesn't support it.** Some proxies buffer streaming responses before forwarding — killing the "stream" aspect of SSE. Test your infrastructure with `curl -N` to verify chunks arrive incrementally.

**WebSocket `readyState` must be checked before sending.** `ws.send()` throws if the connection isn't open. Always check `ws.readyState === WebSocket.OPEN` before calling `send()`.

**Long polling is not free on the server.** Holding thousands of open HTTP connections waiting for data consumes file descriptors and memory. Servers need to be configured for high connection counts (Node.js/nginx defaults may be too low). This is less of a concern with event-driven servers (Node.js, Go) than with thread-per-request models.

**SSE has browser connection limits under HTTP/1.1.** Browsers limit concurrent connections per domain — SSE occupies one of those connections permanently. On HTTP/2, this is irrelevant since connections are multiplexed. Always serve SSE over HTTP/2 in production.

## Interview Questions

**Q (High): Compare WebSockets and SSE. When would you choose one over the other?**

Answer: WebSockets provide full-duplex communication — client and server both send freely on the same connection. SSE is unidirectional — server-to-client only, via plain HTTP streaming. Choose SSE when the server is the only sender: notifications, live activity feeds, real-time dashboards, streaming API responses. SSE is simpler — native auto-reconnect, the `Last-Event-ID` resume mechanism, and standard HTTP compatibility make it lower-friction. Choose WebSockets when the client must also send data in real time on the same channel: chat messages, collaborative editing cursor positions, multiplayer game state. WebSockets also support binary frames natively (SSE is text-only) and have lower per-message overhead after the initial handshake. The practical default: if the use case is purely server-push, start with SSE. Add WebSockets only when you need bidirectional low-latency messaging.

The trap: "WebSockets are always better because they're bidirectional." SSE's auto-reconnect and simpler infrastructure story often makes it the right choice for server-push use cases.

---

**Q (High): How does the WebSocket handshake work, and why does it start as an HTTP request?**

Answer: WebSockets start as HTTP to leverage existing infrastructure — firewalls, proxies, and load balancers understand HTTP. The client sends a standard HTTP GET request with `Upgrade: websocket` and `Connection: Upgrade` headers, along with a `Sec-WebSocket-Key` (a base64-encoded random value). The server, if it supports WebSockets, responds with `101 Switching Protocols` and a `Sec-WebSocket-Accept` header (derived by hashing the key with a magic string). After this, the TCP connection is "upgraded" — no more HTTP framing. Both sides communicate using WebSocket frames: small binary headers indicating frame type, length, and masking, followed by the payload. The HTTP origin allows authentication cookies and CORS to apply to the handshake, giving you the same security model as regular requests.

The trap: Not knowing the handshake is HTTP-based. This matters for security (CSRF on WebSocket handshakes is a real concern — always validate the Origin header server-side).

---

**Q (Medium): How would you implement WebSocket reconnection with exponential backoff, and why is backoff important?**

Answer: Exponential backoff means each reconnection attempt waits progressively longer — 1s, 2s, 4s, 8s, up to a maximum (e.g., 30s). After a successful connection, the delay resets to the minimum. Without backoff, if a server goes down and has 10,000 connected clients, all 10,000 immediately retry simultaneously — creating a "thundering herd" that can overwhelm the recovering server, preventing recovery. Backoff spreads reconnection attempts over time. In code: `onclose`, schedule a `setTimeout(connect, delay)` and double the delay (capped at max). On `onopen`, reset delay to minimum. Jitter (adding randomness to the delay) further spreads simultaneous reconnections from many clients: `delay = min + Math.random() * (max - min)`. Libraries like `reconnecting-websocket` implement this correctly so you don't have to.

The trap: Implementing fixed-delay retry without backoff. This is a well-known production failure pattern.

---

**Q (Medium): What is the `Last-Event-ID` in SSE and why does it matter?**

Answer: `Last-Event-ID` is the mechanism that makes SSE resumable. When the server sends events with an `id` field (e.g., `id: 42`), the browser stores the most recent ID. When the connection drops and the browser automatically reconnects, it sends the `Last-Event-ID: 42` header in the reconnect request. The server reads this header and replays any events that occurred after ID 42, ensuring the client doesn't miss events that were sent during the disconnection window. Without this mechanism, a client that drops during a network hiccup would miss all events that occurred while disconnected — potentially missing critical notifications or state updates. This is the key advantage of SSE over a naive polling approach: SSE can guarantee delivery across reconnects when the server stores events with IDs.

The trap: Thinking SSE is "just like WebSockets but simpler." The auto-reconnect and Last-Event-ID are genuinely differentiating features that WebSockets don't provide natively.

---

**Q (Low): How do you scale a WebSocket server horizontally, and what are the trade-offs of each approach?**

Answer: WebSocket connections are stateful — a client is connected to a specific server instance. Scaling horizontally requires one of: (1) Sticky sessions — the load balancer uses a consistent hash (client IP, cookie) to route each client to the same server always. Simple but fragile — if that server dies, all its clients reconnect distributed to remaining servers, and any state on the dead server is lost. (2) Pub/sub layer — all servers subscribe to a shared message bus (Redis pub/sub, Kafka). When any server needs to send to a specific client, it publishes to the bus; the server that has that client's connection sees the message and delivers it. State is external to servers, enabling stateless server scaling. More infrastructure but much more robust. (3) Managed WebSocket services (Pusher, Ably) — outsource the WebSocket layer entirely; your backend sends HTTP calls to their API; they manage all client connections and delivery. Highest operational simplicity, recurring cost, and vendor dependency.

The trap: Not knowing WebSockets have a statefulness scaling problem at all.

---

## Self-Assessment

Before moving on, check off each item you can do WITHOUT looking at the file.

- [ ] Can explain why standard HTTP can't be used for server-to-client push without polling
- [ ] Can describe how SSE works (HTTP stream, `text/event-stream`, event format)
- [ ] Can explain `Last-Event-ID` and why SSE's auto-reconnect matters
- [ ] Can describe the WebSocket handshake (HTTP upgrade → 101 → full-duplex)
- [ ] Can implement WebSocket reconnection logic with exponential backoff (conceptually)
- [ ] Can explain the WebSocket horizontal scaling problem and the pub/sub solution

---
*Next: BFF Pattern (Backend-for-Frontend) — when and why you introduce a dedicated API layer per client surface, and the aggregation and security problems it solves.*
