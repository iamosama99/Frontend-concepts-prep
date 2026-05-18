# PerformanceObserver

## Quick Reference

| Entry Type | Measures | Key Properties |
|---|---|---|
| `largest-contentful-paint` | LCP | `startTime`, `element`, `size`, `url` |
| `layout-shift` | CLS (individual shifts) | `value`, `hadRecentInput`, `sources` |
| `first-input` | FID / INP candidate | `processingStart`, `startTime`, `name` |
| `event` | INP (all interactions) | `duration`, `processingStart`, `startTime` |
| `long-task` | Tasks >50ms | `startTime`, `duration`, `attribution` |
| `navigation` | Page load timing | `domContentLoadedEventEnd`, `loadEventEnd`, TTFB |
| `resource` | Per-resource load timing | `startTime`, `responseEnd`, `transferSize` |
| `paint` | FP / FCP | `startTime`, `name` ('first-paint', 'first-contentful-paint') |
| `longtask` | Long animation frames | `duration`, `blockingDuration` |

---

## What Is This?

`PerformanceObserver` is the programmatic API for receiving performance timeline entries. The browser already measures everything — LCP, layout shifts, long tasks, resource loads, navigation timing — and records them in the Performance Timeline. `PerformanceObserver` is how your code subscribes to those measurements as they happen.

This is the foundation of all Real User Monitoring (RUM): instead of running Lighthouse in a lab, you observe actual user sessions and send the metrics to your analytics backend.

---

## Why It Exists

Before `PerformanceObserver`, you had `performance.getEntriesByType()` — a snapshot API that returns entries already in the buffer. But for metrics like LCP, you don't know when the LCP candidate will be determined — it evolves as the page loads. You'd have to poll. `PerformanceObserver` replaces polling with event-driven observation: metrics arrive in your callback as they're determined, even if that's several seconds into the page load.

> **Check yourself:** Why does LCP require an observer rather than a single `performance.getEntriesByType('largest-contentful-paint')` call at `window.load`?

---

## Basic Pattern

```js
const observer = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    console.log(entry.entryType, entry.startTime, entry);
  }
});

observer.observe({
  type: 'largest-contentful-paint',
  buffered: true  // include entries that occurred before this observer was created
});
```

`buffered: true` is critical. By the time your JavaScript runs, many performance entries have already been recorded (FCP often fires at 1-2 seconds, well before most script executes). Without `buffered: true`, you miss everything that happened before the observer was registered.

---

## Measuring Core Web Vitals

### LCP (Largest Contentful Paint)

LCP is reported progressively — each time a larger content element renders, a new entry is added. The *last* entry before the user first interacts or the page unloads is the final LCP value.

```js
let lcpValue = 0;
let lcpElement = null;

const lcpObserver = new PerformanceObserver((list) => {
  // Each entry may supersede the previous
  for (const entry of list.getEntries()) {
    lcpValue = entry.startTime;
    lcpElement = entry.element;
  }
});

lcpObserver.observe({ type: 'largest-contentful-paint', buffered: true });

// LCP is "locked in" on first user interaction or page hide
const reportLCP = () => {
  lcpObserver.disconnect();
  sendMetric('lcp', lcpValue, { element: lcpElement?.tagName });
};

addEventListener('visibilitychange', () => {
  if (document.visibilityState === 'hidden') reportLCP();
}, { once: true });
['click', 'keydown', 'touchstart'].forEach(event => {
  addEventListener(event, reportLCP, { once: true, capture: true });
});
```

### CLS (Cumulative Layout Shift)

CLS is the sum of individual layout shift `value`s, excluding shifts that follow recent user input (within 500ms of an interaction).

```js
let clsValue = 0;
let clsEntries = [];

const clsObserver = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    // Exclude shifts caused by user interaction (intentional movement)
    if (!entry.hadRecentInput) {
      clsValue += entry.value;
      clsEntries.push(entry);
    }
  }
});

clsObserver.observe({ type: 'layout-shift', buffered: true });

// Report on page hide
addEventListener('visibilitychange', () => {
  if (document.visibilityState === 'hidden') {
    sendMetric('cls', clsValue);
  }
});
```

Entry `sources` tell you which elements shifted:

```js
entry.sources.forEach(source => {
  console.log('Shifted element:', source.node, 'from', source.previousRect, 'to', source.currentRect);
});
```

### INP (Interaction to Next Paint)

INP measures responsiveness. The `event` entry type captures all interactions. INP is the 98th percentile interaction duration (or the worst for short sessions):

```js
let worstInteractionDuration = 0;

const inpObserver = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    if (entry.duration > worstInteractionDuration) {
      worstInteractionDuration = entry.duration;
    }
  }
});

inpObserver.observe({ type: 'event', buffered: true, durationThreshold: 16 });
// durationThreshold: only report events longer than 16ms
```

### Long Tasks

Long tasks (>50ms on the main thread) cause jank. Observing them helps identify performance bottlenecks:

```js
const longTaskObserver = new PerformanceObserver((list) => {
  for (const task of list.getEntries()) {
    console.warn(`Long task: ${task.duration.toFixed(0)}ms`, task.attribution);
    sendMetric('long_task', task.duration, {
      container: task.attribution[0]?.containerName,
    });
  }
});

longTaskObserver.observe({ type: 'longtask', buffered: true });
```

---

## `buffered: true` vs. `performance.getEntriesByType()`

```js
// Snapshot — only what's in the buffer right now:
const existingEntries = performance.getEntriesByType('largest-contentful-paint');

// Observer — everything now + future:
observer.observe({ type: 'largest-contentful-paint', buffered: true });
// buffered: true means: include existing entries + continue watching
```

