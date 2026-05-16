# Caching Layers: HTTP Cache, Service Worker Cache & CDN

## Quick Reference

| Layer | Where | Controls | Scope |
|---|---|---|---|
| Browser HTTP cache | User's browser | `Cache-Control`, `ETag`, `Last-Modified` headers | Per-browser, per-origin |
| Service Worker cache | Browser (SW context) | Your JavaScript code | Per-origin, offline-capable |
| CDN cache | Edge nodes near users | `Cache-Control`, `Surrogate-Control`, cache tags | Shared across all users |

## What Is This?

Caching is about storing a response so a future identical request can be served without the full round-trip to the origin server. Multiple independent caching layers exist between a user's request and your server — each layer has different rules for what gets cached, for how long, and for whom.

The three main layers a frontend engineer needs to understand:

1. **Browser HTTP cache:** Built into every browser, controlled entirely by HTTP response headers. The browser decides whether to serve from cache before making any network request.

2. **Service Worker cache:** A programmable cache living in the browser's Service Worker context. Unlike the HTTP cache, you write the JavaScript that decides what to cache and how to serve it.

3. **CDN cache:** Geographically distributed edge servers that cache responses for all users. A request from Tokyo can be served from a Tokyo edge node rather than a server in Virginia.

> **Check yourself:** A browser's HTTP cache has a response for `/api/user`. A Service Worker is also registered. Which layer gets to handle the request first?

## HTTP Cache

The browser's HTTP cache is the first layer. Every HTTP response can include headers that instruct the browser (and intermediate caches) how to cache the response.

### `Cache-Control` Directive Anatomy

```http
Cache-Control: public, max-age=3600, stale-while-revalidate=60
```

Key directives:

| Directive | Meaning |
|---|---|
| `public` | Response can be cached by browsers and CDNs |
| `private` | Only the requesting browser can cache it (not CDNs) |
| `no-store` | Never cache — always fetch from origin |
| `no-cache` | Cache, but must revalidate with server before using |
| `max-age=N` | Cache for N seconds from request time |
| `s-maxage=N` | CDN cache duration (overrides `max-age` for shared caches) |
| `immutable` | Content will never change — safe to cache until max-age expires without revalidation |
| `stale-while-revalidate=N` | Serve stale for N seconds while revalidating in background |

### Validation: ETags and Last-Modified

When `max-age` expires, the browser can ask the server "has this changed?" rather than re-downloading the full response:

**ETag:** Server generates a hash or version identifier for the response content:
```http
ETag: "abc123hash"
```
Browser sends on next request:
```http
If-None-Match: "abc123hash"
```
If unchanged, server responds `304 Not Modified` with no body. Browser uses cached copy.

**Last-Modified:**
```http
Last-Modified: Tue, 15 May 2025 12:00:00 GMT
```
Browser sends:
```http
If-Modified-Since: Tue, 15 May 2025 12:00:00 GMT
```
Server responds `304` if unchanged, or `200` with new body if changed.

ETags are more reliable than Last-Modified — timestamps have 1-second granularity and can be wrong if multiple servers serve the same file (different system clocks).

### Cache Busting for Static Assets

Content-hashed filenames are the standard approach for static assets:
```
/static/bundle.js           → browser caches, must expire to pick up changes
/static/bundle.abc123.js    → content-hashed filename; when content changes, the hash changes
```

With content-hashed filenames: serve `Cache-Control: public, max-age=31536000, immutable` (one year). The browser caches forever — and it's correct to do so, because if the file ever changes, it gets a new URL. The HTML that references the asset has `no-cache` or a short max-age, so the browser always gets the current HTML with the current asset URLs.

## Service Worker Cache

The Service Worker (SW) is a JavaScript worker that runs in the background, independent of the page, and acts as a programmable network proxy. Every request from your origin can be intercepted, and you decide: serve from cache, go to network, serve stale while revalidating, etc.

### Basic Cache Setup

