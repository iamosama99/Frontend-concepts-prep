# Event Loop, Call Stack, Task Queue & Microtask Queue

## Quick Reference

| Concept | What it is | Key behavior |
|---|---|---|
| Call stack | LIFO stack of currently executing function frames | Synchronous; blocks while running |
| Task queue (macrotask) | Queue of callbacks from `setTimeout`, `setInterval`, I/O, UI events | One task dequeued per event loop turn |
| Microtask queue | Queue of `Promise` callbacks, `queueMicrotask`, `MutationObserver` | Entire queue drains after every task, before the next task |
| Event loop | The coordinator that picks tasks and runs them | Runs one task → drains microtasks → renders → repeat |

## What Is This?

JavaScript is single-threaded — only one piece of code runs at any given moment. The event loop is the mechanism that makes a single thread feel like it can handle many things at once: timers, network responses, user clicks, promise resolutions. It does this by queuing work and executing it turn by turn.

At any moment, the browser is doing exactly one of:
1. Executing synchronous JavaScript (the call stack is non-empty)
2. Draining the microtask queue
3. Running a rendering step (style, layout, paint)
4. Picking the next macrotask

None of these overlap. Everything is serial.

```javascript
console.log('1');             // synchronous

setTimeout(() => {
  console.log('4');           // macrotask — queued, runs later
}, 0);

Promise.resolve().then(() => {
  console.log('3');           // microtask — runs before next macrotask
});

console.log('2');             // synchronous

// Output: 1, 2, 3, 4
```

> **Check yourself:** Why does the `setTimeout` with delay `0` still log after the Promise callback, even though `0` means "as fast as possible"?

## Why Does It Exist?

Browsers must handle many concurrent concerns — user input, network responses, animation frames, timers — while running a single-threaded language. The event loop is the scheduling contract that makes this safe and predictable.

