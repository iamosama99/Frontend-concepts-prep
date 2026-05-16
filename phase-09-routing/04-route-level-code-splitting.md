# Route-level Code Splitting

## Quick Reference

| Mechanism | Framework | How |
|---|---|---|
| `React.lazy` + `Suspense` | React (any router) | `const Page = lazy(() => import('./Page'))` |
| `lazy()` in route config | React Router v6 | `lazy: () => import('./Page').then(m => ({ Component: m.default }))` |
| Automatic per-page splitting | Next.js, SvelteKit, Remix | Built-in — each page is a separate chunk |
| `defineAsyncComponent` | Vue 3 | `const Page = defineAsyncComponent(() => import('./Page.vue'))` |
| `loadChildren` | Angular | `loadChildren: () => import('./feature/module').then(m => m.FeatureModule)` |

---

## What Is This?

By default, bundlers (webpack, Vite) pack all application code into a single JavaScript bundle. Every user downloads all routes — even ones they never visit. Route-level code splitting creates a separate bundle chunk per route. When the user navigates to `/settings`, the settings chunk loads. The `/dashboard` chunk never downloads unless the user goes there.

The result: a smaller initial bundle, faster first load, same total bytes consumed over a full session.

```
Without splitting:           With route splitting:
main.bundle.js               main.bundle.js       (small — router + shell)
  ├─ Home component            chunks/
  ├─ Dashboard component         home.[hash].js     (loads on /)
  ├─ Settings component          dashboard.[hash].js (loads on /dashboard)
  ├─ Reports component           settings.[hash].js  (loads on /settings)
  └─ ...all routes               reports.[hash].js   (loads on /reports)
```

> **Check yourself:** Why does route splitting reduce initial load time but not total bytes over a session?

---

## Why It Matters

For large apps, the initial bundle can easily exceed 1MB. Route splitting is the first and highest-impact code splitting technique. A user who only uses the main dashboard never downloads the admin panel code. Understanding how to implement it — and its loading state implications — is standard senior-level knowledge.

---

## React: `React.lazy` + `Suspense`

The primitive approach works with any router:

```jsx
import { lazy, Suspense } from 'react';

const Dashboard = lazy(() => import('./pages/Dashboard'));
const Settings = lazy(() => import('./pages/Settings'));

function App() {
  return (
    <Suspense fallback={<PageSpinner />}>
      <Routes>
        <Route path="/dashboard" element={<Dashboard />} />
        <Route path="/settings" element={<Settings />} />
      </Routes>
    </Suspense>
  );
}
```

`lazy()` takes a function that returns a promise resolving to a module with a `default` export. The bundler statically analyzes the `import()` call and creates a separate chunk for `./pages/Dashboard`. The component renders only after the chunk has loaded. During loading, `Suspense` renders the `fallback`.

### Named exports with `lazy`

`lazy` requires a default export. For named exports:

```js
const Settings = lazy(() =>
  import('./pages/Settings').then(module => ({ default: module.Settings }))
);
```

---

## React Router v6 — `lazy` in Route Config

React Router v6.4+ supports a `lazy` property on route objects that integrates with the router's fetch lifecycle:

```jsx
const router = createBrowserRouter([
  {
    path: '/dashboard',
    lazy: async () => {
      const { default: Component, loader } = await import('./pages/Dashboard');
      return { Component, loader }; // can also return loader, action, errorElement, etc.
    },
  },
  {
    path: '/settings',
    lazy: () => import('./pages/Settings').then(m => ({ Component: m.default })),
  },
]);
```

The advantage over `React.lazy`: the route's `loader` function is also code-split. The chunk containing both the component and the data-fetching logic loads simultaneously when the route is activated, so data fetching starts without waiting for the component to render first.

---

## Next.js — Automatic Code Splitting

Next.js handles route splitting automatically. Every `page.js` / `app/**/page.js` is a separate chunk. No configuration required.

For non-route components you want to split manually:

```jsx
import dynamic from 'next/dynamic';

// Lazy-load a heavy component (e.g., a chart library)
const HeavyChart = dynamic(() => import('./HeavyChart'), {
  loading: () => <ChartSkeleton />,
  ssr: false, // don't render on the server (useful for canvas/WebGL)
});

export default function Dashboard() {
  return <HeavyChart data={data} />;
}
```

