# CSS Performance: Critical CSS, CSS-in-JS, Utility CSS

## Quick Reference

| Approach | Rendering blocking | Bundle size | Runtime cost | Best for |
|---|---|---|---|---|
| External stylesheet | Blocks rendering until loaded | Separate HTTP request | Zero | Most applications |
| Critical CSS inlined | Non-blocking (inline) + async remainder | Inline + async | Zero | High-LCP pages, SSR |
| CSS-in-JS (runtime) | Depends on framework | JS bundle includes styles | Serialize + inject on each render | Component-level isolation with dynamic styles |
| CSS-in-JS (static extract) | Same as external stylesheet | Extracted at build time | Zero | Component isolation without runtime cost |
| Utility CSS (Tailwind) | Blocks, but stylesheet is tiny | Only used utilities shipped | Zero | Rapid UI, consistent design system |
| CSS Modules | Same as external stylesheet | Per-component, dead-code possible | Zero | Scoped styles without JS overhead |

---

## What Is This?

CSS performance covers how stylesheets are loaded, how they block rendering, and how different styling approaches affect runtime performance. CSS is render-blocking by default — the browser cannot display any content until all CSS in the `<head>` has been downloaded, parsed, and the CSSOM is constructed. Poor CSS loading strategy is a common cause of slow LCP.

Beyond loading, runtime CSS performance addresses style recalculation cost: how many elements the browser must check when a class changes, and which CSS properties trigger layout (reflow) vs. composite-only updates.

> **Check yourself:** Why does a `<link rel="stylesheet">` block rendering? What specifically does the browser wait for before painting the first frame?

---

## Why Does It Exist?

The browser's rendering pipeline requires both the DOM and the CSSOM before it can construct the Render Tree and compute layout. A `<link rel="stylesheet">` in `<head>` is render-blocking: the browser halts HTML parsing, downloads the CSS file, parses it, and builds the CSSOM before painting anything. Every millisecond the CSS takes to download delays the first paint.

As applications grew, stylesheet sizes ballooned. CSS-in-JS emerged to solve scope pollution and dead style problems. Utility frameworks emerged to solve specificity wars and CSS bloat. Each approach makes different trade-offs on file size, rendering timing, and runtime cost.

---

## Critical CSS Extraction

### The Problem

A typical stylesheet contains styles for every page, every component, and every state. The browser downloads the entire stylesheet before painting — even though only the styles for the above-the-fold content matter for the initial render.

**Critical CSS** is the minimal set of CSS rules needed to render above-the-fold content. If inlined in the `<head>` as a `<style>` block, it eliminates the render-blocking network request for the initial paint. The full stylesheet then loads asynchronously.

### How It Works

```html
<head>
  <!-- 1. Inline critical CSS — no network request, no render block -->
  <style>
    /* Only styles needed for above-the-fold content */
    body { margin: 0; font-family: Inter, sans-serif; }
    .hero { background: #0047AB; color: #fff; padding: 4rem 2rem; }
    .hero h1 { font-size: 2.5rem; font-weight: 700; margin: 0; }
    .nav { display: flex; gap: 1rem; padding: 1rem 2rem; background: #fff; }
  </style>

  <!-- 2. Load full stylesheet asynchronously — won't block rendering -->
  <link rel="preload" as="style" href="/styles.css" onload="this.onload=null;this.rel='stylesheet'" />
  <noscript><link rel="stylesheet" href="/styles.css" /></noscript>
</head>
```

The `rel="preload"` + `onload` trick: preload fetches the stylesheet at high priority without blocking rendering. When it loads, the `onload` handler switches `rel` to `stylesheet`, applying it to the page. The `noscript` fallback handles JS-disabled environments.

### Tooling

```bash
# critical — extracts above-the-fold CSS for given HTML
npx critical index.html --inline --base dist/ > dist/index.html

# critters — webpack/vite plugin for inlining critical CSS
npm install critters
```

**Critters in Next.js:**

Next.js can be configured with `critters` or uses its own critical CSS extraction. In Remix and Astro, critical CSS is handled at the framework level.

**The threshold problem:** Critical CSS extraction is heuristic — it simulates a viewport and extracts styles for visible elements. Dynamic content (JS-rendered components) may be missed. Always test actual LCP improvement after implementing critical CSS, not just theoretical.

---

## CSS-in-JS

### Runtime CSS-in-JS

Libraries like styled-components (before v6) and Emotion in runtime mode generate CSS class names and inject `<style>` tags at runtime — in the browser, on every render.

