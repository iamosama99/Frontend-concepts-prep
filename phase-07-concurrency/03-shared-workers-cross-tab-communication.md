# Shared Workers & Cross-tab Communication

## Quick Reference

| Tool | Scope | Use case |
|---|---|---|
| Dedicated Worker | One page, one worker | Off-thread computation for a single page |
| Shared Worker | One worker, many tabs on same origin | Shared state, de-duplicated connections, cross-tab coordination |
| BroadcastChannel | Main thread to main thread, across tabs | Simple pub/sub without a shared worker |
| `MessagePort` | Worker ↔ client channel | The wire Shared Workers use to talk to each page |

---

## What Is This?

A Shared Worker is a Web Worker that can be connected to by multiple browsing contexts — tabs, iframes, or other workers — from the same origin. All contexts share the same worker instance: one thread, one heap, one set of variables.

```js
// Any tab on the same origin
const worker = new SharedWorker('/shared.js');
worker.port.start();
worker.port.postMessage('hello');
worker.port.onmessage = (e) => console.log(e.data);
```

The key difference from a Dedicated Worker: where a Dedicated Worker gets a single `onmessage` handler, a Shared Worker receives a `connect` event each time a new context connects, giving it a `MessagePort` for each connection.

```js
// shared.js
const ports = [];

self.addEventListener('connect', (event) => {
  const port = event.ports[0];
  ports.push(port);
  port.start();

  port.onmessage = (e) => {
    // Broadcast to all connected tabs
    ports.forEach(p => p.postMessage({ from: 'worker', data: e.data }));
  };
});
```

> **Check yourself:** What happens to a Shared Worker when the last tab connected to it closes?

---

## Why Does It Exist?

Some problems require shared state across multiple tabs:

- **Single WebSocket connection**: opening one WebSocket per tab is wasteful (and some servers impose connection limits). A Shared Worker holds one socket and fans out messages to all tabs.
- **Shared in-memory cache**: a computed dataset that all tabs need, without fetching or computing it N times.
- **Cross-tab coordination**: "only one tab should poll the server at a time."

Before Shared Workers, the hacks were: `localStorage` change events (limited, synchronous writes), `BroadcastChannel` (no shared state, just messages), or server-side coordination. Shared Workers are the only browser primitive that gives you a genuine shared heap across tabs.

---

## How It Works

### Identity by URL

Two calls to `new SharedWorker('/shared.js')` from different tabs connect to the **same worker** — identified by script URL + (optionally) name. The worker is instantiated once; subsequent connections trigger additional `connect` events.

```js
// Tab A and Tab B both do:
const w = new SharedWorker('/shared.js', 'my-worker');
// They're connected to the exact same instance
```

Changing the URL or name creates a new worker. Shared Workers are origin-scoped — `https://example.com` and `https://app.example.com` cannot share a worker.

### The port model

Communication goes through `MessagePort`, not directly on the worker. Every connected context gets a `port` from the `connect` event:

```
Tab A                  Shared Worker                  Tab B
  │                         │                           │
  │──port.postMessage()────▶│                           │
  │                         │──portB.postMessage()─────▶│
  │                         │                           │
```

You must call `port.start()` explicitly (unlike Dedicated Workers where `onmessage` auto-starts the port). Forgetting `port.start()` is the most common Shared Worker bug — messages are queued but never received.

### Lifecycle

The Shared Worker starts on the first `new SharedWorker(url)` call. It terminates when no ports remain connected — which happens when all connecting contexts are closed. The browser garbage-collects it.

```js
// shared.js — tracking live ports
const ports = new Set();

self.addEventListener('connect', (e) => {
  const port = e.ports[0];
  ports.add(port);
  port.start();

  // No close event exists — clean up via heartbeat or SharedWorker-specific patterns
  port.onmessage = (e) => {
    if (e.data === 'disconnect') ports.delete(port);
  };
});
```

