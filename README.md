# Frontend Engineering Interview Prep

Senior frontend engineer interview preparation — 147 topics across 23 phases.

---

## How This Works

Every topic starts with **what it is and why it exists** before going anywhere near mechanics. The goal is that nothing feels arbitrary — every API choice, every constraint, every gotcha has a reason, and that reason is explained.

Language is plain but terminology is exact. Mental models are built from first principles so that when you're in an interview and the question takes an unexpected angle, you can *reason your way* to the right answer rather than recall a memorized one.

**Workflow:** One topic at a time, at your pace. Say "next" to continue in sequence or name any topic to jump to it.

**Code examples:** Plain JavaScript for Phases 1–13. TypeScript from Phase 14 onwards.

**Interview question labels:** Each question is tagged with its importance — `High` (core concept, expect it every time), `Medium` (common but less universal), `Low` (edge case, rarely tested). Questions are ordered High → Medium → Low within each file — most important first.

### Every topic file contains

| Section | Purpose |
|---|---|
| **Quick Reference** | 2–5 row table at the top — re-anchor in 10 seconds after weeks away |
| **Check yourself prompts** | Inline active recall questions after major sections — 2–3 per file |
| **Self-Assessment checklist** | End-of-file checklist — marks what you own vs. what you'd still need to look up |
| **Interview Q&A** | Full reference answers + the interviewer trap, ordered High → Medium → Low |

---

## Progress

