# Immutable Data Structures

## Quick Reference

| Concept | What it means | Why it matters |
|---------|--------------|----------------|
| Immutability | Values never mutate; updates produce new references | Referential equality checks are reliable |
| Structural sharing | New values share unchanged sub-trees with old values | Memory efficiency with large data |
| Immer | Produces new immutable state via a mutable "draft" | Ergonomic immutable updates |
| Object.freeze | Shallow immutability at runtime | Simple, native, does NOT deep-freeze |

---

## What Is This?

An immutable data structure is one that, once created, can never be changed. "Updating" immutable data means producing a new data structure that shares as much structure as possible with the original (structural sharing).

```js
// Mutable — mutates in place, same reference
const arr = [1, 2, 3];
arr.push(4); // arr is now [1, 2, 3, 4], same reference

// Immutable — new reference, original unchanged
const arr = [1, 2, 3];
const next = [...arr, 4]; // arr is still [1, 2, 3], next is [1, 2, 3, 4]
```

In the immutable version, `arr !== next`. This difference in reference is the mechanism that makes React's rendering, Redux's state, and memoization work correctly.

> **Check yourself:** Why does React (and most frontend state management) prefer immutable updates? What breaks if you mutate state in place?

---

## Why Does It Exist?

### The Mutation Problem

React's reconciliation relies on reference equality to decide whether state changed:

```js
// This breaks React
const [items, setItems] = useState([1, 2, 3]);

function addItem() {
  items.push(4);       // mutates the existing array
  setItems(items);     // same reference → React thinks nothing changed
}
```

`setItems(items)` passes the same array reference. React does a shallow equality check: `prevState === nextState` → `true` → no re-render. The UI doesn't update even though the data did. The mutation silently swallowed the update.

### Time-Travel Debugging

Redux devtools let you replay and rewind through state history. This requires that each state is a separate value — if states share mutable references, rewinding "corrects" the current state and all past states simultaneously, which is useless.

### Predictability

With immutable state, the history is preserved. You can always compare `prevState` and `nextState` by reference. You can serialize and deserialize state without side effects. You can memoize derived values and know they're stale only when the input reference changes.

---

## How It Works

### Spread Operator (ES6)

The simplest immutable update tool for objects and arrays:

```js
// Immutable object update
const user = { name: 'Osama', role: 'admin', settings: { theme: 'dark' } };

// Shallow update — only top-level properties are new
const updated = { ...user, role: 'editor' };

// Deep update — must spread at every level
const deepUpdated = {
  ...user,
  settings: {
    ...user.settings,
    theme: 'light',
  },
};
```

The problem: deeply nested updates are verbose. Three levels deep and the spread syntax becomes difficult to read.

### Array Methods

Prefer non-mutating array methods:

```js
const items = [1, 2, 3, 4, 5];

// Mutating (avoid in state)        // Immutable alternative
items.push(6)                        [...items, 6]
items.pop()                          items.slice(0, -1)
items.splice(2, 1)                   items.filter((_, i) => i !== 2)
items[2] = 99                        items.map((v, i) => i === 2 ? 99 : v)
items.sort(...)                      [...items].sort(...)
items.reverse()                      [...items].reverse()
```

### Immer

Immer lets you write mutations on a "draft" — a proxy that records all changes — and produces a new immutable value:

```js
import { produce } from 'immer';

const state = {
  users: [
    { id: 1, name: 'Osama', active: true },
    { id: 2, name: 'Ali', active: false },
  ],
};

// Without Immer — verbose deep spread
const next = {
  ...state,
  users: state.users.map(u =>
    u.id === 2 ? { ...u, active: true } : u
  ),
};

// With Immer — write the mutation, Immer makes it immutable
const next = produce(state, draft => {
  const user = draft.users.find(u => u.id === 2);
  user.active = true; // looks like mutation but produces a new reference
});

console.log(state === next); // false — new reference
console.log(state.users[0] === next.users[0]); // true — structural sharing, unchanged parts reused
```

