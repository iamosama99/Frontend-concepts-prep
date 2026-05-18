# Graceful Degradation & Progressive Enhancement

## Quick Reference

| Approach | Philosophy | Starting Point | Failure Mode |
|---|---|---|---|
| Progressive Enhancement | Build from baseline up | Works without JS/CSS | Loses enhancements on old browsers |
| Graceful Degradation | Build full experience, add fallbacks | Full-featured on modern browsers | May be incomplete on old browsers |

---

## The Core Distinction

These two approaches share a goal — every user gets a usable experience — but they go in opposite directions.

**Progressive Enhancement:** Start with the minimum viable experience (semantic HTML, no JS). Layer CSS on top of that. Layer JavaScript on top of CSS. Each layer enhances what works below it. If a layer fails to load or execute, the previous layer still works.

**Graceful Degradation:** Build the full experience for modern browsers. Then add fallbacks so the experience degrades reasonably on older browsers or when features aren't available. If a feature isn't supported, the fallback takes over.

**In practice:** Progressive enhancement is a design philosophy applied upfront. Graceful degradation is often applied after the fact — "what happens if this breaks?"

Both are relevant in modern frontend. You can think of them as the same coin: progressive enhancement is degradation done proactively from the start.

---

## Progressive Enhancement in Practice

**HTML first — the baseline:**

A form that works without JavaScript:

```html
<!-- Works with zero JS — browser handles submit natively -->
<form method="POST" action="/subscribe">
  <label for="email">Email</label>
  <input type="email" id="email" name="email" required>
  <button type="submit">Subscribe</button>
</form>
```

**Enhanced with JavaScript:**

```typescript
// Enhance the form with async submission (no page reload)
const form = document.querySelector('form');
form?.addEventListener('submit', async (e) => {
  e.preventDefault(); // only intercept if JS is running

  const data = new FormData(form);
  try {
    await fetch('/subscribe', { method: 'POST', body: data });
    showSuccessMessage();
  } catch {
    // JS failed — let the form submit natively as fallback
    form.submit();
  }
});
```

The form works without JS (native submit), works better with JS (no page reload), and falls back to native submit if the fetch fails.

**CSS enhancement pattern:**

```css
/* Baseline: works everywhere */
.menu { display: block; }

/* Enhancement: if grid is supported, use it */
@supports (display: grid) {
  .menu { display: grid; grid-template-columns: repeat(auto-fit, minmax(200px, 1fr)); }
}
```

---

## Feature Detection

Feature detection is the technical mechanism behind both approaches. You check whether a feature exists before using it, rather than checking the browser name.

```typescript
// Bad: browser sniffing — brittle, wrong as browsers update
if (navigator.userAgent.includes('Chrome')) {
  enableIntersectionObserver();
}

// Good: feature detection
if ('IntersectionObserver' in window) {
  enableLazyLoading();
} else {
  // Load all images immediately (graceful degradation)
  loadAllImages();
}
```

```typescript
// CSS feature detection in JS
function supportsDisplayGrid(): boolean {
  const el = document.createElement('div');
  el.style.display = 'grid';
  return el.style.display === 'grid';
}

// Prefer CSS @supports over JS for CSS features
```

**The `@supports` at-rule — CSS feature detection:**

```css
/* Animation: use GPU-composited transform if supported, fallback to position */
.animated {
  left: 0; /* fallback */
  transition: left 0.3s;
}

@supports (transform: translateX(0)) {
  .animated {
    left: auto;
    transform: translateX(0);
    transition: transform 0.3s;
  }
}
```

---

## Graceful Degradation Patterns

**`<noscript>` for no-JS environments:**

```html
<noscript>
  <style>.js-only { display: none; }</style>
  <p>For the best experience, please enable JavaScript.</p>
</noscript>
```

**Capability fallback:**

```typescript
async function copyToClipboard(text: string): Promise<void> {
  if (navigator.clipboard?.writeText) {
    await navigator.clipboard.writeText(text);
    return;
  }

  // Fallback: old execCommand technique (deprecated but wide support)
  const textarea = document.createElement('textarea');
  textarea.value = text;
  textarea.style.position = 'fixed';
  textarea.style.opacity = '0';
  document.body.appendChild(textarea);
  textarea.focus();
  textarea.select();
  document.execCommand('copy');
  document.body.removeChild(textarea);
}
```

**Picture element — image format degradation:**

```html
<!-- Browser picks the first source it supports -->
<picture>
  <source srcset="/image.avif" type="image/avif">
  <source srcset="/image.webp" type="image/webp">
  <img src="/image.jpg" alt="Description" loading="lazy"> <!-- JPEG fallback -->
</picture>
```

