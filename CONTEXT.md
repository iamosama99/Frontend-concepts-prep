# Project Context — Frontend Engineering Interview Prep

> Hand this file to any new conversation to continue exactly where we left off.

---

## What This Project Is

A personal frontend engineering interview prep repository for a **senior frontend engineer**. The goal is to cover 147 topics across 23 phases, one topic at a time, at the learner's pace. Quality over speed — there is no rush.

Topics span the full breadth of senior frontend knowledge: rendering strategies, browser internals, networking, state management, performance, security, TypeScript, testing, accessibility, and more. Topics are generated as individual markdown files. Each file is a standalone, self-contained reference on that topic — written to build genuine understanding, not to memorize facts.

---

## Who You're Teaching

- Senior frontend engineer level — use correct terminology freely, don't oversimplify
- Prefers intuitive understanding over rote knowledge: reasoning and "why" matter as much as "what"
- Wants to connect the dots between concepts — each topic should feel like a logical step from the last
- No caps on length, no caps on number of interview questions — go as deep as the topic warrants
- Code examples: **plain JavaScript for Phases 1–13**, **TypeScript from Phase 14 onwards**

---

## How to Continue

When the user says **"next"**, generate the next topic in sequence (see progress tracker below).
When the user names a **specific topic**, jump directly to it.

Write the topic as a markdown file and save it to the correct path (listed per topic below).
After writing, commit it with `git add <file> && git commit`.

---

## Content Format — Follow This For Every Topic

Each topic file must follow this structure. Do not skip sections that apply. There is no length limit.

```
# [Topic Name]

## Quick Reference

A small table (2–5 rows) or tight bullet list placed immediately after the title.
Maps the core concept → mechanism → practical implication at a glance.
A reader returning after weeks should be able to re-anchor in under 10 seconds
without re-reading the whole file.

Example shape:
| You write / see | What it actually is | Why it matters |
|---|---|---|
| fetch() | HTTP request API | Replaces XHR, promise-based |

Keep it to 2–5 rows. One idea per row. No prose.

## What Is This?
Plain explanation of what the concept IS. Before any mechanics. Make it feel grounded.
One or two short paragraphs. A code snippet here is fine if it helps orient.

> **Check yourself:** [A tight question about the just-covered concept — forces the
> reader to actively recall before moving on. Think before scrolling.]

## Why Does It Exist?
The problem it solves. The history if relevant. Why the API is designed this way.
If the reason is self-evident, keep this short. If it's non-obvious or historically
interesting, go deep. Build the reasoning — nothing should feel arbitrary.

## How It Works
The mechanism. Internals where relevant. Mental models. Step-by-step if clarifying.
Use code to show the mechanism, not just the API.

> **Check yourself:** [A question that tests mechanical understanding — not just
> naming, but reasoning about cause and effect.]

## [Other sections as needed — use judgment]
E.g.: "The Old vs New Approach", "Side-by-Side Comparison", "In Production",
"Common Patterns", "Edge Cases". Name them for what they actually cover.
Only create a section if it genuinely adds value. No filler sections.
Each major section may have its own Check yourself prompt if the section introduces
a new idea worth testing. Target 2–3 prompts per file total; don't pad.

## Gotchas
The real ones. The ones that trip people up in interviews or production.
No padding. If there are two, write two. If there are six, write six.
Every gotcha that is interview-worthy must also appear as a Q&A entry below —
don't list it here and then omit it from the interview section.

## Interview Questions

Each question has an importance label: `High` (core concept, asked in almost every
senior interview on this topic), `Medium` (common but less universal), or `Low`
(edge case, rarely tested in typical senior interviews).

ORDER QUESTIONS from most to least important: High → Medium → Low.
Study time is finite — the High questions are asked in nearly every senior
interview, so they should be encountered and drilled first.

**Q (Medium): [Question text]**

Answer: [What a senior engineer would say. Be complete — this is the reference answer.]

The trap: [What the interviewer is watching for. What weaker candidates say or miss.]

[Repeat for every question the topic genuinely warrants. No artificial cap.]

---

## Self-Assessment

Before moving on, check off each item you can do WITHOUT looking at the file.
Leave unchecked anything you'd need to read to answer — that's what to revisit.

- [ ] [Concrete capability: "Can explain X in one sentence without notes"]
- [ ] [Concrete capability: "Can write a minimal code example from memory"]
- [ ] [Concrete capability: "Can name the gotcha and explain why it happens"]
- [ ] [Add 2–4 more items specific to the topic — aim for 4–6 total]

---
*Next: [Next topic name] — [One sentence on why it follows naturally from this one.]*
```

