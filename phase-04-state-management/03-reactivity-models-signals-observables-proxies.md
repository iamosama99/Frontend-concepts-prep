# Reactivity Models: Signals, Observables (RxJS), Proxies

## Quick Reference

| Model | Pull vs. Push | Granularity | Sync/Async | Used in |
|-------|--------------|-------------|------------|---------|
| Signals | Push (fine-grained) | Per-value | Synchronous | Solid, Preact, Angular 17+, Vue Refs |
| Observables (RxJS) | Push (stream-based) | Per-event | Async-first | Angular, any app with complex event composition |
| Proxies | Push (transparent) | Per-property | Synchronous | Vue 3 reactivity, MobX 6 |

---

## What Is This?

Reactivity is the mechanism by which a UI automatically updates when the data it depends on changes. The question is: how does the framework know *which* parts of the view depend on *which* parts of the state, and how does it propagate changes efficiently?

Three distinct models answer this differently:

- **Signals**: fine-grained reactive primitives that track reads and writes explicitly, updating only the exact subscribers that depend on a given value
- **Observables**: lazy push-based sequences of values over time, composable with operators, suited for async event streams
- **Proxies**: transparent wrappers around plain objects that intercept property reads and writes to build dependency graphs automatically

Each model has a different mental model for what "reactive" means and different performance characteristics.

> **Check yourself:** What is the fundamental problem all three models are solving? (Hint: what would you have to do manually without any of them?)

---

## Why Does It Exist?

Without a reactivity system, you'd re-render everything on every state change — which is React's VDOM model in its simplest form. This works because the VDOM diff is fast, but it still means re-running all render functions every time anything changes, even if 99% of the UI is unaffected.

The motivation behind fine-grained reactivity (signals and proxies) is to skip the VDOM entirely: track exactly which DOM nodes depend on which pieces of state, and update only those nodes when that state changes. No diffing. No re-running component functions. Surgical updates.

Observables solve a different problem: they're about *streams of values over time*, not a single reactive value. They excel at composing complex async flows — debouncing input, combining multiple event sources, canceling stale requests — in a declarative, composable way.

---

## How It Works

### Signals

A signal is a reactive cell holding a single value. Reading a signal inside a reactive context (a component, a computed, an effect) registers that read as a dependency. When the signal's value changes, only the registered dependents re-run.

```js
// Solid.js signals
import { createSignal, createEffect, createMemo } from 'solid-js';

const [count, setCount] = createSignal(0);

// This effect tracks count — it re-runs only when count changes
createEffect(() => {
  console.log('count is', count()); // Note: called as function, not .value
});

// Computed/derived signal — recalculates only when count changes
const doubled = createMemo(() => count() * 2);

setCount(5); // triggers the effect and recomputes doubled
```

The key difference from React: in Solid, components are functions that run *once* to set up subscriptions, not on every re-render. Only the fine-grained signal reads inside JSX update — not the component function itself.

**Preact Signals** bring this model into React-compatible environments:

```js
import { signal, computed, effect } from '@preact/signals-react';

const count = signal(0);
const doubled = computed(() => count.value * 2);

effect(() => {
  console.log(count.value); // re-runs when count changes
});

// In a React component — only the span re-renders, not the whole component
function Counter() {
  return <span>{count}</span>; // pass signal directly, not count.value
}
```

**Angular Signals** (Angular 17+) integrate the same model into the Angular change detection system, allowing fine-grained updates without triggering the full zone.js change detection cycle.

### How Signals Track Dependencies

Signals use a global "current context" tracker. When a signal is read inside a reactive context (effect, computed), it registers the current context as a subscriber. When the signal writes a new value, it notifies all subscribers:

```js
// Conceptual implementation
let currentObserver = null;

function createSignal(initialValue) {
  let value = initialValue;
  const subscribers = new Set();

  const read = () => {
    if (currentObserver) subscribers.add(currentObserver);
    return value;
  };

  const write = (newValue) => {
    value = newValue;
    subscribers.forEach(sub => sub());
  };

  return [read, write];
}

function createEffect(fn) {
  const effect = () => {
    currentObserver = effect;
    fn(); // reads happen here, registering subscriptions
    currentObserver = null;
  };
  effect();
}
```

This is automatic dependency tracking — no need to declare dependencies like React's `useEffect([dep1, dep2])`.

> **Check yourself:** In Solid.js, if a component function runs once and sets up subscriptions, what happens when a signal inside a conditional branch changes? What's the risk if the condition changes?

### Observables (RxJS)

An Observable is a lazy, cancellable, composable push-based stream. It produces zero or more values over time. Nothing happens until you subscribe.

```js
import { fromEvent, debounceTime, distinctUntilChanged, switchMap } from 'rxjs';
import { ajax } from 'rxjs/ajax';

const searchInput = document.getElementById('search');

fromEvent(searchInput, 'input').pipe(
  debounceTime(300),              // wait for pause in typing
  distinctUntilChanged(),         // ignore if same value
  switchMap(event =>              // cancel previous request, start new one
    ajax.getJSON(`/api/search?q=${event.target.value}`)
  )
).subscribe(results => {
  renderResults(results);
});
```

