# Client-Side Rendering (CSR)

## Quick Reference

| Concept | Mechanism | Implication |
|---|---|---|
| HTML shell | Server sends nearly empty `<div id="root">` | Browser must wait for JS to paint anything meaningful |
| JS bundle executes | Framework mounts components into the DOM | First paint is blank; content appears after JS runs |
| Routing | JS intercepts navigation, swaps components | No full-page reload; URL changes via History API |
| API fetching | Components fire requests after mount | Data waterfall: JS loads → mount → fetch → render |

## What Is This?

Client-Side Rendering is the rendering model where the server sends a minimal HTML document — essentially a blank shell with a `<script>` tag — and the browser does all the work of building the UI. When the JavaScript bundle downloads and executes, it creates DOM nodes, fetches data from APIs, and produces the final visible page entirely inside the browser.

```html
<!-- What the server actually sends -->
<!DOCTYPE html>
<html>
  <head><title>My App</title></head>
  <body>
    <div id="root"></div>
    <script src="/bundle.js"></script>
  </body>
</html>
```

The `<div id="root">` is genuinely empty. No content, no text, no structure. Everything the user sees is created by JavaScript after the browser finishes downloading, parsing, and executing the bundle.

> **Check yourself:** If a CSR app's JavaScript fails to load, what does the user see? Why does this matter for resilience?

## Why Does It Exist?

Before SPAs, every interaction required a round-trip to the server: click a link → request → server renders HTML → browser parses → page appears. This works but feels slow because every navigation incurs network latency, and the browser has to re-parse the full document.

CSR emerged as a response to the desire for app-like experiences in the browser. Once the initial bundle is loaded, all subsequent navigation happens instantly — the browser already has all the code it needs. The UI updates by swapping out DOM nodes rather than loading new pages. This is why Gmail, Google Maps, and Figma feel like native apps despite running in a browser.

The SPA model also cleanly separates the frontend from the backend: the frontend is a static bundle served from a CDN, and the backend exposes pure data APIs. Teams can deploy independently, scale the static assets separately from the API, and use the same API for mobile apps.

## How It Works

1. **User requests a URL** — browser sends a GET request.
2. **Server responds** with the HTML shell (almost always the same file regardless of the path).
3. **Browser parses HTML**, finds `<script>`, starts downloading the JS bundle.
4. **JS executes** — the framework (React, Vue, etc.) initializes, reads the current URL, and renders the matching route's component tree into `#root`.
5. **Components mount**, fire `useEffect`/`onMounted` callbacks, which trigger API requests.
6. **API responses arrive**, state updates, components re-render with real data.
7. **User navigates** — the router intercepts the click, updates the URL via `history.pushState`, and swaps the rendered component tree. No request to the server.

```javascript
// React example — the full lifecycle in a single component
function UserProfile({ userId }) {
  const [user, setUser] = useState(null);

  useEffect(() => {
    // This fires AFTER the component mounts — after JS runs, after first paint
    // The user sees a spinner (or nothing) during this window
    fetch(`/api/users/${userId}`)
      .then(r => r.json())
      .then(setUser);
  }, [userId]);

  if (!user) return <Spinner />;
  return <div>{user.name}</div>;
}
```

The sequence above is why CSR apps often feel "poppy" on first load: blank shell → spinner → content. Every network waterfall adds a beat to this sequence.

> **Check yourself:** Why does a CSR app with three nested data-fetching components have worse waterfall behavior than one component that fetches three things in parallel?

## Performance Characteristics

**Time to First Byte (TTFB):** Excellent. The server sends a tiny static file immediately, often from a CDN edge node.

**First Contentful Paint (FCP):** Poor. Nothing meaningful paints until the JS bundle downloads and executes. On a slow 3G connection with a 200KB bundle, this can be 5–10 seconds.

**Time to Interactive (TTI):** Tied to FCP since hydration isn't a separate step — the JS that renders also makes the UI interactive.

**Subsequent navigations:** Excellent. Zero network round-trips for navigation, instant feel.

