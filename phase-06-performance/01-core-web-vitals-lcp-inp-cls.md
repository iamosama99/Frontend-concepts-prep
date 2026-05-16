# Core Web Vitals: LCP, INP, CLS

## Quick Reference

| Metric | What it measures | Good | Poor | Measured from |
|---|---|---|---|---|
| LCP | Largest content paint (loading) | ≤ 2.5s | > 4.0s | Navigation start |
| INP | Interaction to Next Paint (interactivity) | ≤ 200ms | > 500ms | User interaction |
| CLS | Cumulative Layout Shift (visual stability) | ≤ 0.1 | > 0.25 | Page lifetime |

---

## What Is This?

Core Web Vitals are a set of real-user metrics defined by Google that measure the quality of a user's experience across three dimensions: loading performance (LCP), runtime interactivity (INP), and visual stability (CLS). They are the user-facing performance contract that maps directly to how people perceive whether a page is fast and stable.

They matter for two reasons that compound each other. First, they map to real user frustration — a high CLS means elements jump under the user's cursor; a high INP means the UI freezes after clicks. Second, Google uses Core Web Vitals as a ranking signal in search, so poor scores carry an SEO cost.

> **Check yourself:** What is the difference between FID (now deprecated) and INP? Why did Google replace one with the other?

---

## Why Does It Exist?

Before Core Web Vitals, performance metrics were largely lab-based: Lighthouse scores, synthetic load times in a controlled environment. These correlate poorly with real user experience because they cannot capture interaction responsiveness under real-world CPU load, layout shifts from late-loading ads, or how the metric behaves across the actual distribution of user devices.

Core Web Vitals are field metrics — collected from Chrome users via the Chrome User Experience Report (CrUX). The 75th percentile of a URL's real-user distribution determines its score. This means you are not optimizing for the best case; you are optimizing so that the slowest 25% of your users still get an acceptable experience.

---

## How It Works

### LCP — Largest Contentful Paint

LCP measures when the largest visible element in the viewport has finished rendering. The browser considers text blocks, `<img>`, `<image>` inside SVG, `<video>` poster frames, and CSS background images (only if loaded via `url()`).

**What the browser tracks:**

The browser emits `LargestContentfulPaint` performance entries as candidates appear. The last entry before the user first interacts (scroll, click, keypress) is the final LCP. Once the user interacts, candidate tracking stops because the viewport has changed intentionally.

```js
// Observe LCP using the Performance Observer API
const observer = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    console.log('LCP candidate:', entry.startTime, entry.element);
  }
});

observer.observe({ type: 'largest-contentful-paint', buffered: true });
```

**Common LCP elements:**

- Hero image (`<img>` or CSS background)
- Large heading text painted above the fold
- Video poster frame
- A server-rendered content block

**Root causes of slow LCP:**

| Cause | Fix |
|---|---|
| Render-blocking resources (CSS, synchronous JS in `<head>`) | Defer non-critical JS, inline critical CSS |
| Slow server response (TTFB > 600ms) | CDN, SSR caching, edge rendering |
| LCP image not prioritized | `<img loading="eager" fetchpriority="high">` + preload hint |
| LCP image loaded as CSS background | Convert to `<img>` so browser can prioritize it |
| Large uncompressed images | WebP/AVIF, correct sizing, responsive images |

```html
<!-- Prioritize the LCP image — tell the browser this is critical -->
<link rel="preload" as="image" href="/hero.webp" fetchpriority="high" />

<img
  src="/hero.webp"
  alt="Hero"
  width="1200"
  height="600"
  fetchpriority="high"
  loading="eager"
/>
```

> **Check yourself:** Why does putting a `fetchpriority="high"` attribute on an image without a matching `<link rel="preload">` still help LCP, even for images that are in the initial HTML?

---

### INP — Interaction to Next Paint

INP replaced FID (First Input Delay) in March 2024. FID only measured the delay before the browser *started* processing the first input. INP measures the full duration from any user interaction (click, keypress, tap) to when the browser finishes painting the resulting visual change.

INP is the 98th percentile of all interaction latencies measured during the page lifetime — not just the first interaction, not the worst outlier.

**Why FID was insufficient:**

FID could be 50ms (good) if the first click happened to land between long tasks. But if every subsequent click took 800ms because the main thread was busy, FID never knew. INP captures the full picture.

**The interaction lifecycle:**

