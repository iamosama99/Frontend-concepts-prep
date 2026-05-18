# Performance Budgets & Regression Alerts

## Quick Reference

| Budget Type | What It Limits | Example Threshold |
|---|---|---|
| Bundle size budget | JS/CSS/image bytes | Main bundle < 200 KB gzipped |
| Metric budget | Lighthouse score / CWV | LCP < 2.5s (P75), Perf score ≥ 80 |
| Timing budget | Custom milestones | Time-to-interactive < 4s on simulated 3G |
| RUM regression alert | % change in field metric | Alert if P75 LCP grows > 20% vs. 7-day baseline |

---

## What a Performance Budget Is

A performance budget is a quantitative limit on a performance metric. Crossing the limit fails a CI check, blocks a deploy, or triggers an alert. The budget makes performance a first-class engineering constraint alongside correctness and security.

Without budgets, performance degrades by default. Each feature adds a few KB, a few milliseconds, and the cumulative effect is invisible until the product is noticeably slow. Budgets make each regression visible at the commit that introduced it.

---

## Bundle Size Budgets

Bundle size budgets are the most actionable: they catch the problem (a new dependency or unintentional import) at the point of introduction.

**Using `bundlesize` (standalone):**

```json
// package.json
{
  "bundlesize": [
    { "path": "./dist/main.*.js", "maxSize": "200 kB" },
    { "path": "./dist/vendor.*.js", "maxSize": "150 kB" },
    { "path": "./dist/styles.*.css", "maxSize": "50 kB" }
  ],
  "scripts": {
    "check:bundle": "bundlesize"
  }
}
```

**Using webpack-bundle-analyzer for investigation:**

```typescript
// webpack.config.ts
import { BundleAnalyzerPlugin } from 'webpack-bundle-analyzer';

export default {
  plugins: [
    process.env.ANALYZE
      ? new BundleAnalyzerPlugin({ analyzerMode: 'static', reportFilename: 'bundle-report.html' })
      : null,
  ].filter(Boolean),
};
```

```bash
ANALYZE=true npm run build  # opens visual treemap of bundle contents
```

**Vite bundle size via rollup-plugin-visualizer:**

```typescript
// vite.config.ts
import { visualizer } from 'rollup-plugin-visualizer';

export default defineConfig({
  plugins: [visualizer({ open: true, gzipSize: true, brotliSize: true })],
});
```

**Diff reporting on PRs:**

The most actionable format for teams is a PR comment showing the diff:

```
Bundle size diff:
  main.js: 198 KB → 248 KB (+50 KB, +25%) ⚠️  OVER BUDGET
  vendor.js: 142 KB → 142 KB (no change)
  styles.css: 32 KB → 34 KB (+2 KB)

Budget: main.js must be < 200 KB gzipped
```

Tools like `size-limit` or GitHub Actions with `bundlewatch` can post this automatically.

---

## Lighthouse CI Budgets

Lighthouse CI runs the Lighthouse audit on a URL and asserts scores and metric values:

```javascript
// lighthouserc.js
module.exports = {
  ci: {
    collect: {
      url: ['https://staging.myapp.com/', 'https://staging.myapp.com/pricing'],
      numberOfRuns: 5, // median of 5 runs for stability
      settings: {
        preset: 'desktop',
        // Simulate slow 3G for mobile run separately
      },
    },
    assert: {
      preset: 'lighthouse:recommended',
      assertions: {
        // Performance category
        'categories:performance': ['error', { minScore: 0.8 }],

        // Core Web Vitals
        'largest-contentful-paint': ['error', { maxNumericValue: 3000 }], // 3s
        'cumulative-layout-shift': ['error', { maxNumericValue: 0.1 }],
        'total-blocking-time': ['warn', { maxNumericValue: 300 }],        // INP proxy

        // Bundle size
        'unused-javascript': ['warn', { maxLength: 0 }],
        'uses-optimized-images': ['error', { maxLength: 0 }],

        // Accessibility — don't regress while fixing performance
        'categories:accessibility': ['warn', { minScore: 0.9 }],
      },
    },
  },
};
```

**CI pipeline integration (GitHub Actions):**

```yaml
# .github/workflows/perf.yml
name: Performance Budget

on: [pull_request]

jobs:
  lighthouse:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
      - run: npm ci && npm run build

      - name: Start server
        run: npm run preview &
        env:
          PORT: 3000

      - name: Wait for server
        run: npx wait-on http://localhost:3000

      - name: Run Lighthouse CI
        run: npx lhci autorun
        env:
          LHCI_GITHUB_APP_TOKEN: ${{ secrets.LHCI_GITHUB_APP_TOKEN }}
```

---

## RUM Regression Alerts

Synthetic budgets catch regressions in controlled tests. RUM alerts catch regressions in field data — the real user experience.

**What to alert on:**

```typescript
// Pseudocode for a RUM alerting rule
const alert = {
  metric: 'lcp_p75',
  condition: 'greater_than',
  threshold: 3000, // 3 seconds
  comparison_window: '1h',
  baseline_window: '7d',
  min_samples: 100, // don't alert on noise from low traffic
  severity: 'warning',
};

// Or: alert on relative change
const regressionAlert = {
  metric: 'lcp_p75',
  condition: 'percent_change_greater_than',
  threshold: 20, // 20% worse than baseline
  baseline_window: '7d',
  comparison_window: '1h',
};
```

