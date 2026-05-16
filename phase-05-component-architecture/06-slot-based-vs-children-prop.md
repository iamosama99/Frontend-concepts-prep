# Slot-based Composition vs. Children Prop Patterns

## Quick Reference

| Pattern | Mechanism | Best for |
|---------|-----------|----------|
| `children` prop | Single content injection point — any JSX | Wrapping a single piece of unknown content |
| Named props as slots | `header={<H2>Title</H2>}`, `footer={<Button>OK</Button>}` | Multiple distinct injection points on a component |
| Children as function | `{children(state)}` — render prop via children | Passing dynamic state to consumer-controlled rendering |
| `React.Children` + `cloneElement` | Iterate and clone child elements to inject props | Compound component index injection (narrow legitimate use) |
| `ReactDOM.createPortal` | Render children into a different DOM node | Modals, tooltips, dropdowns escaping stacking context |

---

## What Is This?

When you write `<Button>Click me</Button>` in React, "Click me" arrives inside `Button` as `props.children`. This is React's primary composition primitive: the ability for a component to render content it did not define, supplied by whoever uses it.

This single mechanism — `children` — is the foundation of layout components, wrapper components, and the entire concept of "composition over configuration." But the basic `children` prop is just one injection point. Real applications need more: a `Modal` needs separate header, body, and footer content; a `Layout` needs a sidebar and a main area. This file covers every pattern React engineers use to create multiple injection points, and the distinct mechanism of Portals for rendering outside the component's DOM position entirely.

```js
// The simplest form: children as a single slot
function Card({ children }) {
  return <div className="card">{children}</div>;
}

<Card>
  <h2>Title</h2>
  <p>Any content here — Card doesn't need to know what it is.</p>
</Card>
```

> **Check yourself:** If a component receives `children`, can it render those children in any position in its own JSX? What constraint does `children` impose on the parent component?

---

## Why Does It Exist?

### The Configuration vs. Composition Divide

A component that doesn't accept children is a *configuration-driven* component: `<Button variant="primary" label="Submit" />`. Everything it renders is predetermined. The consumer can only adjust what the component anticipated.

A component that accepts `children` is a *composition-driven* component: `<Button variant="primary">Submit</Button>`. The consumer controls what goes inside. The component defines structure and behavior; the consumer provides content. This division mirrors the web's foundational model — HTML elements like `<div>` and `<section>` are composition primitives that wrap arbitrary content.

React's `children` prop was designed from day one to enable this pattern, because React UI is assembled by composition. Components that don't accept children are leaf nodes; components that do are structural.

### Why Named Slots Were Needed

The single `children` slot works for one injection point. When components need multiple independent regions — sidebar vs. main, header vs. body vs. footer — a single `children` prop can't distinguish them. Web Components solved this with the native `<slot>` element and named slots. React has no native slot mechanism, so the community converged on the named prop pattern as the idiomatic equivalent.

---

## How It Works

### `children` — The Single Injection Point

`children` is not a special React feature. It is a conventional prop name that JSX populates automatically for content between opening and closing tags:

```js
// These two are identical after JSX compilation
<Wrapper>Hello</Wrapper>
React.createElement(Wrapper, null, 'Hello');

// The children prop receives whatever is between the tags
function Wrapper({ children }) {
  return <div className="wrapper">{children}</div>;
}
```

`children` can be anything: a string, a number, a single React element, an array of elements, `null`, or a function. This flexibility is both powerful and treacherous (see Gotchas).

Multiple children arrive as an array:

```js
<List>
  <Item>One</Item>
  <Item>Two</Item>
  <Item>Three</Item>
</List>
// List receives: children = [<Item>One</Item>, <Item>Two</Item>, <Item>Three</Item>]

function List({ children }) {
  // children is an array here — can be rendered directly
  return <ul>{children}</ul>;
}
```

But a single child is not an array — it's the element itself. This inconsistency is why `React.Children.map` exists (it normalizes both cases).

---

### Named Props as Slots — Multiple Injection Points

The React idiom for named slots is simply: use named props that accept JSX.

