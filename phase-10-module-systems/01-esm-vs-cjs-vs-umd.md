# ESM vs. CJS vs. UMD — Interop Challenges

## Quick Reference

| | ESM | CommonJS (CJS) | UMD |
|---|---|---|---|
| Syntax | `import`/`export` | `require()`/`module.exports` | Wraps AMD/CJS/global |
| Evaluation | Static, at parse time | Dynamic, at runtime |Dynamic, at runtime |
| Tree-shakeable | Yes (static shape) | No | No |
| Native browser support | Yes (`<script type="module">`) | No | No |
| Node.js support | Yes (`.mjs` or `"type":"module"`) | Yes (default) | Via bundler |
| Top-level `await` | Yes | No | No |

---

## What Is This?

JavaScript has had multiple module systems throughout its history because it was initially designed without one. Each system was invented to fill a gap:

- **CommonJS** — Node.js's synchronous `require()` system. Built in 2009 for server-side code where synchronous file I/O was acceptable.
- **UMD (Universal Module Definition)** — A wrapper pattern that detects the environment (AMD, CommonJS, or browser globals) and exports accordingly. Designed to ship one file that works everywhere, before ESM existed.
- **ESM (ES Modules)** — The language-native module system added in ES2015 (ES6). Static `import`/`export` at the top level, evaluated asynchronously.

```
Timeline:
  2009 — Node.js ships with CommonJS (require/module.exports)
  2011 — AMD (RequireJS) for browsers, UMD wrappers appear
  2015 — ESM specified in ES2015
  2017 — Node.js experiments with ESM support
  2020 — Node.js 12+ ships stable ESM
  2023 — All major runtimes and browsers support ESM natively
```

> **Check yourself:** Why is ESM tree-shakeable but CommonJS is not?

---

## Why It Matters

Most npm packages still ship CJS. Most modern apps write ESM. Bundlers paper over this mismatch, but the seams show in dual-package hazards, `__esModule` interop shims, `.mjs`/`.cjs` file extension confusion, and subtle bugs when mixing the two systems. Understanding the root difference prevents hours of debugging unexplained module behavior.

---

## ESM (ES Modules)

### Static analysis

ESM imports are **declarations**, not function calls. They must appear at the top level of a file — never inside `if` blocks, functions, or loops. This static structure lets bundlers build a complete dependency graph at parse time without executing code.

```js
// Valid — static, top-level
import { useState } from 'react';
import utils from './utils.js';

// Invalid — cannot be dynamic (use dynamic import() instead)
if (condition) {
  import { foo } from './foo.js'; // SyntaxError
}
```

### Live bindings

ESM exports are **live bindings** — if the exporting module changes a variable, importers see the updated value.

```js
// counter.js
export let count = 0;
export function increment() { count++; }

// main.js
import { count, increment } from './counter.js';
console.log(count); // 0
increment();
console.log(count); // 1 — live binding, not a copy
```

### Named vs. default exports

```js
// Named exports — multiple per file
export const add = (a, b) => a + b;
export const subtract = (a, b) => a - b;

// Default export — one per file
export default function multiply(a, b) { return a * b; }

// Importing
import multiply, { add, subtract } from './math.js';
import * as math from './math.js'; // namespace import
```

### Dynamic `import()`

For cases that genuinely need runtime module loading:

```js
// Returns a Promise — always async
const { default: lodash } = await import('lodash');

// Code splitting — bundlers treat this as a split point
const module = await import('./heavy-component.js');
```

### Browser native ESM

```html
<script type="module" src="./main.js"></script>
<!-- Modules are deferred by default — fire after DOMContentLoaded -->
<!-- Modules are only executed once even if referenced multiple times -->
<!-- Modules have their own scope — no global pollution -->
```

---

## CommonJS (CJS)

### Synchronous require

```js
// CJS — require is a function call, evaluated at runtime
const fs = require('fs');
const { join } = require('path');

// Conditionally loading a module — perfectly valid in CJS
let helper;
if (process.env.NODE_ENV === 'production') {
  helper = require('./prod-helper');
} else {
  helper = require('./dev-helper');
}

// module.exports can be anything
module.exports = { add, subtract };
module.exports = function() {}; // default-style export
```

### Why CJS isn't tree-shakeable

Because `require()` is a function call, its result can be computed at runtime. A bundler can't know at parse time which exports are used without executing the code:

```js
// These are all valid CJS — a static analyzer can't determine the shape
const mod = require(someVariable);               // dynamic module path
const { [key]: value } = require('./lib');       // computed property
if (condition) module.exports = require('./alt'); // conditional export
```

The bundler must include the entire CJS module because it can't prove statically which parts are used.

