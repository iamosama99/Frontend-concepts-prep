# Incremental Static Regeneration (ISR)

## Quick Reference

| Concept | Mechanism | Implication |
|---|---|---|
| stale-while-revalidate | Serve cached page immediately; regenerate in background after TTL expires | Users always get a fast response; content is at most TTL-seconds stale |
| Per-page TTL | Each page defines its own `revalidate` seconds | Fine-grained control — news articles revalidate in 60s, docs in 3600s |
| On-demand ISR | Purge and regenerate a specific page via API call | Content deploys instantly when CMS publishes, without a full build |
| No full rebuild | Only changed pages regenerate | A site with 100k pages can update one page in seconds |

## What Is This?

Incremental Static Regeneration is an extension of SSG where pre-rendered HTML pages are automatically regenerated in the background, page by page, after a configurable time-to-live expires. Users always get served a cached, statically-generated page — so performance is static-file fast — but the cache refreshes automatically, so content doesn't stay stale indefinitely.

The core pattern is HTTP's stale-while-revalidate: serve the cached version immediately, then in the background check if it's stale and regenerate it if so. The next request after regeneration gets the fresh version.

```javascript
// Next.js Pages Router
export async function getStaticProps() {
  const data = await fetchFromCMS();
  return {
    props: { data },
    revalidate: 60  // seconds — this is the ISR TTL
  };
}
```

With `revalidate: 60`: a user at second 0 gets the static page, a user at second 45 gets the same cached static page, a user at second 65 gets the same cached static page AND the framework triggers a background regeneration, a user at second 70 (after the regeneration completes) gets the freshly rendered page.

> **Check yourself:** With `revalidate: 60`, if your database updates at 12:00:00, what's the worst-case time a user could see stale data? What's the best case?

## Why Does It Exist?

SSG forces a binary choice: build everything upfront (slow builds, stale data) or do nothing at build time and fall back to SSR (fast builds, no static benefits). Most content falls between these extremes — it doesn't change with every request (making SSR's per-request render wasteful), but it also shouldn't require a full site rebuild for every update.

ISR targets this middle ground: the vast majority of requests get served from the cache at static-file speed, while the system maintains freshness by regenerating pages asynchronously. The site grows incrementally — only visited pages exist in the cache, and only stale pages regenerate. A 500,000-page catalog doesn't require a 500,000-page build.

It also decouples the deployment pipeline from the content pipeline. With pure SSG, a content team publishing an article must wait for a deploy. With ISR, publishing an article means the next user to visit (after the TTL) gets the new content automatically.

## How It Works

### Time-Based ISR

```
Initial state: page is pre-rendered at build time, cached at edge
User A (t=0):    cache hit → serve cached HTML (fast)
User B (t=45s):  cache hit → serve cached HTML (fast), TTL hasn't expired
User C (t=65s):  cache is stale → serve STALE cached HTML (still fast)
                 background: trigger regeneration (fetch data, re-render)
User D (t=70s):  cache updated → serve FRESH cached HTML (fast)
```

The "stale" user (User C) still gets a fast response — they're never blocked waiting for a re-render. The stale content is a deliberate trade-off for consistent performance.

### On-Demand ISR

Time-based ISR still has a maximum staleness window equal to the revalidate TTL. On-demand ISR eliminates that window by allowing external systems to trigger a page regeneration explicitly.

```javascript
// API route that CMS can call when content is published
// pages/api/revalidate.js (Next.js)
export default async function handler(req, res) {
  if (req.query.secret !== process.env.REVALIDATION_SECRET) {
    return res.status(401).json({ message: 'Invalid token' });
  }

  const path = req.query.path;  // e.g. '/products/123'
  await res.revalidate(path);
  return res.json({ revalidated: true });
}
```

When the CMS publishes an update to product 123, it calls:
```
POST /api/revalidate?secret=xxx&path=/products/123
```

Next.js immediately regenerates that specific page. The next request gets fresh content. No full rebuild, no deploy — just targeted cache invalidation and regeneration.

