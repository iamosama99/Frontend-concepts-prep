# Monorepo Shared Type Packages

## Quick Reference

| Concept | Detail |
|---|---|
| Composite projects | `"composite": true` enables incremental build and cross-project references |
| Project references | `references: [{ path: '../shared' }]` — explicit TypeScript project graph |
| Path mapping | `"paths": { "@myapp/*": ["packages/*/src"] }` — alias resolution |
| `skipLibCheck` | Skip type checking `.d.ts` files — speeds compilation, hides real errors |
| `declaration` | Emit `.d.ts` alongside `.js` — required for library packages |
| `declarationMap` | Emit `.d.ts.map` — enables go-to-definition to source (not compiled output) |

---

## What Is This?

In a TypeScript monorepo, multiple packages share code: a `shared` package with domain types, a `ui` package with components, an `api` package with server logic. Without explicit configuration, TypeScript doesn't know about these cross-package dependencies — it treats each package as isolated. Composite projects and project references are TypeScript's mechanism for making cross-package type checking work correctly and efficiently.

---

## The Problem Without Configuration

```
monorepo/
  packages/
    shared/   — defines User, Product types
    web/      — imports from shared
    server/   — imports from shared
```

Without TypeScript project references:
- `web` has to import from `shared/src/index.ts` — source file imports, not package imports
- Or `web` imports from `shared/dist/index.js` but TypeScript uses `.d.ts` that may be stale
- Changes in `shared` aren't reflected in `web` until a manual rebuild
- Full type checking across all packages is slow — TypeScript re-checks everything

---

## Composite Projects

Enabling composite mode in a package's `tsconfig.json` is the prerequisite for being referenced by other packages:

```json
// packages/shared/tsconfig.json
{
  "compilerOptions": {
    "composite": true,          // enables project references
    "declaration": true,        // emit .d.ts files — required for composite
    "declarationMap": true,     // .d.ts.map for go-to-source in IDE
    "outDir": "./dist",
    "rootDir": "./src"
  },
  "include": ["src/**/*"]
}
```

`composite: true` requires:
- `declaration: true` (the referenced project must emit declarations)
- `rootDir` must be set
- All files in `include`/`files` must be covered by `rootDir`

---

## Project References

A package declares which other TypeScript projects it depends on:

```json
// packages/web/tsconfig.json
{
  "compilerOptions": {
    "outDir": "./dist",
    "rootDir": "./src"
  },
  "references": [
    { "path": "../shared" },  // declares: web depends on shared
    { "path": "../ui" }
  ]
}
```

With this configuration:
- TypeScript knows to check `shared` first when building `web`
- If `shared`'s source hasn't changed since the last build, TypeScript uses cached `.d.ts` files — fast incremental builds
- `go to definition` in an IDE on a type from `shared` navigates to the source file (via `.d.ts.map`)

### Building with project references

```bash
# -b flag: build mode — builds referenced projects first
tsc -b packages/web
# or build everything from root:
tsc -b
```

TypeScript builds in dependency order: `shared` first, then `web` and `server` (which both depend on `shared`).

---

## Path Mapping

Path mapping creates import aliases so consumers don't need to write relative paths like `../../../shared/src/user`:

```json
// root tsconfig.json (often tsconfig.base.json):
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@myapp/shared": ["packages/shared/src/index.ts"],
      "@myapp/shared/*": ["packages/shared/src/*"],
      "@myapp/ui": ["packages/ui/src/index.ts"],
      "@myapp/ui/*": ["packages/ui/src/*"]
    }
  }
}
```

```ts
// In packages/web/src/app.ts — clean imports:
import { User } from '@myapp/shared';
import { Button } from '@myapp/ui';
```

**Important:** TypeScript's `paths` is for type checking only — it does not configure how the bundler (Vite, webpack, Node.js) resolves modules at runtime. You must configure the bundler separately:

```js
// vite.config.ts
import { defineConfig } from 'vite';
import { resolve } from 'path';

export default defineConfig({
  resolve: {
    alias: {
      '@myapp/shared': resolve(__dirname, 'packages/shared/src'),
      '@myapp/ui': resolve(__dirname, 'packages/ui/src'),
    },
  },
});
```

For Node.js: use `tsconfig-paths` or configure `exports` in `package.json`:

```json
// packages/shared/package.json
{
  "name": "@myapp/shared",
  "exports": {
    ".": {
      "import": "./dist/index.js",
      "types": "./dist/index.d.ts"
    }
  }
}
```

---

## The `skipLibCheck` Trade-off

`skipLibCheck: true` tells TypeScript not to type-check the content of `.d.ts` files — only the usage of the types they export.

**Why people enable it:** `.d.ts` files from `@types/*` and library packages can have type errors themselves (version conflicts, incomplete types). Without `skipLibCheck`, these errors appear in your error output even though you didn't write those files.

**The trade-off:** You might miss real errors. If your `shared` package emits broken `.d.ts` files (due to a TypeScript version mismatch or a type error in your declarations), `skipLibCheck` in a consuming package will hide those errors. You'll see runtime breakage instead.