---

## UMD (Universal Module Definition)

UMD is a hand-written IIFE wrapper:

```js
(function (root, factory) {
  if (typeof define === 'function' && define.amd) {
    // AMD (RequireJS)
    define(['dependency'], factory);
  } else if (typeof module === 'object' && module.exports) {
    // CommonJS (Node.js)
    module.exports = factory(require('dependency'));
  } else {
    // Browser global
    root.MyLibrary = factory(root.Dependency);
  }
})(typeof self !== 'undefined' ? self : this, function (dependency) {
  // actual library code here
  return {};
});
```

UMD was the standard for library distribution before ESM. You still encounter it in legacy packages and CDN-served libraries (`<script src="cdn/library.umd.js">`).

---

## Interop Challenges

### 1. The `__esModule` flag

When Babel/TypeScript transpile ESM to CJS, they add a marker:

```js
// ESM source:
export default function foo() {}
export const bar = 1;

// Transpiled CJS output:
Object.defineProperty(exports, '__esModule', { value: true });
exports.default = function foo() {};
exports.bar = 1;
```

When another CJS file requires this:

```js
const mod = require('./module');
// mod = { __esModule: true, default: [Function], bar: 1 }

// Interop helper (what bundlers add):
const mod = require('./module');
const defaultExport = mod.__esModule ? mod.default : mod;
```

This works with transpiled code, but **native ESM default imports from true CJS modules** behave differently — the entire `module.exports` object becomes the default.

### 2. CJS `require()` of ESM — it's forbidden

Node.js does not allow synchronous `require()` of ESM files:

```
Error [ERR_REQUIRE_ESM]: require() of ES Module ./module.mjs not supported.
```

The reason: ESM supports top-level `await`, which means module evaluation can be asynchronous. `require()` is synchronous. There's no way to reconcile the two.

**Fix:** use dynamic `import()` (async) instead of `require()` to consume ESM from CJS:

```js
// CJS file wanting to use an ESM module
async function loadESM() {
  const { default: mod } = await import('./esm-module.js');
  return mod;
}
```

### 3. The dual-package hazard

When a package ships both ESM and CJS (dual-mode), `package.json` `exports` field routes each environment:

```json
{
  "name": "my-lib",
  "exports": {
    ".": {
      "import": "./dist/esm/index.js",
      "require": "./dist/cjs/index.js"
    }
  }
}
```

The hazard: if both the ESM and CJS version of the package are loaded in the same process (e.g., one file imports ESM, another requires CJS), you get **two separate instances of the module** — two separate singletons, two separate caches. A singleton pattern breaks completely.

```
App
├── feature-a.mjs  → import 'my-lib'  → loads ./dist/esm/index.js  (instance A)
└── legacy.cjs     → require('my-lib') → loads ./dist/cjs/index.js (instance B)

A and B are different objects — state is not shared between them.
```

**Mitigation:** library authors can write a single CJS version and export it as both, or use a stateless library design, or make the dual-package hazard documented behavior.

### 4. Named exports from CJS modules

Native ESM can import named exports from CJS only if Node.js can statically determine them (limited static analysis on `exports`). In practice, bundlers handle this at build time with CJS interop transforms.

```js
// Works in bundlers (Vite/webpack do static analysis):
import { debounce } from 'lodash'; // lodash is CJS

// In native Node.js ESM, this may fail for complex CJS modules:
// Node.js can only do named exports from CJS if it can statically analyze the module
// For many packages, you must use default import:
import lodash from 'lodash';
const { debounce } = lodash;
```

---

## File Extensions and `"type"` Field

```json
// package.json
{
  "type": "module"    // .js files are treated as ESM
  // "type": "commonjs" // .js files are treated as CJS (default)
}
```

| Extension | Meaning |
|---|---|
| `.mjs` | Always ESM, regardless of `"type"` |
| `.cjs` | Always CJS, regardless of `"type"` |
| `.js` | Determined by nearest `package.json` `"type"` field |

Use `.mjs`/`.cjs` when you need to mix formats within one package. Use `"type": "module"` and `.js` when the whole package is ESM.

---

## Interop Summary

```
CJS → CJS:        require() works, synchronous
CJS → ESM:        require() throws — use async import() instead
ESM → CJS:        import works — entire module.exports = default
ESM → ESM:        import works natively
Bundler context:  all combinations work — bundler handles interop transforms
```

---

## Gotchas

**1. `import.meta.url` doesn't exist in CJS**
`import.meta` is ESM-only. If your code uses it (`__dirname` equivalent, asset URLs), it can't run in CJS without transpilation. Replace with:
```js
// CJS equivalent of import.meta.url
const { pathToFileURL } = require('url');
const fileURL = pathToFileURL(__filename).href;
```