```
User interaction
    → Input event queued (start of delay)
    → Main thread available (end of delay, start of processing)
    → Event handlers execute
    → React/framework re-renders
    → Browser layout
    → Browser paint
    → Frame committed to screen ← INP measured to here
```

**Diagnosing slow INP:**

```js
const observer = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    // entry.duration = total interaction time
    // entry.processingStart - entry.startTime = input delay
    // entry.duration - entry.processingEnd = presentation delay
    if (entry.duration > 200) {
      console.warn('Slow interaction:', {
        target: entry.target,
        duration: entry.duration,
        inputDelay: entry.processingStart - entry.startTime,
        processingTime: entry.processingEnd - entry.processingStart,
        presentationDelay: entry.duration - entry.processingEnd,
      });
    }
  }
});

observer.observe({ type: 'event', durationThreshold: 16, buffered: true });
```

**Root causes of slow INP:**

| Phase | Cause | Fix |
|---|---|---|
| Input delay | Long task occupying main thread | Break long tasks with `scheduler.yield()` or `setTimeout(fn, 0)` |
| Processing time | Heavy synchronous event handler | Defer non-critical work; memoize expensive computations |
| Processing time | Synchronous state update triggers re-render of entire tree | Memoize components, use `startTransition` for non-urgent updates |
| Presentation delay | Large DOM re-render, forced reflow | Batch DOM writes, avoid layout thrashing |

```js
// Break a long task to keep the main thread available for inputs
async function processLargeList(items) {
  for (let i = 0; i < items.length; i++) {
    processItem(items[i]);

    // Yield every 50 items so the browser can handle inputs
    if (i % 50 === 0) {
      await scheduler.yield(); // or: await new Promise(r => setTimeout(r, 0));
    }
  }
}
```

```js
// In React: defer non-urgent state updates so they don't block the interaction response
import { startTransition } from 'react';

function SearchInput({ onSearch }) {
  const handleChange = (e) => {
    const value = e.target.value;

    // Urgent: update the input value immediately
    setInputValue(value);

    // Non-urgent: the search results can wait — won't block the next paint
    startTransition(() => {
      onSearch(value);
    });
  };

  return <input onChange={handleChange} />;
}
```

> **Check yourself:** An INP audit shows input delay is 0ms but processing time is 480ms. The handler calls `setState` with a new value. What is the most likely cause, and what React-specific tool would you reach for first?

---

### CLS — Cumulative Layout Shift

CLS measures the total unexpected layout shift that occurs during the page's lifetime. Each layout shift is scored as the fraction of the viewport that moved multiplied by the fraction of the distance it moved. CLS accumulates these scores over 5-second session windows, taking the highest window's score.

```
Layout Shift Score = Impact Fraction × Distance Fraction

Impact Fraction: area of viewport affected by the shift
Distance Fraction: maximum distance any element moved, as a fraction of viewport dimension
```

A shift is only counted if it was not caused by user interaction (scroll, click, keypress, touch). A tap that opens a dropdown and pushes content down is not a shift — the user caused it.

**Root causes of CLS:**

| Cause | Example | Fix |
|---|---|---|
| Images without dimensions | `<img src="hero.jpg">` | Always set `width` + `height` or use `aspect-ratio` |
| Late-loading ads/embeds | Ad slot dimensions unknown at render time | Reserve space with `min-height` on the container |
| Web fonts causing FOUT | Font loads late, text reflowed | `font-display: optional` or `font-display: swap` with size-adjust |
| Dynamic content injected above existing content | Cookie banners, notification bars | Insert above-fold banners before paint, or use `position: fixed` |
| Animations that trigger layout | Animating `width`, `height`, `top`, `left` | Animate `transform` and `opacity` only |

```html
<!-- Reserve space for images — prevents layout shift when image loads -->
<img
  src="/product.webp"
  alt="Product"
  width="400"
  height="300"
  style="aspect-ratio: 4/3;"
/>
```

```css
/* Reserve space for a late-loading ad slot */
.ad-slot {
  min-height: 250px; /* standard ad unit height */
  width: 300px;
  content-visibility: auto; /* optional: skip rendering until near viewport */
}
```

```css
/* Font swap with size-adjust to minimize FOUT layout shift */
@font-face {
  font-family: 'Roboto';
  src: url('/fonts/roboto.woff2') format('woff2');
  font-display: swap;
  size-adjust: 100.6%; /* nudge fallback font to match Roboto metrics */
  ascent-override: 92.7%;
  descent-override: 24.4%;
}
```

