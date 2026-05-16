# Tree Shaking & Dead Code Elimination

## Quick Reference

| Term | Definition | Performed by |
|---|---|---|
| Tree shaking | Remove unused ES module exports from the bundle | Bundler (Rollup, Vite, Webpack 2+) |
| Dead code elimination (DCE) | Remove unreachable code paths | Minifier (Terser, esbuild, SWC) |
| Side-effect-free marking | Tell the bundler a module has no side effects and can be safely removed if unused | `package.json "sideEffects"` |
| `/*#__PURE__*/` | Annotation telling the minifier a function call has no side effects | Developer/library author |

---

## What Is This?

Tree shaking is the process by which a bundler statically analyzes ES module import/export statements, determines which exports are actually used by the application, and excludes unused exports from the final bundle. The name comes from the idea of shaking a dependency tree: live code that is used stays on the tree; dead code that is not used falls off.

Dead code elimination (DCE) is a broader term: the minifier's pass that removes unreachable code — `if (false) { ... }`, unused variables, functions that are never called — after the bundler has resolved the module graph.

These two processes work together: tree shaking removes entire unused module exports from the graph; DCE removes unreachable code within the modules that remain.

> **Check yourself:** If you import a module with `import _ from 'lodash'` versus `import { pick } from 'lodash'`, will tree shaking remove the unused lodash functions in both cases? Why or why not?

---

## Why Does It Exist?

Modern JavaScript libraries are large. `lodash` is ~70 KB gzipped in its CommonJS form. `date-fns` is ~200 KB. `@mui/material` ships hundreds of components. If your application uses 5 lodash utility functions and 3 MUI components, shipping the full library wastes most of the bundle budget on code that never executes.

Tree shaking solves this by treating modules as declaration graphs rather than runtime blobs. Because ES modules declare their dependencies statically (`import` statements at the top level, not inside `if` blocks or function calls), the bundler can trace exactly which exports are consumed and omit the rest.

---

## How It Works

### ES Modules Are Statically Analyzable

The fundamental requirement for tree shaking: **ES module syntax** (`import`/`export`). CommonJS modules (`require()`/`module.exports`) are evaluated at runtime and cannot be statically analyzed.

```js
// ES module — statically analyzable
import { pick, omit } from './utils';
// Bundler knows at build time: only 'pick' and 'omit' are used

// CommonJS — NOT statically analyzable
const utils = require('./utils');
const fn = utils[computedKey()]; // bundler cannot know which export is used
```

When a bundler encounters an ES module, it constructs a symbol table: which names are exported, and which names are imported by other modules. Any export with no incoming import edges in the whole application graph is "dead" and removed.

---

### The `sideEffects` Field

Tree shaking has a fundamental problem: a module may have side effects — code that runs when the module is imported, regardless of whether any exports are used. CSS imports, global polyfills, and registration calls are side effects.

```js
// This import has a side effect: it extends Array.prototype
import 'my-polyfill';

// This import runs code that registers a service worker
import './register-sw';
```

If the bundler removed these modules because nothing imports their exports, the side effects would be lost and the app would break.

By default, bundlers assume every module might have side effects and will not remove it even if its exports are unused. The `"sideEffects"` field in `package.json` explicitly tells the bundler which files are safe to remove if their exports go unused.

```json
// package.json — mark the entire package as side-effect-free
{
  "name": "my-utils",
  "sideEffects": false
}
```

```json
// Mark only CSS files and a specific module as having side effects
{
  "sideEffects": [
    "*.css",
    "./src/polyfills.js"
  ]
}
```

When a library author sets `"sideEffects": false`, the bundler can confidently remove any module in that package whose exports are not used, even if no error would occur at runtime from leaving it in. Without this field, even an imported utility file that exports 100 functions will be fully included if even one function is imported.

> **Check yourself:** You import `import { Button } from '@my-design-system'` and only use `Button`. The design system has `"sideEffects": false` in its `package.json`. Will the other 99 components be tree-shaken away? What if the field was missing?

---

### CommonJS Breaks Tree Shaking

The most common reason tree shaking fails is CommonJS modules — either your code, or a dependency.

```js
// utils.js — CommonJS
module.exports = {
  pick: function(obj, keys) { ... },
  omit: function(obj, keys) { ... },
  merge: function(...objs) { ... },
  // ... 97 more functions
};
```

```js
// app.js
const { pick } = require('./utils');
// Bundler sees require() — dynamic, cannot know which properties you'll use
// The entire utils.js is included in the bundle
```

Even the ES module syntax `import { pick } from './utils'` won't help if `./utils` uses `module.exports` internally — the module format determines analyzability.

**Solution for lodash:** Use `lodash-es` (the ES module edition):

```js
// Bad: ships all of lodash
import _ from 'lodash';
const result = _.pick(obj, ['a', 'b']);

// Good: ships only pick (if bundler + sideEffects: false configured)
import { pick } from 'lodash-es';
const result = pick(obj, ['a', 'b']);

// Also works: babel-plugin-lodash or lodash/pick path import
import pick from 'lodash/pick';  // imports only the pick module file
```

