# Tree Shaking, Scope Hoisting & Chunk Splitting

## Quick Reference

| Technique | What it removes/does | Requires |
|---|---|---|
| Tree shaking | Unused exports from the bundle | ESM static imports, no side effects |
| Scope hoisting | Module wrapper functions | ESM, no circular deps, single usage |
| Route-based splitting | Separate chunks per route | Dynamic `import()` |
| Vendor splitting | Separate chunk for node_modules | `splitChunks` or `manualChunks` |
| Shared chunk splitting | Common code extracted to one chunk | Multiple chunks importing same module |

---

## What Is This?

Three related but distinct optimizations that reduce bundle size and improve load time:

- **Tree shaking** — static analysis that removes exports nobody imports.
- **Scope hoisting** — concatenating ES modules into a single scope, eliminating per-module function wrappers.
- **Chunk splitting** — dividing the bundle into multiple files so browsers can load only what's needed and cache the rest independently.

> **Check yourself:** If you install lodash (CJS) vs. lodash-es (ESM) and import one function, which one tree-shakes?

---

## Why It Matters

A 500KB JS bundle parsed and executed on a mobile device costs roughly 3–5 seconds on slow hardware. Eliminating dead code and splitting intelligently is often the highest-leverage optimization available. These techniques are mechanical — get them right and the benefit is automatic.

---

## Tree Shaking

### How it works

Tree shaking is a form of dead code elimination specific to module boundaries. The bundler builds a module graph and marks every export as "used" or "unused" by tracing which imports are referenced. Unused exports are not included in the output.

```js
// math.js
export const add = (a, b) => a + b;
export const subtract = (a, b) => a - b;  // ← never imported anywhere
export const multiply = (a, b) => a * b;  // ← never imported anywhere

// main.js
import { add } from './math.js';
console.log(add(1, 2));

// Bundle output (after tree shaking):
const add = (a, b) => a + b;
console.log(add(1, 2));
// subtract and multiply are gone
```

### Requirements for tree shaking

1. **ESM `import`/`export`** — the static shape is required for analysis. CJS `require()` cannot be tree-shaken.
2. **No side effects** — a module with side effects must be included even if its exports aren't used.
3. **`sideEffects` flag in `package.json`** — tells the bundler the package is safe to tree-shake.

```json
// package.json — tell bundlers the whole package has no side effects
{
  "sideEffects": false
}

// Or exempt specific files (CSS imports, polyfill imports):
{
  "sideEffects": ["*.css", "./src/polyfills.js"]
}
```

### Side effects — the main footgun

A module "has side effects" if importing it changes global state, even if you don't use any of its exports:

```js
// side-effect-module.js
Array.prototype.flatten = function() { /* polyfill */ };
window.GlobalThing = new Thing();
console.log('I run at import time');

// Even with no named imports used, this module must be included
import './side-effect-module.js';
```

If `sideEffects: false` is set but the module actually has side effects, the bundler may eliminate it and break the app. Be accurate.

### Common tree-shaking footguns

**1. Importing from a barrel file**

```js
// index.js (barrel)
export * from './ComponentA';
export * from './ComponentB';
export * from './ComponentC';

// Importing from the barrel — works with tree shaking IF all re-exports are ESM
import { ComponentA } from './components';

// BUT if any re-exported module has side effects, the bundler may include all of them
// Better for large libraries: import directly from the source file
import { ComponentA } from './components/ComponentA';
```

**2. CJS packages bypass tree shaking**

```js
// lodash is CJS — imports the whole library
import { debounce } from 'lodash';       // ≈ 70KB

// lodash-es is ESM — tree-shakeable
import { debounce } from 'lodash-es';   // ≈ 2KB after tree shaking
```

**3. Class methods are not tree-shaken**

```js
class Utils {
  static format(x) { /* ... */ }
  static parse(x) { /* ... */ }
}

// Even if you only use Utils.format, parse is included — class is atomic
// Prefer named function exports over classes for utility code
export function format(x) { /* ... */ }
export function parse(x) { /* ... */ }
```

**4. Re-assigning exports disables tree shaking**

```js
// Violates the static shape requirement
export let config = {};
// Later:
config = require('./dynamic-config'); // bundler sees dynamic shape, can't tree-shake
```

---

## Scope Hoisting (Module Concatenation)

### What it eliminates

Without scope hoisting, each module is wrapped in a function to isolate its variables:

```js
// Webpack output WITHOUT scope hoisting:
/***/ (function(module, __webpack_exports__, __webpack_require__) {
  "use strict";
  __webpack_require__.r(__webpack_exports__);
  const add = (a, b) => a + b;
  __webpack_exports__["add"] = add;
/***/ }),

/***/ (function(module, __webpack_exports__, __webpack_require__) {
  "use strict";
  var _math = __webpack_require__(1);
  console.log((0, _math.add)(1, 2));
/***/ })
```

With scope hoisting (module concatenation), modules that can be safely merged are inlined:

```js
// Webpack output WITH scope hoisting:
const add = (a, b) => a + b;
console.log(add(1, 2));
```

Benefits: smaller output (no wrapper functions), faster runtime (one fewer scope chain lookup per function), better compression (minifiers work better on flattened code).