> **Check yourself:** A cookie banner is injected into the top of the page 500ms after load and shifts all content down. Why does this count as CLS even though users see it every visit, and how would you fix it without removing the banner?

---

## Measurement & Tooling

**Field data (real users):**

- Google Search Console → Core Web Vitals report
- Chrome UX Report (CrUX) API — URL-level and origin-level data
- `web-vitals` npm package — collect from your own users

```js
import { onLCP, onINP, onCLS } from 'web-vitals';

function sendToAnalytics({ name, value, rating, navigationType }) {
  navigator.sendBeacon('/analytics', JSON.stringify({
    metric: name,
    value: Math.round(name === 'CLS' ? value * 1000 : value),
    rating,      // 'good' | 'needs-improvement' | 'poor'
    navigationType,
  }));
}

onLCP(sendToAnalytics);
onINP(sendToAnalytics);
onCLS(sendToAnalytics);
```

**Lab data (controlled, for debugging):**

- Lighthouse (DevTools, CI via `lighthouse-ci`)
- WebPageTest — detailed waterfall, filmstrip, real device testing
- Chrome DevTools Performance panel — record traces for INP investigations

**Important caveat:** Lab data cannot measure INP or CLS accurately because there are no real users interacting with the page. Use lab tools to identify long tasks and measure LCP in a controlled environment; use field data to assess actual INP and CLS.

---

## The 75th Percentile Rule

Your Core Web Vitals "pass" when at least 75% of page loads at a given URL fall in the "Good" range. This has two critical implications:

1. Optimizing for the median is not enough. A p50 LCP of 2.0s can coexist with a p75 of 4.5s if the slow tail is wide enough.
2. Segment by device type. Desktop CrUX data and mobile CrUX data are separate. A page that passes on desktop may fail on mobile because mobile CPUs are slower (INP), networks are slower (LCP), and ad injection is more common (CLS).

---

## Gotchas

**LCP candidate can change.** The browser emits multiple LCP candidates as content paints. A hero text heading might be the first candidate, then a hero image loads and becomes the final LCP. Only the last candidate before interaction counts — don't optimize for the wrong element.

**INP is not average; it is 98th percentile.** One catastrophically slow interaction (a cold-start service worker, a heavy computation on first render) can tank INP even if 97% of interactions are fast. Find outliers, not just the average.

**CLS is cumulative but windowed.** Layout shifts more than 5 seconds apart reset the window. A page that has constant small shifts spread out over a long session will accumulate a higher CLS than the same shifts compressed into one window.

