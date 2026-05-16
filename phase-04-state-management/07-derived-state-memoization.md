# Derived State & Memoization Strategies

## Quick Reference

| Concept | What it is | When to use |
|---------|------------|-------------|
| Derived state | State computed from other state, not stored | Always — never sync derived state manually |
| `useMemo` | Memoize an expensive computed value | When computation is expensive or reference stability matters |
| `useCallback` | Memoize a function reference | When passing callbacks to memoized children |
| `React.memo` | Skip re-render if props haven't changed | When a child is expensive and its props are stable |
| `createSelector` | Memoized selector with structural sharing | Deriving data from Redux/Zustand store state |

---

## What Is This?

Derived state is any value that can be computed deterministically from existing state. It is not stored — it's calculated on demand. "Memoization" is the technique of caching the result of a calculation and returning the cached value when the inputs haven't changed.

```js
// Anti-pattern: storing derived state
const [items, setItems] = useState([...]);
const [filteredItems, setFilteredItems] = useState([]); // derived from items

useEffect(() => {
  setFilteredItems(items.filter(item => item.active));
}, [items]); // synchronized manually — two states that must stay in sync

// Correct: derive it
const [items, setItems] = useState([...]);
const filteredItems = items.filter(item => item.active); // computed inline
// No sync bug possible — filteredItems is always fresh
```

> **Check yourself:** What bugs does storing derived state introduce that simply computing it avoids?

---

## Why Does It Exist?

### The Synchronization Bug

Derived state stored alongside its source creates a two-state invariant you have to maintain manually. Any time you update the source but forget to update the derived copy, you have a stale view of the world. This is the same class of bug as a stale cache — the source of truth says one thing, the derived copy says another.

```js
// Bug: adding an item updates items but if filteredItems update is in a different handler...
function addItem(item) {
  setItems(prev => [...prev, item]);
  // Forgot: setFilteredItems(...) — now filteredItems is stale
}
```

Computing derived state inline eliminates the bug entirely: there's no second copy to go out of sync.

### The Performance Problem

If derived state is cheap to compute (filtering a 20-item list), compute it inline on every render — no memoization needed. If it's expensive (sorting 100,000 rows, running a complex aggregation, deep tree traversal), you need memoization: run the computation only when its inputs change.

---

## How It Works

### `useMemo` — Memoized Derived Values

`useMemo` runs the function the first time and returns its result. On subsequent renders, it re-runs the function only if the deps array has changed. Between renders with stable deps, it returns the cached result.

```js
function ProductList({ products, sortKey, filterText }) {
  // Without memoization: sorts and filters on every render — O(n log n) every time
  const displayed = products
    .filter(p => p.name.toLowerCase().includes(filterText.toLowerCase()))
    .sort((a, b) => a[sortKey] > b[sortKey] ? 1 : -1);

  // With useMemo: only re-computes when products, sortKey, or filterText changes
  const displayed = useMemo(() => {
    return products
      .filter(p => p.name.toLowerCase().includes(filterText.toLowerCase()))
      .sort((a, b) => a[sortKey] > b[sortKey] ? 1 : -1);
  }, [products, sortKey, filterText]);

  return displayed.map(p => <ProductCard key={p.id} product={p} />);
}
```

`useMemo` has two distinct uses:
1. **Performance**: skip an expensive computation when inputs are stable
2. **Reference stability**: return the same object/array reference between renders so downstream memoization (React.memo, useEffect deps) works correctly

```js
// Reference stability use case
function Parent() {
  const [count, setCount] = useState(0);
  const [name, setName] = useState('Osama');

  // Without useMemo: new object every render, MemoChild always re-renders
  const config = { name, theme: 'dark' };

  // With useMemo: same reference when name hasn't changed
  const config = useMemo(() => ({ name, theme: 'dark' }), [name]);

  return (
    <>
      <MemoChild config={config} />
      <button onClick={() => setCount(c => c + 1)}>{count}</button>
    </>
  );
}
const MemoChild = React.memo(Child);
```

