# GPU Layers & Compositing

## Quick Reference

| Concept | Mechanism | Implication |
|---|---|---|
| Compositor layer | A portion of the page rasterized as a GPU texture | Changes to that region don't require repaint of the rest of the page |
| Layer promotion | Browser assigns element its own layer (implicit or via `will-change`) | Transform/opacity changes on that layer are GPU-only, off main thread |
| Compositing | GPU blends layers in order to produce the final frame | Scroll, transform, opacity changes can be done purely in GPU |
| Layer memory cost | Each layer stores a texture in GPU VRAM | Too many layers exhaust GPU memory; quality degrades or app crashes |

## What Is This?

During the paint step, the browser doesn't necessarily paint the entire page onto a single canvas. It organizes the page into multiple **layers** — independent regions that are painted separately and stored as GPU textures. The compositor then takes these textures and blends them together in the correct stacking order to produce the final visible frame.

This two-step separation (paint layers → composite) is what makes certain kinds of visual changes extremely cheap: instead of repainting pixels, the GPU just repositions or blends existing textures. Moving a layer from x=0 to x=100 is updating a number in a matrix — the texture stays the same.

Think of it like Photoshop layers: each layer is an independent image, and compositing is the final step of flattening them into one. Moving a layer in Photoshop doesn't require redrawing it; it just changes its position in the stack.

> **Check yourself:** If `transform: translateX(200px)` only repositions a GPU texture, what is the texture's content? When was it last drawn?

## Why Does It Exist?

The GPU is designed for exactly this kind of work. It has thousands of small parallel cores optimized for texture blending, matrix multiplication, and pixel operations. The browser's compositor thread runs directly on the GPU (or communicates with it), independently of the main JavaScript thread.

Without compositing, every visual change — scroll position, an animated tooltip, a sticky header — would require the main thread to repaint pixels and flush them to the display. The main thread is shared with JavaScript. When JavaScript is running, the main thread is occupied, and rendering stalls. This is the root cause of jank: JavaScript tasks blocking repaints.

By moving compositing to the GPU and separating it from the main thread, the browser can continue rendering at 60fps even when JavaScript is executing long tasks. Scroll and CSS animations on `transform`/`opacity` keep working because they don't need the main thread.

## How It Works

### Layer Creation

The browser creates compositor layers automatically for certain elements:
- The root document layer (always exists)
- Elements with CSS animations on `transform` or `opacity`
- Elements with `position: fixed` or `position: sticky`
- Elements with 3D transforms (`translateZ`, `rotate3d`)
- `<video>`, `<canvas>`, `<iframe>` elements
- Elements overlapping other promoted elements (implicit promotion — often unintended)
- Elements with `will-change: transform` or `will-change: opacity`

When an element gets its own layer, it gets its own backing texture in GPU memory. The texture is a rasterized (pixel) snapshot of that element's painted content.

### Explicit Promotion with `will-change`

You can explicitly request a layer for an element before it animates:

```css
.flyout-menu {
  will-change: transform;
  /* Browser creates a GPU texture for this element immediately */
}
```

Without `will-change`, when a CSS animation starts on `transform`, the browser promotes the element to its own layer at the start of the animation. This promotion itself requires a paint (to create the texture) and costs one frame. With `will-change`, the texture exists before the animation starts — no first-frame jank.

The older "hack" (still seen in legacy code) is using a 3D transform to force promotion:
```css
.element {
  transform: translateZ(0);  /* or translate3d(0,0,0) */
  /* forces layer promotion — workaround pre-will-change */
}
```

This works because the browser promotes elements with 3D transforms. It's less semantic than `will-change` but is supported in older browsers.

### How the Compositor Uses Layers

After the paint step, the compositor has a stack of textures. To produce a frame:

1. For each layer, calculate its transform matrix (position, scale, rotation)
2. For each layer, calculate its opacity
3. Blend layers in back-to-front order according to z-index
4. Output the composited image to the display

Step 1 and 2 can be modified frame-by-frame without any repaint. This is what makes CSS transitions on `transform` and `opacity` GPU-accelerated: the browser prepaints the layer once and then updates the matrix/opacity value in the compositor on each animation frame.

```css
/* Every frame of this animation only updates the compositor matrix */
/* No JS, no layout, no paint — pure GPU */
@keyframes slide-in {
  from { transform: translateX(-100%); }
  to { transform: translateX(0); }
}

.panel {
  animation: slide-in 300ms ease;
  will-change: transform;  /* pre-promote to avoid first-frame paint */
}
```

### Implicit Layer Squashing and Explosion

An important subtlety: if element A is on its own layer (due to a CSS animation) and element B is partially stacked on top of A in z-order, the browser may need to give B its own layer too — otherwise compositing A would blend incorrectly with unpromoted B content. This is called **implicit layer promotion** or **layer squashing**.

