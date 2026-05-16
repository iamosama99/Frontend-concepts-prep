# Cache API (Service Worker Layer)

## Quick Reference

| Operation | Method | What it does |
|---|---|---|
| Store a response | `cache.put(request, response)` | Maps a Request → Response in a named cache |
| Fetch and store | `cache.add(url)` / `cache.addAll([urls])` | Fetches then stores; rejects on non-2xx |
| Retrieve | `cache.match(request)` | Returns matching Response, or `undefined` |
| Cross-cache match | `caches.match(request)` | Searches all caches; returns first match |
| Delete an entry | `cache.delete(request)` | Removes a specific Request entry |
| List caches | `caches.keys()` | Returns array of named cache names |

---

## What Is This?

The Cache API is a programmable HTTP response cache. It stores `Request → Response` pairs in named caches — buckets you can open, populate, query, and delete by name. Unlike the browser's built-in HTTP cache (which is invisible and automatically managed by `Cache-Control` headers), the Cache API is fully under developer control.

```js
// Open a named cache, store a response, retrieve it
const cache = await caches.open('v1-static');

await cache.addAll(['/app.js', '/styles.css', '/offline.html']);

const response = await cache.match('/app.js');
if (response) {
  console.log('Cache hit'); // serve from cache
}
```

It's the mechanism behind every Service Worker caching strategy covered in Phase 7.2. But it's also available from `window` — you can use it outside a Service Worker.

> **Check yourself:** If `cache.add('/api/data')` receives a 404 response, does it store the 404, or reject?

---

## Why Does It Exist?

Before the Cache API, web developers had no scriptable control over what was cached. The browser's HTTP cache was opaque: you could hint at it with `Cache-Control` headers, but you couldn't preload specific resources, serve them offline, or invalidate specific entries from JavaScript.

The Cache API was designed alongside Service Workers to give developers a fully programmable caching layer that Service Workers can use to implement custom strategies — cache-first, network-first, stale-while-revalidate. It maps naturally to the Service Worker fetch model: intercept a Request, find a matching Response in the Cache, decide what to do.

---

## How It Works

### Cache entries are `Request → Response` pairs

Everything is typed. A cache stores `Request` objects (not just URL strings) mapped to `Response` objects (not just bodies):

```js
const cache = await caches.open('my-cache');

// All three are equivalent for simple URL-based entries
await cache.add('/script.js');
await cache.put('/script.js', await fetch('/script.js'));
await cache.put(new Request('/script.js'), await fetch('/script.js'));
```

### Matching semantics

By default, `cache.match()` matches on URL + method. Query strings and fragment identifiers are included in the match. Headers are NOT included in the default match (request bodies are also ignored — no POST caching by default).

```js
const response = await cache.match('/data?page=2');
// Only matches if a response was stored with that exact URL

// Ignore query string during match
const response = await cache.match('/data?page=2', { ignoreSearch: true });
// Matches any stored /data URL regardless of query string
```

`caches.match(request)` searches all named caches in the order they were created. Useful in Service Worker fetch handlers when you don't know which cache has the resource.

### `cache.add` vs `cache.put`

```js
// cache.add: fetches the URL and stores it. Rejects on non-2xx response.
await cache.add('/logo.png'); // throws if server returns 404

// cache.put: stores any Request→Response pair, including error responses.
// You control what gets stored.
const response = await fetch('/logo.png');
if (response.ok) {
  await cache.put('/logo.png', response.clone()); // only store successes
}
```

`cache.addAll` is atomic — if any fetch fails, nothing is stored:

```js
// If /critical.js is unavailable, nothing gets cached (no partial state)
await cache.addAll(['/app.js', '/styles.css', '/critical.js']);
```

### The `response.clone()` requirement

A `Response` body is a stream — readable exactly once. If you send a response to the browser via `event.respondWith(response)` and also cache it with `cache.put(request, response)`, the second consumer gets an empty body. Always clone before caching:

