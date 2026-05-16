# Web Workers & Dedicated Workers

## Quick Reference

| Concept | What it is | Why it matters |
|---|---|---|
| Web Worker | JS runtime on a separate OS thread | Long tasks don't block the main thread or UI |
| Dedicated Worker | Worker owned by exactly one page | Most common type; closed when page closes |
| `postMessage` | Structured-clone-based message channel | Only way to communicate; no shared memory by default |
| Transferable objects | ArrayBuffer, OffscreenCanvas, MessagePort | Zero-copy hand-off; the sender loses access |
| Worker scope | `DedicatedWorkerGlobalScope`, not `window` | No DOM, no `document`, limited APIs |

---

## What Is This?

A Web Worker is a script that runs in a background thread — completely separate from the main thread where your JavaScript normally executes. The browser gives it its own V8 (or SpiderMonkey) context, its own call stack, its own event loop, and its own memory heap.

```js
// main.js
const worker = new Worker('./worker.js');
worker.postMessage({ numbers: [1, 2, 3, 4, 5] });
worker.onmessage = (e) => console.log('Sum:', e.data);

// worker.js
self.onmessage = (e) => {
  const sum = e.data.numbers.reduce((a, b) => a + b, 0);
  self.postMessage(sum);
};
```

A **dedicated worker** is the most common variant: it's bound 1-to-1 with the page that created it and gets terminated when that page closes or calls `worker.terminate()`. It contrasts with Shared Workers (shared across tabs) and Service Workers (network proxy, independent lifecycle) — those are later topics.

> **Check yourself:** If a Web Worker runs on a separate thread, why can't it access the DOM?

---

## Why Does It Exist?

JavaScript was designed to be single-threaded to avoid the complexity of concurrent DOM mutations. But that design choice has a hard cost: any computation that takes too long *on the main thread* starves the browser of rendering time.

The browser has one main thread responsible for:
- Parsing HTML and running your JS
- Responding to user events (clicks, keyboard)
- Running layout, paint, and compositing

A 200ms computation blocks all of the above. A sort over 1M records, a cryptographic hash, a video frame decode — these are real operations that can freeze your UI.

Workers were introduced (2009, HTML5 spec) to let you move expensive computation off the main thread without redesigning the whole language. The constraint — no shared memory, message passing only — is a deliberate choice to make concurrency safe without locks.

---

## How It Works

### Thread model

Each Worker gets its own OS thread (or a thread-pool slot, depending on browser implementation). V8 creates an Isolate per worker — completely separate heap, no shared GC, no shared object references.

This is why you can't just do:

```js
// This would be a race condition if it could work, but it can't
worker.sharedArray.push(item); // WRONG: no shared object references
```

### The message channel

Communication happens through a structured-clone copy:

```
Main thread                   Worker thread
     │                             │
     │──postMessage(data)──────────▶│  Browser serializes with structured clone
     │                             │  Worker gets a deep copy, not a reference
     │◀──postMessage(result)────────│
     │                             │
```

Structured clone handles most data types — plain objects, arrays, Dates, ArrayBuffers, Blobs, Maps, Sets, even circular references. It cannot clone functions, DOM nodes, or prototypes.

### Transferables: zero-copy handoff

Structured clone copies data by value, which is fine for small payloads but expensive for large `ArrayBuffer`s. Transferables solve this:

```js
const buffer = new ArrayBuffer(1024 * 1024 * 32); // 32MB

// Structured clone: copies 32MB — slow
worker.postMessage({ buffer });

// Transferable: transfers ownership — O(1), main thread can no longer access buffer
worker.postMessage({ buffer }, [buffer]);
console.log(buffer.byteLength); // 0 — detached
```

The buffer is *transferred*, not copied. The main thread's reference is detached. This is the only way to pass large typed arrays efficiently.

### Worker lifecycle

```
new Worker(url)
    → Browser fetches the script (same origin or CORS)
    → Creates new thread + Isolate
    → Worker starts executing script top-to-bottom
    → Worker enters its own event loop (listens for messages)

worker.terminate() or page unloads
    → Thread is killed immediately (no cleanup in worker)
```

