# Unidirectional vs. Bidirectional Data Flow

## Quick Reference

| Model | Direction | How state updates | Frameworks |
|-------|-----------|-------------------|------------|
| Unidirectional | View → Action → State → View | Explicit dispatch/setter | React, Redux, Elm |
| Bidirectional | View ↔ State (both ways) | Two-way binding, automatic | Angular (ngModel), Vue (v-model) |

---

## What Is This?

Data flow describes how changes propagate through an application: when state changes, how does the view update? When the user interacts, how does state change?

**Unidirectional data flow** enforces a strict one-way cycle: state flows down to the view, and user interactions trigger actions that flow up and eventually produce new state. There is no shortcut from view to state.

**Bidirectional data flow** (two-way binding) links state and view directly — a change to the view automatically updates the state, and a change to state automatically updates the view.

```js
// Unidirectional — React controlled input
function Input() {
  const [value, setValue] = useState('');
  return <input value={value} onChange={e => setValue(e.target.value)} />;
}

// Bidirectional — Angular template
// <input [(ngModel)]="username">
// username changes → input updates; input changes → username updates
```

> **Check yourself:** In the React example above, what happens if you remove the `onChange` handler but keep `value={value}`? Why?

---

## Why Does It Exist?

### The case for bidirectional binding

Early frameworks like Knockout.js and Backbone made two-way binding popular because it drastically reduces boilerplate. Wiring an input to state is one line. Angular's `ngModel` and Vue's `v-model` carry this tradition forward. For form-heavy applications, the ergonomics are undeniably faster to write.

### The case for unidirectional flow

React's 2013 introduction of unidirectional data flow was a reaction to a specific class of bugs. In large Angular 1 applications, two-way binding created chains of watchers: state A binds to state B binds to state C, and a change to A triggers a cascade that is difficult to trace and predict. Facebook's newsfeed "mark as read" bug — where notifications re-appeared after being dismissed — was famously caused by this kind of circular update cycle.

Unidirectional flow eliminates the surprise: you always know where state can change (only at the source, via explicit setters or actions), and you always know how it flows (downward, via props or subscriptions).

---

## How It Works

### Unidirectional Flow Cycle

```
User interaction
      ↓
  Dispatch action / call setter
      ↓
  State update (pure function or store mutation)
      ↓
  Re-render with new state
      ↓
  View reflects new state
```

In React + Redux this looks like:

```js
// State lives in the store
const counterSlice = createSlice({
  name: 'counter',
  initialState: { value: 0 },
  reducers: {
    increment: state => { state.value += 1; },
  },
});

// View reads state, dispatches actions — no direct mutation
function Counter() {
  const count = useSelector(state => state.counter.value);
  const dispatch = useDispatch();

  return (
    <button onClick={() => dispatch(increment())}>
      {count}
    </button>
  );
}
```

State only flows in one direction. The component never writes to state directly — it dispatches an intent, the reducer produces new state, and React re-renders with that new state.

### Bidirectional Binding Under the Hood

Two-way binding is syntactic sugar over two one-way bindings wired together. Vue's `v-model` is the clearest example:

```html
<!-- This: -->
<input v-model="username">

<!-- Compiles to this: -->
<input :value="username" @input="username = $event.target.value">
```

It's still "state → view" and "view → state" separately — just automated. The framework wires the setter for you. The risk is that this automation can create implicit dependencies that are hard to trace when things go wrong.

Angular's `ngModel` works the same way using a `ControlValueAccessor` interface.

> **Check yourself:** Vue's v-model is "syntactic sugar." What are the two underlying bindings it combines, and what would you need to write manually to achieve the same behavior?

---

## When Each Model Excels

### Unidirectional shines when:
- Application state is complex and mutations need to be auditable (Redux devtools time-travel)
- Multiple components share state and you need a single source of truth
- State changes have side effects that need explicit control
- You want predictability and testability — pure reducers are trivially testable

### Bidirectional shines when:
- Building forms with many fields and simple state
- State is truly local to a single input (no shared consumers)
- Rapid prototyping where boilerplate reduction matters more than traceability

The practical reality: React uses unidirectional flow, but `v-model` on a local form in Vue or `[(ngModel)]` in Angular for a search input is perfectly reasonable — you're not building a distributed state graph, you're wiring a text box.

---

## Flux Architecture

Flux (Facebook, 2014) formalized unidirectional data flow as an architecture pattern before Redux existed. The core insight: in a complex app, you want one predictable direction of change. The Flux cycle:

```
Action → Dispatcher → Store → View → Action → ...
```

Redux simplified this by replacing the Dispatcher with a single pure function (reducer) and collapsing multiple stores into one. The directional guarantee remains: you cannot update state from the view directly.

---

## Gotchas

**1. React's "read-only" input.** In unidirectional flow, an input with `value={state}` but no `onChange` becomes read-only — React controls the DOM value and won't let the user change it because there's no way to propagate the change back. This is intentional and by design, but confusing when you first hit it.

**2. Bidirectional binding in deep watchers.** Angular's two-way binding with deeply nested objects can cause performance issues because change detection must traverse the object graph. React avoids this by enforcing shallow comparison with explicit immutable updates.