```js
self.addEventListener('fetch', (event) => {
  event.respondWith(
    fetch(event.request).then(async (response) => {
      const cache = await caches.open('dynamic');
      cache.put(event.request, response.clone()); // clone for cache
      return response;                            // original to browser
    })
  );
});
```

> **Check yourself:** Why does the Cache API store `Request` objects rather than just URL strings?

---

## Cache Versioning and Cleanup

Each named cache is independent. A common pattern: version cache names so deploys can use a new cache without conflicting with an old one:

```js
const CACHE_NAME = 'static-v4';

// In install event: populate new cache
self.addEventListener('install', (e) => {
  e.waitUntil(
    caches.open(CACHE_NAME).then(cache => cache.addAll(STATIC_ASSETS))
  );
});

// In activate event: delete old caches
self.addEventListener('activate', (e) => {
  e.waitUntil(
    caches.keys().then(keys =>
      Promise.all(
        keys
          .filter(key => key !== CACHE_NAME && key.startsWith('static-'))
          .map(key => caches.delete(key))
      )
    )
  );
});
```

Without cache cleanup, old caches accumulate and consume storage quota. The pattern: activate always deletes caches from previous versions.

---

## Cache API Outside Service Workers

The Cache API is available from `window` (main thread context) in all modern browsers. This allows pre-caching assets from the page before a Service Worker is registered, or building caching logic that doesn't require a Service Worker at all:

```js
// In main.js — preload next page's assets
const cache = await caches.open('prefetch');
await cache.addAll(['/page-2.js', '/page-2.css']);
```

In practice, most Cache API usage is in Service Workers, but the option exists for non-SW caching workflows.

---

## Differences from the Browser HTTP Cache

| | Browser HTTP Cache | Cache API |
|---|---|---|
| Control | Headers (`Cache-Control`, `ETag`) | Fully scriptable |
| Visibility | Opaque to JS | Fully readable by JS |
| Offline | Not designed for it | Designed for offline |
| Keys | URL + Vary headers | `Request` object (URL + method by default) |
| Capacity | Browser-managed | Quota-governed (like IndexedDB) |
| Invalidation | `Cache-Control: no-cache`, `ETag` revalidation | `cache.delete(request)` |

They're complementary: the HTTP cache handles standard HTTP semantics automatically; the Cache API handles offline-first and custom strategies. A Service Worker can bypass the HTTP cache entirely:

```js
fetch(request, { cache: 'no-store' }) // bypass HTTP cache when fetching for the cache API store
```

---

## Opaque Responses

Cross-origin responses fetched without CORS are "opaque" — `response.status === 0`, `response.type === 'opaque'`, content hidden. Storing an opaque response in the Cache API is risky:

```js
const response = await fetch('https://cdn.example.com/script.js'); // no CORS
// response.status === 0 — could be a success OR a CDN error page
if (response.ok) { /* Never true for opaque responses */ }
await cache.put('https://cdn.example.com/script.js', response); // dangerous!
```

If the server returned an error, you've cached an error. The browser adds a padding of `~7MB` to each opaque response in quota accounting (to prevent quota-based fingerprinting attacks), which inflates your storage consumption massively.

Fix: always use CORS (`crossorigin` attribute on elements or `{mode: 'cors'}` on fetch) for resources you intend to cache with the Cache API.

---

## Gotchas

**1. `cache.add()` rejects on non-2xx responses — `cache.put()` does not**
`add` is the safe, auto-rejecting version. `put` stores whatever you give it, including 404s and 500s. Use `put` with an explicit `response.ok` check to avoid storing error pages.

**2. Opaque responses massively inflate quota**
Each opaque response costs ~7MB in quota accounting regardless of actual size. Avoid caching cross-origin resources without CORS.

**3. No automatic eviction within a named cache**
The Cache API doesn't age out entries. If you cache API responses dynamically, they stay until you delete them. Implement a `maxEntries` or `maxAge` policy in your Service Worker or your cache will grow indefinitely.

