# Polyfill Strategy: Differential Serving, browserslist, core-js

## Quick Reference

| Approach | What it ships | Modern browser cost | Legacy browser cost |
|---|---|---|---|
| Ship all polyfills always | Everything | Unnecessary KB | Covered |
| `useBuiltIns: 'usage'` (core-js) | Only used APIs | Only missing ones | Only missing ones |
| Differential serving | Two bundles, browser picks | Modern bundle, no polyfills | Legacy bundle, all polyfills |
| `<script type="module">` differential | Two bundles, HTML routing | Modern (module) | Legacy (nomodule) |
| polyfill.io CDN | Per-request UA detection | Minimal | Full |

---

## What Is This?

A polyfill is JavaScript code that implements a browser API or language feature for environments that don't natively support it. `Array.prototype.flat`, `Promise.allSettled`, `fetch`, `ResizeObserver` — all have polyfills for older browsers. The challenge is shipping polyfills to browsers that need them without penalizing browsers that don't.

```
Old approach (wrong):
  Load 50KB of polyfills unconditionally
  Chrome 120 user: downloads 50KB of code it never uses

Smart approach:
  Chrome 120 user: downloads 0KB of polyfills
  IE11 user:       downloads 50KB of polyfills they actually need
```

> **Check yourself:** Why does shipping polyfills to browsers that already support a feature waste more than just bandwidth?

---

## Why It Matters

Polyfills add parse and execution time, not just download size. A browser that natively supports `Array.flat` still has to parse and run a polyfill for it if you include it unconditionally. Getting polyfill strategy right is table stakes for performance-conscious frontend work.

---

## browserslist

`browserslist` is a shared configuration format that defines which browsers your project supports. All major build tools (`@babel/preset-env`, `postcss-preset-env`, `eslint-plugin-compat`, `stylelint-no-unsupported-browser-features`) read it.

```
# .browserslistrc (or "browserslist" key in package.json)
> 0.5%           # browsers with > 0.5% global usage
last 2 versions  # last 2 versions of every browser
not dead         # browsers that still receive security updates
not IE 11        # explicitly exclude IE11
```

Common presets:

```
# Conservative (broad compatibility):
> 0.5%, last 2 versions, Firefox ESR, not dead

# Modern (cutting costs on polyfills):
last 2 Chrome versions, last 2 Firefox versions, last 2 Safari versions, last 2 Edge versions

# Next.js default (via preset-env):
browserslist: "defaults"  →  "> 0.5%, last 2 versions, Firefox ESR, not dead"
```

Run `npx browserslist` to see which browsers your config matches. Run `npx browserslist --coverage` to see the % of global usage covered.

---

## Babel and core-js

`@babel/preset-env` transforms JS syntax (arrow functions, optional chaining) and injects polyfills for new APIs (`Promise`, `Array.flat`) based on your browserslist targets.

### `useBuiltIns: 'entry'` — entry-point injection

```js
// .babelrc
{
  "presets": [
    ["@babel/preset-env", {
      "targets": "> 0.5%, last 2 versions, not dead",
      "useBuiltIns": "entry",
      "corejs": 3
    }]
  ]
}

// Your entry point — single import for all needed polyfills:
import "core-js/stable";
import "regenerator-runtime/runtime";
```

Babel replaces `import "core-js/stable"` with specific imports for only the polyfills your target browsers need. Smaller than all of core-js, but still may include polyfills for APIs you don't use.

### `useBuiltIns: 'usage'` — per-usage injection (recommended)

```js
{
  "presets": [
    ["@babel/preset-env", {
      "targets": "> 0.5%, last 2 versions, not dead",
      "useBuiltIns": "usage",
      "corejs": 3
    }]
  ]
}
// No manual import needed — Babel injects polyfills at each usage site
```

Babel analyzes your code and injects polyfills only for APIs you actually use in your code, and only if the target browsers need them. This is the smallest output.

