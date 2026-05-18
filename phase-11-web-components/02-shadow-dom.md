# Shadow DOM: Encapsulation, Open vs. Closed Mode

## Quick Reference

| Concept | Detail |
|---|---|
| Attach shadow root | `element.attachShadow({ mode: 'open' \| 'closed' })` |
| Style scope | CSS inside shadow root does not leak out; outside CSS does not leak in (with exceptions) |
| `open` mode | `element.shadowRoot` returns the root; JS can pierce it |
| `closed` mode | `element.shadowRoot` returns `null`; reference must be kept privately |
| CSS custom properties | Inherited, so they cross shadow boundaries — the designed theming vector |
| Event retargeting | Events from inside shadow DOM appear to originate from the host element |

---

## What Is This?

Shadow DOM is a browser mechanism that attaches a separate, encapsulated DOM tree to an element — the *shadow tree* — alongside its regular children — the *light tree*. CSS and JavaScript in the shadow tree are scoped to it. The host document's styles don't apply inside, and styles inside don't apply outside.

This is the CSS encapsulation problem that frontend frameworks have spent years solving with conventions (BEM), runtime transforms (CSS Modules), or JS-in-CSS (styled-components). Shadow DOM is the platform's native solution: encapsulation enforced by the browser itself, with no build step and no naming conventions required.

---

## Why It Matters

Large design systems and custom elements need reliable style isolation. Without it, a consumer's global `button { color: red }` rule breaks your component's internal button. Shadow DOM makes isolation unconditional — it works regardless of what CSS the host page contains.

> **Check yourself:** If Shadow DOM blocks outside styles, how do design tokens and theming work across the boundary?

---

## Attaching a Shadow Root