Key operators:
- `map`, `filter`, `reduce` — transform values like array methods, but lazily
- `mergeMap` — subscribe to inner observable, merge all results
- `switchMap` — subscribe to inner observable, cancel previous (used for search)
- `concatMap` — queue inner observables, preserve order
- `debounceTime`, `throttleTime` — rate limiting
- `combineLatest`, `zip`, `merge` — combining streams

**Subject** is an Observable that is also an Observer — it's both a stream and a trigger:

```js
const subject = new Subject();

subject.subscribe(val => console.log('A:', val));
subject.subscribe(val => console.log('B:', val));

subject.next(1); // logs "A: 1" and "B: 1"
subject.next(2); // logs "A: 2" and "B: 2"
```

`BehaviorSubject` holds the current value and immediately emits it to new subscribers — closest to a signal in RxJS.

Observables are **not** ideal for simple reactive state. They shine when you have:
- Multiple async sources that need combining
- Rate limiting, deduplication, or transformation of event streams
- Cancellation logic (switchMap cancels in-flight requests)
- Long-lived data streams (WebSocket messages, polling)

### Proxies

ES6 Proxies intercept property access on objects. Vue 3's reactivity system uses this to automatically track dependencies without requiring explicit signal wrappers.

```js
// Vue 3 reactive under the hood
function reactive(target) {
  return new Proxy(target, {
    get(target, key, receiver) {
      track(target, key); // register dependency
      return Reflect.get(target, key, receiver);
    },
    set(target, key, value, receiver) {
      const result = Reflect.set(target, key, value, receiver);
      trigger(target, key); // notify subscribers
      return result;
    },
  });
}

const state = reactive({ count: 0 });

effect(() => {
  console.log(state.count); // reading count → tracked
});

state.count = 1; // setting count → triggers the effect
```

The advantage over signals: you work with plain objects. No `.value`, no `count()` call syntax. You read `state.count`, not `count()`. The Proxy intercepts transparently.

The one limitation of Proxy-based reactivity: primitives can't be wrapped directly — `reactive(42)` doesn't work because Proxy only wraps objects. Vue handles this with `ref()`, which boxes a primitive into `{ value: ... }`. Dynamic property addition is fully supported — Vue 3 Proxies intercept *all* property accesses including new keys, which is precisely what fixed Vue 2's `Object.defineProperty` limitation (where properties added after initialization bypassed reactivity and required `Vue.set()` as a workaround).

**MobX** also uses Proxies (or `Object.defineProperty` in older versions) to make class fields reactive:

```js
import { makeAutoObservable } from 'mobx';

class Store {
  count = 0;

  constructor() {
    makeAutoObservable(this); // wraps every property with Proxy-based reactivity
  }

  increment() {
    this.count++; // mutation triggers any observer that read this.count
  }
}
```

---

## Comparison: When to Use Each

| Scenario | Best fit |
|----------|----------|
| Simple reactive state in a component | Signals or useState |
| Fine-grained DOM updates without VDOM | Signals (Solid, Preact) |
| Complex async event composition | Observables (RxJS) |
| Large mutable state graph on plain objects | Proxies (Vue 3, MobX) |
| Canceling in-flight requests on input | switchMap (Observable) |
| Angular application | Both — Signals for state, RxJS for async |

---

## Gotchas

**1. Signals lose track in conditionals.** If a signal read is inside a conditional that was false on first run, the dependency is never registered. When the condition later becomes true, the effect won't re-run. Solid's `createMemo` and `Show` component handle this correctly; hand-rolled conditionals don't.

**2. Observables are lazy.** An observable does nothing until subscribed. Forgetting to subscribe is a common bug — the code looks right but no side effects happen.

**3. Memory leaks from unsubscribed Observables.** In Angular, if you subscribe to an Observable in a component without unsubscribing in `ngOnDestroy`, the callback continues running after the component is destroyed. The `async` pipe handles this automatically.

**4. Proxy cannot make primitives reactive.** `reactive(42)` doesn't work — Proxy only wraps objects. This is why Vue has `ref()` (which wraps a primitive in `{ value: ... }`) alongside `reactive()`.

**5. Signal granularity can backfire.** Extremely fine-grained signal graphs with thousands of dependencies can have higher overhead than a coarser VDOM diff. Signals win at scale for targeted updates; for very frequent bulk updates, the tracking overhead accumulates.

**6. `switchMap` vs. `mergeMap` confusion.** `switchMap` cancels the previous inner observable on each new emission — correct for search (discard stale results). `mergeMap` runs all concurrently — correct for independent parallel requests. Using `mergeMap` for search causes race conditions where slow responses from stale queries overwrite fresh results.

---

## Interview Questions

**Q (High): What is a signal, and how does it differ from React's useState?**

Answer: A signal is a fine-grained reactive primitive that automatically tracks which parts of the UI depend on it and surgically updates only those parts when it changes. React's `useState` triggers a full component re-render whenever state changes — then a VDOM diff determines what actually changed in the DOM. Signals skip the VDOM entirely: they track reads at the individual property level, so when a signal's value changes, only the specific DOM expressions that read that signal update, without re-running the component function. In Solid.js, a component function runs once; in React, it runs on every re-render. Signals are more efficient for frequently-changing values in large UIs; React's model is simpler to reason about and has a rich ecosystem.

