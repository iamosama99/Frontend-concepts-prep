# Server-Side Rendering (SSR)

## Quick Reference

| Concept | Mechanism | Implication |
|---|---|---|
| Full HTML per request | Server runs JS/template engine, responds with real HTML | Crawler and browser see content immediately |
| TTFB cost | Server must render before sending first byte | Slower TTFB than serving a static file |
| Hydration | Browser downloads JS bundle, attaches event listeners to existing HTML | UI is visible before interactive (hydration gap) |
| Personalization | Render runs per-request with user context | Naturally supports per-user HTML, auth-aware content |

## What Is This?

Server-Side Rendering is the model where the server generates the full HTML for a page on every request and sends it to the browser. When the browser receives the response, it can paint meaningful content immediately — before any JavaScript executes. The page is visible right away; it just isn't interactive yet.

```
GET /products/42
↓
Server: fetch product from DB → run React/template → produce HTML string → send response
↓
Browser receives:
<html>
  <head>...</head>
  <body>
    <h1>Wireless Headphones</h1>  ← real content, not blank shell
    <p>$89.99</p>
    <button>Add to Cart</button>   ← visible but not clickable yet
  </body>
</html>
```

The key distinction from CSR: the HTML that arrives contains the actual content of the page, not a blank container waiting for JavaScript to fill it.

> **Check yourself:** A user with JavaScript disabled opens an SSR page. What happens? What about the same page in a CSR app?

## Why Does It Exist?

SSR is actually the original model — every early web framework (PHP, Rails, Django, ASP.NET) rendered HTML on the server. SPAs and CSR emerged as a counter-trend, optimizing for the post-load interaction experience at the cost of first load.

SSR came back into focus as a deliberate architectural choice when the downsides of CSR became measurable business problems: poor SEO rankings because crawlers couldn't index JavaScript-rendered content, bad LCP scores on mobile causing bounce rates, and the realization that many "apps" were actually content-heavy pages that happened to have some interactivity.

The modern SSR renaissance (Next.js, Nuxt, Remix, SvelteKit) layers a component model on top of the server-render model, letting you write React/Vue/Svelte components that render on the server and then "hydrate" in the browser — getting the HTML-first benefits without giving up the component-based development model.

## How It Works

### The Render Cycle

1. **Request arrives** — user navigates, crawler fetches, link is shared.
2. **Server authenticates** the request (check session/JWT).
3. **Server fetches data** — database queries, API calls happen on the server.
4. **Server renders** — the component tree (or template) runs with that data and produces an HTML string.
5. **Server sends HTML** — the browser receives a full, content-populated document.
6. **Browser paints** immediately — content is visible as the HTML parses.
7. **JS bundle downloads** in parallel with the user reading the page.
8. **Hydration runs** — React/Vue attaches event listeners to the server-rendered HTML, making it interactive.

```javascript
// Next.js Pages Router — getServerSideProps runs on every request
export async function getServerSideProps(context) {
  const { params, req } = context;
  const user = await getSessionUser(req);  // auth on server
  const product = await db.products.findById(params.id);  // data on server

  return {
    props: { product, user }  // serialized into HTML as window.__NEXT_DATA__
  };
}

export default function ProductPage({ product, user }) {
  // This component runs on the server (produces HTML) AND in the browser (hydrates)
  return (
    <div>
      <h1>{product.name}</h1>
      <button onClick={() => addToCart(product.id)}>Add to Cart</button>
    </div>
  );
}
```

### Hydration in Detail

The HTML the server sends contains all visible content, but it's inert — no event listeners, no React state. Hydration is the process where the JS bundle runs and React "takes over" the existing DOM nodes. React reads the server-rendered HTML and matches it to the component tree, then attaches event handlers without re-creating the DOM.

This is different from CSR where React creates DOM nodes from scratch. In SSR, React says "I see this HTML already exists — I'll adopt it and attach behavior."

Hydration has a cost: the entire JS bundle must download and execute before the page becomes interactive. During this window — which can be seconds on slow devices — content is visible but buttons don't respond, forms don't submit, and dynamic interactions don't work. This is the **hydration gap**.

> **Check yourself:** During the hydration gap, a user clicks a button. What happens? What does this mean for perceived interactivity?

## TTFB vs. FCP Trade-off

SSR has a fundamentally different performance profile than CSR:

| Metric | CSR | SSR |
|---|---|---|
| TTFB | Fast (static file from CDN) | Slower (server must render first) |
| FCP / LCP | Slow (JS must execute first) | Fast (HTML arrives with content) |
| TTI | Same as FCP | After FCP (hydration delay) |
| Subsequent navigation | Very fast (no server) | Depends (can pre-fetch) |

