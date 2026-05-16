# Design System Architecture: Headless UI, Design Tokens, Theming

## Quick Reference

| Layer | What it is | Example |
|-------|-----------|---------|
| Design tokens | Named design decisions mapped to values | `--color-brand-primary: #0066FF` |
| Headless / primitives | Behavior + accessibility, zero opinion on style | Radix UI `Dialog`, React Aria `Button` |
| Themed components | Tokens + headless + your visual design | Your product's `Button`, `Modal` |
| Styled component library | All three bundled, opinionated styling | MUI, Chakra UI, Ant Design |

---

## What Is This?

A design system is not a Figma file and not just a component library. At the engineering level it is a **layered contract** between designers and engineers: a shared vocabulary of decisions (tokens), reusable interaction primitives (headless components), and a finished component layer that combines both into the product's visual language.

The three layers form a dependency chain: tokens sit at the base and express every design decision as a named variable. Primitives sit above tokens and provide accessibility and keyboard behavior without imposing visual opinions. Themed components sit at the top and compose primitives with tokens to produce the actual UI your users see.

Understanding this layering is what separates teams that ship maintainable UI at scale from teams whose component library collapses under its own inconsistency.

> **Check yourself:** Without looking at any code, can you describe what problem each of the three layers solves and why merging them into one layer causes pain?

---

## Why Does It Exist?

### The problem without a design system

Before formalized design systems, teams accumulated ad-hoc component files: a `Button.jsx` in one feature, a `Btn.jsx` in another, inline `style` tags scattered everywhere. The consequences were predictable:

- **Visual drift.** The same spacing concept had eight values across the codebase because no one agreed on the canonical value for "large padding."
- **Accessibility regressions.** Every developer reimplemented focus management, aria labels, and keyboard interaction — incorrectly and inconsistently.
- **Rebrand cost.** Changing the brand color meant a grep across hundreds of files and inevitable misses.
- **Multi-product inconsistency.** If the company had three products, each had its own slightly different component set with no shared primitives.

### Why the three-layer model emerged

The naive fix is "build a shared component library." Teams did this and hit a different wall: styled component libraries force a visual opinion on consumers. If your library's `Button` is hard-coded with MUI's visual language, any team that wants a different visual treatment has to either fork the library, fight specificity wars, or accept the lock-in. The library becomes a constraint rather than an enabler.

The insight was to **separate concerns**:
1. Design decisions should be variables, not hardcoded values.
2. Accessibility and interaction logic is universal — it should be shared, not reimplemented.
3. Visual expression is product-specific — it should be composable on top.

This separation is what gives design systems longevity.

---

## How It Works

### Layer 1: Design Tokens

A design token is a named, intentional design decision stored as a variable. Tokens move design values out of component source code and into a single source of truth.

```js
// Without tokens — values are magic numbers scattered everywhere
const Button = styled.button`
  background: #0066FF;
  padding: 8px 16px;
  border-radius: 4px;
  font-size: 14px;
`;

// With tokens — values are referenced by name
const Button = styled.button`
  background: var(--color-action-primary);
  padding: var(--space-2) var(--space-4);
  border-radius: var(--radius-sm);
  font-size: var(--text-sm);
`;
```

The difference is not cosmetic. When the design team decides to change the brand color, updating one token value propagates to every component that references it. When you add a dark mode, you swap token values without touching component code.

### Token Taxonomy: Global → Semantic → Component

Well-designed token systems have at least two levels of abstraction.

**Global tokens** (also called reference tokens) are the raw palette — every possible value the system might use:

```css
/* Global tokens: literal values */
:root {
  --color-blue-500: #0066FF;
  --color-blue-600: #0052CC;
  --color-gray-100: #F5F5F5;
  --color-gray-900: #111111;
  --space-1: 4px;
  --space-2: 8px;
  --space-4: 16px;
}
```

**Semantic tokens** (also called alias tokens) give meaning to global values. They describe *what* the value is used for, not *what* the value is:

```css
/* Semantic tokens: intent-based names */
:root {
  --color-action-primary: var(--color-blue-500);
  --color-action-primary-hover: var(--color-blue-600);
  --color-surface-default: var(--color-gray-100);
  --color-text-default: var(--color-gray-900);
}
```

