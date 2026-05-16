# URL as State: Deep Linking & Shareability

## Quick Reference

| State type | Belongs in URL? | Example |
|---|---|---|
| Navigation destination | Yes | `/products/shoes` |
| Filters & search | Yes | `?category=shoes&sort=price` |
| Pagination | Yes | `?page=3` |
| Modal open (shareable) | Yes | `?modal=returns-policy` |
| Tab (shareable) | Yes | `?tab=reviews` |
| Ephemeral UI (dropdown open) | No | Keep in component state |
| Auth tokens, PII | Never | Security risk |
| Large datasets | No | URL length limits |

---

## What Is This?

URL as state is the principle that meaningful application state — anything a user might want to share, bookmark, or return to — should be encoded in the URL. A user who filters products by "shoes, size 10, under $100" and pastes the URL to a friend should land them on exactly that view.

Deep linking means that any view of the application is directly addressable by a URL. Not just the home page or top-level routes, but specific items, filtered lists, open panels, and selected tabs.

```
Without URL state:              With URL state:
User filters shoes              User filters shoes
copies URL → /products          copies URL → /products?category=shoes&size=10&maxPrice=100
friend lands on → all products  friend lands on → exact same filtered view
```

> **Check yourself:** Name three pieces of UI state that definitely belong in the URL and two that definitely don't.

---

## Why It Matters

Deep linking is a senior-level design concern that touches routing, state management, and UX. Interview questions often probe whether you understand the spectrum between "everything in URL" and "nothing in URL" and can reason about which state deserves URL persistence. Real-world bugs around filters being lost on refresh or share links not working are often rooted in poor URL state design.

---

## Query Parameters as State

Query parameters (`?key=value`) are the primary mechanism for state that doesn't change the route but modifies the view.

### Reading and writing with React Router

```jsx
import { useSearchParams } from 'react-router-dom';

function ProductList() {
  const [searchParams, setSearchParams] = useSearchParams();

  const category = searchParams.get('category') ?? 'all';
  const sort = searchParams.get('sort') ?? 'popular';
  const page = parseInt(searchParams.get('page') ?? '1', 10);

  function updateFilter(key, value) {
    setSearchParams(prev => {
      const next = new URLSearchParams(prev);
      if (value === null || value === undefined) {
        next.delete(key);
      } else {
        next.set(key, String(value));
      }
      next.set('page', '1'); // reset pagination on filter change
      return next;
    });
  }

  return (
    <div>
      <select
        value={category}
        onChange={e => updateFilter('category', e.target.value)}
      >
        <option value="all">All</option>
        <option value="shoes">Shoes</option>
        <option value="bags">Bags</option>
      </select>
      <ProductGrid category={category} sort={sort} page={page} />
    </div>
  );
}
```

`setSearchParams` behaves like React state — it triggers a re-render. Unlike `useState`, the state persists on refresh and is shareable.

### Vue Router

```js
import { useRoute, useRouter } from 'vue-router';
import { computed } from 'vue';

export function useQueryParam(key, defaultValue = null) {
  const route = useRoute();
  const router = useRouter();

  const value = computed(() => route.query[key] ?? defaultValue);

  function setValue(newValue) {
    router.replace({
      query: {
        ...route.query,
        [key]: newValue ?? undefined, // undefined removes the param
      },
    });
  }

  return [value, setValue];
}

// Usage in component:
const [category, setCategory] = useQueryParam('category', 'all');
const [page, setPage] = useQueryParam('page', '1');
```

---

## Encoding Complex State

### Arrays in query params

The URL spec doesn't define how arrays are encoded — different libraries use different conventions:

```
?colors=red&colors=blue          // repeated key (URLSearchParams native)
?colors[]=red&colors[]=blue      // bracket notation (PHP-style, qs library)
?colors=red,blue                 // comma-separated (custom)
```

Using the Web API:

```js
// Writing
const params = new URLSearchParams();
['red', 'blue', 'green'].forEach(color => params.append('colors', color));
// → colors=red&colors=blue&colors=green

// Reading
const colors = params.getAll('colors'); // ['red', 'blue', 'green']
```

### Structured state (use sparingly)