**Building a simple RUM alerting pipeline:**

```typescript
// Aggregate vitals data and check against thresholds
interface VitalsSnapshot {
  metric: string;
  p75: number;
  p95: number;
  sampleCount: number;
  timestamp: number;
}

async function checkVitalsRegression(current: VitalsSnapshot, baseline: VitalsSnapshot): Promise<void> {
  const change = (current.p75 - baseline.p75) / baseline.p75;

  if (current.sampleCount < 100) return; // insufficient data

  if (change > 0.2) { // 20% worse
    await sendAlert({
      metric: current.metric,
      change: `+${(change * 100).toFixed(1)}%`,
      current: `${current.p75}ms`,
      baseline: `${baseline.p75}ms`,
      severity: change > 0.5 ? 'critical' : 'warning',
    });
  }
}
```

---

## A Practical Budget Stack

```
1. Bundle size — enforced in CI on every PR
   Tool: size-limit or bundlesize
   Budget: main bundle < 200 KB gzipped
   Blocks: merge

2. Lighthouse CI — runs against staging on every deploy
   Tool: LHCI
   Budget: Performance ≥ 80, LCP < 3s, CLS < 0.1
   Blocks: deploy to production

3. Synthetic monitor — runs every 5 minutes from 3 regions
   Tool: Datadog Synthetics / Checkly
   Budget: homepage loads in < 4s (desktop), < 6s (simulated mobile 3G)
   Alert: pagerduty if fails 2× in a row

4. RUM alert — watches field data
   Tool: DataDog RUM / custom pipeline
   Budget: P75 LCP < 3s, alert on 20% regression vs. 7-day baseline
   Alert: Slack #perf-alerts (warning), pagerduty (critical)
```

---

> **Check yourself:** Bundle size passes and Lighthouse CI passes, but a week after the deploy RUM shows LCP degraded by 30%. What's your investigation starting point?

---

## Self-Assessment

- [ ] I know what a performance budget is and why it prevents death-by-a-thousand-cuts degradation
- [ ] I can configure `bundlesize` or `size-limit` to fail CI on bundle bloat
- [ ] I know how to set up Lighthouse CI with assertions in a GitHub Actions workflow
- [ ] I understand the difference between synthetic budget failures and RUM regression alerts
- [ ] I know which percentile (P75) to use for CWV thresholds and why
- [ ] I can describe a complete, layered performance budget stack

---

## Interview Q&A

**Q: What is a performance budget and why is it necessary?** `High`

A performance budget is a hard limit on a performance metric (bundle size, Lighthouse score, LCP) that triggers a CI failure or alert when crossed. It's necessary because performance degrades incrementally — each feature adds a little, and no single change is obviously "the" problem. Without budgets, teams only notice degradation after it's significant and widespread. Budgets make each individual regression visible at the commit that caused it, when it's cheapest to fix.

---

**Q: What's the difference between a bundle size budget and a Lighthouse CI budget?** `High`

A bundle size budget limits raw asset bytes — how much JavaScript, CSS, or image data is shipped to the browser. It catches dependency bloat and unintentional imports. A Lighthouse CI budget limits performance *metric* values — LCP, CLS, TBT, Lighthouse score — which reflect the rendered performance on a specific URL in a controlled environment. Both are synthetic (not real users), but they catch different things: bundle size catches what's being shipped; Lighthouse CI catches how it performs when loaded. You need both, and both should run in CI.

---

**Q: Why isn't Lighthouse CI sufficient for catching all performance regressions?** `Medium`

Lighthouse CI runs in a controlled environment: a single machine, a single network condition, a single geographic location, a single simulated device. Real users have different devices, networks, geographic distances to your CDN, and browser caches. A regression that only affects mobile users on 3G in Southeast Asia won't appear in Lighthouse CI (which tests desktop or simulated conditions from your CI server location). RUM field data catches these because it's measuring actual user experiences across the full distribution.

---

**Q: How do you avoid alerting on performance noise in RUM?** `Medium`

Set a minimum sample count before triggering an alert — don't alert on a 30% regression if only 5 users have loaded the page. Use a comparison window and baseline window instead of absolute values — compare the last hour's P75 against the 7-day rolling baseline, which smooths out time-of-day variation. Use percent change thresholds rather than absolute values when baselines are variable. And use severity tiers: a 15% regression gets a Slack notification; a 50% regression pages on-call.

---

**Q: What does a PR bundle size diff comment tell you that a simple pass/fail doesn't?** `Low`

The diff shows which chunk changed and by how much, making it immediately clear which import or dependency is responsible for the size increase. A simple pass/fail just tells you whether you're over budget — you'd have to manually investigate the cause. A diff like "main.js +50 KB" prompts the developer to check what they imported that's 50 KB. Combined with a bundle analyzer treemap, they can identify the specific library and decide whether it's worth the cost. This makes performance a design conversation at review time, not a firefighting exercise after deploy.