```js
// Your source:
const result = [1, [2, 3]].flat();

// Babel output (if target includes browsers without Array.flat):
import "core-js/modules/es.array.flat.js";
const result = [1, [2, 3]].flat();

// Babel output (if all targets support Array.flat):
const result = [1, [2, 3]].flat(); // no polyfill needed
```

### `corejs: 3` vs. `corejs: 2`

Always specify `corejs: 3`. core-js@2 is no longer maintained and missing many modern APIs. core-js@3 is actively maintained and covers through ES2024+.

---

## Differential Serving

Differential serving ships two bundles — a modern bundle (ES2017+, no transforms, minimal polyfills) and a legacy bundle (ES5, fully polyfilled) — and delivers the correct one to each browser.

### Why the split is significant

```
Modern bundle (targeting Chrome/Firefox/Safari):
  - Uses native async/await, arrow functions, optional chaining
  - No Babel transforms, no regenerator runtime
  - ~30% smaller than legacy bundle
  - Faster parse time (native syntax is optimized by JS engines)

Legacy bundle (targeting IE11, older mobile):
  - All syntax transformed to ES5
  - All polyfills included
  - Larger, slower — but necessary for those browsers
```

### Implementation: `<script type="module">` / `nomodule`

The browser's module loading attribute provides a native differential serving mechanism:

```html
<!-- Modern browsers: load this (type="module" is understood, nomodule is ignored) -->
<script type="module" src="bundle.modern.js"></script>

<!-- Legacy browsers: load this (nomodule is ignored because they don't understand it,
     module script is ignored because they don't know type="module")  -->
<!-- Wait — IE11 ignores nomodule and also ignores type="module".
     See gotcha #1 for the full picture. -->
<script nomodule src="bundle.legacy.js"></script>
```

A browser that understands `type="module"` downloads the module script and ignores `nomodule`. A browser that doesn't understand `type="module"` skips that script and downloads the `nomodule` script.

**What "module" browsers support (safe to target):** All ES2017+ features, async/await, most ES2019–2021 syntax, native `fetch`, `Promise`, `Map`, `Set`, `Symbol`.

### Setting up two builds in Vite

```js
// vite.config.ts — modern build
export default defineConfig({
  build: {
    target: 'es2017',    // modern syntax, no transforms
    outDir: 'dist/modern',
  },
});

// Legacy build via @vitejs/plugin-legacy
import legacy from '@vitejs/plugin-legacy';

export default defineConfig({
  plugins: [
    legacy({
      targets: ['defaults', 'IE 11'],  // browserslist for legacy
      additionalLegacyPolyfills: ['regenerator-runtime/runtime'],
    }),
  ],
});
// The plugin automatically injects module/nomodule script tags in HTML
```

### `@vitejs/plugin-legacy` output

```html
<!-- Generated by @vitejs/plugin-legacy -->
<script type="module" crossorigin src="/assets/index-Bx9Z8Kd.js"></script>
<script nomodule id="vite-legacy-polyfill" src="/assets/polyfills-legacy-Cz.js"></script>
<script nomodule id="vite-legacy-entry" data-src="/assets/index-legacy-D1x.js" ...></script>
```

---

## polyfill.io — CDN-based Differential Serving

polyfill.io analyzes the `User-Agent` header and returns only the polyfills the requesting browser needs:

```html
<!-- Request polyfills for just the APIs you use -->
<script src="https://polyfill.io/v3/polyfill.min.js?features=Array.prototype.flat,Promise.allSettled,fetch"></script>
```

Chrome 120 requesting this URL gets a nearly empty response (everything is natively supported). IE11 gets a full polyfill payload.

**Advantages:** No build configuration, serves tailored polyfills per browser, outsources maintenance.

