# OffscreenCanvas

## Quick Reference

| Concept | What it is | Why it matters |
|---|---|---|
| `OffscreenCanvas` | A canvas that can render in any thread (including Workers) | Canvas/WebGL work moves off the main thread |
| `transferControlToOffscreen()` | Detaches a visible `<canvas>` into an OffscreenCanvas | The visible canvas stays in the DOM; drawing happens in a Worker |
| Transfer semantics | OffscreenCanvas is a Transferable | Ownership passes to the Worker — only one thread can draw at a time |
| `commit()` | Pushes an OffscreenCanvas frame to the display | For manually controlled rendering (vs. WebGL/2D which push automatically) |

---

## What Is This?

`OffscreenCanvas` is a canvas rendering surface that can be created and used in any execution context — including Web Workers. Unlike a regular `<canvas>` (which is bound to the DOM and can only be used from the main thread), an `OffscreenCanvas` is a plain object with no DOM attachment.

The key use case: move expensive canvas rendering (complex 2D scenes, WebGL, particle systems, game loops) entirely to a Worker, so the main thread stays free for input handling and UI updates.

```js
// main.js
const canvas = document.getElementById('canvas');
const offscreen = canvas.transferControlToOffscreen();

const worker = new Worker('./renderer.js', { type: 'module' });
worker.postMessage({ canvas: offscreen }, [offscreen]); // transfer ownership

// worker.js
self.onmessage = (e) => {
  const canvas = e.data.canvas;          // OffscreenCanvas
  const ctx = canvas.getContext('2d');   // or 'webgl', 'webgl2', 'bitmaprenderer'

  function render() {
    ctx.clearRect(0, 0, canvas.width, canvas.height);
    ctx.fillStyle = '#f00';
    ctx.fillRect(Math.random() * 100, Math.random() * 100, 50, 50);
    requestAnimationFrame(render); // rAF works in Workers
  }
  render();
};
```

The visible `<canvas>` in the DOM is linked to the `OffscreenCanvas` — the browser composites whatever the Worker draws onto the screen automatically, without any main-thread involvement.

> **Check yourself:** After calling `transferControlToOffscreen()`, can the main thread still draw on the canvas directly?

---

## Why Does It Exist?

### The problem: canvas rendering + main thread = contention

A game loop or data visualization running on the main thread competes with:
- JavaScript event handlers
- React re-renders
- Layout and paint from other DOM mutations

A dropped frame in a canvas animation (WebGL game, chart, simulation) happens when the main thread is busy with something else — the rendering loop can't run. The only fix, before OffscreenCanvas, was to either reduce rendering complexity or accept frame drops.

Web Workers have always supported `ImageData` manipulation — you could do computation in a Worker and `postMessage` the pixel buffer back. But that approach serializes (copies) pixel data on every frame — expensive for large canvases.

OffscreenCanvas eliminates the copy: the Worker holds the rendering context directly. Frames are composited by the GPU without touching the main thread JS stack.

---

## How It Works

### Two creation paths

**Path 1: From a visible `<canvas>` element**

```js
const canvas = document.querySelector('canvas');
const offscreen = canvas.transferControlToOffscreen();
```

The visible canvas becomes "controlled" by the offscreen version. The original `canvas` element loses its 2D/WebGL context — you can no longer call `canvas.getContext()` on it. The offscreen version is then transferred to a Worker.

**Path 2: Standalone OffscreenCanvas (not attached to a visible element)**

```js
// In a Worker (or main thread)
const offscreen = new OffscreenCanvas(800, 600);
const ctx = offscreen.getContext('2d');
ctx.fillRect(0, 0, 100, 100);

// Get a snapshot as ImageBitmap (zero-copy)
const bitmap = offscreen.transferToImageBitmap();
// Send to main thread for display or compositing
self.postMessage({ bitmap }, [bitmap]);
```

This mode is useful for off-screen rendering pipelines — compute a frame, get a bitmap, display it.

### Contexts available

`OffscreenCanvas` supports all four context types:
- `'2d'` — same Canvas 2D API as in the browser
- `'webgl'` — WebGL 1
- `'webgl2'` — WebGL 2
- `'bitmaprenderer'` — render `ImageBitmap` objects directly (most lightweight)

### `requestAnimationFrame` in Workers

Inside a Worker, `self.requestAnimationFrame` is available when the Worker has an OffscreenCanvas. It syncs to the display's frame rate — the same as main-thread `rAF`. This gives you a proper game loop in the Worker:

```js
// renderer.js (Worker)
self.onmessage = (e) => {
  const ctx = e.data.canvas.getContext('2d');
  let x = 0;

  function frame() {
    ctx.clearRect(0, 0, 800, 600);
    ctx.fillRect(x, 100, 40, 40);
    x = (x + 2) % 800;
    requestAnimationFrame(frame);
  }
  requestAnimationFrame(frame);
};
```

The Worker's `rAF` fires at the display's vsync — even if the main thread is blocked for 200ms doing other work, the canvas animation continues rendering.

### `commit()` — manual frame submission

For non-animated use cases (custom rendering pipelines, manual double-buffering), `commit()` explicitly pushes the current canvas state to the display:

```js
ctx.fillRect(0, 0, 100, 100);
canvas.commit(); // push this frame to the compositor
```

When using `rAF` or WebGL, frames are submitted automatically (WebGL flushes on `gl.flush()` / end of frame, 2D on `rAF` tick). `commit()` is for cases where you're not using `rAF`.

---

## Practical: WebGL in a Worker

The most impactful use case — a full WebGL rendering engine off the main thread:

```js
// main.js
const canvas = document.querySelector('canvas');
const offscreen = canvas.transferControlToOffscreen();
const worker = new Worker('./webgl-worker.js', { type: 'module' });
worker.postMessage({ canvas: offscreen, width: canvas.width, height: canvas.height }, [offscreen]);

// Handle resize
window.addEventListener('resize', () => {
  worker.postMessage({ type: 'resize', width: canvas.clientWidth, height: canvas.clientHeight });
});

// Send user input to worker
canvas.addEventListener('mousemove', (e) => {
  worker.postMessage({ type: 'mouse', x: e.clientX, y: e.clientY });
});
```

```js
// webgl-worker.js
let gl, canvas;

self.onmessage = (e) => {
  if (e.data.canvas) {
    canvas = e.data.canvas;
    gl = canvas.getContext('webgl2');
    initScene(gl);
    requestAnimationFrame(render);
  }
  if (e.data.type === 'resize') {
    canvas.width = e.data.width;
    canvas.height = e.data.height;
    gl.viewport(0, 0, e.data.width, e.data.height);
  }
  if (e.data.type === 'mouse') {
    updateCamera(e.data.x, e.data.y);
  }
};

function render() {
  drawScene(gl);
  requestAnimationFrame(render);
}
```

The main thread handles user input (it must — event listeners are main-thread only), forwards events to the Worker via `postMessage`, and the Worker renders without any main-thread involvement.

> **Check yourself:** User input events (mousemove, click) fire on the main thread. In the WebGL-in-Worker pattern above, how do those events reach the Worker's render loop?

---

## `ImageBitmap` — Zero-copy Bitmap Transfer

`ImageBitmap` is a compressed, decoded image that can be transferred between threads as a Transferable:

```js
// Worker: render to OffscreenCanvas, export as ImageBitmap
const frame = offscreen.transferToImageBitmap(); // zero-copy, detaches from canvas
self.postMessage({ frame }, [frame]); // transfer to main thread

// Main thread: draw ImageBitmap to a visible canvas
const ctx = document.querySelector('canvas').getContext('bitmaprenderer');
ctx.transferFromImageBitmap(frame); // zero-copy display
```

This pattern is useful when you want the Worker to compute frames but the main thread to control compositing (e.g., overlaying UI on top).

---

## Gotchas

**1. After `transferControlToOffscreen()`, the `<canvas>` element is locked**
Calling `canvas.getContext()` after transfer throws. You can still read `canvas.width` and `canvas.height`, and resize events still fire — but drawing must happen through the OffscreenCanvas in the Worker.

**2. Only one owner at a time**
OffscreenCanvas is a Transferable. Once transferred to a Worker, the main thread loses access. If you try to `postMessage` it again, it throws. Design your architecture so ownership is clear.

**3. Worker `rAF` throttles in background tabs**
Like main-thread `rAF`, the Worker's `requestAnimationFrame` is throttled when the tab is in the background. This is usually desirable (saves battery) but can cause issues in tests or when running work in background tabs.

**4. CSS sizing vs. canvas sizing**
The `<canvas>` element's visual size (CSS `width`/`height`) and its pixel buffer size (`canvas.width`/`canvas.height`) are separate. `transferControlToOffscreen()` captures the pixel size at the moment of transfer. On `devicePixelRatio > 1` screens, you need to set `canvas.width = canvas.clientWidth * devicePixelRatio` before transfer, and handle resize events by messaging the Worker to update `canvas.width`.

