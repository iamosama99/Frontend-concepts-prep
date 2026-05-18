# Interoperability with Frameworks

## Quick Reference

| | React (pre-19) | React 19+ | Vue 3 | Angular |
|---|---|---|---|---|
| Attribute binding | Yes | Yes | Yes | Yes |
| Property binding | No (string cast) | Yes | Yes (`:`prefix) | Yes (`[prop]`) |
| Custom events (`CustomEvent`) | No (no native handler) | Yes | Yes (`@event`) | Yes (`(event)`) |
| `ref` to element | Yes | Yes | Yes | Yes |

---

## What Is This?

Web Components (custom elements + shadow DOM) are framework-agnostic — they're part of the browser platform. But frameworks each have their own model for binding data to elements: some bind attributes, some bind properties, some handle DOM events, and custom events are not the same as native DOM events.

These differences create friction when embedding a custom element inside a React, Vue, or Angular component. The friction is well-understood and the solutions are consistent — knowing the pattern resolves every integration issue.

---

## Why It Matters

Custom elements are increasingly used at team/organization boundaries: a design system team ships web components consumed by app teams using different frameworks. Understanding the integration model prevents hours of debugging mysterious data binding failures and missing event handlers.

> **Check yourself:** Why does passing an object as an attribute to a custom element fail silently while passing it as a property works correctly?

---

## The Core Mismatch: Properties vs. Attributes

This is the root of most framework/web-component friction.

**Attributes** are always strings. They exist in HTML markup. Frameworks set them with `setAttribute()`.

**Properties** are JavaScript values on the DOM object. They can be objects, arrays, booleans, functions. They don't appear in HTML markup.

```html
<!-- Setting via attribute — always a string -->
<my-chart data="[1,2,3]"></my-chart>  <!-- data attribute = string "[1,2,3]" -->
```

```js
// Setting via property — any JavaScript value
document.querySelector('my-chart').data = [1, 2, 3]; // actual array
```

A well-designed custom element should handle both:

```js
// In the custom element:
set data(value) {
  this._data = value;
  this._render();
}
get data() { return this._data; }

attributeChangedCallback(name, old, value) {
  if (name === 'data') {
    try { this.data = JSON.parse(value); }
    catch { this.data = value; }
  }
}
```

But many web components only implement property-based APIs for complex data — which is correct — and frameworks must be told to use properties.

---

## React

### React pre-19 limitations

React before version 19 always serializes props to attributes (calls `setAttribute`). For a custom element `<my-chart data={chartData}>`, React calls `element.setAttribute('data', '[object Object]')` — the object is coerced to its string representation. The custom element receives `"[object Object]"`, not the actual object.

React pre-19 also does not handle `CustomEvent` — it synthesizes its own event system and only wires up standard DOM events (click, change, input, etc.). A custom element that dispatches `new CustomEvent('chart-click', { detail: { point: x } })` is invisible to React's `onChartClick` prop.

**Workaround with ref:**

```jsx
import { useRef, useEffect } from 'react';

function ChartWrapper({ data, onChartClick }) {
  const ref = useRef(null);

  useEffect(() => {
    const el = ref.current;
    if (!el) return;
    // Set property directly — bypasses React's attribute binding
    el.data = data;
  }, [data]);

  useEffect(() => {
    const el = ref.current;
    if (!el || !onChartClick) return;
    const handler = (e) => onChartClick(e.detail);
    // Manually add native event listener — bypasses React's event system
    el.addEventListener('chart-click', handler);
    return () => el.removeEventListener('chart-click', handler);
  }, [onChartClick]);

  return <my-chart ref={ref} />;
}
```

This ref-based wrapper pattern is the standard approach for React pre-19 + web components.

### React 19

React 19 fully supports web components:

```jsx
// React 19 — properties and custom events work natively
function Chart({ data, onChartClick }) {
  return (
    <my-chart
      data={data}             // React 19 sets as property if not primitive
      onchartclick={e => onChartClick(e.detail)}  // lowercase event name
    />
  );
}
```

React 19 automatically sets non-primitive values as properties rather than attributes, and handles `CustomEvent` dispatched from custom elements. Event names in JSX are lowercase (the DOM event name), not camelCase.

---