**Disadvantages:** External dependency (if polyfill.io is down, polyfills don't load), third-party script, UA detection can be wrong for unusual browsers, adds a blocking `<script>` request.

**Self-hosted alternative:** Cloudflare acquired the polyfill.io service. For production applications, host the polyfill service yourself or use Fastly's mirror, rather than trusting a third-party CDN for critical polyfills.

---

## What Needs Polyfilling (and What Doesn't)

### Core-js covers

- New Array/String/Object/Promise methods (`flat`, `flatMap`, `allSettled`, `at`, `findLast`)
- New data structures (`Map`, `Set`, `WeakRef`, `FinalizationRegistry`)
- New Promise APIs (`any`, `allSettled`)
- Error classes (`AggregateError`)

### Core-js does NOT cover (separate polyfills needed)

- Web APIs: `fetch`, `IntersectionObserver`, `ResizeObserver`, `MutationObserver`
- DOM APIs: `Element.closest`, `Element.matches`
- CSS: CSS custom properties (not polyfillable in true IE11)
- `globalThis` (small separate polyfill)

```js
// Separate polyfills for web APIs:
import 'whatwg-fetch';              // fetch polyfill
import 'intersection-observer';    // IntersectionObserver polyfill
import 'resize-observer-polyfill'; // ResizeObserver polyfill
```

---

## Modern Targets — The Case for Cutting Legacy Support

The performance cost of supporting IE11 or older Safari:

| Feature | Modern bundle | Legacy bundle (IE11 target) |
|---|---|---|
| Async/await | Native | Regenerator runtime (+7KB) |
| Optional chaining | Native | Transform to nested checks |
| Bundle size | Baseline | +15–25% larger |
| Parse time | Faster (native syntax) | Slower (old engine model) |

Most analytics show IE11 < 0.5% usage for consumer web apps (enterprise apps differ). Setting `not IE 11` in your browserslist and dropping `nomodule` bundles yields a significant build simplification and bundle size reduction.

Check your actual analytics before making this decision.

---

## Gotchas

**1. Safari 10.1 and `nomodule` — the double-download bug**
Safari 10.1 understands `type="module"` but also downloads and executes `nomodule` scripts. Both bundles get executed. Fix: add a Safari 10.1 detection snippet or include a `defer` attribute on the `nomodule` script (which Safari 10.1 ignores for inline scripts but handles for src scripts). `@vitejs/plugin-legacy` handles this automatically.

**2. `useBuiltIns: 'usage'` and third-party packages**
By default, Babel's `usage` mode doesn't inject polyfills for APIs used in `node_modules`. If a package uses `Promise.allSettled` internally and your targets don't support it, you need either `entry` mode with explicit imports, or configure Babel to include `node_modules`.

**3. polyfills bundled multiple times**
With code splitting, if multiple chunks import core-js polyfills via `useBuiltIns: 'usage'`, the same polyfill may be included in multiple chunks. Fix: extract polyfills to a shared chunk via webpack's `splitChunks` `cacheGroups` or Rollup's `manualChunks`.

**4. `globalThis` vs. `window`**
Code that uses `globalThis` instead of `window` is more portable (works in workers, SSR) but `globalThis` needs a polyfill for older environments (Safari < 12.1). Add the tiny polyfill manually or rely on core-js@3.

**5. Polyfill for CSS features**
CSS polyfills are generally not possible in the same way JS polyfills are — you can't add `position: sticky` to an old browser with a JS shim that works as well as the native feature. For CSS, the strategy is progressive enhancement: ship a baseline that works without the feature, and the enhanced version for browsers that support it.

---

## Interview Questions

**Q (High): What is differential serving and how does the `type="module"` / `nomodule` technique implement it?**

Answer: Differential serving is the practice of building two separate bundles — a modern one (ES2017+, native syntax, minimal polyfills) and a legacy one (ES5, fully transpiled, all polyfills) — and delivering the right one to each browser. The `type="module"` / `nomodule` HTML attribute pair implements this natively: a browser that understands ES modules interprets `<script type="module">` normally and ignores `<script nomodule>`. A browser that doesn't understand ES modules ignores `<script type="module">` entirely and loads `<script nomodule>`. Since all browsers that understand ES modules also support async/await, arrow functions, and most ES2017+ features, the module bundle can skip all those transforms. Modern browsers get smaller, faster bundles; legacy browsers get the full polyfilled build.

**Q (High): What is the difference between `useBuiltIns: 'entry'` and `useBuiltIns: 'usage'` in Babel's `@babel/preset-env`?**

Answer: Both modes use your `browserslist` targets to determine which polyfills are needed, but they differ in scope. `entry` requires a manual `import "core-js/stable"` at your entry point and replaces that single import with specific imports for all polyfills your target browsers need — regardless of whether you actually use those APIs in your code. This is exhaustive but may include polyfills for APIs you never call. `usage` analyzes your source code and injects polyfill imports at each call site — only for APIs you actually use, and only if your targets don't support them natively. If you import `Array.flat` once in your code, Babel injects the polyfill once. If all your targets support `Array.flat` natively, no polyfill is injected. `usage` produces the smallest possible polyfill payload and is generally preferred.

**Q (Medium): What does browserslist do and why do multiple build tools all read from it?**

Answer: browserslist is a configuration format and query resolver that translates human-readable browser queries (`> 0.5%, last 2 versions, not dead`) into a concrete list of browser versions. The query is resolved against Can I Use data updated regularly. Multiple tools standardize on it because polyfill decisions (`@babel/preset-env`), CSS autoprefixing (`postcss-preset-env`), and compatibility linting (`eslint-plugin-compat`) all need the same answer to "which browsers am I targeting?" Sharing one config source keeps the answer consistent — you don't end up with Babel targeting IE11 while PostCSS assumes modern browsers, causing CSS features to be autoprefixed but JS polyfills to be omitted for IE11 users.

**Q (Medium): What Web APIs does core-js not polyfill, and how do you handle those?**

Answer: core-js covers JavaScript language features specified in ECMAScript — built-in methods and data structures like `Array.flat`, `Promise.allSettled`, `Map`, `Set`, and similar. It does not polyfill Web APIs defined by browser specifications outside ECMAScript: `fetch`, `IntersectionObserver`, `ResizeObserver`, `MutationObserver`, `CustomEvent`, `queueMicrotask`, etc. For these, install separate polyfill packages: `whatwg-fetch` for fetch, `intersection-observer` for IntersectionObserver, `resize-observer-polyfill` for ResizeObserver. These are imported manually at the entry point (since they add to `window`, not to prototype chains) and included in your build regardless of target — or conditionally added to a polyfill bundle in a differential serving setup.

**Q (Low): What is the Safari 10.1 `nomodule` double-download bug?**

Answer: Safari 10.1 was the first Safari to support ES modules (`type="module"`), but it has a bug: it downloads and executes `<script nomodule>` scripts even though it understands ES modules. This means users on Safari 10.1 execute both the modern bundle and the legacy bundle — doubling the execution cost and potentially running conflicting code twice. The fix is to include a small inline script that sets a flag variable before the nomodule script, or to use a `data-src` attribute pattern where the nomodule script's actual URL is injected by JavaScript only if the browser doesn't support modules. `@vitejs/plugin-legacy` handles this automatically with its generated script tags. Safari 10.1 is now a small enough user base that some teams accept the risk, but it's worth knowing in an interview.

---

## Self-Assessment

- [ ] Explain the difference between `useBuiltIns: 'entry'` and `useBuiltIns: 'usage'`
- [ ] Describe how `type="module"` / `nomodule` implements differential serving
- [ ] Explain what browserslist is and why multiple tools read the same config
- [ ] Name three Web APIs core-js doesn't polyfill and where to get those polyfills
- [ ] Describe the polyfill.io approach and its main production risk

---
*Next: Environment Config Management & Package Versioning at Scale — .env strategy, build-time injection, semver, and lockfiles.*