**Component tokens** (optional third level) allow per-component overrides without breaking the semantic layer:

```css
/* Component-level tokens */
:root {
  --button-background: var(--color-action-primary);
  --button-background-hover: var(--color-action-primary-hover);
  --button-padding-x: var(--space-4);
}
```

This taxonomy is why `color-brand-primary` and `color-button-background` are both needed. `color-brand-primary` is a semantic token that expresses your brand. `color-button-background` references `color-brand-primary` today — but tomorrow a designer might decide the button should use a different shade while keeping the brand primary color for other uses. Component tokens give you that flexibility without changing global or semantic tokens.

### Multi-Brand Theming via Token Swapping

The canonical technique is to define your semantic tokens on a theme root selector and swap the entire set by changing the root:

```css
/* Brand A (default) */
[data-theme="brand-a"] {
  --color-action-primary: #0066FF;
  --color-surface-default: #FFFFFF;
  --color-text-default: #111111;
}

/* Brand B */
[data-theme="brand-b"] {
  --color-action-primary: #E63946;
  --color-surface-default: #FAFAFA;
  --color-text-default: #1A1A1A;
}
```

Apply the theme at the root and every component that references semantic tokens responds automatically:

```js
document.documentElement.setAttribute('data-theme', 'brand-b');
```

No component code changes. No grep-and-replace. This is why semantic tokens exist — without them, you would have to replace every literal value across hundreds of components.

> **Check yourself:** If a team uses only global tokens (no semantic layer), what breaks when they try to add dark mode or a second brand?

---

### Layer 2: Headless UI Components

A headless component handles **behavior and accessibility** while being completely agnostic about visual styling. The consumer provides 100% of the visual treatment.

The problem headless components solve is that accessible interactive components are genuinely hard to build correctly. A disclosure widget (`<details>`-like behavior with animation) needs:

- `aria-expanded` toggled on trigger
- `aria-controls` linking trigger to panel
- `id` coordination between elements
- Focus management after open/close
- Keyboard support: Enter and Space to toggle, Escape to close
- Role attribution (`role="button"` if not a native `<button>`)

Every team that rolls their own gets at least some of this wrong. Headless libraries (Radix UI, Headless UI, React Aria) solve this once and let teams layer any visual treatment on top.

```js
// Radix UI Dialog — behavior only, you write all the styles
import * as Dialog from '@radix-ui/react-dialog';

function ConfirmModal({ onConfirm }) {
  return (
    <Dialog.Root>
      <Dialog.Trigger asChild>
        <button className="btn-primary">Delete item</button>
      </Dialog.Trigger>

      <Dialog.Portal>
        <Dialog.Overlay className="modal-overlay" />
        <Dialog.Content className="modal-content">
          <Dialog.Title className="modal-title">Are you sure?</Dialog.Title>
          <Dialog.Description className="modal-description">
            This action cannot be undone.
          </Dialog.Description>
          <button onClick={onConfirm}>Confirm</button>
          <Dialog.Close asChild>
            <button aria-label="Close">×</button>
          </Dialog.Close>
        </Dialog.Content>
      </Dialog.Portal>
    </Dialog.Root>
  );
}
```

Radix handles: focus trapping, scroll locking, `aria-modal`, Escape key dismissal, portal rendering, and `aria-labelledby`/`aria-describedby` wiring. Your `modal-overlay` and `modal-content` classes are 100% yours — any CSS, CSS-in-JS, or Tailwind classes work.

The **style conflict problem** that headless components solve: styled libraries inject their own CSS. If your app also has global CSS, specificity fights are inevitable — you end up writing `&&&&.MuiButton-root` overrides. Headless components have no injected CSS, so there's nothing to fight against.

### `asChild` — the composition primitive

`asChild` is a pattern popularized by Radix that merges the headless component's props onto its single child element rather than rendering its own DOM node:

```js
// Without asChild: renders <button><a href="/home">Go home</a></button> — invalid HTML
<Dialog.Trigger>
  <a href="/home">Go home</a>
</Dialog.Trigger>

// With asChild: renders <a href="/home" ...dialog-trigger-props...>Go home</a>
<Dialog.Trigger asChild>
  <a href="/home">Go home</a>
</Dialog.Trigger>
```

This lets you use any element or component as the trigger while still getting the correct aria attributes and event handlers from the headless layer.

---

### Layer 3: Styled Component Libraries

Libraries like MUI, Chakra UI, and Ant Design bundle all three layers — tokens, headless behavior, and visual styling — into one opinionated package. This is a deliberate trade-off:

**What you gain:** Speed to production. Every component is pre-styled, accessible, and themeable out of the box. For internal tools, admin dashboards, or resource-constrained teams, this is often the right choice.

**What you trade away:** Freedom. Your visual output is constrained by the library's opinion. Customization is possible but works against the grain — you're overriding decisions the library made rather than composing from scratch. Each library has its own theming API that you must learn and are coupled to.

**When to choose each:**

| Approach | Best for | Avoid when |
|----------|----------|------------|
| Headless + tokens (roll your own) | Product with strong brand identity, multiple themes, design system at scale | Small teams without design system infrastructure |
| Styled library (MUI, Chakra) | Internal tools, prototypes, teams without dedicated design | Brand requires significant visual differentiation from the library's defaults |
| White-label design system | Multiple products/brands from one codebase | Single-brand, single-product apps |

---

## Theming Architecture

### CSS Custom Property Cascades

CSS custom properties cascade like any other property — a property defined closer to an element in the DOM tree wins over one defined on an ancestor. This makes scoped theming trivial:

```css
/* Root defines defaults */
:root {
  --color-surface: #FFFFFF;
  --color-text: #111111;
}

/* A specific section can override locally */
.marketing-hero {
  --color-surface: #0066FF;
  --color-text: #FFFFFF;
}
```

Every element inside `.marketing-hero` that references `--color-surface` or `--color-text` gets the overridden values. Nothing outside `.marketing-hero` is affected. No JavaScript needed.

### Dark Mode Implementation Patterns

There are three approaches, each with trade-offs:

**Pattern 1: CSS class on `<html>` or `<body>`**

```css
:root { --color-surface: #FFFFFF; }
.dark { --color-surface: #111111; }
```

```js
document.documentElement.classList.toggle('dark', isDark);
```

Pro: JavaScript-controlled, so you can persist user preference to localStorage and apply it before first paint to avoid flash. Con: requires JS to run before rendering.

**Pattern 2: `prefers-color-scheme` media query only**

```css
:root { --color-surface: #FFFFFF; }

@media (prefers-color-scheme: dark) {
  :root { --color-surface: #111111; }
}
```

Pro: zero JavaScript, respects OS preference automatically. Con: no user override — you can't let users choose light mode on a dark-OS system.

**Pattern 3: `data-theme` attribute with media query fallback (recommended for production)**

```css
/* Default: light */
:root,
[data-theme="light"] {
  --color-surface: #FFFFFF;
  --color-text: #111111;
}

/* Explicit dark */
[data-theme="dark"] {
  --color-surface: #111111;
  --color-text: #EEEEEE;
}

/* OS-level fallback when no data-theme is set */
@media (prefers-color-scheme: dark) {
  :root:not([data-theme]) {
    --color-surface: #111111;
    --color-text: #EEEEEE;
  }
}
```

```js
// Read from localStorage, fall back to OS preference
const saved = localStorage.getItem('theme'); // 'light' | 'dark' | null
if (saved) {
  document.documentElement.setAttribute('data-theme', saved);
}
// If null, the media query handles it — no JS needed
```

This gives you OS-preference fallback, explicit user override, and persistence — without flash of wrong theme if the script is in `<head>` (inline, not deferred).

> **Check yourself:** Why does putting the theme-initialization script in `<head>` as an inline script (not `defer`/`async`) prevent a flash of the wrong color scheme on page load?

---

## Design Token Tooling: Style Dictionary

Style Dictionary (by Amazon) solves the multi-platform distribution problem. You define tokens once in JSON (or JS/YAML) and it transforms them into every platform's native format: CSS custom properties, JS ES module, iOS Swift, Android XML, Figma tokens, etc.