```javascript
// service-worker.js
const CACHE_NAME = 'app-v1';
const PRECACHE_ASSETS = [
  '/',
  '/offline.html',
  '/static/bundle.abc123.js',
  '/static/styles.abc123.css',
];

// Install: precache essential assets
self.addEventListener('install', (event) => {
  event.waitUntil(
    caches.open(CACHE_NAME).then(cache => cache.addAll(PRECACHE_ASSETS))
  );
  self.skipWaiting();  // activate immediately, don't wait for old SW to stop
});

// Activate: clean up old caches
self.addEventListener('activate', (event) => {
  event.waitUntil(
    caches.keys().then(keys =>
      Promise.all(
        keys.filter(key => key !== CACHE_NAME)
            .map(key => caches.delete(key))
      )
    )
  );
  self.clients.claim();  // take control of all open pages immediately
});
```

### Fetch Strategies

The SW's `fetch` event handler is where you implement caching strategies:

**Cache-first (offline-first):** Best for static assets and rarely-changing content.
```javascript
self.addEventListener('fetch', (event) => {
  event.respondWith(
    caches.match(event.request).then(cached => {
      return cached || fetch(event.request).then(response => {
        // Cache new resources as they're fetched
        const clone = response.clone();
        caches.open(CACHE_NAME).then(cache => cache.put(event.request, clone));
        return response;
      });
    })
  );
});
```

**Network-first:** Best for API calls that need to be fresh but should degrade gracefully offline.
```javascript
self.addEventListener('fetch', (event) => {
  event.respondWith(
    fetch(event.request)
      .then(response => {
        const clone = response.clone();
        caches.open(CACHE_NAME).then(cache => cache.put(event.request, clone));
        return response;
      })
      .catch(() => caches.match(event.request))  // fallback to cache on network failure
  );
});
```

**Stale-while-revalidate:** Best for resources where showing stale is acceptable but freshness is desired.
```javascript
self.addEventListener('fetch', (event) => {
  event.respondWith(
    caches.open(CACHE_NAME).then(async cache => {
      const cached = await cache.match(event.request);
      const fetchPromise = fetch(event.request).then(response => {
        cache.put(event.request, response.clone());
        return response;
      });
      return cached || fetchPromise;  // return cached immediately if available
    })
  );
});
```

### SW Cache vs. HTTP Cache

These are separate caches. The SW cache is not the same as the browser's HTTP cache. When a SW intercepts a request and calls `fetch(event.request)`, that sub-request goes through the HTTP cache. When a SW returns from `caches.match()`, it bypasses the HTTP cache entirely.

Important: if you want the SW to be the authoritative cache, configure your assets with `Cache-Control: no-store` so the HTTP cache doesn't also cache them — otherwise you have two caches to manage.

## CDN Cache

A Content Delivery Network is a geographically distributed network of "edge" servers. When a user requests your site, DNS routes them to the nearest edge. If the edge has a cached response, it serves it without reaching your origin server — potentially saving 100–500ms of latency and completely bypassing your server's compute.

### How CDN Caching Works

```
Tokyo user → Tokyo CDN edge node
  → Cache HIT: serve from edge (~5ms TTFB)
  → Cache MISS: fetch from origin (US Virginia, ~200ms), cache it, serve it
```

