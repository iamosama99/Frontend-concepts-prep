# Critical Rendering Path

## Quick Reference

| Stage | Input | Output | Blocks? |
|---|---|---|---|
| Parse HTML | HTML bytes | DOM tree | CSS and sync JS block parsing |
| Parse CSS | CSS bytes | CSSOM tree | Blocks render (not parsing) |
| JavaScript execution | JS bytes | Mutated DOM/CSSOM | Blocks HTML parsing (sync scripts) |
| Style | DOM + CSSOM | Render tree (visible nodes + styles) | — |
| Layout | Render tree | Box model geometry (position, size) | — |
| Paint | Layout tree | Layers of pixel instructions | — |
| Composite | Painted layers | Final frame on screen | — |

## What Is This?

The Critical Rendering Path is the sequence of steps the browser must complete before it can display the first pixel. It spans from receiving the first byte of HTML to producing the first frame — every delay in this path directly delays how long the user waits to see anything.

The stages are: parse HTML → build DOM → parse CSS → build CSSOM → execute JavaScript → combine into render tree → layout → paint → composite. These stages are partially sequential (layout can't start until the render tree exists) and partially parallel (HTML parsing and CSS download can happen simultaneously).

Understanding this pipeline is what separates performance *intuition* from guesswork. Every optimization technique — minifying CSS, deferring JS, inlining critical CSS — is justified by the specific stage it unblocks.

> **Check yourself:** If a browser finishes parsing HTML but the CSS hasn't arrived yet, can it paint anything? Why or why not?

## Why Does It Exist?

The browser receives HTML as raw bytes — it has no idea what the page looks like until it processes them. Each stage transforms one representation into another:

- HTML bytes → DOM (a structured model of the document)
- CSS bytes → CSSOM (a structured model of styles)
- DOM + CSSOM → Render tree (only visible nodes, with their computed styles)
- Render tree + geometry → Layout (where each box is, how large)
- Layout → Paint (the actual pixel drawing instructions)
- Layers → Composite (assembling layers into the final frame)

You can't paint what you haven't laid out. You can't lay out what you haven't styled. You can't style what isn't in the render tree. The pipeline's sequential dependency is fundamental.

## How It Works

### 1. HTML Parsing → DOM

The browser parses HTML bytes into tokens (open tag, text, close tag, attribute, etc.) and assembles them into the Document Object Model — a tree of nodes where each element, text node, and comment is a node.

Parsing is *incremental*: the browser doesn't need the full HTML document to start building the DOM. As bytes stream in, parsing and DOM construction proceed in parallel with the network download.

```html
<html>
  <body>
    <h1>Hello</h1>   ← this node is added to DOM as soon as parsed
    <p>World</p>     ← this follows
  </body>
</html>
```

**Parser-blocking scripts:** When the parser encounters a `<script>` tag without `async` or `defer`, it *stops parsing HTML* and waits for the script to download and execute. Why? The script might call `document.write()` which can inject HTML, changing what the parser would see next. This is why render-blocking scripts at the top of `<body>` delay everything that follows.

```html
<!-- Blocks HTML parsing until this downloads and executes -->
<script src="analytics.js"></script>

<!-- Does NOT block parsing — downloads and executes in parallel -->
<script src="analytics.js" async></script>

<!-- Does NOT block parsing — executes after HTML is parsed -->
<script src="analytics.js" defer></script>
```

**Speculative parsing:** Modern browsers use a "preload scanner" that looks ahead in the HTML even while the main parser is blocked by a script, queuing downloads for other resources it finds (CSS, images, other scripts). This overlaps network fetches with the blocked parser.

### 2. CSS Parsing → CSSOM

CSS is parsed into the CSS Object Model — a tree of style rules with specificity and cascade resolution applied. Like the DOM, it's a tree, but its structure doesn't mirror the HTML hierarchy; it's organized by selector scope.

**CSS is render-blocking but not parser-blocking.** The browser won't paint anything until it has the full CSSOM — because rendering requires knowing which styles apply to each node. But CSS doesn't block HTML parsing (the parser keeps going; it just can't render yet).

This means a slow CSS file delays FCP just as much as a slow HTML file. A CSS file linked in `<head>` is on the critical path.

```html
<!-- This stylesheet is on the critical path — delays first paint -->
<link rel="stylesheet" href="styles.css">

<!-- This is NOT on the critical path for print — media query -->
<link rel="stylesheet" href="print.css" media="print">

<!-- Only blocks rendering on screens matching the query -->
<link rel="stylesheet" href="mobile.css" media="(max-width: 768px)">
```

Stylesheets with media queries that don't match the current viewport are downloaded but don't block rendering.

**CSS also blocks JavaScript:** If a script follows a stylesheet in the HTML, the browser won't execute the script until the CSS is parsed. Reason: the script might query computed styles (`getComputedStyle`), which requires the CSSOM to be complete.

### 3. JavaScript Execution

JavaScript can read and modify both the DOM and the CSSOM. A script executed mid-parse can add nodes, remove nodes, change attributes, and inject styles. This mutability is why the parser must stop for synchronous scripts — the "document" the script sees must be consistent with what the parser has built so far.

The interaction:
- Sync `<script>` → blocks HTML parser
- CSS before sync `<script>` → CSS blocks the script (script waits for CSSOM)
- `async` script → downloads in parallel, executes as soon as downloaded (may run mid-parse)
- `defer` script → downloads in parallel, executes in order after HTML is fully parsed, before DOMContentLoaded

### 4. Render Tree Construction

The render tree is built by combining the DOM and the CSSOM. It contains only visible nodes — elements with `display: none` are excluded, `<head>`, `<script>`, and `<meta>` elements are excluded. Each node in the render tree has its computed styles attached.

```
DOM node <p class="intro"> + CSSOM rule .intro { color: red; font-size: 16px; }
= Render tree node: <p>, color: red, font-size: 16px, display: block, ...
```

Visibility rules:
- `display: none` → node and its subtree excluded from render tree
- `visibility: hidden` → node included in render tree (takes up space) but not painted
- `opacity: 0` → node included and painted (takes up space, triggers compositing)

> **Check yourself:** An element with `display: none` has no geometry and isn't painted. An element with `visibility: hidden` still participates in layout. What does this mean for an animation that toggles visibility?

### 5. Layout (Reflow)

Layout takes the render tree and computes the exact geometry of each element: its position in the viewport, its width and height, its relationship to other elements. The output is a box model for every visible node.

Layout is expensive because it's a cascade: the size of a parent can depend on its children's sizes, the position of an element can affect siblings. Changing a single element's dimensions can require the browser to recompute layout for a significant portion of the tree.

Layout is triggered by:
- Geometric property changes (width, height, padding, margin, border, top, left)
- DOM insertions and removals
- Font size changes
- Viewport resize

### 6. Paint

Paint converts the render tree (with geometry) into drawing instructions — "draw a rectangle at x,y with width/height/color" — organized into layers. The browser's painting algorithm is equivalent to drawing on a canvas: background colors, borders, text, images, and so on, in the correct stacking order.

Paint doesn't necessarily touch every pixel — it paints to layers. Layers that haven't changed don't need repainting.

### 7. Composite

The compositor takes the painted layers and assembles them into the final frame that's sent to the GPU for display. The compositor thread can do this entirely independently of the main thread — this is what allows scroll and transform animations to run at 60fps even when the main thread is busy.

Compositing is cheap. Layers on the GPU can be repositioned, scaled, rotated, and have their opacity changed without triggering layout or paint. This is why `transform` and `opacity` are the GPU-friendly CSS properties.

## The Critical Path's Performance Implications

**Minimize render-blocking resources:** Every CSS file and synchronous JS file in `<head>` delays FCP. The critical path is only as fast as the slowest resource on it.

**Critical CSS inlining:** CSS needed to render above-the-fold content can be inlined in `<style>` tags in `<head>`, eliminating the network round-trip. The rest of the CSS loads asynchronously.

**Resource priority:** The browser has a loading priority system. `<link rel="preload">` elevates a resource's priority so it downloads earlier in the waterfall, shortening the critical path.

**DOMContentLoaded vs. load:**
- `DOMContentLoaded` fires when HTML is parsed and deferred scripts have run — DOM is ready, CSSOM may still be loading
- `load` fires when all resources (images, CSS, scripts) have finished loading

FCP happens before `load`, usually close to when CSS finishes loading. Optimizing CRP means pushing FCP earlier.

## Gotchas

**CSS doesn't block HTML parsing, but it blocks rendering and script execution.** This is a common confusion. The parser keeps running while CSS loads; it just can't render or run scripts. The visual result is the same — the user sees nothing — but the parser is doing work in parallel.

**A single large synchronous script can delay rendering for all subsequent resources.** If a 200KB script is in `<head>` without `async` or `defer`, the parser stops, waits for the download, waits for execution, and only then continues. Everything after it in the HTML is delayed.

**Images are not render-blocking.** Images don't prevent the browser from painting. The page paints immediately without images; images render as they load. CLS (Cumulative Layout Shift) happens when images load and push content — unrelated to blocking.

**Inlining too much CSS defeats the cache.** Inlining critical CSS is good; inlining all CSS means the CSS is never cached separately and must re-download with every HTML document. Extract and inline only what's needed for above-the-fold.

**JavaScript that queries layout forces synchronous reflow.** Reading `element.offsetWidth`, `getBoundingClientRect()`, or `getComputedStyle()` forces the browser to flush pending style changes and complete layout synchronously before returning — even in the middle of a JavaScript task. This is a "forced synchronous layout" or "layout thrashing." More on this in the next topic.

## Interview Questions

**Q (High): What are the stages of the Critical Rendering Path, and what makes each stage a potential bottleneck?**

Answer: The CRP is: HTML parsing → DOM construction → CSS parsing → CSSOM construction → (JavaScript execution) → Render tree → Layout → Paint → Composite. Each stage is a bottleneck for different reasons. HTML parsing is slowed by synchronous scripts — the parser must stop and wait. CSSOM construction blocks rendering and script execution — a slow CSS download delays FCP even if the HTML is ready. JavaScript execution blocks the parser if the script is synchronous, and can delay the render tree if it manipulates the DOM mid-parse. Layout is expensive when many elements change geometry. Paint is expensive for complex visual effects (gradients, shadows, opacity). Compositing is cheap because it happens on the GPU, but promoting too many elements to their own layers wastes GPU memory. The practical focus for optimization: minimize render-blocking resources (inline critical CSS, defer JS) and reduce layout and paint work triggered by JavaScript.

The trap: Not knowing that CSS blocks rendering but not parsing. Confusing `DOMContentLoaded` (DOM ready) with FCP (first paint).

---

**Q (High): What is the difference between `async` and `defer` on a `<script>` tag, and when would you use each?**

Answer: Both `async` and `defer` download the script in parallel with HTML parsing without blocking the parser. They differ in when execution happens. `async`: the script executes as soon as it's downloaded, even mid-parse. Multiple async scripts execute in download-completion order, not document order. `defer`: the script executes after the HTML is fully parsed and the DOM is built, but before `DOMContentLoaded`. Multiple deferred scripts execute in document order. Use `async` for scripts that don't depend on other scripts and don't need the DOM — analytics, ad tags, chat widgets. Use `defer` for scripts that depend on DOM being ready or on each other's execution order — most application scripts. Omitting both (plain `<script>`) is for scripts that need to execute synchronously mid-parse, which is rarely correct in modern web development.

The trap: Thinking `async` is always better because it's "more parallel." Scripts that depend on the DOM or on each other will break if they execute mid-parse via async.

---

**Q (High): Why is CSS render-blocking, and what happens if a stylesheet hasn't loaded when JavaScript tries to run?**

Answer: CSS is render-blocking because the browser needs a complete CSSOM before it can construct the render tree — and without the render tree, it can't lay out or paint anything. Displaying content before CSS loads would create a Flash of Unstyled Content (FOUC): raw HTML with no styles, which then jumps to the styled version when CSS arrives. This is a jarring experience, so the browser waits. For JavaScript: if a sync script appears in the HTML after a `<link>` stylesheet, the browser delays script execution until the CSS is parsed. This prevents scripts from reading incorrect computed styles via `getComputedStyle()`. Implication: a slow CSS download delays both rendering and any subsequent script execution. This is why CSS should come first in `<head>`, load quickly (be small, use compression), and only include what's needed for above-the-fold content.

The trap: Thinking CSS only matters for visual rendering and doesn't interact with JavaScript. The CSS-blocks-script dependency is a real source of slow rendering that teams miss.

---

**Q (Medium): What is "critical CSS" and how does inlining it improve performance?**

Answer: Critical CSS is the minimal set of styles needed to render the above-the-fold content — what the user sees immediately on page load without scrolling. Inlining critical CSS directly in a `<style>` block in `<head>` eliminates the network round-trip for that CSS, allowing the browser to start building the render tree and painting immediately. Non-critical CSS (styles for below-fold content, hover states, rarely-used components) loads asynchronously. Tools like Critical, PurgeCSS, and Vite plugins can automatically extract critical CSS at build time. The trade-off: inlined CSS can't be cached separately by the browser — it must re-download with every HTML document. The solution is to inline only a small, minimal set (< 10–15KB) and load the rest asynchronously via `<link rel="preload" as="style">`.

The trap: Inlining all CSS or too much CSS. The cache trade-off matters — if your users visit multiple pages, cached CSS is more efficient than inlined-every-time CSS.

---

**Q (Low): How does the browser's preload scanner help performance, and what are its limitations?**

Answer: The preload scanner (also called the speculative parser) is a secondary HTML scanner that looks ahead in the HTML document even while the main parser is blocked by a synchronous script. Its sole job is to discover resources — `<link>`, `<img>`, `<script>` tags — and initiate network requests early. When the main parser is blocked waiting for `analytics.js` to download and execute, the preload scanner has already seen the following `<link rel="stylesheet">` and started downloading the CSS in parallel. This overlaps network fetches with the blocking period. Limitations: the preload scanner only discovers resources that are statically present in the HTML — it can't find dynamically injected resources (CSS added by JavaScript, images added by JS), `@import` rules inside CSS, or resources behind JavaScript-computed URLs. For those, explicit `<link rel="preload">` hints are needed.

The trap: Thinking the preload scanner makes parser-blocking scripts free. It helps with parallel downloads but doesn't unblock the parser — rendering is still delayed until the blocking script runs.

---

## Self-Assessment

Before moving on, check off each item you can do WITHOUT looking at the file.

- [ ] Can list the 7 stages of the CRP in order (Parse HTML, Parse CSS, JS Execute, Render Tree, Layout, Paint, Composite)
- [ ] Can explain why CSS is render-blocking but not parser-blocking
- [ ] Can explain the difference between `async` and `defer` scripts and when to use each
- [ ] Can explain why a script following a stylesheet in HTML must wait for the CSS to load before executing
- [ ] Can explain what DOMContentLoaded vs. load events represent
- [ ] Can explain why inlining all CSS is not the optimal approach

---
*Next: Reflow vs. Repaint vs. Composite-only Changes — the cost hierarchy of visual updates once the initial render is complete, and the techniques that keep animations at 60fps.*

*See also: [Phase 1 — Streaming SSR](../phase-01-rendering-strategies/05-streaming-ssr.md) — shows how streaming exploits the incremental nature of the CRP by flushing HTML in chunks so the browser can begin parsing and painting before the full response is ready.*
