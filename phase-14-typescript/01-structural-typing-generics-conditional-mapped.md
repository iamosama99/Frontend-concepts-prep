# Structural Typing, Generics, Conditional & Mapped Types

## Quick Reference

| Concept | One-liner |
|---|---|
| Structural typing | Types are compatible if shapes match, not if names match |
| Generics | Types parameterized over other types — `Array<T>`, `Promise<T>` |
| Conditional types | `T extends U ? X : Y` — types that branch based on type relationships |
| Mapped types | Transform every property of an object type: `{ [K in keyof T]: ... }` |
| Template literal types | String-level type manipulation: `` `on${Capitalize<K>}` `` |
| Infer | Extract a type from within a conditional type: `T extends Array<infer U> ? U : never` |

---

## What Is This?

TypeScript's type system operates on four core mechanisms: structural compatibility, generic parameterization, conditional branching, and mapped transformation. Together they form an expressive type algebra — a language for deriving types from types, which enables catching entire classes of errors at compile time that would be runtime surprises in JavaScript.

---

## Structural Typing

TypeScript uses **structural** typing, not **nominal** typing. Two types are compatible if they have the same shape — the same set of required properties with compatible types. Names are irrelevant.

```ts
type Point = { x: number; y: number };
type Coordinate = { x: number; y: number };

const pt: Point = { x: 1, y: 2 };
const coord: Coordinate = pt; // valid — same structure, even though types have different names
```

