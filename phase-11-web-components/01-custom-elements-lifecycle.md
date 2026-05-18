# Custom Elements Lifecycle

## Quick Reference

| Callback | When it fires | Common use |
|---|---|---|
| `constructor()` | Element created (not yet in DOM) | Initialize state, attach shadow root |
| `connectedCallback()` | Element inserted into a document | Start timers, fetch data, attach event listeners |
| `disconnectedCallback()` | Element removed from a document | Cleanup timers, remove listeners |
| `attributeChangedCallback(name, oldVal, newVal)` | Observed attribute changed | Re-render, sync property to attribute |
| `adoptedCallback()` | Element moved to a different document | Rare — iframes, document.adoptNode |

---

## What Is This?

Custom Elements is the API that lets you define your own HTML elements — `<my-card>`, `<app-header>`, `<data-table>` — with their own behavior, lifecycle, and DOM representation. The browser treats them exactly like built-in elements: they appear in the DOM, respond to attributes, participate in forms, and fire events.

The lifecycle callbacks are how your element reacts to changes in its relationship with the document. Understanding exactly when each fires, and in what order, prevents the most common bugs with custom elements.

---

## Why It Exists

Before Custom Elements, reusable UI meant framework-specific components with no portability. A React component couldn't be dropped into a Vue app. Custom Elements are the platform's answer: component primitives that work everywhere, without a framework runtime.

> **Check yourself:** If connectedCallback fires every time an element is inserted into the DOM — not just the first time — why does that matter for cleanup?

---

## Defining a Custom Element

```js
class MyCard extends HTMLElement {
  static observedAttributes = ['title', 'expanded'];

  constructor() {
    super(); // always first
    // Safe here: set up instance variables, attach shadow DOM
    // NOT safe: read attributes (may not be set yet), access children (not parsed yet)
    this._data = null;
  }

  connectedCallback() {
    // Element is now in the document
    // Safe to: read attributes, access children (if upgrade timing is right), start work
    this.render();
    this._interval = setInterval(() => this.tick(), 1000);
  }

  disconnectedCallback() {
    // Element removed from document — clean up everything connectedCallback started
    clearInterval(this._interval);
  }

  attributeChangedCallback(name, oldValue, newValue) {
    if (oldValue === newValue) return; // no actual change
    if (name === 'title') this._updateTitle(newValue);
    if (name === 'expanded') this._updateExpanded(newValue !== null);
  }

  adoptedCallback() {
    // Moved to a new document (e.g., via document.adoptNode into an iframe's document)
    // Rarely needed — mostly relevant for drag-and-drop between browser windows
  }
}

customElements.define('my-card', MyCard);
```

---

## Lifecycle in Depth

### `constructor()`

Called when the element is created — either via `document.createElement('my-card')` or when the parser encounters `<my-card>` in HTML. The element exists but:

- It is **not connected** to the document yet
- Its **attributes are not set** yet (when parsed from HTML, attributes are set after construction)
- Its **children are not parsed** yet

The `super()` call must come first — it establishes the `this` binding as an HTMLElement. The constructor is the only place to safely attach a shadow root.

```js
constructor() {
  super();
  // This is the correct place for shadow root attachment
  this.attachShadow({ mode: 'open' });
  // NOT: this.getAttribute('foo') — may return null even if present in HTML
  // NOT: this.children — will be empty even if children exist in HTML
}
```

### `connectedCallback()`

Fires each time the element is inserted into a connected document — including moves. If you drag an element from one part of the DOM tree to another (`appendChild`), `disconnectedCallback` fires on removal and `connectedCallback` fires on reinsertion.

This is the right place to:
- Start timers, intervals, or animation loops
- Add event listeners to the document or window (not self — use the constructor or shadow root for that)
- Initiate data fetches
- Perform initial rendering if render depends on being in the document

```js
connectedCallback() {
  // Guard against double-calls if element is moved:
  // Work that should only happen once belongs in a first-connect flag
  if (!this._initialized) {
    this._initialized = true;
    this._firstConnect();
  }
  // Work that should happen every connect:
  this._addGlobalListeners();
}
```

### `disconnectedCallback()`

Fires each time the element is removed from a connected document. The element still exists — it's just not in the document. Always mirror `connectedCallback` cleanup here:

```js
connectedCallback() {
  this._onScroll = () => this._handleScroll();
  window.addEventListener('scroll', this._onScroll);
}

disconnectedCallback() {
  window.removeEventListener('scroll', this._onScroll);
}
```

Failing to clean up here is the primary source of memory leaks in custom elements. Closures in event listeners keep the element object alive even after it's removed.

### `attributeChangedCallback(name, oldValue, newValue)`

Only fires for attributes listed in the static `observedAttributes` array. This is a performance optimization — the browser doesn't have to notify every element for every attribute change on every element in the document.

```js
static observedAttributes = ['src', 'width', 'height'];

attributeChangedCallback(name, oldValue, newValue) {
  // oldValue is null on first set (element being upgraded or attribute added)
  // newValue is null when attribute is removed
  switch (name) {
    case 'src': this._loadImage(newValue); break;
    case 'width':
    case 'height': this._resize(); break;
  }
}
```

**Boolean attributes:** HTML boolean attributes (like `disabled`, `hidden`, `expanded`) work by presence/absence, not value. When present, `newValue` is an empty string `''`. When absent, `newValue` is `null`.

```js
// Setting: element.setAttribute('expanded', '') or element.expanded = true
// Removing: element.removeAttribute('expanded') or element.expanded = false
attributeChangedCallback(name, old, value) {
  if (name === 'expanded') {
    const isExpanded = value !== null; // presence = true
    this._updateExpanded(isExpanded);
  }
}
```

### Properties vs. Attributes