---

### `/*#__PURE__*/` Annotation

Some function calls have no side effects — a class declaration or a factory call that only creates a value without mutating global state. But the minifier doesn't know this and won't remove the call even if the result is unused.

The `/*#__PURE__*/` annotation tells the minifier: "This call is pure — if the result is not used anywhere, you can safely remove the entire call."

```js
// Without annotation — minifier keeps this even if Foo is never used
// (it might have a side effect inside the function call)
const Foo = createComponent({ ... });

// With annotation — minifier can remove if Foo is unused
const Foo = /*#__PURE__*/ createComponent({ ... });
```

This is especially important for class declarations with inheritance or HOCs (Higher-Order Components):

```js
// React HOC — without annotation, will never be tree-shaken
export const EnhancedButton = withTheme(Button);

// With annotation — can be removed if EnhancedButton is never imported
export const EnhancedButton = /*#__PURE__*/ withTheme(Button);
```

Library authors add `/*#__PURE__*/` to their build outputs. Application developers rarely need to add it manually — it is more relevant when authoring reusable libraries.

---

### Verifying Tree Shaking Works

**Step 1: Check the bundle output**

```bash
# Webpack — generate stats file and analyze
npx webpack --profile --json > stats.json
npx webpack-bundle-analyzer stats.json

# Vite with rollup-plugin-visualizer
npx vite build
# Open dist/stats.html
```

Look for unexpectedly large modules in the bundle that you only import partially.

**Step 2: Write a test import and check bundle size**

```js
// Test: import only one tiny function and check if the rest of the library is included
import { clamp } from 'lodash-es';
console.log(clamp(5, 0, 10));
```

Build, then inspect the output. If the entire lodash-es bundle appears in the output, tree shaking isn't working for that import.

**Step 3: Check for CJS interop**

If your bundler logs "module is not pure ESM" or similar, the imported package ships CJS only. Check if there's an `lodash-es`-style alternative or use per-file path imports.

---

### Webpack Configuration for Tree Shaking

Tree shaking in Webpack requires:

1. ES module source (your code and imported libraries)
2. Production mode (`mode: 'production'`) or explicitly setting `optimization.usedExports: true`
3. The `sideEffects` field in your `package.json` or the library's

```js
// webpack.config.js
module.exports = {
  mode: 'production',  // enables tree shaking + minification automatically
  optimization: {
    usedExports: true,  // marks unused exports in module scope
    minimize: true,     // runs Terser to actually remove dead code
    sideEffects: true,  // respects "sideEffects" in package.json
  },
};
```

In development mode, tree shaking is disabled by default to preserve readable source maps and faster builds.

---

### Vite / Rollup Tree Shaking

Vite uses Rollup for production builds. Rollup has excellent tree shaking with no extra configuration:

```js
// vite.config.js — tree shaking works by default in production builds
export default {
  build: {
    rollupOptions: {
      // treeshake: true is the default
    },
  },
};
```

Rollup's tree shaking is generally more aggressive than Webpack's because Rollup was designed for library bundling where tree shaking is critical. Vite inherits this advantage over Webpack for production builds.

---

## Gotchas

**Barrel files neutralize tree shaking.** A barrel file re-exports everything from a directory:

```js
// index.js — barrel file
export { Button } from './Button';
export { Modal } from './Modal';
export { Table } from './Table';
// ... 100 more exports
```

If any module in the barrel file has side effects (or lacks `"sideEffects": false`), importing *anything* from the barrel file pulls in the *entire* barrel. This is why many design systems like MUI historically had large bundle sizes — the barrel file pattern plus CJS dependencies.

```js
// May pull in the entire library if barrel file has side effects
import { Button } from '@my-ui';

// Direct import avoids the barrel file
import Button from '@my-ui/Button';
```

**Dynamic imports break tree shaking.** If you use `require()` or computed `import()` with a variable, the bundler cannot determine at build time which exports you need:

```js
// Cannot tree-shake — bundler must include all exports of the module
const module = await import(`./components/${componentName}`);
```

**Terser and `class` declarations.** Class declarations with static methods can sometimes prevent tree shaking because the bundler conservatively assumes the class constructor might have side effects. The `/*#__PURE__*/` annotation on class expressions helps.

**Unintentional re-exports in your own code.** If your `utils/index.js` re-exports everything from subdirectories and one subdirectory imports a global polyfill, that polyfill gets included whenever anyone imports anything from `utils/index.js` — even if they only use `utils/format.js`.

---

## Interview Questions

**Q (High): Why does tree shaking require ES modules and not work with CommonJS?**
Answer: Tree shaking depends on static analysis of the module graph — the bundler must know at build time exactly which exports are imported. ES module `import` and `export` statements are statically declared: they appear at the top level, cannot be conditional, and cannot use dynamic keys. The bundler can trace the complete symbol graph without executing any code. CommonJS `require()` is a function call evaluated at runtime. The module system doesn't know until runtime what `require()` returns or which properties you'll access on it — `const fn = utils[runtimeKey]` is perfectly valid CJS. Because the bundler cannot know at build time, it must include the entire `require()`d module in the bundle. Additionally, `module.exports` is a mutable object that can be modified at any point during module evaluation, making static analysis impossible.
The trap: Thinking "I use `import` syntax therefore tree shaking works." If the *library* uses CJS internally (even if you use `import` in your code), tree shaking fails for that library.