---

## Style Rules

- Start every topic with **what it is**, then **why it exists**, then **how it works**
- Use plain language but exact terminology — never dumb down, never use vague words when a precise term exists
- Build reasoning from first principles — every constraint, API choice, and gotcha should feel logical
- Write the "how it works" sections as a senior engineer explaining over coffee, not as documentation
- Connect concepts to each other — reference prior topics when relevant, set up the next topic at the end
- Never pad. If a section doesn't apply, don't invent content to fill it
- Interview questions: write as many as the topic genuinely warrants. Include the answer AND the trap

---

## Repository Structure

```
/Users/osama/Developer/Projects/Frontend concepts prep/
├── CONTEXT.md                          ← this file
├── README.md                           ← full topic index with checkboxes
├── .gitignore
├── phase-01-rendering-strategies/      # JS examples
├── phase-02-browser-internals/         # JS examples
├── phase-03-networking/                # JS examples
├── phase-04-state-management/          # JS examples
├── phase-05-component-architecture/    # JS examples
├── phase-06-performance/               # JS examples
├── phase-07-concurrency/               # JS examples
├── phase-08-browser-storage/           # JS examples
├── phase-09-routing/                   # JS examples
├── phase-10-module-systems/            # JS examples
├── phase-11-web-components/            # JS examples
├── phase-12-observer-apis/             # JS examples
├── phase-13-security/                  # JS examples
├── phase-14-typescript/                # TS examples ← switch to TS here
├── phase-15-memory-management/         # TS examples
├── phase-16-webassembly/               # TS examples
├── phase-17-resilience-observability/  # TS examples
├── phase-18-cicd-deployment/           # TS examples
├── phase-19-seo/                       # TS examples
├── phase-20-accessibility/             # TS examples
├── phase-21-i18n/                      # TS examples
├── phase-22-testing/                   # TS examples
└── phase-23-dx-tooling/                # TS examples
```

---

## Progress Tracker

Legend: ✅ Done | ⬜ Not started | 👉 **Next up**

### Phase 1 — Rendering Strategies & Patterns (8 topics)

| # | Topic | File | Status |
|---|-------|------|--------|
| 1 | Client-Side Rendering (CSR) | `phase-01-rendering-strategies/01-csr-client-side-rendering.md` | ✅ |
| 2 | Server-Side Rendering (SSR) | `phase-01-rendering-strategies/02-ssr-server-side-rendering.md` | ✅ |
| 3 | Static Site Generation (SSG) | `phase-01-rendering-strategies/03-ssg-static-site-generation.md` | ✅ |
| 4 | Incremental Static Regeneration (ISR) | `phase-01-rendering-strategies/04-isr-incremental-static-regeneration.md` | ✅ |
| 5 | Streaming SSR | `phase-01-rendering-strategies/05-streaming-ssr.md` | ✅ |
| 6 | Partial Hydration & Island Architecture | `phase-01-rendering-strategies/06-partial-hydration-island-architecture.md` | ✅ |
| 7 | Progressive Rehydration | `phase-01-rendering-strategies/07-progressive-rehydration.md` | ✅ |
| 8 | Resumability (Qwik model) | `phase-01-rendering-strategies/08-resumability-qwik-model.md` | ✅ |

### Phase 2 — Browser Internals & Rendering Pipeline (5 topics)

| # | Topic | File | Status |
|---|-------|------|--------|
| 1 | Event Loop, Call Stack, Task Queue, Microtask Queue | `phase-02-browser-internals/01-event-loop-call-stack-task-microtask-queue.md` | ✅ |
| 2 | Critical Rendering Path | `phase-02-browser-internals/02-critical-rendering-path.md` | ✅ |
| 3 | Reflow vs. Repaint vs. Composite-only changes | `phase-02-browser-internals/03-reflow-repaint-composite.md` | ✅ |
| 4 | GPU Layers & Compositing | `phase-02-browser-internals/04-gpu-layers-compositing.md` | ✅ |
| 5 | Main Thread vs. Compositor Thread | `phase-02-browser-internals/05-main-thread-vs-compositor-thread.md` | ✅ |

### Phase 3 — Networking & Data Communication (7 topics)