**2. Circular dependencies behave differently**
ESM handles circular imports via live bindings — a partially-initialized binding is visible. CJS returns the partially-populated `module.exports` at the point the cycle is hit. Both can lead to `undefined` values in subtle ways, but the mechanism differs.

**3. `.js` extension required in native ESM**
In Node.js native ESM (and browser ESM), relative imports must include the file extension:
```js
import { foo } from './foo';     // Error in Node.js ESM: file extension required
import { foo } from './foo.js';  // Correct
```
Bundlers (Vite, webpack) resolve extensions automatically, so this only matters in Node.js scripts.

**4. Tree shaking only works with ESM at the entry**
Even if all your application code is ESM, if the bundler entry is CJS, tree shaking may not fully work. The ESM entry must be the one analyzed.

---

## Interview Questions

**Q (High): Why is ESM tree-shakeable but CommonJS is not?**

Answer: ESM imports and exports are **static declarations** — they appear only at the top level of a module and their shape is fixed at parse time, before any code executes. A bundler can build a complete graph of what's imported and what's exported across the entire codebase without executing anything. Any export that nothing imports can be removed. CommonJS `require()` is a function call — it can appear anywhere (inside conditions, loops, closures), take dynamic arguments, and the result can be computed at runtime. A bundler cannot statically determine which exports will be accessed without executing the program. So the entire module must be included. This is why switching from CJS to ESM-authored packages (like lodash-es vs. lodash) can dramatically reduce bundle size.

**Q (High): What is the dual-package hazard and when does it occur?**

Answer: When an npm package ships both ESM (`"import"`) and CJS (`"require"`) variants via `package.json` exports conditions, and both variants are loaded in the same process, two separate module instances are created. Any mutable state (singletons, caches, event emitters, React context) is not shared between the two instances because they are entirely separate objects in memory. This occurs when one part of a codebase imports the ESM version (via `import`) and another part requires the CJS version (via `require()`). Library authors mitigate it by: (1) making the library stateless; (2) shipping only one format; or (3) documenting the hazard and making the ESM build re-export from the CJS singleton (so there's only one real instance).

**Q (High): Why can't you `require()` an ESM module synchronously in Node.js?**

Answer: `require()` is synchronous — it blocks the calling thread until the required module is loaded and evaluated. ESM supports top-level `await`, which means an ESM module's evaluation can be genuinely asynchronous. There is no way to synchronously wait for an asynchronous operation in JavaScript without blocking the event loop. Node.js cannot satisfy a synchronous `require()` contract for something that may be async. The solution is `import()`, which returns a Promise and is always asynchronous — it can wait for any top-level `await` in the loaded module to resolve before returning. As of Node.js 22, there is experimental support for synchronous `require()` of ESM modules that don't use top-level `await`.

**Q (Medium): What does the `__esModule` flag do and who adds it?**

Answer: When Babel or TypeScript compiles ESM source to CJS output, they add `Object.defineProperty(exports, '__esModule', { value: true })` to signal that this CJS output was originally ESM. When another compiled module imports from this file, the interop helper checks for `__esModule`: if `true`, it uses `exports.default` as the default export; if `false`, it uses the entire `exports` object as the default. This makes transpiled ESM-to-CJS interop work correctly for code compiled by the same toolchain. However, this is a convention — it's not enforced by the runtime. Native Node.js ESM ignores `__esModule` when doing `import` of a CJS file.

**Q (Low): What is the `.mjs` extension for and when would you use it?**

Answer: `.mjs` explicitly marks a file as ESM regardless of the `package.json` `"type"` field. It's useful when a package has `"type": "commonjs"` (or no `"type"`) but you want one specific file to be treated as ESM — for example, an ESM entry point in a dual-format package alongside CJS source. The complementary extension `.cjs` forces CJS treatment in a package with `"type": "module"`. The preference for larger packages is usually to set `"type": "module"` and use `.js` throughout, reserving `.cjs` for any legacy files. Mixed-extension codebases are harder to reason about, so the extensions are most valuable at package distribution boundaries, not inside application source.

---

## Self-Assessment

- [ ] Explain why ESM is statically analyzable and CJS is not
- [ ] Describe what happens when CJS tries to `require()` an ESM module
- [ ] Explain the dual-package hazard and name a library design that avoids it
- [ ] Describe what `__esModule` does and who produces it
- [ ] Explain when you'd use `.mjs` vs. setting `"type": "module"` in `package.json`

---
*Next: Bundlers — Webpack, Vite, Esbuild, Turbopack, Rollup — dev server strategy, HMR, and production output trade-offs.*