**Q (High): What is the `"sideEffects"` field in `package.json`, and what happens if a library omits it?**
Answer: `"sideEffects"` tells the bundler whether any modules in the package perform side effects when imported — mutations of global state, CSS injection, prototype extension — independent of their exports. `"sideEffects": false` means every module can be safely removed if none of its exports are used. An array value like `["*.css", "./polyfills.js"]` marks only specific files as having side effects. If the field is omitted, the bundler conservatively assumes all modules might have side effects and keeps every imported module fully in the bundle, even if only one export from it is used. This defeats tree shaking: a library with 100 utility functions that omits `"sideEffects": false` will ship all 100 functions even if you import only one.
The trap: Not distinguishing between a library omitting the field (bundler is conservative) vs. setting it to `false` (bundler can tree-shake aggressively).

**Q (High): You add `import { pick } from 'lodash'` to your app and the bundle analyzer shows the entire lodash library in the output. Why, and how do you fix it?**
Answer: Lodash's main package ships CommonJS modules. Even with ES module `import` syntax, when Webpack resolves the `lodash` package it gets the CJS edition — `require()` with `module.exports`. Webpack's interop layer wraps the CJS module but cannot statically analyze it, so all 300+ functions are included. Fixes: (1) Use `lodash-es`, which is the official ES module edition: `import { pick } from 'lodash-es'` — tree shaking will work. (2) Use direct path imports: `import pick from 'lodash/pick'` — each function is a separate file, so only the `pick` file is imported. (3) Use `babel-plugin-lodash` which transforms `import { pick } from 'lodash'` into direct file imports at build time. The root cause is that ES import syntax does not mean ES module format — what matters is the format of the *imported package*, not the syntax used to import it.
The trap: Thinking the problem is in your code rather than the library's module format.

**Q (Medium): What is a barrel file and how does it impact tree shaking?**
Answer: A barrel file is an `index.js` that re-exports everything from a directory, creating a convenient single import point. The problem is that if any file re-exported from the barrel has side effects, or if the barrel file itself lacks a `"sideEffects": false` annotation, importing anything from the barrel forces the bundler to evaluate and include every re-exported module. Even if you only import `Button`, the bundler must process `Modal`, `Table`, and all others to be safe. The fix: set `"sideEffects": false` in the package's `package.json`; use per-file direct imports instead of the barrel; or configure your bundler with a custom resolver that knows which barrel exports are side-effect-free. Many large UI libraries switched from barrel files to per-component imports precisely because of this issue.
The trap: Not realizing that `"sideEffects": false` applied to the package.json resolves the problem even with barrel files, as long as individual files truly have no side effects.

**Q (Medium): What does `/*#__PURE__*/` do and when is it used?**
Answer: It is a compiler hint to the minifier (Terser, esbuild, etc.) that the annotated function call is free of side effects. By default, the minifier won't remove any function call — it might log to the console, mutate global state, or trigger network requests. `/*#__PURE__*/` tells the minifier: "If the result of this call is never used, it is safe to delete the entire call." This enables dead code elimination to remove class factories, HOC wrappers, and other function calls whose results go unused. It is primarily used by library authors in their build outputs — e.g., React's JSX transform emits `/*#__PURE__*/ React.createElement(...)` so that unused component definitions can be eliminated. Application developers rarely add it manually unless they are writing code that compiles to other people's bundles.

**Q (Low): Tree shaking is disabled in development by default. Why, and does this affect your development workflow?**
Answer: In development, tree shaking and minification are disabled for two reasons: build speed and debuggability. Running Terser and performing dead code analysis significantly slows the build. Keeping all code also ensures that error messages, source maps, and variable names remain intact for debugging. This means your development bundle is much larger than production — sometimes 5–10x. It does not affect your development workflow beyond bundle size, since the dev server serves files over localhost. What it does mean is that you should run `npm run build` and inspect the production bundle with a bundle analyzer before shipping — dead code that "appears" to work locally may not be eliminated, but you should verify production output explicitly.

---

## Self-Assessment

Before moving on, check off each item you can do WITHOUT looking at the file.

- [ ] Explain why tree shaking requires ES modules and fails with CommonJS
- [ ] Describe what the `"sideEffects"` field in `package.json` does and the consequence of omitting it
- [ ] Diagnose why `import { pick } from 'lodash'` includes all of lodash in the bundle and give three fixes
- [ ] Explain what a barrel file is and how it can defeat tree shaking even with `"sideEffects": false` in the wrong place
- [ ] Describe what `/*#__PURE__*/` does and who is responsible for adding it
- [ ] Name the two Webpack `optimization` flags that enable tree shaking in production

---
*Next: Asset Optimization — tree shaking eliminates unused JavaScript; asset optimization tackles the images and fonts that often dominate network payloads.*