The trap: Saying signals are "just like React state but faster." The architecture is fundamentally different — no VDOM, no component re-renders.

**Q (High): What is an Observable (RxJS) and when would you choose it over a signal or useState?**

Answer: An Observable is a lazy push-based stream that produces values over time. You choose it when you need to *compose* multiple async sources — debouncing, filtering, rate-limiting, combining, or canceling streams. A classic example: search-as-you-type needs `debounceTime` (rate limiting), `distinctUntilChanged` (skip duplicates), and `switchMap` (cancel stale requests). Doing this imperatively is possible but verbose. RxJS operators compose these declaratively in a pipeline. Signals or useState are the better fit for reactive *state* that holds a single current value; Observables are the better fit for *event streams* that need transformation and composition.

The trap: Recommending RxJS for simple state. "An Observable of a single count value" is overkill — that's what BehaviorSubject awkwardly approximates.

**Q (High): How does Vue 3's Proxy-based reactivity work? How is it better than Vue 2's Object.defineProperty approach?**

Answer: Vue 3 wraps reactive objects in ES6 Proxies that intercept property gets (to track dependencies) and sets (to trigger updates). When a component renders, Vue records every property access via the get trap — that component is now subscribed to those properties. When a property is set, the set trap triggers re-rendering of subscribed components. Vue 2 used `Object.defineProperty` to define getters and setters for each known property at creation time. The problem: `Object.defineProperty` only works for properties that exist at definition time. Adding a new property to a reactive object (`this.newProp = value`) bypassed the reactivity system entirely, requiring `Vue.set()` as a workaround. Proxies intercept *all* property accesses, including new ones, solving this completely.

The trap: Not knowing about the Vue 2 limitation and why Proxy fixed it. This is a common interview angle for Vue.

**Q (Medium): What is the difference between switchMap, mergeMap, and concatMap? When does the choice matter?**

Answer: All three take an emission from an outer Observable and map it to an inner Observable. They differ in how they handle overlapping inner observables. `switchMap` cancels the previous inner observable when a new emission arrives — essential for search-as-you-type where you want to discard stale results. `mergeMap` runs all inner observables concurrently and merges their output — appropriate for parallel, independent requests. `concatMap` queues inner observables and runs them one at a time in order — appropriate for operations that must be sequential (e.g., form submission retry). Using `mergeMap` for search is a common race condition bug: a slow response from an earlier query can arrive after a newer one and overwrite correct results.

The trap: Treating them as interchangeable. The wrong choice for search is a genuine race condition in production.

**Q (Medium): Why are Observables lazy? What does that mean in practice?**

Answer: An Observable is a description of a computation, not the computation itself. Creating an Observable with `new Observable(subscriber => ...)` or `fromEvent(...)` doesn't execute the callback. The callback runs only when `.subscribe()` is called. This means: (1) multiple subscriptions each get their own independent execution of the source; (2) an Observable can be passed around and composed without side effects until someone subscribes; (3) unsubscribing cancels the underlying operation (e.g., an in-flight HTTP request or an interval timer). In practice, forgetting to subscribe is a silent bug — the pipeline is wired up but nothing executes.

The trap: Confusing Observable creation with execution. A common mistake: `const search$ = ajax.getJSON(url)` and then never calling `.subscribe()`, or forgetting to pipe the Angular `async` template pipe.

**Q (Low): What is automatic dependency tracking, and how do signals implement it?**

Answer: Automatic dependency tracking means you don't have to declare what a computation depends on — the system figures it out by observing which signals are read during execution. The implementation uses a global "current observer" variable. When an effect or computed value runs, it sets itself as `currentObserver`. When a signal is read, it checks `currentObserver` and registers it as a subscriber. When the signal changes, it notifies all registered subscribers. This is how Solid, Vue, and MobX work. React's `useEffect` deps array is manual dependency tracking — you tell React what to watch. Automatic tracking eliminates the bugs that come from stale deps arrays (not including a dep that changed) or over-deps (including a dep that doesn't affect the output, causing unnecessary re-runs).

The trap: Not knowing how the mechanism works. Interviewers at companies using Solid/Vue/MobX expect you to understand why you don't need a deps array.

---

## Self-Assessment

Before moving on, check off each item you can do WITHOUT looking at this file.

- [ ] Can explain how signals track dependencies (global currentObserver pattern)
- [ ] Can explain the difference between switchMap, mergeMap, and concatMap with a concrete use case for each
- [ ] Can explain why Vue 3 uses Proxy instead of Object.defineProperty and what bug it fixes
- [ ] Can choose between signals, observables, and proxies for a given use case
- [ ] Can explain why Observables are lazy and what happens if you forget to subscribe

---
*Next: Global Store vs. Prop Drilling vs. Context/DI — now that we understand how reactivity works, we look at where state lives and how it's shared across a component tree.*