```js
// styled-components — runtime CSS-in-JS
import styled from 'styled-components';

const Button = styled.button`
  background: ${({ $primary }) => $primary ? '#0047AB' : '#fff'};
  color: ${({ $primary }) => $primary ? '#fff' : '#0047AB'};
  padding: 0.5rem 1rem;
  border: 2px solid #0047AB;
  border-radius: 4px;
`;

// Usage
<Button $primary>Save</Button>
<Button>Cancel</Button>
```

**Runtime CSS-in-JS cost:**

1. **Serialization**: Every render evaluates the template literal, interpolates dynamic values, and generates a unique class name string.
2. **Style injection**: A new `<style>` rule is injected into the `<head>` if the class name is novel.
3. **Bundle size**: The CSS-in-JS runtime (styled-components: ~16 KB, emotion: ~7 KB gzipped) is included in every page.
4. **SSR complexity**: Dynamic styles must be collected server-side and serialized into the HTML. Without SSR support, the browser must re-inject styles on hydration, causing a flash of unstyled content.

Runtime CSS-in-JS has real INP cost because style serialization and injection happen synchronously during React's render phase, blocking the main thread.

---

### Zero-Runtime CSS-in-JS (Build-Time Extraction)

Libraries like Linaria, Vanilla Extract, and styled-components v6 extract CSS at build time. The output is a static `.css` file — zero runtime cost.

```js
// Vanilla Extract — zero-runtime CSS-in-JS
// button.css.ts — TypeScript file, not a CSS file
import { style, styleVariants } from '@vanilla-extract/css';

export const base = style({
  padding: '0.5rem 1rem',
  border: '2px solid #0047AB',
  borderRadius: '4px',
  cursor: 'pointer',
});

export const variants = styleVariants({
  primary: {
    background: '#0047AB',
    color: '#fff',
  },
  secondary: {
    background: '#fff',
    color: '#0047AB',
  },
});
```

```js
// Button.jsx
import * as styles from './button.css';
import clsx from 'clsx';

function Button({ variant = 'secondary', children }) {
  return (
    <button className={clsx(styles.base, styles.variants[variant])}>
      {children}
    </button>
  );
}
```

At build time, Vanilla Extract generates `button.css` with real class names. There is no runtime style injection — the component applies class names, and the static CSS is loaded as a normal stylesheet.

