# Real User Monitoring (RUM) vs. Synthetic Monitoring

## Quick Reference

| | Real User Monitoring (RUM) | Synthetic Monitoring |
|---|---|---|
| Data source | Actual user browsers in production | Automated scripts in controlled environments |
| Traffic | All users / sampled production traffic | Scheduled probes (every 1–5 min) |
| Variability | High — real network, real devices | Low — reproducible, controlled |
| Pre-launch use | No (requires real users) | Yes — test before users see it |
| Regression detection | Slow (needs traffic to accumulate) | Fast (runs on every deploy) |
| Coverage | Reflects real user distribution | Only tests defined paths |

---

## What Each Tool Answers

**RUM answers:** "What are my real users actually experiencing?"
- P75 / P95 LCP for users on 4G in India
- How CLS varies by country and device type
- Which pages have the worst INP in production
- Memory growth trends across user sessions

**Synthetic answers:** "Is the application working as expected in a controlled test?"
- Did this deploy break the homepage load time?
- Is the critical path (login → dashboard → key action) still completing under threshold?
- Is the app reachable from multiple geographic regions right now?

Neither is a substitute for the other. They're complementary layers.

---

## Real User Monitoring

RUM instruments the browser using the Performance and Web Vitals APIs to collect metrics from actual user sessions. Data is sent back to an analytics endpoint for aggregation.

**Core Web Vitals via the `web-vitals` library:**

```typescript
import { onLCP, onINP, onCLS, onFCP, onTTFB } from 'web-vitals';

function sendToAnalytics(metric: { name: string; value: number; rating: string }): void {
  navigator.sendBeacon('/analytics/vitals', JSON.stringify({
    metric: metric.name,
    value: metric.value,
    rating: metric.rating, // 'good' | 'needs-improvement' | 'poor'
    url: window.location.pathname,
    timestamp: Date.now(),
  }));
}

onLCP(sendToAnalytics);
onINP(sendToAnalytics);
onCLS(sendToAnalytics);
onFCP(sendToAnalytics);
onTTFB(sendToAnalytics);
```

**Collecting custom timing:**

```typescript
// Instrument a specific user flow
function measureCheckoutFlow(): void {
  const startMark = 'checkout:start';
  const endMark = 'checkout:complete';

  performance.mark(startMark);

  // ... checkout steps ...

  performance.mark(endMark);
  performance.measure('checkout-duration', startMark, endMark);

  const [measure] = performance.getEntriesByName('checkout-duration');
  sendToAnalytics({ name: 'checkout_duration', value: measure.duration, rating: 'measured' });
}
```

**Sampling for high-traffic sites:**

```typescript
// Only instrument 10% of sessions to reduce overhead and cost
const SAMPLE_RATE = 0.1;

if (Math.random() < SAMPLE_RATE) {
  onLCP(sendToAnalytics);
  onINP(sendToAnalytics);
  onCLS(sendToAnalytics);
}
```

**RUM data characteristics:**
- Represents the full distribution of user experience
- Segments naturally by browser, device, geography, connection type
- Requires significant traffic to get statistically significant P95/P99 values
- Has delay — you see data after users experience it, not before

---

## Synthetic Monitoring

Synthetic monitoring runs automated browser scripts on a schedule from controlled environments. It's deterministic — the same test, the same environment, every time.

**What it catches that RUM doesn't:**
- Broken deploys (synthetic runs immediately; RUM requires users to visit)
- Geographic availability (run from multiple regions)
- Regression from a specific deploy (compare before/after on a controlled baseline)

**Synthetic tools:**
- **Lighthouse CI** — runs Lighthouse in CI/CD, compares scores to a threshold
- **WebPageTest** — detailed waterfall analysis, filmstrip, available via API
- **Datadog Synthetics / Checkly** — scheduled browser tests from multiple regions
- **Playwright/Cypress with performance assertions** — custom assertions on timing

**Lighthouse CI configuration:**

```javascript
// lighthouserc.js
module.exports = {
  ci: {
    collect: {
      url: ['https://staging.myapp.com/', 'https://staging.myapp.com/dashboard'],
      numberOfRuns: 3, // run 3 times, take median
    },
    assert: {
      assertions: {
        'categories:performance': ['error', { minScore: 0.8 }], // fail CI if < 80
        'first-contentful-paint': ['warn', { maxNumericValue: 2000 }],
        'largest-contentful-paint': ['error', { maxNumericValue: 3000 }],
        'cumulative-layout-shift': ['error', { maxNumericValue: 0.1 }],
      },
    },
    upload: {
      target: 'lhci',
      serverBaseUrl: 'https://lhci.myapp.com',
    },
  },
};
```

**WebPageTest via API (Node.js):**