There is no `disconnect` event on `MessagePort`. To track when a tab closes, use a heartbeat: tabs ping the worker periodically; the worker evicts ports that miss N heartbeats.

---

## BroadcastChannel — Simpler Cross-tab Messaging

When you don't need shared state — just need to send events between tabs — `BroadcastChannel` is simpler:

```js
// In any tab
const channel = new BroadcastChannel('app-updates');

// Send to all other tabs on same origin
channel.postMessage({ type: 'USER_LOGGED_OUT' });

// Receive in any tab
channel.onmessage = (e) => {
  if (e.data.type === 'USER_LOGGED_OUT') logout();
};
```

`BroadcastChannel` is fire-and-forget: there's no reply mechanism, no shared state, and the sender does NOT receive its own messages. It's essentially pub/sub over origin-scoped named channels.

When to choose which:

| Need | Tool |
|---|---|
| Shared in-memory state across tabs | Shared Worker |
| Single WebSocket / polling connection | Shared Worker |
| One-tab broadcasts events to others | BroadcastChannel |
| Leader election (one tab acts as primary) | BroadcastChannel or Web Locks API |

> **Check yourself:** Why does `BroadcastChannel` not satisfy the "single WebSocket across tabs" requirement?

---

## Practical Example: Shared WebSocket

The canonical use case — one WebSocket connection, many consumers:

```js
// shared.js
let socket;
const ports = new Set();

function ensureSocket() {
  if (socket && socket.readyState === WebSocket.OPEN) return;
  socket = new WebSocket('wss://api.example.com/live');
  socket.onmessage = (e) => {
    ports.forEach(port => port.postMessage(JSON.parse(e.data)));
  };
  socket.onclose = () => setTimeout(ensureSocket, 2000); // reconnect
}

self.addEventListener('connect', (e) => {
  const port = e.ports[0];
  ports.add(port);
  port.start();
  ensureSocket();

  port.onmessage = (msg) => {
    if (msg.data.type === 'SEND') socket.send(JSON.stringify(msg.data.payload));
    if (msg.data.type === 'DISCONNECT') ports.delete(port);
  };
});
```

---

## Web Locks API — Coordination Primitive

For cross-tab coordination without shared state, the Web Locks API provides mutex semantics:

```js
// Only one tab at a time enters this block
await navigator.locks.request('sync-lock', async (lock) => {
  await performSyncOperation();
});
// If another tab already holds 'sync-lock', this queues until it releases
```

Use cases:
- Ensuring only one tab polls an API
- Preventing concurrent IndexedDB migrations
- Leader election (the tab holding the lock is the leader)

Locks are automatically released when the holder tab closes or crashes — no deadlock risk from unexpected termination.

---

## Gotchas

**1. Forgetting `port.start()`**
Unlike Dedicated Workers, Shared Worker ports don't auto-start. Without `port.start()`, messages pile up in an internal queue and are never delivered. No error is thrown — it just silently doesn't work.

**2. No disconnect event**
When a tab closes, the Shared Worker's port does not fire a `close` or `disconnect` event. You'll accumulate dead ports in a `Set` forever unless you implement a heartbeat or cleanup protocol.

**3. DevTools support is poor**
Chrome's Task Manager shows Shared Workers. Chrome DevTools requires navigating to `chrome://inspect/#workers` to attach a debugger. Firefox's DevTools are better. This makes debugging significantly harder than Dedicated Workers.

**4. Module Shared Workers have limited browser support**
`new SharedWorker(url, { type: 'module' })` is only supported in Chrome 83+. Firefox and Safari support classic Shared Workers but not module workers. Check compatibility before using ES module syntax.

**5. Shared Workers don't survive origin changes**
A Shared Worker is scoped to an origin. If your app migrates from `app.example.com` to `example.com/app`, all workers are different instances. Any shared state is lost.