| Phase | Topics | Area |
|-------|--------|------|
| [Phase 1](#phase-1--rendering-strategies--patterns) | 8 | Rendering strategies & patterns |
| [Phase 2](#phase-2--browser-internals--rendering-pipeline) | 5 | Browser internals & rendering pipeline |
| [Phase 3](#phase-3--networking--data-communication) | 7 | Networking & data communication |
| [Phase 4](#phase-4--state-management--data-flow) | 7 | State management & data flow |
| [Phase 5](#phase-5--component-architecture--scalability) | 6 | Component architecture & scalability |
| [Phase 6](#phase-6--performance-optimization) | 8 | Performance optimization |
| [Phase 7](#phase-7--concurrency-parallelism--scheduling) | 7 | Concurrency, parallelism & scheduling |
| [Phase 8](#phase-8--browser-storage--offline) | 7 | Browser storage & offline |
| [Phase 9](#phase-9--client-side-routing--url-design) | 7 | Client-side routing & URL design |
| [Phase 10](#phase-10--module-systems--build-infrastructure) | 6 | Module systems & build infrastructure |
| [Phase 11](#phase-11--web-components--native-browser-primitives) | 4 | Web components & native browser primitives |
| [Phase 12](#phase-12--observer--intersection-apis) | 5 | Observer & intersection APIs |
| [Phase 13](#phase-13--security) | 8 | Security |
| [Phase 14](#phase-14--typescript--type-system-architecture) | 5 | TypeScript & type system architecture |
| [Phase 15](#phase-15--memory-management) | 4 | Memory management |
| [Phase 16](#phase-16--webassembly-wasm) | 4 | WebAssembly (WASM) |
| [Phase 17](#phase-17--resilience--observability) | 5 | Resilience & observability |
| [Phase 18](#phase-18--cicd-deployment--infrastructure) | 7 | CI/CD, deployment & infrastructure |
| [Phase 19](#phase-19--seo--discoverability) | 5 | SEO & discoverability |
| [Phase 20](#phase-20--accessibility-a11y) | 5 | Accessibility (A11y) |
| [Phase 21](#phase-21--internationalization-i18n--localization-l10n) | 5 | Internationalization (i18n) & localization (l10n) |
| [Phase 22](#phase-22--testing-strategy) | 7 | Testing strategy |
| [Phase 23](#phase-23--developer-experience-dx--tooling) | 6 | Developer experience (DX) & tooling |
| **Total** | **147** | |

---

## Phase 1 — Rendering Strategies & Patterns

> The foundational split that drives architectural decisions. Every other topic builds on understanding when HTML is produced, where, and by whom.

- [ ] [Client-Side Rendering (CSR)](phase-01-rendering-strategies/01-csr-client-side-rendering.md) — the SPA model, JS bundle executes in browser, blank HTML shell
- [ ] [Server-Side Rendering (SSR)](phase-01-rendering-strategies/02-ssr-server-side-rendering.md) — full HTML on every request, TTFB vs. hydration cost
- [ ] [Static Site Generation (SSG)](phase-01-rendering-strategies/03-ssg-static-site-generation.md) — HTML at build time, CDN-served, staleness trade-offs
- [ ] [Incremental Static Regeneration (ISR)](phase-01-rendering-strategies/04-isr-incremental-static-regeneration.md) — stale-while-revalidate for pages, background regeneration
- [ ] [Streaming SSR](phase-01-rendering-strategies/05-streaming-ssr.md) — chunked HTML over HTTP, Suspense boundaries as flush points
- [ ] [Partial Hydration & Island Architecture](phase-01-rendering-strategies/06-partial-hydration-island-architecture.md) — Astro/Fresh model, only interactive components ship JS
- [ ] [Progressive Rehydration](phase-01-rendering-strategies/07-progressive-rehydration.md) — prioritized hydration by viewport / interaction, reduces TTI
- [ ] [Resumability (Qwik model)](phase-01-rendering-strategies/08-resumability-qwik-model.md) — serialize execution state, no hydration replay

---

## Phase 2 — Browser Internals & Rendering Pipeline

> What the browser actually does with your HTML, CSS, and JS. Understanding this is what separates performance intuition from guesswork.

- [ ] [Event Loop, Call Stack, Task Queue, Microtask Queue](phase-02-browser-internals/01-event-loop-call-stack-task-microtask-queue.md) — how JS single-threading works, why microtasks flush before next task
- [ ] [Critical Rendering Path](phase-02-browser-internals/02-critical-rendering-path.md) — Parse → Style → Layout → Paint → Composite pipeline
- [ ] [Reflow vs. Repaint vs. Composite-only Changes](phase-02-browser-internals/03-reflow-repaint-composite.md) — cost hierarchy, which CSS properties trigger which
- [ ] [GPU Layers & Compositing](phase-02-browser-internals/04-gpu-layers-compositing.md) — will-change, transform hacks, when to promote to layer
- [ ] [Main Thread vs. Compositor Thread](phase-02-browser-internals/05-main-thread-vs-compositor-thread.md) — what runs off main thread, why it matters for 60fps

---

## Phase 3 — Networking & Data Communication

> The protocol and pattern layer. How data moves between browser and server, and the trade-offs of each approach.

- [ ] [API Paradigms: REST, GraphQL, gRPC-web, tRPC](phase-03-networking/01-api-paradigms-rest-graphql-grpc-trpc.md) — when to use each, over-fetching vs. type safety
- [ ] [Real-time Communication: WebSockets, SSE, Long Polling](phase-03-networking/02-real-time-websockets-sse-long-polling.md) — bidirectional vs. unidirectional, connection overhead
- [ ] [BFF Pattern (Backend-for-Frontend)](phase-03-networking/03-bff-pattern.md) — why one API per client surface, aggregation & auth at the BFF
- [ ] [Data Fetching Strategies: SWR, React Query, Optimistic UI](phase-03-networking/04-data-fetching-swr-react-query-optimistic-ui.md) — stale-while-revalidate, cache + refetch patterns
- [ ] [Caching Layers: HTTP Cache, Service Worker Cache, CDN](phase-03-networking/05-caching-layers-http-service-worker-cdn.md) — Cache-Control, ETag, layered caching model
- [ ] [Payload Optimization: Compression, Protocol Buffers, Binary](phase-03-networking/06-payload-optimization-compression-protobuf-binary.md) — Brotli vs. Gzip, protobuf vs. JSON size
- [ ] [HTTP/2 Multiplexing & HTTP/3 (QUIC)](phase-03-networking/07-http2-multiplexing-http3-quic.md) — head-of-line blocking, 0-RTT, impact on asset loading

---

## Phase 4 — State Management & Data Flow

> How state is categorized, where it lives, and how changes propagate through an application.

- [ ] [Server State vs. Client State vs. URL State vs. Form State](phase-04-state-management/01-server-client-url-form-state.md) — the four state categories, why mixing them causes bugs
- [ ] [Unidirectional vs. Bidirectional Data Flow](phase-04-state-management/02-unidirectional-vs-bidirectional-data-flow.md) — React/Redux vs. Angular two-way binding, predictability trade-off
- [ ] [Reactivity Models: Signals, Observables (RxJS), Proxies](phase-04-state-management/03-reactivity-models-signals-observables-proxies.md) — fine-grained reactivity vs. virtual DOM diffing
- [ ] [Global Store vs. Prop Drilling vs. Context / DI](phase-04-state-management/04-global-store-prop-drilling-context-di.md) — when each scales, performance implications of context
- [ ] [Finite State Machines (XState)](phase-04-state-management/05-finite-state-machines-xstate.md) — explicit states beat boolean flags, impossible state prevention
- [ ] [Immutable Data Structures](phase-04-state-management/06-immutable-data-structures.md) — why immutability enables change detection, structural sharing
- [ ] [Derived State & Memoization Strategies](phase-04-state-management/07-derived-state-memoization.md) — compute vs. store, memoization boundaries

---

## Phase 5 — Component Architecture & Scalability

> How to structure large frontend codebases so they don't collapse under their own weight.

- [ ] [Micro-Frontends: Module Federation, Iframes, Web Components](phase-05-component-architecture/01-micro-frontends-module-federation.md) — independent deploy, shared dependencies, the integration cost
- [ ] [Monorepo Management vs. Polyrepo](phase-05-component-architecture/02-monorepo-vs-polyrepo.md) — Nx vs. Turborepo, dependency graph, atomic commits
- [ ] [Design System Architecture: Headless UI, Design Tokens, Theming](phase-05-component-architecture/03-design-system-headless-ui-tokens-theming.md) — separation of behavior and style, token taxonomy
- [ ] [Atomic Design Principles](phase-05-component-architecture/04-atomic-design-principles.md) — atoms → molecules → organisms, where it helps and where it doesn't
- [ ] [Component Composition Patterns: HOCs, Render Props, Hooks, Compound](phase-05-component-architecture/05-component-composition-patterns.md) — evolution of patterns, when each still applies
- [ ] [Slot-based Composition vs. Children Prop Patterns](phase-05-component-architecture/06-slot-based-vs-children-prop.md) — named slots, asChild pattern, Radix-style composition

---

## Phase 6 — Performance Optimization

> The full performance toolkit — from metrics through techniques. Knowing *which* tool fixes *which* problem.

- [ ] [Core Web Vitals: LCP, INP, CLS](phase-06-performance/01-core-web-vitals-lcp-inp-cls.md) — what each measures, what moves each metric
- [ ] [Resource Hinting: Preload, Prefetch, Preconnect, Prerender](phase-06-performance/02-resource-hinting-preload-prefetch-preconnect.md) — declarative priority signals to the browser
- [ ] [Code Splitting & Dynamic Imports](phase-06-performance/03-code-splitting-dynamic-imports.md) — route-level, component-level, import() patterns
- [ ] [List Virtualization & Windowing](phase-06-performance/04-list-virtualization-windowing.md) — render only what's visible, react-virtual / react-window
- [ ] [Tree Shaking & Dead Code Elimination](phase-06-performance/05-tree-shaking-dead-code-elimination.md) — ESM static analysis, sideEffects flag, common footguns
- [ ] [Asset Optimization: WebP/AVIF, Font Subsetting, SVGO](phase-06-performance/06-asset-optimization-webp-avif-fonts-svgo.md) — format selection, subsetting, build-time optimization
- [ ] [CSS Performance: Critical CSS, CSS-in-JS, Utility CSS](phase-06-performance/07-css-performance-critical-css-in-js-utility.md) — render-blocking styles, runtime injection cost
- [ ] [Animation Performance: FLIP, CSS vs. JS, requestAnimationFrame](phase-06-performance/08-animation-performance-flip-css-vs-js-raf.md) — compositor-only animations, FLIP for layout animations

---

## Phase 7 — Concurrency, Parallelism & Scheduling

> Moving work off the main thread. The browser's concurrency model and the APIs it exposes.

- [ ] [Web Workers & Dedicated Workers](phase-07-concurrency/01-web-workers-dedicated-workers.md) — true parallelism for CPU-bound tasks, postMessage boundary
- [ ] [Service Workers: Lifecycle, Caching, Background Sync](phase-07-concurrency/02-service-workers-lifecycle-caching-background-sync.md) — install/activate/fetch lifecycle, network proxy
- [ ] [Shared Workers & Cross-tab Communication](phase-07-concurrency/03-shared-workers-cross-tab-communication.md) — shared state across tabs, BroadcastChannel alternative
- [ ] [Worklets: Audio, Paint, Animation](phase-07-concurrency/04-worklets-audio-paint-animation.md) — CSS Houdini, custom layout and paint, audio processing
- [ ] [SharedArrayBuffer & Atomics](phase-07-concurrency/05-sharedarraybuffer-atomics.md) — true shared memory, COOP/COEP headers requirement
- [ ] [Concurrency Scheduling: React Concurrent Mode, scheduler.postTask](phase-07-concurrency/06-concurrency-scheduling-react-concurrent-scheduler.md) — cooperative scheduling, priority lanes
- [ ] [OffscreenCanvas](phase-07-concurrency/07-offscreen-canvas.md) — canvas rendering in a worker, transferable objects

---

## Phase 8 — Browser Storage & Offline

> The full storage landscape. Which API fits which access pattern, and how to build apps that work without a network.

- [ ] [Cookie Strategy: HttpOnly, SameSite, Secure, Partitioned](phase-08-browser-storage/01-cookie-strategy-httponly-samesite-secure.md) — security attributes, third-party cookie deprecation
- [ ] [LocalStorage vs. SessionStorage](phase-08-browser-storage/02-localstorage-vs-sessionstorage.md) — synchronous, string-only, why not for auth tokens
- [ ] [IndexedDB: Schema Design & Transactions](phase-08-browser-storage/03-indexeddb-schema-transactions.md) — object stores, indexes, versioned migrations
- [ ] [Cache API (Service Worker Layer)](phase-08-browser-storage/04-cache-api-service-worker-layer.md) — request/response storage, cache-first vs. network-first strategies
- [ ] [Storage Quotas & Eviction Policies](phase-08-browser-storage/05-storage-quotas-eviction-policies.md) — origin limits, persistent storage, LRU eviction
- [ ] [PWA: App Shell, Background Sync, Web Push](phase-08-browser-storage/06-pwa-app-shell-background-sync-web-push.md) — installability, offline UI shell, push notification flow
- [ ] [Offline-first Architecture Patterns](phase-08-browser-storage/07-offline-first-architecture.md) — optimistic local writes, sync-on-reconnect, conflict resolution

---

## Phase 9 — Client-Side Routing & URL Design

> How SPAs manage navigation without full page reloads, and why the URL is a first-class state store.

- [ ] [History API vs. Hash Routing](phase-09-routing/01-history-api-vs-hash-routing.md) — pushState, replaceState, why hash routing still exists
- [ ] [File-system Based vs. Config-based Routing](phase-09-routing/02-filesystem-vs-config-based-routing.md) — Next.js/Remix file conventions vs. React Router config
- [ ] [Nested Routing & Layout Inheritance](phase-09-routing/03-nested-routing-layout-inheritance.md) — Outlet pattern, shared layouts, URL hierarchy maps to UI hierarchy
- [ ] [Route-level Code Splitting](phase-09-routing/04-route-level-code-splitting.md) — lazy routes, loading state, prefetch on hover
- [ ] [Navigation Guards & Middleware](phase-09-routing/05-navigation-guards-middleware.md) — auth redirects, route lifecycle, loader-based guards
- [ ] [URL as State: Deep Linking & Shareability](phase-09-routing/06-url-as-state-deep-linking.md) — search params, when to push state to URL vs. component state
- [ ] [Scroll Restoration](phase-09-routing/07-scroll-restoration.md) — browser default behavior, manual control, per-route memory

---

## Phase 10 — Module Systems & Build Infrastructure

> The mechanics of how JS is packaged, transformed, and delivered. What bundlers actually do and why it matters.

- [ ] [ESM vs. CJS vs. UMD — Interop Challenges](phase-10-module-systems/01-esm-vs-cjs-vs-umd.md) — static vs. dynamic imports, dual-package hazard, interop shims
- [ ] [Bundlers: Webpack, Vite, Esbuild, Turbopack, Rollup](phase-10-module-systems/02-bundlers-webpack-vite-esbuild-turbopack-rollup.md) — dev server strategy, HMR, production output trade-offs
- [ ] [Tree Shaking, Scope Hoisting, Chunk Splitting](phase-10-module-systems/03-tree-shaking-scope-hoisting-chunk-splitting.md) — dead code elimination at bundle level, module concatenation
- [ ] [Source Maps Strategy (dev vs. prod)](phase-10-module-systems/04-source-maps-strategy.md) — types, security implications, eval vs. file maps
- [ ] [Polyfill Strategy: Differential Serving, browserslist, core-js](phase-10-module-systems/05-polyfill-strategy-differential-serving.md) — `<script type="module">` differential, usage data-driven polyfills
- [ ] [Environment Config Management & Package Versioning at Scale](phase-10-module-systems/06-environment-config-package-versioning.md) — .env vs. build-time injection, semver, lockfiles

---

## Phase 11 — Web Components & Native Browser Primitives

> The platform's own component model. Increasingly relevant as a boundary in micro-frontend and design system architectures.

- [ ] [Custom Elements Lifecycle](phase-11-web-components/01-custom-elements-lifecycle.md) — connectedCallback, disconnectedCallback, attributeChangedCallback
- [ ] [Shadow DOM: Encapsulation, Open vs. Closed Mode](phase-11-web-components/02-shadow-dom.md) — style isolation, event retargeting, piercing shadow root
- [ ] [HTML Templates & Slots](phase-11-web-components/03-html-templates-slots.md) — `<template>`, named slots, slotchange event
- [ ] [Interoperability with Frameworks](phase-11-web-components/04-framework-interoperability.md) — React/Vue/Angular handling of custom events, property vs. attribute

---

## Phase 12 — Observer & Intersection APIs

> The modern event-driven APIs that replace scroll/resize listeners. Non-negotiable for performance-conscious frontend work.

- [ ] [IntersectionObserver](phase-12-observer-apis/01-intersection-observer.md) — lazy loading, infinite scroll, analytics visibility tracking
- [ ] [ResizeObserver](phase-12-observer-apis/02-resize-observer.md) — element-level resize, replaces window resize for component-aware layout
- [ ] [MutationObserver](phase-12-observer-apis/03-mutation-observer.md) — DOM change detection, use cases and performance gotchas
- [ ] [PerformanceObserver](phase-12-observer-apis/04-performance-observer.md) — observing LCP/CLS/FID entries programmatically
- [ ] [Observer APIs vs. Event Listener Patterns](phase-12-observer-apis/05-observer-vs-event-listeners.md) — async observation vs. synchronous events, when each is appropriate

---

## Phase 13 — Security

> The attack surface of modern web applications. Know both the attack and the defense.

- [ ] [Cross-Site Scripting (XSS): DOM vs. Reflected vs. Stored](phase-13-security/01-xss-dom-reflected-stored.md) — injection points, framework escaping, dangerouslySetInnerHTML
- [ ] [Cross-Site Request Forgery (CSRF) Protection Patterns](phase-13-security/02-csrf-protection-patterns.md) — SameSite cookies, CSRF tokens, when SPAs are safe
- [ ] [Content Security Policy (CSP): Nonces, Hashes, strict-dynamic](phase-13-security/03-content-security-policy-csp.md) — blocking inline scripts, nonce rotation in SSR, reporting
- [ ] [CORS Configuration & Preflight](phase-13-security/04-cors-preflight.md) — simple vs. preflighted requests, credentials, common misconfigurations
- [ ] [Auth & Authorization: JWT, OAuth2, OIDC, HttpOnly Cookies](phase-13-security/05-authentication-jwt-oauth2-oidc-cookies.md) — token storage trade-offs, refresh flow, silent renew
- [ ] [Subresource Integrity (SRI)](phase-13-security/06-subresource-integrity-sri.md) — hash validation for CDN-served assets
- [ ] [Clickjacking: X-Frame-Options & frame-ancestors](phase-13-security/07-clickjacking-x-frame-options.md) — framing attacks, CSP frame-ancestors vs. X-Frame-Options
- [ ] [Trusted Types API](phase-13-security/08-trusted-types-api.md) — browser-enforced DOM XSS prevention, policy objects

---

## Phase 14 — TypeScript & Type System Architecture

> TypeScript beyond the basics. The patterns that make large-scale typed codebases maintainable.

- [ ] [Structural Typing, Generics, Conditional & Mapped Types](phase-14-typescript/01-structural-typing-generics-conditional-mapped.md) — duck typing, type algebra, derive types from types
- [ ] [Module Augmentation & Declaration Merging](phase-14-typescript/02-module-augmentation-declaration-merging.md) — extending third-party types, global augmentation
- [ ] [Strict Mode Trade-offs](phase-14-typescript/03-strict-mode-tradeoffs.md) — noUncheckedIndexedAccess, exactOptionalPropertyTypes, migration cost
- [ ] [Type-safe API Contracts: Zod, tRPC, OpenAPI Codegen](phase-14-typescript/04-type-safe-api-contracts-zod-trpc-openapi.md) — runtime validation, end-to-end type safety approaches
- [ ] [Monorepo Shared Type Packages](phase-14-typescript/05-monorepo-shared-type-packages.md) — package boundaries, path mapping, composite projects

---

## Phase 15 — Memory Management

> What leaks, why it leaks, and how to find it. Browser JS has GC but it's not magic.

- [ ] [Memory Leaks: Detached DOM Nodes, Closures, Event Listeners](phase-15-memory-management/01-memory-leaks-detached-dom-closures-listeners.md) — the three main leak patterns, how GC roots work
- [ ] [WeakMap, WeakRef & FinalizationRegistry](phase-15-memory-management/02-weakmap-weakref-finalizationregistry.md) — weak references, GC-friendly caches, cleanup callbacks
- [ ] [Heap Profiling & DevTools Memory Snapshots](phase-15-memory-management/03-heap-profiling-devtools-snapshots.md) — allocation timeline, retained size vs. shallow size, finding leaks
- [ ] [Object Pooling for High-frequency Allocations](phase-15-memory-management/04-object-pooling.md) — reduce GC pressure in hot paths, game loops and animation

---

## Phase 16 — WebAssembly (WASM)

> When JavaScript is the bottleneck and you need near-native performance in the browser.

- [ ] [WASM Use Cases: Compute-heavy Workloads, Porting Native Code](phase-16-webassembly/01-wasm-use-cases.md) — image processing, codecs, physics engines, the JS boundary cost
- [ ] [JS ↔ WASM Interop & Memory Model](phase-16-webassembly/02-js-wasm-interop-memory-model.md) — linear memory, typed arrays, string marshalling overhead
- [ ] [WASM Threads & SIMD](phase-16-webassembly/03-wasm-threads-simd.md) — SharedArrayBuffer requirement, data parallelism with SIMD
- [ ] [WASM Tools: Emscripten, wasm-pack, AssemblyScript](phase-16-webassembly/04-wasm-tools-emscripten-wasm-pack-assemblyscript.md) — C/C++ vs. Rust vs. TypeScript-like compilation paths

---

## Phase 17 — Resilience & Observability

> What happens when things go wrong, and how you know they went wrong before your users tell you.

- [ ] [Error Boundaries & Global Error Handlers](phase-17-resilience-observability/01-error-boundaries-global-error-handlers.md) — React error boundaries, window.onerror, unhandledrejection
- [ ] [Graceful Degradation & Progressive Enhancement](phase-17-resilience-observability/02-graceful-degradation-progressive-enhancement.md) — baseline vs. enhanced experience, feature detection
- [ ] [Real User Monitoring (RUM) vs. Synthetic Monitoring](phase-17-resilience-observability/03-rum-vs-synthetic-monitoring.md) — real traffic data vs. controlled lab testing, complementary roles
- [ ] [Logging, Tracing & Session Replay](phase-17-resilience-observability/04-logging-tracing-session-replay.md) — Sentry, LogRocket, OpenTelemetry, what to instrument
- [ ] [Performance Budgets & Regression Alerts](phase-17-resilience-observability/05-performance-budgets-regression-alerts.md) — enforcing limits in CI, alerting on metric degradation

---

## Phase 18 — CI/CD, Deployment & Infrastructure

> The delivery pipeline for frontend code. From commit to users, and the patterns that make it safe.

- [ ] [CDN Strategy & Cache Invalidation](phase-18-cicd-deployment/01-cdn-strategy-cache-invalidation.md) — content-hashed filenames, cache-busting, stale asset risk
- [ ] [Edge Functions & Edge Rendering](phase-18-cicd-deployment/02-edge-functions-edge-rendering.md) — colocating compute with users, cold starts, runtime constraints
- [ ] [Blue-Green, Canary & Rolling Deployments](phase-18-cicd-deployment/03-blue-green-canary-rolling-deployments.md) — zero-downtime strategies, traffic splitting, rollback
- [ ] [Feature Flags: Client-side vs. Server-side Evaluation](phase-18-cicd-deployment/04-feature-flags-client-vs-server.md) — LaunchDarkly/GrowthBook, flicker, consistent bucketing
- [ ] [A/B Testing & Experimentation Infrastructure](phase-18-cicd-deployment/05-ab-testing-experimentation.md) — assignment, holdouts, statistical significance, client-side lag
- [ ] [Bundle Analysis in CI Pipeline](phase-18-cicd-deployment/06-bundle-analysis-in-ci.md) — size limits, diff reports, catching accidental bloat
- [ ] [Lighthouse CI & Performance Budgeting in CI](phase-18-cicd-deployment/07-lighthouse-ci-performance-budgeting.md) — automated perf regression detection, thresholds and reporting

---

## Phase 19 — SEO & Discoverability

> How search engines and social platforms see your app, and what rendering strategy choices cost you.

- [ ] [Crawlability: SSR/SSG vs. CSR for Bots](phase-19-seo/01-crawlability-ssr-ssg-vs-csr.md) — Googlebot's JS rendering, crawl budget, why CSR still hurts indexing
- [ ] [Structured Data: JSON-LD & Schema.org](phase-19-seo/02-structured-data-json-ld-schema.md) — rich results, entity types, validation
- [ ] [Open Graph & Twitter Card Meta](phase-19-seo/03-open-graph-twitter-card-meta.md) — social sharing previews, og:image generation
- [ ] [Sitemap Generation & Canonical URLs](phase-19-seo/04-sitemap-canonical-urls.md) — programmatic sitemaps, duplicate content signals, rel=canonical
- [ ] [Dynamic Meta for SPAs](phase-19-seo/05-dynamic-meta-for-spas.md) — client-side title/meta updates, SSR meta injection, prerendering services

---

## Phase 20 — Accessibility (A11y)

> Building interfaces usable by everyone, including users with assistive technology. A legal and ethical requirement.

- [ ] [WAI-ARIA: Roles, States & Properties](phase-20-accessibility/01-wai-aria-roles-states-properties.md) — landmark roles, live regions, aria-expanded/pressed/selected
- [ ] [Keyboard Navigation & Focus Management](phase-20-accessibility/02-keyboard-navigation-focus-management.md) — focus trap in modals, skip links, programmatic focus, tabindex
- [ ] [Screen Reader Behavior & Announcement Patterns](phase-20-accessibility/03-screen-reader-behavior-announcements.md) — NVDA/VoiceOver differences, aria-live, polite vs. assertive
- [ ] [Color Contrast & Motion Sensitivity](phase-20-accessibility/04-color-contrast-motion-sensitivity.md) — WCAG contrast ratios, prefers-reduced-motion media query
- [ ] [Accessibility Tree & How Browsers Expose It](phase-20-accessibility/05-accessibility-tree-browser-exposure.md) — parallel tree to DOM, AXObject model, DevTools a11y panel

---

## Phase 21 — Internationalization (i18n) & Localization (l10n)

> Shipping software that works for users in every locale, language, and script direction.

- [ ] [Translation Loading Strategies: Static Bundle vs. On-demand](phase-21-i18n/01-translation-loading-strategies.md) — bundle splitting by locale, namespace loading, cold-start vs. waterfall
- [ ] [RTL Layout Support](phase-21-i18n/02-rtl-layout-support.md) — CSS logical properties, dir attribute, bidirectional text edge cases
- [ ] [Pluralization Rules (CLDR)](phase-21-i18n/03-pluralization-rules-cldr.md) — beyond singular/plural, ICU message format, 6 CLDR plural categories
- [ ] [Date, Time, Number & Currency Formatting (Intl API)](phase-21-i18n/04-date-time-number-currency-intl-api.md) — Intl.DateTimeFormat, Intl.NumberFormat, timezone edge cases
- [ ] [Locale Detection & Negotiation](phase-21-i18n/05-locale-detection-negotiation.md) — Accept-Language header, navigator.language, user override persistence

---

## Phase 22 — Testing Strategy

> Not just which tools to use, but what to test, at which layer, and why the layers matter.

- [ ] [Unit Testing: Logic, Utils & Hooks](phase-22-testing/01-unit-testing-logic-utils-hooks.md) — isolated functions, pure logic testing, renderHook
- [ ] [Component Testing: Testing Library Philosophy](phase-22-testing/02-component-testing-testing-library.md) — query by role not testid, user behavior over implementation
- [ ] [Integration Testing](phase-22-testing/03-integration-testing.md) — multiple units together, MSW for API mocking, test reality
- [ ] [End-to-End Testing: Playwright & Cypress](phase-22-testing/04-e2e-testing-playwright-cypress.md) — full browser automation, when it's worth the cost
- [ ] [Visual Regression Testing: Percy & Chromatic](phase-22-testing/05-visual-regression-testing-percy-chromatic.md) — screenshot diffing, Storybook integration, approval workflow
- [ ] [Contract Testing: Pact & Schema Validation](phase-22-testing/06-contract-testing-pact-schema-validation.md) — consumer-driven contracts, catching API drift without E2E
- [ ] [Performance Budgeting in CI](phase-22-testing/07-performance-budgeting-in-ci.md) — Lighthouse CI, bundlesize, automated regressions

---

## Phase 23 — Developer Experience (DX) & Tooling

> The tooling layer that keeps large teams productive and consistent. Often undervalued, always noticed when missing.

- [ ] [Linting & Formatting at Scale](phase-23-dx-tooling/01-linting-formatting-at-scale.md) — ESLint flat config, shared configs, Prettier integration, performance
- [ ] [Commit Hygiene: Husky, lint-staged, Conventional Commits](phase-23-dx-tooling/02-commit-hygiene-husky-lint-staged.md) — pre-commit hooks, incremental linting, changelog generation
- [ ] [Local Development Parity with Production](phase-23-dx-tooling/03-local-dev-parity-docker-env-mocking.md) — Docker compose, env var management, mocking external services
- [ ] [Storybook-driven Development](phase-23-dx-tooling/04-storybook-driven-development.md) — component isolation, CSF format, interaction testing, a11y addon
- [ ] [Code Generation: Plop, Hygen, OpenAPI Codegen](phase-23-dx-tooling/05-code-generation-plop-hygen-openapi.md) — scaffolding consistency, API client generation from spec
- [ ] [IDE Integration & TypeScript Server Performance](phase-23-dx-tooling/06-ide-integration-typescript-server-performance.md) — tsserver, project references, skipLibCheck, language server tuning