Immer uses Proxies to intercept the mutations on the draft. At the end of the `produce` call, it applies those mutations to a new immutable structure, carrying over unchanged parts by reference (structural sharing).

Redux Toolkit uses Immer internally in `createSlice` — that's why you can "mutate" state in reducers:

```js
const usersSlice = createSlice({
  name: 'users',
  initialState: [],
  reducers: {
    activate: (state, action) => {
      const user = state.find(u => u.id === action.payload);
      if (user) user.active = true; // this is fine — Immer handles it
    },
  },
});
```

> **Check yourself:** In the Immer example above, `state.users[0] === next.users[0]` is `true`. Why does this matter for React performance? What does it mean for memoized components?

### Structural Sharing

Naive immutability would copy the entire data structure on every update — expensive for large datasets. Structural sharing avoids this: only the path from the root to the changed node is new; everything else is shared by reference.

```
Before: root → [A, B, C]
Update C:
After:  root' → [A, B, C']
                     ↑
                 Shared reference — not copied
```

Persistent data structure libraries like **Immutable.js** implement this with tree-based structures (HAMTs — Hash Array Mapped Tries) that guarantee O(log n) updates with structural sharing.

```js
import { Map, List } from 'immutable';

const state = Map({ count: 0, items: List([1, 2, 3]) });
const next = state.set('count', 1); // new Map, same items List reference
state.get('items') === next.get('items'); // true — shared
```

Immutable.js fell out of widespread use with React because it requires converting back and forth between Immutable.js types and plain JS, which adds friction. Immer with plain JS objects (and native structural sharing via spread) covers most use cases without this cost.

### Object.freeze

Native shallow immutability:

```js
const obj = Object.freeze({ name: 'Osama', settings: { theme: 'dark' } });

obj.name = 'Ali';           // silently fails in non-strict mode, throws in strict
obj.settings.theme = 'light'; // WORKS — freeze is shallow, settings is not frozen
```

`Object.freeze` is shallow. Nested objects are not frozen. It's useful for configuration objects or action type constants, but not for deeply nested state.

---

## Immutability in Practice

### React state updates

```js
// Wrong — reference never changes, React doesn't re-render
setState(prevState => {
  prevState.count++;
  return prevState; // same reference
});

// Correct — new reference
setState(prevState => ({ ...prevState, count: prevState.count + 1 }));
```

### useMemo / React.memo depend on reference stability

```js
const items = [{ id: 1, name: 'Apple' }];

// If items is recreated on every render (new array literal in JSX), memo does nothing
const MemoList = React.memo(List);
<MemoList items={items} /> // new array reference every render → memo never skips

// Fix: stabilize the reference
const [items, setItems] = useState([{ id: 1, name: 'Apple' }]);
<MemoList items={items} /> // same reference between renders → memo works
```

---

## Gotchas

**1. Spread is shallow.** `{ ...obj }` only copies one level. Nested objects are still shared references. Mutating a nested property on the copy mutates the original. Always spread at every level you're modifying.

**2. Array sort and reverse mutate in place.** `[...arr].sort(fn)` — always clone before sorting if you need the original.

**3. Immer only wraps objects and arrays.** Primitives don't need Immer — just return the new value. Trying to produce a primitive draft doesn't work.

**4. Object.freeze is shallow.** A common mistake is to freeze a config object and assume nested properties are also protected. They're not.

**5. Reference equality ≠ value equality.** `prevState !== nextState` means the reference changed, not necessarily the content. If you spread an object but don't change anything, you get a new reference and trigger a re-render even though the data is identical. Don't create unnecessary new references.

**6. Structural sharing doesn't help if you clone the whole tree.** A naive deep clone (`JSON.parse(JSON.stringify(state))`) destroys structural sharing — everything is new. Immer preserves structural sharing; JSON round-trip doesn't.

---

## Interview Questions

**Q (High): Why does React require immutable state updates? What breaks if you mutate state in place?**