For complex state, encode as base64 JSON — but only when the structure is stable and the URL doesn't need to be human-readable:

```js
function encodeState(obj) {
  return btoa(JSON.stringify(obj));
}

function decodeState(encoded) {
  try {
    return JSON.parse(atob(encoded));
  } catch {
    return null; // handle malformed/old URLs gracefully
  }
}

// ?view=eyJsYXlvdXQiOiJncmlkIiwiY29sdW1ucyI6M30=
const viewState = decodeState(searchParams.get('view'));
```

This is appropriate for view configuration (grid vs. list, column visibility) but not for data that should be readable in the URL.

---

## URL-Addressable Modals

Opening a modal at a URL-backed path enables deep linking and browser history integration:

```jsx
// Pattern 1: Query parameter modal
// /products?modal=returns-policy
function ProductPage() {
  const [searchParams, setSearchParams] = useSearchParams();
  const isModalOpen = searchParams.has('modal');
  const modalId = searchParams.get('modal');

  function openModal(id) {
    setSearchParams(prev => {
      const next = new URLSearchParams(prev);
      next.set('modal', id);
      return next;
    });
  }

  function closeModal() {
    setSearchParams(prev => {
      const next = new URLSearchParams(prev);
      next.delete('modal');
      return next;
    });
  }

  return (
    <>
      <button onClick={() => openModal('returns-policy')}>Returns Policy</button>
      {isModalOpen && (
        <Modal id={modalId} onClose={closeModal} />
      )}
    </>
  );
}
```

```jsx
// Pattern 2: Parallel routes (Next.js App Router)
// Navigating to /photos/42 renders the photo modal over the gallery
// Pressing back returns to the gallery without the modal
// app/@modal/photos/[id]/page.js
```

The query parameter pattern is simpler; parallel routes are more powerful (can handle the back button natively and support server rendering of the modal).

---

## Replace vs. Push for State Updates

When updating URL state, choose between `pushState` (adds to history) and `replaceState` (overwrites current entry):

```jsx
// Replace: filter changes (back button should go to previous page, not prev filter)
setSearchParams({ category: 'shoes' }, { replace: true });

// Push: navigating to a distinct view the user might want to go back to
navigate('/products/42'); // new page — push

// Rules of thumb:
// - Filter/sort/search changes → replace
// - Pagination → replace (or push if the user should go back page-by-page)
// - Tab changes → replace
// - Opening a new view/record → push
// - Closing a modal → replace (or back())
```

---

## Syncing URL State with Server State

When URL params drive data fetching, keep them in sync with the fetch:

```jsx
function ProductList() {
  const [searchParams] = useSearchParams();
  const filters = Object.fromEntries(searchParams);

  // React Query: the query key includes all URL params
  // When URL changes, the key changes, and data refetches automatically
  const { data, isLoading } = useQuery({
    queryKey: ['products', filters],
    queryFn: () => fetchProducts(filters),
  });

  // ...
}
```

The URL is the single source of truth for the query — no separate filter state needed.

---

## URL Length Limits

Browsers and servers impose URL length limits. While the spec has no limit, practical limits:
- Chrome: ~2MB
- IE/Edge (legacy): 2,083 characters
- Nginx default: 4,096 bytes
- Most CDNs: 8,192 bytes

For complex filter states, prefer:
- Short param values (IDs, codes, not full text)
- POST requests for search with bodies instead of URL params
- Server-stored sessions with a short session ID in the URL

---

## Gotchas

**1. Treating URL state as write-only**
URL state is also input — on initial load, the URL is the source of truth. Components must read filter state from URL params, not from internal state that was "set once". Always initialize from the URL.

**2. Filter changes on one page affecting unrelated pages**
If you use `setSearchParams({ category: 'shoes' })` (replacing all params), you'll lose other params that were in the URL. Always merge:

```js
setSearchParams(prev => {
  const next = new URLSearchParams(prev);
  next.set('category', 'shoes');
  return next;
});
```

**3. Pushing to history on every filter change**
Changing a dropdown from "shoes" to "bags" should replace history, not push. If the user presses Back, they expect to go to the previous page, not the previous filter. Use `replace: true` for incremental state refinements.

