# MutationObserver

## Quick Reference

| Concept | Detail |
|---|---|
| Purpose | Observe changes to the DOM tree: attribute changes, child additions/removals, text changes |
| Replaces | Deprecated `MutationEvent` API (synchronous, killed performance) |
| Callback timing | Microtask — fires after current script finishes, before next task |
| `observe()` options | `attributes`, `childList`, `subtree`, `characterData`, `attributeFilter`, `oldValue` |
| Entry type | `MutationRecord` — describes each individual mutation |
| `takeRecords()` | Flush pending (undelivered) mutations synchronously |

---

## What Is This?

`MutationObserver` watches a DOM node and notifies you when it changes. Changes can be: attributes added/removed/changed, child nodes inserted or removed, or text content changed. You configure exactly what changes to watch, and the browser batches and delivers them asynchronously as an array of `MutationRecord` objects.

---

## Why Microtask Timing Matters

MutationObserver callbacks fire as microtasks — after the current JS task completes but before the browser paints or handles the next event. This means:

```js
const observer = new MutationObserver((mutations) => {
  console.log('2. Mutations delivered');
});
observer.observe(document.body, { childList: true });

document.body.appendChild(document.createElement('div'));
console.log('1. Appended');

// Output order:
// 1. Appended
// 2. Mutations delivered
// (then browser paints)
```

All DOM mutations made in the same synchronous task are batched into a single callback invocation. This is efficient — if you append 100 elements in a loop, you get one callback with 100 `MutationRecord` entries, not 100 callbacks.

> **Check yourself:** Why was the deprecated `MutationEvent` API (which fired synchronously) a performance problem that MutationObserver was designed to solve?

---

## The `observe()` Options

```js
observer.observe(targetNode, {
  // Choose which mutations to watch:
  attributes: true,          // attribute additions, removals, changes
  attributeFilter: ['class', 'data-state'], // only these attributes (subset of attributes: true)
  attributeOldValue: true,   // include old value in MutationRecord

  childList: true,           // direct child additions/removals
  subtree: true,             // extend childList/attributes to all descendants

  characterData: true,       // text node content changes
  characterDataOldValue: true, // include old text value
});
```

None of these are on by default — you must enable at least one or `observe()` throws.

---

## MutationRecord

Each record describes a single mutation:

```js
const observer = new MutationObserver((mutations) => {
  mutations.forEach(record => {
    record.type;          // 'attributes' | 'childList' | 'characterData'
    record.target;        // the node that was mutated
    record.attributeName; // attribute name if type === 'attributes', else null
    record.oldValue;      // previous value (only if attributeOldValue/characterDataOldValue: true)
    record.addedNodes;    // NodeList of added nodes (type === 'childList')
    record.removedNodes;  // NodeList of removed nodes (type === 'childList')
    record.previousSibling; // sibling before added/removed nodes
    record.nextSibling;     // sibling after added/removed nodes
  });
});
```

---

## Use Cases

### 1. Reacting to third-party DOM changes

When a third-party library, analytics script, or browser extension mutates the DOM and you need to respond:

```js
// Watch for a dynamically injected cookie banner and auto-dismiss
const observer = new MutationObserver((mutations) => {
  for (const { addedNodes } of mutations) {
    for (const node of addedNodes) {
      if (node.matches?.('#cookie-banner')) {
        node.querySelector('[data-dismiss]')?.click();
        observer.disconnect();
      }
    }
  }
});

observer.observe(document.body, { childList: true, subtree: true });
```

### 2. Attribute-driven behavior in non-framework code

When you want to react to attribute changes without a framework:

```js
const observer = new MutationObserver((mutations) => {
  for (const { attributeName, target, oldValue } of mutations) {
    if (attributeName === 'aria-expanded') {
      const isExpanded = target.getAttribute('aria-expanded') === 'true';
      updatePanelVisibility(target, isExpanded);
    }
  }
});

observer.observe(document.querySelector('.accordion'), {
  attributes: true,
  attributeFilter: ['aria-expanded'],
  subtree: true,
});
```

### 3. Custom Elements (internal use)

When a custom element needs to react to changes in its light DOM children — `MutationObserver` is the way to watch light DOM from inside a shadow root:

```js
class DynamicList extends HTMLElement {
  connectedCallback() {
    this._observer = new MutationObserver(() => this._renderList());
    this._observer.observe(this, { childList: true, subtree: false });
    this._renderList();
  }

  disconnectedCallback() {
    this._observer.disconnect();
  }

  _renderList() {
    const items = [...this.children].map(child => child.textContent);
    this.shadowRoot.querySelector('.list').innerHTML =
      items.map(item => `<li>${item}</li>`).join('');
  }
}
```

### 4. Polyfilling features

Many browser feature polyfills rely on MutationObserver to detect when elements matching a new feature are inserted:

```js
// Simplified — how polyfills watch for new target elements
const polyfill = new MutationObserver((mutations) => {
  for (const { addedNodes } of mutations) {
    for (const node of addedNodes) {
      if (node.nodeType === Node.ELEMENT_NODE) {
        node.querySelectorAll('[data-tooltip]').forEach(initTooltip);
      }
    }
  }
});

polyfill.observe(document.body, { childList: true, subtree: true });
```

---

## `takeRecords()`