### When scope hoisting doesn't apply

- Circular dependencies (can't merge without changing evaluation order)
- A module imported by multiple chunks (can't duplicate it; it needs its own boundary)
- CJS modules (not statically analyzable)
- Dynamic `import()` split points (by definition a chunk boundary)

---

## Chunk Splitting

Chunk splitting divides the bundle into multiple files. The browser can load only the chunks it needs for the current page and cache other chunks for later.

### Why it matters

```
Single bundle approach:
  User visits home page
  → loads 2MB bundle.js (entire app)
  → includes code for admin dashboard, settings page, user profile, etc.
  → 80% of the code was unnecessary for this page load

Chunked approach:
  User visits home page
  → loads 100KB main.js + 200KB vendor.js
  → Admin chunk, settings chunk not loaded
  → Much faster initial load
```

### Types of chunk splitting

#### 1. Route-based splitting (dynamic import)

```js
// React Router with lazy loading — each route is a separate chunk
import { lazy, Suspense } from 'react';

const Dashboard = lazy(() => import('./pages/Dashboard'));
const Settings = lazy(() => import('./pages/Settings'));
const UserProfile = lazy(() => import('./pages/UserProfile'));

function App() {
  return (
    <Suspense fallback={<Loading />}>
      <Routes>
        <Route path="/dashboard" element={<Dashboard />} />
        <Route path="/settings" element={<Settings />} />
        <Route path="/profile" element={<UserProfile />} />
      </Routes>
    </Suspense>
  );
}
```

Each `import()` becomes a split point. The bundler emits separate chunks. The browser downloads a route's chunk only when navigating to it.

#### 2. Vendor chunk splitting

```js
// vite.config.ts
export default defineConfig({
  build: {
    rollupOptions: {
      output: {
        manualChunks: {
          // Group these into a stable vendor chunk
          react: ['react', 'react-dom', 'react-router-dom'],
          charts: ['recharts', 'd3'],
          forms: ['react-hook-form', 'zod'],
        },
      },
    },
  },
});
```

**Why vendor splitting matters for caching:** Your app code changes every deploy. `react` almost never changes version. If they're in one bundle, every deploy busts the cache for both. Separated, `react` can be cached across deploys indefinitely.

#### 3. Webpack `splitChunks`

```js
// webpack.config.js
optimization: {
  splitChunks: {
    chunks: 'all',           // split both async and sync chunks
    minSize: 20000,          // minimum chunk size in bytes
    maxAsyncRequests: 30,    // max parallel requests for async chunks
    maxInitialRequests: 30,  // max parallel requests for initial page load
    cacheGroups: {
      // Separate vendor chunk
      vendor: {
        test: /[\\/]node_modules[\\/]/,
        name: 'vendors',
        priority: -10,
        reuseExistingChunk: true,
      },
      // Separate commonly-shared chunk
      common: {
        name: 'common',
        minChunks: 2,      // must be used by at least 2 chunks
        priority: -20,
        reuseExistingChunk: true,
      },
    },
  },
  runtimeChunk: 'single',   // extract webpack runtime into its own small chunk
}
```

`runtimeChunk: 'single'` extracts webpack's module registry code into a separate tiny file so that changing one module doesn't change the hash of all other chunks.

#### 4. Content hashing for cache busting

```js
// webpack.config.js
output: {
  filename: '[name].[contenthash:8].js',   // main.a1b2c3d4.js
  chunkFilename: '[name].[contenthash:8].chunk.js',
}
```

`[contenthash]` changes only when the file's content changes. Unmodified chunks keep their hash across deploys — long-lived browser cache applies.

---

## Prefetching Split Chunks

Split chunks that aren't loaded immediately can be prefetched:

```js
// Webpack magic comment — prefetch
const Dashboard = lazy(() => import(/* webpackPrefetch: true */ './Dashboard'));

// Vite — rel=prefetch via plugin or manual
// or use <link rel="prefetch"> in the HTML
```

Prefetch: browser downloads the chunk in idle time, before the user navigates. When navigation happens, the chunk is already in cache. Useful for the next likely route.

Preload: browser downloads with high priority, alongside the current page load. For chunks needed very soon (e.g., above-fold content that's code-split).

---

## Analyzing the Bundle

```bash
# Webpack bundle analysis
npx webpack-bundle-analyzer dist/stats.json

# Vite built-in
npx vite-bundle-visualizer

# Source map explorer
npx source-map-explorer dist/main.*.js
```

The analyzer shows:
- Which modules are largest
- Which packages are duplicated (two versions of the same library)
- Whether tree shaking worked (expected-to-be-removed code still present)

---

## Gotchas

**1. `babel-plugin-transform-modules-commonjs` defeats tree shaking**
If Babel transforms your ESM to CJS before the bundler sees it, the bundler can't tree-shake because it receives CJS. Ensure your Babel config preserves ESM (set `modules: false` in `@babel/preset-env` when using webpack or Rollup).

```json
// .babelrc — let the bundler handle modules, not Babel
{
  "presets": [
    ["@babel/preset-env", { "modules": false }]
  ]
}
```

**2. Dynamic import `()` prevents tree shaking of the loaded module**
A dynamically imported module's exact exports are unknown at parse time, so the entire module must be included in the chunk.

**3. Over-splitting is a real problem**
Splitting into 100 tiny chunks means 100 network requests. HTTP/2 mitigates this (multiplexed) but there's still overhead. Aim for chunks between 50KB–150KB gzipped. Use `minSize` and `maxInitialRequests` to prevent over-granulation.

**4. `@mui/material` without tree shaking**
MUI's default import pattern requires proper tree shaking:
```js
// This works with tree shaking in MUI v5+:
import Button from '@mui/material/Button'; // direct file import — always works
import { Button } from '@mui/material';    // tree-shaken in v5 with ESM
```

---

## Interview Questions

**Q (High): What is tree shaking, and what are the two requirements for it to work?**

Answer: Tree shaking is static analysis that removes exports from the bundle that are never imported anywhere in the dependency graph. It treats the module graph like a tree and "shakes" unused branches off. The two requirements: (1) **ESM `import`/`export`** — the imports must be static declarations at the top level so the bundler can build the complete dependency graph at parse time without executing code. CommonJS `require()` is dynamic and can't be analyzed statically. (2) **No side effects** — if importing a module changes global state, the module must be included even if none of its named exports are used. The `sideEffects: false` field in `package.json` signals to bundlers that a package is safe to tree-shake; inaccurate marking causes either missing code (false `false`) or missed shaking (false `true`).

**Q (High): Why does splitting vendor code into a separate chunk improve caching, and what makes a good split boundary?**

Answer: Your application code changes on every deploy — the content hash changes, and users must re-download it. Third-party packages like React, Lodash, or D3 change rarely — typically only on explicit version upgrades. If they share a chunk with application code, every deploy invalidates the cache for the entire chunk, including the stable vendor code. Splitting them into a separate chunk means the vendor chunk can be cached for months with a long-lived `Cache-Control: max-age` header, while only the application chunk needs to be refetched. A good split boundary is one where the two sides have very different change frequencies. Routes are natural split boundaries (change together, loaded together). React + ReactDOM are a natural vendor boundary (change rarely, needed everywhere). Over-splitting (one chunk per component) creates excessive HTTP requests — aim for chunks large enough to justify a separate request.

**Q (Medium): What is scope hoisting and what are the conditions that prevent it?**

Answer: Scope hoisting (webpack calls it module concatenation) inlines the content of multiple ES modules into a single scope in the output, eliminating the wrapper functions that normally isolate each module's variables. Without it, each module's code is wrapped in a function that creates a closure — adding call overhead, extra bytes, and making minifiers less effective. With it, small modules that form a linear chain are concatenated into one flat scope. Conditions that prevent scope hoisting for a module: (1) circular dependencies between modules — merging would change evaluation order; (2) the module is imported by multiple chunks — it can't be inlined without duplication; (3) CJS module — not statically analyzable; (4) dynamic `import()` split point — the chunk boundary is intentional.

**Q (Medium): What is `runtimeChunk: 'single'` in webpack and why should you enable it?**

Answer: Webpack's runtime is a small piece of bootstrap code that maintains the module registry — a map of module IDs to their factory functions. By default, this runtime is included in the entry chunk. Every time any module changes, the module IDs can shift, which changes the runtime, which changes the hash of the entry chunk. If your entry chunk (`main.js`) and your vendor chunk both include portions of the runtime, changing app code causes vendor chunk hashes to change — breaking their long-lived cache unnecessarily. `runtimeChunk: 'single'` extracts the runtime into its own tiny chunk (`runtime.abc123.js`). Now only the runtime and the actual changed modules have new hashes. Vendor chunks and unchanged app chunks keep their hashes across deploys.

**Q (Low): How does importing from a barrel file (`index.js` that re-exports everything) affect tree shaking?**

Answer: Barrel files can work fine with tree shaking if all re-exported modules are pure ESM with `sideEffects: false`. The bundler traces through the barrel's re-exports and only includes what's actually used. However, barrels commonly break tree shaking when: (1) any re-exported module has side effects — the bundler must include it regardless of whether its exports are used; (2) the re-exports use CJS internally (common in legacy codebases); (3) the barrel file itself has side effects (imports that run code). The safest approach for large libraries is direct file imports (`import { Button } from './Button'` instead of `import { Button } from './components'`) — this avoids the barrel entirely and guarantees only the imported file is processed. Modern bundlers (Vite, webpack 5) handle pure-ESM barrels correctly, so direct import optimization is mainly important when working with CJS packages or complex mixed-format codebases.

---

## Self-Assessment

- [ ] Explain why CJS packages don't tree-shake and how to find an ESM alternative
- [ ] Describe what `sideEffects: false` does and how to mark file-level exceptions
- [ ] Explain scope hoisting and name two conditions that prevent it
- [ ] Implement a Vite `manualChunks` config that splits React and charting libraries
- [ ] Explain why `runtimeChunk: 'single'` improves long-term caching in webpack

---
*Next: Source Maps Strategy — types, security implications, and choosing between dev and prod configurations.*