**6. State is lost on last tab close**
Since the Shared Worker terminates when no ports remain, in-memory state doesn't persist across sessions. If you need persistence, write to IndexedDB before shutdown — but there's no reliable shutdown hook. Design Shared Worker state as ephemeral.

---

## Interview Questions

**Q (High): What is a Shared Worker and how does it differ from a Dedicated Worker?**

Answer: A Shared Worker is a Web Worker accessible by multiple browsing contexts (tabs, iframes, workers) on the same origin. Where a Dedicated Worker is owned 1-to-1 by the creating context and terminates when that context closes, a Shared Worker persists as long as any context is connected. Communication uses `MessagePort` rather than direct `onmessage` — the worker receives a `connect` event for each new connection, gets a port from `event.ports[0]`, and maintains a set of ports to communicate with all connected clients. The classic use case is holding a single WebSocket connection that multiple tabs share.

The trap: candidates say "shared across tabs" but can't describe the `connect` event or `MessagePort` model — they're conflating BroadcastChannel with Shared Workers.

**Q (High): Compare BroadcastChannel and Shared Workers for cross-tab communication. When would you choose each?**

Answer: BroadcastChannel is a simple pub/sub mechanism — any message you post goes to all other listeners on the same named channel, no shared state. It's best when you need to broadcast events (logout, theme change, new notification) and don't need a reply or shared heap. Shared Workers provide a live JS context shared by all tabs, with bi-directional communication per port. They're necessary when you need genuine shared state (a cache, a WebSocket, a polling interval) that must only exist once regardless of how many tabs are open. BroadcastChannel can't replace a Shared Worker for the WebSocket use case because each tab would still need to establish its own connection — there's no shared execution context.

**Q (Medium): Why is there no disconnect event when a tab closes in a Shared Worker? How do you work around it?**

Answer: When a browsing context closes, its `MessagePort` is GC'd but there's no clean close notification sent to the other end — the spec doesn't require it. The port silently becomes inert. The workaround is a heartbeat protocol: each tab pings the worker on a regular interval (every few seconds), and the worker tracks the last-seen timestamp per port. If a port hasn't responded in N intervals, the worker removes it from its active set. The worker can also try posting to all ports and catch the `postMessage` error on dead ports (though this is not reliable cross-browser).

**Q (Medium): What is the Web Locks API and when does it solve a problem that Shared Workers can't?**

Answer: Web Locks provides mutex/semaphore semantics across tabs and workers. `navigator.locks.request(name, callback)` acquires a named lock — only one holder at a time, others queue. The lock is automatically released when the callback resolves or the holder crashes. It solves problems requiring mutual exclusion — one tab pollingthe server, one tab running a migration — without needing any shared state. Compared to a Shared Worker leader-election approach, Locks are simpler (no worker setup), safer (no deadlock on crash), and browser-native. A Shared Worker is better when you need the actual shared computation context; Locks are better when you just need "only one actor at a time."

**Q (Low): What happens if two tabs load `new SharedWorker('/sw.js')` simultaneously?**

Answer: The browser guarantees that only one worker instance is created. The first call starts the worker; the second call connects to the same instance — it fires a second `connect` event on the worker. This is atomic from the browser's perspective; there's no race condition where two instances could be created. The worker identity is keyed by (origin, script URL, name). If the name differs, they're different workers.

---

## Self-Assessment

- [ ] Explain the `connect` event + `MessagePort` pattern in a Shared Worker from memory
- [ ] Write a Shared Worker that fans out messages to all connected tabs
- [ ] Explain why forgetting `port.start()` causes silent message loss
- [ ] Describe when to choose BroadcastChannel vs. Shared Worker vs. Web Locks
- [ ] Explain why Shared Worker state is ephemeral and how you'd persist it

---
*Next: Worklets — a lightweight sibling of Workers designed to extend specific browser rendering pipelines: CSS custom paint (Houdini), Web Audio processing, and CSS animation control — each running in a dedicated, sandboxed context.*