The observer approach with `buffered: true` is equivalent to reading the current buffer *plus* watching for future entries — it's the union of both approaches.

---

## Multiple Types in One Observer

```js
const observer = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    switch (entry.entryType) {
      case 'navigation': handleNavigation(entry); break;
      case 'resource': handleResource(entry); break;
      case 'paint': handlePaint(entry); break;
    }
  }
});

// Can use observe() multiple times, or use entryTypes (deprecated in favor of type):
observer.observe({ type: 'navigation', buffered: true });
observer.observe({ type: 'resource', buffered: true });
observer.observe({ type: 'paint', buffered: true });
```

---

## Resource Timing

Every network request generates a resource timing entry. Useful for identifying slow resources:

```js
const resourceObserver = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    const duration = entry.responseEnd - entry.startTime;
    if (duration > 1000) {
      console.warn(`Slow resource: ${entry.name} (${duration.toFixed(0)}ms)`);
    }
  }
});

resourceObserver.observe({ type: 'resource', buffered: true });
```

Key resource timing properties:

```
startTime → domainLookupStart → domainLookupEnd
  → connectStart → connectEnd
  → requestStart → responseStart → responseEnd
```

`responseStart - requestStart` = server response time (TTFB for that resource)
`responseEnd - responseStart` = download time

---

## Gotchas

**LCP stops updating after first interaction.** The browser stops recording new LCP candidates once the user interacts. This is by design — LCP measures perceived load before interaction.

**`visibilitychange` to 'hidden' is the most reliable "page exit" event.** `beforeunload` and `unload` are unreliable on mobile (page may be killed without firing them). `pagehide` and `visibilitychange` are the modern approach.

**Some entry types require same-origin.** Cross-origin resource timing entries have most fields zeroed out for privacy (Timing-Allow-Origin header can unlock them).

**Buffer limits.** Browsers have per-type buffer limits. Entries beyond the limit are dropped. Use the observer (not `getEntriesByType`) for entry types that may exceed the buffer.

**`disconnect()` stops future observations but doesn't clear the buffer.** Entries already collected are still accessible via `list.getEntries()` from within the last callback.

---

## Interview Questions

**Q (High): Why must you use `buffered: true` when observing LCP, and what happens if you don't?**

Answer: LCP entries are recorded early in the page load timeline — often at 1-2 seconds. JavaScript typically runs later. By the time your `PerformanceObserver` is registered (after DOMContentLoaded, or when your bundle parses), the LCP entry for the initial page load may have already been recorded in the browser's performance buffer. Without `buffered: true`, the observer only receives entries that are recorded *after* it's created — it misses everything that came before. With `buffered: true`, the observer immediately delivers all entries of that type currently in the buffer, then continues watching for new ones. For LCP specifically, missing the buffered entry means your RUM metric is missing data for a large fraction of page loads where LCP settled before the observer registered.

**Q (High): How do you correctly measure CLS? What does `hadRecentInput` mean and why does it matter?**

Answer: CLS is the sum of all individual `layout-shift` entry `value` fields, but only for shifts that weren't caused by user interaction. The `hadRecentInput` flag is `true` when the layout shift occurred within 500ms of a pointer down, key press, or tap. Shifts caused by interaction are expected — clicking a button that expands content is intentional movement, not a surprise shift. Including those would penalize intentional UI. So the correct CLS sum filters: `if (!entry.hadRecentInput) clsValue += entry.value`. CLS continues accumulating throughout the page session. The final value is reported on `visibilitychange` to `hidden` — when the page is backgrounded or closed. This captures the full accumulated shift for the session, not just the initial load.

**Q (Medium): What is the difference between the `longtask` entry type and the `event` entry type, and what metric is each used for?**

Answer: `longtask` entries represent tasks on the main thread that took longer than 50ms — the threshold beyond which frames are dropped and the browser appears unresponsive. These are used to identify JavaScript execution, style/layout work, or other main-thread activity that causes jank. The metric informed by long tasks is TBT (Total Blocking Time) in lab tools. `event` entries represent user interaction events (clicks, key presses, pointer events) and capture when the event was dispatched, when the main thread started processing it, and when it finished. The gap from `startTime` to `processingStart` is the input delay (time waiting for the main thread). The total `duration` is the metric for INP (Interaction to Next Paint). `longtask` finds what's blocking; `event` finds how interactions feel to users.

**Q (Low): A third-party script loads cross-origin resources. The resource timing entries for those resources have all their timing fields set to zero. Why, and how can the resource owner fix it?**

Answer: Cross-origin resources are subject to the Timing Allow Origin restriction. Exposing the detailed timing breakdown (DNS lookup, TCP connect, request start, response start, response end) for cross-origin resources would allow timing attacks — inferring whether a cross-origin page exists or when it last changed based on cache timing. By default, the browser zeroes out all sub-resource timing fields for cross-origin requests. Only `startTime`, `fetchStart`, `responseEnd`, and `transferSize` remain. The resource owner (CDN or third-party script host) can opt in to exposing their timing data by setting the response header `Timing-Allow-Origin: *` (or a specific origin). This signals that the owner consents to the timing data being observable from the page that loaded the resource.

---

## Self-Assessment

- [ ] Implement a complete LCP measurement that handles the progressive update and reports on page hide
- [ ] Explain what `buffered: true` does and why it's required for most metric types
- [ ] Implement a CLS accumulator that correctly filters `hadRecentInput` entries
- [ ] Explain what `longtask` entries measure vs. `event` entries
- [ ] Describe the Timing-Allow-Origin header and why cross-origin timing is blocked by default

---
*Next: Observer APIs vs. Event Listener Patterns — when to choose async observation over synchronous event handling.*