The CDN respects `Cache-Control` headers from your origin. `Cache-Control: public, max-age=3600` tells the CDN to cache for one hour. `Cache-Control: private` tells the CDN not to cache (only the user's browser can).

`s-maxage` overrides `max-age` for CDNs (shared caches) while letting browsers use `max-age`:
```http
Cache-Control: public, max-age=60, s-maxage=3600
# Browser caches for 60 seconds
# CDN caches for 1 hour
```

### Cache Invalidation

CDN-cached responses can't be invalidated by changing the content — the edge has a copy and serves it until `s-maxage` expires. To force freshness before expiry:

1. **Content-hashed URLs:** Change the URL when content changes (standard for static assets)
2. **CDN purge API:** Most CDN providers expose an API to purge specific URLs or cache tags programmatically — call this on deploy to invalidate old responses
3. **Cache tags (surrogate keys):** Assign semantic tags to responses (`Cache-Tag: product-123`), then purge all responses tagged `product-123` when that product changes — this enables targeted invalidation without purging unrelated content

### Layered Caching Strategy

A production caching strategy layers all three:

```
Static assets (JS, CSS, fonts, images):
  → Cache-Control: public, max-age=31536000, immutable
  → CDN: caches forever (URL changes when content changes)
  → Browser: caches forever (same reason)
  → SW: precache in install event — offline-capable

HTML documents:
  → Cache-Control: public, max-age=0, s-maxage=300, stale-while-revalidate=60
  → CDN: cache 5 minutes, serve stale for 60 seconds while revalidating
  → Browser: always revalidate (no-store or max-age=0)
  → SW: network-first with cache fallback

API responses (public data):
  → Cache-Control: public, max-age=60, s-maxage=600
  → CDN: cache 10 minutes
  → Browser: cache 1 minute

API responses (personalized data):
  → Cache-Control: private, max-age=0
  → CDN: don't cache (private — would leak one user's data to another)
  → Browser: always revalidate
```

## Gotchas

**`Cache-Control: no-cache` does not mean "don't cache."** It means "cache but always revalidate with the server before using." `no-store` means don't cache at all. This is a perennial source of confusion — `no-cache` allows the browser to cache the response, it just must check for freshness every time.

**ETags must be consistent across server instances.** If you have multiple web servers and each generates its own ETag hash from memory, the same file may have different ETags on different servers. The browser sends `If-None-Match` with the ETag from server A; the request hits server B, which has a different ETag, responds 200 with the full body — unnecessary download. ETags must be based on file content (content-hash), not server-specific state.

**Service Worker updates are delayed.** A new SW only activates after all open tabs using the old SW are closed. If you deploy a new SW and a user has your site open in five tabs, the new SW waits until all five close. Use `skipWaiting()` + `clients.claim()` in the install/activate events to take control immediately — but be aware this can cause version inconsistency if tabs run different code simultaneously.

**CDN purge propagation is not instant.** After calling a CDN's purge API, the invalidation signal propagates across edge nodes — this takes seconds to a few minutes globally. Users in regions where the signal arrives later may still receive the old cached response during this window.

**Service Workers can only serve requests on the same origin.** A SW registered at `https://example.com` can only intercept requests from `https://example.com`. Cross-origin requests (CDN-hosted assets, third-party APIs) are not interceptable.

## Interview Questions

**Q (High): What is the difference between `Cache-Control: no-cache` and `Cache-Control: no-store`?**

Answer: `no-cache` means "you may cache this response, but you must revalidate with the server before serving it from cache." The browser stores the response and sends a conditional request on the next access — `If-None-Match` with the ETag, or `If-Modified-Since`. If unchanged, the server responds `304 Not Modified` and the browser uses the cached copy. The win: only the headers are exchanged if unchanged, not the full body. `no-store` means "do not cache at all — every request must go to origin." Nothing is stored — no disk, no memory. Use `no-cache` for frequently-changing content where you want the cache entry for conditional requests (saves bandwidth on unchanged responses). Use `no-store` for sensitive data that should never be persisted (authentication responses, private user data).

The trap: Thinking `no-cache` means the browser won't cache. It's the opposite — it caches but always validates.

---

**Q (High): What are content-hashed filenames and why are they the correct approach for caching static assets?**

Answer: Content-hashed filenames incorporate a hash of the file's content into the filename: `bundle.abc123def.js`. This means the URL changes whenever the content changes. With stable URLs (like `bundle.js`), you can't cache aggressively — if you set `max-age=31536000` (one year), browsers keep the old file for a year after you deploy new code. You'd have to set a short max-age (creating unnecessary revalidation requests) or manually bust the cache. With content-hashed filenames, the correct strategy is `Cache-Control: public, max-age=31536000, immutable`: cache forever, because if the file changes, it's at a new URL. The HTML document that references the assets has a short cache or `no-cache` — it always fetches the current HTML, which contains the current hashed asset URLs. This is how all production build tools (Vite, Webpack, etc.) work by default.

The trap: Explaining content hashing without explaining the HTML → assets cache strategy relationship. The HTML must not be aggressively cached, or users won't pick up the new asset URLs.

---

**Q (High): What is the Service Worker cache and how does it differ from the browser HTTP cache?**

Answer: The browser HTTP cache is automatic — it caches based on response headers, the developer has no direct control over what gets cached or how it's served. The Service Worker cache is programmable — you write JavaScript in the SW's `fetch` event handler that explicitly decides: go to network, serve from cache, serve stale while revalidating, or return a custom offline response. The SW cache uses the Cache Storage API (`caches.open()`, `cache.put()`, `cache.match()`) and persists independently of the HTTP cache. Critically, the SW runs as a network proxy — every request from your origin goes through the SW's `fetch` handler, allowing you to intercept navigation requests, return pre-cached HTML, and serve the app offline. The HTTP cache can't intercept navigation requests or serve content when the server is unreachable — the SW can. This is what enables offline-capable Progressive Web Apps.

The trap: Thinking the SW and HTTP cache are the same thing. They're completely separate caches with different control mechanisms and different capabilities.

---

**Q (Medium): Explain the layered caching strategy for a production web app — what cache headers would you set for HTML documents vs. static assets vs. API responses?**

Answer: Static assets (JS, CSS, images): `Cache-Control: public, max-age=31536000, immutable`. Content-hashed filenames mean URLs change when content changes, so it's safe to cache forever. HTML documents: `Cache-Control: no-cache` or `max-age=0, s-maxage=300`. HTML must be revalidated to pick up updated asset URLs, but CDN can cache for 5 minutes for performance. Authenticated/personalized API responses: `Cache-Control: private, no-store`. CDN must not cache — sharing one user's response with another is a data leak. Public API responses: `Cache-Control: public, max-age=60, s-maxage=600`. Browser caches briefly; CDN caches longer. The underlying principle: cache aggressively where content is shared and stable; prevent caching where content is user-specific or highly volatile.

The trap: Applying the same cache policy to everything. The distinction between content-hashed static assets (cache forever), HTML (don't cache or short cache), and personalized responses (private, no CDN) is the testable substance.

---

**Q (Low): Why is ETags consistency important in a multi-server deployment and how do you ensure it?**

Answer: ETags are validation tokens sent in responses that clients include in conditional requests (`If-None-Match`) to ask "has this changed?" If the server generates ETags based on in-memory state or timestamps rather than content, different server instances may generate different ETags for the same file. A client receives an ETag from server A, sends a conditional request to server B, which has a different ETag — server B responds 200 with the full body even though the content hasn't changed. This defeats the purpose of ETags (bandwidth savings via 304 responses). The solution: generate ETags as a hash of the response body content. The same content produces the same hash regardless of which server generates it. File-serving middleware (nginx, Express static middleware) typically does this correctly by default. For dynamically generated responses, explicitly compute and return a content-based ETag.

The trap: Not knowing ETags are a per-server concern in multi-server deployments.

---

## Self-Assessment

Before moving on, check off each item you can do WITHOUT looking at the file.

- [ ] Can explain `no-cache` vs. `no-store` precisely (no-cache = cache + always revalidate; no-store = never cache)
- [ ] Can explain why content-hashed filenames allow `max-age=31536000, immutable`
- [ ] Can describe the three SW caching strategies: cache-first, network-first, stale-while-revalidate
- [ ] Can explain how CDN caching works and why `private` responses can't be CDN-cached
- [ ] Can describe the correct cache headers for HTML, static assets, and API responses
- [ ] Can explain why SW cache and HTTP cache are separate and what the SW can do that HTTP cache can't

---
*Next: Payload Optimization — compression, Protocol Buffers, and binary encoding — how to shrink the bytes that travel over the wire.*