Answer: React uses referential equality (`===`) to detect state changes. If you mutate state in place and pass the same object reference to setState, React sees `prevState === nextState` as true and skips re-rendering — the UI doesn't update even though the data changed. Beyond rendering, mutation also breaks: (1) React's concurrent mode, which may re-use or discard intermediate renders based on the assumption that state snapshots are stable; (2) devtools time-travel, which depends on each state being a distinct, independent value; (3) `React.memo` and `useMemo` memoization, which use referential equality to determine if inputs changed. Immutable updates give you reliable, cheap change detection.

The trap: Saying "because React's docs say so." The answer should name the specific mechanism (referential equality check) and the consequences of violating it.

**Q (High): What is structural sharing and why does Immer implement it?**

Answer: Structural sharing means that when you produce an updated immutable value, unchanged subtrees are not copied — they're shared by reference between the old and new values. Only the path from the root to the changed node is new. Immer implements this so that updating a deeply nested property doesn't clone the entire data structure. In practice, this means: if you update `state.users[2].active`, the new state shares `state.items`, `state.settings`, and every other top-level key by reference with the old state. Components that subscribe to `state.items` via a selector will see the same reference and won't re-render — which is exactly the performance behavior you want.

The trap: Thinking Immer just does a deep clone. It explicitly doesn't. It's structurally sharing everything it can.

**Q (Medium): What is the difference between Immer and Immutable.js? Which is more common today and why?**

Answer: Immutable.js (Facebook, 2014) provides custom persistent data structures (Map, List, Set, etc.) implemented as HAMTs — Hash Array Mapped Tries — that guarantee efficient immutable updates with structural sharing. You interact with these special types, not plain JS objects. Immer (2019) works with plain JS objects by using Proxies to record mutations on a draft, then producing a new immutable plain object with structural sharing. Immer dominates today because: (1) you work with plain JS — no conversion between types, no special API for reads; (2) it integrates seamlessly with TypeScript; (3) Redux Toolkit adopted it for reducers. Immutable.js's custom types created friction at integration points (JSON serialization, third-party libraries, type inference).

The trap: Not knowing why Immutable.js fell out of favor. The "special types" problem is the crux.

**Q (Medium): Why is Array.sort() a common source of bugs in immutable state patterns? How do you fix it?**

Answer: `Array.sort()` and `Array.reverse()` mutate the array in place and return the same reference. If you call `state.items.sort(fn)`, you've mutated `state.items` directly — the same array that's stored in state is now in a different order. React may or may not re-render depending on timing, but the original state has been corrupted. Fix: clone before sorting — `[...state.items].sort(fn)`. Same for reverse: `[...state.items].reverse()`. The spread clone is shallow, but for sorting you only care about the top-level ordering, not cloning the item objects themselves.

The trap: Not knowing that sort returns the same reference. Developers from other languages assume sort returns a new array.

**Q (Low): When would you use Object.freeze over Immer for immutability?**

Answer: `Object.freeze` is appropriate for: configuration objects that should never change at runtime, action type constants in Redux (though string literals are simpler), and small objects where you want a runtime error (in strict mode) if someone accidentally mutates. Immer is better for state management — it handles deep updates ergonomically, produces structural sharing, and works with nested structures. `Object.freeze` is shallow and doesn't stop nested mutations. You wouldn't use Object.freeze in a React state update flow; you'd use it for static config that is defined once and should never change.

The trap: Confusing freeze with "deep freeze." Object.freeze is always shallow.

---

## Self-Assessment

Before moving on, check off each item you can do WITHOUT looking at this file.

- [ ] Can explain why React requires immutable updates (referential equality check)
- [ ] Can write a deeply-nested immutable update using spread syntax
- [ ] Can use Immer's `produce` to update a nested property and explain what structural sharing means in that context
- [ ] Can name the array methods that mutate in place and their immutable equivalents
- [ ] Can explain why Object.freeze is shallow and when that matters

---
*Next: Derived State & Memoization Strategies — with immutable data, you can safely derive computed values from state and memoize them: the reference stability of immutable updates is exactly what memoization relies on.*
