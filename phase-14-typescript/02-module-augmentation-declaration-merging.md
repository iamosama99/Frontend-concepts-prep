# Module Augmentation & Declaration Merging

## Quick Reference

| Technique | Use case |
|---|---|
| Declaration merging (interfaces) | Add properties to an existing interface across files |
| Module augmentation | Add types to an existing module without forking it |
| Global augmentation | Extend `Window`, `globalThis`, built-in types |
| Ambient declarations (`.d.ts`) | Type declarations for JS-only code or untyped packages |
| `declare module` | Override or extend a module's type exports |
| `declare global` | Extend global types from within a module file |

---

## What Is This?

TypeScript merges type declarations that have the same name. This isn't an accident — it's a designed mechanism that lets you extend existing types without modifying their source, enabling:
- Adding properties to third-party library interfaces
- Extending the global `Window` object with your injected globals
- Adding express middleware properties to `Request`
- Describing the types of assets imported via a bundler (SVG, CSS modules)

---

## Interface Declaration Merging

When two `interface` declarations with the same name exist in the same scope, TypeScript merges them into one:

```ts
// a.ts
interface User {
  name: string;
}

// b.ts
interface User {
  role: 'admin' | 'user';
}

// Result — User has both:
const u: User = { name: 'Ana', role: 'admin' }; // OK
```

This works across files in the same TypeScript project. It does **not** work with `type` aliases — only `interface` supports merging.

### Merging with methods

When methods have the same name in merged interfaces, they become overloads. Order matters: later-declared (further down in the file, or declared in later files) comes first in the overload resolution:

```ts
interface EventEmitter {
  on(event: 'close', handler: () => void): void;
}

interface EventEmitter {
  on(event: 'data', handler: (chunk: Buffer) => void): void;
}

// Merged:
// on(event: 'data', handler: ...): void  ← later declaration first
// on(event: 'close', handler: ...): void
```

---

## Module Augmentation

Module augmentation adds to or modifies the types exported by an existing module. The key requirement: the file doing the augmentation must be a *module* (contain at least one `import` or `export`).

```ts
// Adding a property to Express's Request type:
import 'express'; // make this file a module (or any real import)

declare module 'express-serve-static-core' {
  interface Request {
    user?: { id: string; role: string };
  }
}

// Now in your route handlers:
app.get('/profile', (req, res) => {
  console.log(req.user?.id); // typed — no error
});
```

The string inside `declare module` must exactly match the module specifier as TypeScript resolves it. For Express, the types live in `express-serve-static-core` (an internal module), not in `express` itself.

### Adding to a library's types

```ts
// Extending lodash with a custom method you've added via mixin:
import 'lodash';

declare module 'lodash' {
  interface LoDashStatic {
    customMethod(input: string[]): string;
  }
}
```

### Augmenting a local module

You can augment your own modules too — useful in large codebases where you want to extend a shared interface from multiple locations:

```ts
// core/events.ts
export interface AppEventMap {}

// features/auth/index.ts
import './core/events';
declare module './core/events' {
  interface AppEventMap {
    'auth:login': { userId: string };
    'auth:logout': void;
  }
}
```

---

## Global Augmentation

Extend global types (types that exist without any import) using `declare global`:

```ts
// env.d.ts — typed access to environment variables
declare global {
  interface Window {
    analytics: AnalyticsInstance;
    __APP_CONFIG__: {
      apiUrl: string;
      featureFlags: Record<string, boolean>;
    };
  }

  // globalThis is typed through Window in browser contexts
  var __DEBUG__: boolean; // adds typeof globalThis['__DEBUG__']
}

export {}; // make this file a module — required for declare global
```

The `export {}` at the end is required: `declare global` must be inside a module, and a file without `import`/`export` is a script (globals), not a module.

### Built-in type augmentation

```ts
declare global {
  interface Array<T> {
    last(): T | undefined;
  }
}

// Polyfill the implementation:
Array.prototype.last = function() {
  return this[this.length - 1];
};
```

TypeScript allows augmenting built-in types this way. Use with care — patching built-ins is controversial and can conflict with future standard additions.

---

## Ambient Declarations (`.d.ts` files)

`.d.ts` files contain only type declarations — no runtime JavaScript. Used to:
- Describe the types of plain JS libraries that have no type information
- Describe assets imported via bundlers (SVG, CSS, images)
- Ship type declarations alongside a compiled library

### Asset types for bundlers

```ts
// declarations.d.ts — teach TypeScript about non-JS imports

// SVG files imported as React components (SVGR):
declare module '*.svg' {
  import React from 'react';
  const SVGComponent: React.FunctionComponent<React.SVGProps<SVGSVGElement>>;
  export default SVGComponent;
  export const ReactComponent: typeof SVGComponent;
}

// CSS modules:
declare module '*.module.css' {
  const classes: Record<string, string>;
  export default classes;
}

// PNG/JPG image paths:
declare module '*.png' {
  const url: string;
  export default url;
}
```