```js
function Modal({ header, children, footer }) {
  return (
    <div className="modal">
      <div className="modal-header">{header}</div>
      <div className="modal-body">{children}</div>
      <div className="modal-footer">{footer}</div>
    </div>
  );
}

// Usage — consumer controls the content of each slot independently
<Modal
  header={<h2>Confirm Delete</h2>}
  footer={
    <>
      <Button variant="ghost" onClick={onCancel}>Cancel</Button>
      <Button variant="danger" onClick={onConfirm}>Delete</Button>
    </>
  }
>
  <p>Are you sure you want to delete this item? This cannot be undone.</p>
</Modal>
```

This is arguably cleaner than named slots in Web Components because the slot assignment is explicit at the call site — you can see exactly what goes where. There is no ambiguity about which child ends up in the header vs. the footer.

The naming convention matters: `children` is the conventional name for the "main" or "default" slot. All other slots get descriptive names: `header`, `footer`, `sidebar`, `actions`, `icon`, `label`. Libraries like MUI, Chakra UI, and Headless UI use this pattern extensively.

**When to prefer named props over a single `children`:**
- Multiple, semantically distinct regions
- The component needs to apply different layout or styling to different regions
- The consumer should not need to know the internal structure to place content correctly

**When to stick with `children`:**
- A single content area with no structural distinction
- A wrapper component where the consumer controls the entire inside

---

### Web Components `<slot>` vs. React `children` — The Conceptual Difference

Web Components have a native browser mechanism: `<slot>`. Content written between a custom element's tags is "projected" into the `<slot>` in the Shadow DOM:

```html
<!-- Web Component definition (Shadow DOM) -->
<template>
  <header>
    <slot name="header"></slot>
  </header>
  <main>
    <slot></slot>  <!-- default slot -->
  </main>
</template>

<!-- Consumer usage -->
<my-modal>
  <h2 slot="header">Title</h2>  <!-- projected into named slot -->
  <p>Body content</p>           <!-- projected into default slot -->
</my-modal>
```

The browser performs real DOM projection: the `<h2>` physically remains in the light DOM but is visually rendered at the `<slot>` position inside the Shadow DOM. Event delegation and CSS inheritance are affected by this split — light DOM children inherit styles from the document, not the shadow root.

React's `children` is fundamentally different: it is a function argument. There is no projection, no Shadow DOM, no split between light and shadow. When React renders `<Card><p>Hello</p></Card>`, the `<p>` is rendered wherever `{children}` appears in Card's JSX. It's a value being passed to a function — simple and explicit. There is no browser-level slot resolution.

The practical implication: React's model is easier to reason about (it's just a prop), but it doesn't provide Shadow DOM style encapsulation. Web Components' `<slot>` provides real DOM encapsulation but adds the complexity of the light/shadow split. They're different tools for different problems.

---

### `React.Children` Utilities

`React.Children` is a set of utilities for working with the `children` prop when you need to treat it as a collection rather than opaque content.

**`React.Children.map`** — normalizes children into an array regardless of whether there's one child or many, and maps over them:

```js
function RadioGroup({ children }) {
  const [selected, setSelected] = useState(null);

  return (
    <div role="radiogroup">
      {React.Children.map(children, (child, index) => {
        // Inject the shared group state into each Radio child
        return React.cloneElement(child, {
          checked: child.props.value === selected,
          onChange: () => setSelected(child.props.value),
        });
      })}
    </div>
  );
}
```

**`React.Children.only`** — throws if `children` is not exactly one element. Used to enforce the constraint that a component accepts a single child:

```js
function Tooltip({ children, text }) {
  // Throw if caller tries to pass multiple children — clearer error than a cryptic render failure
  const child = React.Children.only(children);
  return (
    <span title={text}>
      {child}
    </span>
  );
}
```

**`React.Children.count`** — returns the count of children, handling null/undefined/arrays correctly:

```js
function Grid({ children, columns = 3 }) {
  const count = React.Children.count(children);
  // Use count to compute layout
}
```

