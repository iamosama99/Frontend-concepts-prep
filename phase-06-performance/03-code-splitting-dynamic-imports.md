# Code Splitting & Dynamic Imports

## Quick Reference

| Strategy | Trigger | Bundler mechanism | Use case |
|---|---|---|---|
| Route-level splitting | Navigation | `React.lazy` + `import()` | Each route is its own chunk |
| Component-level splitting | Interaction / condition | `React.lazy` + `import()` | Modals, heavy editors, charts |
| Vendor chunk splitting | Build config | `splitChunks` / `manualChunks` | Separate stable dependencies for caching |
| Dynamic import | Runtime condition | `import()` | Features loaded on demand (polyfills, locales) |
| `modulepreload` | Anticipation | `<link rel="modulepreload">` | Pre-fetch next route's chunks on hover |

---

## What Is This?

Code splitting is the practice of dividing a JavaScript bundle into multiple smaller chunks that are loaded on demand rather than all at once. Instead of shipping one large `bundle.js` that contains every route, component, and library, the browser downloads only what is needed to render the current view. The remaining chunks are fetched lazily as the user navigates or interacts.

The mechanism that enables code splitting is the **dynamic `import()`** expression. Unlike static `import` declarations (which are resolved at build time and always included in the bundle), dynamic `import()` returns a Promise that resolves to the module when it is loaded. Bundlers like Webpack and Vite recognize this pattern and automatically emit a separate chunk for the dynamically imported module.

> **Check yourself:** What is the difference between `import React from 'react'` and `const { default: React } = await import('react')`? Which one can be code-split?

---

## Why Does It Exist?

JavaScript is the most expensive resource type on the web. Every byte of JS must be downloaded, parsed, compiled, and executed before it can do anything. A 1 MB JavaScript bundle parsed on a mid-range Android device takes 6–10 seconds to evaluate — even after the bytes arrive.

Most of that 1 MB is code the user will never see on this page load: the checkout flow loaded on the homepage, the admin dashboard loaded for a regular user, the rich-text editor loaded before the user clicks "Create Post." Code splitting ensures the browser does only the parsing and compilation work that the current view actually requires.

---

## How It Works

### Dynamic `import()`

`import()` is an ES2020 expression that instructs the bundler to split the target module into a separate chunk file. At runtime, it fetches and evaluates that chunk asynchronously.

```js
// Static import — always in the main bundle
import HeavyChart from './HeavyChart';

// Dynamic import — split into a separate chunk, fetched on demand
const { default: HeavyChart } = await import('./HeavyChart');

// As a Promise
import('./HeavyChart').then(({ default: HeavyChart }) => {
  // HeavyChart is available here after the chunk loads
});
```

Webpack and Vite detect `import('./HeavyChart')` at build time and emit a file like `chunk-HeavyChart-abc123.js`. That file is not in the initial bundle — it is fetched from the server only when `import('./HeavyChart')` executes.

---

### Route-Level Splitting with `React.lazy`

`React.lazy` wraps a dynamic import and integrates it with React's rendering lifecycle and `Suspense`. When the component is rendered for the first time, React initiates the dynamic import, shows the `Suspense` fallback while loading, and renders the component once the chunk is available.

```js
import React, { Suspense, lazy } from 'react';

// Each route component is a separate chunk
const Home = lazy(() => import('./pages/Home'));
const Checkout = lazy(() => import('./pages/Checkout'));
const Dashboard = lazy(() => import('./pages/Dashboard'));

function App() {
  return (
    <Suspense fallback={<div className="page-loader" />}>
      <Routes>
        <Route path="/" element={<Home />} />
        <Route path="/checkout" element={<Checkout />} />
        <Route path="/dashboard" element={<Dashboard />} />
      </Routes>
    </Suspense>
  );
}
```

The `Suspense` boundary catches the loading state. Always wrap lazily-loaded components with an `ErrorBoundary` as well — if the chunk fails to load (network error, deploy invalidating old chunk URLs), the Promise rejects and an error boundary prevents a full tree crash.

```js
import { Component } from 'react';

class ChunkErrorBoundary extends Component {
  state = { hasError: false };

  static getDerivedStateFromError() {
    return { hasError: true };
  }

  render() {
    if (this.state.hasError) {
      return <button onClick={() => window.location.reload()}>Reload page</button>;
    }
    return this.props.children;
  }
}

// Usage
<ChunkErrorBoundary>
  <Suspense fallback={<Spinner />}>
    <Checkout />
  </Suspense>
</ChunkErrorBoundary>
```

---

### Component-Level Splitting

Split components that are heavy and not needed on initial render: rich-text editors, syntax highlighters, image croppers, PDF viewers, complex charts.

```js
import { lazy, Suspense, useState } from 'react';

// Only loaded when the user clicks "Write a review"
const RichTextEditor = lazy(() => import('./RichTextEditor'));

function ReviewForm() {
  const [open, setOpen] = useState(false);

  return (
    <div>
      <button onClick={() => setOpen(true)}>Write a review</button>
      {open && (
        <Suspense fallback={<div>Loading editor...</div>}>
          <RichTextEditor />
        </Suspense>
      )}
    </div>
  );
}
```

