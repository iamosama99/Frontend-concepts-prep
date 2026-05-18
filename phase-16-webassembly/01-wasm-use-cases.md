# WASM Use Cases: Compute-heavy Workloads, Porting Native Code

## Quick Reference

| Use Case | Why WASM | Example |
|---|---|---|
| Image / video processing | CPU-bound loops benefit from near-native speed | ffmpeg.wasm, Squoosh |
| Audio codecs | Real-time DSP needs predictable throughput | Opus, MP3 decoding |
| Physics engines | Many math operations per frame | Ammo.js (Bullet port) |
| Cryptography | Constant-time operations, native speed | Argon2, AES-GCM |
| Porting existing C/C++/Rust code | Avoid rewriting native libraries in JS | SQLite, OpenCV |

---

## What WebAssembly Actually Is

WebAssembly (WASM) is a **binary instruction format** that runs in a stack-based virtual machine built into browsers and other runtimes. It is not assembly language for any specific CPU — it's a portable bytecode that browsers JIT-compile to native machine code.

Key properties:
- **Near-native performance:** WASM is compiled ahead of time into machine code during loading, then JIT-compiled further during execution. It avoids the overhead of JS parsing, property lookups, and dynamic type checks.
- **Sandboxed by default:** WASM runs in the same origin-based security sandbox as JavaScript. It has no direct access to the DOM, file system, or OS unless JS explicitly provides functions for that.
- **Language-agnostic:** C, C++, Rust, Go, AssemblyScript, and others can compile to WASM.
- **Runs alongside JS:** WASM modules are loaded and called from JavaScript. They don't replace JS; they extend it.

---

## Why JavaScript Has a Speed Ceiling

JavaScript is fast for its class of language, but it has structural limits:

1. **Dynamic typing:** the engine can't know the shape of objects statically, so it speculates and bails out to deoptimized paths when speculation fails.
2. **Garbage collection pauses:** JS allocates and the GC collects. In a tight loop doing lots of allocation, GC pauses break throughput.
3. **JIT warm-up time:** V8's optimizing compiler (Turbofan) needs to observe code running to emit optimized machine code. Before it's warmed up, code runs interpreted or lightly optimized.
4. **No SIMD in JS (with exceptions):** WASM has native SIMD support that can process multiple data elements in a single CPU instruction.

WASM sidesteps most of these: it's statically typed at the binary level, can use linear memory without GC, and compiles to near-optimal machine code immediately.

---

## Where the JS Boundary Cost Lives

WASM is not uniformly faster than JavaScript. The cost lives at the **JS ↔ WASM boundary**:

- Calling a WASM function from JS incurs overhead for the type marshalling and call setup.
- Passing complex data (strings, objects) requires serialization through shared memory.

The performance gain from WASM comes from keeping compute **inside** WASM, minimizing how often you cross the boundary. An image filter that calls WASM once per pixel is no faster than JS; one that passes the entire pixel buffer and processes it entirely within WASM is dramatically faster.

```typescript
// Inefficient: crosses boundary per pixel
for (let i = 0; i < pixels.length; i += 4) {
  pixels[i] = wasmModule.processPixel(pixels[i]); // thousands of boundary crossings
}

// Efficient: cross boundary once with the whole buffer
const ptr = wasmModule.allocate(pixels.length);
wasmMemory.set(pixels, ptr); // write buffer into WASM linear memory
wasmModule.processImage(ptr, pixels.length); // process entirely in WASM
wasmMemory.copyToOutput(ptr, pixels); // read results back
wasmModule.free(ptr);
```

---

## Real-World Use Cases

### Image and Video Processing

Squoosh (Google's image compression tool) uses WASM to run codec encoders (WebP, AVIF, MozJPEG) in the browser. These are written in C/C++ and compiled to WASM. The alternative — rewriting them in JS — would be both slower and a massive engineering effort.

ffmpeg.wasm ports the entire FFmpeg library to the browser, enabling video transcoding client-side without a server.

### Cryptography

Cryptographic operations benefit doubly from WASM: they're CPU-bound, and security-critical operations need constant-time behavior (predictable execution that doesn't vary with input values, preventing timing attacks). WASM is much more predictable in this regard than JavaScript.