In complex UIs with many promoted elements, layer squashing can cause dozens or hundreds of unintended layers. This is the "layer explosion" problem: more layers than you intended, consuming GPU memory for textures the GPU doesn't need.

To inspect layers in Chrome DevTools: Rendering panel → Layer Borders (green = layer, orange = tiles). The Layers panel shows the full layer tree with memory cost.

## GPU Memory Implications

Each compositor layer stores a texture whose size is proportional to the element's pixel dimensions (times device pixel ratio). A 1000×500px element on a 2× DPR screen stores a 2000×1000 pixel texture = 8MB of GPU VRAM (4 bytes per pixel, RGBA).

A page that promotes 50 elements — each a card, a modal, a sidebar — could consume hundreds of MB of GPU memory. On a desktop this is often fine. On a mid-range mobile device with 256MB–512MB GPU memory shared with system graphics, this causes:
- GPU memory pressure
- Texture eviction (textures get swapped back to CPU RAM)
- Slower compositing as evicted textures must be re-uploaded
- In extreme cases, browser crashes or blank layers

The rule: promote only elements that genuinely need GPU compositing for performance reasons (elements that animate frequently, the scroll layer, fixed elements). Remove `will-change` from elements that have finished animating.

## Tiled Rendering

The browser doesn't paint a single monolithic texture per layer. For large layers (like the main page scroll layer), it divides the layer into tiles (typically 256×256 or 512×512 pixels). Only tiles within or near the viewport get fully rasterized. Off-screen tiles may be rasterized at a lower resolution or not at all.

This is why you see "checkerboard" patterns in browsers when scrolling fast — the browser is displaying unfilled tiles because the rasterizer hasn't caught up. The tile-based model allows the browser to prioritize rendering what's visible and defer off-screen work.

> **Check yourself:** A full-page background image is on the main layer. Scrolling changes which tiles are visible. Does this require a repaint? Does it require compositing?

## Practical Optimization Patterns

**Promote animated elements before the animation starts:**
```css
.drawer {
  will-change: transform;  /* layer exists before interaction */
  transform: translateX(-100%);  /* starts off-screen */
  transition: transform 300ms;
}
.drawer.open {
  transform: translateX(0);  /* pure compositor operation */
}
```

**Don't promote everything — profile first:**
```javascript
// Bad: promotes every card on the page
.card { will-change: transform; }

// Better: only promote during hover interaction
el.addEventListener('mouseenter', () => el.style.willChange = 'transform');
el.addEventListener('mouseleave', () => el.style.willChange = 'auto');
```

**Use DevTools to find unintended layers:**
Chrome DevTools → More tools → Layers panel shows every layer, its memory cost, and why it was created. "Layer was created because X" reasons include CSS animations, `will-change`, explicit 3D transforms, and implicit overlap promotion.

**Prefer `transform` for any position/size change during animation:**
Instead of changing `left`, `top`, `width`, `height` in animation frames, calculate the final visual appearance and express it as `transform`. This keeps the animation in the compositor and off the main thread.

## Gotchas

**`overflow: hidden` with `border-radius` can force a repaint.** When a promoted child element is clipped by a parent with `overflow: hidden` and `border-radius`, the browser can't do the clipping in the compositor (which doesn't know about the parent's border-radius). It falls back to painting the clipped result instead of compositing the layer. This is a common source of unexpected paint during animations.

**Fixed-position elements always get their own layer.** `position: fixed` creates a compositor layer on every browser, because fixed elements don't scroll with the page and must be composited at the right position independently. Having many fixed elements (headers, CTAs, cookie banners, live chat bubbles) means many layers.

**Filters can force repaint.** CSS `filter: blur()` and `filter: drop-shadow()` require pixel-level manipulation and are often done on the CPU, not the GPU, even on promoted layers. They trigger paint rather than composite-only changes. `filter: hue-rotate` is GPU-accelerated in some browsers.

**`transform` doesn't affect layout flow.** An element with `transform: translateX(100px)` visually moves 100px to the right, but its layout box stays in its original position. Other elements still flow around its original position. This is the trade-off of composite-only changes: they're off-layout, so they don't interact with the document flow.

## Interview Questions

**Q (High): What is a compositor layer and when does the browser create one?**

Answer: A compositor layer is a portion of the page's content that is rasterized into its own GPU texture and composited separately from the rest of the page. The browser creates layers automatically for: elements with CSS animations or transitions on `transform` or `opacity`, elements with `position: fixed` or `position: sticky`, elements with 3D transforms, `<video>`, `<canvas>`, and `<iframe>` elements, and elements with `will-change: transform` or `will-change: opacity`. The browser also creates layers implicitly when promoted elements need to be correctly composited with stacked neighbors (layer squashing). The significance: changes to `transform` or `opacity` on a composited layer are handled entirely by the GPU compositor, without involving the main thread or requiring repaint — enabling smooth 60fps animations independent of JavaScript execution.