The alternative — true multithreading with shared memory — requires locks, mutexes, and race condition handling. JavaScript avoided this intentionally. The event loop gives you concurrency (things can happen "at the same time" from the user's perspective) without parallelism (code never actually runs simultaneously, so you never corrupt shared state with a race).

The two-queue model (macrotask + microtask) emerged from practical needs: Promise resolution callbacks need to run *after the current operation completes* but *before any other unrelated work* (like a timer or click handler) — they're logically part of the current async operation, not a separate task.

## How It Works

### The Call Stack

A LIFO (last in, first out) stack of execution contexts. When a function is called, a new frame is pushed. When it returns, the frame is popped. When the stack is empty, the event loop can pick the next task.

```javascript
function a() {
  b();  // pushes b
}
function b() {
  console.log('hello');  // pushes console.log, pops it
}  // pops b
a();  // pushes a, which calls b, which runs, all pops — stack empty
```

A stack overflow (`Maximum call stack size exceeded`) happens when function calls recurse so deeply the stack exceeds the engine's limit — each recursive call adds a frame without anything popping.

### Macrotask Queue (Task Queue)

Sources: `setTimeout`, `setInterval`, `setImmediate` (Node.js), I/O callbacks, UI events (click, keydown, scroll), `MessageChannel`, `postMessage`.

The event loop picks **one macrotask per turn**, runs it to completion (until the call stack is empty), then moves on.

```javascript
setTimeout(() => console.log('task 1'), 0);
setTimeout(() => console.log('task 2'), 0);
// These are two separate tasks — each gets its own event loop turn
// Between them, the browser can render if needed
```

"Run to completion" is the key invariant: once a task starts executing, it runs until the stack empties. No other task can interrupt it.

### Microtask Queue

Sources: `Promise.then/catch/finally`, `queueMicrotask()`, `MutationObserver` callbacks, `async/await` continuations.

After every task (or after the initial synchronous code completes), the event loop drains the **entire** microtask queue before doing anything else — including rendering and picking the next macrotask. "Drains entirely" means: even if a microtask enqueues another microtask, that new microtask runs before control returns to the event loop.

```javascript
function recursive() {
  Promise.resolve().then(recursive);  // infinite microtask loop
}
recursive();
// The browser freezes — the microtask queue never empties,
// so the event loop never renders or processes input
```

This is why `queueMicrotask` can starve rendering: an infinite chain of microtasks never lets the render step run.

### The Full Loop

```
event loop turn:
  1. Dequeue one macrotask → run it (call stack runs to empty)
  2. Drain microtask queue:
       while (microtask queue not empty):
         dequeue microtask → run it
         (new microtasks added here run in this same drain)
  3. If rendering is due (≈16ms has passed for 60fps):
       style → layout → paint → composite
  4. Go to step 1
```

The rendering step only happens when the browser decides it's needed (approximately every 16ms for 60fps displays). Between two macrotasks, there may be zero rendering steps or one — but there's always a complete microtask drain.

> **Check yourself:** A `MutationObserver` callback is a microtask. If a DOM mutation fires during step 3 (rendering), when does the MutationObserver callback run?

### `async/await` in Terms of the Event Loop

`async/await` is syntactic sugar over Promises. Every `await` is a microtask boundary:

```javascript
async function fetchData() {
  console.log('A');          // synchronous — current call stack
  const data = await fetch('/api');  // suspends here, fetch is a macrotask
  console.log('B');          // microtask continuation — runs after fetch resolves
}

fetchData();
console.log('C');            // synchronous — runs before B

// Output: A, C, B
```

When execution hits `await`, the async function suspends and returns a Promise. The remaining code after `await` is registered as a microtask `.then` callback. Control returns to the caller immediately (hence 'C' logs before 'B').

Each `await` point yields the microtask queue: the continuation runs as a microtask after the awaited Promise resolves.

## `requestAnimationFrame` and the Rendering Pipeline

`requestAnimationFrame` (rAF) callbacks run as part of the rendering step — after microtasks but before the actual style/layout/paint. They're neither macrotasks nor microtasks; they're render tasks.

```javascript
requestAnimationFrame(() => {
  // Runs just before the browser paints
  // Guaranteed to have a consistent timestamp
  // Ideal for reading layout (no forced reflow risk) and queuing DOM mutations
});
```

The rAF callback runs once per frame, synchronized to the browser's rendering rate. Using `setTimeout(fn, 16)` to animate is wrong — it has timing drift and doesn't know when the browser will actually render. rAF fires at exactly the right moment.

## Common Misconceptions and Gotchas

**`setTimeout(fn, 0)` is not "immediate."** A 0ms timeout still goes into the macrotask queue. All pending microtasks drain first. All pending rendering steps happen first. The actual minimum delay is browser-dependent (4ms in some browsers after nesting depth > 4).

**Long synchronous tasks block everything.** A single synchronous task that runs for 200ms (parsing a large JSON, sorting a large array) blocks the event loop for 200ms. No rendering, no input handling, no timers. This is the cause of "jank" — use Web Workers for CPU-heavy work.

**Microtasks can delay rendering.** If your code creates long chains of microtasks (Promise chains with heavy computation in `.then` callbacks), the microtask drain can take so long that the rendering step is delayed, causing dropped frames.

**`Promise` constructor executor is synchronous.** The code inside `new Promise((resolve, reject) => { ... })` runs immediately, synchronously. Only `.then` and `.catch` callbacks are microtasks.

```javascript
new Promise((resolve) => {
  console.log('A');  // synchronous — in the constructor
  resolve();
}).then(() => {
  console.log('C');  // microtask
});
console.log('B');  // synchronous — runs before microtask

// Output: A, B, C
```

**Task queue vs. microtask queue priority:** Microtasks always run before the next macrotask. This is non-negotiable. A macrotask enqueued during microtask processing must wait for all current and newly-enqueued microtasks to drain.

**`queueMicrotask` vs. `Promise.resolve().then`:** Functionally equivalent for scheduling microtasks. `queueMicrotask` is cleaner when you don't need a Promise value, and has slightly less overhead.

## Interview Questions

**Q (High): Why do microtasks run before the next macrotask, and what are the practical implications?**

Answer: The event loop specification requires that the microtask checkpoint fires after every task and after the initial synchronous script. The microtask queue must drain completely before the event loop proceeds to the next macrotask or rendering step. This is intentional: Promises were designed to provide consistent async sequencing — a Promise resolution should run "as soon as possible" relative to the current operation, before any unrelated work. Practical implications: (1) Promise callbacks run before any `setTimeout`, even `setTimeout(fn, 0)`. (2) You can queue multiple microtasks from within a microtask and they all run before anything else. (3) An infinite Promise chain (a microtask that always queues another microtask) will freeze the browser — the event loop can never advance to render or process input. (4) DOM mutations observed by `MutationObserver` trigger callbacks during the microtask drain, so they run before the next paint.

The trap: Thinking `setTimeout(fn, 0)` runs before Promise callbacks because "0ms means immediately." `setTimeout` is a macrotask regardless of its delay value.

---

**Q (High): What happens to the rendering pipeline when a long synchronous JavaScript task runs?**

Answer: The rendering step (style recalculation, layout, paint) only runs when the call stack is empty and the microtask queue is empty — it's part of the event loop after tasks complete. A synchronous JavaScript task blocks the call stack for its entire duration. During that time, the event loop can't advance, so no rendering steps run, no input events process, and no timers fire. If a task takes 200ms and the browser targets 60fps (one frame every ~16ms), it misses approximately 12 frames — the page is visually frozen and unresponsive to input. This is why heavy synchronous operations (sorting large arrays, parsing large JSON, running complex algorithms) should be moved to Web Workers, broken into smaller chunks using `setTimeout` or scheduler.yield(), or pushed to build time.

The trap: Thinking `requestAnimationFrame` can interrupt a long task. It can't — rAF callbacks are part of the rendering pipeline, which only runs when the stack is clear.

---

**Q (High): Explain `async/await` in terms of the event loop. Where does execution go after an `await` expression?**

Answer: `async/await` compiles to Promise chains. When an `await` expression is reached, the async function suspends: the code after the `await` is registered as a `.then` microtask callback on the awaited Promise, and the function returns control to its caller. The call stack unwinds back to whoever called the async function. When the awaited Promise resolves, the continuation (code after `await`) is added to the microtask queue and runs as a microtask in the next microtask drain — which happens after the current task or synchronous code completes. Each `await` point creates one microtask boundary. If you `await` a `setTimeout`-backed Promise, you're crossing from the microtask queue to the macrotask queue — the continuation won't run until the setTimeout fires and that macrotask completes.

The trap: Thinking `await` literally pauses execution of the thread. It suspends the async function and returns control to the caller immediately. "Pausing" is scoped to that specific function.

---

**Q (Medium): What is the difference between `setTimeout(fn, 0)`, `Promise.resolve().then(fn)`, and `requestAnimationFrame(fn)` in terms of when `fn` executes?**

Answer: `Promise.resolve().then(fn)` schedules `fn` as a microtask — it runs at the end of the current task, before any rendering or the next macrotask. `setTimeout(fn, 0)` schedules `fn` as a macrotask — it runs in a future event loop turn, after any pending microtasks drain and potentially after a rendering step. `requestAnimationFrame(fn)` schedules `fn` to run just before the next browser paint — it's neither a microtask nor a regular macrotask but a render task. It's throttled to the display's refresh rate and has a precise timestamp. Order if all three are queued simultaneously: Promise microtask first, then the rAF callback when the next frame is due, then the setTimeout macro task (which might be the same frame or a later one).

The trap: Placing rAF between microtasks and macrotasks in the wrong order. rAF fires just before paint, which is after microtasks drain but the timing relative to macrotasks depends on whether the browser decides to render in this event loop turn.

---

**Q (Medium): How can a Promise chain cause dropped animation frames, and how do you prevent it?**

Answer: Microtasks drain completely before the rendering step runs. A long chain of `.then` callbacks, each doing some computation, can delay the rendering step if the total microtask drain time exceeds 16ms (one frame budget at 60fps). For example: a recursive Promise chain that processes 10,000 items, one per `.then`, blocks the rendering step until all 10,000 resolve. Prevention: (1) Move heavy computation to a Web Worker — runs in parallel, doesn't touch the main thread at all. (2) Use `scheduler.yield()` (Chrome) or `setTimeout(fn, 0)` to break computation into macrotasks — each chunk lets the render step run between them. (3) Use `requestIdleCallback` for non-urgent computation — runs only when the browser has idle time after rendering.

The trap: Assuming that because Promises are async, they don't block rendering. Microtasks are "async" relative to macrotasks but synchronous relative to rendering — they block the render step.

---

**Q (Low): What does "run to completion" mean and why is it important for correctness?**

Answer: "Run to completion" means once a task starts executing on the call stack, it runs until the stack is empty — no other task can interrupt it. There are no pre-emptive context switches in JavaScript's execution model. This guarantee means: (1) You never need locks or mutexes to protect shared data — a function reading and writing a variable cannot be interrupted mid-read by another function that also modifies that variable. (2) State transitions in an event handler are atomic from the perspective of other JavaScript. You don't need to worry about another click handler running while your current one is partially complete. (3) You can model async operations with Promises safely — intermediate state between async operations is never observed by other code unless you explicitly `await`. The cost of this guarantee is that long tasks block the entire thread.

The trap: Comparing JavaScript's run-to-completion model to multithreaded race conditions. The whole point is that JavaScript *doesn't have* those races.

---

## Self-Assessment

Before moving on, check off each item you can do WITHOUT looking at the file.

- [ ] Can describe the event loop in one sentence (task → microtask drain → optional render → repeat)
- [ ] Can explain why `setTimeout(fn, 0)` runs after `Promise.resolve().then(fn)`
- [ ] Can trace the output order of a code snippet mixing synchronous code, Promises, and setTimeout
- [ ] Can explain what happens to rendering when a long synchronous task runs
- [ ] Can explain why an infinite microtask chain freezes the browser
- [ ] Can describe what `await` does in terms of the event loop (suspends function, code after await is a microtask)

---
*Next: Critical Rendering Path — how the browser turns the HTML, CSS, and JS the event loop just ran into pixels on screen.*
