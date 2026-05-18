# Strict Mode Trade-offs

## Quick Reference

| Flag | What it catches | Cost |
|---|---|---|
| `strict` | Umbrella: enables all below | Medium |
| `strictNullChecks` | `null`/`undefined` can't be assigned to non-nullable types | High — most impactful |
| `noImplicitAny` | Variables must be explicitly typed or inferrable | Low |
| `strictFunctionTypes` | Method parameter variance checks | Low-medium |
| `strictPropertyInitialization` | Class properties must be initialized in constructor | Medium |
| `noUncheckedIndexedAccess` | Array index / object key access returns `T \| undefined` | High — many false positives |
| `exactOptionalPropertyTypes` | `{ x?: string }` means x is `string \| undefined`, not `string` | Medium |
| `noUncheckedSideEffectImports` | Imports without bindings must be explicitly typed as side-effect | Low |

---

## What Is This?

TypeScript's `strict` flag is a shorthand that enables a set of stricter compiler checks. Beyond `strict`, there are additional flags that provide even more precise type safety but are not part of the `strict` umbrella — they require explicit opt-in because they generate significantly more errors and require code changes that may not be justified in every codebase.

Understanding what each flag catches — and what it costs in terms of code changes and false positives — lets you make an informed decision about which flags are worth enabling for your project.

---

## The `strict` Umbrella

`"strict": true` in `tsconfig.json` enables:
- `strictNullChecks`
- `noImplicitAny`
- `strictFunctionTypes`
- `strictBindCallApply`
- `strictPropertyInitialization`
- `useUnknownInCatchVariables`
- `noImplicitThis`
- `alwaysStrict` (emits `'use strict'`)

These are the baseline. Modern greenfield projects should always start with `strict: true`.

---

## `strictNullChecks` — The Most Impactful Flag

Without `strictNullChecks`, every type implicitly includes `null` and `undefined`. `string` means `string | null | undefined`. This is TypeScript's pre-2016 default and the reason millions of runtime null-reference errors went undetected.

```ts
// Without strictNullChecks:
let name: string = null; // allowed

// With strictNullChecks:
let name: string = null; // Error: Type 'null' is not assignable to type 'string'
let nameOrNull: string | null = null; // explicit — OK
```

With `strictNullChecks`, TypeScript narrows types based on null checks:

```ts
function greet(user: User | null) {
  console.log(user.name); // Error — user might be null

  if (user !== null) {
    console.log(user.name); // OK — TypeScript narrowed to User
  }

  console.log(user?.name); // OK — optional chaining
  console.log(user!.name); // OK but unsafe — non-null assertion
}
```

**Cost:** Adds `| null` or `| undefined` to many types. Requires null checks before property access. Can generate hundreds of errors in existing code. Worth it: the null checks you add expose real bugs.

---

## `noUncheckedIndexedAccess`

Array index access and object string-index access return `T | undefined`, not `T`. This reflects reality: accessing `arr[5]` when the array has 3 elements returns `undefined`, not a `string`.

```ts
// Without noUncheckedIndexedAccess:
const arr: string[] = ['a', 'b', 'c'];
const first: string = arr[0]; // OK — TypeScript assumes arr[0] is string
const fifth: string = arr[100]; // OK — TypeScript is wrong — this is undefined

// With noUncheckedIndexedAccess:
const first = arr[0]; // type: string | undefined
const safe = arr[0] ?? 'default'; // OK — handle the undefined case

// Must check before use:
if (arr[0] !== undefined) {
  const value: string = arr[0]; // narrowed to string
}
```

Same for object index signatures:

```ts
const map: Record<string, number> = { a: 1 };
const val = map['b']; // type: number | undefined (not number)
```

**Trade-off:** Eliminates a real class of undefined-access bugs. The cost is high: every array access and map lookup requires null handling. Many loops that "obviously" have values become verbose:

```ts
// With noUncheckedIndexedAccess — annoying but correct:
for (let i = 0; i < arr.length; i++) {
  const item = arr[i]; // string | undefined
  if (item === undefined) continue; // required
  processItem(item);
}

// for-of is unaffected:
for (const item of arr) { // string — not undefined
  processItem(item);
}
```