The worker script URL must be same-origin, OR the server must allow cross-origin with appropriate CORS headers. Inline workers are possible via `Blob` URLs:

```js
const code = `self.onmessage = (e) => self.postMessage(e.data * 2);`;
const blob = new Blob([code], { type: 'application/javascript' });
const worker = new Worker(URL.createObjectURL(blob));
```

### Worker APIs

Workers have access to a subset of browser APIs:
- `fetch`, `XMLHttpRequest` — full network access
- `setTimeout`, `setInterval`, `queueMicrotask`
- `WebSockets`, `IndexedDB`, `Cache API`
- `crypto`, `TextEncoder/Decoder`, `URL`
- `importScripts(url)` — classic workers only; ESM workers use `import`

No access to:
- `window`, `document`, `navigator.geolocation` (most navigator properties)
- DOM APIs of any kind
- `localStorage` (synchronous; would block; IndexedDB is the alternative)

### Module workers

Classic workers use `importScripts()`. Module workers (Chrome 80+, Firefox 114+) let you use ES module syntax:

```js
const worker = new Worker('./worker.js', { type: 'module' });

// worker.js
import { expensiveComputation } from './math.js';
self.onmessage = async (e) => {
  const result = await expensiveComputation(e.data);
  self.postMessage(result);
};
```

Module workers respect the module graph, support static `import`, and don't need `importScripts`. They also have stricter CORS requirements for imports.

> **Check yourself:** Why does transferring an ArrayBuffer detach the original reference — and why is that the right design choice?

---

## Practical Patterns

### Worker pool

One worker per heavy task is wasteful on startup overhead. A pool keeps N workers alive and queues tasks to idle ones:

```js
class WorkerPool {
  constructor(url, size = navigator.hardwareConcurrency) {
    this.workers = Array.from({ length: size }, () => ({
      worker: new Worker(url, { type: 'module' }),
      busy: false,
    }));
    this.queue = [];
  }

  run(data, transfer = []) {
    return new Promise((resolve, reject) => {
      const idle = this.workers.find(w => !w.busy);
      if (idle) {
        this._dispatch(idle, data, transfer, resolve, reject);
      } else {
        this.queue.push({ data, transfer, resolve, reject });
      }
    });
  }

  _dispatch(slot, data, transfer, resolve, reject) {
    slot.busy = true;
    slot.worker.postMessage(data, transfer);
    slot.worker.onmessage = (e) => {
      resolve(e.data);
      slot.busy = false;
      if (this.queue.length) {
        const next = this.queue.shift();
        this._dispatch(slot, next.data, next.transfer, next.resolve, next.reject);
      }
    };
    slot.worker.onerror = (e) => {
      reject(e);
      slot.busy = false;
    };
  }
}
```

### Comlink (ergonomic RPC over workers)

