# Streaming SSR

## Quick Reference

| Concept | Mechanism | Implication |
|---|---|---|
| Chunked HTML | Server sends HTML in pieces over a single HTTP connection | Browser can parse and paint earlier chunks while later ones are still rendering |
| Suspense boundaries | React components wrapped in `<Suspense>` emit a placeholder, then flush real content when ready | Parts of the page become visible and interactive before the full document arrives |
| Out-of-order flushing | Content can flush in a different order than it appears in the DOM | A slow sidebar doesn't block a fast hero section from painting |
| TTFB improvement | First byte sends immediately (shell, head, above-fold HTML) | Browser starts downloading assets (fonts, JS, CSS) sooner |

## What Is This?

Traditional SSR has a fundamental bottleneck: the server must finish rendering the *entire* page before it can send the first byte of HTML. If the page has a slow database query for a sidebar widget, the whole page waits.

Streaming SSR breaks that constraint by sending HTML as a chunked response — not all at once, but in pieces as each part of the page becomes ready. The browser receives and processes each chunk incrementally, painting visible content long before the complete document arrives.

```
Traditional SSR:
  server: render everything (500ms) → send complete HTML
  browser: receive → parse → paint

Streaming SSR:
  server: render header (5ms) → flush
  server: render product info (50ms) → flush
  server: render reviews (500ms) → flush
  browser: paint header at 5ms, product at 55ms, reviews at 500ms
```

The user sees meaningful content hundreds of milliseconds earlier, even though the total server work is identical.

> **Check yourself:** Streaming SSR doesn't reduce the total server rendering time. What exactly does it improve?

## Why Does It Exist?

SSR's promise — faster FCP than CSR — is undermined when any part of the page requires slow data. If rendering the page requires a 400ms database query, the user waits 400ms before seeing anything. The first byte of HTML arrives only after the last piece of data is ready.

Streaming fixes this by changing the question from "when is the page done?" to "when is each part done?" Parts of the page that are ready early (the navigation, the above-the-fold hero, the document `<head>` with its stylesheets and preload hints) can reach the browser immediately. The browser starts:
- Parsing the `<head>` and beginning to download CSS and JS
- Rendering and painting visible above-fold content
- Progressively revealing below-fold content as it streams in

This is the same principle used for streaming video: you don't wait for the full file to download before showing the first frame.

## How It Works

### HTTP Chunked Transfer Encoding

Streaming relies on HTTP's built-in chunked transfer encoding. Instead of a response with a known `Content-Length`, the server sends a response with `Transfer-Encoding: chunked`. Each chunk is a fragment of HTML followed by a size header. The browser processes each chunk as it arrives without waiting for the complete response.

Node.js HTTP and all modern edge runtimes support this natively. The web standard for producing streaming responses is the `ReadableStream` API.

### React Suspense as the Streaming Primitive

React 18 introduced first-class streaming support via `renderToPipeableStream` (Node) and `renderToReadableStream` (edge/Deno). The mechanism:

1. React starts rendering the component tree.
2. When it encounters a `<Suspense>` boundary wrapping a component that's waiting for data (a lazy-loaded component, or a component using a suspense-compatible data library), React doesn't block — it renders the fallback content (a spinner, skeleton) and **continues rendering the rest of the tree**.
3. The already-rendered HTML (shell, header, fallback placeholders) is flushed to the browser immediately.
4. When the suspended component's data resolves, React renders that component and flushes the HTML chunk, along with a small inline `<script>` that tells the browser where to insert it in the DOM.

```jsx
// The server renders what it can immediately, streams placeholders for the rest
function ProductPage() {
  return (
    <html>
      <body>
        <Header />  {/* renders instantly */}
        <ProductHero productId="123" />  {/* fast query, renders soon */}
        
        <Suspense fallback={<ReviewsSkeleton />}>
          <Reviews productId="123" />  {/* slow query, streams later */}
        </Suspense>
        
        <Suspense fallback={<RecommendationsSkeleton />}>
          <Recommendations />  {/* external API, streams when ready */}
        </Suspense>
      </body>
    </html>
  );
}
```

### The Browser's Role

When the browser receives a late-arriving chunk for a `<Suspense>` boundary, it contains:
1. The rendered HTML for the component (hidden initially via inline CSS)
2. An inline script that updates the DOM, replacing the skeleton with the real content

This script runs as soon as the chunk arrives — no waiting for the full page to load. The user sees the skeleton replaced with real content progressively as each Suspense boundary resolves.

