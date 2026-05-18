# JS ↔ WASM Interop & Memory Model

## Quick Reference

| Concept | Description |
|---|---|
| Linear memory | A flat `ArrayBuffer` shared between JS and WASM — the only memory WASM can access |
| Imports | Functions JS provides to WASM at instantiation (e.g., `console.log`, `fetch`) |
| Exports | Functions and memory WASM exposes to JS |
| Memory page | 64 KiB unit — WASM memory grows in page increments |
| String marshalling | Converting JS strings to/from WASM bytes — requires explicit encode/decode |

---

## The WASM Memory Model

WASM has one type of memory: a **linear memory** — a contiguous, byte-addressable array of bytes. It's exposed to JavaScript as an `ArrayBuffer` (via `WebAssembly.Memory`). Both WASM code and JS code can read and write the same buffer, making it the primary data exchange channel.

Key properties:
- **No objects, no GC:** WASM's linear memory has no concept of objects, garbage collection, or heap management beyond what the WASM module implements itself. When compiled from Rust, Rust's allocator manages the linear memory. When compiled from C, malloc/free do.
- **Grows in pages:** Memory is allocated in 64 KiB pages. You specify an initial size and optional maximum. Memory can grow (via `memory.grow`) but never shrinks.
- **Type system is minimal:** WASM natively knows only `i32`, `i64`, `f32`, `f64`. Everything else (strings, structs, arrays) is manually laid out in linear memory.

```typescript
// Creating a WASM memory object in JS
const memory = new WebAssembly.Memory({
  initial: 16,  // 16 pages = 1 MiB
  maximum: 256  // 256 pages = 16 MiB
});

// Access as ArrayBuffer
const buffer = memory.buffer; // ArrayBuffer
const view = new Uint8Array(buffer);
```

---

## Loading and Instantiating a WASM Module

```typescript
async function loadWasm(url: string, imports: WebAssembly.Imports): Promise<WebAssembly.Instance> {
  // Streaming instantiation — most efficient
  const { instance } = await WebAssembly.instantiateStreaming(
    fetch(url),
    imports
  );
  return instance;
}

// Example: a module that does math
const mathModule = await loadWasm('/math.wasm', {
  env: {
    memory: new WebAssembly.Memory({ initial: 1 }),
    // Provide JS functions the WASM module imports
    log: (value: number) => console.log(value),
  }
});

// Call an exported WASM function
const result = (mathModule.exports.add as (a: number, b: number) => number)(3, 4);
```

**`instantiateStreaming` vs `instantiate`:**

`instantiateStreaming` compiles the WASM binary as it downloads, overlapping network transfer and compilation. `instantiate` requires the full binary to be in memory first. Always prefer `instantiateStreaming` for initial loads.

---

## Passing Numbers — The Simple Case

Numbers (integers and floats) map directly between JS and WASM:

```typescript
// WASM function signature: (i32, i32) -> i32
const add = mathModule.exports.add as (a: number, b: number) => number;
const sum = add(10, 20); // 30 — no overhead, direct type mapping
```

JS numbers are `f64` by default; WASM `i32` operations truncate. Type coercion happens automatically at the boundary, but be aware of overflow for large integers.

---

## Passing Strings — The Non-trivial Case

WASM doesn't understand JS strings. A string must be encoded into bytes, written into linear memory, and then the offset and byte length are passed to WASM.

```typescript
// Helpers for reading/writing strings through linear memory
function encodeString(
  memory: WebAssembly.Memory,
  allocate: (size: number) => number, // WASM export: malloc-equivalent
  str: string
): [offset: number, length: number] {
  const encoded = new TextEncoder().encode(str);
  const ptr = allocate(encoded.byteLength);
  new Uint8Array(memory.buffer).set(encoded, ptr);
  return [ptr, encoded.byteLength];
}

function decodeString(memory: WebAssembly.Memory, ptr: number, len: number): string {
  const bytes = new Uint8Array(memory.buffer, ptr, len);
  return new TextDecoder().decode(bytes);
}

// Usage
const [ptr, len] = encodeString(memory, wasmModule.exports.malloc as (n: number) => number, 'hello');
const resultPtr = (wasmModule.exports.processString as (ptr: number, len: number) => number)(ptr, len);
const result = decodeString(memory, resultPtr, /* length from module */ 5);
```

**The `memory.buffer` invalidation problem:** when WASM's memory grows (via `memory.grow()`), the underlying `ArrayBuffer` is detached and replaced with a new one. Any views created before the growth are invalidated. Always re-read `memory.buffer` after any operation that might grow memory.

```typescript
// Wrong — view may be stale if memory grew
const view = new Uint8Array(memory.buffer);
wasmModule.exports.maybeGrow();
view[0] = 1; // view might be detached!

// Correct — re-read buffer each time
wasmModule.exports.maybeGrow();
new Uint8Array(memory.buffer)[0] = 1; // fresh view
```

---

## Passing Arrays and Structs

For arrays, write the raw bytes into linear memory and pass the pointer:

```typescript
function processArray(
  memory: WebAssembly.Memory,
  allocate: (n: number) => number,
  free: (ptr: number) => void,
  processInWasm: (ptr: number, len: number) => void,
  data: Float32Array
): void {
  const byteLength = data.byteLength;
  const ptr = allocate(byteLength);

  // Write array into WASM memory
  new Float32Array(memory.buffer).set(data, ptr / Float32Array.BYTES_PER_ELEMENT);

  // Call WASM
  processInWasm(ptr, data.length);

  // Read results back (WASM wrote into same memory)
  const result = new Float32Array(memory.buffer, ptr, data.length);
  console.log(result[0]); // first result

  free(ptr);
}
```

For structs, you lay out fields manually using offsets (just like C's memory layout):

```typescript
// WASM struct: { x: f32 at offset 0, y: f32 at offset 4, active: i32 at offset 8 }
const STRUCT_SIZE = 12;

function readParticle(memory: WebAssembly.Memory, ptr: number): { x: number; y: number; active: boolean } {
  const view = new DataView(memory.buffer);
  return {
    x: view.getFloat32(ptr, true),        // offset 0, little-endian
    y: view.getFloat32(ptr + 4, true),    // offset 4
    active: view.getInt32(ptr + 8, true) !== 0, // offset 8
  };
}
```

---

## Imports and Exports — The Full Contract

WASM modules declare what they need from the outside world (imports) and what they expose (exports). JS is responsible for satisfying all imports at instantiation time.

```typescript
// A WASM module that imports and exports

const imports: WebAssembly.Imports = {
  env: {
    // Memory shared between JS and WASM
    memory: sharedMemory,

    // JS functions that WASM can call
    js_log: (ptr: number, len: number) => {
      const str = decodeString(sharedMemory, ptr, len);
      console.log(str);
    },

    js_now: () => Date.now(),

    // Math functions WASM might need
    Math_sin: Math.sin,
    Math_cos: Math.cos,
  }
};

const { instance } = await WebAssembly.instantiateStreaming(fetch('/module.wasm'), imports);

// Exports the WASM exposes to JS
const {
  memory: wasmMemory,     // if the module exports its own memory
  malloc,                 // allocator
  free,                   // deallocator
  processImage,           // business logic
  getResultPtr,           // get pointer to result buffer
} = instance.exports as {
  memory: WebAssembly.Memory;
  malloc: (size: number) => number;
  free: (ptr: number) => void;
  processImage: (ptr: number, width: number, height: number) => void;
  getResultPtr: () => number;
};
```

---

> **Check yourself:** After calling `memory.grow(1)` in WASM, you try to access data via a `Uint8Array` view you created before the call. What happens, and how do you fix it?

---

## Self-Assessment

- [ ] I can explain what linear memory is and why it's the only way to pass complex data
- [ ] I understand the process for passing a string to a WASM module (encode → write → pass pointer)
- [ ] I know why `memory.buffer` can become invalid after memory growth
- [ ] I understand the difference between WASM imports and exports
- [ ] I know when to use `instantiateStreaming` vs `instantiate`
- [ ] I can lay out a struct in linear memory using byte offsets

---

## Interview Q&A

**Q: How does JavaScript communicate with a WebAssembly module?** `High`

Through two mechanisms: function calls and shared linear memory. Function calls work for simple numeric arguments — you call a WASM-exported function with numbers and get a number back. For complex data (strings, arrays, structs), you write the data into the shared `ArrayBuffer` (WASM's linear memory), pass the byte offset and length to the WASM function, and read the result back from memory afterward. JS and WASM share the same `ArrayBuffer`, so no copy is needed for the data itself — the transfer is pointer passing.

---

**Q: Why does string passing between JS and WASM require extra work?** `High`

WASM's type system only understands integers and floats. A JS string is a high-level object with character encoding, length metadata, and Unicode semantics. To pass a string to WASM you must: encode it to bytes (UTF-8 via `TextEncoder`), allocate space in WASM's linear memory (via the module's exported allocator), write the encoded bytes into that memory at the allocated offset, and pass the offset and byte length to the WASM function. The reverse requires reading bytes from memory and decoding them with `TextDecoder`. This marshalling overhead is why you batch operations — doing it per-character would be slower than pure JS.

---

**Q: What happens to a `Uint8Array` view when WASM memory grows?** `Medium`

The underlying `ArrayBuffer` is replaced with a new, larger one. Any typed array view or `DataView` created over the old buffer is detached — accessing it throws a `TypeError`. You must re-read `memory.buffer` after any operation that may grow the memory and create fresh views. Toolchains like Emscripten handle this automatically through their generated wrapper code, but when writing manual interop you must be aware of it.

---

**Q: What is the difference between WASM imports and exports?** `Medium`

Imports are what the WASM module requires from the host (JS) to run: memory, functions it wants to call from JS-land (logging, math, fetch), and any other host-provided APIs. They're satisfied at instantiation time. Exports are what the WASM module makes available to JS: typically memory, allocator functions, and the business logic functions JS will call. You must provide all imports declared in the module, and you can call any exported function or read any exported memory from JS.

---

**Q: Why is `WebAssembly.instantiateStreaming` preferred over `instantiate`?** `Low`

`instantiateStreaming` compiles the WASM binary progressively as it downloads, overlapping the network transfer with compilation. `instantiate` requires the full binary to be downloaded and available as a `BufferSource` before compilation starts — so download and compile happen serially. On large WASM modules over slow connections, `instantiateStreaming` can be significantly faster at getting the module ready to execute. It also requires less peak memory since you don't need to hold the full binary in memory before starting compilation.
