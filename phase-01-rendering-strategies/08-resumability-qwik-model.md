# Resumability (Qwik Model)

## Quick Reference

| Concept | Mechanism | Implication |
|---|---|---|
| Serialized execution state | Component state, listeners, and closures are serialized into HTML | The browser can resume execution without replaying anything |
| No hydration replay | Qwik doesn't re-run component code to discover event handlers | Zero upfront JS execution — O(1) startup regardless of app size |
| Lazy execution | Event handlers download and execute only when the user triggers them | A click on a button downloads only that button's handler code |
| HTML as state container | `data-qwikloader` attributes in HTML encode framework state | No separate JS runtime needed to understand which code handles what |

## What Is This?

Every rendering model covered so far — SSR, SSG, ISR, streaming, islands, progressive rehydration — solves the performance problem by controlling *when* or *how much* of the JavaScript executes. None of them challenge the fundamental assumption: **the browser must re-execute the application code to discover what to do.**

In SSR + hydration, the server renders HTML, sends it, and then the browser runs the *same application code again* to attach event listeners and reconstruct state. This is hydration: re-running the work the server already did to put the browser in the same state.

Resumability eliminates this redundant replay. Instead of re-executing, the browser *resumes* from a serialized state snapshot. The execution state (component tree, event handlers, reactive subscriptions) is serialized into the HTML by the server. When the user interacts, only the specific code for that interaction downloads and runs — no startup cost, no re-execution of the entire app.

```html
<!-- What a Qwik-rendered button looks like in HTML -->
<button on:click="./chunk-abc123.js#buttonClick">
  Click me
</button>
```

The `on:click` attribute encodes exactly where to find the click handler's code. When the user clicks, Qwik's tiny runtime (~1KB) fetches `chunk-abc123.js`, executes `buttonClick`, and responds to the click. No other JavaScript ran before this moment.

> **Check yourself:** In standard SSR hydration, when does the click handler for a button get attached? In Qwik, when does the same handler run?

## Why Does It Exist?

Hydration is fundamentally wasteful for large applications. The cost of hydration scales linearly with app complexity — more components, more event handlers, more reactive subscriptions all mean more work the browser does at startup just to reconstruct state that the server already had.

On a mid-range mobile device, hydrating a large e-commerce SPA can block the main thread for 2–5 seconds. Users see a fully-rendered page that looks interactive but isn't. Progressive hydration and islands chip away at this problem, but they're still bound by the premise that JavaScript must execute to make things interactive.

Qwik's insight, formulated by Miško Hevery (creator of AngularJS): **the problem isn't how much JS runs — it's that any JS must run before the page is interactive.** The goal is O(1) startup time regardless of application size. The solution: serialize the execution graph into HTML so the browser knows what code to run for each interaction without running any code first.

## How It Works

### The Serialization Phase (Server)

When Qwik renders on the server, it does more than produce HTML. It also:
1. Serializes the component tree's state into the HTML (via `data-*` attributes and inline JSON)
2. Records which event handler handles which event on which element
3. Records component boundaries and their reactive subscriptions

```html
<!-- Qwik serializes state into the document -->
<script type="qwik/json">
{
  "ctx": { /* component state tree */ },
  "subs": [ /* reactive subscriptions */ ],
  "objs": [ /* serialized objects */ ]
}
</script>

<button q:id="0" on:click="./q-bundle.js#clickHandler_0">
  Add to Cart
</button>
```

This is a complete record of what event listeners exist, where their code lives, and what state they'll operate on. The browser doesn't need to execute any code to discover this — it's right there in the HTML.

### The QwikLoader (Browser)

Qwik ships a tiny (~1KB) bootstrapper called the QwikLoader. It runs immediately and does one thing: attach a single global event listener that intercepts all interactions (clicks, inputs, keypresses) on the document.

When an interaction occurs:
1. QwikLoader reads the `on:click` attribute of the interacted element
2. The attribute contains a URL fragment pointing to the handler code
3. QwikLoader dynamically imports just that chunk
4. The chunk executes and handles the event

```javascript
// The QwikLoader is conceptually this simple:
document.addEventListener('click', async (event) => {
  const target = event.target.closest('[on:click]');
  if (!target) return;
  
  const handlerRef = target.getAttribute('on:click');
  // handlerRef = './chunk-abc.js#clickHandler'
  
  const [modulePath, exportName] = handlerRef.split('#');
  const module = await import(modulePath);
  await module[exportName](event);
});
```

Nothing in the application runs until a user interaction triggers it. A complex page with 200 interactive components downloads zero component code at startup.

### Fine-Grained Reactivity