| # | Topic | File | Status |
|---|-------|------|--------|
| 1 | API Paradigms: REST, GraphQL, gRPC-web, tRPC | `phase-03-networking/01-api-paradigms-rest-graphql-grpc-trpc.md` | ✅ |
| 2 | Real-time Communication: WebSockets, SSE, Long Polling | `phase-03-networking/02-real-time-websockets-sse-long-polling.md` | ✅ |
| 3 | BFF Pattern (Backend-for-Frontend) | `phase-03-networking/03-bff-pattern.md` | ✅ |
| 4 | Data Fetching Strategies: SWR, React Query, Optimistic UI | `phase-03-networking/04-data-fetching-swr-react-query-optimistic-ui.md` | ✅ |
| 5 | Caching Layers: HTTP Cache, Service Worker Cache, CDN | `phase-03-networking/05-caching-layers-http-service-worker-cdn.md` | ✅ |
| 6 | Payload Optimization: Compression, Protocol Buffers, Binary | `phase-03-networking/06-payload-optimization-compression-protobuf-binary.md` | ✅ |
| 7 | HTTP/2 Multiplexing & HTTP/3 (QUIC) | `phase-03-networking/07-http2-multiplexing-http3-quic.md` | ✅ |

### Phase 4 — State Management & Data Flow (7 topics)

| # | Topic | File | Status |
|---|-------|------|--------|
| 1 | Server State vs. Client State vs. URL State vs. Form State | `phase-04-state-management/01-server-client-url-form-state.md` | ✅ |
| 2 | Unidirectional vs. Bidirectional Data Flow | `phase-04-state-management/02-unidirectional-vs-bidirectional-data-flow.md` | ✅ |
| 3 | Reactivity Models: Signals, Observables (RxJS), Proxies | `phase-04-state-management/03-reactivity-models-signals-observables-proxies.md` | ✅ |
| 4 | Global Store vs. Prop Drilling vs. Context / DI | `phase-04-state-management/04-global-store-prop-drilling-context-di.md` | ✅ |
| 5 | Finite State Machines (XState) | `phase-04-state-management/05-finite-state-machines-xstate.md` | ✅ |
| 6 | Immutable Data Structures | `phase-04-state-management/06-immutable-data-structures.md` | ✅ |
| 7 | Derived State & Memoization Strategies | `phase-04-state-management/07-derived-state-memoization.md` | ✅ |

### Phase 5 — Component Architecture & Scalability (6 topics)

| # | Topic | File | Status |
|---|-------|------|--------|
| 1 | Micro-Frontends: Module Federation, Iframes, Web Components | `phase-05-component-architecture/01-micro-frontends-module-federation.md` | ✅ |
| 2 | Monorepo Management vs. Polyrepo | `phase-05-component-architecture/02-monorepo-vs-polyrepo.md` | ✅ |
| 3 | Design System Architecture: Headless UI, Design Tokens, Theming | `phase-05-component-architecture/03-design-system-headless-ui-tokens-theming.md` | ✅ |
| 4 | Atomic Design Principles | `phase-05-component-architecture/04-atomic-design-principles.md` | ✅ |
| 5 | Component Composition Patterns: HOCs, Render Props, Hooks, Compound | `phase-05-component-architecture/05-component-composition-patterns.md` | ✅ |
| 6 | Slot-based Composition vs. Children Prop Patterns | `phase-05-component-architecture/06-slot-based-vs-children-prop.md` | ✅ |

### Phase 6 — Performance Optimization (8 topics)

| # | Topic | File | Status |
|---|-------|------|--------|
| 1 | Core Web Vitals: LCP, INP, CLS | `phase-06-performance/01-core-web-vitals-lcp-inp-cls.md` | ✅ |
| 2 | Resource Hinting: Preload, Prefetch, Preconnect, Prerender | `phase-06-performance/02-resource-hinting-preload-prefetch-preconnect.md` | ✅ |
| 3 | Code Splitting & Dynamic Imports | `phase-06-performance/03-code-splitting-dynamic-imports.md` | ✅ |
| 4 | List Virtualization & Windowing | `phase-06-performance/04-list-virtualization-windowing.md` | ✅ |
| 5 | Tree Shaking & Dead Code Elimination | `phase-06-performance/05-tree-shaking-dead-code-elimination.md` | ✅ |
| 6 | Asset Optimization: WebP/AVIF, Font Subsetting, SVGO | `phase-06-performance/06-asset-optimization-webp-avif-fonts-svgo.md` | ✅ |
| 7 | CSS Performance: Critical CSS, CSS-in-JS, Utility CSS | `phase-06-performance/07-css-performance-critical-css-in-js-utility.md` | ✅ |
| 8 | Animation Performance: FLIP, CSS vs. JS, requestAnimationFrame | `phase-06-performance/08-animation-performance-flip-css-vs-js-raf.md` | ✅ |