**4. `response.clone()` before both use and cache**
The one-read stream constraint bites regularly. Set up a mental checklist: "do I use this response AND store it?" → must clone.

**5. Cache entries survive SW registration changes**
Caches are per-origin, not per-SW-script. If you rename your SW file and unregister the old one, the old caches still exist and consume quota until deleted. Always clean up in `activate`.

**6. `caches.match()` order is unspecified**
When multiple caches have the same URL, `caches.match()` returns the first match but the order caches are searched is not guaranteed by spec. Use `cache.match()` on a specific named cache if order matters.

---

## Interview Questions

**Q (High): What is the Cache API and how does it differ from the browser's HTTP cache?**

Answer: The Cache API is a developer-controlled, scriptable store of `Request → Response` pairs organized in named caches. It's fully opaque to HTTP semantics — you decide what goes in, what comes out, and when entries expire. The browser's HTTP cache is automatic and governed by `Cache-Control`, `ETag`, and `Last-Modified` headers — you can hint at its behavior but can't directly read or write entries from JavaScript. The Cache API is designed for offline-first scenarios where you need guaranteed, predictable access to specific resources regardless of network state. The two are complementary: you can use both simultaneously, or bypass the HTTP cache when fetching resources to populate the Cache API.

**Q (High): Why is `response.clone()` necessary before caching? What goes wrong without it?**

Answer: A `Response` body is a `ReadableStream` that can only be consumed once. If you pass the same `Response` to `event.respondWith(response)` (to send to the browser) and `cache.put(request, response)` (to cache), whichever reads the body first leaves the other with an empty stream. The browser receives an empty response or the cache stores an empty body. `response.clone()` creates a second `Response` backed by a tee of the same stream — both consumers get the full body. The pattern is always: `cache.put(request, response.clone()); return response;`.

**Q (Medium): What happens when `cache.addAll()` is given a URL that returns a 404?**

Answer: `cache.addAll()` is atomic — it rejects the entire operation if any URL fails to fetch successfully (non-2xx status). Nothing is stored. This is intentional: the install event uses `addAll` to pre-cache a set of critical assets, and a partial cache (some assets present, some missing) would result in a broken offline experience. The Service Worker's install fails, and the browser retries on the next navigation. For dynamic caching where partial failure is acceptable, use `cache.put()` with explicit status checks instead of `cache.addAll()`.

**Q (Medium): What is an opaque response and why is it dangerous to cache?**

Answer: An opaque response is the result of a cross-origin fetch without CORS — its status is always 0, type is `'opaque'`, and body content is hidden from JavaScript. You can't tell if it's a success or a server error. Caching an opaque error response means you'll serve that error forever. Additionally, browsers account for opaque responses with a ~7MB quota penalty each (to prevent storage-based timing attacks), so caching many cross-origin resources without CORS can exhaust your quota. The fix: add `crossorigin` to HTML elements loading cross-origin resources, or use `fetch(url, { mode: 'cors' })` for resources you intend to cache.

**Q (Low): Can you use the Cache API from the main thread (outside a Service Worker)?**

Answer: Yes. `caches` is available as a global on `window` in all modern browsers, not just in Service Workers. You can pre-populate caches from the page, query cached resources, or implement a simple caching layer without a Service Worker. The use cases: pre-caching next-page assets from the page JS (ahead of a Service Worker taking control), or building cache-based features in browsers where SW is restricted. In practice, most Cache API usage belongs in the Service Worker where the fetch event gives you the right intercept hook, but the API is not SW-exclusive.

---

## Self-Assessment

- [ ] Explain the difference between `cache.add()`, `cache.addAll()`, and `cache.put()`
- [ ] Explain why `response.clone()` is necessary and what breaks without it
- [ ] Describe what an opaque response is and the two risks of caching one
- [ ] Write a Service Worker activate handler that deletes previous version caches
- [ ] Explain how the Cache API differs from the browser's HTTP cache

---
*Next: Storage Quotas & Eviction Policies — how browsers allocate storage across APIs, how you query and request persistent storage, and what happens when you run out.*
