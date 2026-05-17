# Bundlers: Webpack, Vite, Esbuild, Turbopack, Rollup

## Quick Reference

| Bundler | Language | Dev server strategy | HMR | Best for |
|---|---|---|---|---|
| Webpack | JavaScript | Bundle everything upfront | Module replacement | Large apps, maximum plugin ecosystem |
| Vite | JS + Go (esbuild) | Native ESM, unbundled | Native ESM hot reload | Modern apps, fast iteration |
| Esbuild | Go | N/A (library) | N/A | Build pipeline component, CLI transforms |
| Turbopack | Rust | Incremental, lazy | Incremental HMR | Next.js ecosystem |
| Rollup | JavaScript | Minimal (via plugins) | Plugin-based | Library authoring, clean ESM output |

---

## What Is This?

A bundler takes many source files (JS, TS, CSS, images) and combines them into fewer, optimized output files that browsers can load efficiently. Without bundling, loading a modern app would require hundreds or thousands of individual HTTP requests. Bundlers also transform code (TypeScript → JS, JSX → JS, newer syntax → compatible syntax) and apply optimizations like tree shaking and minification.

```
Source:                        Output:
src/
  index.tsx     ─────────────► bundle.js (minified, tree-shaken)
  components/                  bundle.css
    Button.tsx  ──┤             chunk-vendor.js (code split)
    Modal.tsx   ──┤
  utils/
    format.ts   ──┘
node_modules/
  react/        (tree-shaken)
  lodash-es/    (tree-shaken)
```

> **Check yourself:** What problem does a dev server bundling strategy solve that production bundling doesn't need to solve?

---

## Why It Matters

The bundler choice affects developer iteration speed (HMR speed, cold start time), production output quality (chunk strategy, tree shaking), and the build pipeline's extensibility. The shift from Webpack to Vite-era tooling represents a fundamental architectural change — not just a speed improvement.

---

## Webpack

### Architecture

Webpack builds a complete dependency graph by traversing all `require()`/`import` statements from the entry point. It processes every module through a **loader pipeline** (transforming individual files) and then applies **plugins** that operate on the full compilation.

```
Entry: src/index.js
    │
    ├─ webpack resolves imports recursively
    ├─ Each file passes through matching loaders:
    │     .tsx → babel-loader → .js
    │     .css → css-loader → style-loader
    │     .png → file-loader
    │
    ├─ Module graph is built
    ├─ Plugins run on the full compilation (HtmlWebpackPlugin, etc.)
    └─ Output: chunked, minified files
```

### Dev server

Webpack Dev Server bundles the entire application upfront, then serves from memory. As you edit files, webpack rebuilds the affected module and its dependents. HMR replaces the changed module without a full page reload.

**The cold start problem:** On a large app, the initial build takes 30–60+ seconds because webpack must process every file before serving anything. Fast Refresh improvements help, but the fundamental issue is eager compilation.

### Configuration

```js
// webpack.config.js
module.exports = {
  entry: './src/index.tsx',
  output: {
    path: path.resolve(__dirname, 'dist'),
    filename: '[name].[contenthash].js',
    clean: true,
  },
  module: {
    rules: [
      {
        test: /\.(ts|tsx)$/,
        use: 'babel-loader',
        exclude: /node_modules/,
      },
      {
        test: /\.css$/,
        use: ['style-loader', 'css-loader', 'postcss-loader'],
      },
    ],
  },
  optimization: {
    splitChunks: {
      chunks: 'all',
      cacheGroups: {
        vendor: {
          test: /[\\/]node_modules[\\/]/,
          name: 'vendors',
          chunks: 'all',
        },
      },
    },
    runtimeChunk: 'single',
  },
};
```

### When to choose Webpack

