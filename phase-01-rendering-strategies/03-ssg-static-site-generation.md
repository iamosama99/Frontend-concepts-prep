# Static Site Generation (SSG)

## Quick Reference

| Concept | Mechanism | Implication |
|---|---|---|
| Build-time render | Pages render during `npm run build`, not per-request | Zero server cost at runtime; stale until next build |
| CDN distribution | HTML files served directly from edge nodes globally | Fastest possible TTFB — no compute between request and response |
| Staleness | Content reflects data at build time | Published changes require a new build and deploy |
| Pre-rendered paths | Framework generates one HTML file per known URL | Dynamic paths (user profiles, search results) require workarounds |

## What Is This?

Static Site Generation is the rendering model where all pages are built into HTML files ahead of time — during the build step, not when users request them. The output of `npm run build` is a directory of `.html` files, CSS, and JS assets. These files are uploaded to a CDN, and every request gets a pre-built file served directly from the nearest edge node.

```
Build phase (once, at deploy time):
  /products/headphones  → products/headphones/index.html
  /products/keyboard    → products/keyboard/index.html
  /blog/post-1          → blog/post-1/index.html

Runtime (every request):
  User → CDN edge → serve pre-built HTML file
  No server involved. No render on request.
```

The HTML file has real content in it — not a blank shell. But unlike SSR, no server computed that HTML in response to the request. It was pre-computed during the build.

> **Check yourself:** If a product's price changes in the database after a SSG site is deployed, what does the user see? What's the only way to fix it?

## Why Does It Exist?

SSG takes the most expensive part of SSR — the per-request render — and moves it to a one-time cost at build time. For content that doesn't change per-user or per-request (blog posts, marketing pages, documentation, product catalogs with infrequent updates), there's no reason to re-render the same HTML millions of times. Render it once, cache it everywhere.

The benefits compound: when your pages are static files, you can serve them from a global CDN with TTFB measured in single-digit milliseconds. There's no server to scale, no runtime to crash, no database query under load. The site can absorb a traffic spike that would bring down a dynamic server because CDN edge nodes are designed exactly for this — serving cached files at massive scale.

This is also why static sites have excellent security profiles: with no server executing code, there's no server to compromise, no SQL injection surface, no server-side auth bypass.

## How It Works

### Build Phase

The framework (Next.js, Gatsby, Astro, Eleventy) runs your page components at build time, calls your data-fetching functions, receives data, renders the component tree, and writes the resulting HTML string to a file.

```javascript
// Next.js Pages Router — getStaticProps runs ONCE at build time
export async function getStaticProps({ params }) {
  const product = await db.products.findById(params.id);
  // This is a build-time fetch — not a runtime fetch
  return {
    props: { product }
  };
}

// getStaticPaths tells the framework which URLs to pre-render
export async function getStaticPaths() {
  const products = await db.products.findAll({ select: ['id'] });
  return {
    paths: products.map(p => ({ params: { id: p.id } })),
    fallback: false  // 404 for any path not listed here
  };
}

export default function ProductPage({ product }) {
  // This renders at build time — result is saved as HTML
  return <h1>{product.name}</h1>;
}
```

At build time, Next.js calls `getStaticPaths` to discover all valid product IDs, then calls `getStaticProps` for each one, renders the component, and writes the HTML. The result is hundreds or thousands of pre-rendered files.

### Runtime

No rendering happens at runtime. The CDN just maps URLs to files:

```
GET /products/123
→ CDN serves products/123/index.html
→ File was built at deploy time
→ File is served in ~10ms from edge node nearest the user
```

The browser receives the HTML, parses it, shows content immediately (FCP ~= TTFB), then downloads the JS bundle and hydrates — same hydration process as SSR.

### The Fallback Problem

SSG requires knowing all URLs at build time. For a blog with 100 posts, fine. For a product catalog with 50,000 SKUs, building 50,000 HTML files takes minutes and the build output is enormous. For user-generated content with millions of URLs, it's infeasible.

Next.js `fallback` options address this:
- `fallback: false` — any unbuilt path returns 404
- `fallback: true` — unbuilt paths serve a loading state, render on first request, cache the result
- `fallback: 'blocking'` — unbuilt paths SSR on first request, subsequent requests get the cached result

