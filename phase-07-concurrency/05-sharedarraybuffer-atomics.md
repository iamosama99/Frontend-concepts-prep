# SharedArrayBuffer & Atomics

## Quick Reference

| Concept | What it is | Why it matters |
|---|---|---|
| `SharedArrayBuffer` | ArrayBuffer whose memory is shared across threads | Zero-copy data sharing — no structured clone overhead |
| `Atomics` | API for atomic read/modify/write operations on shared memory | Prevents data races on shared integers |
| `Atomics.wait()` | Blocks the current thread until a condition is met | Mutex/semaphore primitive (can't use on main thread) |
| `Atomics.notify()` | Wakes threads waiting on a shared index | The other half of wait/notify — like a condition variable |
| COOP + COEP headers | Cross-Origin-Opener-Policy + Cross-Origin-Embedder-Policy | Required by browsers before they enable SAB (Spectre mitigation) |

---

## What Is This?

`SharedArrayBuffer` (SAB) is a raw memory buffer that can be simultaneously referenced by the main thread and one or more Workers. Unlike a normal `ArrayBuffer` (which is copied on `postMessage` or transferred with ownership change), a `SharedArrayBuffer` is backed by a single region of memory. Multiple threads hold different views of the same bytes.

```js
// main.js
const sab = new SharedArrayBuffer(4); // 4-byte shared memory region
const view = new Int32Array(sab);     // typed view over it
view[0] = 42;

worker.postMessage({ sab }); // sab is shared, not copied

// worker.js
self.onmessage = (e) => {
  const view = new Int32Array(e.data.sab);
  console.log(view[0]); // 42 — same memory
  view[0] = 100;        // visible to main thread immediately
};
```

`Atomics` is the namespace of functions for safely reading and writing shared memory. Without it, you have a data race: two threads can read-modify-write the same byte non-atomically, producing corrupted state.

> **Check yourself:** Why is `view[0]++` a data race even if `++` looks like a single operation?

---

## Why Does It Exist?

### The case for shared memory

`postMessage` + structured clone works for most Worker communication, but it has a cost: **copying**. For a 100MB typed array, structured clone serializes and copies all 100MB — O(n) in transfer time and memory. Transferables solve the copy cost but transfer ownership (the sender loses the buffer).

Some workloads need:
1. **Both threads to access the same data simultaneously** (not ownership transfer)
2. **Zero-copy access** to large buffers (image pixel data, audio samples, WASM memory)
3. **Lock-free signaling** between threads (too fast for `postMessage` round-trips)

SAB + Atomics fills this gap. It's the basis for WASM threading (WASM's shared linear memory maps to a SAB), high-performance audio buffers, and parallel computation frameworks.

### Spectre and the header requirement

SAB was shipped in Chrome 60, then globally **disabled in 2018** after Spectre — shared memory enables high-resolution timers (via `Atomics.wait()`) that can measure cache timing side-channels to read arbitrary memory. It was re-enabled in 2020 with a mandatory cross-origin isolation requirement that neutralizes Spectre's attack surface.

---

## How It Works

### Creating and sharing a SAB

```js
const sab = new SharedArrayBuffer(16); // 16 bytes
// Typed views work the same as ArrayBuffer
const i32 = new Int32Array(sab);   // 4 Int32s
const u8  = new Uint8Array(sab);   // 16 Uint8s (same memory, different view)
```

The SAB can be sent via `postMessage` to any number of Workers — it is passed as a reference, not copied. All workers immediately see each other's writes (subject to memory model ordering, discussed below).

### Atomics API

`Atomics` operations guarantee indivisibility — a read-modify-write is one uninterruptible CPU instruction (using hardware `LOCK CMPXCHG`, `XADD`, etc.):

```js
const shared = new Int32Array(new SharedArrayBuffer(4));

// Atomic add: reads current value, adds delta, writes result — all as one operation
Atomics.add(shared, 0, 1);   // index 0, add 1
Atomics.sub(shared, 0, 5);
Atomics.and(shared, 0, 0xFF);
Atomics.or(shared, 0, 0x01);
Atomics.xor(shared, 0, 0xFF);

// Atomic load/store
const val = Atomics.load(shared, 0);
Atomics.store(shared, 0, 42);

// Compare-and-swap: write newVal only if current === expected
// Returns the OLD value
const old = Atomics.compareExchange(shared, 0, expected, newVal);
const didSwap = old === expected;
```

`Atomics.compareExchange` (CAS) is the building block for all lock-free data structures: spin-locks, queues, stacks.

> **Check yourself:** Two workers simultaneously call `Atomics.add(shared, 0, 1)`. What is the final value if `shared[0]` starts at 0?