---

## Vue 3 — `defineAsyncComponent` + Vue Router Lazy Routes

### Component-level

```js
import { defineAsyncComponent } from 'vue';

const HeavyModal = defineAsyncComponent({
  loader: () => import('./HeavyModal.vue'),
  loadingComponent: Spinner,
  errorComponent: ErrorDisplay,
  delay: 200,     // wait 200ms before showing loading state (prevents flicker)
  timeout: 5000,  // show error if load takes >5s
});
```

### Route-level (automatic in Vue Router)

Vue Router lazy-loads route components when you pass a function instead of a component directly:

```js
const routes = [
  {
    path: '/dashboard',
    component: () => import('./views/Dashboard.vue'), // auto-split
  },
  {
    path: '/settings',
    component: () => import('./views/Settings.vue'),
  },
];
```

Each route's component becomes a separate chunk with zero configuration. The bundler detects the dynamic `import()` and splits automatically.

---

## Prefetching Route Chunks

Loading a chunk on navigation introduces latency — the user clicks, then waits for the chunk to download before the page renders. Prefetching eliminates this by loading chunks before the user navigates to them.

### On hover (most impactful)

```jsx
function NavLink({ to, children }) {
  const handleMouseEnter = () => {
    // React Router doesn't built-in prefetch, but you can trigger dynamic imports manually
    import(`./pages/${to}`); // bundler splits this per-call if paths are static
  };

  return (
    <Link to={to} onMouseEnter={handleMouseEnter}>
      {children}
    </Link>
  );
}
```

### On link visibility (IntersectionObserver)

```js
const observer = new IntersectionObserver((entries) => {
  for (const entry of entries) {
    if (entry.isIntersecting) {
      const href = entry.target.getAttribute('href');
      prefetchRoute(href);
      observer.unobserve(entry.target);
    }
  }
});

document.querySelectorAll('a[data-prefetch]').forEach(link => observer.observe(link));
```

### Next.js — automatic prefetch

Next.js's `<Link>` component prefetches the linked page's chunk when the link enters the viewport (in production). Opt out with `prefetch={false}`. This is why Next.js apps feel instant even with route splitting.

---

## Avoiding Loading State Flicker

A chunk that loads in <100ms will cause the loading indicator to flash briefly, which looks worse than no indicator at all. Delay the fallback:

```jsx
// Custom lazy component with delay
function LazyRoute({ component: LazyComponent }) {
  const [showFallback, setShowFallback] = useState(false);

  useEffect(() => {
    const timer = setTimeout(() => setShowFallback(true), 200);
    return () => clearTimeout(timer);
  }, []);

  return (
    <Suspense fallback={showFallback ? <PageSpinner /> : null}>
      <LazyComponent />
    </Suspense>
  );
}
```

Vue Router's `defineAsyncComponent` has a built-in `delay` option that does this automatically.

---

## Chunk Naming and Analysis

By default, chunks get hash-based names (`3f8a.js`). Name them with webpackChunkName magic comments:

```js
const Dashboard = lazy(() =>
  import(/* webpackChunkName: "dashboard" */ './pages/Dashboard')
);
```

With Vite, use `rollupOptions.output.chunkFileNames` or `vite-plugin-inspect` to analyze chunks. Always run a bundle analysis before and after adding splitting to verify it's having the intended effect.

```js
// vite.config.js
import { defineConfig } from 'vite';

export default defineConfig({
  build: {
    rollupOptions: {
      output: {
        manualChunks: {
          vendor: ['react', 'react-dom', 'react-router-dom'],
          // Split vendor code from app code
        },
      },
    },
  },
});
```

---

## Gotchas

**1. Every `Suspense` boundary catches all lazy children below it**
If you put one `<Suspense>` at the app root with a full-page spinner, navigating to any route shows the full-page spinner — even for tiny chunks. Scope `Suspense` boundaries to the route level: show a page skeleton instead.

**2. `React.lazy` only works with default exports**
Named export modules require the `.then(m => ({ default: m.Named }))` workaround. Forgetting this produces a cryptic error about the component not being valid.

**3. Code splitting doesn't help if all routes share a large vendor chunk**
If your charting library is imported in the main bundle, splitting route chunks saves nothing for users who load that library anyway. Put heavy dependencies in their own chunk or import them only in the routes that need them.