The last two are hybrid approaches — static for known paths, server-rendered on first request for unknown ones.

> **Check yourself:** A blog has 10,000 posts. You use `getStaticPaths` with `fallback: 'blocking'`. A new post is published. What happens the first time a user visits it? What happens on the second request?

## Content Freshness

SSG's fundamental constraint is that content reflects the data at build time. Fresh data requires a new build. The question is how often you can build and how stale data can be.

**High-frequency builds**: if a CMS publishes a new post, it triggers a build webhook, the site rebuilds in 30 seconds, and the new content is live. For a site with hundreds of pages this is fine. For tens of thousands, rebuild time becomes a problem.

**On-demand rebuilding**: some platforms (Vercel, Netlify) support on-demand ISR (see the next topic) — rebuild only the specific pages that changed, not the entire site.

**Hybrid static/dynamic**: the "shell" of the page (header, layout, product template) is static; the dynamic content (user reviews, real-time stock, personalized pricing) is fetched client-side after hydration. This keeps the benefits of fast static delivery while allowing some parts to update without a rebuild.

## When SSG Is the Right Choice

- **Marketing sites, landing pages** — content changes infrequently, SEO matters, speed matters.
- **Documentation sites** — content is authored, not database-driven, and the whole doc set can rebuild in seconds.
- **Blogs and content sites** — each post is a static HTML file; publishing triggers a build.
- **E-commerce product pages** — the template is static; real-time inventory/price can be loaded client-side.
- **Any page where the content is the same for all users** and changes predictably (not in real time).

SSG is wrong when content must reflect database state within seconds of a change, when the number of URLs makes building all pages impractical, or when pages require per-user personalization in the initial HTML.

## Gotchas

**Stale content is a user-facing bug.** If a product goes out of stock but the SSG page says "In Stock," users will add it to cart and hit an error at checkout. SSG requires designing the data model so that staleness-tolerant content lives in the static HTML, and time-sensitive content is loaded dynamically.

**Large site build times.** A site with 100,000 pages pre-rendering each one takes significant time. If every page requires one database query, that's 100,000 queries during build. Build time scales linearly (at best) with the number of pages. This becomes a deployment bottleneck — a one-minute code change takes 20 minutes to deploy because of build time.

**Content preview / staging is awkward.** Editors want to preview a draft post before publishing. With SSG, the only way to see rendered content is to rebuild — either a full build or a preview build against draft data. CMS platforms have integrated solutions (Contentful preview API, Sanity preview) but it's inherently more friction than SSR preview.

**Client-side fetching undermines static benefits.** If every "static" page makes several API calls after hydration to show dynamic content, the initial static HTML is hollow — just a shell with a loading spinner. You get the rebuild cost of SSG without the content delivery benefit. The static content should be meaningful on its own.

**URL structure must be known at build time.** Dynamic routes that don't enumerate all paths in `getStaticPaths` return 404 unless you use fallback rendering. This forces you to think carefully about what paths exist and whether they can all be enumerated at build time.

## Interview Questions

**Q (High): What is the difference between SSR and SSG, and when would you choose one over the other?**

Answer: The key difference is *when* rendering happens. SSR renders on every request — the server generates HTML fresh for each incoming request, with access to the live database and user context. SSG renders once at build time — HTML files are pre-generated and served statically from a CDN. SSR is better when content changes frequently (within seconds), when content must be personalized per-user in the initial HTML, or when the number of URLs is too large to pre-render. SSG is better when content is relatively stable (updates every few minutes to hours is fine), when the content is the same for all users, and when maximum performance and scalability matter. Most real systems use both: SSG for stable public pages (product listings, blog posts, marketing), SSR for dynamic personalized pages (checkout, account, search results).

The trap: Saying "SSG is always faster" without acknowledging the staleness problem, or "SSR is always better" without recognizing the server cost and scalability difference.

---

**Q (High): What are the content freshness implications of SSG, and how do teams mitigate them?**

