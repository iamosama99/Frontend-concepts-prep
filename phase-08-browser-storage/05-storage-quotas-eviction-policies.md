# Storage Quotas & Eviction Policies

## Quick Reference

| API | Storage type | Eviction risk |
|---|---|---|
| `navigator.storage.estimate()` | Query quota + usage | — |
| `navigator.storage.persist()` | Request persistent storage | Low/none if granted |
| Best-effort (default) | Browser may evict under pressure | High — no eviction order guarantee |
| Persistent | Browser will not evict | Requires user/browser permission |
| localStorage | Origin-scoped, ~5–10MB | Browser-dependent; generally not evicted arbitrarily |

---

## What Is This?

Every storage API — IndexedDB, Cache API, localStorage, service worker registrations, and opaque response caches — draws from a shared per-origin quota. The browser allocates a slice of disk space to each origin; once you exceed it, writes fail. Under storage pressure (device low on disk), the browser can evict **best-effort** storage from origins that haven't been visited recently.

```js
const { quota, usage } = await navigator.storage.estimate();
console.log(`Using ${(usage / quota * 100).toFixed(1)}% of available storage`);
console.log(`${(quota - usage) / 1e6} MB remaining`);
```

> **Check yourself:** If a device is low on disk space and your origin uses only "best-effort" storage, what can the browser do to your data?

---

## Why It Exists

Without quotas, a malicious or poorly written app could consume all of the device's disk space. Without eviction policies, a browser that accumulates cached data over time would run out of disk on its own. Quotas and eviction are the browser's internal resource manager for persistent client-side state.

Developers need to understand this because:
1. **Quota exhaustion fails silently in some paths** — IndexedDB writes throw, but not all APIs are obvious about it
2. **Eviction can destroy offline-first app data** — a user who hasn't visited in 2 weeks might return to find their cached data gone
3. **Opaque responses have disproportionate quota costs** — a few cross-origin images without CORS can consume gigs of quota allowance on paper

---

## How Quotas Work

### Per-origin storage pool

All quota-governed storage (IndexedDB, Cache API, FileSystem) draws from a single pool per origin. localStorage has its own separate, smaller limit (~5–10MB) managed independently.

### Quota allocation

The exact allocation is browser-defined and varies:
- Chrome: typically 60% of total disk space, split across all origins
- Firefox: similar proportions
- Safari: more conservative, historically more restrictive

No spec mandates exact numbers — `navigator.storage.estimate()` returns the actual allocation for the current origin:

```js
const estimate = await navigator.storage.estimate();
// {
//   quota: 3298534883328,  // bytes allocated to this origin
//   usage: 1048576,        // bytes currently used
//   usageDetails: {
//     caches: 524288,      // Cache API
//     indexedDB: 524288,   // IndexedDB
//     serviceWorkerRegistrations: 0
//   }
// }
```

`usageDetails` is not universally supported — don't rely on it for production logic, but it's useful for debugging.

### Opaque response quota inflation

Cross-origin responses fetched without CORS are stored as opaque — their actual size is unknown to the browser's quota accounting system. To prevent side-channel attacks that infer content size from quota changes, browsers pad opaque response entries to ~7MB in quota accounting (regardless of actual size). A single opaque response from a tiny cross-origin script costs 7MB of your quota allowance.

With CORS enabled, the actual byte size is used. Always serve cross-origin resources with appropriate CORS headers if they'll be cached.

---

## Best-Effort vs. Persistent Storage

### Best-effort (default)

All storage starts as best-effort. The browser **may evict it at any time** under storage pressure, without asking the user, following an LRU-like policy (origins not visited recently are evicted first).

This is fine for cache data (Service Worker caches, prefetched assets) — losing it just means a slower next visit, not data loss. But it's a problem for user-generated data stored client-side.

### Persistent storage

An origin can request persistent storage:

```js
const granted = await navigator.storage.persist();
console.log(granted ? 'Persistent' : 'Best-effort');
```

