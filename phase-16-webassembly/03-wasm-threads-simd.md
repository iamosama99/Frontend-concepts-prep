# WASM Threads & SIMD

## Quick Reference

| Feature | Requirement | What It Enables |
|---|---|---|
| WASM Threads | `SharedArrayBuffer` + COOP/COEP headers | True parallelism via shared memory + atomics |
| WASM SIMD | Nothing extra (stable in all modern browsers) | 128-bit vector operations for data parallelism |
| `SharedArrayBuffer` | COOP + COEP response headers | Shared memory between threads/workers |
| `Atomics` | `SharedArrayBuffer` | Lock-free synchronization primitives |

---

## WASM Threads: How They Work

WebAssembly alone is single-threaded. WASM threading is built on top of JavaScript's Web Workers and the `SharedArrayBuffer` API:

1. The main thread creates a `SharedArrayBuffer` and passes it to multiple Web Workers.
2. Each Worker instantiates the same WASM module with the same `SharedArrayBuffer` as its linear memory.
3. All WASM instances read and write the same physical memory — this is true shared-memory parallelism.
4. `WebAssembly.Memory` with `{ shared: true }` wraps the `SharedArrayBuffer`.

```typescript
// Main thread
const sharedMemory = new WebAssembly.Memory({
  initial: 256,  // pages (16 MiB)
  maximum: 256,
  shared: true,  // backed by SharedArrayBuffer
});

// Transfer memory to worker (structured clone — no copy, just a new view)
const worker = new Worker('/wasm-worker.js');
worker.postMessage({ memory: sharedMemory });
```

```typescript
// wasm-worker.ts
self.onmessage = async ({ data: { memory } }) => {
  const { instance } = await WebAssembly.instantiateStreaming(
    fetch('/module.wasm'),
    { env: { memory } } // same physical memory as main thread
  );

  // This WASM instance shares memory with all other workers
  (instance.exports.compute as () => void)();
};
```

---

## COOP and COEP Headers — Why They're Required

`SharedArrayBuffer` was disabled in 2018 after the Spectre/Meltdown vulnerabilities demonstrated that high-resolution timers (which `SharedArrayBuffer` enables via `Atomics.wait` busy-polling) could be used to exploit CPU speculative execution.

It was re-enabled in 2020 behind **cross-origin isolation**:

```
Cross-Origin-Opener-Policy: same-origin
Cross-Origin-Embedder-Policy: require-corp
```

These headers create an isolated browsing context where cross-origin documents are prevented from sharing the same process, eliminating the side-channel risk. Without them, `SharedArrayBuffer` is `undefined` and WASM threads don't work.

**What cross-origin isolation breaks:**
- OAuth popups from other origins (common problem)
- Embedding cross-origin iframes without `Cross-Origin-Resource-Policy: cross-origin` on the embedded resource
- Any resource loaded from a CDN that doesn't send `CORP` headers

This is a deployment-level consideration — your server must send these headers, and third-party resources must cooperate.

---

## Synchronizing Across Threads with Atomics

Shared memory + multiple threads = race conditions. WASM threads use `WebAssembly.atomic` instructions (at the WASM binary level) for synchronization. From the JS side, the equivalent is the `Atomics` API on `SharedArrayBuffer`-backed `TypedArray`s.

```typescript
const sharedBuffer = new SharedArrayBuffer(4);
const sharedInt = new Int32Array(sharedBuffer);

// Worker 1: increment a shared counter atomically
Atomics.add(sharedInt, 0, 1); // atomic: no race condition

// Worker 2: wait until index 0 equals 1
Atomics.wait(sharedInt, 0, 0); // block until value ≠ 0

// Main thread: notify waiting worker
Atomics.notify(sharedInt, 0, 1); // wake up 1 waiter
```

**`Atomics.wait` blocks the thread.** It cannot be called on the main thread (which would freeze the page). Use it only in Workers, or use `Atomics.waitAsync` on the main thread, which returns a Promise.

---

## Practical WASM Threading Pattern

A common pattern: main thread coordinates, workers do compute.

```typescript
// Coordinator
class WasmThreadPool {
  private workers: Worker[] = [];
  private memory: WebAssembly.Memory;

  constructor(workerCount: number) {
    this.memory = new WebAssembly.Memory({ initial: 256, maximum: 256, shared: true });

    for (let i = 0; i < workerCount; i++) {
      const worker = new Worker('/compute-worker.js');
      worker.postMessage({
        memory: this.memory,
        workerId: i,
        totalWorkers: workerCount,
      });
      this.workers.push(worker);
    }
  }

  async runParallel(inputPtr: number, length: number): Promise<void> {
    const chunkSize = Math.ceil(length / this.workers.length);
    const promises = this.workers.map((worker, i) => {
      const start = i * chunkSize;
      const end = Math.min(start + chunkSize, length);
      return new Promise<void>(resolve => {
        worker.onmessage = () => resolve();
        worker.postMessage({ inputPtr, start, end });
      });
    });
    await Promise.all(promises);
  }
}
```

---

## WASM SIMD: Data Parallelism

SIMD (**Single Instruction, Multiple Data**) is a CPU feature that applies one instruction to multiple data elements simultaneously. Modern CPUs have 128-bit to 512-bit vector registers that can process 4, 8, or 16 values at once.

WASM SIMD exposes 128-bit vectors with operations like `f32x4.add` (add four 32-bit floats in one instruction) and `i8x16.eq` (compare 16 bytes at once).