Answer: In SSG, content reflects database state at the time of the last build. After deployment, any change in the database is invisible to users until the site rebuilds and redeploys. Mitigations: (1) Webhook-triggered rebuilds — the CMS triggers a build hook on content publish, the CI pipeline runs in 1–5 minutes, and the updated page is live quickly. (2) Client-side data fetching for volatile data — the static HTML shows stable data (product name, description) while JavaScript fetches live data (inventory, pricing) after mount. (3) Incremental Static Regeneration (ISR) — only rebuild individual pages on-demand or after a time interval, rather than the full site. (4) Fallback rendering — new content triggers a server-render on first access, then caches. The appropriate mitigation depends on how quickly staleness becomes a user-visible problem.

The trap: Not acknowledging that staleness is a real product problem, or claiming webhook-triggered builds fully solve it (they don't — there's always a build lag window).

---

**Q (Medium): How does Next.js `getStaticPaths` with `fallback: 'blocking'` work, and what are the trade-offs?**

Answer: `getStaticPaths` tells Next.js which paths to pre-render at build time. With `fallback: 'blocking'`, any request for a path not in the pre-rendered set is handled by the server on first access: it runs `getStaticProps` at request time (like SSR), renders the page, saves the result as a static file, and serves it. Subsequent requests for that path hit the cached static file. Trade-offs: pro — new URLs become available without a full rebuild; the long tail of rarely-visited pages isn't pre-built (saving build time and storage). Con — first visitor to a new path experiences SSR latency (TTFB includes server render time), not static-file speed. This is the right model for large catalogs where you pre-build the top 10,000 products by traffic but let the tail render on-demand.

The trap: Confusing `fallback: true` and `fallback: 'blocking'`. The difference: `true` serves a loading state immediately while rendering in the background (requires handling the loading state in the component); `blocking` holds the request until the render completes (user waits, no loading state needed).

---

**Q (Medium): Why do static sites have a better security posture than server-rendered sites, and what security surface remains?**

Answer: A static site has no server-side compute at runtime — no application server running code, no database connection, no session management, no file system access. The attack surfaces that don't exist: SQL injection (no database queries on request), server-side code execution (no runtime), session hijacking at the server layer (no server sessions), SSRF (no server making requests). What security surface remains: the client-side JavaScript, which has the same XSS surface as any SPA; third-party CDN scripts (supply chain attacks); the build pipeline, which does have server access during builds; and any APIs the static frontend calls (the API has its own attack surface). Static sites move risk from runtime server vulnerabilities to build-time and client-side vulnerabilities.

The trap: "Static sites are completely secure." There's still a client-side attack surface, and the APIs the frontend calls are just as vulnerable as before.

---

**Q (Low): How would you handle a site with 500,000 product pages that need to be SEO-indexed using SSG?**

Answer: Pre-building 500,000 pages is impractical — build time would be enormous and every content update would rebuild pages nobody visits. The pragmatic approach: (1) Pre-build only the high-traffic pages during build time — the top 10,000–50,000 pages by traffic or commercial importance. (2) Use `fallback: 'blocking'` for the remaining ~450,000 pages — they render on first access and get cached as static files. (3) Generate a comprehensive XML sitemap so crawlers know all URLs exist. (4) Accept that new or rarely-visited pages will have SSR-speed first access rather than static-file speed. This approach keeps build times manageable, ensures the most important pages have optimal performance, and still makes all pages crawlable and indexable.

The trap: Either "build all 500,000" (impractical at scale) or "give up on SEO for the tail" (misses the fallback approach).

---

## Self-Assessment

Before moving on, check off each item you can do WITHOUT looking at the file.

- [ ] Can explain when rendering happens in SSG (build time) and contrast it with SSR (request time)
- [ ] Can describe what happens at runtime for a static site — no server, CDN serves a file
- [ ] Can explain the staleness problem and name three mitigation strategies
- [ ] Can explain what `getStaticPaths` does and why it's needed for dynamic routes
- [ ] Can describe the trade-off of `fallback: 'blocking'` and when to use it
- [ ] Can name two scenarios where SSG is appropriate and two where it's the wrong choice

---
*Next: Incremental Static Regeneration (ISR) — the evolution of SSG that adds per-page background revalidation, getting near-static performance with much fresher content.*