The trap: Confusing when layers are created with when they help. Not every layer promotion is beneficial — unintended implicit layers consume GPU memory with no benefit.

---

**Q (High): Why can transform/opacity animations run at 60fps even when the main thread is busy with JavaScript?**

Answer: The compositor thread runs independently from the main thread. While JavaScript executes on the main thread, the compositor thread continues running on its own schedule, reading the layer list and producing frames. CSS animations and transitions on `transform` and `opacity` on composited layers are handed off to the compositor at the start of the animation — the compositor interpolates the values frame by frame without involving the main thread. A JavaScript task that takes 100ms doesn't interrupt the compositor's 60fps animation cycle, because they operate on separate threads. This is the fundamental reason to use `transform` over `left`/`top`: position-based layout animations must go through the main thread for layout recalculation, while `transform` is a compositor-only operation.

The trap: Thinking "the browser pauses animations while JavaScript runs." It only pauses main-thread-dependent animations (layout/paint-based). Compositor-only animations are immune to main thread congestion.

---

**Q (Medium): What is the GPU memory cost of compositor layers and how does over-promotion hurt performance?**

Answer: Each compositor layer stores a GPU texture whose size is proportional to its rendered pixel dimensions. A 1000×500px element on a 2× DPR device requires a 2000×1000px texture ≈ 8MB of GPU VRAM. On a page with many promoted elements — each card, sticky header, animated tooltip, dropdown — memory adds up quickly. On constrained mobile devices with limited GPU VRAM (256–512MB shared with system graphics), over-promotion causes texture eviction: the GPU swaps textures back to CPU RAM, requiring re-upload on every compositor pass. This makes compositing slower and can cause the GPU to miss frame deadlines, producing jank — the opposite of the intended result. The fix: profile with DevTools Layers panel, remove `will-change` from elements that aren't actively animating, and apply promotion only where there's a measured performance benefit.

The trap: "More layers is always better." Memory pressure from over-promotion causes degradation, not improvement, especially on mobile.

---

**Q (Medium): What is implicit layer promotion (layer squashing) and when does it occur?**

Answer: When an element is promoted to its own compositor layer, the browser must ensure that elements stacked on top of it in z-order are correctly composited. If a non-promoted element overlaps a promoted one, the browser can't composite the promoted layer correctly without knowing the visual appearance of the overlapping element. To resolve this, the browser may promote the overlapping element to its own layer too — even if nothing requested it. This "implicit" promotion cascades: if that newly promoted element overlaps another, that one gets promoted too. In a complex z-ordered UI, promoting one element (e.g., a sticky header) can trigger layer creation for dozens of other elements, rapidly consuming GPU memory. To diagnose: Chrome DevTools → Layers panel shows "Layer was created because: element overlaps other composited content."

The trap: Not knowing implicit promotion exists. Teams often add `will-change` to one element and are surprised to see the page's layer count jump by 20.

---

**Q (Low): Why does `position: fixed` always create a compositor layer, and what are the performance implications of many fixed elements?**

Answer: Fixed elements are positioned relative to the viewport, not the document. They don't scroll with the page. During scroll, the compositor moves the main scroll layer, but fixed elements must stay at their viewport-relative positions. The only way to do this is to put fixed elements on their own layers — the compositor can then keep their position constant while scrolling the main layer behind them. Every `position: fixed` element gets its own layer automatically. Performance implications: each fixed element consumes GPU memory. Common fixed elements in production UIs — sticky nav, floating action button, chat widget, cookie banner, A/B test overlays — can add up to 8–10 layers on a single page. On memory-constrained devices, this is a real concern. Minimize fixed elements; consider whether `position: sticky` (which only creates a layer when actively "stuck") achieves the same UI goal with less overhead.

The trap: Not knowing `position: fixed` always promotes. Or not knowing the difference between `fixed` and `sticky` in terms of layer creation.

---

## Self-Assessment

Before moving on, check off each item you can do WITHOUT looking at the file.

- [ ] Can explain what a compositor layer is and give three reasons the browser creates one
- [ ] Can explain why transform/opacity animations run on the GPU independently of JS
- [ ] Can explain the GPU memory cost of layers and why over-promotion degrades performance
- [ ] Can describe implicit layer promotion and what triggers it
- [ ] Can explain why `position: fixed` always creates a layer
- [ ] Can name the DevTools panel used to inspect compositor layers

---
*Next: Main Thread vs. Compositor Thread — the architectural separation that makes smooth scroll and animation possible, and exactly what work belongs on each thread.*