Compare to nominal typing (Java, C#): `Point` and `Coordinate` would be incompatible even if identical, because they're different declared types.

**Excess property checking** is the exception: when you assign an object literal directly to a typed variable, TypeScript checks for extra properties. This prevents typos in object literals:

```ts
type User = { name: string; age: number };

const u: User = { name: 'Ana', age: 30, role: 'admin' }; // Error — excess property 'role'

const obj = { name: 'Ana', age: 30, role: 'admin' };
const u2: User = obj; // OK — obj is widened type, not an excess check
```

Excess checking only fires on *fresh object literals*, not on variables.

**Structural subtypes:** A type `B` is a subtype of `A` if `B` has at least all the properties `A` requires:

```ts
type Animal = { name: string };
type Dog = { name: string; breed: string }; // Dog extends Animal structurally

function greet(a: Animal) { console.log(a.name); }
const dog: Dog = { name: 'Rex', breed: 'Lab' };
greet(dog); // valid — Dog has all of Animal's required properties
```

> **Check yourself:** If TypeScript is structural, why does the excess property check feel like nominal typing?

---

## Generics

Generics parameterize types. They let you write type-safe code that works over *a range* of types rather than any specific one.

### Basic generics

```ts
function identity<T>(value: T): T {
  return value;
}

identity<string>('hello'); // T = string
identity(42);              // T inferred as number
```

### Generic constraints

Constrain `T` to types that satisfy a shape:

```ts
function getLength<T extends { length: number }>(item: T): number {
  return item.length;
}

getLength('hello');   // string has .length — OK
getLength([1, 2, 3]); // array has .length — OK
getLength(42);        // number has no .length — Error
```

### Generic interfaces and classes

```ts
interface Repository<T> {
  findById(id: string): Promise<T | null>;
  save(item: T): Promise<T>;
  delete(id: string): Promise<void>;
}

class UserRepository implements Repository<User> {
  async findById(id: string): Promise<User | null> { /* ... */ }
  async save(user: User): Promise<User> { /* ... */ }
  async delete(id: string): Promise<void> { /* ... */ }
}
```

### Multiple type parameters

```ts
function pair<K, V>(key: K, value: V): [K, V] {
  return [key, value];
}

const p = pair('age', 30); // [string, number]
```

### Default type parameters

```ts
interface ApiResponse<T = unknown> {
  data: T;
  status: number;
}

type DefaultResponse = ApiResponse;           // data: unknown
type UserResponse = ApiResponse<User>;        // data: User
```

---

## Conditional Types

Types that branch: `T extends U ? X : Y`.

```ts
type IsString<T> = T extends string ? true : false;

type A = IsString<string>;   // true
type B = IsString<number>;   // false
type C = IsString<'hello'>;  // true (string literal extends string)
```

### Distributive conditional types

When `T` is a union and the conditional type is applied, it distributes over each member:

```ts
type ToArray<T> = T extends unknown ? T[] : never;

type Str = ToArray<string>;          // string[]
type StrOrNum = ToArray<string | number>; // string[] | number[] (distributed)
```

This is the mechanism behind `NonNullable`, `Exclude`, and `Extract`:

```ts
type NonNullable<T> = T extends null | undefined ? never : T;
// NonNullable<string | null | undefined> → string

type Exclude<T, U> = T extends U ? never : T;
// Exclude<'a' | 'b' | 'c', 'a'> → 'b' | 'c'

type Extract<T, U> = T extends U ? T : never;
// Extract<'a' | 'b' | number, string> → 'a' | 'b'
```

### `infer` — extracting types

`infer` captures a type from within a conditional:

```ts
// Extract the element type from an array:
type ElementType<T> = T extends Array<infer U> ? U : never;
type E = ElementType<string[]>; // string

// Extract the return type of a function:
type ReturnType<T> = T extends (...args: any[]) => infer R ? R : never;
type R = ReturnType<() => number>; // number

// Extract the first argument type:
type FirstArg<T> = T extends (first: infer F, ...rest: any[]) => any ? F : never;
type F = FirstArg<(id: string, opts: object) => void>; // string

// Unwrap a Promise:
type Awaited<T> = T extends Promise<infer U> ? Awaited<U> : T;
```

---

## Mapped Types

Transform every property of an object type. The syntax is `{ [K in keyof T]: transformation }`.

### Built-in mapped types

```ts
// Make all properties optional:
type Partial<T> = { [K in keyof T]?: T[K] };

// Make all properties required:
type Required<T> = { [K in keyof T]-?: T[K] }; // -? removes optional

// Make all properties readonly:
type Readonly<T> = { readonly [K in keyof T]: T[K] };

// Remove readonly:
type Mutable<T> = { -readonly [K in keyof T]: T[K] };

// Create an object type from a union of keys:
type Record<K extends string | number | symbol, V> = { [P in K]: V };
```

### Key remapping with `as`

Transform property names in the output:

```ts
type Getters<T> = {
  [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K]
};

type User = { name: string; age: number };
type UserGetters = Getters<User>;
// { getName: () => string; getAge: () => number }
```

Exclude properties by mapping to `never`:

```ts
type OmitPrivate<T> = {
  [K in keyof T as K extends `_${string}` ? never : K]: T[K]
};

type Impl = { name: string; _internal: boolean };
type Public = OmitPrivate<Impl>; // { name: string }
```

### Combining mapped + conditional

```ts
// Make properties optional if their value is nullable:
type PartialNullable<T> = {
  [K in keyof T]: null extends T[K] ? T[K] | undefined : T[K]
};

// Deep partial:
type DeepPartial<T> = {
  [K in keyof T]?: T[K] extends object ? DeepPartial<T[K]> : T[K]
};
```

---

## Template Literal Types

String manipulation at the type level:

```ts
type EventName<T extends string> = `on${Capitalize<T>}`;
type ClickEvent = EventName<'click'>; // 'onClick'

type Routes = '/users' | '/posts' | '/settings';
type ApiEndpoints = `api${Routes}`;
// 'api/users' | 'api/posts' | 'api/settings'

// Paired with mapped types:
type EventHandlers<T extends string> = {
  [K in T as `on${Capitalize<K>}`]?: (event: Event) => void
};
```

---

## Practical Patterns

### Pick and Omit

```ts
type Pick<T, K extends keyof T> = { [P in K]: T[P] };
type Omit<T, K extends keyof T> = Pick<T, Exclude<keyof T, K>>;

type User = { id: string; name: string; email: string; password: string };
type PublicUser = Omit<User, 'password'>;
// { id: string; name: string; email: string }
```

### Parameters and ReturnType

```ts
type Parameters<T extends (...args: any) => any> =
  T extends (...args: infer P) => any ? P : never;

function createUser(name: string, age: number, role: 'admin' | 'user') {}
type CreateUserParams = Parameters<typeof createUser>;
// [name: string, age: number, role: 'admin' | 'user']
```

### Discriminated unions

```ts
type Result<T> =
  | { ok: true; value: T }
  | { ok: false; error: string };

function processResult<T>(result: Result<T>): T {
  if (result.ok) {
    return result.value; // TypeScript narrows: result is { ok: true; value: T }
  }
  throw new Error(result.error); // narrowed: result is { ok: false; error: string }
}
```

---

## Interview Questions

**Q (High): What is structural typing and how does it differ from nominal typing? Give an example of where it causes a surprise.**

Answer: In structural typing, two types are compatible if they have the same shape — the same properties with compatible types. The names of the types don't matter. TypeScript uses structural typing: a type `Dog` with `{ name: string; breed: string }` is assignable to `Animal` with `{ name: string }` because Dog has everything Animal requires. Nominal typing (Java, C#) would reject this unless `Dog` explicitly declared `implements Animal`. The main surprise in TypeScript is excess property checking: directly assigning an object literal with extra properties fails (`{ name: 'x', extra: 1 }` to a type requiring only `{ name: string }`) — but assigning through an intermediate variable works. This inconsistency exists because fresh object literals are the most common typo source, so TypeScript adds that specific check only on direct assignments.

**Q (High): What does `infer` do in a conditional type? Write a type that extracts the resolved value of a Promise.**

Answer: `infer` captures a type variable from within a conditional type's extends clause. It's only valid inside the `true` branch of a conditional and allows you to extract a type that TypeScript would otherwise not let you name directly. Example that extracts a Promise's resolved type: `type Awaited<T> = T extends Promise<infer U> ? Awaited<U> : T`. When `T` is `Promise<string>`, TypeScript matches it against `Promise<infer U>` and binds `U = string`, so the type evaluates to `Awaited<string>`, which recurses to the base case (string doesn't extend Promise) and returns `string`. The recursion handles nested promises: `Promise<Promise<number>>` → `Awaited<Promise<number>>` → `Awaited<number>` → `number`. This exact pattern is used in TypeScript's built-in `Awaited<T>` utility type.