The editor chunk is only fetched when `open` becomes true — when the user clicks the button. Until then, the chunk does not appear in the browser's network activity.

---

### Vendor Chunk Splitting

Third-party libraries (React, lodash, date-fns) change rarely. Splitting them into a separate vendor chunk allows the browser to cache that chunk across deploys — as long as the libraries haven't updated, the vendor chunk hash stays the same and the browser uses its cached version rather than re-downloading.

**Webpack:**

```js
// webpack.config.js
module.exports = {
  optimization: {
    splitChunks: {
      cacheGroups: {
        react: {
          test: /node_modules\/(react|react-dom)\//,
          name: 'vendor-react',
          chunks: 'all',
          priority: 30,
        },
        vendor: {
          test: /node_modules/,
          name: 'vendor',
          chunks: 'all',
          priority: 20,
        },
      },
    },
  },
};
```

**Vite:**

```js
// vite.config.js
export default {
  build: {
    rollupOptions: {
      output: {
        manualChunks: {
          'vendor-react': ['react', 'react-dom'],
          'vendor-router': ['react-router-dom'],
          'vendor-ui': ['@radix-ui/react-dialog', '@radix-ui/react-dropdown-menu'],
        },
      },
    },
  },
};
```

> **Check yourself:** If your React version updates from 18.2.0 to 18.3.0, what happens to the vendor chunk hash? Does the user re-download just the vendor chunk or the entire application?

---

### Magic Comments for Chunk Control

Webpack supports magic comments inside `import()` to control chunk naming and grouping.

```js
// Named chunk — easier to identify in bundle analysis
const Chart = lazy(() =>
  import(/* webpackChunkName: "chart" */ './HeavyChart')
);

// Prefetch — browser fetches this chunk during idle time after page load
const Checkout = lazy(() =>
  import(/* webpackPrefetch: true */ './pages/Checkout')
);

// Preload — browser fetches this chunk at high priority during current navigation
const Modal = lazy(() =>
  import(/* webpackPreload: true */ './Modal')
);
```

`webpackPrefetch: true` generates `<link rel="prefetch">` for the chunk. `webpackPreload: true` generates `<link rel="preload">`. Use prefetch for next-page resources; use preload only for chunks needed very soon on the current page.

---

### Prefetching on Hover

A common pattern: prefetch the chunk for a route when the user hovers the navigation link. This gives ~200ms of fetch time before the click.

```js
import { lazy, Suspense } from 'react';

const Checkout = lazy(() => import('./pages/Checkout'));

// Trigger the dynamic import on hover — chunk starts loading before click
function prefetchCheckout() {
  import('./pages/Checkout');
}

function Nav() {
  return (
    <nav>
      <Link to="/checkout" onMouseEnter={prefetchCheckout}>
        Checkout
      </Link>
    </nav>
  );
}
```

Calling `import('./pages/Checkout')` a second time does not re-fetch the chunk — the browser (and the module system) caches the result. The `React.lazy` call will resolve instantly from cache when the route renders.

---

## Analyzing Chunk Output

Before splitting, analyze what is in your bundle. Tools:

- **Webpack Bundle Analyzer** — visual treemap of chunk contents
- **Vite's `rollup-plugin-visualizer`** — same concept for Vite/Rollup
- **`source-map-explorer`** — works on any bundled output with source maps

```bash
# Webpack
npx webpack-bundle-analyzer stats.json

# Vite
npx vite build --mode=analyze  # with visualizer plugin configured
```