**5. Text rendering in OffscreenCanvas has gaps**
Some text APIs (especially font loading with `FontFace`) have limited support inside Workers. If you need rich text rendering, do it on the main thread with a separate canvas and composite.

**6. Safari support is limited**
Safari added OffscreenCanvas support in version 16.4 (2023), but has had bugs. WebGL inside OffscreenCanvas in a Worker has had patchy support. Test carefully if Safari is in your browser matrix.

---

## Interview Questions

**Q (High): What is OffscreenCanvas and what problem does it solve?**

Answer: OffscreenCanvas is a canvas rendering surface that can be created and used in a Web Worker. It solves the contention between expensive canvas rendering (game loops, WebGL, complex data visualizations) and main-thread responsiveness. Before OffscreenCanvas, all canvas drawing had to happen on the main thread, competing with input handling, layout, and React rendering for CPU time. Dropped frames occurred whenever the main thread was busy. OffscreenCanvas moves the entire rendering pipeline to a Worker — the Worker holds the WebGL context, runs the render loop, and the GPU composites frames directly to the screen without main-thread involvement.

**Q (High): What happens to the original `<canvas>` element after `transferControlToOffscreen()`?**

Answer: The visible `<canvas>` element remains in the DOM and is still composited to the screen — whatever the Worker renders via the OffscreenCanvas appears in it. But the original `canvas` element loses its rendering context. Calling `canvas.getContext()` after transfer throws an error; the element is "controlled" by the OffscreenCanvas. The original canvas can still be resized (and the Worker notified via `postMessage`), but all drawing must go through the transferred OffscreenCanvas.

**Q (Medium): How do you handle user input in a Worker-based rendering architecture?**

Answer: DOM event listeners can only run on the main thread — Workers have no access to the DOM. In a Worker-based renderer, the main thread listens for input events (`mousemove`, `click`, `keydown`, `resize`) and forwards them to the Worker via `postMessage`. The Worker receives the coordinates or key codes, updates its internal state (camera, game logic, simulation), and those changes appear in the next rendered frame. The `postMessage` hop adds sub-millisecond latency for each event, which is generally acceptable. For extremely latency-sensitive input (pointer lock games), you'd minimize the transformation logic done on the main thread and let the Worker's state drive rendering.

**Q (Medium): What is the difference between `transferControlToOffscreen()` and `new OffscreenCanvas()`?**

Answer: `transferControlToOffscreen()` links the OffscreenCanvas to a visible `<canvas>` element — whatever you render in the Worker appears in the visible canvas automatically, composited by the browser. It's the right choice when you want a Worker to drive a visible canvas in the DOM. `new OffscreenCanvas(w, h)` creates a standalone offscreen buffer — it's not linked to any DOM element. It's useful for off-screen computation: render a frame, export it as an `ImageBitmap` via `transferToImageBitmap()`, and send the bitmap back to the main thread for display or further compositing.

**Q (Low): Why does `rAF` inside a Worker sync to the display's vsync, and what happens in background tabs?**

Answer: When a Worker has an OffscreenCanvas, the browser gives it access to `requestAnimationFrame`. The browser hooks this to the same vsync signal used for main-thread `rAF` — the display driver interrupt that fires ~60 or 120 times per second. This ensures the Worker's rendered frames arrive at the same cadence as the display, preventing tearing. In background tabs, the same throttling applies: `rAF` fires at 1fps or less (or is suspended) to conserve battery and CPU. This is usually desirable but can break tests or real-time simulations that need to run at full speed regardless of visibility.

---

## Self-Assessment

- [ ] Explain what `transferControlToOffscreen()` does to the original `<canvas>` element
- [ ] Write the main-thread + Worker setup for rendering to a visible canvas via OffscreenCanvas
- [ ] Explain how user input reaches a Worker-based rendering loop
- [ ] Describe the two creation paths for OffscreenCanvas and when to use each
- [ ] Explain the CSS size vs. pixel buffer size distinction and why it matters for high-DPI screens

---
*Next: Phase 8 — Browser Storage & Offline. Starting with Cookie Strategy: the `HttpOnly`, `SameSite`, `Secure`, and `Partitioned` attributes that define cookie security and the current landscape of third-party cookie deprecation.*
