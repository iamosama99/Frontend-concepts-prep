# Main Thread vs. Compositor Thread

## Quick Reference

| | Main Thread | Compositor Thread |
|---|---|---|
| What runs here | JavaScript, style, layout, paint | Compositing, scroll, transform/opacity animations |
| Blocking risk | JS tasks block everything else | Runs independently of JS |
| Frame deadline | Must produce layer tree before compositor can run | Must composite before vsync deadline |
| Can be interrupted | No (run-to-completion) | Yes — at vsync boundaries |
| Optimization goal | Keep tasks < 50ms; yield often | Avoid promoting too many layers |

## What Is This?

The browser doesn't do all its work on a single thread. The rendering architecture is specifically designed to separate work that *must* touch JavaScript and the DOM (main thread) from work that can run independently (compositor thread). This separation is why your CSS animation can run at 60fps even when JavaScript is executing a long computation.

**Main thread:** Runs JavaScript, style calculation, layout, and paint. Single-threaded. When JavaScript runs, everything else on this thread waits.

**Compositor thread:** Takes the layer list produced by the main thread and assembles the final frame. Handles scroll, `transform`, `opacity`. Runs continuously at the display's refresh rate. Does not need to wait for JavaScript.

When these two threads work in concert, you get smooth rendering. When the main thread falls behind — when JavaScript tasks run too long — only main-thread-dependent rendering suffers. Compositor-thread operations keep running, which is why scroll still feels smooth even on a janky page (to a degree).

> **Check yourself:** A CSS animation on `transform` is running at 60fps. JavaScript freezes the main thread for 200ms with a synchronous computation. Does the animation stop? Why or why not?

## Why Does It Exist?

The browser's core tension: JavaScript is single-threaded and can take arbitrarily long, but the display must update at 60fps (every 16.67ms) for smooth rendering. If all rendering work — including compositing — happened on the main thread, any JavaScript task longer than 16ms would cause a dropped frame.

The solution: move as much rendering work as possible off the main thread. The compositor thread runs entirely independently, communicating with the main thread only to receive an updated layer list. If the main thread doesn't have an updated layer list in time for a frame, the compositor uses the previous one — it keeps running, producing the best frame it can with stale data.

This is why two categories of performance problem feel different to users:
- **Jank:** Dropped frames from the main thread missing its paint deadline — choppy, inconsistent motion
- **Stuck:** The main thread is so overwhelmed it can't process input events — clicks and taps don't register

## How It Works

### The Rendering Pipeline Across Threads

```
Main Thread:
  JavaScript → Style → Layout → Paint → Commit layer tree
                                                    ↓
Compositor Thread:                         Receive layer tree
                                          → Composite → Display
                                          → (also handles scroll input)
                                          → (also handles transform/opacity animations)
```

The "commit" step is the handoff between threads: the main thread finishes painting, produces a new layer tree, and hands it to the compositor. The compositor then takes over and assembles the final frame on its own schedule.

### What the Main Thread Owns

**JavaScript execution:** All JavaScript runs on the main thread. A 100ms task blocks layout, paint, and input handling for 100ms.

**Style calculation:** The browser computes which CSS rules apply to which DOM nodes. This runs on the main thread. For pages with thousands of rules and nodes, this can take tens of milliseconds.

**Layout:** Computing the geometry of the entire render tree. Main thread only. Any JavaScript that reads layout-triggering properties (offsetWidth, getBoundingClientRect) forces the main thread to complete any pending layout work synchronously.

**Paint:** Rasterizing the render tree into layer bitmaps. Main thread, though Chrome uses a dedicated raster thread pool for the actual pixel-drawing work — the main thread produces the display list, raster workers execute it.

### What the Compositor Thread Owns

**Compositing frames:** Reading the layer tree and assembling the final image. Does not need the main thread.

**Scroll:** When the user scrolls, the compositor thread moves the scroll layer independently. It does not wait for JavaScript to acknowledge the scroll. This is why native browser scroll is always smooth — it's compositor-driven, not JavaScript-driven.

**CSS animations on `transform` and `opacity`:** Once the browser hands these animations to the compositor (at the start), the compositor interpolates the animation values each frame. JavaScript doesn't run them frame-by-frame.

**Mouse/touch input routing:** The compositor thread receives input events first, determines which layer they're over, and decides whether the event should be dispatched to JavaScript or handled purely in the compositor (e.g., for scroll).

### The 16ms Budget

At 60fps, the browser has 16.67ms between frames. Within that budget:

1. Main thread must complete any pending JavaScript tasks
2. Main thread runs style + layout + paint if needed
3. Main thread commits an updated layer tree (or compositor uses previous)
4. Compositor composites the layer tree
5. Frame is delivered to the display

If steps 1–3 take longer than ~12ms (leaving 4ms for compositing), the frame is late. If the main thread takes 30ms, the compositor might still produce a frame using the previous layer tree — scroll and transform animations continue, but new JavaScript-driven changes aren't reflected until the next frame.