- Legacy codebase with existing webpack config and plugins
- Requires a plugin that only exists in the webpack ecosystem (custom loaders, complex build transforms)
- Module Federation for micro-frontends (webpack's Module Federation is still the most mature implementation)

---

## Vite

### Architecture

Vite fundamentally changes the dev server model by **not bundling during development**. It serves source files as native ESM directly to the browser. The browser's native module loading handles the dependency graph — each `import` triggers a separate request, and Vite transforms only the requested file on demand.

```
Dev mode — no bundling:
  Browser requests /src/main.tsx
      │
      Vite transforms it on-demand (esbuild: < 1ms)
      Returns native ESM
      │
  Browser sees 'import React from "react"'
      │
  Browser requests /node_modules/.vite/react.js (pre-bundled)
```

**Pre-bundling:** Vite pre-bundles `node_modules` with esbuild into a single ESM chunk. This avoids the browser making hundreds of requests for node_modules (e.g., lodash's 600 modules). Pre-bundled deps are cached aggressively.

### HMR

Because each module is its own request, HMR only invalidates and re-requests the exact changed module and its direct dependents. On large apps, Vite's HMR stays fast regardless of app size — it scales O(1) per change rather than O(graph size).

```js
// vite.config.ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  build: {
    rollupOptions: {
      output: {
        manualChunks: {
          react: ['react', 'react-dom'],
          router: ['react-router-dom'],
        },
      },
    },
  },
  server: {
    port: 3000,
    proxy: {
      '/api': 'http://localhost:8080',
    },
  },
});
```

### Production build

Vite uses **Rollup** for production builds (not esbuild). Rollup produces cleaner, better-optimized output — better tree shaking, scope hoisting, and chunk control. Esbuild is kept in the dev path for speed.

### When to choose Vite

- New projects without legacy constraints
- Teams where fast iteration (HMR speed, cold start) is a priority
- Apps that don't need webpack-specific plugins
- Projects using modern frameworks (React, Vue, Svelte)

---

## Esbuild

### Architecture

Esbuild is written in Go and runs natively — not through Node.js. It parallelizes parsing, printing, and linking across all CPU cores. Benchmarks consistently show 10–100x faster transforms than JS-based bundlers.

```
esbuild src/index.ts --bundle --outfile=dist/bundle.js --minify
```

Esbuild is primarily used as a **library/transform layer** inside other tools (Vite uses it for pre-bundling and TS/JSX transforms, Turbopack uses it conceptually, Remix uses it). Using esbuild directly as your bundler is uncommon for production apps because its plugin API is limited and it doesn't produce the same chunk optimization quality as Rollup.

```js
// Using esbuild directly (rare for app bundling)
import * as esbuild from 'esbuild';

await esbuild.build({
  entryPoints: ['src/index.ts'],
  bundle: true,
  minify: true,
  splitting: true, // ESM code splitting
  format: 'esm',
  outdir: 'dist',
  target: ['chrome90', 'firefox88'],
});
```

### When esbuild wins

- CI/CD type-stripping transforms (strip TypeScript annotations, not type-check)
- Build pipeline steps that don't need full bundler features (babel replacement)
- As the engine inside Vite/other tools where its speed is the value

---

## Turbopack

### Architecture

Turbopack (Vercel, built in Rust) is designed for the Next.js ecosystem. Its key innovation is **incremental computation** — it caches the result of every transform at a fine-grained level and only recomputes what changed. On a large Next.js app, it's significantly faster than webpack for HMR.

```
First build: compute and cache all transforms
Subsequent builds:
  File changes → only affected transforms re-run
  Unaffected modules: served from cache (sub-millisecond)
```

As of 2024, Turbopack's dev server is stable in Next.js (`next dev --turbo`). The production build with Turbopack is still in progress — Next.js production builds still use webpack.

### When to choose Turbopack

- Next.js projects experiencing slow webpack HMR on large codebases
- Teams willing to accept some ecosystem immaturity for faster dev iteration

---

## Rollup

### Architecture

Rollup's strength is **library output** — it produces the cleanest, most readable output of any bundler. It pioneered tree shaking and scope hoisting (now called module concatenation). Its output is often smaller than webpack's for library use cases because it doesn't wrap every module in a function.

```js
// rollup.config.js — for library publishing
import resolve from '@rollup/plugin-node-resolution';
import commonjs from '@rollup/plugin-commonjs';
import typescript from '@rollup/plugin-typescript';

export default [
  // ESM build
  {
    input: 'src/index.ts',
    output: { file: 'dist/index.esm.js', format: 'esm' },
    plugins: [resolve(), typescript()],
    external: ['react', 'react-dom'], // don't bundle peer deps
  },
  // CJS build
  {
    input: 'src/index.ts',
    output: { file: 'dist/index.cjs.js', format: 'cjs' },
    plugins: [resolve(), commonjs(), typescript()],
    external: ['react', 'react-dom'],
  },
];
```

### Scope hoisting

Rollup inlines all ES modules into a single scope (when bundling for non-splitting outputs):

```js
// Source: a.js + b.js as two separate modules

// Rollup output: one scope, no module wrapper overhead
const add = (a, b) => a + b;
const value = add(1, 2);
console.log(value);
```

### When to choose Rollup

- Publishing an npm library (component library, utility package)
- Need clean ESM output that other bundlers can tree-shake
- Shipping a dual CJS+ESM package

---

## HMR Deep Dive

Hot Module Replacement replaces changed modules without a full page reload. The implementation differs by bundler:

```js
// Webpack HMR API (low-level — usually abstracted by framework plugins)
if (module.hot) {
  module.hot.accept('./Counter', () => {
    // Module updated — re-render
    const NewCounter = require('./Counter').default;
    ReactDOM.render(<NewCounter />, document.getElementById('root'));
  });
}

// Vite HMR API (more explicit, rarely needed — handled by @vitejs/plugin-react)
if (import.meta.hot) {
  import.meta.hot.accept((newModule) => {
    // Called with the updated module
  });
  import.meta.hot.dispose(() => {
    // Clean up side effects of the old module version
  });
}
```

**React Fast Refresh** (used by both Webpack and Vite via plugins) preserves component state across HMR updates when only function body changes. If component signature changes, state is reset.

---

## Production vs. Dev Build Differences

| Concern | Dev | Prod |
|---|---|---|
| Minification | No | Yes (Terser/esbuild) |
| Source maps | Full (eval-source-map) | External file or none |
| Tree shaking | Usually off for speed | On |
| Code splitting | Minimal | Optimized chunks |
| Environment vars | `NODE_ENV=development` | `NODE_ENV=production` |
| Console logs | Kept | Optionally stripped |

```js
// Vite production build
vite build

// Analyze output (vite-bundle-visualizer)
npx vite-bundle-visualizer

// Webpack bundle analysis
npx webpack-bundle-analyzer stats.json
```

---

## Gotchas

**1. Vite uses Rollup in production — different behavior from dev**
Features that work in Vite's esbuild-powered dev server may behave differently in Rollup-powered production builds. Always test production builds, not just dev.

**2. CJS packages in Vite pre-bundling**
Vite pre-bundles CJS packages to ESM. If a CJS package has side effects or uses `__filename`/`__dirname` in surprising ways, pre-bundling can break it. The fix is `optimizeDeps.exclude: ['problem-package']` in `vite.config.ts`.

**3. Webpack `contenthash` vs. `chunkhash`**
`[chunkhash]` changes when the chunk changes. `[contenthash]` changes when the file content changes. For CSS extracted with `MiniCssExtractPlugin`, use `[contenthash]` — if you use `[chunkhash]`, updating JS causes CSS hash to change (cache invalidation) even though CSS didn't change.

**4. `externals` in Rollup vs. webpack**
In Rollup, `external: ['react']` prevents bundling React. In webpack, `externals: { react: 'React' }` means "when code imports React, use the global `window.React` instead." Different semantics — webpack's externals map to global variables; Rollup's are just excluded.

**5. Esbuild doesn't emit TypeScript errors**
Esbuild strips types without checking them. Running `tsc --noEmit` separately is required for type safety in esbuild-based pipelines. Vite's `vite build` also uses esbuild for transforms — run `tsc` separately in CI.

---

## Interview Questions

**Q (High): What is the fundamental architectural difference between Vite's dev server and Webpack's dev server?**

Answer: Webpack Dev Server bundles the entire application on startup — it processes every file in the dependency graph and holds the result in memory before serving anything. Cold start time grows with app size. Vite's dev server serves source files as native ESM without bundling. The browser's import system builds the dependency graph on demand — each `import` triggers a request, and Vite transforms only the file that was requested. Cold start is nearly instant regardless of app size because Vite only processes files the browser asks for. For HMR, webpack must determine which modules are affected by a change and rebuild those portions of the graph. Vite's HMR invalidates only the changed module and its direct importers, which stays fast as the app grows. The tradeoff is that Vite's dev behavior (many small requests, native ESM) differs from its production behavior (Rollup bundle), so production-only bugs are possible.

**Q (High): Why does Vite use Rollup for production builds but esbuild for dev transforms?**

Answer: Esbuild is orders of magnitude faster than any JS-based tool, making it ideal for dev transforms where speed is the primary concern. However, esbuild's code splitting, tree shaking, and chunk optimization are less sophisticated than Rollup's. Rollup produces smaller, better-structured production bundles with superior tree shaking (via its original scope hoisting design), finer-grained chunk control via `manualChunks`, and cleaner output. Using Rollup for production gives better bundle quality at the cost of build time — acceptable for production builds (which run in CI, not on every file save). This means developers must test production builds, not just the dev server, since the tools are different.

**Q (Medium): What is Module Federation in Webpack and when would you use it?**

Answer: Module Federation (webpack 5+) allows separately deployed webpack bundles to share modules at runtime. One "host" app can dynamically load components or modules from a "remote" app — including sharing dependencies like React so both apps use the same instance. This is the primary technical mechanism for micro-frontend architectures where independent teams ship their own bundles that compose into one UI. Use it when: multiple teams own separate parts of the UI, deployments need to be independent, and you want dependency sharing without duplicating large packages. The alternatives are iframes (full isolation, no sharing) and npm-package-based component libraries (shared at build time, not runtime). Module Federation's downsides are webpack lock-in, operational complexity, and version skew challenges when shared modules update.

**Q (Medium): What is scope hoisting and what bundler introduced it?**

Answer: Scope hoisting (introduced by Rollup, later added to Webpack as "module concatenation") inlines multiple ES module files into a single scope in the output, eliminating the wrapper functions that bundlers traditionally add around each module to isolate their scope. Without scope hoisting, each module is wrapped in a function that creates its own scope; the bundler calls these functions to execute modules. With scope hoisting, when modules form a chain with no circular dependencies and single usage, they're merged into one scope — fewer functions, fewer closures, smaller output, faster runtime. It only applies when the module graph allows it (static ESM, no side effects that require isolation). This is one reason Rollup output is typically smaller than Webpack output for library code.

**Q (Low): When would you choose esbuild as your primary bundler rather than using it as a tool within Vite or Turbopack?**

Answer: Rarely, for app bundling. Esbuild directly is appropriate for: (1) build pipeline scripts that need to transform TypeScript or JSX quickly without a full bundler's features; (2) CLI tools where build time must be minimal; (3) server-side code bundling (Lambda functions, edge workers) where the full Rollup/webpack optimization isn't worth the complexity; (4) internal tooling where you need maximum build speed and can accept esbuild's limited plugin ecosystem. For consumer-facing apps, esbuild's chunk optimization and plugin ecosystem aren't mature enough — use Vite (which uses esbuild internally) or webpack instead.

---

## Self-Assessment

- [ ] Explain why Vite's cold start is fast regardless of app size
- [ ] Describe why Vite uses Rollup for production and esbuild for dev
- [ ] Explain scope hoisting and which bundler invented it
- [ ] Describe Module Federation and a use case for it
- [ ] Explain the `contenthash` vs. `chunkhash` distinction and when each is correct

---
*Next: Tree Shaking, Scope Hoisting & Chunk Splitting — how dead code is eliminated and bundles are split at the module level.*