**3. Unidirectional doesn't mean synchronous.** Dispatching an action in Redux doesn't immediately update the DOM — the cycle includes the re-render phase. Code that reads state immediately after a dispatch will get the old state.

**4. `v-model` on component props creates implicit contracts.** When a child component uses `v-model`, it's exposing a `modelValue` prop and emitting `update:modelValue` events. This looks like magic but breaks the explicit prop/emit contract. Engineers unfamiliar with the pattern will be confused about where the write is happening.

**5. Circular updates in bidirectional systems.** If state A watches state B and state B watches state A, a change to A triggers B, which triggers A again. Angular 1's digest cycle had a 10-iteration limit specifically to catch infinite loops. Unidirectional systems make this impossible by construction.

---

## Interview Questions

**Q (High): Why did React choose unidirectional data flow instead of two-way binding? What problem does it solve?**

Answer: React's original motivation (per the Flux announcement) was debugging complexity in large applications with Angular 1-style two-way binding. When many components share state through bidirectional bindings, a change in one place ripples through a graph of watchers in ways that are hard to trace. With unidirectional flow, you always know state can only be updated at the source via an explicit action/setter. The view is a pure function of state, not a participant in state mutation. This makes debugging straightforward: you trace state backwards through actions, not through a web of bindings. Time-travel debugging (Redux devtools) is only possible because of this — you can replay actions deterministically.

The trap: Saying "React is just simpler." The answer should name the specific failure mode (watcher cycles, unpredictable mutation) that unidirectional flow solves.

**Q (High): In React, what's the difference between a controlled and an uncontrolled input? Which model represents unidirectional flow?**

Answer: A controlled input has `value` set from React state and an `onChange` handler that updates that state. React owns the value — the DOM reflects what React says. An uncontrolled input has a `defaultValue` but React doesn't control subsequent changes; the DOM owns its own state, accessed via a ref. Controlled inputs are the unidirectional model: state → view is the DOM value; view → state is the onChange dispatch. Uncontrolled inputs are closer to bidirectional — the DOM and component are loosely in sync. In practice, controlled inputs are preferred for anything that needs validation, programmatic reset, or visibility of current form state.

The trap: Saying uncontrolled inputs are "wrong." They're appropriate for simple cases, file inputs (which can't be controlled), and when using a library like react-hook-form that manages its own ref-based state.

**Q (High): What is Flux architecture and how does Redux relate to it?**

Answer: Flux (Facebook, 2014) is an architectural pattern that formalizes unidirectional data flow for complex UIs. The cycle is: UI fires an Action → Dispatcher broadcasts it → Stores update themselves → Views re-render from store state. Redux is a simplified implementation of Flux: it merges multiple stores into one, replaces the Dispatcher with pure reducer functions, and uses middleware for async actions. The core guarantee is the same: state can only change through explicit, serializable actions — never via direct mutation from the view layer.

The trap: Conflating Flux and Redux. They're related but not identical. Flux had multiple stores and a centralized dispatcher; Redux has one store and no dispatcher.

**Q (Medium): How does Vue's v-model work under the hood? Is Vue really a bidirectional data flow system?**

Answer: `v-model` is syntactic sugar for `:value="prop" @input="prop = $event.target.value"`. It's two one-way bindings automated. Vue's *reactivity system* is bidirectional in the sense that it tracks dependencies automatically and re-runs effects when reactive values change. But the data flow direction within the component hierarchy is still expected to be one-way: props flow down, events flow up. `v-model` on a component (not a native input) emits `update:modelValue`, which the parent handles — still explicit event → parent state update. So Vue's macro-level pattern is unidirectional; the reactive system just automates the subscription wiring.

The trap: Saying "Vue is bidirectional." Vue's Composition API and props/emit model are explicitly unidirectional. The reactivity system is what's automatic, not the data flow direction.

**Q (Medium): Can you have time-travel debugging with bidirectional data flow? Why or why not?**

Answer: Not reliably. Time-travel debugging works by replaying a sequence of actions against a reducer to reconstruct any past state. This requires that state changes are: (a) captured as discrete, serializable events, and (b) deterministic given the same starting state and action sequence. Bidirectional binding doesn't guarantee either — state can be mutated from many places without a central action log, and reactive watchers can fire in a non-deterministic order. Redux's strict unidirectional model makes time-travel possible because every state transition is a pure function of (previousState, action).

The trap: Not connecting time-travel to the purity requirement. The key is that reducers are pure functions — same input always produces same output.

---

## Self-Assessment

Before moving on, check off each item you can do WITHOUT looking at this file.

- [ ] Can explain the unidirectional flow cycle (view → action → state → view) without notes
- [ ] Can explain what v-model compiles to and why Vue is still considered unidirectional at the component level
- [ ] Can explain why React's controlled input becomes read-only without an onChange handler
- [ ] Can name the specific bug category (watcher cycles, unpredictable mutation) that unidirectional flow eliminates
- [ ] Can explain why time-travel debugging requires unidirectional flow + pure reducers

---
*Next: Reactivity Models (Signals, Observables, Proxies) — the next layer down: how frameworks actually track which pieces of state a given view depends on, and notify only the right subscribers when it changes.*