This is the production pattern for CMS-driven sites: combine a high `revalidate` TTL (e.g., 3600 seconds) as a safety net with on-demand ISR webhooks for instant content updates.

> **Check yourself:** Why is the `secret` query parameter in the on-demand ISR API route important? What happens if it's missing?

## ISR vs. SSR vs. SSG

| | SSG | ISR (time-based) | ISR (on-demand) | SSR |
|---|---|---|---|---|
| Rendering location | Build time | Background, after TTL | Background, on trigger | Per request |
| Content freshness | Stale until rebuild | Stale up to TTL | Near-instant after publish | Always fresh |
| Cold request TTFB | CDN (~10ms) | CDN (stale: ~10ms) | CDN (stale: ~10ms) | Server (100–500ms) |
| Scale | Infinite (CDN) | Infinite (CDN) | Infinite (CDN) | Server-bound |
| Personalization | No | No | No | Yes |

ISR shares SSG's CDN-distributed performance model. The only time a user experiences server latency is during the background regeneration — and even then, that user got the stale response at CDN speed, not the regeneration latency.

## Edge Cases and Behavior

**What happens on the first request after a revalidation triggers but before it completes?** Multiple concurrent requests during regeneration all receive the stale cached version. Once regeneration completes, the cache atomically updates. There's no race condition where some users get the old version and others get the new one within a single regeneration cycle.

**What if regeneration fails (API error, server crash)?** ISR keeps serving the last good cached version. Stale content is better than no content. The next TTL expiry triggers another regeneration attempt.

**Does ISR work with personalized content?** No — ISR is still fundamentally a shared cache. The cached HTML is the same for all users. Personalization (user name, cart, auth state) must still be fetched client-side after hydration.

## Gotchas

**The stale user problem.** The first user after a TTL expires gets stale content. If your TTL is 60 seconds and a price drops, users in that 60-second window see the wrong price. For data where staleness has financial consequences (inventory, pricing), ISR requires supplementing the static HTML with client-side real-time fetches.

**On-demand ISR requires a server.** Calling `res.revalidate()` runs server-side code. Platforms that don't support serverless functions (pure CDN hosting, GitHub Pages) can't support on-demand ISR. You need a platform that has a compute layer alongside the CDN (Vercel, AWS CloudFront + Lambda).

**Cache invalidation across edge nodes.** In a globally distributed CDN, invalidating a cached page means purging it from potentially hundreds of edge nodes worldwide. This takes time (seconds to minutes) to propagate. A user in Tokyo might see stale content for longer than a user in New York after a revalidation because Tokyo's edge node takes longer to receive the cache invalidation signal.

**Revalidation secret exposure.** On-demand ISR requires exposing an API endpoint that triggers page regeneration. Without a secret/token check, anyone who knows the URL can trigger expensive regenerations (DoS via cache stampede). The secret should be in an environment variable, never committed to source.

**ISR doesn't solve the 500,000-page build problem — it changes it.** You still need the initial build to pre-render pages. ISR helps keep them fresh after the fact. The solution to impractical build sizes is combining ISR with `fallback: 'blocking'` — build only your top pages, let the rest render on first access, then keep all pages fresh via ISR revalidation.

## Interview Questions

**Q (High): Explain how ISR's stale-while-revalidate behavior works and what guarantee it makes to users.**

Answer: ISR serves the cached, pre-rendered HTML to every user immediately — CDN-fast, regardless of whether that cache is current or expired. After the configured TTL, the next request that arrives finds the cache stale. That request is still served the stale cached version instantly (no blocking wait), but it triggers an asynchronous background regeneration: the server fetches fresh data, re-renders the page, and updates the cache. Subsequent requests get the freshly rendered version. The guarantee: every user gets a fast response (CDN-served, no server wait). The cost: users who arrive in the stale window (after TTL expires, before regeneration completes) see content that may be up to TTL-seconds old. This is a deliberate trade-off — consistent latency is worth more than maximally fresh content for most use cases.

The trap: Confusing "the user who triggers revalidation waits for the regeneration." They don't — they get the stale version while regeneration happens asynchronously.

---