> **Check yourself:** What's the difference between a "janky" animation and a "frozen" page from a threading perspective?

### Input Handling and `passive` Event Listeners

When the compositor thread receives a scroll or touch event, it needs to know whether any JavaScript event listener might call `preventDefault()` to cancel the scroll. If such a listener exists, the compositor must wait for JavaScript to run the listener before proceeding with scroll — it can't scroll the page if JavaScript is going to cancel it.

This is a significant source of scroll jank: if you have a `touchstart` or `touchmove` listener anywhere in the scroll chain, the compositor waits for JavaScript on every touch event. Even if the listener never calls `preventDefault()`, the compositor can't know that without running it.

The fix: `passive: true` on event listeners is a promise to the browser that the listener will never call `preventDefault()`. The compositor can now scroll without waiting.

```javascript
// Bad — compositor must wait for JS on every touch event
window.addEventListener('touchstart', handleTouch);

// Good — compositor can scroll immediately; JS still runs but doesn't block scroll
window.addEventListener('touchstart', handleTouch, { passive: true });
```

Chrome automatically makes touch and wheel listeners passive on `window`, `document`, and `body` to prevent accidental scroll blocking, but explicit listeners on child elements still need `passive: true`.

### `requestAnimationFrame` and Thread Synchronization

`requestAnimationFrame` callbacks run on the main thread, just before the paint step. They're synchronized to the compositor's rendering cycle — they fire once per frame, at the right moment to make changes that will appear in the next painted frame.

```javascript
// This runs in sync with the rendering pipeline
function animateWithRaf(timestamp) {
  element.style.transform = `translateX(${easeOutCubic(timestamp)}px)`;
  requestAnimationFrame(animateWithRaf);
}
requestAnimationFrame(animateWithRaf);
```

This is correct for JavaScript-driven animations that can't be expressed as pure CSS. But it runs on the main thread — if the main thread is busy, the rAF callback is delayed and the animation drops frames. CSS animations on `transform`/`opacity` are compositor-driven and don't have this problem.

The newer `Animation` API and `Web Animations API` can run CSS-compatible animations that the browser may promote to the compositor:

```javascript
element.animate(
  [{ transform: 'translateX(0)' }, { transform: 'translateX(100px)' }],
  { duration: 300, easing: 'ease-out' }
);
// Browser may run this on the compositor thread
```

## Off-Main-Thread Work

Modern browser architectures push more work off the main thread:

**Raster threads:** Actual pixel painting (rasterization) happens in a thread pool, not on the main thread. The main thread produces a "display list" (drawing instructions), and raster threads execute it in parallel.

**Web Workers:** JavaScript computation can be moved off the main thread entirely. Workers have no DOM access but can do any computation — parsing, sorting, compression, image processing — without blocking rendering.

**Worklets (CSS Houdini):** Paint Worklets run in a separate context from the main thread, allowing custom CSS painting without blocking.

**OffscreenCanvas:** Canvas rendering can be moved to a Worker thread.

## Gotchas

**Scroll event listeners vs. native scroll:** JavaScript `scroll` events fire on the main thread, *after* the compositor has already scrolled. They're asynchronous from the compositor's perspective. `IntersectionObserver` and `ResizeObserver` are better APIs for responding to scroll/resize because they're batched and don't force the compositor to wait.

**`transform: translateZ(0)` is not free.** This old "GPU hack" promotes the element to a compositor layer. It prevents that element from being painted with its neighbors, which can hurt performance if the element doesn't actually benefit from layer promotion. Profile before applying.

**JavaScript-driven scroll (`scrollTop`, `scrollIntoView`) is main-thread-driven.** Unlike native scroll (compositor-driven), programmatically setting `scrollTop` goes through the main thread. If the main thread is busy, programmatic scroll is delayed. This is why programmatic scroll can feel unresponsive during heavy computation.

**`will-change` triggers layer creation on the main thread.** The initial promotion — creating the texture — is a paint operation on the main thread. It's fast, but happens synchronously. Applying `will-change` at runtime during an animation can cause a first-frame hiccup.

**`pointer-events: none` doesn't move hit testing off the main thread.** Hit testing (which element did the user click?) is still computed by the compositor thread for scroll, but JavaScript event dispatch is always on the main thread. Complex hit-testing logic in event handlers still costs main thread time.

## Interview Questions

**Q (High): Why do CSS animations on `transform` survive a jank-causing JavaScript task, but animations that change `width` don't?**

Answer: CSS animations on `transform` (and `opacity`) are handed off to the compositor thread at the start of the animation. The compositor interpolates values and produces frames on its own thread, completely independent of the main thread. When a JavaScript task runs for 200ms and monopolizes the main thread, the compositor continues running at 60fps — it doesn't need the main thread to produce the next animation frame. Animations that change `width` (or any layout property) are layout-based: changing `width` requires recomputing geometry for the element and its neighbors, which is main thread work. The compositor cannot perform layout recalculation — it only assembles pre-painted layers. So width-based animations are tied to the main thread's schedule. When JavaScript blocks the main thread, width animations stall.

