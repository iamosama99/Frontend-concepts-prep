# Resource Hinting: Preload, Prefetch, Preconnect, Prerender

## Quick Reference

| Hint | Timing | Priority | What it does | When to use |
|---|---|---|---|---|
| `preload` | Current navigation | High | Fetches a resource needed for this page | LCP images, critical fonts, late-discovered CSS |
| `prefetch` | Next navigation | Lowest | Fetches a resource for a future page | Links the user is likely to click next |
| `preconnect` | Current navigation | Medium | Opens TCP + TLS to an origin | Third-party CDNs you know you'll need |
| `dns-prefetch` | Current navigation | Low | Resolves DNS only | Origins where full preconnect is too expensive |
| `modulepreload` | Current navigation | High | Fetches + parses an ES module | Critical JS modules loaded dynamically |

---

## What Is This?

Resource hints are `<link>` elements that tell the browser what to fetch before it would discover those resources through normal HTML parsing. The browser's preload scanner reads the HTML as bytes arrive and speculatively fetches resources referenced in `<link>`, `<script>`, and `<img>` tags — but it cannot see resources discovered by JavaScript at runtime, resources referenced in CSS, or resources on pages the user hasn't visited yet. Resource hints close these gaps.

> **Check yourself:** What is the difference between a resource hint and a regular `<link rel="stylesheet">` tag? Does a preload hint apply the resource automatically?

---

## Why Does It Exist?

The browser's preload scanner is a secondary parser that runs ahead of the main HTML parser. When the main parser is blocked by a render-blocking script, the preload scanner reads ahead and dispatches fetch requests for images, scripts, and stylesheets it can see in the markup. This is one of the most impactful browser performance features.

But the preload scanner is limited to what it can see in the HTML. It cannot:

- Discover a font referenced inside a `.css` file (the CSS must be fetched and parsed first)
- Know about a dynamic `import()` that only executes after some JS runs
- Prefetch a page the user hasn't navigated to yet
- Pre-warm a connection to a CDN that will be needed once JS executes

Resource hints expose these intentions to the browser explicitly, before discovery would naturally happen.

---

## How It Works

### `preload` — fetch this resource now, I need it for this page

`preload` is a mandatory fetch. It tells the browser: "This resource is definitely needed for the current page — start fetching it at high priority now, even if you haven't discovered it yet through normal parsing."

```html
<!-- Preload the LCP image — before the browser can discover it in the CSS or lazy JS -->
<link rel="preload" as="image" href="/hero.webp" fetchpriority="high" />

<!-- Preload a font — before the browser parses the CSS that references it -->
<link rel="preload" as="font" href="/fonts/inter.woff2" type="font/woff2" crossorigin />

<!-- Preload a late-discovered critical CSS file -->
<link rel="preload" as="style" href="/critical.css" />
<link rel="stylesheet" href="/critical.css" /> <!-- still need this to apply it -->
```

**The `as` attribute is mandatory.** It tells the browser the resource type so it can set the correct request priority, apply the correct Content Security Policy, and match the resource to its eventual consumer. Without `as`, the browser fetches at lowest priority and may fetch the resource twice.

**`crossorigin` is required for fonts.** Fonts are fetched in CORS mode regardless of origin. If you omit `crossorigin` on a font preload, the preloaded response will not match the font fetch and the browser fetches it twice.

```html
<!-- Wrong: missing crossorigin — browser fetches font twice -->
<link rel="preload" as="font" href="/inter.woff2" type="font/woff2" />

<!-- Correct -->
<link rel="preload" as="font" href="/inter.woff2" type="font/woff2" crossorigin />
```

**`modulepreload` for ES modules:**

```html
<!-- Preload a critical ES module AND its dependency graph -->
<link rel="modulepreload" href="/src/checkout.js" />
```

`modulepreload` does more than `preload as="script"`: it fetches, compiles, and caches the module in the module map. Regular `preload as="script"` only fetches the bytes. For dynamic `import()` calls that are critical to the current page, `modulepreload` is the right hint.

> **Check yourself:** If you preload a stylesheet with `<link rel="preload" as="style" href="/foo.css">` but forget to add `<link rel="stylesheet" href="/foo.css">`, what happens? Is the stylesheet applied?

