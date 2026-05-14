# Partial Hydration & Island Architecture

## Quick Reference

| Concept | Mechanism | Implication |
|---|---|---|
| Partial hydration | Only interactive components receive JavaScript; static content stays as HTML | JS bundle size scales with interactivity, not page size |
| Island | A self-contained interactive component embedded in static HTML | Each island is independently hydrated; surrounding HTML is inert |
| Zero-JS by default | Static components ship no JavaScript to the browser | A page that's 90% static sends 90% less JS than a full SPA |
| Hydration directives | `client:load`, `client:idle`, `client:visible` (Astro) control when each island hydrates | Fine-grained control over hydration timing and priority |

## What Is This?

In standard React SSR + hydration, every component ships JavaScript to the browser — even components that are purely presentational and never change. A static header, a text paragraph, a decorative image — all of them get bundled into the JavaScript payload and hydrated, even though they don't need to be.

Partial hydration (also called the Island Architecture) solves this by inverting the default: **components are static HTML by default**. Only components explicitly marked as interactive receive a JavaScript bundle and get hydrated. The "islands" of interactivity float in a "sea of static HTML."

```
Traditional SSR + hydration:
  Page = [Header] + [Nav] + [Article] + [Comments] + [Footer]
  All five components → JS bundle → all five hydrated

Island architecture:
  Page = <Header/> + <Nav/> + <Article/> + <Comments/> + <Footer/>
  Only <Comments/> is interactive → only Comments → JS bundle → only Comments hydrated
  Everything else stays as inert server-rendered HTML
```

The result: a blog post page with only a comment form might ship 10KB of JavaScript instead of 200KB.

> **Check yourself:** On a page using island architecture, a user clicks a "Like" button that's defined as a static component (no JS). What happens?

## Why Does It Exist?

The hydration problem: React's hydration model requires hydrating the entire component tree, even for content that will never change. This costs JavaScript bundle size (it all downloads) and execution time (it all parses and runs) even for components that are functionally equivalent to plain HTML.

The insight behind islands: most content on most websites is static. A documentation site is 95% text with a 5% interactive search widget. A blog post is 99% text with a comment form. Shipping a full React app with hydration for 99% static content is wasteful — the cost is paid in JS download and parse time, hitting users on every page load.