**Recommendation:** Enable for new codebases with discipline. In large existing codebases, the noise-to-signal ratio can be high. Destructuring and for-of loops are safer alternatives to index access.

> **Check yourself:** Why does `for...of` avoid the `noUncheckedIndexedAccess` problem while `arr[i]` inside a `for` loop does not?

---

## `exactOptionalPropertyTypes`

Without this flag, an optional property `x?: string` is treated as `x?: string | undefined` — assigning `undefined` explicitly is allowed:

```ts
interface User {
  name?: string;
}

// Without exactOptionalPropertyTypes:
const u: User = { name: undefined }; // allowed — treated as if name is absent
```

With `exactOptionalPropertyTypes`:

```ts
// With exactOptionalPropertyTypes:
const u: User = { name: undefined }; // Error — name must be string, not undefined
const u2: User = {}; // OK — name is absent (different from undefined)
const u3: User = { name: 'Ana' }; // OK
```

This distinguishes between "the property is absent" (key doesn't exist) and "the property is set to undefined" (key exists, value is undefined) — which are genuinely different in JavaScript (`'name' in u` vs. `u.name === undefined`).

**Trade-off:** More precise but breaks code that assigns `undefined` to optional properties to effectively "unset" them. Requires explicit `?? undefined` handling in many spread/merge patterns.

**Interaction with `Object.assign` and spread:**

```ts
// exactOptionalPropertyTypes + spread:
function mergeUser(base: User, updates: Partial<User>): User {
  return { ...base, ...updates }; // Error if Partial<User> allows undefined values
}
// Fix: use exactOptionalPropertyTypes-aware Partial:
type StrictPartial<T> = { [K in keyof T]?: T[K] }; // same as Partial but without | undefined added
```

---

## `noImplicitAny`

Variables whose types can't be inferred must be explicitly typed. Without it, TypeScript silently uses `any` when it can't infer a type:

```ts
// Without noImplicitAny:
function process(data) { // data is implicitly any
  return data.x + data.y; // no type checking
}

// With noImplicitAny:
function process(data) { // Error: Parameter 'data' implicitly has an 'any' type
  /* ... */
}

// Fix:
function process(data: { x: number; y: number }) { /* ... */ }
// or:
function process(data: unknown) {
  if (typeof data === 'object' && data !== null && 'x' in data) {
    /* ... */
  }
}
```

**Cost:** Low. Mostly requires adding type annotations to function parameters. Good discipline from the start.

---

## `strictPropertyInitialization`

Class properties declared without an initializer must be assigned in the constructor:

```ts
class UserService {
  private db: Database; // Error — property not initialized in constructor

  constructor() {
    // must assign this.db here
  }
}

// Fix 1: Initialize in constructor:
class UserService {
  private db: Database;
  constructor(db: Database) {
    this.db = db;
  }
}

// Fix 2: Definite assignment assertion (use sparingly — tells TypeScript "trust me"):
class UserService {
  private db!: Database; // ! says "I guarantee this is initialized before use"
}
```

**Trade-off:** The `!` (definite assignment assertion) is an escape hatch that defeats the check. Dependency injection frameworks and decorators often require `!` because they assign properties outside the constructor. This is a real limitation — use `!` only when the framework genuinely guarantees initialization.

---

## `useUnknownInCatchVariables`

Caught errors default to `unknown` instead of `any`:

```ts
// Without useUnknownInCatchVariables (or pre-TS 4.0):
try {
  doSomething();
} catch (e) {
  e.message; // OK — e is any
}

// With useUnknownInCatchVariables:
try {
  doSomething();
} catch (e) {
  e.message; // Error — e is unknown
  if (e instanceof Error) {
    e.message; // OK — narrowed to Error
  }
}
```

**Cost:** Low. Adds `instanceof Error` checks in catch blocks. Always correct practice.

---

## Progressive Strictness Migration

Enabling `strict: true` on an existing JavaScript-converted codebase typically generates hundreds or thousands of errors. The approach:

1. Enable `"strict": false` and specific flags one at a time, starting with the least impactful (`noImplicitAny`)
2. Use `// @ts-nocheck` on files not yet migrated
3. Track progress with error counts per file
4. Use `allowJs: true` + `checkJs: false` to migrate file-by-file

For `noUncheckedIndexedAccess` and `exactOptionalPropertyTypes` specifically, the cost-benefit calculation depends on codebase size and team discipline. Many mature TypeScript codebases run `strict: true` but skip these two.

---

## Interview Questions

**Q (High): What is `noUncheckedIndexedAccess` and why does it exist?**

Answer: `noUncheckedIndexedAccess` makes TypeScript add `| undefined` to the type of any array index access (`arr[i]`) or object with an index signature (`map[key]`). Without it, `arr[0]` is typed as `string` even if the array might be empty — TypeScript trusts the index without checking the array bounds. At runtime, accessing out-of-bounds returns `undefined`, but TypeScript reports the type as `string`. This is a real source of "cannot read property of undefined" errors. The flag exists to accurately model what JavaScript actually returns from index operations. The trade-off is verbosity: now every index access requires a null check. for-of loops and destructuring avoid this because TypeScript understands they iterate over guaranteed-existing elements.

**Q (High): Explain the difference between `exactOptionalPropertyTypes` enabled and disabled.**

Answer: An optional property `name?: string` without `exactOptionalPropertyTypes` is treated as `name?: string | undefined` — you can explicitly assign `undefined` to it. This conflates "property absent" with "property set to undefined," which are genuinely different in JavaScript (`'name' in obj` returns different results). With `exactOptionalPropertyTypes`, `name?: string` means the property is either absent or a `string` — you cannot assign `undefined` to it explicitly. To "remove" a property, you omit it in the object literal or use delete — you can't assign `undefined`. This is more precise but breaks code that relies on `obj.name = undefined` to effectively unset a property, and it interacts poorly with `Partial<T>` types used for updates/patches.

**Q (Medium): A codebase has `strict: true` but still has runtime null errors. What additional flags should you consider?**

Answer: `strict: true` enables `strictNullChecks`, which catches null/undefined in typed positions. But runtime null errors can still come from: (1) index access — `noUncheckedIndexedAccess` makes TypeScript report `arr[i]` as `T | undefined` instead of `T`; (2) type assertions (`value as SomeType` bypasses null checks); (3) `!` (non-null assertions) overriding the check; (4) `any` types that entered the codebase via third-party code or intentional any usage. The additional flags to consider: `noUncheckedIndexedAccess` for index access, `exactOptionalPropertyTypes` for optional property precision. Audit `!` usage and explicit `as` casts — these are the places where developers told TypeScript to trust them and TypeScript complied.

**Q (Low): When is the `!` definite assignment assertion (`property!: Type`) appropriate on a class property?**

Answer: It's appropriate when a property is guaranteed to be assigned before any method accesses it, but the assignment happens outside the constructor — specifically in dependency injection containers (Angular's DI assigns properties via decorators), ORMs (Prisma, TypeORM inject properties), and testing utilities (some test harnesses initialize properties via `beforeEach`). In all these cases, the framework guarantees assignment before use, but TypeScript can't see that guarantee. Using `!` tells TypeScript "I've verified this is safe by external means." It should NOT be used as a shortcut to avoid initializing properties you've genuinely forgotten — that's exactly the bug `strictPropertyInitialization` is designed to catch.

---

## Self-Assessment

- [ ] Explain what `strictNullChecks` enables and what code changes it typically requires
- [ ] Describe the behavior change `noUncheckedIndexedAccess` makes and why for-of loops avoid it
- [ ] Explain the JavaScript distinction between "absent property" and "property set to undefined"
- [ ] Describe a situation where `!` (definite assignment assertion) is legitimately needed
- [ ] Outline a strategy for enabling strict mode on an existing large TypeScript codebase

---
*Next: Type-safe API Contracts — Zod, tRPC, and OpenAPI Codegen.*