```json
// tokens/color.json — the single source of truth
{
  "color": {
    "brand": {
      "primary": { "value": "#0066FF", "type": "color" },
      "secondary": { "value": "#E63946", "type": "color" }
    },
    "surface": {
      "default": { "value": "{color.brand.primary}", "type": "color" }
    }
  }
}
```

Style Dictionary resolves references (`{color.brand.primary}`) and outputs:

```css
/* web/tokens.css */
:root {
  --color-brand-primary: #0066FF;
  --color-brand-secondary: #E63946;
  --color-surface-default: #0066FF;
}
```

```js
// web/tokens.js
export const tokens = {
  ColorBrandPrimary: '#0066FF',
  ColorBrandSecondary: '#E63946',
  ColorSurfaceDefault: '#0066FF',
};
```

The same JSON produces output for iOS Swift and Android XML. This is the mechanism that ensures a designer changing one value in the JSON propagates correctly to every platform with a single build step.

---

## Versioning and Breaking Changes

A shared design system is a dependency. Consuming teams (product teams) take a version and build against it. Treating this carelessly leads to system-wide pain.

### Semantic versioning in practice

Design systems should follow semver strictly:

- **Patch (1.0.x):** Bug fixes, accessibility corrections, typo fixes. Safe to auto-upgrade.
- **Minor (1.x.0):** New components, new tokens, new props. Backward-compatible additions.
- **Major (x.0.0):** Removed components, renamed tokens, changed prop APIs, changed DOM structure (which can break CSS selectors). Requires consuming teams to migrate.

### Strategies for managing breaking changes

**Token aliasing during migrations.** When renaming `--color-primary` to `--color-action-primary`, keep the old token pointing to the new one for one major version:

```css
:root {
  --color-action-primary: #0066FF;
  /* deprecated — will be removed in v4.0 */
  --color-primary: var(--color-action-primary);
}
```

Consuming teams get deprecation warnings in their design system linter but their UI doesn't break. They migrate at their own pace before the major version cut.

**Codemods.** For renamed props or component API changes, provide an AST-based codemod that consuming teams can run to automatically migrate their code.

**Changelogs with migration guides.** Every major version needs a dedicated migration guide — not just "Button prop `variant` renamed to `appearance`" but a before/after code example and an explanation of why.

**Canary channels.** Publish pre-release versions (`1.0.0-rc.1`) so early-adopter product teams can validate their integration before the stable release.

---

## Gotchas

**Semantic tokens are not optional.** Teams that skip the semantic layer and use global tokens directly find that dark mode requires touching every component — because `--color-blue-500` is hardcoded where `--color-text-default` should be. By the time this bites you, the codebase is large enough that the fix is painful.

**`data-theme` on `:root` vs. a wrapper div matters.** If you put `data-theme` on a wrapper `<div>` rather than `<html>`, elements that render outside that wrapper (portals, modals, tooltips) will not inherit the theme. Portals append to `document.body`, outside your wrapper. Put your theme attributes on `document.documentElement`.

**CSS custom properties are not compile-time constants.** Unlike preprocessor variables (Sass `$var`), CSS custom properties are resolved at runtime in the browser. This means they can be changed with JavaScript and cascade correctly — but it also means you cannot use them in `@media` queries or `@keyframes` steps that need static values. The browser evaluates them per element, not globally.

**Headless components still need a rendering host.** Radix, Headless UI, and React Aria all require React (or a supported framework). They are not true framework-agnostic web components. If you need framework-agnostic behavior primitives, look at Ariakit or web-component-based approaches.

**Token naming collisions in multi-brand setups.** If two brands override the same semantic token to different values and both run in the same page (e.g., an iframe embed or a microfrontend scenario), token values conflict. The solution is either shadow DOM isolation or a JS-in-CSS approach (CSS-in-JS with dynamic generation) rather than global custom properties.

**Versioning drift kills design system adoption.** If the design system team ships breaking changes without codemods or migration guides, product teams stop upgrading. A 12-month-old pinned version means the design system benefits (new components, accessibility fixes) never reach users. Migration tooling is not optional work — it is what makes the system a shared asset rather than a shared burden.