## Vue 3

Vue 3 has first-class web component support. Vue binds:
- Static values → attributes (`setAttribute`)
- `:prop="value"` syntax → properties (dot notation on DOM object)
- `@event="handler"` → native DOM event listener (`addEventListener`)

```html
<!-- Vue 3 template -->
<template>
  <my-chart
    :data="chartData"            <!-- binds as property: el.data = chartData -->
    title="Sales Chart"          <!-- attribute string binding -->
    @chart-click="handleClick"   <!-- addEventListener('chart-click', ...) -->
  />
</template>

<script setup>
import { ref } from 'vue';
const chartData = ref([1, 2, 3]);
const handleClick = (e) => console.log(e.detail);
</script>
```

Vue also handles the `.prop` modifier explicitly for cases where Vue's heuristic doesn't detect property vs. attribute:

```html
<my-element .someComplexProp="objectValue" />  <!-- force property binding -->
```

Vue 3 recognizes custom elements (those with hyphens in the name) and skips its own component resolution for them.

---

## Angular

Angular uses brackets `[prop]="value"` for property binding and parentheses `(event)="handler($event)"` for event binding. Both work naturally with custom elements:

```html
<!-- Angular template -->
<my-chart
  [data]="chartData"              <!-- property binding: el.data = chartData -->
  title="Sales Chart"             <!-- attribute binding: setAttribute('title', ...) -->
  (chart-click)="handleClick($event)"  <!-- event listener: addEventListener -->
></my-chart>
```

Angular requires the `CUSTOM_ELEMENTS_SCHEMA` to allow unknown element names, or the element must be registered in an Angular module:

```ts
import { NgModule, CUSTOM_ELEMENTS_SCHEMA } from '@angular/core';

@NgModule({
  schemas: [CUSTOM_ELEMENTS_SCHEMA]
})
export class AppModule {}
```

In standalone components (Angular 17+):

```ts
@Component({
  selector: 'app-chart',
  template: `<my-chart [data]="chartData" (chart-click)="handle($event)"></my-chart>`,
  schemas: [CUSTOM_ELEMENTS_SCHEMA]
})
```

---

## Custom Events: The Full Pattern

For maximum framework compatibility, dispatch custom events correctly:

```js
// Inside the custom element — dispatch at the element level (not shadow root)
this.dispatchEvent(new CustomEvent('chart-click', {
  bubbles: true,    // bubble up through DOM
  composed: true,   // cross shadow boundary
  detail: { point: { x: 10, y: 20 } }
}));
```

- `bubbles: true` — required for parent component listeners to catch it
- `composed: true` — required to cross the shadow DOM boundary into the light DOM where the framework listener is attached
- `detail` — the payload, accessible as `event.detail`

Framework handler patterns:

```jsx
// React 19
<my-chart onchart-click={(e) => console.log(e.detail)} />

// Vue 3
<my-chart @chart-click="(e) => console.log(e.detail)" />

// Angular
<my-chart (chart-click)="onChartClick($event)"> // $event is the CustomEvent
```

---

## Framework Wrappers (the Production Pattern)

For design systems distributing web components to React consumers, generating thin wrapper components eliminates the ref boilerplate:

```js
// Auto-generated React wrapper (by tools like @lit-labs/react):
import { createComponent } from '@lit-labs/react';
import React from 'react';
import { MyChart } from './my-chart.js'; // the custom element class

export const MyChartReact = createComponent({
  tagName: 'my-chart',
  elementClass: MyChart,
  react: React,
  events: {
    onChartClick: 'chart-click', // React prop name → DOM event name
  },
});

// Usage in React (pre-19 or 19+) — works like a normal React component
<MyChartReact data={chartData} onChartClick={(e) => console.log(e.detail)} />
```

Tools that generate these wrappers: `@lit-labs/react`, `@microsoft/fast-react-wrapper`, custom codegen from the custom elements manifest.

---

## Gotchas

**React's synthetic event system doesn't catch CustomEvents in pre-19.** No `onFoo` prop maps to a custom event. Must use `ref` + `addEventListener`.

**All attribute values are strings.** Even `<my-el count={42}>` becomes `setAttribute('count', '42')` in most frameworks (except Vue's `:count` and Angular's `[count]`). Parse in `attributeChangedCallback`.