Argon2 (a memory-hard password hashing algorithm) is commonly run via WASM in browser-based password managers.

### Physics Engines

Ammo.js is a JavaScript/WASM port of the Bullet physics library (C++). Physics simulations involve many floating-point operations per frame. Running them in WASM while keeping the rendering in WebGL achieves frame rates that pure JS can't match for complex scenes.

### SQLite in the Browser

SQLite compiled to WASM runs in the browser, enabling full SQL query execution without a backend. This is used in offline-first apps, local data analysis tools, and browser-based development environments.

---

## What WASM Does NOT Help With

- **DOM manipulation:** WASM has no DOM access. All DOM operations go through JS. Moving DOM manipulation to WASM would require constant JS boundary crossings, making it slower, not faster.
- **Network requests:** same — no direct access, must call through JS.
- **IO-bound workloads:** WASM is only faster for CPU-bound work. If your bottleneck is waiting for a network response or reading from IndexedDB, WASM doesn't help.
- **Startup time:** WASM binaries are larger than equivalent JS and take time to download, parse, and compile. Small utilities don't benefit.

---

> **Check yourself:** You're building a feature that resizes images in the browser. The implementation does 10 operations per pixel. Would WASM help? What would you need to do to make the boundary cost acceptable?

---

## Self-Assessment

- [ ] I can explain what WASM is in terms a non-specialist would understand
- [ ] I know why JS has a performance ceiling and what WASM sidesteps
- [ ] I understand the JS ↔ WASM boundary cost and how to minimize it
- [ ] I can name 3 real-world use cases where WASM provides meaningful benefit
- [ ] I know what WASM does NOT help with (DOM, IO-bound work)

---

## Interview Q&A

**Q: What is WebAssembly and why does it exist?** `High`

WebAssembly is a binary instruction format that browsers compile to native machine code and execute in a sandboxed virtual machine. It exists because JavaScript has structural performance limits — dynamic typing, garbage collection pauses, JIT warm-up — that make it too slow for compute-heavy workloads like image processing, audio codecs, physics simulations, and cryptography. WASM lets native code (C, C++, Rust) run in the browser at near-native speed, without being a security risk, without a plugin, and without requiring a server for the heavy computation.

---

**Q: What kind of workloads benefit from WASM?** `High`

CPU-bound workloads where the computation stays inside WASM: image/video processing, audio codecs, physics engines, cryptographic operations, data compression, and porting large native codebases (SQLite, OpenCV, FFmpeg). The key condition is that the hot work runs inside WASM with minimal JS boundary crossings. IO-bound workloads, DOM manipulation, and anything requiring frequent JS interop don't benefit — the boundary overhead negates the computational speed advantage.

---

**Q: Why isn't WASM faster for everything?** `Medium`

The JS ↔ WASM boundary has a non-trivial crossing cost: type marshalling, call setup, and data serialization. For small operations called frequently from JS, this overhead can make WASM slower than equivalent JS. Also, WASM has no DOM access — DOM operations must still go through JS, meaning WASM can't help with rendering or event handling. WASM shines when large chunks of computation run entirely inside the module. It's a targeted tool for specific bottlenecks, not a wholesale replacement for JavaScript.

---

**Q: How do you pass data between JavaScript and a WASM module?** `Medium`

Through **linear memory** — a shared `ArrayBuffer` that both JS and WASM can read and write. To pass a string or byte array to WASM, you write it into the shared buffer at an offset, then pass that offset to the WASM function. The WASM function reads from memory directly. Results are written back to memory and read from JS. Complex objects must be serialized (e.g., to JSON bytes or a binary format) before being passed, because WASM's type system is limited to integers and floats. This is why minimizing boundary crossings matters — serialization has a cost.

---

**Q: Can WASM access the DOM directly?** `Low`

No. WASM runs in a sandboxed linear memory model and has no access to the DOM, Web APIs, or anything outside its own memory unless JavaScript explicitly provides an import function that bridges to those APIs. If you want WASM to trigger a DOM update, you pass data back to JS and JS does the DOM update. This design is intentional — it keeps WASM safely sandboxed and means the DOM's threading model (single-threaded, main thread only) doesn't need to be re-specified.
