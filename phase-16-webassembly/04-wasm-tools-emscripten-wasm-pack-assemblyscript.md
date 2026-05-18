# WASM Tools: Emscripten, wasm-pack, AssemblyScript

## Quick Reference

| Tool | Source Language | Best For | Output |
|---|---|---|---|
| Emscripten | C / C++ | Porting existing native libraries | `.wasm` + JS glue code |
| wasm-pack | Rust | New WASM modules with idiomatic Rust | `.wasm` + TS bindings + npm package |
| AssemblyScript | TypeScript-like syntax | JS devs new to WASM, lighter-weight modules | `.wasm` (no external toolchain) |

---

## Why the Toolchain Matters

WASM is a binary format — you don't write it by hand. Toolchains compile higher-level languages to the WASM binary, and they also generate the JavaScript glue code that manages the string marshalling, memory allocation, and function binding boilerplate covered in the interop topic.

The choice of toolchain determines:
- Which source language you work in
- How complex the JS interop layer is
- How much of the memory management is automated vs. manual
- Whether you get TypeScript types for your WASM exports

---

## Emscripten — C/C++ to WASM

Emscripten is the mature toolchain for compiling C and C++ to WASM. It was built to support porting large native codebases (games, multimedia libraries, scientific computing) to the browser.

**What Emscripten does:**
- Compiles C/C++ to WASM
- Generates a JavaScript wrapper file (`module.js`) that handles instantiation, memory setup, and exposes a friendly API
- Provides POSIX compatibility shims (file system, sockets via emulated APIs)
- Supports SDL, OpenGL (via WebGL), and other native APIs

**Basic compilation:**

```bash
# Install the Emscripten SDK
git clone https://github.com/emscripten-core/emsdk.git
./emsdk install latest
./emsdk activate latest
source ./emsdk_env.sh

# Compile C to WASM + JS glue
emcc src/image_filter.c \
  -O3 \                         # optimize for size/speed
  -s WASM=1 \                   # output wasm (not asm.js)
  -s EXPORTED_FUNCTIONS='["_apply_filter", "_malloc", "_free"]' \
  -s EXPORTED_RUNTIME_METHODS='["ccall", "cwrap"]' \
  -o dist/image_filter.js       # generates image_filter.js + image_filter.wasm
```

**Using the generated module:**

```typescript
// Emscripten-generated module is a CommonJS module (or ES module with flags)
import createModule from './image_filter.js';

async function processImage(data: Uint8Array): Promise<Uint8Array> {
  const Module = await createModule();

  // cwrap creates a typed wrapper around a C function
  const applyFilter = Module.cwrap('apply_filter', 'number', ['number', 'number']);

  const ptr = Module._malloc(data.length);
  Module.HEAPU8.set(data, ptr);

  applyFilter(ptr, data.length);

  const result = Module.HEAPU8.slice(ptr, ptr + data.length);
  Module._free(ptr);

  return result;
}
```

**Key Emscripten concepts:**
- `HEAP8`, `HEAP16`, `HEAP32`, `HEAPF32`, `HEAPU8`, etc. — TypedArray views into linear memory
- `_malloc` / `_free` — the C allocator, exposed to JS
- `cwrap` / `ccall` — JS wrappers for calling C functions with type conversion

**Emscripten is the right choice when:**
- You have existing C/C++ code (FFmpeg, SQLite, OpenCV, Bullet)
- You need POSIX compatibility shims
- You're porting a large codebase rather than writing new code

---

## wasm-pack — Rust to WASM