**Font display degradation:**

```css
@font-face {
  font-family: 'CustomFont';
  src: url('font.woff2') format('woff2'), url('font.woff') format('woff');
  font-display: swap; /* show system font while loading, swap when ready */
}
```

---

## Server-Side Rendering as Progressive Enhancement

SSR is a form of progressive enhancement at the rendering layer: the server sends complete HTML that renders without any client-side JavaScript. JS then "enhances" this into an interactive SPA (hydration). If hydration fails or is slow, the static HTML is still readable.

```typescript
// Next.js page — HTML arrives fully rendered from the server
// JS hydration enhances it to be interactive
export default function Article({ content }: { content: string }) {
  return (
    <article>
      <div dangerouslySetInnerHTML={{ __html: content }} /> {/* readable without JS */}
      <ShareButtons /> {/* interactive enhancement — only works after hydration */}
    </article>
  );
}
```

---

## Where to Draw the Line

Not everything needs a no-JS fallback. A web app requiring JavaScript is acceptable — the browser isn't a constraint that demands HTML-only support by default. The question is: **what's the reasonable baseline for your users?**

- Public content pages (news, docs, blogs): strong case for full progressive enhancement — search bots and users on slow connections benefit from HTML-first.
- Authenticated app dashboards: JavaScript is a baseline assumption — graceful degradation for individual features is sufficient.
- Third-party embeds: strong case for `<noscript>` fallbacks since you don't control the embedding context.

> **Check yourself:** You're building a modal component. How would you design it using progressive enhancement principles? What works without JS?

---

## Self-Assessment

- [ ] I can articulate the difference between progressive enhancement and graceful degradation
- [ ] I know why feature detection is preferred over browser sniffing
- [ ] I can implement CSS-only fallbacks using `@supports`
- [ ] I understand how SSR + hydration is a form of progressive enhancement
- [ ] I know when to use `<picture>` + `<source>` for image format degradation
- [ ] I can reason about which features warrant no-JS fallbacks vs. which don't

---

## Interview Q&A

**Q: What is progressive enhancement and how does it differ from graceful degradation?** `High`

Progressive enhancement starts with a working baseline experience (HTML that renders and functions without CSS or JS) and layers improvements on top. Graceful degradation starts with the full-featured, modern experience and adds fallbacks for when features aren't available. Both aim for broad usability, but progressive enhancement is a design-first approach while graceful degradation is often retrofitted. In practice, a well-structured codebase does both: it's designed with progressive enhancement in mind and has specific degradation handling for capabilities that may not be available.

---

**Q: Why is feature detection preferred over browser sniffing?** `High`

Browser sniffing uses the user agent string to infer capabilities. This is brittle for several reasons: the UA string is easily spoofed, browser capabilities change with updates, new browsers won't be in your list, and some browsers lie about their identity for compatibility. Feature detection tests whether a specific API or property exists right now in the current environment. It's accurate regardless of which browser is running and stays correct as browsers add support for new features. It also works correctly in non-browser environments (SSR, testing frameworks) where UA sniffing produces nonsense.

---

**Q: How does SSR relate to progressive enhancement?** `Medium`

SSR sends fully rendered HTML from the server, which is readable and functional (links, forms) before any client JavaScript runs. This is the "baseline" in progressive enhancement terms. JavaScript then hydrates the page to make it interactive. If hydration fails or the network is slow, the user can still read the content and use native form submissions. This is particularly valuable for public content pages where search engines and users on slow connections benefit from the immediate HTML response.

---

**Q: What is the `@supports` rule and when do you use it?** `Medium`

`@supports` is a CSS at-rule that applies styles only when a specific CSS property and value combination is supported. It's the CSS equivalent of feature detection in JavaScript. You use it to layer enhancements: define the baseline style outside the `@supports` block, then override with the enhanced version inside. Browsers that don't support the feature ignore the `@supports` block and use the baseline. It's more reliable than testing CSS support in JS and keeps the styling logic in CSS where it belongs.

---

**Q: When is it acceptable to require JavaScript as a baseline?** `Low`

For authenticated application interfaces (dashboards, editors, admin panels), JavaScript can reasonably be treated as a requirement. The user has already accepted an account-based product and the browser is a known baseline. The no-JS concern is most relevant for public-facing content pages (SEO, accessibility, slow networks), embedded widgets in third-party contexts, and critical user flows like checkout or form submission. For content that should be indexed by search engines or accessible without configuration, progressive enhancement to a working HTML state is worth the investment.