**Why `React.Children.map` instead of `children.map`:** if `children` is a single element (not an array), `children.map` throws — arrays have `.map`, React elements do not. `React.Children.map` normalizes this, always treating children as a collection.

---

### `cloneElement` — The Sharp Edge

`React.cloneElement(element, additionalProps)` creates a copy of an existing React element with merged props. This is how `React.Children.map` is combined with injection:

```js
// Compound component injecting index into Tab children
React.Children.map(children, (child, index) =>
  React.cloneElement(child, { index })
);
```

This is often a sign of poor API design because it breaks encapsulation: the parent must know the internal prop contract of its children. If `<Tab>` changes its prop name from `index` to `tabIndex`, the `cloneElement` call in `TabList` silently breaks.

The two legitimate uses:
1. **Injecting positional/structural metadata** that children cannot compute themselves (the index in a list, or `isFirst`/`isLast`)
2. **Merging event handlers** in design systems (e.g., a `FocusRing` wrapper merging its own `onFocus` with the child's `onFocus`)

For everything else — passing theme data, access to shared state, callbacks — use Context. It's explicit, maintainable, and doesn't require iterating children.

> **Check yourself:** You have a `<ButtonGroup>` component. It needs to pass a `size` prop to every `<Button>` child. Should you use `cloneElement` or Context? What breaks first if you use `cloneElement`?

---

### Children as Function — Render Props via `children`

The render prop pattern (covered in File 5) can be expressed through `children` instead of a named prop:

```js
function Toggle({ children }) {
  const [on, setOn] = useState(false);
  return children({ on, toggle: () => setOn(prev => !prev) });
}

// Usage
<Toggle>
  {({ on, toggle }) => (
    <div>
      <button onClick={toggle}>{on ? 'Hide' : 'Show'}</button>
      {on && <p>Now you see me</p>}
    </div>
  )}
</Toggle>
```

This is identical in capability to `render={...}` render props. The "children as function" form is preferred by some library authors because it reads more naturally in JSX — the function is in the "body" position. Downside: `React.Children.only` and `React.Children.count` don't work as expected when children is a function, not an element.

Use this pattern when the component needs to hand dynamic state to consumer-controlled rendering, and when a custom hook cannot replace it (e.g., the state is tied to a component's lifecycle in a way that requires the component wrapper).

---

### Component Slots via Prop-based API — When to Prefer Over `children`

The named-prop slot pattern — `<Modal header={...} footer={...}>body</Modal>` — has a structural advantage over trying to use `children` for everything with internal type-detection:

```js
// Anti-pattern: using children for everything and type-detecting
function Modal({ children }) {
  let header, body, footer;
  React.Children.forEach(children, child => {
    if (child.type === ModalHeader) header = child;
    else if (child.type === ModalFooter) footer = child;
    else body = child;
  });
  return (
    <div>
      <div className="header">{header}</div>
      <div className="body">{body}</div>
      <div className="footer">{footer}</div>
    </div>
  );
}

// Usage
<Modal>
  <ModalHeader><h2>Title</h2></ModalHeader>
  <p>Body</p>
  <ModalFooter><Button>OK</Button></ModalFooter>
</Modal>
// Order matters implicitly. consumer must use ModalHeader/ModalFooter wrappers.
// Type detection via child.type is brittle — breaks with HOC-wrapped children.
```

Compare to the explicit prop-based API:

```js
// Clear, explicit, no type detection
<Modal
  header={<h2>Title</h2>}
  footer={<Button>OK</Button>}
>
  <p>Body</p>
</Modal>
```

The prop-based API is explicit about intent, immune to ordering issues, and doesn't require consumers to import and use `ModalHeader`/`ModalFooter` wrapper components. The trade-off: `children` allows the consumer to compose freely and even add structure between sections; the prop-based API fixes the structure.

The default slot anti-pattern is `<Layout><Sidebar /><Main /></Layout>` relying on ordering: the first child is the sidebar, the second is the main area. This is implicit, fragile, and invisible to tooling. The named-prop version `<Layout sidebar={<Sidebar />} main={<Main />}>` is always preferable.

---

### Portals — Rendering Outside the Component Tree

`ReactDOM.createPortal(children, domNode)` renders `children` into a different DOM node than the component's natural DOM position:

```js
import { createPortal } from 'react-dom';

function Modal({ isOpen, onClose, children }) {
  if (!isOpen) return null;

  // Render into document.body instead of wherever Modal sits in the DOM
  return createPortal(
    <div className="modal-overlay" onClick={onClose}>
      <div className="modal-content" onClick={e => e.stopPropagation()}>
        {children}
      </div>
    </div>,
    document.body
  );
}
```

**Why Portals exist — the stacking context problem:**

When a modal or tooltip is nested deep in the DOM, it inherits its ancestor's `overflow: hidden`, `z-index`, and `transform` stacking contexts. A `z-index: 9999` on the modal does nothing if its parent has `overflow: hidden` or creates a new stacking context with a lower z-index. The only reliable fix is to render the modal at the top of the DOM tree — `document.body` — where it escapes all ancestor stacking contexts.

```
DOM tree without portal:         DOM tree with portal:
<div style="overflow:hidden">    <div style="overflow:hidden">
  <Modal>                          <!-- Modal is gone from here -->
    <div z-index="9999">         </div>
      <!-- clipped! -->          <body>
    </div>                         <div class="modal-overlay">
  </Modal>                           <div class="modal-content">
</div>                                 <!-- renders correctly -->
                                   </div>
                                 </div>
                               </body>
```

**The event bubbling nuance — the most important Portal gotcha:**

Even though the Portal's DOM output is physically at `document.body`, React events still bubble through the *React component tree*, not the DOM tree. This is intentional and critical:

```js
function App() {
  const handleClick = (e) => {
    console.log('App received click');
    // This WILL fire when you click inside the Portal's modal
    // even though the modal is at document.body in the DOM
  };

  return (
    <div onClick={handleClick}>
      <Modal isOpen={true}>
        <button>Click me</button>
      </Modal>
    </div>
  );
}
```

Clicking "Click me" in the modal:
- DOM event bubbles: `button → .modal-content → .modal-overlay → body → html → document`
- React synthetic event bubbles: `button → Modal → div → App`

The React event reaches `App`'s `onClick` handler. This is because React's event system is attached to the React root, and it simulates bubbling through the React component hierarchy regardless of DOM position. This means click-outside detection patterns work correctly: the modal component is inside the React tree of the component that opened it, so clicks inside the modal do bubble to the opener's click handler — which is what you typically want (and need to `stopPropagation` on if you don't).

> **Check yourself:** A modal is inside a component with `onClick={closeModal}`. The modal is rendered via Portal. When the user clicks inside the modal, does `closeModal` fire? Why or why not?

---

## Gotchas

**1. `children` can be undefined.** If a component is used without children — `<Card />` instead of `<Card>content</Card>` — `children` is `undefined`. Rendering `{children}` with `undefined` is safe (renders nothing). But `React.Children.map(undefined, ...)` returns `undefined` too, not an empty array. Guard: `React.Children.count(children) > 0` before iterating.

**2. A single child is not an array.** `<List><Item /></List>` — `children` is an element, not `[element]`. `children.map` throws. `React.Children.map` handles both cases. Always use the utility when you need to iterate.

**3. `cloneElement` breaks with HOC-wrapped children.** If the consumer wraps `<Tab>` in a HOC: `<TabList><withTooltip(Tab) /></TabList>`, the `child.type` is the HOC function, not `Tab`. Any `if (child.type === Tab)` check fails silently. Context-based communication is immune to this — it doesn't care about the component's type.

**4. `cloneElement` breaks with Fragment children.** `React.Children.map` counts a Fragment as one child. If the consumer puts `<Tabs.Tab>` items inside a `<>...</>`, the cloneElement-based index injection receives the Fragment as one child, not its contents. Libraries that use Context-based registration avoid this entirely.

**5. Portal event bubbling surprises.** Developers expect that because the portal's DOM is at `document.body`, click events on the modal won't bubble to React ancestors. They do — through the React tree. A "click outside to close" implementation that listens at the React ancestor level will fire even for clicks inside a portal modal, unless the modal calls `e.stopPropagation()`.

**6. Portal accessibility.** Rendering a modal at `document.body` without managing focus means keyboard users can still tab through background content. Portals do not provide focus trapping. You must implement focus trapping separately — either with a library (focus-trap-react) or manually by intercepting Tab and Shift+Tab keydowns and confining focus to the modal's focusable elements.

**7. Prop-based slots vs. compound components — static vs. dynamic.** Named props for slots accept JSX values that are evaluated at the call site. If the content needs access to state that's inside the `Modal` component (e.g., the modal's own close animation state), a named prop can't get it without lifting state. Compound components using Context can expose internal state to their children. Choose accordingly.

