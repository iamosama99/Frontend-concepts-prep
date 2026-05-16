# Offline-first Architecture Patterns

## Quick Reference

| Pattern | Core idea | Best for |
|---|---|---|
| Optimistic UI | Apply changes locally, sync in background | Low-conflict user actions (toggle, add) |
| Sync queue | Serialize mutations offline, replay when online | Form submissions, transactional writes |
| Cache-then-network | Render stale data immediately, update when fresh arrives | Feeds, dashboards |
| CRDT | Data structure that merges without conflict | Collaborative editing, distributed state |
| Local-first | Canonical state is on-device; server is just sync | High-autonomy offline apps |

---

## What Is This?

Offline-first architecture treats network connectivity as an enhancement rather than a requirement. The app works fully — reads, writes, navigation — without a network connection. When connectivity is restored, changes sync to the server.

This is a fundamentally different mental model from "online-required with offline graceful degradation": rather than designing for online and adding offline fallbacks, you design for offline and let the network improve the experience.

```
Online-first:            Offline-first:
if (online)              write to local store immediately
  fetch()                UI updates instantly
else                     ↓ background: sync queue
  show error             ↓ if online: flush to server
                         ↓ if offline: retry on reconnect
```

> **Check yourself:** What is the core difference between "offline-capable" and "offline-first"?

---

## Why It Matters

Mobile users routinely experience intermittent connectivity — elevators, tunnels, congested networks. Even on fast connections, network round-trips add latency to every interaction. Offline-first patterns solve both:
- **Offline**: the app still works
- **Online**: interactions are instant (local) and sync happens invisibly in the background

Apps like Notion, Linear, Figma, and Google Docs work offline-first: you can write, edit, and create while disconnected; changes merge when you reconnect.

---

## Optimistic UI

### The pattern

Apply the mutation locally (to local state or IndexedDB) immediately. Show the result. Send the request to the server in the background. If the server confirms, great — nothing to do. If the server rejects, roll back and notify the user.

```js
async function toggleFavorite(itemId) {
  // 1. Update local state immediately
  updateLocalState(itemId, { favorited: true });
  renderUI(); // user sees change instantly

  // 2. Sync to server in background
  try {
    await fetch(`/api/items/${itemId}/favorite`, { method: 'POST' });
  } catch (error) {
    // 3. Roll back on failure
    updateLocalState(itemId, { favorited: false });
    renderUI();
    showError('Failed to save — please try again');
  }
}
```

Optimistic UI is appropriate for:
- Low-conflict, reversible actions (like, favorite, toggle)
- Operations where the user has authority (their own data)
- High-latency networks where waiting feels broken

Not appropriate for:
- Purchases, financial transactions (must confirm server acceptance before showing success)
- Unique constraint operations (booking a seat — two users might both succeed locally)

> **Check yourself:** Why is optimistic UI dangerous for a seat reservation or payment flow?

---

## Sync Queue

### The pattern

For mutations that need to reach the server (form submissions, deletes, posts), queue them locally and flush the queue when online. This is the Background Sync pattern from Phase 7.2 / 8.6, applied at the architecture level.

```js
// Local sync queue backed by IndexedDB
const db = await openDB('sync-db', 1, {
  upgrade(db) {
    db.createObjectStore('queue', { autoIncrement: true });
  }
});

async function submitComment(comment) {
  // Always write locally first
  await db.add('queue', {
    type: 'CREATE_COMMENT',
    payload: comment,
    timestamp: Date.now(),
  });

  // Try to flush immediately if online
  if (navigator.onLine) {
    await flushQueue();
  }
  // Background Sync will flush when back online
}

async function flushQueue() {
  const tx = db.transaction('queue', 'readwrite');
  const store = tx.objectStore('queue');
  const all = await store.getAll();
  const keys = await store.getAllKeys();

  for (let i = 0; i < all.length; i++) {
    const item = all[i];
    try {
      await sendToServer(item);
      await db.delete('queue', keys[i]);
    } catch (err) {
      // Stop on first failure — preserve order
      break;
    }
  }
}
```

### Queue ordering considerations

If operations are independent (favoriting different items), they can flush in parallel. If they're dependent (create a document, then add content to it), they must flush in order. Design the queue structure accordingly.

---

## Cache-then-Network

### The pattern

Render stale cached data immediately, then update the UI when fresh data arrives from the network. The user sees content instantly (no loading state) and gets the updated version a moment later.

```js
async function loadFeed() {
  // 1. Render cached data immediately (may be stale)
  const cached = await localDB.get('feed-cache');
  if (cached) {
    renderFeed(cached.data);
  }

  // 2. Fetch fresh data from network
  try {
    const response = await fetch('/api/feed');
    const fresh = await response.json();

    // 3. Update UI and cache with fresh data
    renderFeed(fresh);
    await localDB.put('feed-cache', { data: fresh, timestamp: Date.now() });
  } catch {
    if (!cached) {
      showError('You appear to be offline');
    }
    // If we have cached data, it's already shown — no error needed
  }
}
```