wasm-pack is the official tool for building Rust-based WASM libraries. It integrates with Cargo (Rust's package manager), generates TypeScript type definitions, and produces npm-compatible packages.

**Why Rust for WASM:**
- No garbage collector — no GC pauses in the WASM module
- Ownership system prevents memory errors without runtime overhead
- First-class WASM support in the Rust toolchain
- `wasm-bindgen` generates idiomatic JS/TS interop automatically

**Basic Rust WASM module:**

```rust
// src/lib.rs
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
pub fn apply_sepia(data: &mut [u8]) {
    for chunk in data.chunks_mut(4) {
        let r = chunk[0] as f32;
        let g = chunk[1] as f32;
        let b = chunk[2] as f32;

        chunk[0] = ((r * 0.393 + g * 0.769 + b * 0.189) as u32).min(255) as u8;
        chunk[1] = ((r * 0.349 + g * 0.686 + b * 0.168) as u32).min(255) as u8;
        chunk[2] = ((r * 0.272 + g * 0.534 + b * 0.131) as u32).min(255) as u8;
        // chunk[3] is alpha, unchanged
    }
}

#[wasm_bindgen]
pub struct ImageProcessor {
    width: u32,
    height: u32,
}

#[wasm_bindgen]
impl ImageProcessor {
    #[wasm_bindgen(constructor)]
    pub fn new(width: u32, height: u32) -> Self {
        Self { width, height }
    }

    pub fn dimensions(&self) -> String {
        format!("{}x{}", self.width, self.height)
    }
}
```

**Build and use:**

```bash
# Build for web (outputs to pkg/ directory)
wasm-pack build --target web

# Output: pkg/
#   image_processor.js      (JS glue)
#   image_processor_bg.wasm (binary)
#   image_processor.d.ts    (TypeScript types)
#   package.json
```

```typescript
// TypeScript usage — fully typed, no manual memory management
import init, { apply_sepia, ImageProcessor } from './pkg/image_processor.js';

await init(); // load and instantiate the WASM module

// Rust slice becomes a JS Uint8Array — memory is handled automatically
const imageData = new Uint8Array(canvas.width * canvas.height * 4);
apply_sepia(imageData); // wasm-bindgen handles the memory layout

// Rust structs become JS classes
const processor = new ImageProcessor(1920, 1080);
console.log(processor.dimensions()); // "1920x1080"
processor.free(); // explicit free for non-GC'd WASM objects
```

**wasm-bindgen's key contribution:** it generates the marshalling code for you. Rust slices become typed arrays, Rust structs become JS classes, and the generated TypeScript definitions let your editor autocomplete WASM exports.

---

## AssemblyScript — TypeScript-like Syntax for WASM

AssemblyScript is a language that looks like TypeScript but compiles directly to WASM. It's not TypeScript — it's a statically typed subset designed specifically for WASM targets.

**When to use AssemblyScript:**
- Your team knows TypeScript but not Rust or C/C++
- You want to write WASM without learning a new language ecosystem
- The use case is moderate complexity (doesn't need full Rust/C ecosystem)

**Limitations vs. Rust:**
- No Rust ecosystem (no `crates.io` packages)
- No borrow checker — memory safety is the programmer's responsibility
- Smaller community and less mature tooling
- Generated WASM is typically less optimized than Rust output

**Example:**

```typescript
// assembly/index.ts (AssemblyScript, not TypeScript)
// Types like i32, f64 are WASM primitives, not JS types

export function addInts(a: i32, b: i32): i32 {
  return a + b;
}

export function dotProduct(aPtr: i32, bPtr: i32, length: i32): f64 {
  let result: f64 = 0;
  for (let i: i32 = 0; i < length; i++) {
    const a = load<f64>(aPtr + i * 8);
    const b = load<f64>(bPtr + i * 8);
    result += a * b;
  }
  return result;
}
```

```bash
# Build
npx asc assembly/index.ts --target release --outFile build/release.wasm
```

```typescript
// Usage from JS — similar to any other WASM module
const { instance } = await WebAssembly.instantiateStreaming(fetch('/release.wasm'));
const add = instance.exports.addInts as (a: number, b: number) => number;
console.log(add(3, 4)); // 7
```

---

## Choosing a Toolchain

```
Do you have existing C/C++ code?
  → Emscripten (designed for exactly this)

Are you writing new compute-heavy code and performance is critical?
  → Rust + wasm-pack (best performance, best safety, best JS integration)

Does your team only know TypeScript and the use case is moderate?
  → AssemblyScript (TS-like syntax, smaller ecosystem)

Are you prototyping or exploring WASM concepts?
  → AssemblyScript (quickest to get started)
  → wasm-pack if your team has any Rust exposure
```

---

> **Check yourself:** What does `#[wasm_bindgen]` do in a Rust WASM project, and what does it generate?

---

## Self-Assessment

- [ ] I know what Emscripten is for and what types of projects should use it
- [ ] I understand what wasm-pack generates and why the TypeScript types are valuable
- [ ] I know what `wasm-bindgen` does in a Rust WASM project
- [ ] I can compare AssemblyScript's trade-offs vs. Rust for new WASM code
- [ ] I can reason about which toolchain to recommend given a project's requirements
- [ ] I understand why Rust is well-suited for WASM (no GC, ownership safety)

---

## Interview Q&A

**Q: What is Emscripten and what problem does it solve?** `High`

Emscripten is a compiler toolchain that translates C and C++ code into WebAssembly (and optionally JavaScript). Its primary use case is porting existing native libraries to the browser without rewriting them. Projects like ffmpeg.wasm (video processing), Ammo.js (Bullet physics engine), and SQLite-WASM all use Emscripten. It also generates JavaScript glue code that handles WASM instantiation, memory setup, and provides a JS-friendly API over the raw binary, so callers don't have to manage raw pointers manually.

---

**Q: What does wasm-pack do and why is it preferred over Emscripten for new Rust projects?** `High`

wasm-pack builds Rust code into WASM and generates an npm-compatible package that includes the WASM binary, JavaScript glue code, and TypeScript type definitions. The type definitions are generated by `wasm-bindgen`, which analyzes the Rust function signatures and produces accurate `.d.ts` files — so TypeScript code calling into WASM gets autocomplete and type checking. For new code, Rust with wasm-pack is preferred over Emscripten because Rust's ownership system prevents the memory bugs common in C/C++, Rust has no garbage collector so WASM modules have no GC pauses, and the developer experience is better.

---

**Q: What is AssemblyScript and how does it compare to Rust for WASM?** `Medium`

AssemblyScript is a language with TypeScript-like syntax that compiles to WASM. It uses WASM-native types (`i32`, `f64`) and provides `load`/`store` for direct memory access. Compared to Rust: AssemblyScript is easier to pick up for TypeScript developers, has a much smaller ecosystem (no equivalent of `crates.io`), lacks Rust's borrow checker so memory safety is the developer's responsibility, and typically produces less optimized output. It's a good fit for teams that need a moderate-complexity WASM module and don't have Rust expertise.

---

**Q: What is `wasm-bindgen` and what does it generate?** `Medium`

`wasm-bindgen` is a Rust library and code generation tool that creates the interop layer between Rust/WASM and JavaScript. It reads `#[wasm_bindgen]` annotations on Rust functions and structs, and generates: JavaScript glue code that handles string marshalling, memory allocation, and function binding; and TypeScript `.d.ts` declaration files that type the exports correctly. Without `wasm-bindgen`, calling WASM from JS would require manually managing pointers and byte offsets for everything except raw numbers. With it, Rust slices become typed arrays, Rust structs become JS classes with proper TypeScript types.

---

**Q: Why does Rust produce particularly good WASM compared to garbage-collected languages?** `Low`

Rust's ownership system ensures memory safety at compile time without a runtime garbage collector. When Rust compiles to WASM, the output has no GC pauses because there's no GC — memory is managed deterministically by Rust's ownership rules. Languages like Java, C#, or Go compile to WASM and bring their garbage collectors with them, adding binary size and non-deterministic pauses. Rust also gives the compiler more information about lifetimes and aliasing, enabling more aggressive optimization. The result is smaller, faster WASM with predictable performance characteristics.