### `useCallback` — Memoized Functions

`useCallback(fn, deps)` is equivalent to `useMemo(() => fn, deps)`. It returns a stable function reference across renders when deps don't change.

```js
function Parent() {
  const [items, setItems] = useState([...]);

  // Without useCallback: new function reference every render
  const handleDelete = (id) => {
    setItems(prev => prev.filter(item => item.id !== id));
  };

  // With useCallback: same reference when items doesn't change
  const handleDelete = useCallback((id) => {
    setItems(prev => prev.filter(item => item.id !== id));
  }, []); // [] because we use the functional updater form — no stale closure

  return items.map(item =>
    <MemoItem key={item.id} item={item} onDelete={handleDelete} />
  );
}
const MemoItem = React.memo(Item);
```

`useCallback` is only valuable when the function is passed as a prop to a memoized child. If the child isn't wrapped in `React.memo`, the callback's reference stability doesn't matter — the child re-renders regardless.

### `React.memo` — Component Memoization

`React.memo` wraps a component and prevents re-rendering if its props haven't changed (shallow comparison):

```js
// Will re-render only when product prop changes by reference
const ProductCard = React.memo(function ProductCard({ product, onDelete }) {
  return (
    <div>
      <span>{product.name}</span>
      <button onClick={() => onDelete(product.id)}>Delete</button>
    </div>
  );
});
```

For `React.memo` to be effective, all props must be referentially stable between renders. If `onDelete` is a new function every render (no `useCallback`), `React.memo` always re-renders.

The three must be used together: `useMemo` for stable object props, `useCallback` for stable function props, `React.memo` for components that should only re-render on prop changes.

> **Check yourself:** Why is `useCallback` useless without `React.memo` on the child? What has to be true for `React.memo` to actually skip a render?

### `createSelector` (Reselect) — Store Selectors

For state derived from a Redux (or Zustand) store, `createSelector` from Reselect computes derived values and memoizes them:

```js
import { createSelector } from 'reselect';

// Input selectors — read raw state
const selectAllUsers = state => state.users;
const selectFilter = state => state.ui.filter;

// Derived selector — only re-runs when selectAllUsers or selectFilter changes
const selectFilteredUsers = createSelector(
  [selectAllUsers, selectFilter],
  (users, filter) => users.filter(u => u.role === filter)
);

function UserList() {
  const users = useSelector(selectFilteredUsers);
  // Re-renders only when the filtered result changes, not on every store update
}
```

Without `createSelector`, `useSelector(state => state.users.filter(...))` creates a new array on every call, causing re-renders even when the underlying data hasn't changed. `createSelector` memoizes the result and returns the same reference when inputs are unchanged.

### Derived State in Signal-Based Systems

In signals-based systems (Solid, Vue Computed, Angular computed), derived state is first-class — computed values automatically track their signal dependencies:

```js
// Solid.js
const [items, setItems] = createSignal([...]);
const [filter, setFilter] = createSignal('active');

// Automatically re-computes when items() or filter() changes
const filteredItems = createMemo(() =>
  items().filter(item => item.status === filter())
);
```

`createMemo` is like `useMemo` but with automatic dependency tracking — no deps array. The signal system knows `filteredItems` reads `items` and `filter`, so it re-runs only when either changes.

---

## When NOT to Memoize

Memoization has a cost: the deps comparison itself, the cached value in memory, and the cognitive overhead of reasoning about what's cached. It's not free.

Don't memoize when:
- **The computation is cheap.** `items.find(x => x.id === id)` on a 10-item list is faster than the useMemo bookkeeping.
- **The component always re-renders anyway.** If the parent re-renders every time and passes new prop references, `React.memo` does nothing useful.
- **The deps change on every render.** If the dependency is always new (an inline object, an inline array), the memo never hits the cache.
- **You don't have a measured performance problem.** Don't premature optimize. Add memoization when you've measured that re-renders are causing dropped frames, not in advance.

