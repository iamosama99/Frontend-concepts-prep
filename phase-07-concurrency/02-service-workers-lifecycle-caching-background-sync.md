# Service Workers: Lifecycle, Caching, Background Sync

## Quick Reference

| Concept | What it is | Why it matters |
|---|---|---|
| Service Worker | Scriptable network proxy between page and internet | Intercepts every fetch, enables offline, push, background sync |
| Install → Activate → Idle | Three-phase lifecycle | Defines when a new SW takes control and old caches are cleared |
| `event.waitUntil()` | Extends a lifecycle event until a promise resolves | Prevents the browser from moving to the next phase prematurely |
| Cache API | Key-value store for `Request → Response` pairs | The mechanism behind offline-first; totally separate from HTTP cache |
| Background Sync | Deferred network requests that retry when online | Lets users queue work offline and trust it will complete later |

---

## What Is This?

A Service Worker is a type of Web Worker that sits between your web page and the network. Every `fetch()` call, every `<img src>`, every `<script>` request passes through it — and it can intercept, modify, respond from cache, or ignore each one.

Unlike a Dedicated Worker (which is tied to a page's lifetime), a Service Worker has an independent lifecycle. It can start up to handle a push notification or background sync event even when no tabs of your site are open.

```
Browser tab              Service Worker            Network
     │                        │                      │
     │─── fetch('/api') ──────▶│                      │
     │                        │─── cache hit ────────▶│ (skips network)
     │◀── response ───────────│                      │
```

> **Check yourself:** A Dedicated Worker and a Service Worker both run off the main thread. What is the fundamental architectural difference between them?

---

## Why Does It Exist?

The web had no reliable offline story before Service Workers. `AppCache` (the previous attempt, deprecated) was a declarative manifest that was famously confusing — opaque update semantics, difficult debugging, and no programmatic control.

Service Workers replaced it with a scriptable, event-driven proxy that gives developers full control:
- Decide exactly which requests to serve from cache
- Define update strategies per URL pattern
- Handle background tasks (sync, push) without an open tab

The network-proxy model is the right abstraction because it maps to what developers already think about: "for this type of request, do this thing."

---

## How It Works

### Registration

```js
// main.js — runs in the page
if ('serviceWorker' in navigator) {
  navigator.serviceWorker.register('/sw.js', { scope: '/' })
    .then(reg => console.log('SW registered', reg.scope))
    .catch(err => console.error('SW failed', err));
}
```

The `scope` determines which URLs the SW intercepts. A SW at `/sw.js` with scope `/app/` only intercepts requests under `/app/`. The script must be served from the origin it controls (no CDN hosting without workarounds).

### The Lifecycle

```
Register
    │
    ▼
Installing  ← sw.js is fetched, parsed, install event fires
    │           event.waitUntil(cacheAssets()) — delays until promise settles
    │
    ▼
Waiting     ← new SW waits while old SW still controls open tabs
    │           skipWaiting() skips this phase
    │
    ▼
Activating  ← activate event fires, safe to clear old caches
    │           event.waitUntil(deleteOldCaches())
    │
    ▼
Active      ← SW controls all in-scope clients
    │
    ▼
Idle / Terminated  ← browser may terminate idle SWs; they restart on events
```

A critical invariant: **only one Service Worker controls a given scope at a time**. A new SW doesn't replace the old one until all tabs using the old SW are closed (or `skipWaiting()` is called).

```js
// sw.js
self.addEventListener('install', (event) => {
  event.waitUntil(
    caches.open('v1').then(cache => cache.addAll([
      '/',
      '/app.js',
      '/styles.css',
    ]))
  );
});

self.addEventListener('activate', (event) => {
  event.waitUntil(
    caches.keys().then(keys =>
      Promise.all(keys.filter(k => k !== 'v1').map(k => caches.delete(k)))
    )
  );
  // Take control of all existing clients immediately
  event.waitUntil(self.clients.claim());
});
```

### The fetch event

This is where routing logic lives:

```js
self.addEventListener('fetch', (event) => {
  event.respondWith(handle(event.request));
});
```

`event.respondWith()` takes a promise that resolves to a `Response`. If you don't call it, the request goes to the network normally.

> **Check yourself:** What happens if `event.waitUntil()` is not called in the install event, and the cache population promise rejects?

---

## Caching Strategies

These are the five standard strategies — not a framework concept, just patterns for what `fetch` event handlers look like.

### Cache First (offline-first)
Best for: static assets (JS, CSS, images with versioned filenames).

```js
async function cacheFirst(request) {
  const cached = await caches.match(request);
  return cached ?? fetch(request);
}
```

Returns stale data if network is down. For versioned assets, "stale" is fine because the filename changes with each deploy.

### Network First (freshness-first)
Best for: API responses where you want fresh data but need offline fallback.

```js
async function networkFirst(request) {
  try {
    const response = await fetch(request);
    const cache = await caches.open('dynamic');
    cache.put(request, response.clone()); // clone because body can only be consumed once
    return response;
  } catch {
    return caches.match(request);
  }
}
```

### Stale-While-Revalidate
Best for: content that's okay to be slightly stale (news feed, profile data).

```js
async function staleWhileRevalidate(request) {
  const cache = await caches.open('dynamic');
  const cached = await cache.match(request);
  const networkFetch = fetch(request).then(response => {
    cache.put(request, response.clone());
    return response;
  });
  return cached ?? networkFetch;
}
```

Returns cached immediately, updates cache in background. The next request gets fresh data.

### Cache Only / Network Only
Trivial cases: serve everything from cache (pure offline mode) or bypass cache entirely (analytics, POST requests).

---

## Background Sync

Background Sync defers network requests until the user has connectivity, even if they navigate away or close the tab.

```js
// In the page: register a sync when saving a form
async function saveOffline(data) {
  const db = await openDB();
  await db.put('pending-sync', data);
  const reg = await navigator.serviceWorker.ready;
  await reg.sync.register('sync-user-data');
}

// In sw.js: handle the sync event when online
self.addEventListener('sync', (event) => {
  if (event.tag === 'sync-user-data') {
    event.waitUntil(syncUserData());
  }
});

async function syncUserData() {
  const pending = await getAllPendingFromDB();
  await Promise.all(pending.map(item => fetch('/api/save', {
    method: 'POST',
    body: JSON.stringify(item),
  })));
  await clearPendingFromDB();
}
```

The browser calls the `sync` event when connectivity is restored. If it fails, the browser retries with exponential backoff. `event.lastChance` is `true` on the final retry attempt.

**Periodic Background Sync** (Chrome only, needs permission) allows periodic wake-ups to refresh content — like a news app pre-fetching while offline.

---

## Updating a Service Worker

The browser refetches `sw.js` on every navigation (if ≥1 day since last check, or on manual `registration.update()`). If the byte content differs by even one character, it's a new version.

The update flow is the source of most SW bugs:

1. New SW installs but waits — old SW still controls open tabs
2. User must close all tabs and reopen, OR the new SW calls `self.skipWaiting()`
3. `skipWaiting()` + `clients.claim()` is the "force update" pattern — but it can cause a version mismatch mid-session (old page JS talking to new SW caches)

For most apps: prompt the user to reload when a new SW is detected.

```js
// main.js
const reg = await navigator.serviceWorker.register('/sw.js');
reg.addEventListener('updatefound', () => {
  const newSW = reg.installing;
  newSW.addEventListener('statechange', () => {
    if (newSW.state === 'installed' && navigator.serviceWorker.controller) {
      showUpdateBanner(); // "New version available, click to reload"
    }
  });
});
```

---

## Gotchas

**1. Service Workers only work on HTTPS (or localhost)**
This is a security requirement — a network proxy needs to be tamper-proof. `http://` deployments silently fail to register a SW.

**2. `response.clone()` before caching**
A `Response` body is a stream that can only be consumed once. If you call `cache.put(req, response)` and then `return response`, the body is already consumed. Always `.clone()` before caching: `cache.put(req, response.clone()); return response;`.

**3. Waiting phase is the biggest UX footgun**
New SW deploys sit in "waiting" while the user has old tabs open. Users on long sessions never see updates. Teams often add `skipWaiting()` without understanding the race condition it introduces (new SW + old page JS).

**4. Non-GET requests aren't cached by default**
`caches.match()` only matches GET requests. POST/PUT/DELETE requests pass through to the network unless explicitly handled. This is correct behavior — POST semantics shouldn't be cached — but surprises developers who expect all requests to be intercepted the same way.

**5. The SW script has a 24-hour max-age cap**
Even if you set `Cache-Control: max-age=31536000` on `sw.js`, browsers cap it at 24 hours for update checks. This prevents a broken SW from permanently blocking updates.

**6. `importScripts()` in a SW is version-coupled**
If `sw.js` calls `importScripts('/lib.js')` and `lib.js` changes, the SW won't update unless `sw.js` itself changes too. Bust the cache by including a hash in the importScripts URL.

**7. Opaque responses and cache poisoning**
Cross-origin responses fetched without CORS are "opaque" — status is always 0, size is always 0, content is hidden. If you cache an opaque response that's actually an error, you'll serve that error from cache indefinitely. Use `response.ok` checks before caching.

---

## Interview Questions

**Q (High): What is the Service Worker lifecycle, and why does the "waiting" phase exist?**

Answer: A Service Worker goes through three phases: install (SW script is fetched and parsed; `install` event fires, typically used to pre-cache assets), activate (SW is ready to take control; `activate` event fires, typically used to delete old caches), and then active (SW handles all requests for in-scope clients). The waiting phase sits between install and activate — a newly installed SW waits while the previous SW still controls open tabs. This prevents a version mismatch: if the new SW activated immediately, old page JavaScript expecting old cache structures would interact with new caches. The waiting phase ensures all tabs are on the same version before the new SW takes over.

The trap: candidates skip the waiting phase entirely or conflate install with activation. The interviewer is checking whether you understand why a new deploy doesn't instantly affect users with open tabs.

**Q (High): Describe three caching strategies and when you'd choose each one.**

Answer: Cache-first returns cached responses immediately without hitting the network, falling back to the network only on a cache miss. Choose it for static, versioned assets (bundled JS/CSS) where staleness is not a concern because the filename changes on each deploy. Network-first always attempts the network, falling back to cache only when offline. Choose it for API calls where fresh data matters but offline fallback is needed. Stale-while-revalidate returns the cached response immediately and revalidates in the background, so the next visit is fresh. Choose it for content that changes occasionally but where immediate staleness is acceptable — a user profile, a navigation menu.

The trap: candidates describe only one strategy (usually cache-first), or they treat stale-while-revalidate as "cache first with a background update" without explaining that the current request still serves stale.

**Q (High): Why must you call `response.clone()` before storing in the Cache API?**

Answer: A `Response` body is backed by a `ReadableStream` that can only be consumed once. The first read drains it. If you pass the same `Response` object to both `cache.put()` and back to `event.respondWith()`, one of them will get an empty body. `response.clone()` creates a second `Response` object backed by a tee of the same stream, so both consumers get the full body.

The trap: candidates don't know why — they know you have to clone but describe it vaguely. The interviewer is checking for understanding of the stream-based body model.

**Q (Medium): How does Background Sync work, and what guarantees does it provide?**

Answer: Background Sync lets you register a sync tag when the user performs an action offline. The browser queues the tag and fires a `sync` event on the Service Worker when connectivity is restored, even if all tabs are closed. The SW receives `event.tag` to identify which sync to run and `event.lastChance` to know if this is the final retry. Guarantees: the browser will retry on reconnection with exponential backoff; the SW gets a brief execution window to complete the sync. What it doesn't guarantee: a specific number of retries, a specific time delay, or success (if all retries fail and `event.lastChance` is true, the tag is dropped).

The trap: candidates assume Background Sync guarantees delivery — it doesn't; it guarantees retries. Or they don't know about `event.lastChance`.

**Q (Medium): Explain `skipWaiting()` and `clients.claim()` — when should you use them, and what are the risks?**

Answer: `skipWaiting()`, called inside the SW's `install` event, causes the newly installed SW to bypass the waiting phase and activate immediately, even while old tabs are still open. `clients.claim()`, called in `activate`, makes the newly activated SW take control of all existing clients without requiring a page reload. Together, they force-activate a new SW instantly on deploy. The risk: old page JavaScript may have cached references, event listeners, or state built against the old SW's cache structure. A mismatch between page-JS version and SW-cache version can cause subtle bugs — serving wrong assets, broken API assumptions. The safer approach: use neither, detect the new SW in the page, and show a "reload for update" prompt.

The trap: candidates recommend `skipWaiting()` + `clients.claim()` as a best practice without acknowledging the race condition.

**Q (Low): What is an opaque response, and why is caching one dangerous?**

Answer: An opaque response is the result of a cross-origin `fetch()` without CORS — the browser hides the response status, headers, and body for security. Its `status` is always 0 and `type` is `'opaque'`. Caching an opaque response is dangerous because you can't tell if it's a success or an error — a CDN returning a 503 or a Cloudflare error page looks identical to a 200. If you cache that error response, you'll serve it forever. The mitigation: check `response.type !== 'opaque'` before caching, or only cache resources you fetch with CORS enabled.

---

## Self-Assessment

- [ ] Draw the Service Worker lifecycle (install → waiting → activate → active) and explain what triggers each transition
- [ ] Implement a stale-while-revalidate fetch handler from memory
- [ ] Explain why `response.clone()` is required and what goes wrong without it
- [ ] Describe what Background Sync guarantees vs. what it doesn't
- [ ] Explain why a newly deployed SW doesn't immediately affect users with open tabs

---
*Next: Shared Workers — unlike Dedicated Workers (one page, one worker) and Service Workers (network proxy), Shared Workers expose a single worker instance to multiple tabs at once, enabling true cross-tab state sharing without a server.*