**Q (High): What is a mapped type? Write one that makes all properties in an object type optional and readonly simultaneously.**

Answer: A mapped type iterates over the keys of an object type and produces a new object type by transforming each property. The syntax is `{ [K in keyof T]: transformation }`. To make properties optional and readonly: `type ReadonlyPartial<T> = { readonly [K in keyof T]?: T[K] }`. Breaking it down: `readonly` makes each property immutable; `?` makes each property optional; `K in keyof T` iterates over all keys of T; `T[K]` is the original property type, preserved unchanged. Mapped types underpin all of TypeScript's built-in utility types: `Partial`, `Required`, `Readonly`, `Pick`, `Omit`, and `Record`.

**Q (Medium): What is distributive conditional types? When is it useful and how do you opt out of it?**

Answer: When a conditional type is written as `T extends U ? X : Y` and `T` is a naked type parameter that receives a union type, TypeScript distributes the conditional over each member of the union. `ToArray<string | number>` with `type ToArray<T> = T extends unknown ? T[] : never` evaluates as `(string extends unknown ? string[] : never) | (number extends unknown ? number[] : never)` = `string[] | number[]`. This is intentional and useful — it's how `Exclude<T, U>` and `NonNullable<T>` work. To opt out, wrap the type in a tuple: `type ToArray<T> = [T] extends [unknown] ? T[] : never`. Now `ToArray<string | number>` = `(string | number)[]` — the union is not distributed.

**Q (Low): What does the key remapping `as` clause do in a mapped type?**

Answer: The `as` clause in a mapped type transforms the output property names. Without it, `{ [K in keyof T]: ... }` produces the same keys as the input. With `as`, you can: rename keys using template literal types (e.g., `` [K in keyof T as `get${Capitalize<K & string>}`] `` creates getter names); filter keys by mapping to `never` (e.g., `[K in keyof T as K extends '_internal' ? never : K]` removes private-prefixed keys); or any other key transformation. It's the mechanism for deriving a differently-shaped object type from an existing one while maintaining the link between input and output types, enabling the mapped type to carry the original property types through `T[K]`.

---

## Self-Assessment

- [ ] Explain structural typing and the excess property check exception
- [ ] Write a generic `Maybe<T>` type that is either `T | null | undefined`
- [ ] Write a conditional type that extracts the element type from an array
- [ ] Write a mapped type that converts all methods in an object type to return `Promise<ReturnType<method>>`
- [ ] Explain what distributive conditional types are and how to disable distribution

---
*Next: Module Augmentation & Declaration Merging — extending third-party types and global augmentation.*