### Ambient module declarations

For a JS library with no `@types/...` package:

```ts
// types/some-untyped-lib.d.ts
declare module 'some-untyped-lib' {
  export function doSomething(input: string): Promise<void>;
  export const VERSION: string;
}
```

Or if you just want to silence errors without spending time on types:

```ts
declare module 'some-untyped-lib'; // types everything as `any`
```

---

## Declaration Merging with Classes and Namespaces

TypeScript allows merging namespaces with interfaces, classes, and functions:

```ts
// Merge a namespace with a function to add static properties:
function createValidator(schema: Schema): Validator {
  /* ... */
}

namespace createValidator {
  export const version = '1.0';
  export type Options = { strict: boolean };
}

// Usage:
const v = createValidator(schema);
console.log(createValidator.version); // '1.0'
```

This pattern is common in older APIs where a function doubles as a namespace for its associated types and constants.

---

## The `@types` Ecosystem

`@types/react`, `@types/node`, etc. are pure declaration packages — `.d.ts` files that describe the types of untyped npm packages. They live in `node_modules/@types/` and TypeScript automatically includes them.

When `@types` declarations are wrong or incomplete, you augment them:

```ts
// Fix an incorrect type in @types/some-lib:
import 'some-lib';
declare module 'some-lib' {
  // Override the incorrect type by re-declaring with correct type:
  interface IncorrectInterface {
    missingProperty: string; // add a property the @types missed
  }
}
```

You can't remove a property from an existing interface through augmentation (only add). For breaking changes, fork the types or use `Omit` locally.

---

## Interview Questions

**Q (High): What is the difference between module augmentation and declaration merging?**

Answer: Declaration merging is the general TypeScript behavior where multiple declarations of the same name in the same scope are merged into one. It works for interfaces (merged into a wider interface), namespaces (merged by adding members), and some combinations with classes and functions. Module augmentation is a specific use of declaration merging applied to external modules: using `declare module 'module-name' { ... }` inside a module file adds new type declarations to an existing module's exports. Declaration merging is the mechanism; module augmentation is the pattern that uses it across module boundaries. The practical distinction: declaration merging for your own interfaces is just "write two interface declarations with the same name." Module augmentation for third-party types requires the `declare module 'package-name'` wrapper.

**Q (High): You're using Express and want `req.user` to be typed as your `AuthUser` interface. How do you add it without modifying Express's source files?**

Answer: Use module augmentation. Create a declaration file (e.g., `types/express.d.ts`): import any real module to make the file a module (e.g., `import 'express'`), then declare the augmentation: `declare module 'express-serve-static-core' { interface Request { user?: AuthUser; } }`. The target module is `express-serve-static-core` rather than `express` because that's where Express's `Request` interface is actually defined. TypeScript merges your augmented `Request` interface with Express's, making `req.user` typed throughout your application. Ensure this file is included in your `tsconfig.json`'s `include` array or referenced from somewhere TypeScript resolves.

**Q (Medium): Why is `export {}` required at the end of a file containing `declare global`?**

Answer: TypeScript treats files differently based on whether they contain top-level `import` or `export` statements. A file without either is a *script* — its declarations are automatically global (in ambient scope). A file with at least one import/export is a *module* — its declarations are scoped to that module. `declare global { ... }` is the way to add to the global scope from within a module. If the file is a script (no imports/exports), you can declare globals directly — no `declare global` needed, but everything is global by default. If the file is a module, `declare global` is required for global augmentations. `export {}` is the minimal way to make a file a module without exporting anything real, enabling `declare global` to work correctly.

**Q (Low): How do you add TypeScript types for CSS module files (`.module.css`) so imports are typed?**

Answer: Create a `.d.ts` file with an ambient module declaration for the file extension pattern. Since CSS modules return an object mapping class names to string hashes, the type is `Record<string, string>`. Example: `declare module '*.module.css' { const classes: Record<string, string>; export default classes; }`. Place this in a `declarations.d.ts` or `types/assets.d.ts` file in your project and ensure it's included in your TypeScript compilation. After this, `import styles from './Button.module.css'` will type `styles` as `Record<string, string>`, and `styles.button` will be typed as `string`. For stricter typing (autocompletion of actual class names), you'd need a code generation tool that creates per-file `.d.ts` files listing the specific class names.

---

## Self-Assessment

- [ ] Explain why `interface` can be merged but `type` cannot
- [ ] Write the module augmentation to add `req.session.userId: string` to Express's Request
- [ ] Write a global augmentation that adds a `formatDate` method to the `Date` prototype
- [ ] Explain why `export {}` is needed in a file with `declare global`
- [ ] Write an ambient module declaration for `*.svg` imports that returns a URL string

---
*Next: Strict Mode Trade-offs — noUncheckedIndexedAccess, exactOptionalPropertyTypes, and migration cost.*