TTFB is worse in SSR because the server can't send the first byte until it finishes rendering. A complex page with multiple database queries might take 200–500ms of server work before the first byte of HTML goes out. A CDN serving a static file sends the first byte in under 10ms.

But FCP is better because the content is in that first response. The browser starts painting as soon as bytes arrive and parses the HTML incrementally. Users see real content within the TTFB, not a blank page.

## Caching SSR Responses

The performance tax of SSR can be largely mitigated with caching. If the rendered HTML for `/products/42` is the same for all users (it's public, non-personalized content), it can be cached at the CDN edge. The server renders it once; all subsequent requests hit the cache and get the fast static-file TTFB.

The challenge: personalized content (showing a user's name, cart count, authentication state) can't be cached at the CDN without leaking one user's data to another. This drives architectures where the "shell" is cached (header, nav, layout) and personalized fragments are either:
- Fetched client-side after the shell renders (hybrid approach)
- Rendered with edge computing per-user (edge SSR)
- Split into separate cacheable and uncacheable sections

## Gotchas

**The hydration gap is a real UX problem.** Content visible, interactions broken is a confusing and frustrating state. On a fast connection it lasts milliseconds; on a slow device with a 500KB bundle it can last several seconds. Users click buttons during this window and nothing happens — the click fires but React hasn't attached the handler yet.

**Hydration mismatch errors.** React compares the server-rendered HTML to what it would render client-side. If they differ — because of browser-specific values (window width, Date.now(), navigator.language), rendering logic that behaves differently server vs. client, or race conditions — React throws a hydration error and falls back to a full client-side re-render, destroying and recreating the DOM. This negates the SSR benefit for that render and may cause a flash of content.

**Server execution environment differs from browser.** `window`, `document`, `localStorage`, `navigator` don't exist on the server. Code that runs universally (isomorphic code) must guard against this:
```javascript
const isClient = typeof window !== 'undefined';
// This is fragile and easy to forget
```

**TTFB can become a bottleneck.** If a page requires multiple slow database queries before it can render, users wait for all of them before seeing anything. A 300ms query on the server becomes 300ms of white screen — the same as CSR waiting for a fetch, but without the option to show a loading state during SSR render time.

**Serverless cold starts amplify TTFB.** Many modern SSR deployments run on serverless functions (Vercel, Netlify Functions, Lambda). Cold start time — when the function isn't cached in memory — adds 200–500ms or more to TTFB on the first request after idle.

**Memory and CPU cost is per request.** SSR servers run your rendering code for every request. A CPU spike from a complex render affects all concurrent users sharing that server. This is unlike CSR where rendering work is distributed across users' devices.

## Interview Questions

**Q (High): What is hydration and why does it create a window where the page is visible but not interactive?**

Answer: Hydration is the process where a JavaScript framework takes ownership of server-rendered HTML by attaching event listeners and reconciling the component tree with the existing DOM nodes. The server sends real HTML that the browser paints immediately — content is visible. But the page isn't interactive because JavaScript hasn't run yet. The framework must download its full bundle, parse it, execute it, walk the component tree, and attach handlers to each element before any interaction works. This process takes time — tens to hundreds of milliseconds on fast devices, potentially seconds on slow devices with large bundles. During this window, buttons are visible but unresponsive. The solution is to keep the bundle small so hydration completes quickly, and in modern frameworks like React, Selective Hydration lets high-priority interactive elements hydrate first.

The trap: Saying "the page is fully loaded after hydration." Weaker answers conflate SSR + hydration as simply "server does more work." The key insight is the visible-but-broken intermediate state and its implications for UX.

---

**Q (High): Compare TTFB, FCP, and TTI between CSR and SSR. When does each model win?**

Answer: CSR has better TTFB (a tiny HTML shell serves from a CDN in milliseconds) but terrible FCP/LCP (nothing meaningful renders until the JS bundle executes). SSR has worse TTFB (the server must render before sending the first byte, potentially 100–500ms for data fetching) but much better FCP/LCP (the full HTML arrives with content, the browser paints immediately). For TTI: in CSR, TTI ≈ FCP since the JS that renders also makes things interactive. In SSR, TTI comes after FCP because hydration adds a second phase after the HTML arrives. SSR wins for perceived performance (users see content faster), public pages, SEO, and constrained devices. CSR wins for dashboard-style apps behind auth, highly interactive tools, and cases where fast subsequent navigation matters more than first load.

The trap: Claiming "SSR is always better for performance." The TTFB cost of SSR is real, and caching doesn't apply to personalized content. The correct answer acknowledges both the win and the cost.

---

**Q (High): What causes a React hydration mismatch and what are the consequences?**

Answer: A hydration mismatch occurs when the HTML React tries to adopt in the browser differs from what the server rendered. Common causes: rendering content that depends on browser-only values (window.innerWidth, Date.now(), cookies, localStorage, navigator.language) during the initial render — these values don't exist on the server, so the server renders one thing and the browser renders another. Other causes include conditional rendering that depends on client-side state, third-party browser extensions modifying the DOM, and using Math.random() or timestamps without seeding. The consequences: React logs a warning, discards the server-rendered HTML, and falls back to a full client-side render from scratch. This negates all SSR benefits for that component — instead of a fast paint, the user sees a flash as the DOM is torn down and rebuilt.

The trap: Not knowing what a hydration mismatch actually causes at runtime. Many candidates know mismatch is bad without understanding that it discards server HTML and re-renders client-side.

---

**Q (Medium): How would you handle authentication in an SSR app, and what are the trade-offs of different approaches?**

Answer: Authentication in SSR requires sending the user's credential with the server request so the server can render personalized content. HttpOnly cookies are the natural fit: they're sent automatically with every request, inaccessible to JavaScript (XSS-safe), and available in the server rendering context through the request headers. The server reads the cookie, validates the session, fetches user-specific data, and renders the appropriate HTML. The trade-off: this content can't be CDN-cached because it's personalized per user. Alternatives include: rendering a generic authenticated shell on the server and fetching user data client-side after hydration (cacheable shell, JS waterfall for data), or using edge functions that can read cookies at the CDN layer and render personalized HTML near the user (edge SSR).

The trap: "Just use localStorage for auth tokens." localStorage is inaccessible on the server. This is a common mistake from developers who built CSR apps and haven't reasoned through the SSR authentication model.

---

**Q (Medium): What is an "isomorphic" or "universal" application, and what constraints does running code in both environments impose?**

Answer: An isomorphic (or universal) application uses the same code on the server and in the browser. SSR frameworks require this: the same React component tree renders on the server (producing HTML) and in the browser (hydrating). This imposes constraints because the two environments have different global APIs. The browser has `window`, `document`, `localStorage`, `navigator`, `requestAnimationFrame`. The server has none of these. Code that accesses these APIs without guarding will throw on the server. Common patterns: `typeof window !== 'undefined'` checks, wrapping browser-only code in `useEffect` (which only runs client-side), or using the framework's client-only component boundaries. The subtler constraint is that server and browser rendering must produce identical output — any divergence causes hydration mismatches.

The trap: Not knowing that `useEffect` doesn't run on the server. This is why useEffect is the standard place for browser-only initialization in React.

---

**Q (Low): How does SSR interact with CDN caching, and what patterns make SSR responses cacheable?**

Answer: SSR responses are HTML strings generated per-request, which makes naive SSR fully dynamic and un-cacheable at the CDN layer. To get SSR performance while leveraging CDN caching: (1) Content that's the same for all users (public product pages, blog posts) can be cached at the CDN by setting appropriate Cache-Control headers — the server renders once per cache period. (2) Stale-While-Revalidate allows the CDN to serve a slightly stale response immediately while revalidating in the background. (3) Fragment caching caches expensive data queries on the server in Redis/Memcached even if the full response varies. (4) Edge SSR (Cloudflare Workers, Vercel Edge) runs the render close to the user, reducing latency even when not cached. The fundamental limit: responses that include user-specific data (username, cart, session state) cannot be shared in a CDN cache without leaking data across users.

The trap: Assuming all SSR responses are uncacheable, or not knowing that Cache-Control headers apply to SSR responses the same as any other HTTP response.

---

## Self-Assessment

Before moving on, check off each item you can do WITHOUT looking at the file.

- [ ] Can explain what HTML the browser receives in SSR vs. CSR and why it matters
- [ ] Can walk through the hydration process step-by-step, including the visible-but-not-interactive phase
- [ ] Can compare TTFB, FCP, and TTI between CSR and SSR with correct directional claims
- [ ] Can explain what causes a hydration mismatch and what React does when one occurs
- [ ] Can name two reasons SSR responses are harder to cache than CSR apps
- [ ] Can name two browser-only APIs that don't exist on the server and how to guard against them

---
*Next: Static Site Generation (SSG) — the model where server rendering happens once at build time rather than per-request, getting both the HTML-first benefits of SSR and the CDN-cached speed of CSR.*
