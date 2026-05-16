# History API vs. Hash Routing

## Quick Reference

| | History API | Hash Routing |
|---|---|---|
| URL form | `/products/42` | `/#/products/42` |
| Browser support | IE10+ | All |
| Server requirement | Must serve index.html for all paths | None — server never sees the hash |
| Deep link works? | Only with server config | Always |
| SEO | Crawlable by default | Hash fragments ignored by crawlers |
| API | `pushState`, `replaceState`, `popstate` | `hashchange`, `location.hash` |

---

## What Is This?

Single-page applications (SPAs) must simulate navigation without triggering full-page reloads. Two browser mechanisms support this:

**History API** — manipulates the real URL path using `pushState`/`replaceState`. The browser displays `/products/42` but does not send a new HTTP request. The server must serve `index.html` for all paths so a direct visit or refresh still loads the app.

**Hash routing** — stores the route in the URL fragment (`#/products/42`). The browser never sends the fragment to the server, so any server config works. JavaScript reads `location.hash` to determine the active route.

```
History API:                   Hash Routing:
User navigates to /products    User navigates to /#/products
    │                              │
    ├─ pushState('/products')      ├─ location.hash = '#/products'
    ├─ URL bar: /products          ├─ URL bar: /#/products
    ├─ No HTTP request made        ├─ No HTTP request made
    └─ 'popstate' fires on back    └─ 'hashchange' fires on back
```

> **Check yourself:** Why does History API routing require server-side configuration that hash routing does not?

---

## Why It Matters

The routing mechanism determines URL aesthetics, SEO behavior, server requirements, and how external links behave. Most modern frameworks default to History API routing, but understanding when hash routing is appropriate (and why) is a real senior-level question.

---

## History API

### Core methods

```js
// Navigate to a new route (pushes onto the history stack)
history.pushState({ userId: 42 }, '', '/users/42');

// Replace current entry (no new history entry — good for redirects)
history.replaceState({ userId: 42 }, '', '/users/42');

// Go back, forward, or N steps
history.back();
history.forward();
history.go(-2);
```

The second argument (`title`) is ignored by all browsers and can be `''`.

### Listening for navigation

```js
// Fires on popstate (back/forward), NOT on pushState/replaceState calls
window.addEventListener('popstate', (event) => {
  const state = event.state; // whatever you passed to pushState
  renderRoute(location.pathname);
});

// To intercept all link clicks (not just back/forward):
document.addEventListener('click', (event) => {
  const link = event.target.closest('a[href]');
  if (!link) return;
  const url = new URL(link.href);
  if (url.origin !== location.origin) return; // external — let it go

  event.preventDefault();
  history.pushState({}, '', url.pathname);
  renderRoute(url.pathname);
});
```

### The server config requirement

When the user refreshes `/products/42`, the browser sends `GET /products/42` to the server. If the server doesn't know about this path, it returns 404. The fix: serve `index.html` for every path and let the JS router handle the rest.

Nginx:
```nginx
location / {
  try_files $uri $uri/ /index.html;
}
```

Express:
```js
app.get('*', (req, res) => {
  res.sendFile(path.join(__dirname, 'dist', 'index.html'));
});
```

This is the single biggest footgun when deploying an SPA — the app works in development (Vite/webpack-dev-server handles it automatically) but returns 404s on production servers that weren't configured for it.

---

## Hash Routing

### How it works

```js
// Set the route
location.hash = '#/products/42';

// Read the route
const route = location.hash.slice(1); // '/products/42'

// Listen for changes
window.addEventListener('hashchange', () => {
  renderRoute(location.hash.slice(1));
});

// Also handle initial load (hashchange doesn't fire on first load)
document.addEventListener('DOMContentLoaded', () => {
  renderRoute(location.hash.slice(1) || '/');
});
```

### Building a minimal hash router