**Q (High): What is on-demand ISR and how does it differ from time-based ISR?**

Answer: Time-based ISR regenerates a page automatically after a TTL expires — useful as a safety net but means content can be up to TTL-seconds stale. On-demand ISR allows external systems (a CMS, a webhook, an admin action) to explicitly trigger regeneration of specific pages without waiting for a TTL. When a content editor publishes an article, the CMS calls the on-demand ISR API endpoint, which immediately purges the cache for that specific page and triggers a regeneration. The next user gets fresh content. The key difference: time-based ISR is pull-based (check on a schedule), on-demand ISR is push-based (regenerate when told to). In production, both are often combined: on-demand for timely content updates, a high TTL as a fallback safety net for pages where the webhook might fail.

The trap: Not knowing on-demand ISR exists. Many candidates only know about `revalidate: n` and miss the webhook/trigger model entirely.

---

**Q (Medium): How does ISR handle regeneration failures, and what does a user see?**

Answer: If a background regeneration fails — data API is down, server throws an error, timeout — ISR keeps serving the last successfully cached version. The failed regeneration is discarded, and the stale cache remains in place. On the next request after the TTL expires, regeneration will be attempted again. This is the correct behavior: serving stale content is better than serving an error page or blocking users while a retry happens. It means ISR has a graceful degradation property — a transient backend outage causes staleness, not downtime. The implication for engineers: you can't rely on ISR for real-time data where stale = wrong (e.g., prices, inventory) because a downstream failure could keep a cached page stale indefinitely.

The trap: Assuming ISR shows an error page when regeneration fails. It doesn't — it silently keeps the old content, which can be hard to detect in monitoring.

---

**Q (Medium): Can ISR be used for personalized content? Why or why not?**

Answer: No. ISR pages are cached and shared across all users — the HTML file for `/products/123` is the same file served to every visitor. Personalization means the HTML varies per user (showing "Hi, Alice" vs. "Hi, Bob"), which is fundamentally incompatible with a shared cache. The architecture for personalization on ISR sites is a split: static HTML delivers the common structure (product info, layout, navigation links), and client-side JavaScript fetches personalized data after hydration (user name, cart count, recently viewed, auth state). This gives you the performance of static delivery for the structural content and the flexibility of dynamic fetching for the personal bits. For pages where personalized content is the majority of the page (account settings, order history), SSR is more appropriate.

The trap: Claiming ISR can handle personalization, or not knowing the hybrid static + client-side fetch pattern.

---

**Q (Low): How does ISR cache invalidation propagate across a global CDN, and what latency should you expect?**

Answer: When a page is regenerated via ISR, the updated HTML must be distributed to CDN edge nodes worldwide. This isn't instant. Modern CDN architectures (Vercel's Edge Network, Cloudflare) use a combination of cache invalidation signals and the natural TTL-based expiry. Propagation can take seconds to minutes depending on the CDN provider and number of edge nodes. During propagation, a user in a data center that hasn't received the update yet still sees the old cached version. For on-demand ISR triggered by a CMS publish, this means "instant" in practice means "within a minute or two globally." For most content use cases this is acceptable. For time-sensitive content (breaking news, flash sales with exact start times), you need to account for this propagation lag in the user experience design.

The trap: Assuming cache invalidation is globally instant, or not knowing that CDN edge distribution has propagation delays.

---

## Self-Assessment

Before moving on, check off each item you can do WITHOUT looking at the file.

- [ ] Can explain the stale-while-revalidate behavior: who gets the stale response and what happens in the background
- [ ] Can describe what happens when a regeneration fails (serves stale, retries next TTL)
- [ ] Can explain the difference between time-based and on-demand ISR
- [ ] Can describe why ISR cannot serve personalized content
- [ ] Can name the `revalidate` property in `getStaticProps` and explain what it controls
- [ ] Can explain why on-demand ISR requires a secret token in the API endpoint

---
*Next: Streaming SSR — extending server-side rendering by flushing HTML in chunks, letting the browser paint above-the-fold content before below-the-fold content finishes rendering on the server.*
