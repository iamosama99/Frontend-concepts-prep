# Scroll Restoration

## Quick Reference

| Scenario | Expected behavior | Mechanism |
|---|---|---|
| Navigate forward to new page | Scroll to top | `window.scrollTo(0, 0)` on navigation |
| Press Back button | Restore previous scroll position | Browser native or manual save/restore |
| Navigate to hash anchor | Scroll to element | Browser native `#id` handling |
| Back to list after detail view | Restore list scroll position | Manual save in session storage or history state |
| Tab change within page | Restore tab's scroll position | Component-level scroll management |

---

## What Is This?

Scroll restoration is the behavior of managing scroll position across route navigations. In multi-page apps, browsers handle this automatically: navigating to a new page scrolls to the top; pressing Back restores the scroll position of the previous page. In SPAs, the browser's native scroll restoration often misfires — the page doesn't fully reload, so the browser tries to restore scroll on content that hasn't loaded yet, or new route content appears without scrolling to top.

```
MPA (native browser behavior):
  /products → scroll to top automatically
  press Back → browser restores /products scroll position
  ✓ works perfectly

SPA (broken without intervention):
  /products → no reload, scroll stays wherever it was
  press Back → browser tries to restore, but React hasn't rendered yet
  ✗ broken
```

> **Check yourself:** Why does the browser's native scroll restoration fail in SPAs?

---

## Why It Matters

Scroll restoration is a common source of UX bugs in SPAs — users navigate to a detail page and return to the list at the top, losing their place. Or content renders at the bottom of the page because scroll was never reset. Getting this right is a detail-oriented senior concern.

---

## Browser's `history.scrollRestoration`

Browsers have a native scroll restoration mechanism tied to the History API. You can control it:

```js
// 'auto': browser tries to restore scroll on popstate (default)
// 'manual': you handle all scroll restoration yourself
history.scrollRestoration = 'manual';
```

For SPAs, set `scrollRestoration = 'manual'` immediately. The browser's `'auto'` mode fires before your framework has rendered the new route's content, restoring scroll to a position that may not exist yet (page too short) or the wrong element.

---

## Scroll to Top on Navigation

The baseline: when navigating forward to a new page, scroll to top.

### React Router v6

```jsx
import { useEffect } from 'react';
import { useLocation } from 'react-router-dom';

function ScrollToTop() {
  const { pathname } = useLocation();

  useEffect(() => {
    window.scrollTo({ top: 0, behavior: 'instant' });
  }, [pathname]);

  return null;
}

// Place inside the router:
function App() {
  return (
    <BrowserRouter>
      <ScrollToTop />
      <Routes>
        {/* ... */}
      </Routes>
    </BrowserRouter>
  );
}
```

`behavior: 'instant'` avoids a visible scroll animation when the new page renders. `behavior: 'smooth'` on every navigation is distracting.

### React Router v6.4+ ScrollRestoration component

React Router ships a `<ScrollRestoration>` component that handles both scroll-to-top on forward navigation and position restoration on back navigation:

```jsx
import { ScrollRestoration } from 'react-router-dom';

function RootLayout() {
  return (
    <>
      <Outlet />
      <ScrollRestoration
        getKey={(location) => {
          // Use location.key (default) for full restoration
          // or location.pathname for restoration by path only
          return location.pathname;
        }}
      />
    </>
  );
}
```

---

## Restoring Scroll on Back Navigation

When the user navigates Back, they expect to see the list at the same position they left it. This requires:

1. Saving the scroll position when navigating away
2. Restoring it when navigating back

### Using history state

```js
// Before navigating away — save scroll position in history state
function navigateTo(path) {
  history.replaceState(
    { ...history.state, scrollY: window.scrollY },
    ''
  );
  history.pushState({}, '', path);
  window.scrollTo(0, 0);
}

// On popstate (back/forward) — restore scroll position
window.addEventListener('popstate', (event) => {
  const savedScrollY = event.state?.scrollY;
  renderRoute(location.pathname);

  if (savedScrollY !== undefined) {
    // Wait for content to render before restoring scroll
    requestAnimationFrame(() => {
      requestAnimationFrame(() => {
        window.scrollTo(0, savedScrollY);
      });
    });
  }
});
```

The double `requestAnimationFrame` gives the framework one render cycle to paint the content before scrolling to a position within it.

### Using session storage