### Phase 7 — Concurrency, Parallelism & Scheduling (7 topics)

| # | Topic | File | Status |
|---|-------|------|--------|
| 1 | Web Workers & Dedicated Workers | `phase-07-concurrency/01-web-workers-dedicated-workers.md` | ✅ |
| 2 | Service Workers: Lifecycle, Caching, Background Sync | `phase-07-concurrency/02-service-workers-lifecycle-caching-background-sync.md` | ✅ |
| 3 | Shared Workers & Cross-tab Communication | `phase-07-concurrency/03-shared-workers-cross-tab-communication.md` | ✅ |
| 4 | Worklets: Audio, Paint, Animation | `phase-07-concurrency/04-worklets-audio-paint-animation.md` | ✅ |
| 5 | SharedArrayBuffer & Atomics | `phase-07-concurrency/05-sharedarraybuffer-atomics.md` | ✅ |
| 6 | Concurrency Scheduling: React Concurrent Mode, scheduler.postTask | `phase-07-concurrency/06-concurrency-scheduling-react-concurrent-scheduler.md` | ✅ |
| 7 | OffscreenCanvas | `phase-07-concurrency/07-offscreen-canvas.md` | ✅ |

### Phase 8 — Browser Storage & Offline (7 topics)

| # | Topic | File | Status |
|---|-------|------|--------|
| 1 | Cookie Strategy: HttpOnly, SameSite, Secure, Partitioned | `phase-08-browser-storage/01-cookie-strategy-httponly-samesite-secure.md` | ✅ |
| 2 | LocalStorage vs. SessionStorage | `phase-08-browser-storage/02-localstorage-vs-sessionstorage.md` | ✅ |
| 3 | IndexedDB: Schema Design & Transactions | `phase-08-browser-storage/03-indexeddb-schema-transactions.md` | ✅ |
| 4 | Cache API (Service Worker Layer) | `phase-08-browser-storage/04-cache-api-service-worker-layer.md` | ✅ |
| 5 | Storage Quotas & Eviction Policies | `phase-08-browser-storage/05-storage-quotas-eviction-policies.md` | ✅ |
| 6 | PWA: App Shell, Background Sync, Web Push | `phase-08-browser-storage/06-pwa-app-shell-background-sync-web-push.md` | ✅ |
| 7 | Offline-first Architecture Patterns | `phase-08-browser-storage/07-offline-first-architecture.md` | ✅ |

### Phase 9 — Client-Side Routing & URL Design (7 topics)

| # | Topic | File | Status |
|---|-------|------|--------|
| 1 | History API vs. Hash Routing | `phase-09-routing/01-history-api-vs-hash-routing.md` | ✅ |
| 2 | File-system Based vs. Config-based Routing | `phase-09-routing/02-filesystem-vs-config-based-routing.md` | ✅ |
| 3 | Nested Routing & Layout Inheritance | `phase-09-routing/03-nested-routing-layout-inheritance.md` | ✅ |
| 4 | Route-level Code Splitting | `phase-09-routing/04-route-level-code-splitting.md` | ✅ |
| 5 | Navigation Guards & Middleware | `phase-09-routing/05-navigation-guards-middleware.md` | ✅ |
| 6 | URL as State: Deep Linking & Shareability | `phase-09-routing/06-url-as-state-deep-linking.md` | ✅ |
| 7 | Scroll Restoration | `phase-09-routing/07-scroll-restoration.md` | ✅ |

### Phase 10 — Module Systems & Build Infrastructure (6 topics)