**4. Dynamic import paths must be statically analyzable**
`import(`./${routeName}`)` with a runtime variable works at runtime but the bundler can't determine which files to split. Vite/webpack will either warn or include the entire directory as potential chunks. Always use static strings or static partial paths.

**5. Server-side rendering and `React.lazy`**
`React.lazy` does not work with SSR out of the box. Use `next/dynamic` (Next.js), `@loadable/component` (React), or framework-specific lazy loading that supports SSR. Streaming SSR (React 18) enables Suspense on the server, but `React.lazy` still requires hydration-safe handling.

---

## Interview Questions

**Q (High): How does route-level code splitting work? Walk through the technical mechanism.**

Answer: When a bundler (webpack, Vite, Rollup) encounters a dynamic `import()` call with a static path, it creates a separate output chunk for that module and its unique dependencies. At runtime, when the `import()` function is called — triggered by route navigation — the browser makes an additional HTTP request for that chunk file. `React.lazy()` wraps this async import and integrates with `Suspense` to show a fallback while the chunk loads. Once the chunk is available, the component renders. The bundler statically analyzes all `import()` calls to determine the chunk graph at build time; the split happens at build, the loading happens at runtime.

**Q (High): What is the advantage of React Router v6's route-level `lazy` property over using `React.lazy` directly?**

Answer: React Router's `lazy` property code-splits both the component and the route's `loader` (data fetching function) into the same chunk. When the user navigates to the route, the chunk loads, and then the loader runs — data fetching starts as soon as the chunk is available, without waiting for the component to render first. With `React.lazy`, the component chunk loads, the component mounts, and then data fetching begins inside `useEffect` or `useLoaderData` — a waterfall. The route-level `lazy` property eliminates this waterfall by colocating the component and loader in the same chunk that loads in one request.

**Q (Medium): How do you prefetch route chunks to eliminate navigation latency?**

Answer: Prefetch by triggering the dynamic `import()` before the user navigates. On hover over a navigation link is the most impactful: users typically hover ~150ms before clicking, which is enough time for a cached or fast CDN response. On viewport entry (IntersectionObserver) is useful for below-the-fold links that will likely be clicked as the user scrolls. Next.js's `<Link>` does viewport-based prefetching automatically in production. For manual implementations, call `import('./pages/Dashboard')` early — browsers cache the module so the actual navigation is instant. The key insight: the prefetch call returns a promise; you don't need to handle the result, just trigger the load.

**Q (Medium): Why does placing a single `<Suspense>` at the app root cause a poor user experience with route splitting?**

Answer: A single root Suspense boundary means any lazy chunk loading anywhere in the tree causes the root fallback to render — typically a full-page spinner or blank screen. This replaces the entire app UI, including the navigation and layout that should persist, with a loading state. Users lose context of where they are in the app. The better approach: place `<Suspense>` at each route boundary, wrapping only the page content area. The layout, navigation, and shell remain rendered; only the page content area shows a skeleton or spinner during chunk loading. This matches the user's mental model (navigation stayed, content is loading) and preserves UI context.

**Q (Low): What does `ssr: false` do in Next.js `dynamic()` and when would you use it?**

Answer: `ssr: false` tells Next.js not to render or execute the component during server-side rendering. The component only loads and renders in the browser. Use it for components that depend on browser-only APIs (`window`, `document`, `canvas`, WebGL, `navigator`) that aren't available in Node.js. Also use it for heavy visualization libraries (D3, Chart.js, Three.js) when you don't need server-rendered output — the component loads client-side after hydration, keeping the server bundle clean. Without `ssr: false`, Next.js would attempt to server-render the component, fail if it uses browser APIs, and throw a hydration error.

---

## Self-Assessment

- [ ] Explain how dynamic `import()` causes the bundler to create a separate chunk
- [ ] Implement route-level splitting with `React.lazy` and `Suspense`
- [ ] Describe the advantage of React Router v6's `lazy` over `React.lazy` for data loading
- [ ] Explain two strategies for prefetching route chunks
- [ ] Describe why a single root-level Suspense boundary produces a bad loading experience

---
*Next: Navigation Guards & Middleware — how to intercept and control navigation before routes render.*