### Wait and Notify (mutex semantics)

For scenarios where you need a thread to block until a condition is met:

```js
// Worker A — waits until index 0 is no longer 0
// Atomics.wait(typedArray, index, expectedValue, timeout?)
Atomics.wait(shared, 0, 0); // blocks until shared[0] !== 0

// Worker B — signals Worker A to wake up
Atomics.store(shared, 0, 1);
Atomics.notify(shared, 0, 1); // wake up 1 waiter at index 0
```

`Atomics.wait()` can only be called from a Worker — calling it on the main thread throws a `TypeError` because blocking the main thread is forbidden (it would freeze the tab).

`Atomics.waitAsync()` is the non-blocking variant (returns a promise) that works on the main thread.

### JavaScript Memory Model

SAB introduces the concept of a memory model to JS. Writes to shared memory are **not instantly visible** to other threads without explicit synchronization. CPUs and compilers reorder memory operations for performance.

Atomics operations have sequential consistency — they establish a happens-before relationship. A write with `Atomics.store()` is guaranteed to be visible to any thread that subsequently calls `Atomics.load()`.

Non-atomic reads (`shared[0]`) on shared memory have **data race** semantics — the browser can return any value, including a torn read (bytes from two different writes interleaved). Always use `Atomics.*` for shared memory that is written from multiple threads.

---

## Cross-Origin Isolation Headers

To use `SharedArrayBuffer`, the page must be cross-origin isolated. This requires two HTTP response headers on the page's HTML document:

```http
Cross-Origin-Opener-Policy: same-origin
Cross-Origin-Embedder-Policy: require-corp
```

`COOP: same-origin` — the page gets its own browsing context group; it can't be opened by or open cross-origin windows (closes the Spectre timing attack vector by preventing cross-origin memory reads).

`COEP: require-corp` — every cross-origin resource on the page must either have `Cross-Origin-Resource-Policy: cross-origin` (opt-in) or `crossorigin` attribute on the HTML element. This prevents a page from embedding untrusted cross-origin resources that could be read via a Spectre side-channel.

You can check if a page is isolated:

```js
console.log(self.crossOriginIsolated); // true if COOP + COEP both set
if (!self.crossOriginIsolated) {
  // SharedArrayBuffer unavailable
}
```

---

## Practical Use: Lock-free Ring Buffer

A ring buffer for passing audio data from a Worker to an AudioWorklet without `postMessage`:

```js
// Shared layout: [readHead (1 int), writeHead (1 int), data (N ints)]
const CAPACITY = 1024;
const sab = new SharedArrayBuffer((2 + CAPACITY) * 4);
const ctrl = new Int32Array(sab, 0, 2);  // [readHead, writeHead]
const data = new Float32Array(sab, 8, CAPACITY);

function write(value) {
  const write = Atomics.load(ctrl, 1);
  const nextWrite = (write + 1) % CAPACITY;
  const read = Atomics.load(ctrl, 0);
  if (nextWrite === read) return false; // full
  data[write] = value;
  Atomics.store(ctrl, 1, nextWrite); // publish new write head
  return true;
}

function read() {
  const read = Atomics.load(ctrl, 0);
  const write = Atomics.load(ctrl, 1);
  if (read === write) return null; // empty
  const value = data[read];
  Atomics.store(ctrl, 0, (read + 1) % CAPACITY);
  return value;
}
```

This is lock-free for single-producer single-consumer (SPSC) — no `Atomics.wait()` needed because the two index updates are independent.

---

## Gotchas

**1. `Atomics.wait()` throws on the main thread**
Non-blocking browsers won't let you block the main thread. Use `Atomics.waitAsync()` on the main thread and `Atomics.wait()` only in Workers.

**2. Non-atomic reads on shared memory have data race semantics**
`shared[0] = 1` from one thread and `console.log(shared[0])` from another, without Atomics, is undefined behavior in the JS memory model. The value can be stale, torn, or speculative. Always use `Atomics.load` / `Atomics.store` for shared writes.

**3. COOP breaks cross-origin postMessage with window references**
`COOP: same-origin` means `window.opener` is null and you can't send messages to/from cross-origin popup windows. If your app depends on OAuth popup flows or cross-origin communication via `window.postMessage`, COOP will break it.

**4. COEP blocks cross-origin images/scripts without CORP or CORS**
With `COEP: require-corp`, every cross-origin resource needs opt-in. Embedding a random CDN image without `crossorigin` attribute causes it to be blocked. Third-party scripts, widgets, and embeds need to be audited.