---

## Interview Questions

**Q (High): A product team wants to add a second brand (white-label) to an existing app. The app currently hardcodes color values in CSS. Walk through the full migration to a token-based theming system.**

Answer: The migration has four stages. First, audit the existing codebase and extract every unique design value (colors, spacing, radii, typography) into a global token set — named by value, not by intent. Second, define a semantic token layer that maps global tokens to intent-based names. At this stage, work with design to agree on naming: `--color-action-primary`, `--color-surface-default`, `--color-text-muted`. Third, do a sweep of all component CSS, replacing literal values with semantic token references. This is mechanical and automatable with a regex-based codemod if values are consistent. Fourth, define two `[data-theme]` rulesets — one for each brand — that map semantic tokens to different global token values. The swap is now a single `setAttribute` call. The key risk in this migration is skipping the semantic layer and mapping directly from global tokens to components — this saves time now but means adding a third brand later requires touching every component again.

The trap: Candidates describe adding CSS variables without explaining the two-level taxonomy. They treat all tokens as equivalent. The semantic layer is the payoff — without it you haven't solved multi-brand theming, you've just renamed your hardcoded values.

**Q (High): What is the "style conflict" problem that headless UI libraries solve, and how does the `asChild` pattern help?**

Answer: Styled libraries (and even partially-styled ones) inject CSS into the document. That CSS defines base styles for components via class names. When your application's own CSS targets the same elements — or when you try to override styles — you enter a specificity war. The library's styles may use `!important`, compound selectors, or high-specificity class chains. Overrides require matching or exceeding that specificity, which leads to fragile, order-dependent CSS. Headless components have no injected styles, so there is nothing to conflict with. You provide all classes and the headless layer provides only aria attributes and event handlers. The `asChild` pattern extends this by letting the headless component merge its behavior onto any element you choose — rather than rendering its own host element that you then style or replace. Without `asChild`, you get a headless wrapper element plus your element: two DOM nodes, one of which you can't fully control. With `asChild`, you get one element that is entirely yours but fully wired with the headless behavior.

The trap: Candidates describe `asChild` as "just a way to avoid wrapper divs." The real point is it preserves full ownership of the rendered element — semantic HTML, custom class names, native element behavior — while composing the headless behavior on top. The DOM-node count is a symptom, not the cause.

**Q (High): Explain why a flash of the wrong color scheme happens and how to prevent it.**

Answer: The flash happens because CSS is applied after the browser parses the `<html>` document and runs scripts. If your theme-initialization code lives in an external JS file loaded with `defer` or `async`, or at the bottom of `<body>`, the browser renders the page with default token values first, then JavaScript runs and sets `data-theme="dark"` — the user sees a white flash before dark mode applies. The prevention is to put a small inline `<script>` tag in `<head>` — before any `<link>` or `<body>` content — that reads from `localStorage` and sets the `data-theme` attribute synchronously. Because it is synchronous and in `<head>`, it runs before the browser paints. The browser then applies the correct token values on the first paint. The inline script must be small (no imports, no async) and must not block for long — its only job is reading a localStorage key and setting one attribute.

The trap: Candidates say "use `prefers-color-scheme`" — which only handles the OS default, not user override. Or they say "set it in CSS" — which cannot read localStorage. Or they add it to the JS bundle with `defer` — which reintroduces the flash because deferred scripts run after parsing.

**Q (Medium): What is the difference between global tokens and semantic tokens? Why do you need both?**

Answer: Global tokens are the full palette — every raw value available in the system, named by what the value is: `--color-blue-500: #0066FF`. They form the design vocabulary. Semantic tokens are a mapping from intent to a global token: `--color-action-primary: var(--color-blue-500)`. They form the usage contract. You need both because global tokens alone have no context — a component referencing `--color-blue-500` is tightly coupled to a specific shade of blue. If the brand color changes from blue to purple, you update `--color-blue-500`'s value but now the name is wrong and you have to rename every reference. With semantic tokens, `--color-action-primary` is always the primary action color regardless of what shade it resolves to. Swap its global token reference from `--color-blue-500` to `--color-purple-500` and nothing else changes. This is the mechanism that makes theming, rebranding, and dark mode feasible without touching component code.