```js
// Key by location key (unique per history entry in React Router)
function saveScrollPosition(key) {
  sessionStorage.setItem(`scroll-${key}`, String(window.scrollY));
}

function getScrollPosition(key) {
  const saved = sessionStorage.getItem(`scroll-${key}`);
  return saved ? parseInt(saved, 10) : 0;
}

// In a React hook:
function useScrollRestoration() {
  const location = useLocation();
  const key = location.key; // React Router assigns a unique key per entry

  // Save position before navigating away
  useEffect(() => {
    return () => saveScrollPosition(key); // cleanup fires before new render
  }, [key]);

  // Restore position after rendering
  useEffect(() => {
    const saved = getScrollPosition(key);
    window.scrollTo(0, saved);
  }, [key]);
}
```

---

## Hash-based Anchor Scroll

When a URL contains a hash (`/docs/guide#installation`), the browser should scroll to the element with that ID. In SPAs, this often doesn't work because the content is rendered after the initial HTML — the element doesn't exist when the browser tries to scroll to it.

### React Router v6 — automatic hash scroll

React Router handles hash scrolling automatically for same-page navigation when using `<Link to="#section">` within the same route. For initial load:

```jsx
function ScrollToHash() {
  const { hash } = useLocation();

  useEffect(() => {
    if (!hash) return;

    const id = hash.slice(1); // remove #
    const element = document.getElementById(id);

    if (element) {
      element.scrollIntoView({ behavior: 'smooth', block: 'start' });
    } else {
      // Element not yet rendered — wait for it
      const observer = new MutationObserver(() => {
        const el = document.getElementById(id);
        if (el) {
          el.scrollIntoView({ behavior: 'smooth' });
          observer.disconnect();
        }
      });
      observer.observe(document.body, { childList: true, subtree: true });

      // Clean up if element never appears
      setTimeout(() => observer.disconnect(), 3000);
    }
  }, [hash]);

  return null;
}
```

---

## Scroll Restoration with Async Content

When a page loads data asynchronously, scroll restoration must wait for the content to render:

```jsx
function ProductList() {
  const { data, isLoading } = useQuery({ queryKey: ['products'], queryFn: fetchProducts });
  const location = useLocation();

  useEffect(() => {
    if (isLoading) return; // don't restore yet

    const savedY = sessionStorage.getItem(`scroll-${location.key}`);
    if (savedY) {
      window.scrollTo(0, parseInt(savedY, 10));
      sessionStorage.removeItem(`scroll-${location.key}`);
    }
  }, [isLoading, location.key]); // run when loading finishes

  // Save before navigation
  useEffect(() => {
    return () => {
      sessionStorage.setItem(`scroll-${location.key}`, String(window.scrollY));
    };
  }, [location.key]);

  if (isLoading) return <Skeleton />;
  return <List items={data} />;
}
```

The key insight: scroll restoration must trigger after the DOM is populated with enough content to scroll to the target position.

---

## Next.js Scroll Behavior

Next.js App Router resets scroll to top on navigation by default. Control it:

```jsx
// Disable scroll reset for a specific Link
<Link href="/products?page=2" scroll={false}>
  Next Page
</Link>

// Programmatic navigation with scroll control
import { useRouter } from 'next/navigation';
const router = useRouter();
router.push('/products', { scroll: false }); // don't scroll to top
```

For full scroll restoration (back button), Next.js handles this automatically in the App Router via its client-side router. The Pages Router requires manual implementation.

---

## Virtual Lists and Scroll Position

For infinite lists or virtualized content (react-window, react-virtual), scroll restoration is more complex — the list must know how many items to render before scrolling to the saved position:

```js
// With react-virtual:
function VirtualList({ data }) {
  const location = useLocation();
  const savedScrollKey = `virtual-scroll-${location.key}`;

  const rowVirtualizer = useVirtualizer({
    count: data.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 60,
    initialOffset: parseInt(sessionStorage.getItem(savedScrollKey) || '0', 10),
  });

  useEffect(() => {
    return () => {
      sessionStorage.setItem(
        savedScrollKey,
        String(rowVirtualizer.scrollOffset)
      );
    };
  }, [location.key]);

  // ...
}
```

---

## Gotchas

**1. The browser restores scroll before React renders content**
Without `history.scrollRestoration = 'manual'`, the browser fires scroll restoration synchronously on `popstate` — before your React components mount and the DOM is populated. The scroll targets nothing, or the content is too short, and you end up at the top anyway. Always set to `'manual'` in SPAs.

**2. Double scroll: both the browser and your code restore**
If you forget to disable the browser's auto restoration (`history.scrollRestoration = 'manual'`), you end up with two scroll operations fighting each other. One wins based on timing, which varies by browser. The result is inconsistent behavior.

**3. Scroll restoration on the wrong element**
`window.scrollTo` scrolls the document. If the scrollable area in your app is a `<div>` (common with fixed headers and scrollable main areas), `window.scrollY` is always 0. Save and restore scroll on the correct container element, not `window`.