---

### `prefetch` — fetch this for a future navigation

`prefetch` is a speculative, lowest-priority fetch. The browser downloads the resource after the current page finishes loading and stores it in the HTTP cache. When the user navigates to the page that needs it, the resource is already in cache and the navigation feels instant.

```html
<!-- Prefetch the checkout page when the user is on the cart page -->
<link rel="prefetch" href="/checkout" />

<!-- Prefetch a JS chunk for the next route -->
<link rel="prefetch" as="script" href="/chunks/checkout.js" />
```

**Prefetch is best-effort.** The browser may ignore it under memory pressure, poor network conditions, or Data Saver mode. Never rely on prefetch for resources required for the current page — that is `preload`'s job.

**In React Router / Next.js, prefetch is often handled automatically.** Next.js prefetches all `<Link>` targets when they enter the viewport. React Router v6 has no automatic prefetching but you can trigger it on hover:

```js
// Manually trigger a prefetch when the user hovers a link
function PrefetchLink({ to, children }) {
  const handleMouseEnter = () => {
    const link = document.createElement('link');
    link.rel = 'prefetch';
    link.href = to;
    document.head.appendChild(link);
  };

  return <a href={to} onMouseEnter={handleMouseEnter}>{children}</a>;
}
```

---

### `preconnect` — warm up the connection to this origin

`preconnect` tells the browser to perform DNS resolution, TCP handshake, and TLS negotiation for an origin — but not to fetch any resource. This eliminates the connection setup latency (100–500ms on mobile) for resources that will definitely be needed from that origin.

```html
<!-- Warm connection to Google Fonts before the CSS is fetched -->
<link rel="preconnect" href="https://fonts.googleapis.com" />
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin />

<!-- Warm connection to your API -->
<link rel="preconnect" href="https://api.example.com" />

<!-- Warm connection to an image CDN -->
<link rel="preconnect" href="https://images.example.com" />
```

**`crossorigin` on preconnect** is needed when the resource will be fetched with CORS (fonts, cross-origin scripts). Without it, the browser opens an anonymous connection; when the CORS fetch arrives, it requires a different connection and the warmup is wasted.

**Don't overuse preconnect.** Each open connection consumes CPU (TLS) and memory. Preconnect is appropriate for 2–3 origins you are certain will be needed immediately. For origins that might be needed later (analytics, ads), use `dns-prefetch` instead.

---

### `dns-prefetch` — resolve DNS only

`dns-prefetch` is a lighter-weight version of `preconnect`. It only performs DNS resolution — no TCP handshake, no TLS. The savings are smaller (20–120ms) but the cost is nearly zero.

```html
<!-- DNS prefetch for origins you might need but aren't certain about -->
<link rel="dns-prefetch" href="https://analytics.google.com" />
<link rel="dns-prefetch" href="https://cdn.intercom.io" />
```

**Rule of thumb:** Use `preconnect` for at most 2–3 critical third-party origins. Use `dns-prefetch` for everything else you might need.

---

## Priority and Interaction with the Preload Scanner

The browser assigns fetch priorities based on the resource type and when it is discovered. Preload overrides the default discovery timing but not necessarily the priority — you must use `fetchpriority` for that.

```html
<!-- Tell the browser this preloaded image is the highest priority fetch -->
<link rel="preload" as="image" href="/hero.webp" fetchpriority="high" />

<!-- Tell the browser a non-critical preloaded script is low priority -->
<link rel="preload" as="script" href="/analytics.js" fetchpriority="low" />
```

`fetchpriority` accepts `high`, `low`, or `auto` (default). It is a hint, not a command — the browser may override it. Use `fetchpriority="high"` specifically for the LCP image to ensure it competes well against other critical resources.

---

## Responsive Image Preloading

When your LCP image uses `srcset`, you need to preload the correct size for the current viewport — or the browser may preload a different image than what it ends up using.

```html
<!-- Responsive preload with imagesrcset and imagesizes -->
<link
  rel="preload"
  as="image"
  imagesrcset="
    /hero-400.webp 400w,
    /hero-800.webp 800w,
    /hero-1200.webp 1200w
  "
  imagesizes="(max-width: 600px) 100vw, 50vw"
  fetchpriority="high"
/>
```