This is the application-layer equivalent of the stale-while-revalidate Service Worker strategy. The challenge: if the cached and network data are significantly different, the UI update can be jarring (content jumping). Handle this with smooth transitions or only updating changed fields.

---

## Conflict Resolution

When the same data is modified offline by multiple users (or by the same user on multiple devices), conflicts arise. How you resolve them defines the app's consistency model.

### Last-Write-Wins (LWW)

Simplest: the most recent write (by timestamp or vector clock) wins. The losing write is discarded.

Good for: non-collaborative data (user settings, personal preferences). Bad for: collaborative data where both writes should be preserved.

```js
function merge(local, server) {
  return local.updatedAt > server.updatedAt ? local : server;
}
```

### Three-way merge

Compare the common ancestor with both branches. Apply changes from both relative to the ancestor. Conflict only if both branches changed the same field.

This is what git does. Some offline-first libraries (PouchDB, Dexie Syncable) implement this automatically.

### CRDTs (Conflict-free Replicated Data Types)

CRDTs are data structures mathematically designed to merge without conflicts. Any order of merges produces the same result (commutativity + associativity).

Examples:
- **G-Counter**: a grow-only counter where each device has its own slot
- **2P-Set**: a set with a grow-only add-set and grow-only remove-set
- **LWW-Register**: a last-write-wins register using vector clocks
- **RGA (Replicated Growable Array)**: collaborative text editing

```js
// G-Counter CRDT: each client has its own slot
class GCounter {
  constructor(nodeId) {
    this.nodeId = nodeId;
    this.counts = {};
  }

  increment() {
    this.counts[this.nodeId] = (this.counts[this.nodeId] || 0) + 1;
  }

  value() {
    return Object.values(this.counts).reduce((a, b) => a + b, 0);
  }

  merge(other) {
    const merged = new GCounter(this.nodeId);
    const allNodes = new Set([...Object.keys(this.counts), ...Object.keys(other.counts)]);
    for (const node of allNodes) {
      merged.counts[node] = Math.max(this.counts[node] || 0, other.counts[node] || 0);
    }
    return merged;
  }
}
```

Libraries like **Automerge** and **Yjs** implement full CRDT suites for JSON documents and text, powering real-time collaborative apps.

---

## Local-First Principles

"Local-first" (coined by Martin Kleppmann et al., 2019) is an architecture philosophy:

1. **Data lives on your device** — the local copy is the canonical source, not a cache
2. **The network syncs, not serves** — connectivity enables collaboration, not basic functionality
3. **You own your data** — the app works even if the vendor's servers go down

Implications for implementation:
- Use IndexedDB as the primary data store, not just a cache
- Write to local first, sync in background (not "write to server, cache locally")
- Use CRDTs or operational transforms for concurrent edits
- Structure sync as "sync" (bidirectional) not "upload/download"

Apps that approach this: Obsidian, Local-first Figma plugins, Linear (partial), Notion (partial).

---

## Putting It Together: Architecture Sketch

```
User action
    │
    ▼
Local write (IndexedDB)        ← always fast, always works
    │
    ▼
Optimistic UI update           ← instant feedback
    │
    ├── online → flush sync queue → server confirms
    │                                │
    │                            Update local (server truth wins)
    │
    └── offline → Background Sync registers → retry later
                       │
                   Browser fires sync event on reconnect
                       │
                   Queue flush → server reconciliation
```

---

## Gotchas

**1. `navigator.onLine` is unreliable**
`navigator.onLine` is `false` only when there's literally no network interface. A user on a captive portal (hotel Wi-Fi login page) or with high packet loss reports `navigator.onLine === true`. Design for network failure via `fetch` catching errors, not by checking `onLine`.

**2. Conflict resolution is a product decision, not just a technical one**
How conflicts resolve affects user expectations. Last-write-wins feels wrong for collaborative documents. Preserving both writes (a la git) confuses non-technical users. Involve product in defining what "conflict" means in your domain.