| # | Topic | File | Status |
|---|-------|------|--------|
| 1 | ESM vs. CJS vs. UMD — Interop Challenges | `phase-10-module-systems/01-esm-vs-cjs-vs-umd.md` | 👉 **Next** |
| 2 | Bundlers: Webpack, Vite, Esbuild, Turbopack, Rollup | `phase-10-module-systems/02-bundlers-webpack-vite-esbuild-turbopack-rollup.md` | ⬜ |
| 3 | Tree Shaking, Scope Hoisting, Chunk Splitting | `phase-10-module-systems/03-tree-shaking-scope-hoisting-chunk-splitting.md` | ⬜ |
| 4 | Source Maps Strategy (dev vs. prod) | `phase-10-module-systems/04-source-maps-strategy.md` | ⬜ |
| 5 | Polyfill Strategy: Differential Serving, browserslist, core-js | `phase-10-module-systems/05-polyfill-strategy-differential-serving.md` | ⬜ |
| 6 | Environment Config Management & Package Versioning at Scale | `phase-10-module-systems/06-environment-config-package-versioning.md` | ⬜ |

### Phase 11 — Web Components & Native Browser Primitives (4 topics)

| # | Topic | File | Status |
|---|-------|------|--------|
| 1 | Custom Elements Lifecycle | `phase-11-web-components/01-custom-elements-lifecycle.md` | ⬜ |
| 2 | Shadow DOM: Encapsulation, Open vs. Closed Mode | `phase-11-web-components/02-shadow-dom.md` | ⬜ |
| 3 | HTML Templates & Slots | `phase-11-web-components/03-html-templates-slots.md` | ⬜ |
| 4 | Interoperability with Frameworks | `phase-11-web-components/04-framework-interoperability.md` | ⬜ |

### Phase 12 — Observer & Intersection APIs (5 topics)

| # | Topic | File | Status |
|---|-------|------|--------|
| 1 | IntersectionObserver | `phase-12-observer-apis/01-intersection-observer.md` | ⬜ |
| 2 | ResizeObserver | `phase-12-observer-apis/02-resize-observer.md` | ⬜ |
| 3 | MutationObserver | `phase-12-observer-apis/03-mutation-observer.md` | ⬜ |
| 4 | PerformanceObserver | `phase-12-observer-apis/04-performance-observer.md` | ⬜ |
| 5 | Observer APIs vs. Event Listener Patterns | `phase-12-observer-apis/05-observer-vs-event-listeners.md` | ⬜ |

### Phase 13 — Security (8 topics)

| # | Topic | File | Status |
|---|-------|------|--------|
| 1 | Cross-Site Scripting (XSS): DOM vs. Reflected vs. Stored | `phase-13-security/01-xss-dom-reflected-stored.md` | ⬜ |
| 2 | Cross-Site Request Forgery (CSRF) Protection Patterns | `phase-13-security/02-csrf-protection-patterns.md` | ⬜ |
| 3 | Content Security Policy (CSP): Nonces, Hashes, strict-dynamic | `phase-13-security/03-content-security-policy-csp.md` | ⬜ |
| 4 | CORS Configuration & Preflight | `phase-13-security/04-cors-preflight.md` | ⬜ |
| 5 | Auth & Authorization: JWT, OAuth2, OIDC, HttpOnly Cookies | `phase-13-security/05-authentication-jwt-oauth2-oidc-cookies.md` | ⬜ |
| 6 | Subresource Integrity (SRI) | `phase-13-security/06-subresource-integrity-sri.md` | ⬜ |
| 7 | Clickjacking: X-Frame-Options & frame-ancestors | `phase-13-security/07-clickjacking-x-frame-options.md` | ⬜ |
| 8 | Trusted Types API | `phase-13-security/08-trusted-types-api.md` | ⬜ |

### Phase 14 — TypeScript & Type System Architecture (5 topics) — TS examples start here

| # | Topic | File | Status |
|---|-------|------|--------|
| 1 | Structural Typing, Generics, Conditional & Mapped Types | `phase-14-typescript/01-structural-typing-generics-conditional-mapped.md` | ⬜ |
| 2 | Module Augmentation & Declaration Merging | `phase-14-typescript/02-module-augmentation-declaration-merging.md` | ⬜ |
| 3 | Strict Mode Trade-offs | `phase-14-typescript/03-strict-mode-tradeoffs.md` | ⬜ |
| 4 | Type-safe API Contracts: Zod, tRPC, OpenAPI Codegen | `phase-14-typescript/04-type-safe-api-contracts-zod-trpc-openapi.md` | ⬜ |
| 5 | Monorepo Shared Type Packages | `phase-14-typescript/05-monorepo-shared-type-packages.md` | ⬜ |