Look for:
- Large libraries duplicated across chunks
- Vendor code in app chunks (means splitChunks isn't working)
- Test utilities or dev-only code in production output

---

## Gotchas

**Chunk loading failures after a deploy.** When you deploy a new version, old chunk filenames (content-hashed) no longer exist on the server. Users with the old page open will get a 404 when navigating (which triggers a new chunk load). Without an ErrorBoundary, this causes a white screen. Strategies: catch the error and `window.location.reload()`, or configure your CDN to serve old chunks for a grace period.

**Too many small chunks cause waterfall fetching.** If you split aggressively and chunk A imports chunk B at runtime (and B imports C), the browser loads them sequentially: A, then B, then C. Each is a separate network round-trip. Use `modulepreload` or webpack's prefetch comments to load known dependencies in parallel.

**Suspense boundaries in the wrong place.** A single top-level `<Suspense>` causes the entire UI to show a spinner whenever any chunk loads. Use granular Suspense boundaries around the lazy component so only the relevant region shows the loading state.

**`React.lazy` only works with default exports.** `const Foo = lazy(() => import('./Foo'))` expects the module to have a `default` export. Named exports require a wrapper:

```js
// Module has only named export
const { FooComponent } = await import('./Foo');

// Wrapper to satisfy React.lazy's expectation
const Foo = lazy(() => import('./Foo').then(m => ({ default: m.FooComponent })));
```

---

## Interview Questions

**Q (High): How does `React.lazy` work under the hood, and what does the bundler do when it sees `import('./Checkout')`?**
Answer: `React.lazy` accepts a function that returns a Promise resolving to a module with a `default` export. When React renders a `lazy` component for the first time, it calls the function, gets the Promise, and throws it. The nearest `Suspense` boundary catches the thrown Promise (this is how Suspense works — it catches Promises thrown during render), shows the fallback, and re-renders the subtree when the Promise resolves. The bundler (Webpack, Vite) detects `import('./Checkout')` at build time and emits a separate chunk file containing Checkout and its dependencies. The chunk filename is content-hashed. The dynamic import at runtime becomes a fetch to that chunk URL + module evaluation. The result is cached in the module registry, so subsequent calls resolve instantly without a network request.
The trap: Not knowing that Suspense works by catching thrown Promises, or thinking `React.lazy` does the chunk fetching itself instead of delegating to `import()`.

**Q (High): A deploy causes a chunk-loading failure for users who have the app open. How do you handle this?**
Answer: Content-hashed chunk filenames change with every deploy. A user who loaded the app before the deploy has the old HTML, which references old chunk URLs. When they navigate to a new route (triggering a dynamic import), the browser requests the old chunk URL — which no longer exists on the server, returning a 404. `React.lazy`'s Promise rejects, and without an ErrorBoundary, the entire app crashes. Solutions: (1) Wrap every lazy-loaded route in an ErrorBoundary that catches chunk-load errors and calls `window.location.reload()` — the reload fetches the new HTML with the correct new chunk URLs. (2) Configure your CDN or server to retain old chunk files for a grace period (e.g., 30 minutes) so in-flight users can still load them. (3) Detect the error type (`error.name === 'ChunkLoadError'`) and show a "New version available — refresh" banner instead of a hard reload.
The trap: Thinking the problem doesn't happen because you use content hashing. Content hashing is what *causes* the problem by changing chunk URLs on every deploy.

**Q (High): What is the difference between `webpackPrefetch`, `webpackPreload`, and calling `import()` manually on hover?**
Answer: `webpackPrefetch: true` causes Webpack to emit a `<link rel="prefetch">` for the chunk in the parent chunk's output. The browser fetches it during idle time after the current page loads — this is a browser-managed, lowest-priority download. `webpackPreload: true` emits `<link rel="preload">` — the browser fetches it at high priority during the current navigation, for chunks needed soon on the current page. Manually calling `import('./Checkout')` on a hover event starts the fetch immediately in JavaScript, using the module system's fetch. The browser-managed prefetch can be cancelled under memory pressure; the manual import cannot. Manual hover-based imports are more reliable for "fetch on hover so click feels instant" patterns. `webpackPrefetch` is better for "load this while the user is idle" because it integrates with the browser's idle-time scheduling and respects network conditions.
The trap: Thinking `webpackPreload` and `webpackPrefetch` do the same thing at different times.

**Q (Medium): Why do vendor chunks improve caching, and what undermines this benefit?**
Answer: Vendor chunks contain third-party library code that rarely changes. By splitting them separately from application code, their content hash stays stable across deploys that only change app code. A user who visited yesterday has the vendor chunk in their browser cache; after a deploy, they only re-download the app chunks (which have new hashes), not the vendor chunk. The benefit is undermined when: (1) You put too many libraries in one chunk — updating any single library changes the hash for the entire vendor chunk, forcing re-download of everything. (2) You leave transitive dependencies that frequently update in the vendor chunk. (3) Your vendor chunk config is wrong and app code leaks into it. Best practice: separate React from other vendors (React is very stable), create a separate chunk for frequently-updated UI libraries, and keep utility functions in app chunks.
The trap: Thinking one big `vendor` chunk is always better than split vendor chunks.

**Q (Medium): `React.lazy` only supports default exports. How do you code-split a named export?**
Answer: Wrap the named import in an adapter that satisfies `React.lazy`'s expectation of `{ default: Component }`:
```js
const Foo = lazy(() =>
  import('./Foo').then(module => ({ default: module.FooComponent }))
);
```
The `.then()` transforms the module object before it reaches `React.lazy`. This is necessary because `React.lazy` reads `module.default` from the resolved module. An alternative pattern is to re-export the component as default in a thin wrapper file, but the `.then()` approach avoids creating an extra file.
The trap: Not knowing the limitation exists, or trying to pass a named-import wrapper without the `.then()`.

---

## Self-Assessment

Before moving on, check off each item you can do WITHOUT looking at the file.

- [ ] Explain what the bundler does when it encounters `import('./Checkout')` and how that relates to what's loaded in the browser
- [ ] Write a route-based code-split React app with `React.lazy`, `Suspense`, and an `ErrorBoundary`
- [ ] Explain why content-hashed chunk filenames cause chunk-load failures after deploys and describe two fixes
- [ ] Describe vendor chunk splitting — what it achieves, how to configure it in Vite, and what undermines the caching benefit
- [ ] Explain the difference between `webpackPrefetch` and `webpackPreload` and write a hover-based manual prefetch
- [ ] Wrap a named export in a `React.lazy`-compatible adapter

---
*Next: List Virtualization & Windowing — code splitting reduces the JS payload; virtualization reduces the DOM payload when rendering large lists.*
