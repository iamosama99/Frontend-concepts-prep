# HTML Templates & Slots

## Quick Reference

| Feature | Purpose |
|---|---|
| `<template>` | Inert HTML fragment — parsed but not rendered, cloneable |
| `<slot>` | Placeholder inside shadow DOM where light DOM children are projected |
| `<slot name="...">` | Named slot — receives only light DOM elements with matching `slot="..."` attribute |
| Default slot | Unnamed `<slot>` — receives all light DOM content not assigned to a named slot |
| `slotchange` event | Fires on `<slot>` when assigned nodes change |
| `slot.assignedNodes()` | Returns the nodes projected into a slot at runtime |

---

## What Is This?

`<template>` and `<slot>` are two separate but often used-together browser primitives for Web Components.

**`<template>`** is an HTML element whose content is parsed into a `DocumentFragment` but never rendered. It's inert — scripts don't execute, images don't load, styles don't apply — until you explicitly clone and insert the content into the document.

**`<slot>`** is the composition mechanism inside Shadow DOM. It lets light DOM children (the content placed between a custom element's opening and closing tags) be *projected* into specific locations in the shadow tree, without being moved there. The content stays in the light DOM; the slot just determines where it appears visually.

---

## Why They Exist

Without `<template>`, stamping out repeated HTML structures requires innerHTML concatenation (XSS risk) or imperative DOM construction (`createElement`, `appendChild`). Templates provide a parsed, ready-to-clone HTML fragment with no security risk and no performance cost until clone time.

Without `<slot>`, Shadow DOM would be a black box — consumers couldn't put content inside a component. Slots are the API surface for content composition, analogous to React `children` and Vue's `<slot>` (which borrowed the same concept).

> **Check yourself:** Why is cloning a `<template>`'s content faster than parsing innerHTML on each use?

---

## The `<template>` Element

### Defining a template

```html
<template id="card-template">
  <article class="card">
    <header>
      <h2 class="card-title"></h2>
    </header>
    <section class="card-body"></section>
  </article>
</template>
```

The HTML inside `<template>` is **parsed** (the browser validates it and builds the DOM tree) but the resulting `DocumentFragment` is **not rendered**. No images load, no scripts run, no styles apply.

### Cloning and using a template

```js
const template = document.getElementById('card-template');
const clone = template.content.cloneNode(true); // deep clone

// Mutate the clone before inserting
clone.querySelector('.card-title').textContent = 'My Card';

// Insert into document — only now does it render
document.body.appendChild(clone);
```

`template.content` is a `DocumentFragment`. `cloneNode(true)` deep-clones all children. After `appendChild`, the DocumentFragment itself is consumed (its children move to the target), so each use requires a fresh clone.

### Templates inside custom elements

The common pattern: define the template once, clone per element instance:

```js
const template = document.createElement('template');
template.innerHTML = `
  <style>
    :host { display: block; }
    .card { border: 1px solid #ddd; padding: 16px; }
  </style>
  <article class="card">
    <slot name="title"><span>Untitled</span></slot>
    <slot></slot>
  </article>
`;

class CardElement extends HTMLElement {
  constructor() {
    super();
    const shadow = this.attachShadow({ mode: 'open' });
    shadow.appendChild(template.content.cloneNode(true));
  }
}

customElements.define('card-element', CardElement);
```

The template is created once at module evaluation time — not per constructor call. Each element instance gets a cheap clone, not a full re-parse.

---

## Slots: Content Projection

### Default slot

An unnamed `<slot>` captures all light DOM content not assigned to a named slot:

```html
<!-- Shadow root template -->
<div class="wrapper">
  <slot></slot>
</div>
```

```html
<!-- Usage -->
<my-wrapper>
  <p>This paragraph is projected into the default slot.</p>
  <span>So is this span.</span>
</my-wrapper>
```

The `<p>` and `<span>` visually appear inside `.wrapper` but remain in the light DOM — they're not moved into the shadow tree.

### Named slots

Named slots allow targeted content distribution. The slot has a `name` attribute; the light DOM element declares which slot it belongs to with the `slot` attribute:

```html
<!-- Shadow root template -->
<article class="card">
  <header>
    <slot name="title">Default Title</slot>
  </header>
  <div class="body">
    <slot></slot>  <!-- default slot for body content -->
  </div>
  <footer>
    <slot name="actions"></slot>
  </footer>
</article>
```

```html
<!-- Usage -->
<card-element>
  <h2 slot="title">My Article</h2>
  <p>Body content goes into the default slot.</p>
  <button slot="actions">Read More</button>
</card-element>
```

Rules:
- Light DOM element with `slot="title"` → goes to `<slot name="title">`
- Light DOM element with no `slot` attribute → goes to the default `<slot>`
- Light DOM element with `slot="unknown"` → rendered nowhere (orphaned)

### Slot fallback content

Content inside a `<slot>` element is the fallback — shown when nothing is projected into that slot:

```html
<slot name="title">
  <span class="placeholder">Untitled</span>  <!-- shown only when slot is empty -->
</slot>
```

If the consumer provides content for that slot, the fallback is replaced entirely.

---

## Slot Assignment: What Actually Happens

Slotted content is not moved to the shadow DOM — it's *flattened* for rendering. The light DOM children are *assigned* to slots:

```js
const slot = element.shadowRoot.querySelector('slot[name="title"]');

// Get assigned nodes (what's currently projected into this slot)
slot.assignedNodes(); // returns light DOM nodes assigned to this slot
slot.assignedNodes({ flatten: true }); // also includes fallback content if slot is empty
slot.assignedElements(); // same but only Element nodes (no text nodes)
```

This is useful for reading the actual projected content from inside the component:

```js
connectedCallback() {
  const titleSlot = this.shadowRoot.querySelector('slot[name="title"]');
  const assignedTitle = titleSlot.assignedElements()[0];
  if (assignedTitle) {
    document.title = assignedTitle.textContent; // sync page title to slot content
  }
}
```

---

## The `slotchange` Event

`slotchange` fires on a `<slot>` element when the set of nodes assigned to it changes — content added, removed, or replaced:

```js
class MyCard extends HTMLElement {
  constructor() {
    super();
    const shadow = this.attachShadow({ mode: 'open' });
    shadow.innerHTML = `<slot name="title"></slot><slot></slot>`;

    const titleSlot = shadow.querySelector('slot[name="title"]');
    titleSlot.addEventListener('slotchange', (e) => {
      const nodes = e.target.assignedNodes();
      console.log('Title slot content changed:', nodes);
      // React to new content — update ARIA labels, sync external state, etc.
    });
  }
}
```

**Important:** `slotchange` fires asynchronously (as a microtask after the DOM mutation). It does not fire during initial rendering.

---

## Practical Patterns

### Observing slot content for ARIA

When a component needs to expose the text content of a slot (e.g., for an aria-label fallback):

```js
connectedCallback() {
  const slot = this.shadowRoot.querySelector('slot');
  const updateLabel = () => {
    const text = slot.assignedNodes()
      .map(n => n.textContent)
      .join('').trim();
    if (text) this.setAttribute('aria-label', text);
  };
  slot.addEventListener('slotchange', updateLabel);
  updateLabel(); // initial read
}
```

### Styling slotted content

From inside the shadow root, use `::slotted()` — limited to direct children:

```css
::slotted(*) {
  margin: 0; /* remove default margin from any slotted element */
}

::slotted(h2) {
  font-size: 1.5rem;
}

/* Invalid — cannot select descendants of slotted elements */
::slotted(div p) { color: red; } /* ignored */
```

From outside (the light DOM), styles apply normally since slotted content lives in the light DOM.

---

## Gotchas

**Slot only works in Shadow DOM.** `<slot>` is meaningless outside a shadow root — the browser doesn't recognize it as a projection mechanism.

**Slotted content stays in the light DOM.** CSS applied to slotted content from inside `::slotted()` is limited. The consumer's (light DOM) styles still apply and take precedence because the specificity battle happens in the light DOM.

**`slotchange` fires asynchronously.** You cannot read slot content synchronously right after the element is connected. Listen for `slotchange` or use `requestAnimationFrame` / microtask to defer the read.

**Dynamically added slot content doesn't guarantee ordering.** If multiple light DOM children are added in rapid succession, `slotchange` may fire once for all of them combined (batch), or once per change depending on the implementation.

**Template content is a DocumentFragment, not a real DOM.** Querying `template.content.querySelector()` works. But event listeners attached to template content are lost on `cloneNode()` — you must re-attach them after cloning.

---

## Interview Questions

**Q (High): What is the difference between `<template>` and setting innerHTML, and when does it matter?**

Answer: Both parse HTML, but `<template>` parses into an inert `DocumentFragment` that is never rendered until explicitly inserted. innerHTML assigns parsed content directly into the live DOM, immediately rendering it and executing any scripts. `<template>` is therefore faster for repeated use: parse once at definition time, clone cheaply at runtime. It's also safer from certain parsing edge cases — `<template>` content is parsed in a context where certain elements like `<td>` without a table ancestor are valid, whereas they'd be silently dropped in innerHTML. For custom elements that stamp out structure per instance, the template-clone pattern avoids re-parsing the same HTML for every element instance. The performance difference is significant when creating hundreds of instances.

**Q (High): What does it mean that slotted content "stays in the light DOM"?**

Answer: When light DOM children are assigned to a `<slot>` in a shadow root, they are not moved — they remain in the light DOM as children of the custom element. The slot is a rendering placeholder. The browser's rendering engine "flattens" the composed tree (shadow tree + assigned nodes) for display, so the slotted content visually appears inside the shadow root. But in terms of DOM APIs (`parentNode`, `children`, `querySelector`), the slotted elements are still children of the custom element in the light DOM, not of any shadow root element. This has real implications: light DOM styles still apply to slotted content, `document.querySelector` can find them, form elements inside slots are still associated with surrounding forms, and event listeners on them work normally. The shadow root does not own them.

**Q (Medium): When does `slotchange` fire and what does it tell you?**

Answer: `slotchange` fires on a `<slot>` element when the set of nodes assigned to it changes — nodes are added, removed, or replaced in the light DOM in ways that affect slot assignment. It fires asynchronously as a microtask after the DOM mutation. It does not fire during initial rendering when the element first connects. It also doesn't fire when the content of a slotted node changes (e.g., text content update within a slotted `<p>`) — only when the slot *assignment* changes (which nodes are assigned to it). To react to content changes, combine `slotchange` with `MutationObserver` on the assigned nodes.

**Q (Low): How would you use `assignedNodes({ flatten: true })` versus `assignedNodes()`?**

Answer: Without `flatten`, `assignedNodes()` returns only the nodes explicitly assigned from the light DOM. If the slot is empty (nothing projected), it returns an empty array — the fallback content inside the `<slot>` element itself is not included. With `{ flatten: true }`, if the slot is empty, the fallback content (the child nodes of the `<slot>` element) is included in the result instead. This is useful when component logic needs to know what's actually rendered — whether from projection or fallback — without caring which source it came from. For example, computing the text content for an aria-label needs to include fallback text if no content is projected.

---

## Self-Assessment

- [ ] Explain why template content is faster to reuse than re-parsing innerHTML
- [ ] Describe what happens to slotted content in terms of DOM ownership
- [ ] Write a shadow root template with a named slot for "header" and a default slot for content
- [ ] Explain when `slotchange` fires and when it does not
- [ ] Describe the difference between `assignedNodes()` and `assignedNodes({ flatten: true })`

---
*Next: Interoperability with Frameworks — how React, Vue, and Angular handle custom events and the property vs. attribute mismatch.*
