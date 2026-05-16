# Concurrency Scheduling: React Concurrent Mode, scheduler.postTask

## Quick Reference

| API / Concept | What it is | Key capability |
|---|---|---|
| React Concurrent Mode | React's interruptible rendering engine | Long renders yield to user input, preventing INP regressions |
| React Scheduler | Facebook's internal priority queue for tasks | Powers Concurrent Mode; also available as `scheduler` npm package |
| `scheduler.postTask()` | Native browser API for priority-tagged tasks | Schedule work at `user-blocking`, `user-visible`, or `background` priority |
| `isInputPending()` | Browser hint: is there unprocessed user input? | Lets long tasks yield cooperatively without a fixed time budget |
| `requestIdleCallback` | Execute work when the browser is between frames | Original "low priority" API; no priority levels, short deadlines |

---

## What Is This?

Concurrency scheduling is the discipline of deciding **what to run, when, and in what order** on the single main thread — to keep the UI responsive while making progress on expensive work.

JavaScript is single-threaded. The main thread processes events, runs JS, does layout, and paints frames — all in sequence. A long-running task blocks everything until it finishes. The solution isn't always "move it to a Worker" (some work — React rendering, DOM updates — can't leave the main thread). The solution is to break work into smaller chunks and schedule them intelligently, yielding between chunks to let the browser process input and paint.

```
Without scheduling:                With scheduling (time-sliced):
[====Long Task====][paint]         [chunk][yield][input?][chunk][yield][chunk][paint]
                                    ↑ browser can handle clicks here
```

> **Check yourself:** If React rendering is expensive but must touch the DOM, why can't you simply move it to a Web Worker?

---

## Why Does It Exist?

### The 50ms rule and Long Tasks

Chrome's "Long Tasks" are tasks exceeding 50ms (the threshold for perceptible delay). A long synchronous render — a large component tree, a complex diff — blocks input handling, causing high INP (Interaction to Next Paint).

React's pre-Concurrent architecture (React 17 and earlier) was synchronous: once `setState` triggered a render, it ran to completion. A large tree meant a long task. The only mitigations were manual optimizations (`shouldComponentUpdate`, `React.memo`) and splitting work by windowing. There was no structural way to interrupt a render.

React Concurrent Mode (released stable in React 18) rebuilt the renderer to be interruptible — it can pause mid-render, hand control back to the browser, and resume.

---

## React Concurrent Mode

### The Fiber Architecture