**Why this matters:**

An image processing operation like "multiply each pixel's RGB values by a scale factor" touches millions of bytes. Without SIMD, you loop over each byte individually. With SIMD, you process 16 bytes per instruction — up to 16x faster for vectorizable operations.

WASM SIMD is emitted by compilers (Rust, C/C++ with `-msimd128`) automatically when it detects vectorizable loops. You rarely write SIMD by hand; you write the algorithm and let the compiler vectorize it.

```
Scalar (no SIMD): process 1 pixel component per iteration
SIMD (f32x4):     process 4 pixel components per iteration
SIMD (i8x16):     process 16 bytes per iteration
```

**Enabling SIMD from Rust:**

```bash
# Build with SIMD support (wasm32 target with SIMD feature flag)
RUSTFLAGS="-C target-feature=+simd128" cargo build --target wasm32-unknown-unknown --release
```

---

## SIMD Feature Detection

Unlike WASM threads (which require explicit headers), WASM SIMD is now stable and supported in all major browsers (Chrome 91+, Firefox 89+, Safari 16.4+). You can use it without feature detection for modern target browsers, or fall back to a non-SIMD build for older browsers.

```typescript
async function loadWasmWithSIMDFallback(simdUrl: string, fallbackUrl: string): Promise<WebAssembly.Instance> {
  const supportsSIMD = await checkSIMDSupport();
  const url = supportsSIMD ? simdUrl : fallbackUrl;
  const { instance } = await WebAssembly.instantiateStreaming(fetch(url), {});
  return instance;
}

async function checkSIMDSupport(): Promise<boolean> {
  // Try to validate a WASM module that uses SIMD
  try {
    const simdTest = Uint8Array.from([
      0x00, 0x61, 0x73, 0x6d, // WASM magic
      0x01, 0x00, 0x00, 0x00, // version
      // ... minimal SIMD instruction
    ]);
    await WebAssembly.validate(simdTest);
    return true;
  } catch {
    return false;
  }
}
```

---

> **Check yourself:** Your server hosts a WASM app that uses threads. Users report it's not working. What's the first thing you check?

---

## Self-Assessment

- [ ] I can explain how WASM threads are built on top of SharedArrayBuffer and Web Workers
- [ ] I know why COOP and COEP headers are required for SharedArrayBuffer
- [ ] I understand what `Atomics.wait` does and why it can't run on the main thread
- [ ] I know what SIMD is and the scale of speedup it can provide for vectorizable workloads
- [ ] I understand that SIMD is emitted by compilers, not typically hand-written
- [ ] I can explain the difference between thread-based parallelism and SIMD data parallelism

---

## Interview Q&A

**Q: How does WebAssembly achieve true multi-threading?** `High`

WASM multi-threading uses JavaScript's Web Workers as thread carriers and `SharedArrayBuffer` as shared linear memory. Multiple Workers each instantiate the same WASM module and are given the same `WebAssembly.Memory` object (backed by a `SharedArrayBuffer`). All instances read and write the same physical memory. Synchronization between threads uses WASM's atomic instructions at the binary level, which correspond to the `Atomics` API in JavaScript. This is genuine shared-memory parallelism — not message-passing.

---

**Q: Why do WASM threads require `Cross-Origin-Opener-Policy` and `Cross-Origin-Embedder-Policy` headers?** `High`

`SharedArrayBuffer` was disabled after Spectre because it enables high-resolution timing via `Atomics.wait` busy-polling, which can be used to extract data from other processes through CPU cache timing side-channels. The re-enabling condition is cross-origin isolation: these two headers restrict the page to a same-origin process (COOP) and prevent it from loading cross-origin resources unless those resources explicitly opt in (COEP). This eliminates the cross-process attack vector, making `SharedArrayBuffer` safe to enable.

---

**Q: What is WASM SIMD and what kinds of workloads benefit from it?** `Medium`

WASM SIMD exposes 128-bit vector registers, letting one instruction operate on multiple data values simultaneously — for example, adding four 32-bit floats in one cycle instead of four separate operations. Workloads that benefit are those with regular, repetitive operations on arrays of numeric data: image processing (apply a filter to all pixels), audio processing (process all samples), matrix multiplication, data analysis. The compiler vectorizes eligible loops automatically, so you write normal scalar code and the optimizer emits SIMD instructions.

---

**Q: What's the difference between WASM threading parallelism and SIMD parallelism?** `Medium`

Threading parallelism distributes work across multiple CPU cores simultaneously — different threads run truly in parallel, each on a different core. It's useful when work can be divided into independent chunks (process the left half of an image on thread 1, the right half on thread 2). SIMD parallelism operates within a single instruction on a single core, processing multiple data elements at once using wide registers. It's useful when you do the same operation repeatedly on a sequence of values. These are complementary: a well-optimized WASM workload might use both — multiple threads each using SIMD internally.

---

**Q: Can you call `Atomics.wait` on the main browser thread?** `Low`

No. `Atomics.wait` blocks the calling thread until the condition is met. On the main thread, blocking would freeze the entire page — no rendering, no events, no user interaction. Browsers explicitly throw a `TypeError` if you call `Atomics.wait` on the main thread. The alternatives: use `Atomics.waitAsync`, which returns a Promise and doesn't block; or use Worker threads for all code that needs to wait on shared memory.