Qwik doesn't use a virtual DOM or React-style re-rendering. It uses fine-grained reactivity — when state changes, only the specific DOM node that depends on that state updates. This is serializable: the reactive dependency graph is part of the state that gets serialized into HTML.

```javascript
// Qwik component — reactivity is explicit and serializable
import { component$, useSignal } from '@builder.io/qwik';

// The $ suffix marks functions that can be lazy-loaded
export const Counter = component$(() => {
  const count = useSignal(0);  // serializable reactive state

  return (
    <div>
      <span>{count.value}</span>
      {/* This handler is a separate lazy-loadable chunk */}
      <button onClick$={() => count.value++}>Increment</button>
    </div>
  );
});
```

The `$` suffix on `component$` and `onClick$` is Qwik's optimizer hint: these functions will be split into separate chunks by the Qwik optimizer at build time. The optimizer analyzes the code and extracts each event handler into its own bundle.

> **Check yourself:** Why is the `$` suffix significant at build time? What would happen if Qwik didn't split event handlers into separate chunks?

## The Optimizer

The Qwik Optimizer (a Vite/Rollup plugin) transforms Qwik code at build time:

```javascript
// Before optimization (source code)
const Counter = component$(() => {
  const count = useSignal(0);
  return <button onClick$={() => count.value++}>{count.value}</button>;
});

// After optimization (two separate chunks):

// Chunk 1: counter-component.js (the component factory)
const Counter = component$(Symbol('./onClick-handler.js#clickHandler'));

// Chunk 2: onClick-handler.js (the event handler — only loads on click)
export const clickHandler = (event, count) => {
  count.value++;
};
```

Each handler becomes its own tiny, independently loadable module. The component factory just contains references to these handlers, not their code.

## Resumability vs. Hydration: The Key Difference

| | Hydration (React SSR) | Resumability (Qwik) |
|---|---|---|
| Server output | HTML | HTML + serialized execution state |
| Browser startup JS | Run entire app to discover handlers | ~1KB QwikLoader only |
| When component JS loads | Upfront (full bundle) | On first interaction with that component |
| Time to interactive | After full hydration (O(n) with app size) | Immediately (O(1) regardless of app size) |
| State reconstruction | Re-execute app code | Deserialize from HTML |
| Trade-off | Fast subsequent interactions | First-interaction latency on each component |

The Qwik trade-off: the first time you interact with any given component, there's a small network fetch for that component's code. After that, the code is cached. React's trade-off: all code downloads upfront, all interactions are instant after the initial cost.

## Limitations and Trade-offs

**First-interaction latency.** The first click on a never-used component triggers a dynamic import. On a slow connection, this could take 200–500ms. The component appears frozen until the code arrives. Qwik mitigates this with prefetching strategies (prefetch visible components in idle time) but the latency is structural when there's no cache.

**Serialization constraints.** Not everything can be serialized into HTML. Closures that capture non-serializable values (functions, class instances, DOM references) can't be included in Qwik's state snapshot. Qwik's optimizer handles most cases automatically, but there are patterns that require rethinking.

**Ecosystem immaturity.** Qwik is significantly newer than React, Vue, or Svelte. The ecosystem of libraries, tooling, and community knowledge is smaller. Third-party integrations require Qwik-compatible wrappers.

**Mental model shift.** The `$` suffix convention, explicit serialization boundaries, and fine-grained reactivity are genuinely different from virtual-DOM frameworks. Teams with deep React experience face a real learning curve.

**Not the right choice for all apps.** Qwik's benefits are most pronounced for large, complex pages where hydration cost is measurably high. A simple CRUD app with modest interactivity doesn't benefit enough to justify the ecosystem trade-off.

## Interview Questions

**Q (High): What is the fundamental difference between hydration and resumability?**

Answer: Hydration is the process of re-running application code in the browser to reconstruct execution state that the server already computed — attaching event handlers, recreating reactive subscriptions, restoring component state. It's a replay of work the server already did. The cost scales with app size: more components and handlers mean more replay work. Resumability eliminates the replay entirely. The server serializes the execution state — the component tree, event handler locations, reactive subscriptions — directly into the HTML output. The browser reads this serialized state to know what code handles what, without running any code first. When a user interaction occurs, only the code for that specific handler is fetched and executed. Startup cost is O(1) regardless of app size because no application code runs at startup. The trade-off is a small network fetch on first interaction with each component.

The trap: Vague answers like "Qwik doesn't need hydration." The follow-up is always "why not and what does it do instead." The answer requires explaining serialization of execution state.

---

**Q (High): How does Qwik achieve lazy execution of event handlers, and what role does the Qwik Optimizer play?**