---

## Interview Questions

**Q (High): What is `ReactDOM.createPortal` and why does it exist? What problem does it solve that normal rendering cannot?**

Answer: `createPortal` renders a component's output into a different DOM node than where the component sits in the React tree. It exists because CSS stacking contexts make deep DOM nesting problematic for overlaying UI. A modal nested inside a component with `overflow: hidden` or a `transform` property will be clipped or have its `z-index` constrained, regardless of how high the `z-index` value is. By rendering at `document.body`, the modal escapes all ancestor stacking contexts and renders on top of everything. Portals maintain the React component tree relationship — the portal's children are still inside the component in the React hierarchy — but their DOM output is physically at the target node. This has an important consequence for events: React synthetic events bubble through the React tree, not the DOM tree, so clicks inside a portal still bubble to the React ancestors of the portal component.

The trap: Not knowing the event bubbling behavior. This is the single most likely follow-up question and many candidates don't know it.

**Q (High): Explain the difference between `children` as a slot and named props as slots. When would you choose each?**

Answer: `children` is React's single default injection point — whatever is between the opening and closing tags of a component. It works well for components with one content area (a `<Card>`, a `<Button>`, a `<Section>`). Named props as slots — `<Modal header={...} footer={...}>body</Modal>` — provide multiple distinct, named injection points. They're explicit: the consumer states exactly which content goes where, ordering doesn't matter, and the component can apply different layout/styling to each region independently. The trade-off is ergonomics: `children` in JSX position feels natural, named props for layout content can look awkward for large JSX values. The choice depends on how many distinct content regions the component needs. One region: `children`. Two or more distinct regions with different styling or behavior: named props.