Synchronously flushes any pending (queued but not yet delivered) mutation records:

```js
const pending = observer.takeRecords();
// pending is an array of MutationRecord — same format as callback receives
// the queue is cleared — these records won't be delivered to the callback
```

Useful in cleanup: before disconnecting, call `takeRecords()` to process any outstanding mutations before they're discarded:

```js
function cleanup() {
  const pending = observer.takeRecords();
  if (pending.length) processMutations(pending);
  observer.disconnect();
}
```

---

## Performance Considerations

**`subtree: true` is expensive.** Watching an entire document subtree for any change means every DOM mutation in the page triggers an observation. Use the narrowest possible scope — watch a specific container, not `document.body`, unless necessary.

**`attributeFilter` is cheaper than `attributes: true`.** Specifying exactly which attributes to watch means the browser only notifies on relevant changes, not on every attribute mutation in the subtree.

**Callback work should be fast.** The callback fires as a microtask and can delay rendering if it's heavy. For intensive work, schedule with `requestAnimationFrame` or a scheduler.

**Never mutate the observed node inside the callback.** It triggers another mutation, which triggers another callback — infinite loop. If you must mutate, `disconnect()` first, mutate, then `observe()` again, or use a flag to skip re-entrant calls.

---

## Gotchas

**NodeList in `addedNodes`/`removedNodes` is live.** By the time you iterate it, the DOM may have changed again. Spread into an array: `[...record.addedNodes]`.

**MutationObserver does not watch style changes via `element.style.property = ...`** unless they're reflected as attribute changes. Inline style mutations are observable via `attributes: true` (the `style` attribute), but computed style changes (via stylesheet) are invisible to MutationObserver.

**Shadow DOM is a boundary.** An observer on a host element does not observe changes inside its shadow root. Observe the shadow root directly to watch shadow DOM mutations.

**Disconnect in cleanup.** A connected MutationObserver keeps the target node and the callback closure alive (prevents GC). Always `disconnect()` when no longer needed.

---

## Interview Questions

**Q (High): Why did browsers deprecate `MutationEvent` and replace it with `MutationObserver`?**

Answer: `MutationEvent` (e.g., `DOMNodeInserted`, `DOMAttrModified`) fired synchronously — in the middle of the DOM mutation, before control returned to the code that made the change. This had two catastrophic consequences. First, the event handler could make additional DOM changes, which fired more events, recursively, creating cascading synchronous callbacks that could stall the browser for seconds. Second, even without mutation, the browser couldn't batch DOM changes for efficiency — every single DOM write had to pause, fire an event, wait for all handlers, then resume. This made even simple bulk DOM operations (appending 100 nodes) dramatically slower. `MutationObserver` fixes both problems: callbacks fire as microtasks after the current script, and all mutations in the same task are batched into one callback call. This allows browsers to optimize bulk DOM operations normally and deliver a single efficient notification afterward.

**Q (High): What options would you use to watch only for `class` attribute changes on all descendants of a container?**

Answer: `observer.observe(container, { attributes: true, attributeFilter: ['class'], subtree: true })`. `attributes: true` enables attribute observation. `attributeFilter: ['class']` limits notifications to only the `class` attribute — without this, every attribute change on every observed node fires a callback. `subtree: true` extends the observation to all descendants of the container, not just the container itself. Combining `subtree` with `attributeFilter` is the performant pattern: you get notifications for the specific attribute across the whole subtree without the noise of every other attribute change. Adding `attributeOldValue: true` would also include the previous class value in `record.oldValue` if you need to compute a diff.

**Q (Medium): When would you call `takeRecords()` and why?**

Answer: `takeRecords()` synchronously flushes the pending mutation queue — mutations that have occurred but haven't been delivered to the callback yet (because the microtask hasn't run). The two main use cases: first, cleanup — before calling `disconnect()`, call `takeRecords()` to retrieve and process any pending mutations that would otherwise be lost when the observer is disconnected. Second, synchronous reads — if your code needs to know the current state of mutations right now (not asynchronously), `takeRecords()` gives you the records immediately. After `takeRecords()`, the queue is cleared — those records won't appear in the next callback invocation.

**Q (Low): Why can't MutationObserver see changes to an element's computed style?**

Answer: MutationObserver watches the DOM — specifically, the `style` attribute (inline styles) and other attributes, child node additions/removals, and text content. It does not have access to the style engine. When a CSS rule in a stylesheet changes an element's computed color, for example, no attribute on the element changes — the stylesheet is external to the element. Even if you add or remove a class and a CSS rule targeting that class changes the element's appearance, MutationObserver only sees the class attribute change (which you can observe with `attributeFilter: ['class']`), not the resulting visual change. To react to computed style changes, you must use ResizeObserver (for size changes), or observe the class/attribute that drives the style change, not the style effect itself.

---

## Self-Assessment

- [ ] Explain why MutationEvent was deprecated and how MutationObserver solves the same problem
- [ ] Write the observe() call to watch only `data-state` attribute changes on all descendants of `.sidebar`
- [ ] Explain what `subtree: true` does and when to avoid it
- [ ] Describe when to use `takeRecords()` and what happens to those records afterward
- [ ] Explain why mutating an observed node inside the callback is dangerous

---
*Next: PerformanceObserver — observing LCP, CLS, FID, and other performance entries programmatically.*