```js
// Useless memo — name changes on every render (string literal is stable, but let's say it's computed)
const greeting = useMemo(() => `Hello, ${name}`, [name]);
// String concatenation is O(n) in string length — this is never the bottleneck
// Just write: const greeting = `Hello, ${name}`;
```

---

## The Stale Closure Problem

`useCallback` and `useMemo` create closures that capture values at the time of creation. If the deps array is incomplete, the closure captures a stale value:

```js
function Counter() {
  const [count, setCount] = useState(0);

  // Bug: count is captured as 0, never updates
  const double = useCallback(() => {
    console.log(count * 2); // always logs 0
  }, []); // missing count in deps

  // Fix A: add count to deps (creates new function when count changes)
  const double = useCallback(() => {
    console.log(count * 2);
  }, [count]);

  // Fix B: use functional updater (no stale closure because we don't read count directly)
  const increment = useCallback(() => {
    setCount(prev => prev + 1); // prev is always current
  }, []);
}
```

React's `eslint-plugin-react-hooks` `exhaustive-deps` rule catches missing dependencies. Always run it.

---

## Gotchas

**1. useMemo does not guarantee the cached value is kept.** React may discard the cache (in concurrent mode, when memory is constrained, during future optimizations). `useMemo` is a performance hint, not a guarantee. If your code is logically incorrect without the cache (e.g., you rely on the same reference for identity checks), that's a design bug.

**2. Object/array in deps triggers every time.** `useMemo(() => ..., [{ id: 1 }])` — the inline object `{ id: 1 }` is a new reference every render, so the memo never caches. Deps must be primitives or stable references.

**3. useCallback doesn't help with render-triggered functions.** If a parent re-renders and creates a new inline function, `useCallback` in the parent still creates a new function reference if its deps changed. The memoization only helps when deps are stable.

**4. Selector identity in Redux.** If you call `createSelector` inside a component, each component instance gets its own memoized selector — which is what you want for per-instance input (like `selectItemById(state, id)`). If you define the selector outside the component, all instances share one cache. Be deliberate about which you need.

**5. React.memo with default comparison is shallow.** If a prop is a deeply nested object, `React.memo` will only compare the top-level reference. It doesn't deep-compare. If you're passing a prop that's always a new shallow reference but has the same deep content, you need a custom comparison function: `React.memo(Component, (prevProps, nextProps) => isEqual(prevProps, nextProps))`.

---

## Interview Questions

**Q (High): Why should you never store derived state — what problems does it cause?**

Answer: Storing derived state creates a two-value invariant: both the source and the derived copy must be kept in sync. Any code path that updates the source and forgets to update the derived copy introduces a stale data bug. The derived copy and the source diverge. Additionally, storing derived state means you need a `useEffect` (or similar) to synchronize them — which runs asynchronously after render, creating a flash where the UI shows stale derived data. Computing derived state inline eliminates both: there's only one source of truth, and the derived value is always fresh with no synchronization lag.

The trap: Thinking storage is needed for performance. If the computation is expensive, the fix is `useMemo`, not a second state variable.

**Q (High): What is the difference between useMemo and useCallback? When does each apply?**

Answer: `useMemo(() => value, deps)` memoizes a *value* — the return of the factory function. `useCallback(fn, deps)` memoizes a *function* — equivalent to `useMemo(() => fn, deps)`. Use `useMemo` for expensive computations or when reference stability of an object/array matters for downstream memoization. Use `useCallback` when you need a stable function reference to pass to a memoized child (React.memo) or to include in a useEffect deps array without causing the effect to re-run unnecessarily. Neither helps in isolation: `useCallback` is only valuable when the child is wrapped in `React.memo`; `React.memo` is only effective when all props are referentially stable.