Without `imagesrcset`, a plain `href` preload fetches a single image. If the browser then uses a different srcset candidate based on viewport width, the preloaded image is wasted and the correct one is fetched late.

---

## Common Mistakes

**Preloading too many resources.** Every preloaded resource competes for bandwidth with the resources the browser has already decided to fetch. Preloading 10 things defeats the purpose — the bandwidth is split and nothing is actually faster. Preload 1–3 resources that are genuinely critical and late-discovered.

**Unused preloads.** If the browser fetches a preloaded resource but it is never consumed within ~3 seconds, DevTools shows a "resource was preloaded but not used" warning. This means the fetch was wasted. Check the `as` attribute and `crossorigin` attribute are correct so the preloaded response is matched to the actual consumer.

**Preloading resources the preload scanner already finds.** A `<img>` in the initial HTML is already found by the preload scanner at high priority. Adding a `<link rel="preload">` for it is redundant and can actually lower its priority if done without `fetchpriority="high"`.

**Using `prefetch` for critical resources.** Prefetch is lowest priority and may never execute. Using it for resources needed on the current page will leave them loading at the worst possible priority.

**Preconnecting to too many origins.** Preconnect opens full TLS connections. Preconnecting to 10 origins means 10 TLS handshakes at page load, consuming CPU on both client and server for connections that may never be used.

---

## Practical Checklist

```html
<head>
  <!-- 1. Preconnect to critical third-party origins (max 2-3) -->
  <link rel="preconnect" href="https://fonts.googleapis.com" />
  <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin />

  <!-- 2. DNS prefetch for other known origins -->
  <link rel="dns-prefetch" href="https://analytics.example.com" />

  <!-- 3. Preload LCP image with high priority -->
  <link
    rel="preload"
    as="image"
    href="/hero.webp"
    fetchpriority="high"
  />

  <!-- 4. Preload critical font (crossorigin required) -->
  <link
    rel="preload"
    as="font"
    href="/fonts/inter-var.woff2"
    type="font/woff2"
    crossorigin
  />

  <!-- 5. Prefetch next-page chunks (after page loads, idle time) -->
  <link rel="prefetch" href="/chunks/checkout.js" />
</head>
```

---

## Interview Questions

**Q (High): What is the difference between `preload` and `prefetch`, and when would you use each?**
Answer: `preload` is a mandatory, high-priority fetch for a resource needed on the current page. The browser fetches it immediately, regardless of whether it has discovered it through HTML parsing. Use it for resources that are late-discovered but needed soon: LCP images in CSS backgrounds, fonts referenced in stylesheets, critical JS chunks loaded via dynamic import. `prefetch` is a speculative, lowest-priority fetch for a resource needed on a future page. The browser fetches it during idle time after the current page loads, and the response goes into the HTTP cache. Use it for routes the user is likely to navigate to next: checkout page chunks on the cart page, product detail chunks on the product listing page. The critical mistake is using `prefetch` for current-page resources — it may not execute, and if it does, it competes at the lowest priority.
The trap: Thinking "preload is better because it's higher priority." Preload for a future page would waste bandwidth and potentially delay current-page resources.

**Q (High): Why does a font preload require the `crossorigin` attribute, and what breaks without it?**
Answer: Browsers always fetch fonts in CORS mode, regardless of whether the font is same-origin or cross-origin. This means the request is made with a CORS fetch, which requires a different request mode than a non-CORS fetch. The browser maintains separate cache entries for CORS and non-CORS requests to the same URL. If you omit `crossorigin` on the preload, the browser fetches the font bytes in non-CORS mode, stores the response in cache. When the browser later processes the `@font-face` rule and initiates the actual font fetch in CORS mode, it cannot use the cached non-CORS response — it must fetch again. The font is downloaded twice, the preload is wasted, and the font may still appear late. The fix is always including `crossorigin` (or `crossorigin="anonymous"`) on any `<link rel="preload" as="font">`.
The trap: Thinking `crossorigin` is only needed for cross-origin fonts. Same-origin fonts also require it.