React 16 introduced Fibers — a linked-list representation of the component tree that replaces the previous recursive tree walk. A recursive render is uninterruptible (you're inside a function call chain). A Fiber walk is iterative and can be paused between nodes.

Each Fiber node has:
- A reference to the component
- Effect lists (what DOM changes to apply)
- Pointers to parent/child/sibling

The render phase (building the Fiber tree, computing effects) is now interruptible. The commit phase (applying DOM mutations) remains synchronous and uninterruptible — React can't pause halfway through mutating the DOM.

```
Interruptible (render phase):    Uninterruptible (commit phase):
  Work on Fiber A                  Apply all DOM mutations
  → yield if input pending         → layout effects
  Work on Fiber B                  → paint effects
  → yield
  ...
  Reconciliation complete
       ↓
  Commit begins (can't stop)
```

### Transitions

`startTransition` marks an update as non-urgent — React can interrupt rendering it to process more urgent updates:

```js
import { startTransition, useState } from 'react';

function SearchPage() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState([]);

  const handleChange = (e) => {
    const val = e.target.value;
    setQuery(val); // urgent — update the input immediately

    startTransition(() => {
      setResults(computeResults(val)); // non-urgent — can be interrupted
    });
  };

  return (
    <>
      <input value={query} onChange={handleChange} />
      <ResultsList results={results} />
    </>
  );
}
```

Without `startTransition`, typing triggers a full synchronous re-render of `ResultsList` on every keystroke. With it, React renders the new results at lower priority — if the user types another character before it finishes, React discards the in-progress render and starts fresh with the new query.

### `useTransition` and `useDeferredValue`

```js
// useTransition — gives you a pending flag
const [isPending, startTransition] = useTransition();

// useDeferredValue — defer a value update; show stale while computing fresh
const deferredQuery = useDeferredValue(query);
// deferredQuery lags behind query; the component re-renders twice:
// once with the urgent query update, again (deferred) with new results
```

### Suspense and Concurrent rendering

Concurrent Mode enables Suspense for data fetching — a component can "suspend" (throw a Promise) mid-render, and React pauses that subtree until the data is ready, without blocking the rest of the tree:

```js
function UserProfile({ userId }) {
  const user = use(fetchUser(userId)); // throws if not ready
  return <div>{user.name}</div>;
}

// React 18: concurrent rendering means other parts of the UI still render
// while UserProfile is suspended
```

---

## The `scheduler` package (React's internal scheduler)

React's scheduler is available as a standalone package. It implements a priority queue of tasks, each with a priority level:

```js
import { scheduleCallback, ImmediatePriority, UserBlockingPriority,
         NormalPriority, LowPriority, IdlePriority } from 'scheduler';

scheduleCallback(UserBlockingPriority, () => {
  // Runs before normal-priority work
  processInput();
});

scheduleCallback(IdlePriority, () => {
  // Runs only when nothing else is queued
  prefetchData();
});
```

The scheduler uses `MessageChannel` (not `setTimeout`) internally — `postMessage` is the fastest way to queue a macrotask in modern browsers (faster than `setTimeout(fn, 0)` which has a clamped 1–4ms delay).

Tasks are time-sliced: the scheduler gives each task a 5ms time slice (configurable), then yields to let the browser process events and paint. After the browser's frame work is done, the scheduler resumes.

---

## `scheduler.postTask()` — Native Browser Scheduler

The browser's native task priority API, available since Chrome 94:

```js
// Three priority levels:
// 'user-blocking' — critical for user interaction (input response, animation)
// 'user-visible'  — affects what the user sees (rendering, visible transitions)
// 'background'    — work that doesn't affect immediate UX (analytics, prefetch)

const controller = new TaskController({ priority: 'background' });

scheduler.postTask(() => {
  prefetchNextPage();
}, { signal: controller.signal, priority: 'background' });

// Cancel or reprioritize:
controller.abort();
// or:
controller.setPriority('user-visible');
```

`scheduler.postTask()` integrates with the browser's own scheduling — unlike `setTimeout`, the browser can coordinate task priority with its own rendering priorities.

### `scheduler.yield()`

The newest addition (Chrome 124+): an explicit yield point within a long task:

```js
async function processLargeDataset(items) {
  for (let i = 0; i < items.length; i++) {
    process(items[i]);
    if (i % 100 === 0) {
      await scheduler.yield(); // yield to browser, then resume
    }
  }
}
```

`scheduler.yield()` resumes at `user-visible` priority by default — the browser processes input and runs any higher-priority tasks, then resumes your function. This is the cleanest pattern for chunking synchronous loops.

---

## `isInputPending()` — Adaptive Yielding

`navigator.scheduling.isInputPending()` hints whether there's unprocessed user input in the event queue:

```js
function processItems(items) {
  for (const item of items) {
    process(item);
    if (navigator.scheduling.isInputPending()) {
      // Yield immediately — user is trying to interact
      setTimeout(processItems.bind(null, items.slice(currentIndex)), 0);
      return;
    }
  }
}
```

This lets you yield only when needed, avoiding unnecessary scheduling overhead on fast hardware where the full loop completes before input arrives.

> **Check yourself:** Why is `scheduler.postTask()` a better `background` work tool than `setTimeout(fn, 0)`?

---

## requestIdleCallback — The Original Low-priority API

`requestIdleCallback(callback)` runs callback during browser idle time — after the current frame's rendering work is done. The callback receives a `deadline` object:

```js
requestIdleCallback((deadline) => {
  while (deadline.timeRemaining() > 0 && tasks.length > 0) {
    tasks.shift()(); // do a task
  }
  if (tasks.length > 0) requestIdleCallback(processMore); // requeue
}, { timeout: 2000 }); // optional: run even if not idle after 2s
```

Limitations:
- No priority levels — it's just "idle"
- Short deadlines (typically 50ms max)
- Not available in Safari (use a polyfill with `setTimeout`)
- Timing is non-deterministic — can be throttled in background tabs

For most new code, `scheduler.postTask()` with `'background'` priority is the better choice where available.

---

## Gotchas

**1. `startTransition` is not debounce**
Transitions can be interrupted by urgent updates, but they don't batch or deduplicate. If you need debouncing (avoid triggering on every keystroke), you still need `useDeferredValue` or explicit debouncing. `startTransition` is about priority, not timing.

**2. The commit phase is always synchronous**
Even in Concurrent Mode, React cannot interrupt once it starts committing (applying DOM mutations). A commit with thousands of DOM updates is still a long task. Concurrent rendering reduces time-to-first-paint but doesn't eliminate large commit costs.

**3. `requestIdleCallback` is capped at 50ms**
In background tabs, `timeRemaining()` is capped even lower (often 0). Don't schedule meaningful user-visible work in `requestIdleCallback` — it's for true background tasks.

**4. `scheduler.postTask()` is Chrome-only (for now)**
Firefox and Safari do not support `scheduler.postTask()`. Polyfill with the `scheduler` npm package for cross-browser use. `scheduler.yield()` is even newer — don't rely on it without feature detection.

**5. Priority inversion with transitions**
If you wrap too much state in `startTransition`, urgent updates may be delayed because React is preoccupied scheduling transition work. Only mark genuinely non-urgent updates as transitions.

**6. `isInputPending()` can return false negatives**
It only reports inputs that have been dispatched to the browser's input queue — synthetic events, programmatic dispatches, and some touch events may not appear. Treat it as a hint, not a guarantee.

---

## Interview Questions

**Q (High): What is React Concurrent Mode and what problem does it solve?**

Answer: React Concurrent Mode makes React's rendering engine interruptible. Before it, React's render phase was synchronous — once a render started, it ran to completion, blocking the main thread for the duration. On large component trees or complex diffs, this caused high INP: users pressing a key or clicking would wait for the entire render to finish before their input was processed. Concurrent Mode breaks rendering into Fiber-based chunks that can be paused between nodes. When higher-priority work arrives (user input), React pauses the in-progress render, handles the input, and then resumes. The commit phase (DOM mutations) is still synchronous, but the expensive reconciliation phase is now preemptible.

The trap: candidates think Concurrent Mode moves rendering to another thread — it does not. It's still single-threaded; the key is *interruptibility*, not parallelism.

**Q (High): What does `startTransition` do and when should you use it?**

Answer: `startTransition` marks a state update as non-urgent. React renders it at lower priority and will interrupt it if an urgent update (e.g., a keystroke, a click) arrives mid-render. Use it when an update triggers expensive re-rendering that doesn't need to be instantaneous — typically expensive list filtering, search result rendering, or large data transformations. React can abandon the in-progress low-priority render and restart with the latest state, avoiding stale intermediate frames. It pairs with `useTransition` (which gives you an `isPending` flag to show a loading indicator) and `useDeferredValue` (which defers the value used in a subtree, showing stale content while computing fresh).

The trap: candidates think `startTransition` debounces or delays the update. It doesn't — it executes immediately but at lower priority.

**Q (High): What is the difference between `scheduler.postTask()` priorities and how does it compare to `setTimeout`?**

Answer: `scheduler.postTask()` has three priority levels: `user-blocking` (runs as soon as possible, before rendering), `user-visible` (affects what the user sees — default priority), and `background` (runs only when no higher-priority work is pending). `setTimeout(fn, 0)` is clamped to ~4ms minimum delay and runs at an unspecified browser priority, often the same as any other macrotask. `scheduler.postTask()` integrates with the browser's own scheduling decisions — the browser can coordinate its rendering, input handling, and your tasks in a unified priority queue. For background work, `scheduler.postTask({priority: 'background'})` is explicitly deprioritized; `setTimeout(fn, 0)` is not.

**Q (Medium): How does the React Scheduler use `MessageChannel` and why not `setTimeout`?**

Answer: React's scheduler needs a way to yield to the browser and then resume. `setTimeout(fn, 0)` has a minimum 1ms clamp in most environments (4ms after 5 nested calls per spec), which adds up when you're chunking many small tasks. `MessageChannel.port.postMessage()` fires as a macrotask with no minimum delay — typically sub-millisecond. This gives React tighter control over its time slices. The scheduler posts a message and the `onmessage` handler performs the next chunk of work. It's a lower-overhead yield mechanism than `setTimeout`.

**Q (Medium): What is `scheduler.yield()` and how does it differ from `requestIdleCallback`?**

Answer: `scheduler.yield()` explicitly yields execution at the current point in an async function and schedules resumption at `user-visible` priority — the browser processes pending input and higher-priority tasks, then resumes. It's a clean way to chunk a long synchronous loop: `await scheduler.yield()` every N iterations. `requestIdleCallback` runs only during idle time (after all frames are done) at a single unspecified priority, often with a capped time budget (~50ms). `scheduler.yield()` is more predictable — it runs at a defined priority, not "whenever the browser is free." For chunking visible work, `scheduler.yield()` is better; for true background work with no deadline, `requestIdleCallback` is fine.

**Q (Low): What is `isInputPending()` and how does it enable adaptive yielding?**

Answer: `navigator.scheduling.isInputPending()` returns `true` if there's pending unprocessed user input (pointer events, keyboard events) in the browser's event queue. It allows a long-running loop to yield only when a user is actually trying to interact, rather than on a fixed schedule. This is adaptive: on fast hardware with no user interaction, the loop runs to completion with no yield overhead; when a user clicks mid-computation, the loop detects it and yields immediately. It's a hint from the browser's internals — not guaranteed to be exhaustive for all input types — so it should augment, not replace, time-based yield checks.

---

## Self-Assessment

- [ ] Explain why React Concurrent Mode's interruptible rendering is still single-threaded
- [ ] Describe what `startTransition` does and what it doesn't do (not debounce, not async)
- [ ] Name React 18's three low-priority / deferred APIs and the difference between them
- [ ] Explain why the React Scheduler uses `MessageChannel` instead of `setTimeout`
- [ ] Describe the three `scheduler.postTask()` priority levels and a use case for each

---
*Next: OffscreenCanvas — moving Canvas and WebGL rendering entirely off the main thread into a Worker, with zero-copy frame delivery to the screen via `transferControlToOffscreen()`.*