The trap: Recommending using `React.Children` to detect child types and routing them to different positions — this is brittle (breaks with HOC-wrapped children) and implicit.

**Q (High): When is `cloneElement` the right choice, and when is it a code smell?**

Answer: `cloneElement` is appropriate when you need to inject metadata into children that they cannot compute themselves — the canonical case is injecting index positions in a Compound Component's child list so `<Tab>` components know their position without the consumer specifying `index={n}`. It's also acceptable for merging event handlers (e.g., a `FocusRing` wrapper that needs to call both its own and the child's `onFocus`). It is a code smell when used to pass shared state, theme data, or callbacks — all of which Context handles better. The signal for the smell: if the parent has to know the child's internal prop interface to inject props, you have implicit coupling that Context would eliminate. The failure modes of `cloneElement` are concrete: it silently fails when children are HOC-wrapped (type check fails), it breaks with Fragment children, and it couples the parent's implementation to the child's prop names.

The trap: Saying cloneElement is always bad or always fine without distinguishing the legitimate narrow use case.

**Q (High): Why does a React Portal still receive events that bubble up through the React component tree, even though its DOM is at `document.body`?**

Answer: React's synthetic event system is attached to the React root container, not to individual DOM nodes. When a user interacts with a Portal's DOM content, the browser fires a native DOM event that bubbles up through the real DOM tree. React's event delegation at the root intercepts it. React then simulates event bubbling through the *React component tree* (virtual hierarchy), not the DOM tree. Since the Portal component is a child of the component that rendered it in the React tree — regardless of where the Portal's DOM output lives — the synthetic event bubbles through the React hierarchy as if the Portal were physically there. This is intentional: it means you can handle events from a modal dialog at the opener's level without any special wiring. The flip side is that "click outside to close" patterns that listen at the React ancestor level will fire for clicks inside the portal, so you need `e.stopPropagation()` inside the portal's content to prevent unwanted closure.

The trap: Confusing DOM event bubbling path (through document.body) with React synthetic event bubbling path (through the React tree). These are distinct and the difference matters in production.

**Q (Medium): What does `React.Children.map` do that a regular `.map()` on `children` cannot?**

Answer: `React.Children.map` normalizes the `children` prop before mapping. When a component receives a single child, `children` is a React element — not an array. Calling `.map()` on a React element throws because elements don't have a `.map` method. `React.Children.map` handles single elements, arrays of elements, nested arrays, nulls, and undefined correctly, always treating the collection uniformly. It also assigns stable keys to the returned elements. Regular `.map()` only works when you know `children` will always be an array — which requires the consumer to always pass at least two children, a fragile assumption. The safe default for any component that iterates children is `React.Children.map`.

The trap: Not knowing that a single child is an element, not an array, and therefore not seeing why the utility is needed.

**Q (Medium): What is the "children as function" pattern? When is it preferable to a named render prop?**

Answer: The "children as function" pattern is the render prop pattern expressed via the `children` prop rather than a named prop like `render`. Instead of `<DataFetcher render={({ data }) => ...} />`, you write `<DataFetcher>{({ data }) => ...}</DataFetcher>`. The component calls `children(state)` rather than `render(state)`. Both are functionally identical. The "children as function" form is preferred when the dynamic content is conceptually the "body" of the component — the natural JSX position feels more readable. Named render props are clearer when the component has multiple function props (e.g., `renderHeader` and `renderFooter`). The "children as function" pattern is also correctly called a render prop — they are the same pattern, different syntax. Today, both are largely replaced by custom hooks, but you encounter the children form in libraries like React Final Form, Formik, and some animation libraries.

The trap: Treating "children as function" and "render props" as different patterns — they are the same pattern with syntactic variation.

**Q (Low): What is the default slot anti-pattern in React and how do you fix it?**

Answer: The anti-pattern is relying on children's position or order to convey semantic meaning: `<Layout><Sidebar /><Main /></Layout>` where `Layout` assumes the first child is the sidebar and the second is the main area. This is invisible to consumers (nothing in the API surface says "first child = sidebar"), breaks if someone reverses the order, and is impossible for TypeScript or PropTypes to catch. The fix is named props: `<Layout sidebar={<Sidebar />} main={<Main />} />`. This makes the intent explicit, is immune to reordering, and is self-documenting in the component's prop types. Some teams use the Compound Component pattern instead — `<Layout.Sidebar>` and `<Layout.Main>` — which makes the distinction explicit without relying on position. The rule of thumb: if `children` ordering matters and the parent has to maintain positional assumptions about children, switch to named props.

The trap: Not being aware this is a pattern that occurs in real codebases and not having a concrete fix.

---

## Self-Assessment

Before moving on, check off each item you can do WITHOUT looking at this file.

- [ ] Can explain when `children` is sufficient and when named props as slots are the right choice
- [ ] Can explain why `ReactDOM.createPortal` exists and what CSS property combinations make it necessary
- [ ] Can explain why React synthetic events bubble through the React tree (not the DOM tree) when using Portals, and what the practical implication is for "click outside to close" patterns
- [ ] Can name the two legitimate uses of `cloneElement` and explain why Context is better for everything else
- [ ] Can explain why `React.Children.map` is needed instead of `children.map()`
- [ ] Can identify the default slot anti-pattern and rewrite it correctly

---
*Next: Phase 6 — Core Web Vitals: LCP, INP, CLS — with the component architecture patterns established, the next phase covers how architectural choices directly impact the performance metrics browsers and Google measure.*