**`font-display: swap` can worsen CLS.** Swap shows the fallback font immediately, then swaps to the web font when loaded, causing a reflow. If the two fonts have different metrics, this is a layout shift. Use `font-display: optional` (don't show fallback at all) or apply `size-adjust`/`ascent-override`/`descent-override` to close the gap.

**CLS from `position: sticky` or `position: fixed` elements.** Sticky elements do not contribute to CLS themselves, but content around them can shift if their size changes unexpectedly. A sticky header that grows on resize can push all content down.

---

## Interview Questions

**Q (High): Explain what INP measures and why it replaced FID as a Core Web Vital.**
Answer: FID measured only the input delay before the browser began processing the first user interaction — it captured how long the main thread was blocked before handling a click, but not how long the handler took or how long paint took afterward. A page could have a 10ms FID but 800ms interaction latency on every subsequent click and score fine. INP measures the full duration from any interaction (click, tap, keypress) to the next paint, and it represents the 98th percentile across all interactions during the session. It is a far more representative measure of whether a page feels responsive under real usage. The fix space also broadens: not just reducing long blocking tasks (which improved FID) but also optimizing event handlers, reducing re-render scope, and minimizing layout/paint work after interactions.
The trap: Describing INP as "the same as FID but for all interactions." The key difference is that INP includes processing time and presentation delay, not just input delay.

**Q (High): A page has a good LCP but poor INP. Walk through how you would diagnose and fix it.**
Answer: Start by decomposing the INP into its three phases using the `event` PerformanceObserver: input delay (main thread was busy), processing time (event handlers took too long), and presentation delay (post-handler rendering was slow). If input delay is high, find long tasks blocking the main thread — use the Performance panel to identify them and break them up with `scheduler.yield()`. If processing time is high, profile the event handler: is it doing expensive synchronous computation? Is it triggering a React re-render across a large component tree? Memoize components, use `useMemo` for expensive derivations, and wrap non-urgent state updates in `startTransition` so they don't block the interaction response. If presentation delay is high, the issue is in the browser's layout and paint cycle — look for layout thrashing (reading layout properties then writing them in a loop), or a very large, complex DOM subtree being re-rendered.
The trap: Jumping to "use React.memo everywhere" without first decomposing which phase is slow.

**Q (High): How is CLS scored, and how would you fix a cookie banner that causes a layout shift?**
Answer: CLS is scored as the sum of layout shift scores within a 5-second session window. Each shift is `impact fraction × distance fraction` — the area of the viewport affected by the element's movement times the fractional distance it traveled. A cookie banner injected above content after load causes a shift because it was not anticipated by the initial layout. Three fixes: (1) Reserve space for the banner in the initial HTML with an empty `div` of the correct height so no content moves when the banner appears. (2) Use `position: fixed` or `position: sticky` at the bottom of the viewport — fixed-position elements do not participate in document flow and their appearance doesn't shift other elements. (3) Inject the banner server-side so it is present in the initial paint and no shift occurs. The wrong fix is making the banner appear instantly via JS — if it still inserts itself into normal flow, the shift still happens.
The trap: Thinking that because users always see the banner it "doesn't count." CLS counts unexpected shifts regardless of their frequency.

**Q (Medium): Why is the 75th percentile used for Core Web Vitals assessment rather than the median?**
Answer: The 75th percentile ensures you are optimizing for a large fraction of real users, not just the typical case. A p50 metric can mask a very bad tail: a page with a p50 LCP of 1.5s but a p75 of 4.8s means a quarter of users have a poor experience. By using p75 as the threshold, Google forces developers to fix the slow tail rather than just the happy path. Practically, this means you need to look at your real-user distribution segmented by device type, connection speed, and geography — because the p75 for mobile users in a low-bandwidth region may be entirely different from desktop users on broadband.
The trap: Confusing "75th percentile passes" with "75% of users pass." The p75 is the threshold; you need 75% of loads to be in the Good range.

**Q (Medium): What causes FOUT and how does it relate to CLS?**
Answer: FOUT (Flash of Unstyled Text) occurs when the browser renders text in a fallback font and then re-renders it in the web font after it loads. If the fallback and web font have different metrics — different x-height, line-height, or character width — the text block occupies a different amount of space before and after the swap. This size change shifts surrounding content and contributes to CLS. Fixes: `font-display: optional` skips the fallback entirely, showing no text until the font is available (no swap, no shift); `size-adjust`, `ascent-override`, and `descent-override` in the `@font-face` declaration adjust the fallback font's metrics to match the web font, closing the gap so the swap causes minimal or no shift; preloading the web font with `<link rel="preload">` makes it available early enough that the swap either doesn't happen or happens before first paint.
The trap: Recommending `font-display: swap` as a CLS fix. Swap reduces FOIT (invisible text) but can worsen CLS compared to `font-display: block` if the fonts have different metrics.

**Q (Low): What is the difference between LCP field data and lab data, and when do you use each?**
Answer: Lab data (Lighthouse, WebPageTest, Chrome DevTools) measures performance in a controlled environment with no real user interactions, on a fixed simulated device and connection. It is deterministic, great for catching regressions in CI, and useful for profiling and debugging individual elements. Field data (CrUX, web-vitals library) captures the real distribution of user experiences — actual devices, actual connections, actual geographies. Field data is the source of truth for Core Web Vitals assessment because Google's ranking signal uses CrUX, not Lighthouse. Use lab data to identify issues and measure improvements in a controlled way; use field data to validate that real users are experiencing improvements. A common mistake is optimizing Lighthouse scores (lab) without checking if CrUX (field) improves — they can diverge if, for example, the problem only manifests on mobile networks not simulated in lab conditions.

---

## Self-Assessment

Before moving on, check off each item you can do WITHOUT looking at the file.

- [ ] State the thresholds for Good/Poor LCP, INP, and CLS from memory
- [ ] Explain the three phases of an INP interaction and which tools surface each phase
- [ ] Describe how CLS is calculated and give three distinct root causes with fixes
- [ ] Explain why `font-display: swap` can worsen CLS and what the alternative is
- [ ] Write code to collect all three Core Web Vitals from real users and send them to an analytics endpoint
- [ ] Explain why p75 is used and what it means practically for optimization targets

---
*Next: Resource Hinting — preloading the LCP image is one application of resource hints; the full picture covers preload, prefetch, preconnect, and when each is appropriate.*