[Astro](https://astro.build) popularized this with a clear model: write your page as a mix of framework components and plain HTML. Only components with explicit `client:*` directives ship any JS to the browser. The rest render as HTML at build time and are done.

[Fresh](https://fresh.deno.dev) (Deno) and [Marko](https://markojs.com) have similar models.

## How It Works

### Astro's Directive Model

Astro is the clearest reference implementation. Every component is rendered at build or request time by default. To turn a component into a client-side island, you add a directive:

```astro
---
// A .astro file — server renders everything here
import Header from './Header.astro';          // static, no JS shipped
import SearchWidget from './SearchWidget.jsx'; // React component
import CommentForm from './CommentForm.jsx';   // React component
---

<Header />  <!-- static HTML, no JS -->

<!-- This is an island — hydrates immediately on page load -->
<SearchWidget client:load />

<!-- This island waits until the main thread is idle -->
<CommentForm client:idle />

<!-- This island hydrates only when scrolled into viewport -->
<RecommendationWidget client:visible />
```

Each `client:*` directive produces a separate JS bundle containing only that component's code and its React dependency. Components marked without directives contribute zero bytes to the client bundle.

### Hydration Directives

Astro's directives give you fine-grained control:

| Directive | When it hydrates | Use case |
|---|---|---|
| `client:load` | Immediately on page load | Critical interactive elements (search, nav) |
| `client:idle` | When `requestIdleCallback` fires | Non-critical UI that shouldn't compete with initial load |
| `client:visible` | When the component enters the viewport (IntersectionObserver) | Below-fold components |
| `client:media="(max-width: 768px)"` | When a media query matches | Mobile-only interactive elements |
| `client:only="react"` | Never server-renders; only in browser | Components that can't render without browser APIs |

### Multi-Framework Islands

Because each island is independently bundled and hydrated, different islands can use different frameworks. A page could have a React search widget, a Svelte animation component, and a Vue data table — each hydrated independently with their own framework runtime. This is impossible in a traditional React SSR setup because hydration requires the same framework that rendered the HTML.

### Framework-Level Partial Hydration

Outside of Astro, the concept exists in framework-specific forms:

- **React Server Components** (Next.js App Router): Server Components render to HTML only, never ship JS. Client Components (`'use client'`) ship JS and hydrate. The Server/Client Component boundary is effectively the island boundary.
- **Marko**: Automatically analyzes which components need interactivity and only ships JS for those — no explicit annotation required.
- **Qwik**: Takes a different approach (resumability) rather than islands, but achieves similar JS reduction goals.

> **Check yourself:** A React Server Component renders an `<img>` tag with calculated dimensions. A React Client Component renders a `<button>` with an `onClick`. Which one ships JavaScript to the browser? Why?

## The Architecture Trade-off

Islands are not free. The trade-off:

**What you gain:**
- Dramatically smaller initial JS payload
- Faster TTI because less JS needs to parse and execute
- Static HTML for the majority of the page loads instantly with no JS

**What you give up:**
- State cannot be shared between islands using React context or the component tree — each island is isolated. Cross-island state sharing requires global stores (nanostores in Astro, Signals) or URL state.
- Framework component composition works only within an island, not across islands.
- `useEffect`, context, and component lifecycle don't span the island boundary.

This is a fundamentally different mental model from a React SPA where state flows through a unified component tree. Islands trade global state composability for JS efficiency.

## Comparison: Full Hydration vs. Partial Hydration

| | Full Hydration (traditional React SSR) | Partial Hydration / Islands |
|---|---|---|
| JS shipped | All components | Only interactive components |
| State sharing | Component tree, context | Cross-island requires global store |
| Framework assumption | One framework for the whole page | Can mix frameworks per island |
| Complexity | Simpler mental model | Need to think about boundaries |
| Ideal for | Apps with lots of interactivity | Content-heavy pages with selective interactivity |

## Gotchas

**Cross-island state is a significant constraint.** If island A and island B need to share state (e.g., a cart count that both a header and a checkout button need to show), they can't share React state or context across the island boundary. Solutions: a framework-agnostic store (nanostores, Zustand at the window level), URL state, or consolidating both into a single larger island. Teams frequently underestimate this constraint until they build something that needs shared state.

**Nested hydration still happens within islands.** Everything inside a `client:load` component is fully hydrated, including children you think of as static. "Partial" means at the page level — within an island, normal full hydration applies.

**SSR for islands has framework-specific constraints.** An Astro page with React islands server-renders the React components (they appear as HTML in the initial response) and then ships the JS to hydrate them. But if a React component inside an island uses a browser-only API during SSR render time, it will throw on the server. Same isomorphic code constraints as regular React SSR apply within each island.

**Multiple framework runtimes have a size cost.** Using both React and Vue islands on the same page means shipping both the React and Vue runtimes. If the isolated component approach is saving 150KB of application code but adding 60KB of an additional framework runtime, the savings diminish. Islands work best when you're using one primary framework for all islands or when islands are small and self-contained.

## Interview Questions

**Q (High): What problem does island architecture solve that standard SSR + hydration doesn't, and what's the core mechanism?**

Answer: Standard SSR hydrates the entire component tree regardless of whether components are interactive — a static header, a text paragraph, and a decorative image all receive JavaScript and go through the hydration process. On content-heavy pages where 90% of the UI is static, this means shipping and executing JavaScript for content that never changes. Island architecture's core insight: components are static HTML by default and receive JavaScript only when explicitly marked as interactive. The mechanism: only "island" components get bundled into the JS payload. Static components render at build/request time and arrive as inert HTML — no framework runtime, no event listener attachment, no hydration cost. This scales JS bundle size to the amount of actual interactivity on the page, not to the total page size.

The trap: Confusing island architecture with lazy loading. Lazy loading defers when JS loads; islands determine whether JS loads at all.

---

**Q (High): How do Astro's `client:*` directives differ from each other, and how would you decide which to use?**

Answer: Each directive controls when an island's JavaScript is downloaded and executed. `client:load` hydrates immediately on page load — use for above-the-fold interactive elements users interact with right away (search bars, dropdowns, sticky nav). `client:idle` hydrates when `requestIdleCallback` fires — use for below-fold or non-critical interactive elements that shouldn't compete with initial page rendering. `client:visible` uses IntersectionObserver and hydrates only when the component enters the viewport — use for comments sections, recommendation widgets, anything the user might never scroll to. `client:media` only hydrates when a media query matches — use for mobile-only interactive menus. The goal is to defer hydration cost to when it's actually needed, reducing TTI by prioritizing what the user cares about first.

The trap: Using `client:load` for everything defeats the purpose of fine-grained hydration. The interview question is often testing whether you understand the progressive loading model, not just that the directives exist.

---

**Q (Medium): How does state management work across islands, and what are the limitations?**

Answer: Each island is an independently hydrated component tree. Standard React state, context, and the component composition model work within an island but don't cross island boundaries — because the islands aren't part of the same React tree. Cross-island state requires an external mechanism: a framework-agnostic global store (nanostores is Astro's recommendation, nano-stores are tiny and work without any framework), storing state in the URL (query parameters, hash), broadcasting via BroadcastChannel or custom events, or server-side session state for auth-related shared data. The practical implication: before adopting island architecture, map out the state dependencies in your application. If most of your page requires complex shared state between components, the island model's constraint is too costly and full-hydration SSR is more appropriate.

The trap: Assuming React context works across islands. It doesn't — context only exists within a single React tree, and each island is its own tree.

---

**Q (Medium): What is React Server Components (RSC) and how does it relate to the island architecture concept?**

Answer: React Server Components implement a form of partial hydration within the React ecosystem. A Server Component is a React component that runs only on the server — it renders to HTML (or a special wire format) and ships zero JavaScript to the client. A Client Component (`'use client'` directive) runs on both server and client and ships its JavaScript for hydration. The Server/Client boundary is functionally the island boundary: Client Components are the islands, Server Components are the static sea. The key difference from Astro's island model: RSC maintains the unified React component tree and allows state and context to flow through the tree, with the constraint that a Server Component cannot use hooks or browser APIs. RSC's advantage over traditional islands: you can nest Server Components inside Client Components (for static sub-trees within islands), and you get React's full composition model within the Client Component islands.

The trap: Not connecting RSC to island architecture. They're the same conceptual model implemented within React, and interviewers testing both topics will notice if you don't draw the connection.

---

**Q (Low): Can island architecture be used with multiple JavaScript frameworks on the same page, and what's the cost?**

Answer: Yes — because each island is independently bundled and hydrated with no shared component tree, each island can use a different framework. Astro explicitly supports React, Vue, Svelte, Solid, and Preact islands on the same page. The cost is shipping multiple framework runtimes to the browser: if you have both React and Svelte islands, the browser downloads both the React runtime (~40KB gz) and the Svelte runtime (much smaller, ~2KB gz). For most combinations this is acceptable and the per-component JS savings outweigh the runtime overhead. The pragmatic approach: choose one primary framework for interactive islands and use a second framework only when there's a compelling reason (e.g., reusing an existing component from another team's library), and verify the total payload is still smaller than the full-hydration equivalent.

The trap: Either claiming it's free (ignoring the runtime duplication cost) or claiming it's always prohibitive (ignoring the per-component savings that usually outweigh it).

---

## Self-Assessment

Before moving on, check off each item you can do WITHOUT looking at the file.

- [ ] Can explain the core idea: components are static by default, JS ships only for interactive islands
- [ ] Can name at least three Astro `client:*` directives and what each controls
- [ ] Can explain why cross-island state sharing requires a global store (not React context)
- [ ] Can connect React Server Components to the island architecture concept
- [ ] Can articulate when full-hydration SSR is more appropriate than island architecture
- [ ] Can name the cost of using multiple frameworks on the same page with islands

---
*Next: Progressive Rehydration — a middle ground where hydration happens in priority order rather than all at once, improving TTI without the architectural constraints of full islands.*