### Phase 15 — Memory Management (4 topics)

| # | Topic | File | Status |
|---|-------|------|--------|
| 1 | Memory Leaks: Detached DOM Nodes, Closures, Event Listeners | `phase-15-memory-management/01-memory-leaks-detached-dom-closures-listeners.md` | ⬜ |
| 2 | WeakMap, WeakRef & FinalizationRegistry | `phase-15-memory-management/02-weakmap-weakref-finalizationregistry.md` | ⬜ |
| 3 | Heap Profiling & DevTools Memory Snapshots | `phase-15-memory-management/03-heap-profiling-devtools-snapshots.md` | ⬜ |
| 4 | Object Pooling for High-frequency Allocations | `phase-15-memory-management/04-object-pooling.md` | ⬜ |

### Phase 16 — WebAssembly (WASM) (4 topics)

| # | Topic | File | Status |
|---|-------|------|--------|
| 1 | WASM Use Cases: Compute-heavy Workloads, Porting Native Code | `phase-16-webassembly/01-wasm-use-cases.md` | ⬜ |
| 2 | JS ↔ WASM Interop & Memory Model | `phase-16-webassembly/02-js-wasm-interop-memory-model.md` | ⬜ |
| 3 | WASM Threads & SIMD | `phase-16-webassembly/03-wasm-threads-simd.md` | ⬜ |
| 4 | WASM Tools: Emscripten, wasm-pack, AssemblyScript | `phase-16-webassembly/04-wasm-tools-emscripten-wasm-pack-assemblyscript.md` | ⬜ |

### Phase 17 — Resilience & Observability (5 topics)

| # | Topic | File | Status |
|---|-------|------|--------|
| 1 | Error Boundaries & Global Error Handlers | `phase-17-resilience-observability/01-error-boundaries-global-error-handlers.md` | ⬜ |
| 2 | Graceful Degradation & Progressive Enhancement | `phase-17-resilience-observability/02-graceful-degradation-progressive-enhancement.md` | ⬜ |
| 3 | Real User Monitoring (RUM) vs. Synthetic Monitoring | `phase-17-resilience-observability/03-rum-vs-synthetic-monitoring.md` | ⬜ |
| 4 | Logging, Tracing & Session Replay | `phase-17-resilience-observability/04-logging-tracing-session-replay.md` | ⬜ |
| 5 | Performance Budgets & Regression Alerts | `phase-17-resilience-observability/05-performance-budgets-regression-alerts.md` | ⬜ |

### Phase 18 — CI/CD, Deployment & Infrastructure (7 topics)

| # | Topic | File | Status |
|---|-------|------|--------|
| 1 | CDN Strategy & Cache Invalidation | `phase-18-cicd-deployment/01-cdn-strategy-cache-invalidation.md` | ⬜ |
| 2 | Edge Functions & Edge Rendering | `phase-18-cicd-deployment/02-edge-functions-edge-rendering.md` | ⬜ |
| 3 | Blue-Green, Canary & Rolling Deployments | `phase-18-cicd-deployment/03-blue-green-canary-rolling-deployments.md` | ⬜ |
| 4 | Feature Flags: Client-side vs. Server-side Evaluation | `phase-18-cicd-deployment/04-feature-flags-client-vs-server.md` | ⬜ |
| 5 | A/B Testing & Experimentation Infrastructure | `phase-18-cicd-deployment/05-ab-testing-experimentation.md` | ⬜ |
| 6 | Bundle Analysis in CI Pipeline | `phase-18-cicd-deployment/06-bundle-analysis-in-ci.md` | ⬜ |
| 7 | Lighthouse CI & Performance Budgeting in CI | `phase-18-cicd-deployment/07-lighthouse-ci-performance-budgeting.md` | ⬜ |

### Phase 19 — SEO & Discoverability (5 topics)

| # | Topic | File | Status |
|---|-------|------|--------|
| 1 | Crawlability: SSR/SSG vs. CSR for Bots | `phase-19-seo/01-crawlability-ssr-ssg-vs-csr.md` | ⬜ |
| 2 | Structured Data: JSON-LD & Schema.org | `phase-19-seo/02-structured-data-json-ld-schema.md` | ⬜ |
| 3 | Open Graph & Twitter Card Meta | `phase-19-seo/03-open-graph-twitter-card-meta.md` | ⬜ |
| 4 | Sitemap Generation & Canonical URLs | `phase-19-seo/04-sitemap-canonical-urls.md` | ⬜ |
| 5 | Dynamic Meta for SPAs | `phase-19-seo/05-dynamic-meta-for-spas.md` | ⬜ |