Answer: Qwik achieves lazy execution through two mechanisms working together. At build time, the Qwik Optimizer extracts every event handler marked with the `$` suffix into its own separate chunk (a small JS file with a unique URL). The component references handlers by URL rather than by function reference. In the HTML output, each interactive element has `on:click="./chunk-abc.js#handlerName"` attributes — not inline JavaScript, but pointer to where the code lives. At runtime, a ~1KB QwikLoader script intercepts global DOM events and reads the `on:*` attributes to know where to fetch the handler code. The dynamic import only fires when a user interaction actually occurs. A page with 200 interactive components ships zero component handler code until the user triggers the first interaction.

The trap: Describing this as "similar to code splitting." Regular code splitting still downloads all route code upfront. Qwik's granularity is per-event-handler, and loading is purely triggered by user interaction — far more aggressive than route-based splitting.

---

**Q (Medium): What is Qwik's serialization model and what types of values can and cannot be serialized?**

Answer: Qwik serializes the reactive state graph, event handler references, and component tree into the HTML output via an inline `<script type="qwik/json">` block and `data-*` attributes. The serializer can handle: plain objects, arrays, primitives (strings, numbers, booleans), nested objects and arrays, and references to lazy-loadable functions (serialized as URL references to code chunks). It cannot handle: native functions that aren't Qwik-marked (arbitrary closures that capture non-serializable values), DOM references (can be re-acquired from the DOM when needed), class instances (must be plain objects), and Promise objects. The Qwik Optimizer and TypeScript types enforce serialization boundaries — calling a non-serializable function inside a `$`-marked handler is a type error. This means state design in Qwik requires conscious choices about what state needs to survive the server→browser boundary.

The trap: Not knowing that serialization constraints limit what you can put in Qwik components. Interviewers who ask this are testing whether you understand the implementation trade-off.

---

**Q (Medium): When would you choose Qwik over React SSR, and when would you not?**

Answer: Choose Qwik when: you have a large, complex page (e-commerce, news portal) where hydration cost is measurably hurting TTI on mid-range mobile devices; your team is starting a new project and can learn Qwik without legacy React baggage; the content is predominantly static with scattered interactions. Don't choose Qwik when: you have a large existing React codebase (migration cost is enormous); you need a mature ecosystem of third-party components and library integrations; your app is highly interactive throughout (the first-interaction latency cost per-component accumulates); or your team's React expertise is a competitive advantage and the Qwik learning curve would slow delivery. The honest answer for most teams in 2025: Qwik's concepts are worth understanding, React + React Server Components + good code splitting gets you 80% of the benefit with 0% of the ecosystem risk.

The trap: "Qwik is always better for performance." The first-interaction latency and ecosystem immaturity are real costs.

---

**Q (Low): How does Qwik's fine-grained reactivity differ from React's virtual DOM approach, and why does it matter for resumability?**

Answer: React's virtual DOM model re-renders components — when state changes, React re-runs the component function, produces a new virtual DOM tree, diffs it against the previous tree, and applies the minimum set of DOM mutations. This is inherently tree-based: a state change at any level causes re-evaluation up and down the component tree. Qwik uses fine-grained reactivity: each piece of state has a direct, serializable relationship to the specific DOM nodes it affects. A counter value is subscribed to by exactly the `<span>` that displays it — when the counter increments, only that span's text content updates. No virtual DOM diffing, no component re-execution. This matters for resumability because the reactive subscription graph is the state that needs to be serialized. A virtual DOM approach requires re-executing component code to rebuild the subscription graph; fine-grained reactivity encodes the subscriptions directly in the serialized state, allowing the browser to update the right DOM nodes without running component code first.

The trap: Not connecting the reactivity model to the resumability claim. These aren't independent design choices — fine-grained reactivity is what makes the subscription graph serializable.

---

## Self-Assessment

Before moving on, check off each item you can do WITHOUT looking at the file.

- [ ] Can explain the difference between hydration (re-execute) and resumability (deserialize) in one sentence each
- [ ] Can describe what the QwikLoader does and how it intercepts events without running component code
- [ ] Can explain what the `$` suffix means in Qwik components and what the optimizer does with it
- [ ] Can articulate Qwik's first-interaction latency trade-off vs. React's upfront hydration cost
- [ ] Can name two types of values that cannot be serialized in Qwik's state model
- [ ] Can explain when you would and wouldn't recommend Qwik over React SSR

---
*Phase 1 complete. Next phase: Browser Internals & Rendering Pipeline — understanding what the browser actually does with your HTML, CSS, and JS once it arrives: the event loop, the critical rendering path, and what makes layout and paint expensive.*