**4. Not handling malformed URL params gracefully**
A user or external link might pass `?page=abc` or `?minPrice=-100`. Always parse defensively and fall back to safe defaults. `parseInt('abc', 10)` is `NaN` — use `|| 1` or `isNaN` checks.

**5. Encoding sensitive data in URLs**
URLs appear in browser history, server logs, Referer headers, and analytics. Never put tokens, passwords, PII, or sensitive identifiers in URL params. Use POST bodies or session storage for sensitive state.

---

## Interview Questions

**Q (High): Which application state should go in the URL and which should not? Give examples of each.**

Answer: State that belongs in the URL: anything that defines a shareable or bookmarkable view — the current route (path), search queries, active filters, sort order, pagination, selected tab, and URL-addressable modals. A user sharing a filtered product list URL should land their recipient on exactly that view. State that doesn't belong in the URL: ephemeral UI state that has no sharing value — whether a dropdown is open, the currently hovered element, animation states, scroll position (except explicitly bookmarked positions), and draft form values mid-entry. State that must never go in the URL: authentication tokens, passwords, PII (email, name), and sensitive identifiers that appear in server logs, browser history, and Referer headers.

**Q (High): Why should filter changes use `replace` instead of `push` in browser history?**

Answer: When a user changes a filter, they're refining the current view, not navigating to a distinct destination. If each filter change pushes a history entry, the Back button steps through every intermediate filter state instead of returning to the previous page. This is disorienting — pressing Back 5 times to escape a filtered list instead of going back to where you came from. Using `replace` means the current URL entry is overwritten; the user's back button correctly goes to the page before the filter was applied. Exception: if each filter state is genuinely a distinct view the user might want to navigate back to (like pagination pages in some products), push is appropriate.

**Q (High): How do you use React Query (or any data fetching library) with URL-driven state to avoid duplicating state?**

Answer: Make the query key include all relevant URL params. `useSearchParams()` returns the current params; include them — as an object or sorted string — in the `queryKey` array. When the URL changes (user changes a filter), the query key changes, and React Query automatically re-fetches with the new params. This makes the URL the single source of truth: there's no separate `filters` state in a store, no syncing logic between filter state and query state, no bugs from them going out of sync. The URL drives the key, the key drives the fetch, the fetch drives the UI.

**Q (Medium): How do you represent an open modal in the URL? What are the trade-offs between query parameters and parallel routes?**

Answer: Query parameter approach (`?modal=photo-42`): simple to implement, works in any framework, the base page stays at the same URL, and the modal state is easily readable. Trade-off: the modal is not a distinct history entry by default — pressing Back doesn't close the modal unless you push the modal open (not replace), which means pressing Back to close the modal removes it from history and the user can't re-open it with Forward. Parallel routes (Next.js App Router `@modal` slots): the modal renders as a separate route segment simultaneously with the underlying page. Navigation to the modal URL pushes history; pressing Back closes the modal naturally. Trade-off: complex setup, Next.js App Router specific, harder to implement in vanilla routing.

**Q (Low): What URL length limits should you be aware of and what should you do when URL state grows large?**

Answer: Practical limits: most CDNs cap URLs at ~8,192 bytes; Nginx defaults to 4,096 bytes; legacy IE caps at 2,083 characters. For complex filter states that exceed these limits, switch from GET query params to a short shareable ID pattern: serialize the full filter state to a backend (POST `/views` with the filter JSON), get back a short ID, and put just the ID in the URL (`?view=abc123`). When loading that URL, fetch the filter state by ID. This keeps URLs short and shareable while supporting complex state. Alternatively, use POST-based search endpoints that accept a request body — though these aren't directly shareable as URLs.

---

## Self-Assessment

- [ ] Implement a product filter component that reads/writes state to URL query params
- [ ] Explain why filter changes should use `replace` instead of `push`
- [ ] Describe how to use React Query with URL-driven state to avoid duplicated state
- [ ] Implement a URL-addressable modal using query parameters
- [ ] Name three URL state anti-patterns (what not to put in the URL and why)

---
*Next: Scroll Restoration — how browsers and SPAs handle scroll position across navigation.*