### Phase 20 — Accessibility (A11y) (5 topics)

| # | Topic | File | Status |
|---|-------|------|--------|
| 1 | WAI-ARIA: Roles, States & Properties | `phase-20-accessibility/01-wai-aria-roles-states-properties.md` | ⬜ |
| 2 | Keyboard Navigation & Focus Management | `phase-20-accessibility/02-keyboard-navigation-focus-management.md` | ⬜ |
| 3 | Screen Reader Behavior & Announcement Patterns | `phase-20-accessibility/03-screen-reader-behavior-announcements.md` | ⬜ |
| 4 | Color Contrast & Motion Sensitivity | `phase-20-accessibility/04-color-contrast-motion-sensitivity.md` | ⬜ |
| 5 | Accessibility Tree & How Browsers Expose It | `phase-20-accessibility/05-accessibility-tree-browser-exposure.md` | ⬜ |

### Phase 21 — Internationalization (i18n) & Localization (l10n) (5 topics)

| # | Topic | File | Status |
|---|-------|------|--------|
| 1 | Translation Loading Strategies: Static Bundle vs. On-demand | `phase-21-i18n/01-translation-loading-strategies.md` | ⬜ |
| 2 | RTL Layout Support | `phase-21-i18n/02-rtl-layout-support.md` | ⬜ |
| 3 | Pluralization Rules (CLDR) | `phase-21-i18n/03-pluralization-rules-cldr.md` | ⬜ |
| 4 | Date, Time, Number & Currency Formatting (Intl API) | `phase-21-i18n/04-date-time-number-currency-intl-api.md` | ⬜ |
| 5 | Locale Detection & Negotiation | `phase-21-i18n/05-locale-detection-negotiation.md` | ⬜ |

### Phase 22 — Testing Strategy (7 topics)

| # | Topic | File | Status |
|---|-------|------|--------|
| 1 | Unit Testing: Logic, Utils & Hooks | `phase-22-testing/01-unit-testing-logic-utils-hooks.md` | ⬜ |
| 2 | Component Testing: Testing Library Philosophy | `phase-22-testing/02-component-testing-testing-library.md` | ⬜ |
| 3 | Integration Testing | `phase-22-testing/03-integration-testing.md` | ⬜ |
| 4 | End-to-End Testing: Playwright & Cypress | `phase-22-testing/04-e2e-testing-playwright-cypress.md` | ⬜ |
| 5 | Visual Regression Testing: Percy & Chromatic | `phase-22-testing/05-visual-regression-testing-percy-chromatic.md` | ⬜ |
| 6 | Contract Testing: Pact & Schema Validation | `phase-22-testing/06-contract-testing-pact-schema-validation.md` | ⬜ |
| 7 | Performance Budgeting in CI | `phase-22-testing/07-performance-budgeting-in-ci.md` | ⬜ |

### Phase 23 — Developer Experience (DX) & Tooling (6 topics)

| # | Topic | File | Status |
|---|-------|------|--------|
| 1 | Linting & Formatting at Scale | `phase-23-dx-tooling/01-linting-formatting-at-scale.md` | ⬜ |
| 2 | Commit Hygiene: Husky, lint-staged, Conventional Commits | `phase-23-dx-tooling/02-commit-hygiene-husky-lint-staged.md` | ⬜ |
| 3 | Local Development Parity with Production | `phase-23-dx-tooling/03-local-dev-parity-docker-env-mocking.md` | ⬜ |
| 4 | Storybook-driven Development | `phase-23-dx-tooling/04-storybook-driven-development.md` | ⬜ |
| 5 | Code Generation: Plop, Hygen, OpenAPI Codegen | `phase-23-dx-tooling/05-code-generation-plop-hygen-openapi.md` | ⬜ |
| 6 | IDE Integration & TypeScript Server Performance | `phase-23-dx-tooling/06-ide-integration-typescript-server-performance.md` | ⬜ |

---

## How to Update This File

After each topic is completed:
1. Change its status from ⬜ to ✅ in the progress tracker above
2. Move the 👉 **Next** marker to the following topic
3. Commit: `git add CONTEXT.md && git commit -m "Update progress: [topic name] done"`