### Selective Hydration

Streaming SSR in React 18 pairs with Selective Hydration: the browser can begin hydrating the parts of the page that have already streamed in without waiting for the full HTML. If the user interacts with the header while the reviews section is still loading, React prioritizes hydrating the header's event handlers immediately.

> **Check yourself:** A Suspense boundary's fallback shows on screen, then is replaced by the real content. Was there a loading spinner visible during the stream? What's the UX implication?

## Streaming vs. Traditional SSR Performance

Consider a page with three sections:
- Header: 5ms to render
- Product details: 80ms (DB query)
- Customer reviews: 600ms (slow query)

| Metric | Traditional SSR | Streaming SSR |
|---|---|---|
| TTFB | 600ms (blocked on slowest) | ~5ms (shell flushes immediately) |
| Header visible | 600ms | ~5ms |
| Product details visible | 600ms | ~85ms |
| Reviews visible | 600ms | ~600ms |
| Total server work | 600ms | 600ms |

The total server computation is identical. What changes is when each piece reaches the browser. TTFB improves from 600ms to ~5ms — the browser starts downloading CSS, fonts, and the JS bundle 595ms earlier, which speeds up the entire page load even before content is painted.

## Implementation in Next.js App Router

Next.js App Router (Next.js 13+) uses streaming by default. Any `async` component that awaits data is automatically a streaming boundary. The `loading.js` file convention creates the Suspense fallback:

```
app/
  products/
    [id]/
      page.js       ← the actual page component (async, fetches data)
      loading.js    ← shown immediately while page.js is loading
```

```jsx
// app/products/[id]/page.js
// This is an async Server Component — Next.js streams it automatically
export default async function ProductPage({ params }) {
  const product = await db.products.findById(params.id);  // awaited here
  const reviews = await db.reviews.findByProduct(params.id);  // and here

  return (
    <div>
      <h1>{product.name}</h1>
      <ReviewList reviews={reviews} />
    </div>
  );
}
```

With parallel data fetching using `Promise.all`, both queries run concurrently. With nested Server Components, each component is its own streaming boundary.

## Gotchas

**Streaming requires avoiding response-blocking operations in the shell.** The initial flush (the part before any Suspense boundaries) should be fast. If you put a slow database query outside of any Suspense boundary, you've recreated traditional SSR — the entire shell blocks on that query.

**Error handling is more complex.** In traditional SSR, if rendering throws an error, you can send a 500 response before any HTML goes out. With streaming, you may have already sent 200 OK and some HTML before an error occurs. You can't change the HTTP status code retroactively. React handles this by injecting error UI into the stream, but the browser has already committed to a 200 response.

**SSR cache layers need to buffer the full stream.** Some caching proxies and CDNs buffer streaming responses before forwarding them, which negates the streaming benefit. Verify that your CDN and any intermediate proxies support and are configured to pass through chunked responses without buffering.

**Skeleton flash.** If a Suspense boundary resolves quickly (in <100ms), the skeleton/fallback placeholder flashes briefly before being replaced. This can look worse than just waiting for the data and rendering directly. Use `startTransition` and debounce for fast-resolving boundaries, or set a minimum display time for skeletons.

**Out-of-order streaming requires inline scripts.** The mechanism for injecting late-arriving HTML into the correct DOM position uses small inline `<script>` tags. If you have a strict CSP that blocks inline scripts (and you should), you need to use nonces or configure your CSP to accommodate these framework-generated scripts.

## Interview Questions

**Q (High): What problem does Streaming SSR solve that traditional SSR doesn't, and what makes it possible technically?**