```js
class HashRouter {
  constructor(routes) {
    this.routes = routes; // { '/': HomeComponent, '/about': AboutComponent, ... }
    window.addEventListener('hashchange', () => this.render());
    this.render(); // handle initial load
  }

  navigate(path) {
    location.hash = path;
  }

  render() {
    const path = location.hash.slice(1) || '/';
    const Component = this.routes[path] || this.routes['/404'];
    document.getElementById('app').innerHTML = Component();
  }
}

const router = new HashRouter({
  '/': () => '<h1>Home</h1>',
  '/about': () => '<h1>About</h1>',
  '/404': () => '<h1>Not Found</h1>',
});
```

---

## Building a Minimal History API Router

```js
class Router {
  constructor(routes) {
    this.routes = routes;
    window.addEventListener('popstate', () => this.render());
    this.interceptLinks();
    this.render();
  }

  navigate(path) {
    history.pushState({}, '', path);
    this.render();
  }

  redirect(path) {
    history.replaceState({}, '', path);
    this.render();
  }

  interceptLinks() {
    document.addEventListener('click', (e) => {
      const link = e.target.closest('a[href]');
      if (!link) return;
      const url = new URL(link.href, location.origin);
      if (url.origin !== location.origin) return;
      e.preventDefault();
      this.navigate(url.pathname);
    });
  }

  render() {
    const path = location.pathname;
    // Find matching route (exact, then prefix)
    const handler = this.routes[path] || this.matchDynamic(path) || this.routes['/404'];
    document.getElementById('app').innerHTML = handler(this.extractParams(path));
  }

  matchDynamic(path) {
    for (const [pattern, handler] of Object.entries(this.routes)) {
      if (!pattern.includes(':')) continue;
      const regex = new RegExp('^' + pattern.replace(/:([^/]+)/g, '([^/]+)') + '$');
      if (regex.test(path)) return handler;
    }
    return null;
  }

  extractParams(path) {
    for (const [pattern] of Object.entries(this.routes)) {
      if (!pattern.includes(':')) continue;
      const keys = [...pattern.matchAll(/:([^/]+)/g)].map(m => m[1]);
      const regex = new RegExp('^' + pattern.replace(/:([^/]+)/g, '([^/]+)') + '$');
      const match = path.match(regex);
      if (match) {
        return Object.fromEntries(keys.map((k, i) => [k, match[i + 1]]));
      }
    }
    return {};
  }
}
```

---

## URL State Object

`pushState` and `replaceState` accept a state object that persists across back/forward navigation:

```js
history.pushState(
  { scrollPosition: 400, filters: { category: 'shoes' } }, // state
  '',
  '/products?category=shoes'
);

window.addEventListener('popstate', (event) => {
  if (event.state?.scrollPosition) {
    window.scrollTo(0, event.state.scrollPosition);
  }
  if (event.state?.filters) {
    applyFilters(event.state.filters);
  }
  renderRoute(location.pathname);
});
```

The state object is serialized by the browser and survives page sessions in the forward/back cache. Maximum size varies by browser (~640KB in Chrome). Do not store large data here — store identifiers and re-fetch or re-derive data.

---

## When to Use Hash Routing

Hash routing is appropriate when:
- You cannot control server configuration (GitHub Pages without Jekyll, S3 static hosting without redirect rules, embedded apps in third-party iframes)
- The app is embedded in an environment where URL path changes would confuse the host application
- You need IE9 support (rare but exists in enterprise)

In all other modern web apps, History API routing is standard. All major frameworks (React Router, Vue Router, Angular Router, SvelteKit) default to History API routing.

---

## Gotchas

**1. `popstate` does not fire on `pushState`/`replaceState` calls**
You must call your render function manually after programmatic navigation. `popstate` only fires for browser-native back/forward.

**2. Hash routing breaks anchor links**
If you use hash routing (`#/path`) and also use hash anchors (`#section-id`) within a page, they conflict. Both use the fragment. This is a fundamental design limitation of hash routing.

**3. History state is lost on hard refresh**
`location.state` (from `history.pushState`) is not available after a hard refresh in most browsers. Don't rely on it for data that must survive refreshes — use URL params or storage.