The raw `postMessage` API is tedious. [Comlink](https://github.com/GoogleChromeLabs/comlink) wraps it in a Proxy that makes worker calls look like async function calls:

```js
// worker.js
import { expose } from 'comlink';
const api = { sort: (arr) => arr.sort((a, b) => a - b) };
expose(api);

// main.js
import { wrap } from 'comlink';
const worker = new Worker('./worker.js', { type: 'module' });
const api = wrap(worker);
const sorted = await api.sort([3, 1, 2]); // feels like a normal async call
```

Comlink handles message serialization, promise wiring, and error propagation — the underlying mechanism is still `postMessage`.

---

## When to Actually Use Workers

Workers are not free. Spawning a worker has overhead (thread creation, script fetch, JS parse). The rule of thumb:

- **Use a worker** when a task takes >50ms and is CPU-bound (sorting, hashing, image processing, parsing large JSON, compression)
- **Don't use a worker** for tasks that are I/O-bound — `fetch` is already async and doesn't block the main thread
- **Prefer a pool** over ad-hoc workers if the same work happens repeatedly

A common mistake is reaching for a worker to make code "async" — but async/await and Promises already give you non-blocking I/O on the main thread without a worker.

---

## Gotchas

**1. Script URL must be same-origin (by default)**
You can't `new Worker('https://cdn.example.com/worker.js')` unless the server sends `Access-Control-Allow-Origin`. This trips teams that try to load workers from a CDN without CORS headers.

**2. Webpack/Vite bundle the main thread, not workers, by default**
Workers reference a separate file. Bundlers need explicit config (Vite: `?worker` import syntax, Webpack: `worker-loader` or asset modules) to bundle and hash worker scripts correctly. A common bug: the worker file isn't included in the build.

```js
// Vite syntax
import MyWorker from './worker.js?worker';
const worker = new MyWorker();
```

**3. Structured clone silently drops non-clonable properties**
If your message payload contains a class instance, only its own enumerable properties are cloned — prototype methods are dropped. You receive a plain object, not an instance of your class. This surprises engineers moving object graphs between threads.

**4. `worker.terminate()` is synchronous and violent**
The worker thread is killed immediately. There's no cleanup event in the worker. If the worker has in-flight IndexedDB transactions or partially written data, that state is lost. Design worker communication to be stateless or explicitly signal shutdown first.

**5. `navigator.hardwareConcurrency` is a hint, not a guarantee**
Browsers often cap the reported value for privacy reasons, and you don't control how many real threads the OS gives you. Pool sizing should use it as an upper bound, not a precise target.

**6. Memory is NOT shared**
Every `postMessage` copies the payload (unless using Transferables or SharedArrayBuffer). Passing a 10MB JSON object back and forth on every animation frame is expensive. Profile before assuming workers help.

**7. Workers in CSP**
A strict `Content-Security-Policy` can block worker creation. Inline blob URLs require `worker-src blob:`. Module worker imports respect `script-src`. CSP issues with workers are silent — the error fires on the `worker.onerror` handler, not in DevTools console by default.

---

## Interview Questions

**Q (High): What problem do Web Workers solve, and why can't you just use async/await instead?**

Answer: Web Workers move CPU-bound computation off the main thread onto a separate OS thread. `async/await` does not create new threads — it only defers execution using the event loop, which is still single-threaded. So `await heavyComputation()` still blocks the main thread while `heavyComputation` runs; it just yields control to the event loop between async operations. If the computation itself is synchronous work (a tight loop, sorting, parsing), it will block no matter how many `await`s you add. A Worker actually runs the code in parallel.

The trap: candidates confuse "non-blocking" with "parallel." Async/await is non-blocking for I/O (network, timers) because those operations are handled by the browser's native layer. But CPU-bound work — a tight computation loop — runs on the JS thread regardless of async syntax.

**Q (High): How does data get passed between a worker and the main thread? What are the performance implications?**

Answer: Via `postMessage`, which uses the structured clone algorithm to serialize and deserialize the data. This creates a deep copy — the worker and main thread never share the same object in memory. For large binary data (ArrayBuffers, ImageBitmaps), you can use Transferables: the buffer's ownership is transferred to the recipient in O(1), and the sender's reference is detached (`.byteLength` becomes 0). If you send large payloads by value repeatedly, you're paying copy costs proportional to payload size every time.

The trap: candidates say "it's a reference" — it's not, it's a copy. Or they're unaware of Transferables entirely and would naively pass large ArrayBuffers by structured clone.

**Q (High): What APIs are available in a Worker, and what's missing?**

Answer: Workers have full network access (fetch, XHR, WebSockets), timers, crypto, IndexedDB, Cache API, and most Web APIs that don't require a visual context. They do not have `window`, `document`, the DOM, or `localStorage`. The global object is `DedicatedWorkerGlobalScope` (accessible as `self`). The key insight is: anything that requires touching the rendering pipeline is absent, because workers have no rendering context.

The trap: candidates say workers have no network access (wrong) or that they can use localStorage (wrong — it would require synchronous access to a shared resource, which workers can't do).

**Q (Medium): Explain the Transferable Objects mechanism and when you'd use it.**

Answer: Transferables are objects that can be handed off between threads with zero-copy semantics. Instead of structured clone creating a copy, ownership is transferred atomically. The sender's reference becomes detached — you can no longer read from it. Currently supported: `ArrayBuffer`, `MessagePort`, `OffscreenCanvas`, `ReadableStream`, `WritableStream`, `TransformStream`, `ImageBitmap`, `VideoFrame`. You use them when passing large binary data — image pixel buffers, audio samples, typed arrays for WASM — where a copy would be prohibitively expensive. The transfer list is the second argument: `postMessage(payload, [transferable1, transferable2])`.

The trap: candidates don't know that the sender's reference is detached, or they can't name which objects are transferable (vs. all structured-cloneable types).

**Q (Medium): How would you structure a worker pool, and why would you need one?**

Answer: A pool pre-creates N workers (commonly `navigator.hardwareConcurrency` or a fraction thereof) and keeps them alive. Tasks are dispatched to idle workers; if all workers are busy, tasks queue. This amortizes the per-worker startup cost (thread creation + script parse) across many tasks, and caps the number of threads to avoid exhausting the OS thread pool. Without a pool, naively creating a new worker per task is expensive for high-frequency work and can spawn hundreds of threads under load.

The trap: candidates don't know that worker creation has overhead, or they cap pool size at 1 (defeating the parallelism goal).

**Q (Medium): What's the difference between a classic worker and a module worker? When would you choose one?**

Answer: Classic workers use `importScripts()` for dependencies and have a global scope with `self` at the top level. Module workers (`new Worker(url, { type: 'module' })`) support static `import`/`export`, the full ES module semantics, and import maps. Module workers are lazily parsed, have stricter MIME type requirements for imports (`application/javascript`), and require CORS headers for cross-origin module imports. Choose module workers for modern codebases with bundlers that support them — they integrate naturally into the module graph. Classic workers are more portable and are the only option in older browsers.

The trap: candidates don't know about the `{ type: 'module' }` option, or they don't realize `importScripts()` is synchronous and blocking within the worker.

**Q (Low): Can a Worker create another Worker?**

Answer: Yes — a Worker can call `new Worker()` to spawn a sub-worker. This creates a nested tree of workers, each on its own thread. However, the sub-worker is owned by the creating worker, not by the main page. When the parent worker terminates, sub-workers are also terminated. This is rarely used in practice but is valid per spec. The browser may impose limits on total worker count.

The trap: candidates say no — which was actually true in older specs but has been supported in modern browsers since around 2015.

**Q (Low): What happens to a Worker when `worker.terminate()` is called? Is there any cleanup?**

Answer: The thread is terminated immediately and unconditionally. The Worker's JS does not get a chance to run any cleanup code — there's no `beforeunload` or shutdown event inside the worker. Any in-flight work is abandoned. To do graceful shutdown, you should send the worker a "shutdown" message, have it flush state and respond, then call `terminate()` after receiving confirmation. Alternatively, the worker can call `self.close()` to terminate itself.

The trap: candidates assume there's a lifecycle event or cleanup hook. There isn't — it's kill, not close.

---

## Self-Assessment

Before moving on, check off each item you can do WITHOUT looking at the file.

- [ ] Explain why Web Workers don't have DOM access, in terms of the browser's threading model
- [ ] Describe what structured clone is and what it can and cannot serialize
- [ ] Write a minimal Worker + main thread example using `postMessage`/`onmessage` from memory
- [ ] Explain Transferables: what they are, how to use them, and what happens to the sender's reference
- [ ] Name three APIs available in workers and two that are not
- [ ] Explain the difference between a Worker solving a blocking problem vs. `async/await` solving it

---
*Next: Service Workers — they share the same Worker primitive but are a fundamentally different beast: they live independently of any page, intercept network requests, and have their own install/activate/fetch lifecycle.*