**Q (High): What is `fetchpriority` and how does it relate to preload?**
Answer: `fetchpriority` is a hint to the browser's resource scheduler that can be set to `high`, `low`, or `auto`. It is separate from preload: preload changes *when* a resource is discovered (earlier), while `fetchpriority` changes how the browser ranks it against other concurrent fetches. Without `fetchpriority="high"` on an LCP image preload, the browser may still deprioritize the image fetch behind render-blocking CSS or scripts. Setting both ensures the image is discovered early AND fetched at high priority. `fetchpriority` can also be set directly on `<img>`, `<script>`, and `<link>` elements without preload. For example, `<img fetchpriority="high">` on the LCP image in the initial HTML is often sufficient without any preload, because the preload scanner already finds the `<img>` — `fetchpriority` just ensures it wins against competing resources.
The trap: Assuming a preloaded resource automatically has high priority. The `as` attribute sets a default priority for the type, but `fetchpriority` overrides it.

**Q (Medium): What is the difference between `preconnect` and `dns-prefetch`?**
Answer: `dns-prefetch` performs DNS resolution only — it resolves the domain to an IP address. This saves 20–120ms per origin. `preconnect` performs DNS resolution plus TCP handshake plus TLS negotiation. On a modern site over HTTPS, the connection setup phase (before the first byte is sent) can be 100–500ms on mobile. `preconnect` eliminates all of this for connections that will definitely be used early. The trade-off is cost: each open connection consumes CPU for TLS and memory for the connection state. Use `preconnect` for at most 2–3 critical origins you are certain will be needed immediately. Use `dns-prefetch` for origins that might be needed or are needed later (analytics, third-party scripts that load after the main content). Using `preconnect` for everything wastes resources and can actually slow the page down.
The trap: Recommending `preconnect` for all third-party origins without mentioning its cost.

**Q (Medium): How do you correctly preload a responsive LCP image that uses `srcset`?**
Answer: A plain `<link rel="preload" as="image" href="/hero.webp">` preloads a single fixed URL. If the page uses `<img srcset="...">`, the browser will choose the correct srcset candidate based on viewport width, which may differ from the preloaded URL. The solution is `imagesrcset` and `imagesizes` on the preload link: `<link rel="preload" as="image" imagesrcset="hero-400.webp 400w, hero-800.webp 800w" imagesizes="(max-width: 600px) 100vw, 50vw" fetchpriority="high">`. This tells the browser exactly which image candidate to preload based on the current viewport, matching what the `<img srcset>` will eventually select. Without this, either the wrong image is preloaded (wasted bandwidth) or no match occurs and the browser sees a "preloaded but not used" warning.
The trap: Not knowing `imagesrcset` and `imagesizes` exist on `<link>` elements.

**Q (Low): A DevTools audit shows "resource was preloaded but not used within 3 seconds." What are the possible causes?**
Answer: The preloaded resource was fetched but no element consumed it. Possible causes: (1) The `as` attribute is wrong — the preloaded response is keyed by URL and type; if `as="script"` is used but the resource is fetched as a stylesheet, they don't match. (2) The `crossorigin` attribute is missing on a font preload — the font is fetched in CORS mode and the preloaded non-CORS response is a cache miss. (3) The resource URL differs between the preload and the actual fetch — a query string, trailing slash, or protocol mismatch. (4) The component that needs the resource is not rendered on this page — a preload added speculatively for a conditional feature that didn't activate. (5) The resource is truly not needed and the preload should be removed.

---

## Self-Assessment

Before moving on, check off each item you can do WITHOUT looking at the file.

- [ ] Describe the four main resource hints and a concrete use case for each
- [ ] Explain why `crossorigin` is required on font preloads and what breaks without it
- [ ] Explain the difference between `fetchpriority` and `preload` and when you need both
- [ ] Describe when to use `preconnect` vs `dns-prefetch` and the cost of overusing `preconnect`
- [ ] Write the correct HTML to preload a responsive LCP image using `imagesrcset` and `imagesizes`
- [ ] List three ways a preload hint can be "wasted" (consumed twice or not at all)

---
*Next: Code Splitting & Dynamic Imports — resource hints can reduce the cost of loading split chunks; understanding why chunks are split and how they are loaded is the prerequisite.*