**Recommendation:** Enable `skipLibCheck` for applications (they consume `.d.ts` but don't ship them). Consider disabling for libraries (they ship `.d.ts` and should validate them). For the monorepo's own packages, project references + composite mode is better than `skipLibCheck` — you get fast incremental checking with real validation.

---

## Practical Monorepo Structure

```
monorepo/
  tsconfig.base.json          ← shared compiler options
  tsconfig.json               ← root, references all packages
  packages/
    shared/
      src/
        types/user.ts
        types/product.ts
        index.ts
      tsconfig.json           ← composite: true
      package.json
    ui/
      src/
        components/Button.tsx
        index.ts
      tsconfig.json           ← references: [../shared]
      package.json
    web/
      src/
        App.tsx
      tsconfig.json           ← references: [../shared, ../ui]
      package.json
    server/
      src/
        routes/users.ts
      tsconfig.json           ← references: [../shared]
      package.json
```

```json
// tsconfig.base.json — shared options inherited by all packages
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "strict": true,
    "esModuleInterop": true
  }
}
```

```json
// packages/ui/tsconfig.json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "composite": true,
    "declaration": true,
    "declarationMap": true,
    "jsx": "react-jsx",
    "outDir": "./dist",
    "rootDir": "./src"
  },
  "references": [
    { "path": "../shared" }
  ]
}
```

---

## Common Issues

**Stale `.d.ts` files:** If you edit `shared/src/user.ts` but don't rebuild `shared`, `web` still sees the old types. `tsc -b --watch` runs incremental builds on change.

**Package `exports` vs. path mapping:** Path mapping works for TypeScript but the bundler or Node.js needs its own resolution. Align `tsconfig.paths` with bundler aliases or `package.json#exports`.

**`declarationMap` is optional but essential for DX:** Without it, "go to definition" in an IDE on a type from a referenced package jumps to the compiled `.d.ts` file — not the source TypeScript. With `declarationMap: true`, it jumps to the original `.ts` source.

**Circular references:** TypeScript will error on circular project references (A references B, B references A). Circular type references within a single project are allowed; circular project references are not.

---

## Interview Questions

**Q (High): What is the difference between TypeScript composite projects and path mapping, and why do you need both?**

Answer: They solve different problems. Composite projects (`"composite": true` + `"references"`) are TypeScript's cross-project dependency tracking mechanism — they tell TypeScript the build order, enable incremental compilation (using cached `.d.ts` files for unchanged projects), and allow `tsc -b` to build all packages in dependency order. Path mapping (`"paths"`) is a type-checking alias — it tells TypeScript how to resolve import strings like `@myapp/shared` to actual source files. You need both because: composite projects make cross-project type checking efficient and correct; path mapping makes import paths ergonomic. Path mapping alone doesn't give you build graph awareness or incremental compilation. Composite alone doesn't give you clean import aliases. Additionally, path mapping is TypeScript-only — the bundler needs its own alias configuration.

**Q (High): Why can't you rely on TypeScript `paths` configuration for runtime module resolution?**

Answer: TypeScript's `paths` is a compile-time feature that only affects how the TypeScript type checker resolves import strings to source files. TypeScript emits JavaScript (or delegates to a bundler), and the emitted JavaScript still contains the original import strings like `'@myapp/shared'`. At runtime (in Node.js) or at bundle time (in Vite/webpack), the runtime's module resolver doesn't read `tsconfig.json` — it uses its own resolution algorithm. If `@myapp/shared` isn't a real package in `node_modules`, Node.js will throw `MODULE_NOT_FOUND`. The fix is to configure the same alias in the bundler (Vite's `resolve.alias`) or use `package.json#exports` to make the package resolvable by standard Node module resolution. TypeScript and the runtime must both agree on how the alias resolves.

**Q (Medium): What does `declarationMap: true` do and why is it important for DX in a monorepo?**

Answer: `declarationMap: true` generates `.d.ts.map` files alongside `.d.ts` files. These source maps link each line in the compiled declaration file back to the corresponding line in the original TypeScript source. In an IDE with TypeScript Language Server, when you "go to definition" on a type imported from another package, the IDE follows the `.d.ts` to find the type, then uses the `.d.ts.map` to jump to the original `.ts` file. Without `declarationMap`, "go to definition" lands in the compiled `.d.ts` — you see the type signature but not the implementation, comments, or surrounding context. In a monorepo where you frequently navigate between packages, this is a major DX improvement.

**Q (Low): When is `skipLibCheck: true` appropriate and when does it hide real bugs?**

Answer: `skipLibCheck: true` skips type checking the content of all `.d.ts` files (from `node_modules/@types/`, your compiled packages, etc.). It's appropriate for applications that consume libraries but don't ship their own types — it speeds up compilation and avoids noise from version conflicts in `@types/*` packages you don't control. It hides real bugs when your own packages emit broken `.d.ts` files and a consuming package has `skipLibCheck: true`. For example, if `shared` has a TypeScript compiler error that only shows up in its generated `.d.ts` (an edge case with declaration emit), `skipLibCheck` in `web` would mask it. The safer alternative in a monorepo: use composite projects + project references, which validate each package's types within the project rather than relying on the emitted `.d.ts`. Reserve `skipLibCheck: true` for dealing with third-party type conflicts, not as a shortcut for your own packages.

---

## Self-Assessment

- [ ] Explain what `composite: true` enables and what it requires
- [ ] Write the `tsconfig.json` for a `ui` package that references a `shared` package
- [ ] Explain why TypeScript `paths` configuration doesn't affect runtime module resolution
- [ ] Describe what `declarationMap` does and why it matters for IDE navigation
- [ ] Describe when to use `skipLibCheck` and the risk it introduces

---
*Phase 14 complete. Next: Phase 15 — Memory Management — leaks, WeakMap/WeakRef, heap profiling, and object pooling.*