Properties and attributes are separate but should be kept in sync. The idiomatic pattern:

```js
// Property getter reads from attribute
get expanded() {
  return this.hasAttribute('expanded');
}

// Property setter writes to attribute
set expanded(value) {
  if (value) {
    this.setAttribute('expanded', '');
  } else {
    this.removeAttribute('expanded');
  }
}
// Now: element.expanded = true is equivalent to element.setAttribute('expanded', '')
// And attributeChangedCallback will fire either way
```

---

## Upgrade Timing

Elements can exist in the DOM before their definition is registered. The browser creates them as generic `HTMLElement` instances and **upgrades** them when `customElements.define()` is called.

```html
<!-- Element exists in DOM as HTMLElement, not MyCard -->
<my-card title="Hello"></my-card>

<script>
  // At this point, my-card is already in the DOM as HTMLElement
  customElements.define('my-card', MyCard);
  // Now the browser upgrades it: constructor() runs, then connectedCallback() runs
</script>
```

`customElements.whenDefined('my-card')` returns a Promise that resolves when the element is defined — useful for code that needs to interact with an element before its definition is available:

```js
await customElements.whenDefined('my-card');
const card = document.querySelector('my-card');
card.expanded = true; // safe — element is now upgraded
```

---

## Gotchas

**Constructor restrictions:** Don't read attributes or children in the constructor. They aren't set yet for declarative HTML — they're set after construction by the parser.

**connectedCallback fires on moves:** If you do work in connectedCallback that should only happen once, track it with an instance variable.

**`observedAttributes` must be static:** It's a static getter or static field — accessed before any instance is created. Making it an instance property means it's never read.

**Attribute values are always strings:** `this.getAttribute('count')` returns `"5"`, not `5`. Always parse.

**Missing `super()` crashes everything:** The custom elements spec requires `super()` as the first statement in a constructor that extends a built-in. The browser will throw if it's missing or deferred.

---

## Interview Questions

**Q (High): In what order do the custom element lifecycle callbacks fire when an element is parsed from HTML?**

Answer: `constructor()` fires first when the element node is created. At this point, attributes are not yet set and children are not yet parsed. Then the element is connected to the document, so `connectedCallback()` fires. If any observed attributes were present in the HTML, `attributeChangedCallback()` fires for each one — but only after the constructor has returned. In practice, observed attributes set in HTML fire their callbacks during the upgrade process after parsing, so by the time user script runs, all callbacks have fired. The reliable order is: constructor → attributeChangedCallback (for HTML-declared attributes) → connectedCallback.

**Q (High): Why must you clean up in `disconnectedCallback`, and what happens if you don't?**

Answer: `disconnectedCallback` fires when the element is removed from the document, but the element object itself still exists in memory — it's just detached. Any event listeners attached to `window`, `document`, or other long-lived objects via `connectedCallback` still hold a reference to the element through their closure. This reference prevents the garbage collector from reclaiming the element's memory even though nothing in the DOM tree points to it anymore. The element is a memory leak. Additionally, the callbacks keep executing (scroll listeners, timers) even though the element is not visible, burning CPU on detached elements. The pattern is strict: every `addEventListener` in `connectedCallback` must have a matching `removeEventListener` in `disconnectedCallback`.

**Q (Medium): What is the upgrade mechanism and why does it exist?**

Answer: When the HTML parser encounters an unknown element like `<my-card>`, it creates a plain `HTMLElement` with that tag name and inserts it into the DOM. At this point, `customElements.define('my-card', MyCard)` hasn't been called yet. The element sits in the DOM as a generic element. When `define()` is eventually called, the browser upgrades all existing matching elements: it runs the constructor on each, then fires `connectedCallback` (since they're already in the DOM). The upgrade mechanism exists because HTML is parsed before most JavaScript executes — elements need to exist in the DOM tree immediately for layout and rendering, even if their behavior isn't loaded yet. This is the same reason `<script defer>` is the standard: HTML first, behavior second.

**Q (Medium): When should you prefer property-based API over attribute-based API in a custom element?**

Answer: Attributes are always strings — they map to what's expressible in HTML markup. Properties are JavaScript values — they can be objects, arrays, functions, booleans, and numbers. Use attributes for: values that need to be set in HTML markup, simple scalar values that need to be reflected (width, src, type), and CSS attribute selectors (`[expanded]`). Use properties for: complex data (arrays, objects), non-serializable values (functions, DOM references), and values that don't need to appear in markup. The canonical approach is to expose both: a property getter/setter that reads/writes the attribute for simple values, and a property-only API for complex data. Never try to serialize an object into an attribute string.

**Q (Low): What does `adoptedCallback` fire for and when would you actually use it?**

Answer: `adoptedCallback` fires when an element is moved from one `Document` to another via `document.adoptNode()`. This happens when working with iframes — `document.adoptNode(iframe.contentDocument.querySelector('my-el'))` moves the element into the parent document. It also fires during certain drag-and-drop scenarios across browser windows. In practice, `adoptedCallback` is rarely implemented. Most custom elements don't need cross-document behavior. The common case that looks like it needs it (moving elements within the same document) uses `connectedCallback`/`disconnectedCallback` instead, since those fire on any DOM insertion/removal regardless of document.

---

## Self-Assessment

- [ ] State the order lifecycle callbacks fire when an element is parsed from HTML
- [ ] Explain why the constructor must not read attributes or children
- [ ] Write a connectedCallback/disconnectedCallback pair that manages a window scroll listener without leaking
- [ ] Explain what `observedAttributes` is and why it must be static
- [ ] Describe the upgrade mechanism and what `customElements.whenDefined()` solves

---
*Next: Shadow DOM — style encapsulation, open vs. closed mode, event retargeting, and how to pierce the shadow root.*