```typescript
async function runSyntheticTest(url: string): Promise<SyntheticResult> {
  const response = await fetch(
    `https://www.webpagetest.org/runtest.php?url=${encodeURIComponent(url)}&f=json&runs=3`,
    { headers: { 'X-WPT-API-KEY': process.env.WPT_API_KEY! } }
  );

  const { data } = await response.json();
  const testId: string = data.testId;

  // Poll for results
  let result;
  while (!result?.data?.runs) {
    await new Promise(r => setTimeout(r, 5000));
    const statusRes = await fetch(`https://www.webpagetest.org/jsonResult.php?test=${testId}`);
    result = await statusRes.json();
  }

  return {
    lcp: result.data.median.firstView['chromeUserTiming.LargestContentfulPaint'],
    ttfb: result.data.median.firstView.TTFB,
    speedIndex: result.data.median.firstView.SpeedIndex,
  };
}
```

---

## Using Both Together

The standard observability stack uses both layers:

```
Deploy pipeline:
  1. Lighthouse CI runs on staging → blocks deploy if regressions
  2. Synthetic monitor from Datadog confirms prod is healthy post-deploy
  3. RUM (web-vitals + Sentry) collects field data from real users
  4. Alert: if P75 LCP degrades by > 20% over 24h baseline → page on-call
```

**Key RUM percentiles to track:**

- **P50 (median):** typical user experience
- **P75:** CWV thresholds are measured at P75 — "good" means P75 passes
- **P95:** tail experience — high-end impact for poorest-performing sessions
- **P99:** catches serious outliers — often reveals specific device/network segments

---

> **Check yourself:** Lighthouse CI passes for your deploy, but RUM shows LCP degraded for mobile users in Southeast Asia. What explains the gap, and how do you investigate?

---

## Self-Assessment

- [ ] I can articulate what RUM answers vs. what synthetic monitoring answers
- [ ] I know how to instrument Core Web Vitals with the `web-vitals` library
- [ ] I understand why sampling is used in RUM for high-traffic sites
- [ ] I can configure Lighthouse CI to fail a deploy based on performance thresholds
- [ ] I understand why both approaches are needed and how they complement each other
- [ ] I know which percentile (P75) CWV thresholds are measured at and why

---

## Interview Q&A

**Q: What is the difference between RUM and synthetic monitoring?** `High`

RUM (Real User Monitoring) collects performance data from actual users in production — it measures what your real users experience, across their actual devices, networks, and geographic locations. Synthetic monitoring runs automated browser scripts in controlled environments on a schedule. RUM is the ground truth for user experience; synthetic is the early warning system. Synthetic can run before deployment and from multiple regions on demand; RUM requires real traffic and has inherent latency. A complete observability setup uses both.

---

**Q: Why are Core Web Vitals measured at P75 rather than the median?** `High`

The P75 threshold means 75% of page loads must meet the threshold. Using the median (P50) would mean the worst half of users could have a poor experience and you'd still pass. P75 pushes teams to optimize for a larger share of users, especially those on slower devices and networks. Using P95 or P99 would be too strict — rare, catastrophic outliers (network outages, ancient devices) would make passing impossible even for well-optimized pages. P75 is the pragmatic balance: ambitious enough to drive real improvements, achievable for well-built products.

---

**Q: What does Lighthouse CI do and where does it fit in a deploy pipeline?** `Medium`

Lighthouse CI runs Google's Lighthouse audit tool against a URL on a schedule or as part of a CI/CD pipeline. It reports performance scores and specific metric values, and it can assert thresholds — failing the CI build if scores drop below a configured minimum or if a metric (LCP, CLS, etc.) exceeds a maximum value. This catches performance regressions before they reach production. It's typically run against a staging environment as a pre-deploy gate.

---

**Q: Lighthouse CI passes but users in one region see slow LCP. How do you investigate?** `Medium`

Lighthouse CI runs from a single, controlled server location — it doesn't reflect geographic variance. First, check your RUM data segmented by region: compare LCP P75 for the affected region against others. Then run WebPageTest or Datadog Synthetics from a node in that region to reproduce the synthetic measurement with geographic specificity. Common causes: CDN node not caching in that region, origin server latency to that geography, a resource loaded from a server that's slow from that location. The RUM tells you there's a regional problem; the regional synthetic test helps you reproduce and debug it.

---

**Q: Why use `navigator.sendBeacon` for sending RUM data?** `Low`

RUM data is often sent at the end of a page session — on navigation, tab close, or page hide. `sendBeacon` is guaranteed to complete even when the page is unloading, whereas a `fetch` request may be cancelled if the page closes before the request completes. It also doesn't block the main thread or the page close process. The trade-off: no response reading and limited to `POST`. For analytics and monitoring, this is a non-issue since you don't need a response — you just need the data to arrive reliably.