**Trade-off:** No dynamic styles based on runtime JS values (component props that aren't enumerable at build time). All variants must be defined statically. If you need dynamic styles, you must use CSS custom properties:

```js
// Dynamic styles via CSS custom properties — zero runtime overhead
export const dynamicColor = style({
  color: 'var(--button-color)',
});
```

```js
// Set the CSS variable dynamically — browser handles interpolation
<button
  className={styles.dynamicColor}
  style={{ '--button-color': userColor }}
>
```

---

## Utility CSS (Tailwind)

### How Tailwind Works

Tailwind CSS is a utility-first framework: instead of writing `.button { ... }` in a CSS file, you compose styles from single-purpose utility classes directly in HTML/JSX.

```jsx
// Utility CSS approach
function Button({ primary, children }) {
  return (
    <button
      className={`
        px-4 py-2 rounded border-2 border-blue-700 cursor-pointer font-medium
        ${primary
          ? 'bg-blue-700 text-white hover:bg-blue-800'
          : 'bg-white text-blue-700 hover:bg-blue-50'
        }
      `}
    >
      {children}
    </button>
  );
}
```

Tailwind's build step scans all your source files, identifies which utility classes are used, and generates a stylesheet containing only those classes. A large application typically produces a 5–20 KB (gzipped) stylesheet, because only used utilities are included.

```bash
# Tailwind CLI build
npx tailwindcss -i ./src/input.css -o ./dist/output.css --minify

# With content scanning configured in tailwind.config.js
module.exports = {
  content: ['./src/**/*.{html,js,jsx,ts,tsx}'],
  theme: {
    extend: {},
  },
};
```

**Why the stylesheet is small:** A traditional stylesheet grows with every feature added. Tailwind's stylesheet is bounded by the number of utility classes you use — adding a new component reuses existing utilities rather than adding new CSS. In the limit, a large Tailwind app and a small Tailwind app produce similar stylesheet sizes because they reuse the same atomic classes.

### Performance Characteristics

- **No render-blocking overhead** beyond a small stylesheet — less render-blocking time than a traditional per-component stylesheet that grows unbounded.
- **Zero runtime cost** — class names are applied statically in JSX, no style serialization or injection.
- **No dead CSS** — only utilities used in source files are generated.
- **Excellent caching** — the generated stylesheet changes only when you use new utilities or change the Tailwind config.

### Downsides

- JSX becomes verbose — long class strings for complex components.
- Dynamic styles require conditional class concatenation — error-prone without a helper (`clsx`, `cva`).
- Custom design tokens require Tailwind config — harder to use arbitrary values at scale.

```js
import { cva } from 'class-variance-authority';

const button = cva(
  'px-4 py-2 rounded border-2 cursor-pointer font-medium',
  {
    variants: {
      intent: {
        primary: 'border-blue-700 bg-blue-700 text-white hover:bg-blue-800',
        secondary: 'border-blue-700 bg-white text-blue-700 hover:bg-blue-50',
      },
    },
    defaultVariants: {
      intent: 'secondary',
    },
  }
);

<button className={button({ intent: 'primary' })}>Save</button>
```

---

## CSS Selector Performance

Modern browsers are extremely fast at CSS selector matching — selector performance is rarely a meaningful bottleneck in 2025. However, some patterns still matter at scale:

**Avoid deep descendant selectors on frequently-updated elements:**

```css
/* Slow: must walk up the ancestor chain for every .item */
.page .content .sidebar .widget .item { ... }

/* Fast: scoped class, no ancestor traversal needed */
.widget-item { ... }
```

**Avoid `*` selectors in frequently-updated subtrees:**

```css
/* Triggers style recalculation on every element in .list */
.list * { box-sizing: border-box; }

/* Apply at a higher level, not inside a virtual-scroll container */
```

**`contain: strict` for isolated subtrees:**

```css
/* Tell the browser this element's layout does not affect anything outside it */
.virtualized-list {
  contain: strict; /* layout, style, paint, size containment */
}
```

`content-visibility: auto` (a related property) skips layout and paint for off-screen elements:

```css
.article-card {
  content-visibility: auto;
  contain-intrinsic-size: 0 300px; /* estimated height — prevents scroll jump */
}
```

---

## Render-Blocking vs. Non-Blocking CSS

| Load pattern | Render blocking? | How |
|---|---|---|
| `<link rel="stylesheet">` in `<head>` | Yes | Browser halts parsing, downloads, parses, builds CSSOM |
| `<style>` inline in `<head>` | Yes (but no network wait) | Same CSSOM construction, but bytes are in the HTML |
| `<link rel="stylesheet">` at bottom of `<body>` | Practically no | Browser has already painted by the time it's parsed |
| `<link rel="preload" as="style">` with `onload` swap | No | Fetches at high priority, applies after initial paint |
| CSS imported in a dynamic `import()` | No (until script runs) | Not in the initial page load at all |

> **Check yourself:** If you move a `<link rel="stylesheet">` to the end of `<body>`, what happens if the user sees a flash of unstyled content?

---

## Interview Questions

**Q (High): Why is CSS render-blocking, and how does critical CSS extraction improve LCP?**
Answer: The browser cannot paint until it has constructed the Render Tree, which requires both the DOM and the CSSOM. A `<link rel="stylesheet">` in the `<head>` pauses HTML parsing until the stylesheet is downloaded, parsed, and the CSSOM is built. Every millisecond of CSS load time delays the first paint and worsens LCP. Critical CSS extraction solves this by inlining the minimal CSS needed for above-the-fold content directly in the HTML as a `<style>` block — no network request needed, no render block. The remaining CSS is loaded asynchronously (via `rel="preload"` + `onload` swap) so it doesn't block the initial paint. The LCP element, which is typically above the fold, gets styled on the first paint without waiting for the full stylesheet to arrive.
The trap: Thinking inline `<style>` is not render-blocking. It still blocks (the browser still builds the CSSOM), but it eliminates the network latency, which is the actual cost.

**Q (High): What is the runtime cost of CSS-in-JS like styled-components, and how does zero-runtime CSS-in-JS like Vanilla Extract solve it?**
Answer: Runtime CSS-in-JS has three costs: (1) Template literal serialization — on every render, the library evaluates the tagged template, interpolates dynamic props, and generates a hash-based class name. (2) Style injection — if the class name is new, a `<style>` rule is injected into the `<head>` via the CSSOM API. (3) Bundle size — the runtime library (7–16 KB gzipped) must be shipped. In React, this serialization happens synchronously during the render phase, contributing to INP and slower interaction responsiveness. Vanilla Extract and Linaria extract CSS at build time — the TypeScript/JS style definitions are compiled to static `.css` files, and components receive stable class names. There is no runtime serialization, no style injection, and no library code in the JS bundle. The trade-off: styles must be statically expressible at build time; truly dynamic styles (colors derived from runtime user data) require CSS custom properties.
The trap: Not knowing that Emotion and styled-components have zero-runtime modes/successors that change this trade-off.

**Q (High): How does Tailwind CSS end up with a small bundle in production even for large applications?**
Answer: Tailwind scans source files at build time using the `content` configuration and generates a stylesheet containing only the utility classes that actually appear in your code. A large application that uses 150 utilities produces the same stylesheet as a small one using 150 utilities — the stylesheet doesn't grow with features, it grows with unique utilities. Traditional "write custom CSS per component" approaches produce stylesheets that grow linearly with the number of components. Tailwind's stylesheet stabilizes: after you've used the core layout utilities, adding a new component mostly reuses existing utility classes and adds nothing to the stylesheet. Typical production Tailwind stylesheets are 5–20 KB gzipped for full applications. The critical configuration requirement is the `content` array — if it doesn't include all files that use Tailwind classes, those classes won't be generated and styles will be missing in production.
The trap: Thinking Tailwind just ships all utilities or that larger apps always produce larger stylesheets.

**Q (Medium): When would you choose CSS Modules over Tailwind or CSS-in-JS?**
Answer: CSS Modules are appropriate when you want traditional CSS syntax with automatic class name scoping and no runtime cost, and when the team has strong CSS skills or an existing CSS codebase to migrate. They let you write normal CSS selectors, media queries, and pseudo-classes without learning a utility class system. IDE support for autocompletion is better than runtime CSS-in-JS. The downside compared to Tailwind: CSS Modules can accumulate dead styles (classes defined but never used), and there's no purging step. Compared to Vanilla Extract: CSS Modules don't offer type-safety for class names (a typo in a className string is a silent runtime error rather than a compile error). Choose CSS Modules for a team migrating from global CSS, for projects with strict CSS organization conventions, or when you want full CSS syntax support without any build-time preprocessing constraints.
The trap: Not distinguishing CSS Modules from global CSS — the scoping is the entire value proposition.

**Q (Medium): What does `content-visibility: auto` do and what is the performance risk without `contain-intrinsic-size`?**
Answer: `content-visibility: auto` tells the browser to skip layout and paint for off-screen elements — the browser treats them as if they have `visibility: hidden` for rendering purposes, but they remain in the DOM and accessible. For content-heavy pages (long articles, timelines), this can dramatically reduce initial layout and paint work. The risk without `contain-intrinsic-size`: the browser doesn't know how large the skipped element will be. Without a size estimate, it treats the element as 0-height while off-screen, making the scrollbar inaccurate — the page appears shorter than it is, and the scrollbar jumps as the user scrolls and elements are measured. `contain-intrinsic-size: 0 300px` provides an estimated height so the browser reserves space before measuring the actual element. The estimate doesn't need to be exact — just close enough to prevent scroll jumping.
The trap: Recommending `content-visibility: auto` without mentioning `contain-intrinsic-size`.

**Q (Low): Why is moving a stylesheet to the bottom of `<body>` not a valid alternative to critical CSS extraction?**
Answer: Moving a stylesheet to `</body>` avoids the initial render block because the browser has already painted above-the-fold content by the time it encounters the stylesheet. However, users see a Flash of Unstyled Content (FOUC) — the page renders in unstyled HTML for a moment before the styles apply. This is a jarring UX regression worse than waiting an extra few hundred milliseconds for a styled paint. Critical CSS extraction avoids FOUC: the critical inline styles ensure the above-the-fold content looks correct on first paint, and the deferred full stylesheet applies smoothly after. The user never sees unstyled content.

---

## Self-Assessment

Before moving on, check off each item you can do WITHOUT looking at the file.

- [ ] Explain why CSS is render-blocking and write the `rel="preload"` + `onload` pattern for async CSS loading
- [ ] Describe the three runtime costs of CSS-in-JS and how zero-runtime CSS-in-JS (Vanilla Extract) eliminates them
- [ ] Explain why Tailwind's production stylesheet stays small regardless of application size
- [ ] Describe what `content-visibility: auto` does and why `contain-intrinsic-size` is required alongside it
- [ ] Compare CSS Modules, Tailwind, runtime CSS-in-JS, and zero-runtime CSS-in-JS on: render blocking, bundle size, runtime cost
- [ ] Explain why moving a stylesheet to `</body>` is not equivalent to critical CSS extraction

---
*Next: Animation Performance — CSS performance covers how styles load and recalculate; animation performance covers how to move things on screen without triggering layout or causing jank.*