**4. Memory leaks from event listeners**
If your router is created per-page-load (e.g., in a framework component), adding `popstate` listeners in a constructor without cleanup leads to multiple listeners stacking up. Clean up on unmount.

**5. Relative vs. absolute paths**
`history.pushState({}, '', 'products')` (no leading slash) navigates relative to the current path — it's `./products`, not `/products`. Always use absolute paths with leading `/` to avoid subtle bugs in deeply nested routes.

---

## Interview Questions

**Q (High): Explain the difference between History API routing and hash routing. When would you choose each?**

Answer: History API routing uses `pushState`/`replaceState` to modify the full URL path (`/products/42`) without a page reload. The browser shows a clean URL, but the server must be configured to serve `index.html` for all paths because a direct visit or refresh sends a real HTTP request to that path. Hash routing stores the route in the URL fragment (`#/products/42`), which the browser never sends to the server, so no server config is needed. Choose History API routing for any modern app where you control the server — it produces cleaner URLs, works better with SEO, and is the default in all major frameworks. Choose hash routing for apps deployed to static hosts without redirect capabilities (S3 without CloudFront rules, GitHub Pages) or embedded environments where you can't control URL paths.

**Q (High): Why does `popstate` not fire after `pushState`? How do you handle programmatic navigation?**

Answer: `popstate` is a browser event that fires when the active history entry changes due to user action (back/forward buttons) or `history.go()`. It does not fire for `pushState` or `replaceState` because those calls change the history stack without "navigating" to an existing entry — they add or update entries. After a `pushState`, the browser has already updated the URL; the app must call its own render function explicitly. The standard pattern: wrap `pushState` in a router `navigate(path)` method that calls both `history.pushState()` and the internal render function, so navigation is always complete regardless of whether it originated from a link click or a programmatic call.

**Q (Medium): What is the state object in `pushState` and what are its limitations?**

Answer: The state object is a serializable JavaScript value stored with each history entry and restored when the user navigates to that entry (via `event.state` in the `popstate` handler). It's useful for storing UI context alongside the URL — scroll position, expanded filters, previous route data — without encoding it in the URL string. Limitations: it must be serializable (no functions, class instances, or DOM references); browsers impose a size cap (Chrome allows ~640KB); it's not accessible after a full page refresh in some cases; and the state is tab-specific, not shared. Use it for ephemeral UI state, not canonical application data.

**Q (Medium): How do you intercept link clicks to prevent full-page reloads in a History API router?**

Answer: Add a delegated `click` listener on `document` that checks if the clicked element is (or is inside) an `<a>` tag with an `href`. Parse the `href` with `new URL(link.href)` — this handles relative, absolute, and protocol-relative URLs correctly. If the origin matches `location.origin` (same domain), call `event.preventDefault()` and use `history.pushState` + your render function. If the origin differs (external link) or it has a `target="_blank"`, let the browser handle it normally. Also check for `download`, `rel="external"`, and modifier keys (Ctrl/Cmd+click for new tab) — if any are present, don't intercept.

**Q (Low): Why does hash routing conflict with in-page anchor links?**

Answer: Both hash routing (`#/products`) and HTML anchor links (`<a href="#section-id">`) use the URL fragment identifier. They're the same part of the URL. If your router owns `location.hash` to represent routes, there's no clean way to also use hash anchors to scroll to sections within a page — changing the hash for anchoring would trigger your route handler. Some libraries work around this by treating hashes starting with `/` as routes and others as anchors, but this is a known limitation. History API routing avoids this entirely by using the URL pathname, leaving the fragment available for genuine in-page anchors.

---

## Self-Assessment

- [ ] Explain how `pushState` modifies the URL without triggering a page reload
- [ ] Describe why History API routing requires server-side "catch-all" configuration
- [ ] Explain why `popstate` doesn't fire on `pushState` and how to handle it
- [ ] Implement a link-click interceptor that handles external links correctly
- [ ] Describe two scenarios where hash routing is the right choice over History API routing

---
*Next: File-system Based vs. Config-based Routing — two different philosophies for declaring routes in modern frameworks.*