If granted, the browser will not evict this origin's storage without explicit user action (clearing site data). In Chrome, `persist()` is automatically granted if:
- The site is bookmarked by the user
- The site is installed as a PWA
- The user has granted notification permission
- The site has high engagement score

In Firefox, `persist()` triggers a permission prompt. In Safari, behavior varies by version.

You can check current status:

```js
const persisted = await navigator.storage.persisted();
console.log(persisted); // true if granted
```

---

## Eviction Order

When the browser is under storage pressure, it follows a priority order for eviction (per the Storage spec):

1. Origins with best-effort storage, least-recently-visited first
2. Temporary storage (service worker caches, opaque caches)
3. Session storage (doesn't persist across sessions anyway)

Persistent storage origins are never evicted automatically.

In practice, Chrome evicts by LRU across all best-effort origins. An offline-first app that a user hasn't opened in 7–14 days may find its caches and IndexedDB data gone when it next loads.

---

## Handling Quota Errors

IndexedDB write failures on quota exceed:

```js
const request = store.put(largeObject);
request.onerror = (e) => {
  if (e.target.error.name === 'QuotaExceededError') {
    // Notify user, delete old data, or request persistent storage
    handleQuotaError();
  }
};
```

Cache API quota exceed:

```js
try {
  await cache.put(request, response);
} catch (e) {
  if (e.name === 'QuotaExceededError') {
    await evictOldCacheEntries();
    await cache.put(request, response); // retry after cleanup
  }
}
```

localStorage quota exceed (synchronous throw):

```js
try {
  localStorage.setItem('large', data);
} catch (e) {
  if (e.name === 'QuotaExceededError') {
    // Clean up old localStorage keys
  }
}
```

---

## Storage Management Strategies

### LRU eviction within your own caches

Rather than waiting for the browser to evict your whole origin, implement your own cache LRU with a max entry count or max age:

```js
async function trimCache(cacheName, maxEntries) {
  const cache = await caches.open(cacheName);
  const keys = await cache.keys();
  if (keys.length > maxEntries) {
    // Delete oldest entries (first in the list)
    const toDelete = keys.slice(0, keys.length - maxEntries);
    await Promise.all(toDelete.map(key => cache.delete(key)));
  }
}
```

### Request persistent storage on install

For PWAs and offline-first apps, request persistence on first meaningful user interaction (not on page load — browsers may deny it):

```js
async function requestPersistence() {
  if (navigator.storage && navigator.storage.persist) {
    const persisted = await navigator.storage.persisted();
    if (!persisted) {
      const granted = await navigator.storage.persist();
      if (!granted) {
        // Inform user that offline data may be cleared
        showStorageWarning();
      }
    }
  }
}
```

### Monitor usage periodically

For data-heavy apps, monitor and warn before hitting quota:

```js
async function checkStorageHealth() {
  const { quota, usage } = await navigator.storage.estimate();
  const percentUsed = (usage / quota) * 100;
  if (percentUsed > 80) {
    reportToAnalytics('storage_pressure', { percentUsed });
    cleanOldData(); // delete old cached responses, old IndexedDB records
  }
}
```

> **Check yourself:** A user installs your PWA as a home screen app on iOS Safari. Does that automatically grant persistent storage?

---

## Gotchas

**1. Eviction can happen between sessions without warning**
A user who hasn't visited in two weeks may return to find all their offline data gone. Your app must handle a "first visit" state even for returning users, and gracefully degrade when cached data is missing.

**2. `navigator.storage.persist()` outcome varies wildly by browser/OS**
On mobile browsers, the criteria for granting persistent storage differ from desktop. iOS Safari has historically been the most restrictive. Don't assume `persist()` will be granted.

**3. `navigator.storage.estimate()` quotas are not hard limits**
The reported quota is approximate — the browser may grant more or less depending on current device conditions. Treat it as a planning figure, not a contract.

**4. Opaque response quota cost is not reflected in `usage`**
The 7MB accounting for opaque responses is part of quota accounting but may or may not appear accurately in `estimate().usage`. DevTools Application > Storage shows a more accurate breakdown.

**5. Clearing "site data" in browser settings clears persistent storage too**
Persistent storage is protected from browser-initiated eviction, not from user-initiated clearing. If the user goes to browser settings and clears site data, persistent storage is gone. Never rely on local storage alone for critical data — always have a sync mechanism.

**6. localStorage quota is separate from the main pool**
localStorage's ~5MB limit is not drawn from the same quota as IndexedDB and Cache API. They're tracked independently. A full localStorage doesn't affect IndexedDB quota, and vice versa.

---

## Interview Questions

**Q (High): What is the difference between best-effort and persistent storage? When would you request persistent storage?**

Answer: Best-effort storage (the default for all quota-governed APIs — IndexedDB, Cache API, etc.) can be evicted by the browser under storage pressure, typically using LRU against origins not recently visited. Persistent storage is protected: the browser won't evict it without user action. Request persistent storage for offline-first apps or PWAs that store user-generated data client-side — data loss on eviction would be a user-facing bug, not just a slower load. Request it after a meaningful user interaction (`navigator.storage.persist()`), not on page load — browsers may deny automatic requests. If denied, inform the user that offline data may be cleared if device space runs low.

**Q (High): What is the opaque response quota inflation problem?**

Answer: Cross-origin resources fetched without CORS are opaque — their size is hidden from the browser's quota accounting. To prevent side-channel attacks where you could infer resource size by watching quota changes, browsers charge ~7MB per opaque response entry in quota accounting, regardless of the actual file size. A tiny 2KB cross-origin script cached as opaque costs 7MB of quota. Accumulate a few dozen of these and you've burned hundreds of MB of quota allowance on small files. The fix: use CORS for any cross-origin resource you intend to cache (`crossorigin` attribute, or server adding `Access-Control-Allow-Origin`). With CORS, the actual byte size is used.

**Q (Medium): How would you implement cache eviction in a Service Worker to prevent quota exhaustion?**

Answer: After writing new entries to the Cache API, count the total entries and delete the oldest (lowest-index, assuming `keys()` returns in insertion order) down to a maximum count. This can run after a successful cache write in the fetch handler. For time-based eviction, store a timestamp in the cache key or in a parallel IndexedDB store and delete entries older than N days. Run the cleanup in `activate` events (which fire periodically as the SW cycles) and after dynamic cache writes. The pattern: `const keys = await cache.keys(); if (keys.length > MAX) { await cache.delete(keys[0]); }`.

**Q (Medium): What does `navigator.storage.estimate()` return and what are its limitations?**

Answer: It returns a Promise resolving to `{ quota, usage }` in bytes — `quota` is the approximate storage allocation for this origin, `usage` is how much is currently in use. Optionally it includes `usageDetails` breaking down usage by API (IndexedDB, caches, etc.), but this is not universally supported. Limitations: the quota is approximate, not a hard limit; the browser may grant more or less based on device conditions at the time. Opaque response accounting may not accurately appear in `usage`. Use it for monitoring and planning (alert when >80% full), not for exact capacity planning.

**Q (Low): If a PWA requests persistent storage but the browser denies it, what should the app do?**

Answer: Show the user an informational message explaining that cached data and offline functionality may be lost if the device runs low on storage, and advise them to clear other apps or browser data if they want to ensure persistence. The app should still function — just with best-effort storage guarantees. Optionally, implement a sync mechanism that uploads local data to a server on every meaningful change, so eviction doesn't mean permanent data loss. Never silently fail — the user should understand the reliability of the data they're creating.

---

## Self-Assessment

- [ ] Explain the difference between best-effort and persistent storage and when the browser evicts each
- [ ] Describe the opaque response quota inflation problem and its fix
- [ ] Write a code snippet that queries storage usage and warns when over 80% full
- [ ] Explain what `navigator.storage.persist()` does and what factors affect whether it's granted
- [ ] Describe how to implement max-entries cache trimming in a Service Worker

---
*Next: PWA — App Shell, Background Sync, Web Push — how Progressive Web Apps use Service Workers to deliver native-app-like capabilities: offline shell, reliable sync, and re-engagement notifications.*