The trap: Candidates conflate "design tokens" with "CSS variables." All CSS custom properties are CSS variables; design tokens are the *system* and *taxonomy* of naming design decisions. The two-level taxonomy is what makes the system scale.

**Q (Medium): When would you choose a styled component library (MUI/Chakra) over a headless-first approach? What signals tell you that choice was wrong?**

Answer: Choose a styled library when time-to-production and team bandwidth outweigh brand differentiation needs. Internal dashboards, admin tools, B2B SaaS with no white-labeling requirement, and early-stage products that need to validate quickly are all good candidates. The signals that the choice was wrong: the design team starts asking for visual treatments the library makes difficult (custom animation on a modal, a specific button shape that requires overriding multiple nested selectors). Engineers spend more time fighting the library's CSS than building features. Every new design spec involves a new `sx` prop or `styled(MuiComponent)` wrapper that creates a parallel component API alongside the library's. The upgrade path from v4 to v5 of the library requires touching every component. Once you have more override code than usage code, the library has become a liability.

The trap: Candidates give a binary "MUI is bad, headless is good" answer. The correct answer is about context. Styled libraries are genuinely the right choice for many products — the skill is recognizing the inflection point where the lock-in cost exceeds the velocity benefit.

**Q (Medium): How does Style Dictionary work and what problem does it solve that CSS custom properties alone cannot?**

Answer: Style Dictionary solves platform distribution. CSS custom properties exist only in the browser. A design system that serves a web app, a React Native app, an iOS app, and an Android app needs those token values available in four different formats: CSS custom properties for web, a JS object for React Native, Swift enums for iOS, and XML resources for Android. Style Dictionary takes a single JSON source of truth and, using a configurable transform/format pipeline, emits platform-specific output from one build step. It also handles reference resolution — if `color.surface.default` references `{color.brand.primary}`, Style Dictionary resolves the chain before emitting, so each platform gets the correct final value. What CSS custom properties cannot do: they are runtime constructs in a browser. They cannot produce a Swift file, cannot be consumed at build time, and cannot enforce that token names are used correctly (a token linter on top of Style Dictionary fills that gap).

The trap: Candidates describe Style Dictionary as just a "CSS variable generator." The value is cross-platform single source of truth — the web CSS output is incidental to the larger problem.

**Q (Low): What are component-level tokens and when are they worth the complexity?**

Answer: Component tokens are a third level of token indirection, specific to one component: `--button-background: var(--color-action-primary)`. They are worth the complexity when: (1) multiple consuming teams or products need to theme an individual component independently without touching the semantic layer, (2) the component has many internal surfaces (hover, active, disabled, focus ring) that need independent theming control, (3) the design system is published as an npm package and consumers need a stable, documented API for overriding component appearance without forking. They add boilerplate — every additional token is a name to maintain, document, and not break — so they should not be added speculatively. A good heuristic: add component tokens when you receive a support request that amounts to "I need to change X on just this component without changing it everywhere."

The trap: Candidates list component tokens as always necessary. They are an advanced feature that trades API surface area for flexibility. Adding them prematurely makes the token system harder to understand and maintain.

---

## Self-Assessment

Before moving on, check off each item you can do WITHOUT looking at the file.

- [ ] Explain the three-layer model (tokens → headless → themed) and what problem each layer solves
- [ ] Sketch the global → semantic → component token taxonomy and explain why the semantic level is required for multi-brand theming
- [ ] Describe how CSS custom property cascading enables scoped theming and why `data-theme` on `<html>` (not a wrapper div) is critical for portals
- [ ] Explain the flash-of-wrong-theme problem and write the inline `<head>` script that prevents it
- [ ] Compare headless UI libraries to styled component libraries and name the signal that tells you you've outgrown the styled approach
- [ ] Describe how Style Dictionary solves the multi-platform token distribution problem

---

*Next: Atomic Design Principles — design tokens and headless components are the building blocks; Atomic Design is the methodology for composing them into a coherent hierarchy of components.*