This creates an inverted performance profile compared to SSR: bad initial load, great subsequent interactions.

## When CSR Is the Right Choice

- **Authenticated, behind-login dashboards** — SEO doesn't matter, users accept a login loading screen, content is highly dynamic and personalized.
- **Highly interactive tools** — design editors, spreadsheets, IDEs in the browser where every interaction needs instant response.
- **Internal tooling** — users are on fast connections, use the tool repeatedly (cold start cost amortizes), SEO irrelevant.
- **When the backend is a separate team or service** — clean API contract, both sides deploy independently.

CSR is the wrong default for content-heavy public sites where the initial page load matters for SEO and user acquisition.

## Gotchas

**SEO is fundamentally compromised.** Googlebot does execute JavaScript, but with significant caveats: it runs in a headless browser with a rendering queue, can be delayed hours to days, and may time out on complex apps. Social crawlers (Twitter, Slack, Facebook link previews) often don't run JS at all — they see the blank shell and show no preview. For public content that needs to be discovered, CSR is a liability.

**Blank flash on hard reload.** Even fast CSR apps have a moment of nothing between the HTML response and the first JS paint. Users who have fast connections barely notice; users on constrained networks notice a lot.

**Data waterfalls compound.** If component A fetches data, then renders component B which fetches data, then B renders component C which fetches data — you have a serial waterfall that cannot start until the previous step completes. This is not a React problem; it's a structural consequence of fetching in components.

**Bundle size is a first-class concern.** Everything in the bundle must download before anything renders. A 1MB bundle on a mobile device with 50ms parse time per 100KB means ~500ms just for parsing, before any network wait.

**Memory management is your problem.** In SSR, each request creates a fresh process context. In a long-running SPA, every event listener that isn't cleaned up, every interval not cleared, every subscription not unsubscribed accumulates for the session lifetime.

**Back/forward cache (bfcache) complications.** The browser's bfcache freezes the page state and can restore it instantly. CSR apps that have timers, open WebSocket connections, or don't handle `pageshow`/`pagehide` events correctly can behave unexpectedly when restored from bfcache.

## Interview Questions

**Q (High): What is the critical performance weakness of CSR, and what metrics does it hurt?**

Answer: CSR's core weakness is that nothing meaningful renders until the JavaScript bundle downloads, parses, and executes. This directly hurts First Contentful Paint (FCP) and Largest Contentful Paint (LCP) — both measure how quickly users see real content. The blank HTML shell scores well on TTFB but terribly on any metric that measures perceived content. On slow connections or low-end devices, this gap is severe: a 300KB gzipped bundle might take 3–4 seconds to download, plus 1–2 seconds to parse and execute on a mid-range Android device, before any content appears.

The trap: Weaker answers focus on just "slow first load" without naming the specific metrics (LCP, FCP) or understanding that the problem compounds on slow devices because parse/execute time scales with CPU, not just network.

---

**Q (High): How does CSR affect SEO, and is Google's JavaScript rendering a complete solution?**

Answer: CSR fundamentally challenges SEO because the meaningful HTML content only exists after JavaScript executes. While Googlebot does render JavaScript, it does so with significant limitations: it processes pages asynchronously through a rendering queue that can introduce hours or days of delay between crawl and render, it runs with a rendering budget that can time out on complex apps, and it has historically had worse coverage of JS-heavy content than straight HTML. More importantly, social crawlers and most non-Google bots don't execute JavaScript at all — a CSR app's Open Graph tags must be injected statically or through SSR/prerendering, or link previews will be blank. For SEO-critical pages, SSR or SSG is the reliable choice; CSR is acceptable only for authenticated content that doesn't need to be indexed.

The trap: "Google executes JavaScript, so CSR is fine for SEO." This misses the crawl delay problem, the budget limits, and the social crawlers that never execute JS.

---

**Q (High): Explain the data waterfall problem in CSR and how it differs from parallel fetching.**