**5. SAB is not transferable**
You cannot transfer a `SharedArrayBuffer` — it can only be sent by reference via `postMessage`. This is intentional: transferring would mean only one thread owns it, which defeats the purpose.

**6. SAB size cannot change**
`SharedArrayBuffer` has no `.resize()` method (unlike the newer `ArrayBuffer` growable option). Allocate enough upfront, or implement your own dynamic buffer by pre-allocating a large SAB.

---

## Interview Questions

**Q (High): What is SharedArrayBuffer and how does it differ from a Transferable?**

Answer: A `SharedArrayBuffer` is a fixed-size region of raw memory that is simultaneously visible to all threads that hold a reference to it. Writes from one thread are visible to all others (subject to memory ordering). A Transferable (e.g., a normal `ArrayBuffer`) transfers ownership: the sender loses access, and the receiver gets the buffer — zero-copy, but exclusive. SAB is zero-copy and shared simultaneously. Use a Transferable when ownership should change; use SAB when multiple threads need concurrent access — like passing audio samples from a producer Worker to an AudioWorklet consumer, both reading/writing the same ring buffer.

**Q (High): Why was SharedArrayBuffer disabled in 2018 and what was needed to re-enable it?**

Answer: SAB enables a high-resolution timer via `Atomics.wait()` — you can measure how long an operation takes by looping and reading time with it. Combined with Spectre-class speculation vulnerabilities, this timer allows reading arbitrary memory across origins (side-channel attacks). It was disabled globally after Spectre's disclosure. Re-enabling required cross-origin isolation: `Cross-Origin-Opener-Policy: same-origin` + `Cross-Origin-Embedder-Policy: require-corp`. These headers prevent cross-origin memory sharing and limit what a page can embed, closing the Spectre attack surface.

**Q (High): Why is `view[0]++` a data race on shared memory? What should you use instead?**

Answer: `view[0]++` compiles to three operations: load the value, add 1, store the new value. On shared memory, two threads can each load the value (both read 0), both compute 1, and both store 1 — resulting in a final value of 1 instead of 2. This is a classic read-modify-write race. The correct replacement is `Atomics.add(view, 0, 1)`, which is a single atomic CPU instruction that reads and increments without any other thread being able to interleave.

**Q (Medium): What is the purpose of `Atomics.wait()` and why is it forbidden on the main thread?**

Answer: `Atomics.wait(typedArray, index, value)` blocks the calling thread until another thread changes the value at `index` away from `value` (or a timeout elapses). It's used to implement mutexes and condition variables — a worker waits for a signal before proceeding. It's forbidden on the main thread because blocking the main thread would freeze the tab: no rendering, no event handling, no user interaction. `Atomics.waitAsync()` provides a promise-based alternative that doesn't block.

**Q (Medium): Explain why cross-origin isolation headers are needed and what they break.**

Answer: COOP (`same-origin`) puts the page in its own browsing context group — it can't share a browsing context (same OS process) with cross-origin pages, removing the process-level memory sharing that Spectre exploits. COEP (`require-corp`) ensures all cross-origin subresources explicitly opt in to being embedded — preventing a page from loading arbitrary cross-origin data into memory where it could be scanned via a timing side-channel. What they break: `COOP` prevents communication with cross-origin popups (`window.opener` becomes null, `postMessage` to/from cross-origin windows fails). `COEP` blocks cross-origin resources without explicit CORS or `Cross-Origin-Resource-Policy` headers — third-party embeds, CDN images, and analytics scripts may need updates.

**Q (Low): What is a SPSC (single-producer, single-consumer) ring buffer and why doesn't it need locks?**

Answer: A ring buffer with exactly one writer and one reader can be implemented lock-free using two indices (read head, write head) stored in shared memory. The writer only advances the write head; the reader only advances the read head. Since each index is written by only one thread, no compare-and-swap is needed — only `Atomics.store` and `Atomics.load` for memory ordering. Locks are needed when multiple writers or readers compete to update the same index. SPSC eliminates that competition by partitioning who owns each index.

---

## Self-Assessment

- [ ] Explain the difference between `postMessage` with a Transferable vs. a SharedArrayBuffer
- [ ] Name at least five `Atomics` methods and what each does
- [ ] Explain why `view[0]++` is a race condition and what `Atomics.add` does differently
- [ ] Describe the COOP + COEP headers and the security reason they're required
- [ ] Explain why `Atomics.wait()` cannot be called on the main thread

---
*Next: Concurrency Scheduling — how React's Concurrent Mode, the built-in scheduler, and `scheduler.postTask` let you control task priority on the main thread without moving work to another thread.*