**`composed: true` is required for custom events to escape Shadow DOM.** Without it, the event stops at the shadow root and framework listeners in light DOM never see it.

**Property setting after upgrade only.** If a framework sets a property before the custom element is defined (upgraded), the property write lands on the plain HTMLElement object and the custom element's setter is never called. Fix with `customElements.whenDefined` or ensure element definition loads before the component renders.

**TypeScript types for custom elements in JSX.** Without type declarations, TypeScript errors on `<my-chart data={...}>`. Fix by extending JSX's intrinsic elements:

```ts
// declarations.d.ts
declare namespace JSX {
  interface IntrinsicElements {
    'my-chart': React.DetailedHTMLProps<React.HTMLAttributes<HTMLElement>, HTMLElement> & {
      data?: number[];
    };
  }
}
```

---

## Interview Questions

**Q (High): Why does passing an object prop to a custom element in React break, and how do you fix it?**

Answer: React (pre-19) serializes all non-string prop values to attributes using `setAttribute`. When you write `<my-chart data={myArray} />`, React calls `element.setAttribute('data', '[object Object]')` — the array is coerced to its string representation before being set. The custom element's `attributeChangedCallback` receives the string `"[object Object]"`, not the array. The fix is to use a `ref` and set the property directly: `elementRef.current.data = myArray`. This bypasses React's attribute serialization and hits the custom element's property setter, which can receive any JavaScript value. React 19 addresses this at the framework level by detecting non-primitive values and setting them as properties automatically.

**Q (High): What is required for a custom event dispatched inside a web component's shadow DOM to be caught by a parent React/Vue/Angular component?**

Answer: Three things: First, the event must be dispatched on `this` (the host element), not on an element inside the shadow root — or if dispatched inside the shadow tree, it must have `composed: true`. Second, `bubbles: true` is required for the event to bubble up through the DOM to where the framework has attached its listener. Third, for events from inside the shadow DOM, `composed: true` is required to cross the shadow boundary into the light DOM. Without both flags, the event either stops at the shadow root or never reaches the parent component's listener. The dispatch pattern is: `this.dispatchEvent(new CustomEvent('event-name', { bubbles: true, composed: true, detail: payload }))`.

**Q (Medium): How does Vue 3's property binding differ from Angular's and what makes both better than React pre-19 for web components?**

Answer: Vue 3 uses `:prop="value"` (colon prefix) to signal property binding — Vue sets `element.prop = value` directly, bypassing attribute serialization. Angular uses `[prop]="value"` (bracket syntax) for the same purpose. Both frameworks distinguish property vs. attribute binding in their template syntax, giving developers explicit control. React pre-19 has no such distinction — it always serializes to attributes (with a small built-in allowlist of known DOM properties). This is why React pre-19 is the worst of the three for web component integration. React 19 fixes this by detecting non-primitive values and property-binding automatically, bringing it to parity with Vue and Angular without requiring a syntax change.

**Q (Low): What is a framework wrapper and when does a design system team need to generate one?**

Answer: A framework wrapper is a thin component (e.g., a React component) that renders a custom element via `ref` and manually wires up property bindings and custom event listeners — the boilerplate a consumer would otherwise write themselves. Design system teams generate these when distributing web components to React (pre-19) consumers, because React's web component support is incomplete without them. Tools like `@lit-labs/react` automate wrapper generation from custom element metadata (the custom elements manifest). The wrapper gives React consumers an idiomatic experience: import a React component, use standard React prop/callback conventions, and get TypeScript types — with no knowledge of the web component's native API required.

---

## Self-Assessment

- [ ] Explain why React pre-19 can't pass objects to custom elements via props
- [ ] Write a React ref-based wrapper that sets a property and listens for a custom event
- [ ] Explain what `bubbles: true` and `composed: true` are required for in custom event dispatch
- [ ] Describe the Vue 3 syntax for property binding vs. attribute binding vs. event binding
- [ ] Explain what a framework wrapper generates and why design system teams produce them

---
*Next: Phase 12 — Observer & Intersection APIs — IntersectionObserver, ResizeObserver, MutationObserver, PerformanceObserver, and when to choose observers over event listeners.*