```js
class MyButton extends HTMLElement {
  constructor() {
    super();
    // attachShadow must be called in the constructor
    const shadow = this.attachShadow({ mode: 'open' });

    shadow.innerHTML = `
      <style>
        button {
          background: var(--btn-bg, #0070f3);
          color: var(--btn-color, white);
          border: none;
          padding: 8px 16px;
          border-radius: 4px;
          cursor: pointer;
        }
      </style>
      <button><slot></slot></button>
    `;
  }
}
customElements.define('my-button', MyButton);
```

Usage in HTML:

```html
<!-- Light DOM content — "Hello" — is projected into the <slot> -->
<my-button>Hello</my-button>
```

---

## Style Encapsulation: How It Works

### Styles in → out (isolation)

Styles defined inside a shadow root apply only to elements in that shadow tree:

```html
<!-- shadow root internals: -->
<style>
  p { color: red; } /* only affects <p> inside this shadow tree */
</style>
<p>I am red</p>
```

The `<p>` elements in the host document are completely unaffected.

### Styles out → in (blocking)

Styles in the host document do not cascade into shadow trees. This is unconditional:

```css
/* host document stylesheet */
p { font-size: 24px; } /* has NO effect on <p> inside any shadow tree */
```

### Exceptions — what does cross the boundary

**CSS custom properties (variables)** are inherited. Inheritance crosses shadow boundaries. This is intentional — it's the designed theming API:

```css
/* host document */
:root {
  --btn-bg: hotpink;
  --btn-color: white;
}
```

```css
/* inside shadow root */
button {
  background: var(--btn-bg, #0070f3); /* inherits hotpink from host */
}
```

**Inherited CSS properties** (color, font-family, font-size, line-height, etc.) are also inherited through shadow boundaries. The shadow tree doesn't block inheritance — it blocks the cascade. If a host element has `color: blue`, children inside its shadow tree will inherit that color unless overridden inside the shadow root.

**`::part()` pseudo-element** allows outside CSS to style specific parts of a shadow tree that the component explicitly opts in to:

```js
// Inside shadow root:
shadow.innerHTML = `<button part="base">Click</button>`;
```

```css
/* Host document — targets the exposed "base" part */
my-button::part(base) {
  border-radius: 0;
  font-weight: bold;
}
```

---

## Open vs. Closed Mode

The `mode` option controls external JavaScript access to the shadow root.

### `open` mode

```js
const shadow = this.attachShadow({ mode: 'open' });
```

`element.shadowRoot` returns the shadow root. Any script on the page can reach inside:

```js
const btn = document.querySelector('my-button');
const inner = btn.shadowRoot.querySelector('button'); // works
inner.style.background = 'red'; // breaks encapsulation from outside
```

`open` is the default for all native browser elements like `<input>`, `<video>`, `<details>`. DevTools needs this access to show you the shadow tree.

### `closed` mode

```js
const shadow = this.attachShadow({ mode: 'closed' });
// shadow is the ShadowRoot — save it to a private variable
```

`element.shadowRoot` returns `null`. External code cannot access the shadow tree through the standard API:

```js
const btn = document.querySelector('my-button');
btn.shadowRoot; // null — access denied
```

The reference must be kept privately:

```js
const shadowRoots = new WeakMap();

class MyButton extends HTMLElement {
  constructor() {
    super();
    const shadow = this.attachShadow({ mode: 'closed' });
    shadowRoots.set(this, shadow); // keep reference alive but inaccessible to outsiders
  }

  someInternalMethod() {
    const shadow = shadowRoots.get(this);
    shadow.querySelector('button').disabled = true;
  }
}
```

### The security debate

`closed` mode is not a security boundary. Determined attackers can override `attachShadow` itself, intercept the return value, or use other techniques. The spec acknowledges this. `closed` is better described as **encapsulation for accidental misuse** — it prevents consumers from depending on internal DOM structure, which prevents breaking changes from being visible. It communicates intent: "this is private, don't touch it."

Most practical design system components use `open` mode because it allows DevTools inspection and doesn't require the WeakMap dance. `closed` is appropriate for security-sensitive contexts like browser-native password inputs.

---

## Event Retargeting

Events dispatched from inside a shadow tree are *retargeted* when they cross the shadow boundary. An external listener sees the event as if it came from the host element:

```js
class MyComponent extends HTMLElement {
  constructor() {
    super();
    const shadow = this.attachShadow({ mode: 'open' });
    shadow.innerHTML = `<button>Click me</button>`;
    shadow.querySelector('button').addEventListener('click', (e) => {
      console.log('Inside shadow:', e.target); // <button>
    });
  }
}

document.querySelector('my-component').addEventListener('click', (e) => {
  console.log('Outside shadow:', e.target); // <my-component> — retargeted
});
```

Retargeting preserves encapsulation: external code doesn't see internal DOM structure through event targets. The `composedPath()` method returns the full path including shadow tree elements, but only if the shadow root is `open`:

```js
document.querySelector('my-component').addEventListener('click', (e) => {
  console.log(e.composedPath()); // [button, shadow-root, my-component, body, html, document, window]
});
```

### Event composition

Custom events dispatched inside a shadow root do not cross the boundary by default. To make them bubble out, set `composed: true`:

```js
// Inside shadow root — this event stays inside:
this.dispatchEvent(new CustomEvent('internal-click'));

// This crosses the shadow boundary:
this.dispatchEvent(new CustomEvent('my-click', {
  bubbles: true,
  composed: true,
  detail: { value: 42 }
}));
```

---

## The `:host` and `:host-context` Selectors

Inside a shadow root, `:host` refers to the custom element itself:

```css
/* inside shadow root */
:host {
  display: block; /* custom elements default to inline — usually needs this */
  contain: content;
}

:host([disabled]) {
  opacity: 0.5;
  pointer-events: none;
}

:host(:hover) button {
  background: darken(var(--btn-bg), 10%);
}
```

`:host-context(selector)` styles the host based on its ancestors — useful for dark mode or layout-aware theming:

```css
:host-context([data-theme="dark"]) {
  --btn-bg: #444;
}
```

---

## Gotchas

**Custom elements default to `display: inline`.** Always add `:host { display: block }` unless you explicitly want inline behavior.

**Slotted content lives in the light DOM.** Styles inside the shadow root don't apply to slotted content. Use `::slotted()` to style it, with the limitation that you can only select direct children of the slot:

```css
::slotted(span) { color: red; } /* works for direct <span> children of slot */
::slotted(span em) { color: blue; } /* does NOT work — only direct children */
```

**`attachShadow` in the constructor only.** Calling it after the constructor (e.g., in `connectedCallback`) throws in some environments and creates ordering bugs in others.

**Forms and shadow DOM.** `<input>` elements inside a shadow root are not included in a surrounding `<form>`'s `elements` collection unless you implement the form-associated custom elements API (`static formAssociated = true`, `attachInternals()`).

---

## Interview Questions

**Q (High): How does Shadow DOM achieve CSS encapsulation, and what are its limits?**

Answer: Shadow DOM attaches a separate DOM subtree to an element. The browser enforces a style boundary: CSS in the shadow tree only matches elements in that shadow tree, and the cascading stylesheet rules from the host document do not pierce into it. This means a global `p { color: red }` rule on the page has no effect on `<p>` elements inside a shadow root. However, two things do cross: CSS inheritance (inherited properties like `color`, `font-family` flow from host to shadow) and CSS custom properties (variables are inherited, making them the intentional theming API). The `::part()` pseudo-element also allows explicit opt-in to external styling for designated parts of the component. So encapsulation is strong but not total — which is by design, since total isolation would prevent any theming.

**Q (High): What is event retargeting and why does Shadow DOM do it?**

Answer: When an event fires on an element inside a shadow tree and bubbles out past the shadow root, the browser replaces the event's `target` property with the host element (the custom element) when the event is observed by outside listeners. This is called retargeting. A click on an internal `<button>` appears as a click on `<my-component>` to any listener attached to the host or its ancestors. The reason is encapsulation: if external code could see the actual target, it would be observing and depending on the internal DOM structure. Any refactor of the component's internals would become a breaking change. Retargeting hides the implementation detail. Internal shadow listeners still see the real target. The full path is accessible via `composedPath()`, but only on `open` shadow roots.

**Q (Medium): When would you choose `closed` mode over `open` mode, and what are its real-world limitations?**

Answer: `closed` mode is chosen when you want to communicate that the internal DOM is private and not part of the component's public API — preventing consumers from building on internal structure. It's also used in high-security contexts (password managers, browser-internal UI) where avoiding accidental external access matters. The limitation is that `closed` is not a real security boundary: a malicious script can override `HTMLElement.prototype.attachShadow`, intercept the return value, and capture the shadow root before it reaches your component. It also breaks DevTools (no shadow tree inspection) and third-party integrations (accessibility tools, test libraries). Most design system authors use `open` and rely on documentation and TypeScript types to communicate the public API boundary.

**Q (Low): How do you style content that is slotted into a shadow root?**

Answer: Slotted content (the light DOM children projected into `<slot>` elements) remains in the light DOM — the shadow root's CSS doesn't apply to it. To style slotted content from inside the shadow root, use the `::slotted()` pseudo-element, but with an important constraint: `::slotted()` can only match direct children of the slot, not deeper descendants. `::slotted(p)` styles `<p>` elements that are direct children of the slot, but `::slotted(p span)` is invalid and ignored. For deeper styling of slotted content, either use CSS custom properties that the slotted content can consume, or accept that slotted content styling must come from the light DOM (outside the shadow root), which usually gives the parent/consumer control over its own content's appearance.

---

## Self-Assessment

- [ ] Explain what CSS rules cross the shadow boundary and why
- [ ] Describe the difference between `open` and `closed` mode and when to use each
- [ ] Explain event retargeting and how to get the real internal target
- [ ] Write CSS inside a shadow root that styles the host element when it has a `[disabled]` attribute
- [ ] Explain why `::slotted()` can only select direct children of a slot

---
*Next: HTML Templates & Slots — the `<template>` element, named slots, and the slotchange event.*
