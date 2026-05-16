# Worklets: Audio, Paint, Animation

## Quick Reference

| Worklet type | API | Runs in | What you extend |
|---|---|---|---|
| AudioWorklet | Web Audio API | Audio rendering thread | Custom DSP / audio processing nodes |
| PaintWorklet | CSS Houdini (Paint API) | Compositor (browser-defined) | Custom CSS `background`, `border-image`, `mask` values |
| AnimationWorklet | Web Animations API | Compositor thread | Scroll-linked, time-linked custom animations |
| LayoutWorklet | CSS Houdini (Layout API) | Browser layout thread | Custom CSS `display` layout algorithms |

---

## What Is This?

Worklets are lightweight, sandboxed execution contexts designed to plug into specific browser rendering pipelines. They are NOT general-purpose threads like Web Workers. Each Worklet type runs inside a particular browser internal context — the audio rendering thread, the compositor thread, the layout engine — and can only do the narrow thing that pipeline needs.

The defining trait: **Worklets run outside the main thread but also outside a full Worker context**. They are more restricted than Workers (no `fetch`, no IndexedDB, no arbitrary async operations) and more coupled to browser internals (they're called by the browser, not by your code, and must return synchronously in some cases).

```js
// Register a paint worklet (CSS Houdini)
await CSS.paintWorklet.addModule('./my-painter.js');

// CSS
.card { background: paint(confetti); }
```

> **Check yourself:** Why can't a PaintWorklet use `fetch()` to load a texture on the fly?

---

## Why Do Worklets Exist?

The common thread: developers want to extend browser behavior at a rendering-critical layer, but doing so on the main thread causes jank. The three alternatives before Worklets were:

1. **Do it on the main thread** — blocks rendering, degrades INP
2. **Do it in a Web Worker** — can't touch the rendering pipeline at all
3. **Wait for the browser to add a native API** — slow, vendor-driven

Worklets are the escape hatch: a sandboxed context that the browser explicitly invites user code into, at a specific hook point, under controlled constraints. The browser controls when the Worklet runs; you control what it does.

---

## AudioWorklet

### What it is

The Web Audio API has an internal audio rendering thread that processes audio in fixed-size chunks (typically 128 samples at 44100Hz, so ~2.9ms per chunk). AudioWorklet lets you write custom audio processing code that runs on that thread — close to the metal, in real time.

```js
// main.js
const context = new AudioContext();
await context.audioWorklet.addModule('./gain-processor.js');

const gainNode = new AudioWorkletNode(context, 'gain-processor');
gainNode.connect(context.destination);
```

```js
// gain-processor.js (runs in audio thread)
class GainProcessor extends AudioWorkletProcessor {
  static get parameterDescriptors() {
    return [{ name: 'gain', defaultValue: 1, minValue: 0, maxValue: 1 }];
  }

  process(inputs, outputs, parameters) {
    const input = inputs[0][0];   // first input, first channel
    const output = outputs[0][0];
    const gain = parameters.gain[0];

    for (let i = 0; i < input.length; i++) {
      output[i] = input[i] * gain;
    }
    return true; // keep processor alive
  }
}

registerProcessor('gain-processor', GainProcessor);
```

The `process()` method is called synchronously, on every audio frame, by the audio rendering thread. It must complete within one frame (~2.9ms at 44.1kHz). If it doesn't, you get audio glitches.

Before `AudioWorklet`, the only options for custom audio processing were `ScriptProcessorNode` (ran on the main thread — caused audio glitches on any main-thread jank) or native nodes (no customization). `AudioWorklet` moved processing to the audio thread entirely.

Communication with the main thread uses `MessagePort` (via `this.port`) — the same mechanism as Workers, but the audio thread context is much more constrained: no `fetch`, no IndexedDB, ideally no allocations inside `process()`.

### The `SharedArrayBuffer` + `Atomics` bridge

For low-latency parameter updates, `postMessage` has too much overhead (it goes through the event loop). High-performance audio apps use `SharedArrayBuffer` to share state between the main thread and the audio worklet with zero message-passing:

```js
// main.js
const sab = new SharedArrayBuffer(4);
const sharedParam = new Float32Array(sab);
gainNode.port.postMessage({ sab }); // one-time setup

// Update gain with no postMessage overhead
sharedParam[0] = 0.5;
```

---

## PaintWorklet (CSS Houdini — CSS Painting API)

### What it is

PaintWorklet lets you write a custom CSS paint function — a JavaScript function that receives a canvas-like `PaintRenderingContext2D` and draws whatever you want. The result appears wherever you use `paint(worklet-name)` in CSS.

```js
// main.js
await CSS.paintWorklet.addModule('./checker-painter.js');

// CSS
.element {
  --checker-size: 20;
  background: paint(checkerboard);
}
```

```js
// checker-painter.js
registerPaint('checkerboard', class {
  static get inputProperties() {
    return ['--checker-size'];
  }

  paint(ctx, geometry, properties) {
    const size = parseInt(properties.get('--checker-size').toString()) || 10;
    const cols = Math.ceil(geometry.width / size);
    const rows = Math.ceil(geometry.height / size);

    for (let row = 0; row < rows; row++) {
      for (let col = 0; col < cols; col++) {
        ctx.fillStyle = (row + col) % 2 === 0 ? '#eee' : '#fff';
        ctx.fillRect(col * size, row * size, size, size);
      }
    }
  }
});
```

The PaintWorklet is called by the browser whenever the element needs repainting — on resize, on CSS variable change (`--checker-size`), or when the browser decides. The key constraint: `paint()` **must be synchronous** and **must be deterministic** (same inputs → same output). No `fetch`, no timers, no async operations.

`inputProperties` declares which CSS custom properties the worklet observes; the browser only re-invokes `paint()` when those change, providing implicit memoization.

> **Check yourself:** Why must the PaintWorklet's `paint()` function be synchronous and deterministic?

### Current state

PaintWorklet is available in Chrome/Edge. Firefox and Safari lack support. In practice it's used for: complex gradient effects, animated borders (via CSS animations driving custom properties), procedural textures, and patterns that would be expensive with SVG filters.

---

## AnimationWorklet

### What it is

AnimationWorklet runs custom animation logic on the compositor thread, synchronized with the compositor's tick, rather than `requestAnimationFrame` on the main thread. This means it keeps animating even when the main thread is busy — critical for scroll-linked effects.

```js
await CSS.animationWorklet.addModule('./parallax-animator.js');

const scrollTimeline = new ScrollTimeline({
  source: document.scrollingElement,
  orientation: 'vertical',
});

new WorkletAnimation(
  'parallax',             // worklet name
  new KeyframeEffect(
    document.querySelector('.hero'),
    [{ transform: 'translateY(0)' }, { transform: 'translateY(-50%)' }],
    { duration: 1000, fill: 'both' }
  ),
  scrollTimeline
).play();
```

```js
// parallax-animator.js
registerAnimator('parallax', class {
  animate(currentTime, effect) {
    effect.localTime = currentTime * 0.5; // half-speed parallax
  }
});
```

The `animate()` function runs on the compositor thread — the same thread doing scroll position updates. This gives you jank-free scroll-linked animations that are unaffected by main-thread work.

**Current state:** AnimationWorklet has largely been superseded by the `ScrollTimeline` + `ViewTimeline` CSS APIs (now in Chrome 115+ and shipping broadly) which achieve the same result declaratively. AnimationWorklet is still in the spec but Chrome's implementation is behind a flag.

---

## LayoutWorklet (Bonus: CSS Layout API)

A fourth Worklet type — lets you define custom `display:` values with custom layout algorithms (think: implementing a masonry grid layout natively). Registration is similar:

```js
await CSS.layoutWorklet.addModule('./masonry.js');
// CSS: display: layout(masonry);
```

Very experimental, Chrome-only, not production-ready.

---

## Worklets vs. Workers

| | Web Worker | Worklet |
|---|---|---|
| Purpose | General off-thread computation | Extend a specific browser pipeline |
| Who calls your code | You (via postMessage) | The browser (at a pipeline hook) |
| Async operations | Full support (fetch, IDB, timers) | Severely restricted (often none) |
| Lifecycle | Your control | Browser's control |
| Global scope | `DedicatedWorkerGlobalScope` | Worklet-specific scope |
| Multiple instances | One per `new Worker()` | Browser may create multiple instances |

The "browser may create multiple instances" point is critical for PaintWorklet: the browser can instantiate multiple copies of your painter class to satisfy parallel rendering needs. **Worklet instances cannot have shared mutable state**. No module-level singletons that mutate — the browser will break your assumptions.

---

## Gotchas

**1. PaintWorklet instances are not stable**
The browser can create any number of PaintWorklet instances and may destroy and recreate them. State stored on `this` between `paint()` calls is not guaranteed to persist. Any state needed for a repaint must come from CSS properties or be recomputable each frame.

**2. `paint()` is called with unknown frequency**
The browser decides when to call `paint()`. It may be called many times per second on resize, or throttled during low-power mode. Never rely on `paint()` call count for animation — use CSS animations / transitions that drive `inputProperties` instead.

**3. AudioWorklet `process()` must not allocate**
Garbage collection can cause audio glitches — the GC pause interrupts audio frame processing. Pre-allocate all buffers before `process()` runs. No `new Float32Array()` inside `process()`.

**4. No shared state between Worklet instances and the main thread (except via message/SAB)**
PaintWorklet and AudioWorklet run in isolated contexts. You can't read a JS variable from the main thread directly — only through `inputProperties`, `parameters`, or `postMessage`/`SharedArrayBuffer`.

**5. HTTPS required for Worklets**
Same as Service Workers — Worklets only register on secure contexts.

**6. Browser support is fragmented**
PaintWorklet: Chrome/Edge only. AnimationWorklet: effectively deprecated in favor of ScrollTimeline. AudioWorklet: broadly supported (Chrome, Firefox, Safari). Always check MDN before using Houdini APIs in production.

---

## Interview Questions

**Q (High): What is a Worklet and how does it differ from a Web Worker?**

Answer: A Worklet is a lightweight, sandboxed context for extending a specific browser rendering pipeline — audio processing, custom CSS paint, scroll-linked animation. Unlike Web Workers, Worklets are called by the browser (not your code), run at a specific hook in the browser's internal pipelines, and are severely restricted in what they can do (no fetch, no general async). Workers are general-purpose — you control when they run and what they do. The tradeoff: Workers give you flexibility but can't touch rendering pipelines; Worklets give you direct access to those pipelines but only at a narrow hook point.

The trap: candidates say "Worklets are just small Workers" — that misses the key inversion of control. The browser invokes Worklets, not the developer.

**Q (High): Why does AudioWorklet exist, and what problem did `ScriptProcessorNode` have?**

Answer: Web Audio needed a way for developers to write custom DSP code. `ScriptProcessorNode` (now deprecated) processed audio on the main thread — any main-thread jank (layout, script execution) caused audio glitches because the processing thread couldn't get its time slice. AudioWorklet moves the processing to the dedicated audio rendering thread, which the browser prioritizes separately from the main thread. Even with a fully blocked main thread, audio processing continues uninterrupted. The `process()` callback must be synchronous and complete within one audio frame (~2.9ms at 44.1kHz).

**Q (Medium): What is the CSS Painting API (PaintWorklet) and what constraints does it impose?**

Answer: The CSS Painting API lets developers write a JavaScript function that renders to a canvas-like context, and use the result anywhere a CSS image value is accepted (background, border-image, mask). The function — `paint(ctx, geometry, properties)` — must be synchronous and deterministic: same CSS property inputs must always produce identical output. The browser may create multiple instances and may call `paint()` at any time. No async operations, no external state, no side effects. CSS custom properties declared in `inputProperties` are the only inputs; the browser only re-invokes `paint()` when those change, providing implicit caching.

**Q (Medium): Why should a PaintWorklet not store state on `this` between `paint()` calls?**

Answer: The browser may create multiple instances of a Worklet class for parallelism and may recreate them at any time. State on `this` persists only within a single instance's lifetime and is not shared across instances. If two instances are created, they have separate `this` — any state stored in one is invisible to the other. Worse, if the browser recreates an instance (e.g., after memory pressure), state is lost. Worklet instances must be treated as stateless and deterministic — all information must come from CSS properties, geometry, or other browser-provided inputs.

**Q (Low): When would you still choose an AnimationWorklet over CSS ScrollTimeline?**

Answer: For complex, imperative animation logic that cannot be expressed declaratively — for example, scroll position that maps through a custom easing function, or animations that depend on multiple scroll sources simultaneously, or animations that need to read and compare multiple timeline values and branch. CSS ScrollTimeline and ViewTimeline cover the majority of scroll-linked animation cases declaratively and should be preferred where they fit. AnimationWorklet is the escape hatch for animations too complex for CSS keyframes.

---

## Self-Assessment

- [ ] Explain the difference between a Worklet and a Web Worker in terms of who calls your code
- [ ] Write a minimal AudioWorklet processor skeleton (class + `process()` signature) from memory
- [ ] Explain why `paint()` must be synchronous and deterministic
- [ ] Name the three main Worklet types and which browser pipeline each extends
- [ ] Explain why PaintWorklet instances should not store mutable state on `this`

---
*Next: SharedArrayBuffer & Atomics — the only way to share raw memory between JS threads, enabling true zero-copy parallelism but requiring careful use of atomic operations to avoid data races.*