The trap: "CSS animations are always smooth." Only compositor-thread animations are immune to main-thread congestion. CSS animations on `width`, `height`, or any layout-triggering property still require main-thread work.

---

**Q (High): What is the purpose of `passive: true` on event listeners and when is it critical?**

Answer: When a touch or wheel event listener is registered, the compositor thread must wait for JavaScript to run that listener before proceeding with scroll — because the listener might call `preventDefault()` to cancel scroll. This wait turns smooth compositor-driven scroll into main-thread-blocked scroll. Even a listener that never calls `preventDefault()` causes this wait. `passive: true` is a declaration that the listener will never call `preventDefault()`. With this promise, the compositor doesn't wait — it scrolls immediately while the event listener runs concurrently. It's critical for scroll and touch event listeners on mobile where scroll performance is already constrained. Touch handlers for swipe-to-dismiss, pull-to-refresh, and custom gesture recognition should always be `passive: true` unless they genuinely need to cancel scroll.

The trap: Thinking `passive: true` is just a performance hint with no correctness implications. It prevents `preventDefault()` from being called — if your listener does call it, `passive: true` silently ignores it and you lose the ability to cancel scroll.

---

**Q (High): What work runs on the main thread vs. compositor thread, and why does the split matter for performance?**

Answer: Main thread: JavaScript execution, style calculation, layout, paint (producing display lists), commit. Compositor thread: compositing layers into frames, scroll, transform/opacity CSS animations, input hit-testing for scroll. The split matters because the main thread is shared with JavaScript — any long-running JS task delays rendering work on the main thread. The compositor thread is independent — it keeps producing frames at 60fps even when the main thread is occupied. This means: scroll will remain smooth (compositor-driven) even during heavy JavaScript. CSS animations on `transform`/`opacity` will keep running. But JavaScript-driven animations (`requestAnimationFrame` that changes `left`/`top`, React state updates that cause DOM changes) will drop frames. The performance optimization goal is to minimize main-thread work per frame (< 12ms) and use compositor-thread operations wherever possible.

The trap: Thinking the compositor thread eliminates all rendering concerns. Main-thread-bound operations (most JavaScript-driven UI changes) still require meeting the 16ms budget.

---

**Q (Medium): How does programmatic scroll (`element.scrollTop`) differ from native user scroll, and what are the implications?**

Answer: Native user scroll is compositor-driven: when the user drags their finger or uses the mouse wheel, the compositor thread receives the input and moves the scroll layer independently of the main thread. It's smooth because it never waits for JavaScript. Programmatic scroll (`element.scrollTop = 500`, `element.scrollIntoView()`) is main-thread-driven: it requires the main thread to update the scroll position and trigger re-layout/repaint as needed. If the main thread is busy executing JavaScript, programmatic scroll is delayed until the current task completes. Implications: programmatic scroll during heavy computation feels laggy. Smooth-scrolling animations via JavaScript are particularly vulnerable — each frame's scroll update goes through the main thread. For smooth programmatic scroll, use CSS `scroll-behavior: smooth` (can be compositor-assisted) or the `element.scrollTo({ behavior: 'smooth' })` API, which some browsers handle more efficiently.

The trap: Assuming all scroll is compositor-smooth. Native scroll is; programmatic scroll is main-thread-dependent.

---

**Q (Low): What are Web Workers and how do they relate to the main thread/compositor thread model?**

Answer: Web Workers run JavaScript in a background thread completely separate from the main thread and the compositor thread. They have no access to the DOM (no document, no window, no CSS) but can run arbitrary computation — parsing, sorting, image processing, compression, cryptography, WebAssembly. By moving CPU-heavy work to a Worker, you free the main thread to focus on style, layout, paint, and responding to input. The connection back to the DOM is `postMessage` — Workers send results back to the main thread, which then updates the DOM. In the three-thread model: main thread handles DOM and rendering; compositor thread handles compositing and scroll; Worker threads handle computation. The goal is to keep both main and compositor threads light enough to hit their frame deadlines. Workers are the correct solution for long-running computation that would block the main thread.

The trap: Thinking Workers can directly modify the DOM. They can't. The boundary is the `postMessage` bridge, which is asynchronous.

---

## Self-Assessment

Before moving on, check off each item you can do WITHOUT looking at the file.

- [ ] Can list what runs on the main thread (JS, style, layout, paint) vs. compositor thread (compositing, scroll, transform/opacity animations)
- [ ] Can explain why `transform` animations survive a long JS task but `width` animations don't
- [ ] Can explain what `passive: true` does and when failing to use it causes scroll jank
- [ ] Can explain why programmatic scroll (`scrollTop`) is less smooth than native scroll
- [ ] Can describe what Web Workers enable and what their DOM access limitation is
- [ ] Can explain the 16ms frame budget and what happens when the main thread misses it

---
*Phase 2 complete. Next phase: Networking & Data Communication — protocols, patterns, and the trade-offs of how data moves between browser and server.*