Answer: Traditional SSR has an all-or-nothing delivery model: the server must render the complete page before sending the first byte. If any part of the page requires slow data, the entire page is delayed — the browser sits waiting, not downloading assets, not painting anything. Streaming SSR solves this by sending HTML incrementally. The shell (head, navigation, above-fold layout) flushes immediately, so the browser can start downloading CSS, JS, and fonts at ~5ms instead of waiting 600ms for a slow database query. Below-fold or data-heavy sections stream in as their data becomes available. Technically, this is possible via HTTP chunked transfer encoding (which doesn't require a Content-Length up front) and React 18's `renderToPipeableStream`, which uses Suspense boundaries as flush points — React renders what it can, flushes it, and continues rendering suspended sections asynchronously.

The trap: Saying "streaming makes the server faster." It doesn't — total server render time is unchanged. What changes is how and when the results reach the browser.

---

**Q (High): How do React Suspense boundaries function as streaming boundaries, and what does the browser receive for a suspended component?**

Answer: A `<Suspense>` boundary is React's way of saying "if any component inside this boundary is waiting for something, render the fallback instead of blocking." In streaming SSR, when React encounters a suspended component during server rendering, it doesn't wait — it renders the fallback (a skeleton, spinner, or null), continues rendering the rest of the tree, and flushes what's ready to the browser. When the suspended component's data resolves, React renders that component's HTML and streams it to the browser in a new chunk. The chunk includes both the rendered HTML and a small inline `<script>` that tells the browser which placeholder to replace with the new content — this is what enables out-of-order delivery, where a sidebar resolving late doesn't need to wait for earlier siblings.

The trap: Not knowing about the inline script mechanism. Weaker answers describe streaming without explaining how the browser gets out-of-order HTML into the right position.

---

**Q (Medium): Why is TTFB particularly important for streaming SSR, and how does it cascade to other metrics?**

Answer: TTFB is when the browser receives the first byte of the response. For streaming SSR, the first chunk contains the document `<head>` — which has `<link rel="stylesheet">`, `<link rel="preload">`, and the `<script>` tags for the JS bundle. These are the browser's signals to start downloading resources. Every millisecond of TTFB delay is a millisecond before the browser knows to request these assets. With traditional SSR waiting 600ms before sending the first byte, the JS bundle download starts 600ms later — and that delay directly pushes out TTI and hydration completion. Streaming's fast TTFB means asset downloads start almost immediately, narrowing the gap between "user sends request" and "page is fully interactive."

The trap: Thinking TTFB only affects when the user sees the first content. TTFB is the clock start for the entire resource waterfall.

---

**Q (Medium): What happens when an error occurs in a component that's already been streamed to the browser?**

Answer: Once the server has sent the HTTP 200 status code and some HTML bytes, it can't change the response status code or recall what was sent. If a Suspense boundary component throws during server rendering after the stream has started, React handles it in one of two ways depending on where the error occurs. If it's inside a Suspense boundary that has an error boundary wrapping it, React streams the error boundary's fallback UI instead of the component's output. If it's outside any Suspense/error boundary, React can emit a script that triggers a client-side error, causing the browser to re-render in error mode. The practical implication: all critical data fetching that could fail should either be outside streaming boundaries (fail fast, before any HTML sends) or wrapped in error boundaries that have graceful fallback UI.

The trap: Claiming "you just send a 500 error." Once streaming has started with a 200, you can't send a 500. This is a real constraint that affects error handling architecture.

---

**Q (Low): How does Selective Hydration interact with streaming, and why does it improve perceived interactivity?**

Answer: Selective Hydration, introduced in React 18 alongside streaming, allows React to hydrate individual parts of the page independently rather than requiring the complete HTML to be present and the entire JS bundle to be executed before any hydration begins. With streaming SSR, as HTML chunks arrive, React can hydrate the already-delivered portions of the page immediately. If the user scrolls into view or interacts with an area that's been delivered, React prioritizes hydrating that area first — even if other parts of the page are still streaming in or not yet hydrated. This means time-to-interactive for above-fold content is decoupled from the hydration cost of below-fold content. A complex product reviews section below the fold doesn't delay the "Add to Cart" button above the fold from becoming interactive.

The trap: Thinking hydration still happens all-at-once after the full page loads. Selective Hydration is a parallel, prioritized process that starts as soon as any chunk of HTML is available.

---

## Self-Assessment

Before moving on, check off each item you can do WITHOUT looking at the file.

- [ ] Can explain why traditional SSR's TTFB is slow and what streaming does differently
- [ ] Can describe how React's Suspense boundaries act as streaming flush points
- [ ] Can explain what the browser receives in a late-arriving Suspense chunk (HTML + inline script)
- [ ] Can articulate the performance improvement as "earlier TTFB, not faster total render"
- [ ] Can name the error handling limitation of streaming (can't change HTTP status after first byte)
- [ ] Can explain Selective Hydration at a high level

---
*Next: Partial Hydration & Island Architecture — taking the idea further by shipping JavaScript only for interactive components, keeping the rest as static HTML, as popularized by Astro and Fresh.*

*See also: [Phase 2 — Critical Rendering Path](../phase-02-browser-internals/02-critical-rendering-path.md) — explains how the browser processes the first bytes of HTML as they arrive, and why streaming's earlier TTFB translates directly to earlier paint.*