The trap: Using `useCallback` everywhere "just in case." It has overhead — the comparison itself — and is only worth it in specific scenarios.

**Q (High): What is createSelector (Reselect) and what problem does it solve?**

Answer: `createSelector` composes input selectors with a result function and memoizes the result. When a store selector like `state => state.users.filter(u => u.active)` runs inside `useSelector`, it returns a new array on every call — because `filter` always produces a new array, even with identical contents. This causes the component to re-render on every store update, not just when the relevant data changes. `createSelector` caches the previous inputs and result: if the inputs haven't changed, it returns the cached result with the same reference. The component only re-renders when the derived value actually changes.

The trap: Not knowing the root cause (filter returns new array every call). The explanation should be precise about the reference identity issue.

**Q (Medium): Explain the stale closure problem in useCallback and useMemo. How does the functional updater pattern help?**

Answer: `useCallback(fn, deps)` creates a closure that captures variables from the surrounding scope. If `deps` is incomplete, the closure captures a stale version of those variables. For example, a callback with `deps: []` that reads `count` from state will always see `count` as it was when the callback was first created — not the current value. The functional updater form (`setState(prev => prev + 1)`) avoids this by receiving the current state as an argument rather than closing over it. This means you can write `deps: []` without staleness, because the function doesn't close over the state value at all. React's `exhaustive-deps` ESLint rule catches the cases where you're closing over a value without declaring it as a dep.

The trap: Not knowing the functional updater pattern as the clean solution. Saying "just add the value to deps" is correct but creates a new function on every change, which defeats the purpose if you wanted a stable reference.

**Q (Medium): When should you NOT use useMemo? What are the signs of premature memoization?**

Answer: Don't use `useMemo` when: (1) the computation is trivial — string concatenation, simple arithmetic, `.find` on a small array are faster than the memoization overhead; (2) the component always re-renders anyway — if the parent passes a new prop reference on every render, memo never hits the cache; (3) the deps change on every render — an inline object or array in the deps list defeats caching entirely; (4) you don't have a measured problem — add memoization when you've profiled and found a bottleneck, not speculatively. Signs of premature memoization: `useMemo` wrapping a single array access or string format, `useCallback` on every handler in a component where no child is memoized, `React.memo` on components that receive object props recreated every render.

The trap: Treating memoization as always safe. The overhead of useMemo/useCallback (closure creation, deps comparison) can exceed the cost of the computation for cheap operations.

**Q (Low): When would you use React.memo with a custom comparator?**

Answer: `React.memo` uses shallow prop comparison by default — it compares each prop by reference. A custom comparator `React.memo(Comp, (prev, next) => isEqual(prev, next))` is appropriate when: (1) a prop is an object or array whose deep content is stable but whose reference always changes (e.g., a prop derived from an API response that's recreated on each fetch with the same data); (2) you want to bail out of re-rendering even when some irrelevant prop changes (a comparator that ignores certain props). The risk: a custom comparator must be correct — if it returns `true` (equal) when props actually changed, the component shows stale content. `isEqual` deep comparison is also slower than shallow comparison, so it only helps when the re-render is significantly more expensive than the deep compare.

The trap: Using custom comparators as a band-aid for reference instability that should be fixed upstream (with useMemo or useCallback).

---

## Self-Assessment

Before moving on, check off each item you can do WITHOUT looking at this file.

- [ ] Can explain why storing derived state is an anti-pattern and what bug it introduces
- [ ] Can explain the difference between useMemo and useCallback and when each applies
- [ ] Can write a createSelector that memoizes a filtered list from a Redux store
- [ ] Can explain the stale closure problem and how the functional updater pattern avoids it
- [ ] Can name three situations where you should NOT add useMemo

---
*Next: Phase 5 — Component Architecture & Scalability. We've covered where state lives and how it flows; now we look at how components themselves are architected to scale across large teams and codebases.*