Answer: A data waterfall occurs when fetch calls are serialized because each component only requests data after it mounts, and child components only mount after parent data resolves. In a typical CSR component tree: App mounts → fetches user data → on success, renders Dashboard → Dashboard mounts → fetches stats → on success, renders Chart → Chart mounts → fetches chart data. Three serial network round-trips, each waiting for the previous to complete. Total latency is the sum of all RTTs. Parallel fetching — lifting all fetches to the top level or using a query library like React Query with prefetching — means all three requests fire simultaneously, and total latency is the maximum of any single RTT. The structural cause is co-locating data fetching with component rendering: components can only fetch what they know they need, and they only know that at render time.

The trap: Confusing "the components fetch in parallel within their level" with "the tree as a whole fetches in parallel." Sibling components do fetch in parallel; parent-child relationships create serial waterfalls.

---

**Q (Medium): Why does a CSR app served from a CDN need a specific server/CDN configuration for client-side routing to work?**

Answer: When a user navigates within a CSR app, the JavaScript router updates the URL using `history.pushState` without making a server request — so `/users/42` is handled client-side. But if the user directly requests `/users/42` in a fresh browser tab, the CDN or server receives a real HTTP request for that path. If the server responds with a 404 because there's no file at `/users/42`, the app breaks. The fix is to configure the server to return the same `index.html` for all paths (a "catch-all" or "history fallback" rule), letting the client-side router handle the routing once the bundle loads. Netlify calls this a `_redirects` file, nginx uses `try_files $uri /index.html`.

The trap: Assuming deep links always work because navigation within the app works. The distinction is client-side navigation (always works) vs. hard load of a deep link (requires server configuration).

---

**Q (Medium): When is CSR the right architectural choice, and what signals indicate you should use SSR or SSG instead?**

Answer: CSR fits best when: (1) the content is behind authentication and SEO is irrelevant, (2) the app is highly interactive with frequent state changes that would make SSR expensive, (3) the content is fully personalized and can't be statically generated, (4) navigation speed after initial load matters more than initial load speed. Signals to use SSR or SSG instead: public pages that need to appear in search results, content that crawlers need to read, poor LCP scores causing business impact, social sharing where link previews matter, and users on constrained connections where JS parse time is the bottleneck.

The trap: Defaulting to CSR for everything because "it's simpler to deploy a static bundle." The simplicity is real, but the performance and SEO costs are real too.

---

**Q (Low): How does the browser's back/forward cache (bfcache) interact with CSR apps, and what can break it?**

Answer: The bfcache freezes the complete page state — including JavaScript heap, DOM, and open connections — so the browser can restore it instantly when the user navigates back or forward. CSR apps can interfere with bfcache in several ways: open WebSocket connections prevent bfcache entry (the browser must close live connections to freeze the page), `unload` event listeners prevent bfcache eligibility entirely, and IndexedDB transactions that weren't committed keep the page locked. Apps that don't handle the `pageshow` event with `persisted: true` may show stale data when restored from bfcache, since the JS heap was frozen and `useEffect` cleanup/remount doesn't fire. The fix is to remove `unload` listeners, close WebSockets gracefully on `pagehide`, and optionally refresh data on `pageshow` when `event.persisted` is true.

The trap: Not knowing bfcache exists at all, or thinking it only affects server-rendered pages. CSR apps are fully eligible for bfcache but can easily disqualify themselves through common patterns.

---

## Self-Assessment

Before moving on, check off each item you can do WITHOUT looking at the file.

- [ ] Can explain what the browser receives from the server in a CSR app and why there's no visible content initially
- [ ] Can describe the sequence from URL request to visible content (HTML shell → JS download → execute → fetch → render)
- [ ] Can name the three Web Vitals that CSR hurts (LCP, FCP) and explain the mechanism
- [ ] Can explain why CSR apps need a catch-all server redirect for client-side routing
- [ ] Can name two scenarios where CSR is the right choice and two where it's the wrong choice
- [ ] Can explain what a data waterfall is and why it's structural to component-level fetching

---
*Next: Server-Side Rendering (SSR) — the natural contrast to CSR, where the server does the rendering work and sends real HTML, trading server load for better FCP and SEO.*