**4. `behavior: 'smooth'` on restoration feels wrong**
Smooth scrolling for click-initiated anchor navigation is good UX. Smooth scrolling when restoring position on Back navigation is disorienting — the content pans from the wrong position to the right one. Use `behavior: 'instant'` for restoration.

**5. Scroll position saved in component cleanup fires too late in React strict mode**
In React 18 strict mode (development only), effects are double-invoked. The cleanup from the first invocation fires before the second mount, which may save a wrong scroll position. This is only a dev concern but makes restoration unreliable in development. The `location.key` approach is more robust than component lifecycle timing.

---

## Interview Questions

**Q (High): Why does the browser's native scroll restoration often fail in SPAs and how do you fix it?**

Answer: The browser's native scroll restoration (`history.scrollRestoration = 'auto'`) fires on the `popstate` event, which is synchronous and fires before the JavaScript framework has had a chance to render the content for the new route. At that moment, the DOM either shows the previous route's content (wrong element) or is empty (no content to scroll to). The scroll either snaps somewhere wrong or does nothing. Fix it by setting `history.scrollRestoration = 'manual'` immediately — this disables the browser's automatic restoration. You then take over: save scroll position before navigation (in history state or session storage), and restore it after the route component has rendered and the DOM is populated with the new content. The restoration must happen after render, not synchronously with navigation.

**Q (High): How do you restore scroll position when navigating back to a list page that loads data asynchronously?**

Answer: The restoration must be deferred until the async content has loaded, because scroll positions in content that doesn't exist yet are ignored. The pattern: (1) save the scroll position (keyed by `location.key`) before the user leaves the list page — in a cleanup effect that runs when the location changes; (2) after navigating back, in the list component, check for a saved position in session storage or history state; (3) in a `useEffect` that depends on the loading state, when `isLoading` transitions from `true` to `false`, execute `scrollTo` with the saved position. The data loading is the gate — don't attempt restoration until the content is rendered. Remove the saved position from storage after restoring to avoid stale restoration on subsequent visits.

**Q (Medium): How does React Router v6's `<ScrollRestoration>` component work?**

Answer: `<ScrollRestoration>` is placed in the root layout and manages scroll position automatically. On forward navigation, it scrolls to the top. On back/forward navigation (popstate), it restores the previously saved scroll position for that history entry. It uses `location.key` (which React Router assigns as a unique identifier per history entry) as the storage key, so different visits to the same path can have different scroll positions. The `getKey` prop lets you customize the storage key — for example, using `location.pathname` means all visits to the same path share a scroll position (restore the same position regardless of which history entry). It sets `history.scrollRestoration = 'manual'` on mount and serializes positions to session storage.

**Q (Medium): What is the double-rAF trick and why is it needed for scroll restoration?**

Answer: Double `requestAnimationFrame` — two nested `rAF` calls — gives the browser two animation frame cycles before executing code. The first `rAF` fires at the start of the next paint cycle (after the current script finishes). The second fires at the start of the following paint cycle. By the second frame, React (or any framework) has typically committed its render to the DOM, meaning the target elements actually exist. Without this delay, calling `scrollTo` immediately after triggering a re-render scrolls a DOM that hasn't been updated yet — either the elements don't exist or the page height is wrong. This is a pragmatic workaround for the lack of a "DOM committed and painted" hook. The cleaner alternative is to restore scroll inside a `useLayoutEffect` (fires after DOM mutation, before paint) or after an explicit data-loaded signal.

**Q (Low): When should you use `behavior: 'smooth'` vs. `behavior: 'instant'` for scroll in routing?**

Answer: Use `behavior: 'smooth'` for user-initiated in-page anchor navigation (clicking a "Jump to section" link) — the animated pan gives context for where on the page the target is. Use `behavior: 'instant'` for route navigation (scrolling to the top of a new page) and scroll restoration (restoring position on Back) — in both cases, the content is already at the destination and animating to it looks broken (panning from somewhere wrong to the correct position). As a rule: smooth scroll when the movement provides context; instant scroll when the position is just being initialized.

---

## Self-Assessment

- [ ] Explain why `history.scrollRestoration = 'manual'` is necessary in SPAs
- [ ] Implement a `ScrollToTop` component for React Router that runs on forward navigation
- [ ] Describe the double-rAF pattern and when it's needed
- [ ] Implement scroll position save and restore for a list page with async data
- [ ] Explain the difference between `behavior: 'smooth'` and `behavior: 'instant'` in routing contexts

---
*Phase 9 complete. Next: Phase 10 — Module Systems & Build Infrastructure, starting with ESM vs. CJS vs. UMD interop challenges.*