**3. Queue ordering must match operation semantics**
If your queue contains a "delete folder" and then "create file in that folder" (in offline order), flushing them in queue order to the server works. Flushing them reversed or in parallel causes the file creation to fail (folder doesn't exist yet). Order matters; parallel flushing requires careful dependency analysis.

**4. Sync state is hard to test**
Offline behavior requires simulating network conditions. Use DevTools Network throttling, `navigator.serviceWorker` mocking, or Playwright's `page.context().setOffline(true)`. Many bugs only surface on reconnect — test the reconnection flow explicitly.

**5. IndexedDB transaction ordering doesn't guarantee sync order**
Your local writes might be in a different order than what the server expects if multiple writes happen quickly. Assign incrementing sequence numbers to queued operations at write time to enforce ordering during flush.

**6. Large local datasets affect startup performance**
If your local IndexedDB has 50,000 records, rendering the list on startup means either loading all into memory (bad) or using cursor pagination. Design offline datasets to be bounded (last N days of data, not unlimited history).

---

## Interview Questions

**Q (High): What is the difference between offline-capable and offline-first? Give an example of each.**

Answer: Offline-capable means the app adds offline fallbacks to an otherwise online-first design — it works without connectivity, but the primary architecture assumes a network. A typical implementation: show cached data or an "you're offline" message when a fetch fails. Offline-first means the app is designed from the ground up to function without a network, with sync as an enhancement. A typical implementation: all reads and writes go to IndexedDB first; the network layer is a background sync that keeps local data in sync with the server. Example of offline-capable: a news app that shows yesterday's cached articles when offline. Example of offline-first: a notes app like Obsidian where creating, editing, and organizing notes works without any network connection, and sync happens when available.

**Q (High): Describe the optimistic UI pattern and when it's inappropriate.**

Answer: Optimistic UI applies a mutation to local state immediately and shows the result without waiting for server confirmation. The server request happens in the background; if it fails, the UI is rolled back. This gives instant perceived performance. It's inappropriate for operations with uniqueness constraints or high stakes — if two users both optimistically "claim" the last seat, both see success locally, but one will fail on the server. It's also inappropriate for purchases or financial transactions where the user must not see "success" before the server confirms. The rule: use optimistic UI when the operation's failure is unlikely and reversible; use pessimistic (confirm before showing success) when failure has real consequences.

**Q (High): What are CRDTs and what problem do they solve in offline-first apps?**

Answer: CRDTs (Conflict-free Replicated Data Types) are data structures designed so that any number of concurrent modifications, in any order, can be merged into the same final state — no manual conflict resolution needed. They solve the problem of concurrent offline edits: if User A and User B both edit a document offline and then sync, a CRDT-based system automatically produces a consistent merged result without either edit being lost. The mathematical property is that the merge operation is commutative, associative, and idempotent. Libraries like Yjs (used by Tiptap, Hocuspocus) and Automerge implement CRDT-based JSON documents. Without CRDTs, you need explicit conflict resolution policies (last-write-wins, three-way merge) that always trade off between simplicity and correctness.

**Q (Medium): Why is `navigator.onLine` unreliable for detecting connectivity?**

Answer: `navigator.onLine` is `false` only when the device reports no active network interface — no Wi-Fi, no Ethernet, no cellular. If the device has a connection but it's behind a captive portal, has severe packet loss, or the specific server is unreachable, `navigator.onLine` is still `true`. Designing offline logic around it means your app thinks it's online when it can't actually reach any servers. The correct approach: treat every `fetch` call as potentially failing; catch network errors and treat them as offline. Use `navigator.onLine` only as a quick optimization hint (e.g., "skip sync attempt if definitely offline"), not as the authoritative connectivity check.

**Q (Medium): How do you handle conflict resolution in a sync queue when the server rejects a queued operation?**

Answer: The strategy depends on the operation type. For idempotent operations (setting a field to a value), you can retry indefinitely — sending the same value again has no bad side effect. For non-idempotent operations (append to a list, increment a counter), retrying may double the effect if the server already processed it but didn't respond. Use idempotency keys — a UUID assigned when the operation is first queued — so the server can detect and deduplicate retries. On server rejection (4xx), the item should be removed from the queue and the user notified (can't auto-retry a business-logic rejection). On server error (5xx) or network failure, retry with exponential backoff. The queue should never silently drop failed operations that the user explicitly requested.

**Q (Low): What is the "local-first" philosophy and how does it differ from standard offline-capable apps?**

Answer: Local-first (Kleppmann et al., 2019) holds that the device's local data store is the canonical source of truth, not a cache. The server is a sync relay and backup, not the primary database. Network connectivity is optional for all operations. User data belongs to the user — the app works even if the vendor's servers shut down. This differs from a standard offline-capable app where the server is canonical: offline mode is a degraded state where you read from cache and queue writes for later upload. In a local-first app, you're always reading and writing the primary store; sync is a background reconciliation. Obsidian (local markdown files, optional sync) is a local-first app. A typical SaaS app with offline caching is not.

---

## Self-Assessment

- [ ] Explain the optimistic UI pattern and name two scenarios where it shouldn't be used
- [ ] Describe the sync queue pattern: how writes are stored, ordered, and flushed
- [ ] Explain what a CRDT is and why it eliminates manual conflict resolution
- [ ] Explain why `navigator.onLine` is unreliable and what to use instead
- [ ] Describe the "local-first" philosophy in one paragraph from memory

---
*Phase 8 complete. Next: Phase 9 — Client-Side Routing & URL Design, starting with the History API vs. Hash Routing: the two fundamental browser mechanisms for building SPAs without full-page reloads.*
